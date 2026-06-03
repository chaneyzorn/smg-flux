# SMG 主备方案故障影响分析 — 单轮 vs 多轮对话

> 从完整请求链路角度，评估 SMG 节点故障对单轮对话和多轮对话的影响。

---

## 一、请求链路总览

### 1.1 单轮对话 PD 流程

```
Client
  │ ① 发送请求（prompt, model, stream 等）
  ▼
[Higress Gateway]
  │ ② 负载均衡选择一个 SMG（或 Service）
  ▼
[SMG]
  │ ③ cache_aware 前缀树查找
  │    - 命中：选到已有缓存的 P worker
  │    - 未命中：fallback 到最短队列选 P worker
  │ ④ 同时选 D worker（独立策略）
  │ ⑤ 向 P 注入 bootstrap 元数据，向 D 注入相同元数据
  ▼
[Prefill Worker]         [Decode Worker]
  │ ⑥ 计算 prefix KV        │ ⑦ 通过 bootstrap 连接 P
  │                         │ ⑧ 拉取 KV Cache
  │                         │ ⑨ 自回归 decode 生成 token
  ▼                         ▼
[SMG] ←────────────────────┘ ⑩ 收集响应
  │
  ▼
Client ← ⑪ 返回结果（streaming 或 complete）
```

### 1.2 多轮对话 PD 流程（第 N 轮）

```
Client
  │ ① 发送请求（包含前 N-1 轮完整历史）
  ▼
[Higress Gateway]
  │ ② 负载均衡 →  hopefully 同一个 SMG（或同一组）
  ▼
[SMG]
  │ ③ cache_aware 前缀树查找
  │    - 前缀 = "turn1 turn2 ... turn(N-1)"
  │    - 高概率命中（树中已插入过 turn1, turn1 turn2, ...）
  │    - 选到之前处理该对话的 P worker
  │ ④ 选到之前处理该对话的 D worker（或同组内的新 D worker）
  │ ⑤ 向 P/D 注入 bootstrap 元数据
  ▼
[Prefill Worker]
  │ ⑥ Radix Cache match_prefix("turn1 turn2 ... turn(N-1)")
  │    - 命中：只计算第 N 轮新增 token 的 KV
  │    - 这是多轮对话的核心收益！
  ▼
[Decode Worker]
  │ ⑦ 拉取 KV，decode 生成第 N 轮回复
  ▼
Client ← 返回结果
```

**多轮对话的核心收益**：第 N 轮只需要计算**新增部分**的 KV，前 N-1 轮的 KV 直接从 Radix Cache 复用。这让第 2+ 轮的延迟接近纯 decode 时间，而非完整的 prefill + decode。

---

## 二、当前现状（无 HA，Higress 轮询 3 个 SMG）

### 2.1 单轮对话：SMG-1 挂了

**故障前**：Higress 轮询 SMG-1 / SMG-2 / SMG-3，请求均匀分布。

**故障瞬间**：
- 正在 SMG-1 处理的请求：中断，客户端收到 502/503
- Higress outlier detection 检测到 SMG-1 连续失败
- Higress 将 SMG-1 标记为不健康，后续请求只发向 SMG-2 / SMG-3

**故障后（客户端重试）**：
- 客户端重试请求被 Higress 发到 SMG-2 或 SMG-3
- SMG-2/3 的 cache_aware 树中**没有该请求的历史**（因为是新请求）
- 但这没关系，单轮请求本身没有前缀历史要复用
- SMG-2/3 通过 cache_aware 选择 P/D worker，正常处理

**影响评估**：
- 正在处理的请求：**失败，需客户端重试**
- 新请求：无额外影响（单轮无历史）
- Higress 层面：2/3 容量承载全部流量，负载上升 50%

### 2.2 多轮对话：SMG-1 挂了

**场景设定**：用户进行 5 轮对话，第 1-3 轮被 Higress 路由到 SMG-1。

**SMG-1 的 cache_aware 树状态**（第 3 轮结束后）：
- 树中已插入：
  - `"turn1"` → `Prefill-2`
  - `"turn1 turn2"` → `Prefill-2`
  - `"turn1 turn2 turn3"` → `Prefill-2`
