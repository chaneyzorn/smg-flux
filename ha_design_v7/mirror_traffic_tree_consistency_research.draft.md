# SMG HA V7 方案调研 — 镜像流量能否解决 cache tree 分散

> 目标：验证"通过 Envoy/Higress 流量镜像，让所有 SMG 实例接收相同请求，从而建立一致的 cache_aware tree"这一假设是否成立。
>
> 调研范围：Envoy 镜像能力边界、SMG `cache_aware.rs` + `tree.rs` 源码的随机性/不确定性分析、并发插入顺序敏感性。

---

## 一、V7 方案的核心假设

```
外部请求
    │
    ▼
Higress Gateway
    │ ── 主请求 ──→ SMG-1.1 (处理请求 + 更新 tree)
    │ ── 镜像 ────→ SMG-1.2 (仅更新 tree, 不转发给 worker)
    │ ── 镜像 ────→ SMG-1.3 (仅更新 tree, 不转发给 worker)
```

**假设**：如果所有实例都接收同样的请求，它们的 `cache_aware` tree 会记录相同的历史，从而避免 tree 分散问题。

**需要验证的三个层面**：
1. Envoy 能否将同一请求镜像到组内所有实例？
2. SMG 代码层面能否区分"主请求"和"镜像请求"，只更新 tree 不转发给 worker？
3. **最关键**：即使流量一致、代码逻辑一致，tree 结构是否真的能保持一致？

---

## 二、Envoy 镜像能力（已确认可行，但有约束）

| 能力 | 状态 | 说明 |
|------|------|------|
| 多目标镜像 | ✅ Envoy 原生支持 (`request_mirror_policies` repeated) | Higress 需 EnvoyFilter 配置 |
| 自定义 header | ✅ `request_headers_mutations` | 可加 `X-SMG-Shadow: true` 标记 |
| 不阻塞原请求 | ✅ fire-and-forget | 镜像响应丢弃 |
| 请求体限制 | ⚠️ 默认 1MB 不镜像 | LLM 长上下文可能超限，可调 `retry_shadow_buffer_limit` |
| SSE/WebSocket | ❌ 不支持 | 流式场景无法镜像 |

**结论**：Envoy 镜像在技术上可以支撑 V7 方案，但 SSE 流式和超大请求体是硬边界。

---

## 三、SMG 代码层面的随机性分析（核心）

### 3.1 调研方法

逐行阅读 `src/policies/cache_aware.rs` 的 `select_worker()` 和 `src/policies/tree.rs` 的 `insert()` / `prefix_match_with_counts()`，标记所有可能导致同一输入产生不同输出的环节。

### 3.2 已确认的随机/不确定环节

#### ① Tree 缺失时 → **明确随机**（cache_aware.rs:501-502）

```rust
// 当某个 (pool, model) 的 tree 尚未建立时
warn!("...falling back to random worker selection...");
let mut rng = rand::rng();
let random_idx = rng.random_range(0..healthy_indices.len());
Some(healthy_indices[random_idx])
```

**影响**：两个完全相同的 SMG 实例，启动后处理第一批请求时，即使 worker 列表相同，也可能因为随机选择而建立不同的初始 tree。

---

#### ② Load 相同时 → **DashMap 迭代顺序依赖**（cache_aware.rs:329-332）

```rust
let min_load_idx = healthy_indices
    .iter()
    .min_by_key(|&&idx| workers[idx].load())
    .copied()?;
```

当多个 worker 的 `load()` 相同时，`.min_by_key` 选择 `healthy_indices` 中的**第一个**。而 `healthy_indices` 来自：

```rust
// mod.rs:136
pub(crate) fn get_healthy_worker_indices(workers: &[Arc<dyn Worker>]) -> Vec<usize> {
    workers.iter().enumerate().filter(...).map(...).collect()
}
```

`workers` 数组来自 `WorkerRegistry::get_workers_filtered()`，内部遍历 `self.workers`（一个 `DashMap`）。**`DashMap` 的迭代顺序不是确定性的**——即使插入顺序相同，rehash 也可能改变遍历顺序。

**影响**：两个实例收到同一请求、同一时刻、worker 负载相同，但选择的 worker 可能不同，导致 `tree.insert(text, worker_url)` 记录不同的映射。

---

#### ③ Cache hit path 的 worker 匹配 → **迭代顺序依赖**（cache_aware.rs:435-439）

```rust
let tenant_url: &str = &result.tenant;
workers
    .iter()
    .position(|w| w.url() == tenant_url)
    .filter(|&idx| workers[idx].is_healthy())
```

`position()` 返回第一个匹配的 worker，同样取决于 `workers` 数组的迭代顺序。

---

#### ④ Tenant 选择 → **DashMap 迭代顺序依赖**（tree.rs:578-583）

```rust
let t = curr
    .tenant_last_access_time
    .iter()
    .next()
    .map(|kv| Arc::clone(kv.key()))
    .unwrap_or_else(|| Arc::from("empty"));
```

`prefix_match_with_counts` 在匹配到节点后，需要获取该节点关联的 tenant。它先尝试 `last_tenant` 缓存，缓存失效时遍历 `tenant_last_access_time`（`DashMap`）取第一个。**`DashMap.iter().next()` 的顺序不确定**。

这意味着：同一 text 在同一 tree 结构中，两次查询可能返回不同的 tenant（如果该节点有多个 tenant 的历史记录）。

---

#### ⑤ 概率性时间戳更新 → **时机不确定**（tree.rs:604）

```rust
let epoch = get_epoch();
if epoch & 0x7 == 0 {  // 1/8 概率
    curr.tenant_last_access_time.insert(...);
}
```

时间戳更新是概率性的。两个实例处理相同请求时，由于各自的 `EPOCH_COUNTER` 独立递增，一个实例可能触发更新而另一个没有，导致后续 LRU eviction 的淘汰顺序不同。

---

#### ⑥ Node split 时的 tenant 复制 → **状态分歧风险**（tree.rs:448-477）

当插入新 text 导致已有 node 需要 split 时：

```rust
let new_node = Arc::new(Node {
    tenant_last_access_time: matched_node.tenant_last_access_time.clone(),
    last_tenant: parking_lot::RwLock::new(
        matched_node.last_tenant.read().clone(),
    ),
});
```

`DashMap.clone()` 复制的是整个 map 的状态。但后续：

```rust
if !new_node.tenant_last_access_time.contains_key(tenant_id.as_ref()) {
    self.tenant_char_count
        .entry(Arc::clone(&tenant_id))
        .and_modify(|count| *count += matched_text_count)
        .or_insert(matched_text_count);
    new_node.tenant_last_access_time.insert(..., 0);
}
```

`contains_key` + `insert` 是两个独立操作，不是原子的。在并发场景下（多个请求同时插入），两个实例的 race condition 结果可能不同。

---

### 3.3 随机性汇总

| 环节 | 类型 | 是否可消除 |
|------|------|-----------|
| Tree 缺失时的 worker 选择 | `rand::rng()` 显式随机 | ❌ 设计如此（seed 机制未暴露） |
| Load 相同时的 worker 选择 | `DashMap` 迭代顺序 | ❌ DashMap 不保证顺序 |
| Tenant 选择 | `DashMap.iter().next()` | ❌ 同上 |
| 时间戳更新 | 概率性 1/8 | ❌ 性能优化设计 |
| Node split 的并发 | Race condition | ❌ 非原子操作 |

**结论：tree 的建立过程中存在至少 5 个随机/不确定环节，这些都不是配置可以消除的。**

---

## 四、架构层面的不可能性

### 4.1 插入顺序敏感性

SMG 的 tree 是一个**radix tree**（前缀树），其结构对插入顺序敏感：