- SMG-1 已将这 3 个对话前缀都路由到 `Prefill-2`
- `Prefill-2` 的 Radix Cache 中已缓存了 `"turn1 turn2 turn3"` 的完整 KV

**第 4 轮时，SMG-1 挂了**：

1. Higress 将第 4 轮请求路由到 SMG-2
2. SMG-2 的 cache_aware 树中**没有该对话的任何历史**
3. SMG-2 的前缀匹配 `"turn1 turn2 turn3 turn4_prompt"` → **完全未命中**
4. SMG-2 fallback 到最短队列策略，选择一个负载最低的 P worker（假设是 `Prefill-3'`）
5. `Prefill-3'` 的 Radix Cache 中没有 `"turn1 turn2 turn3"` 的 KV
6. `Prefill-3'` 需要从头计算 `"turn1 turn2 turn3 turn4_prompt"` 的**全部 KV**
7. 这意味着第 4 轮的 prefill 延迟 = 第 1 轮（完整 prefill）的延迟，而非多轮对话应有的"增量 prefill"延迟

**第 5 轮**：
- 如果 Higress 继续将请求发到 SMG-2：
  - SMG-2 的 cache_aware 树中现在只有 `"turn1 turn2 turn3 turn4_prompt"` → `Prefill-3'`
  - 第 5 轮前缀 `"turn1 turn2 turn3 turn4_prompt turn4_resp turn5_prompt"`
  - 前缀匹配可能部分命中 `"turn1 turn2 turn3 turn4_prompt"`，但 turn4_resp 和 turn5_prompt 是新的
  - 即使部分命中，`Prefill-3'` 也没有 turn1-3 的 KV，仍然需要重新计算大部分前缀

**影响评估**：
- 正在处理的对话（第 4 轮及以后）：**延迟暴增**（从"增量 prefill"退化为"全量 prefill"）
- 用户体验：多轮对话越到后面本应越快，但 SMG 故障后反而**回到了第 1 轮的延迟水平**
- 如果对话很长（如 20 轮），前缀可能有几千 tokens，重新计算的成本极高
- **这是当前架构最严重的痛点**

---

## 三、官方推荐方案：Session Affinity（一致性哈希）

### 3.1 方案原理

Higress 基于 user ID 或 API key 做一致性哈希，让同一用户的请求尽量落到同一个 SMG：

```
User A ──hash("user-a-id")──► SMG-1
User B ──hash("user-b-id")──► SMG-2
User C ──hash("user-c-id")──► SMG-3
```

### 3.2 单轮对话：SMG-1 挂了

**故障前**：User A 的请求一直落到 SMG-1。

**故障后**：
- SMG-1 被 Higress outlier detection 移除
- 一致性哈希环重新计算，User A 的请求改落到 SMG-2 或 SMG-3
- User A 的新请求（单轮）没有历史前缀，SMG-2/3 正常处理

**影响评估**：
- 和当前现状类似：正在处理的请求失败，新请求无额外影响
- 但 User A 的请求现在固定到新的 SMG，该 SMG 会逐渐建立 User A 的 cache_aware 树

### 3.3 多轮对话：SMG-1 挂了

**场景设定**：User A 的 1-3 轮对话固定到 SMG-1（一致性哈希）。

**第 4 轮时，SMG-1 挂了**：

1. 一致性哈希环重新计算，User A 的请求改落到 SMG-2
2. SMG-2 的 cache_aware 树中**没有 User A 的对话历史**
3. 前缀匹配失败，fallback 选新的 P worker
4. 新的 P worker Radix Cache miss，需要全量 prefill
5. **第 4 轮延迟暴增**

**第 5 轮及以后**：
- User A 的请求继续固定到 SMG-2（一致性哈希）
- SMG-2 的 cache_aware 树中现在有了 `"turn1 turn2 turn3 turn4_prompt"` → 某个 P worker
- 第 5 轮前缀 `"turn1 turn2 turn3 turn4_prompt turn4_resp turn5_prompt"`
- SMG-2 的树部分命中 `"turn1 turn2 turn3 turn4_prompt"`
- 但**这个 P worker 没有 turn1-3 的 KV**！因为第 4 轮是 SMG-2 第一次处理 User A，它选了一个和 SMG-1 不同的 P worker

**关键问题**：
- Session Affinity 只保证了"同一用户的请求落到同一 SMG"
- 但**不保证**新的 SMG 会选到和之前相同的 P worker
- 即使新的 SMG 的 cache_aware 树逐渐建立，树中记录的 worker 地址和之前 SMG 记录的 worker 地址大概率不同
- 所以 P worker 的 Radix Cache 仍然 miss

**影响评估**：
- 切换后的**每一轮**对话都需要全量 prefill（直到对话结束）
- 因为新的 SMG 选的 P worker 和之前的 P worker 不同，KV Cache 无法复用
- **比现状好的一点**：至少同一用户的请求不会再在不同 SMG 之间漂移，新的 SMG 可以稳定地建立自己的树
- **但核心问题未解决**：切换 = 完全 cache miss

---

## 四、推荐方案 A：K8s Service + Readiness 主备切换

### 4.1 方案原理

```
Higress
  │ 轮询 3 个 Service
  ├─────────────┬─────────────┬─────────────┐
  ▼             ▼             ▼
Service-1    Service-2    Service-3
  │             │             │
  ▼             ▼             ▼
主(活跃)      主(活跃)      主(活跃)
备(静默)      备(静默)      备(静默)

主和备共享同一组 P/D Worker
```

**关键前提**：SMG-1-主 和 SMG-1-备 配置**相同的 P/D worker 地址**，共享同一组 worker 池。

### 4.2 单轮对话：SMG-1-主 挂了

**故障前**：Service-1 的 endpoints 中只有 SMG-1-主（readiness=200），备（readiness=503）不在 endpoints 中。

**故障瞬间**：
1. SMG-1-主进程崩溃或无响应
2. K8s readiness probe 连续失败，将 SMG-1-主从 Service-1 endpoints 移除
3. Higress 通过 DNS 解析 Service-1，发现无可用端点
4. 正在发往 SMG-1-主的请求：失败（客户端收到 502/503）

**故障后（备接管）**：
1. SMG-1-备的 peer 探测检测到主连续失败
2. SMG-1-备的 `/readiness` 返回 200
3. K8s 将 SMG-1-备加入 Service-1 endpoints
4. Higress DNS 解析 Service-1，获得 SMG-1-备的 IP
5. 后续请求重新发到 Service-1（现在指向 SMG-1-备）

**单轮请求在 SMG-1-备的处理**：
- SMG-1-备的 cache_aware 树：
  - **Phase 1（无 mesh 同步）**：空树
  - **Phase 2（有 mesh 同步）**：和主基本一致
- 单轮请求本身无历史前缀，cache_aware 树是否命中不影响结果
- SMG-1-备通过 cache_aware 选择 P/D worker（树空时 fallback 到最短队列）
- 由于主备共享同一组 P/D worker，备可以选到**和主之前相同的 P/D worker**
- P worker 正常处理，单轮请求成功

**影响评估**：
- 切换期间（~5-10s）正在处理的请求：失败，需客户端重试
- 切换后新请求：无额外影响（单轮无历史）
- P/D worker 层：不受影响，继续运行，只是从"主发请求"变为"备发请求"

### 4.3 多轮对话：SMG-1-主 挂了

**场景设定**：User A 的 1-3 轮对话都发到 Service-1（Higress 轮询），由 SMG-1-主处理。

**SMG-1-主的状态**（第 3 轮结束后）：
- cache_aware 树：
  - `"turn1"` → `Prefill-2`
  - `"turn1 turn2"` → `Prefill-2`
  - `"turn1 turn2 turn3"` → `Prefill-2`
- P/D worker 状态：`Prefill-2` 的 Radix Cache 已缓存 `"turn1 turn2 turn3"` 的 KV

**第 4 轮时，SMG-1-主 挂了**：

#### 场景 A：Phase 1 MVP（无 mesh 同步）