- 插入 `"hello"` 再插入 `"help"` → 产生一次 node split
- 插入 `"help"` 再插入 `"hello"` → 产生同样的 split，但时间戳不同

如果两个实例接收请求的**时序**不同（即使是镜像流量，网络延迟也不同），node split 的时机不同，导致：
- `tenant_last_access_time` 中 tenant 的数量不同
- `last_tenant` 缓存的内容不同
- 后续查询的 tenant 选择结果不同

### 4.2 并发请求的竞争

镜像流量不能保证两个实例以相同的顺序、相同的并发度处理请求。如果实例 A 和实例 B 同时处理两个请求：

- 实例 A：请求 1 先完成 insert，请求 2 后完成
- 实例 B：请求 2 先完成 insert，请求 1 后完成

由于 `DashMap` 的并发插入没有全局顺序保证，两个实例的 tree 结构会分歧。

### 4.3 Epoch 计数器独立

```rust
static EPOCH_COUNTER: AtomicU64 = AtomicU64::new(0);
```

`EPOCH_COUNTER` 是**进程内全局变量**，每个 SMG 实例有自己独立的计数器。两个实例的时间戳没有任何关联，LRU eviction 的淘汰顺序完全不同。

---

## 五、与 V6 的对比

| 维度 | V7（镜像流量） | V6（事件同步） |
|------|---------------|---------------|
| 一致性保证 | ❌ 无（随机性+时序敏感） | ✅ 有（操作日志有序同步） |
| 对 SMG 代码的侵入 | 中（需加 shadow 分支） | 高（需实现接收端） |
| 运行时开销 | 高（所有实例处理所有请求） | 低（只同步 tree 操作事件） |
| 网络开销 | 高（请求体 N 倍复制） | 低（只传操作日志） |
| SSE/流式支持 | ❌ 不支持 | ✅ 不影响 |
| 大请求支持 | ⚠️ 需调 buffer 限制 | ✅ 不影响 |

---

## 六、结论

**V7 方案的假设不成立。**

> "通过镜像流量让所有实例接收相同请求，从而建立一致的 cache_aware tree"

这个假设忽略了 SMG `cache_aware` 实现中的多层随机性和不确定性：

1. **显式随机**：tree 缺失时 `rand::rng()` 随机选择 worker
2. **隐式随机**：`DashMap` 迭代顺序不确定，影响 load 相同时的 worker 选择和 tenant 选择
3. **概率性更新**：1/8 概率的时间戳更新导致 LRU 状态分歧
4. **时序敏感**：radix tree 对插入顺序敏感，镜像流量不能保证处理时序一致
5. **并发竞争**：node split 时的非原子操作在并发场景下产生状态分歧

**即使流量 100% 一致、即使 SMG 代码配合实现 shadow 模式，tree 结构仍然无法保持一致。**

相比之下，V6（事件同步）通过**有序的操作日志同步**来确保一致性，绕过了所有随机性环节，是更根本的解法。

---

## 七、参考来源

- SMG `cache_aware.rs` — `select_worker()` / `select_worker_min_load()`  
  https://github.com/sgl-project/sglang/blob/main/sgl-model-gateway/src/policies/cache_aware.rs

- SMG `tree.rs` — `insert()` / `prefix_match_with_counts()` / `get_epoch()`  
  https://github.com/sgl-project/sglang/blob/main/sgl-model-gateway/src/policies/tree.rs

- SMG `mod.rs` — `get_healthy_worker_indices()`  
  https://github.com/sgl-project/sglang/blob/main/sgl-model-gateway/src/policies/mod.rs

- `dashmap` crate — 迭代顺序说明  
  https://docs.rs/dashmap/latest/dashmap/struct.DashMap.html（明确说明不保证遍历顺序）

- Envoy `request_mirror_policies` 官方文档  
  https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto.html

- Higress Ingress 镜像注解  
  https://higress.ai/docs/latest/user/annotation-use-case/