1. SMG-1-备接管，readiness=200
2. 第 4 轮请求发到 SMG-1-备
3. SMG-1-备的 cache_aware 树：**空树**
4. 前缀匹配 `"turn1 turn2 turn3 turn4_prompt"` → **完全未命中**
5. Fallback 到最短队列策略，选择负载最低的 P worker
6. **关键问题**：备会选到哪个 P worker？
   - 如果备的 worker 负载和主相同时：可能选到 `Prefill-2`（和主一样）
   - 如果备的 worker 负载不同：可能选到 `Prefill-1` 或其他 worker
7. **情况分析**：
   - **幸运情况（概率 1/x，x=P worker 数量）**：备选到了 `Prefill-2`
     - `Prefill-2` 的 Radix Cache 命中 `"turn1 turn2 turn3"`
     - 只计算 turn4_prompt 的增量 KV
     - 延迟正常！
   - **不幸情况（概率 (x-1)/x）**：备选到了其他 P worker
     - 该 P worker 的 Radix Cache 没有 `"turn1 turn2 turn3"`
     - 需要全量 prefill
     - 延迟暴增

8. **第 5 轮及以后**：
   - SMG-1-备的 cache_aware 树中现在有了 `"turn1 turn2 turn3 turn4_prompt"` → 某个 P worker
   - 后续轮次在该 P worker 上逐渐建立 Radix Cache
   - 但**如果第 4 轮选错了 P worker**，turn1-3 的 KV 永远 miss，直到对话结束

**影响评估（Phase 1）**：
- 切换后的第 1 轮对话：有概率（(x-1)/x）延迟暴增
- 如果选对了 P worker：几乎无损
- 如果选错了 P worker：后续每一轮都需要全量 prefill（因为树中记录的是"错"的 worker）
- **这是一个概率性问题，不稳定**

#### 场景 B：Phase 2（有 mesh 同步）

1. SMG-1-备接管
2. 第 4 轮请求发到 SMG-1-备
3. SMG-1-备的 cache_aware 树：通过 mesh 同步，和主基本一致
4. 前缀匹配 `"turn1 turn2 turn3 turn4_prompt"`：
   - 树中已有 `"turn1 turn2 turn3"` → `Prefill-2`
   - 匹配率 > cache_threshold，命中 `Prefill-2`
5. SMG-1-备选到 `Prefill-2`（和主之前选择的相同）
6. `Prefill-2` 的 Radix Cache 命中 `"turn1 turn2 turn3"`
7. 只计算 turn4_prompt 的增量 KV
8. **延迟正常，完全无缝**

**影响评估（Phase 2）**：
- 切换后的所有轮次：cache_aware 树无缝继承，选到相同的 P worker
- P worker 的 Radix Cache 持续命中
- **几乎无感知**

---

## 五、方案对比总结

### 5.1 单轮对话影响对比

| 评估维度 | 当前现状（轮询） | 官方 Session Affinity | 主备方案 Phase 1 | 主备方案 Phase 2 |
|---------|---------------|---------------------|----------------|----------------|
| **故障瞬间** | 正在处理的请求失败 | 同左 | 同左 | 同左 |
| **切换延迟** | Higress outlier detection（~几秒） | 同左 | K8s readiness probe（~5-10s） | 同左 |
| **新请求处理** | 正常，无历史影响 | 正常 | 正常 | 正常 |
| **P/D worker 影响** | 无，worker 独立运行 | 无 | 无，备共享同一组 worker | 无 |
| **整体评价** | 可接受 | 可接受 | 可接受 | 可接受 |

**结论**：单轮对话对 SMG 故障不敏感，四种方案差异不大。

### 5.2 多轮对话影响对比

| 评估维度 | 当前现状（轮询） | 官方 Session Affinity | 主备方案 Phase 1 | 主备方案 Phase 2 |
|---------|---------------|---------------------|----------------|----------------|
| **切换后第 1 轮** | 全量 prefill（延迟暴增） | 全量 prefill（延迟暴增） | 大概率全量 prefill（延迟暴增） | **增量 prefill（延迟正常）** |
| **切换后第 2+ 轮** | 如果落到不同 SMG，继续 miss | 如果选到不同 P worker，继续 miss | 如果第 1 轮选错 P worker，继续 miss | **持续命中，延迟正常** |
| **cache_aware 树状态** | 分散在多个 SMG，无法继承 | 新的 SMG 从零开始建树 | 备从空树开始，逐渐建立 | **备继承主的树，无缝** |
| **Radix Cache 复用** | 几乎不可能 | 几乎不可能（不同 P worker） | 靠运气（1/x 概率选对） | **100% 复用** |
| **用户体验** | 多轮对话突然变慢 | 多轮对话突然变慢 | 多轮对话可能突然变慢 | **几乎无感知** |
| **可靠性** | 低 | 中（至少请求不漂移） | 中（靠概率） | **高** |

**结论**：多轮对话是主备方案的核心价值场景。Phase 1 仍有概率性问题，Phase 2（mesh 同步）才能真正解决多轮对话的 cache miss 问题。

---

## 六、更深层的问题：Bootstrap Room 和 P/D 连接状态

### 6.1 当前被忽略的细节

在 PD 分离中，SMG 向每个请求注入 `bootstrap_room`（随机生成的 room ID）：

```rust
// src/routers/http/pd_types.rs:60-63
pub fn generate_room_id() -> u64 {
    rand::random::<u64>() & (i64::MAX as u64)
}
```

这意味着：
- 即使是**相同的 prompt**，每次请求的 `bootstrap_room` 也是**不同的**
- D worker 通过 `(bootstrap_host, bootstrap_port, bootstrap_room)` 三元组连接到 P worker 获取 KV

### 6.2 这对主备切换的影响

**如果主备共享同一组 P/D worker**：

- 主处理的请求：P worker 创建了 room A 的 KV
- 主挂了，备接管后处理新请求：P worker 创建 room B 的 KV
- 即使备选到了同一个 P worker，**room ID 不同**，D worker 仍然需要重新从 P 拉取 KV

**但是**：P worker 的 **Radix Cache** 是按 token ID 序列索引的，不是按 room ID 索引的。所以：
- 如果两个请求的 prefix token 序列相同，P worker 的 Radix Cache 会命中
- `bootstrap_room` 只是用于 D→P 的 KV 传输通道标识，不影响 KV Cache 的复用

**验证**：从 `sglang/python/sglang/srt/mem_cache/radix_cache.py` 中可以看到，Radix Cache 的 key 是 `RadixKey(token_ids, extra_key)`，匹配基于 token ID 序列。`bootstrap_room` 不参与到 Radix Cache 的 key 中。

### 6.3 结论

`bootstrap_room` 的随机性**不影响** Radix Cache 的命中。主备切换后，只要：
1. 备选到**相同的 P worker**
2. P worker 的 Radix Cache 中有**相同的 prefix token 序列**

KV Cache 就可以复用，与 room ID 无关。

---

## 七、最终建议

### 7.1 如果业务以单轮对话为主

- **当前现状即可接受**，SMG 故障只影响正在处理的请求
- 如果追求更高可用，**Phase 1 MVP（无 mesh 同步）** 已经足够

### 7.2 如果业务以多轮对话为主（Agent、Chatbot 等）

- **必须做 Phase 2（mesh 同步）**
- Phase 1 的"靠概率选对 P worker"在多轮对话场景下不可接受
- 只有 mesh 同步能确保备的 cache_aware 树和主完全一致，从而稳定选到相同的 P worker

### 7.3 折中方案：不 mesh 同步，但强制绑定 P worker

如果不想做 mesh 同步，有一个折中方案：

- 在 SMG 的 cache_aware 树中，不只在 SMG 内存中维护树，而是将 `(prefix → worker_url)` 映射**持久化到 Redis/共享存储**
- 主和备都读同一个共享存储
- 这样即使备从空树开始，第一次请求后从共享存储读到映射，后续请求就能稳定选到正确的 P worker

但这引入了新的外部依赖，复杂度增加。

### 7.4 最简可行路径

| 业务场景 | 推荐方案 |
|---------|---------|
| 单轮为主（RAG、Embedding 等） | Phase 1 MVP： readiness 主备切换，不做 mesh 同步 |
| 多轮为主（Agent、Chatbot） | Phase 2：readiness 主备切换 + 补全 mesh 接收端 |
| 混合场景 | Phase 2，因为单轮场景也能受益于 mesh 同步 |
