# SMG Active-Standby HA 架构 V6 方案调研 — 对等无主架构与 Cache Tree 同步

> 目标：深入分析 SMG `cache_aware` 路由策略在多 Router 实例部署下的**树分散问题**，探索通过**对等同步**实现无主架构的可行性，避免主备切换带来的 cache miss。
>
> 本方案基于对 sglang 源码的深度分析，结合业界同类系统（NVIDIA Dynamo、llm-d）的工程实践。

---

## 一、问题本质：cache_aware 树为什么会分散？

### 1.1 CacheAwarePolicy 的核心机制（基于源码分析）

`sgl-model-gateway/src/policies/cache_aware.rs` 中的 `CacheAwarePolicy` 是 SMG 的**主版本 cache_aware 策略**：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    外部请求（prompt text）                                 │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│                    Router 实例 A（独立 Tree）                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  DashMap<String, Arc<Tree>>                                      │  │
│  │  tree_key = "pool::model" (如 "regular::gpt-4")                   │  │
│  │                                                                  │  │
│  │  Tree 结构：近似 radix tree，存储 (text_prefix → worker_url)       │  │
│  │  - 每个 insert：tree.insert(text, worker_url)                      │  │
│  │  - 每个 match：tree.prefix_match(text) → 匹配率最高的 worker       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  select_worker 逻辑：                                                     │
│  1. 负载不均衡？→ 最短队列路由                                             │
│  2. match_rate > cache_threshold？→ 路由到匹配的 worker                  │
│  3. 否则 → 路由到负载最低的 worker                                        │
│  4. 无论走哪条路，最后都执行 tree.insert(text, selected_worker_url)      │
└─────────────────────────────────────────────────────────────────────────┘
```

**关键代码**（`cache_aware.rs:374-469`）：

```rust
impl LoadBalancingPolicy for CacheAwarePolicy {
    async fn select_worker(...) -> Option<usize> {
        // ... 负载均衡判断 ...
        let tree = self.trees.get(&tree_key).map(|entry| entry.value().clone());
        if let Some(tree) = tree {
            let result = tree.prefix_match_with_counts(text);
            let match_rate = result.matched_char_count as f32 / result.input_char_count as f32;
            let selected_idx = if match_rate > self.config.cache_threshold {
                // Cache hit: 路由到匹配的 worker
                workers.iter().position(|w| w.url() == result.tenant)
            } else {
                // Cache miss: 路由到负载最低的 worker
                healthy_indices.iter().min_by_key(|&&idx| workers[idx].load())
            };
            // ⚠️ 无论选哪个 worker，都执行 tree.insert(text, worker_url)
            tree.insert(text, workers[idx].url());
            // ...
        }
    }
}
```

### 1.2 树分散问题的根本原因

当**多个 Router 实例并行运行**时（如 K8s 部署了 3 个 SMG Router Pod，通过 Service 轮询接收请求）：

| 时间 | 请求 | 到达的 Router | Router 的 Tree 状态 | 路由结果 | 各 Router Tree 变化 |
|------|------|--------------|-------------------|---------|-------------------|
| t1 | "hello world" | Router-A | 空 | 随机 → w1 | A: {"hello world" → w1} |
| t2 | "hello world" | Router-B | 空 | 随机 → w2 | B: {"hello world" → w2} |
| t3 | "hello world" | Router-C | 空 | 随机 → w3 | C: {"hello world" → w3} |
| t4 | "hello world" | Router-A | {"hello world" → w1} | cache hit → w1 | A: 更新 timestamp |
| t5 | "hello world" | Router-B | {"hello world" → w2} | cache hit → w2 | B: 更新 timestamp |

**问题**：
1. **每个 Router 只看到自己接收到的请求**，tree 内容互不相同
2. **同一个 prompt 在不同 Router 上可能路由到不同 worker**，导致 cache 无法有效复用
3. **Worker 的 KV cache 是分布式的**，Router 的 tree 必须与 worker 实际状态对齐才有意义
4. **Router 增多 → 每个 Router 的 tree "视野"变小 → cache hit 率下降**

### 1.3 这个问题与 HA 的关系

在 V1~V5 的主备方案中：
- 主备切换后，**新的主 Router 的 tree 是空的或与原主不同**
- 切换后的一段时间内，所有请求的 cache hit 率骤降
- 需要等待新的主 Router "预热" tree 才能恢复 cache 效率
- **这是 HA 方案中一个被忽视的实际成本**

---

## 二、sglang 已有的对等同步基础设施

### 2.1 Mesh Sync 机制（主版本，不完整）

`CacheAwarePolicy` 中已经存在 `mesh_sync` 字段和 `sync_tree_operation` 调用：

```rust
// cache_aware.rs:160-165
pub fn set_mesh_sync(&mut self, mesh_sync: OptionalMeshSyncManager) {
    self.mesh_sync = mesh_sync.clone();
    if mesh_sync.is_some() {
        self.restore_tree_state_from_mesh();
    }
}

// cache_aware.rs:450-462
if let Some(ref mesh_sync) = self.mesh_sync {
    let op = TreeOperation::Insert(TreeInsertOp {
        text: text.to_string(),
        tenant: workers[idx].url().to_string(),
    });
    mesh_sync.sync_tree_operation(mesh_key.to_string(), op)?;
}
```

**但代码注释明确指出**：
> "Note: `PolicyRegistry::apply_remote_tree_operation` (the only forwarder) currently has no in-process callers; the receive path is not yet wired, so this method is reachable only via tests today."

**状态**：发送路径已实现，**接收路径尚未连接**。

### 2.2 KV Event Stream 机制（实验版本，较完整）

`experimental/sgl-router/src/policies/cache_aware_zmq.rs` 实现了基于 **ZMQ 事件流**的 cache-aware 策略：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Worker (SGLang Engine)                            │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  ZmqEventPublisher (Python)                                      │  │
│  │  - 发布 BlockStored / BlockRemoved / AllBlocksCleared 事件        │  │
│  │  - 编码：msgspec.msgpack                                          │  │
│  │  - 格式：[ts, [events...], attn_dp_rank]                         │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │ ZMQ PUB socket                           │
└─────────────────────────────┼──────────────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────────────────┐
│                      Router 实例（ZMQ SUB socket）                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  KvEventSubscriberRegistry                                       │  │
│  │  - 每个 (worker_url, dp_rank) 维护一个 ZMQ SUB 连接               │  │
│  │  - 接收 msgpack payload → decode_event_batch()                   │  │
│  │  - 转发到 mpsc channel                                           │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │ mpsc channel                             │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │  pump_loop (tokio task)                                          │  │
│  │  - 从 channel 消费 WorkerEvent                                   │  │
│  │  - 过滤：seq 去重、live_workers 过滤                             │  │
│  │  - 应用：tree.insert() / tree.remove() / tree.clear_worker()     │  │
│  └──────────────────────────┬───────────────────────────────────────┘  │
│                             │                                           │
│  ┌──────────────────────────▼───────────────────────────────────────┐  │
│  │  HashTree (Arc<RwLock<TreeState>>)                               │  │
│  │  - 基于 block hash (i64) 的 radix tree                            │  │
│  │  - 节点持有 worker set (HashSet<KvWorkerId>)                     │  │
│  │  - match_prefix：读锁，O(depth) 查询                             │  │
│  │  - insert/remove：写锁                                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**关键设计**：
- **事件来源是 worker 而非 Router**：worker 主动推送 KV cache 状态变化
- **Tree 基于 block hash 而非原始文本**：更紧凑，无需 tokenization
- **使用 ZMQ PUB/SUB**：fire-and-forget，低延迟
- **pump_loop 处理 seq 去重和 live_workers 过滤**：保证一致性

### 2.3 两个机制的核心差异

| 维度 | Mesh Sync（主版本） | KV Event Stream（实验版本） |
|------|-------------------|---------------------------|
| **事件来源** | Router 自身 | Worker（SGLang Engine） |
| **同步内容** | `TreeOperation::Insert/Remove`（text + tenant） | `BlockStored/BlockRemoved`（block_hashes + worker_id） |
| **Tree 结构** | 文本 radix tree | Block hash radix tree |
| **传输方式** | `smg_mesh`（具体实现未知） | ZMQ PUB/SUB + msgpack |
| **一致性保证** | 未实现接收路径 | seq 去重 + live_workers 过滤 |
| **成熟度** | 半成品 | 较完整（有完整生命周期管理） |

---

## 三、对等无主架构方案

### 3.0 方案定位

V6 不是替代 V1~V5 的方案，而是**在 V5（WASM 被动 failover）基础上，解决 cache 树分散问题的补充方案**。

核心思路：**不需要主备切换，所有 Router 实例是对等的，通过共享事件流保持 tree 状态一致**。

### 3.1 方案 A：基于 KV Event Stream 的全局共享树

**核心设计**：所有 Worker 只向一个**中心事件总线**推送 KV 事件，所有 Router 从同一总线消费。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              外部请求                                        │
│                    api.kimi-router.example.com                               │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
            ┌──────────┐ ┌──────────┐ ┌──────────┐
            │ Router-A │ │ Router-B │ │ Router-C │
            │ (Higress)│ │ (Higress)│ │ (Higress)│
            └────┬─────┘ └────┬─────┘ └────┬─────┘
                 │            │            │
                 │   ┌────────┴────────┐   │
                 │   │                 │   │
                 ▼   ▼                 ▼   ▼
            ┌───────────────────────────────────────┐
            │         KV Event Bus（中心总线）        │
            │  ┌─────────────────────────────────┐  │
            │  │  选项 1：Kafka Topic              │  │
            │  │  - 持久化：支持 Router 重启恢复     │  │
            │  │  - 顺序保证：per-worker partition   │  │
            │  │  - 去重：Kafka consumer group      │  │
            │  └─────────────────────────────────┘  │
            │  ┌─────────────────────────────────┐  │
            │  │  选项 2：Redis Streams            │  │
            │  │  - 轻量：无需额外集群               │  │
            │  │  - 限制：单节点吞吐量               │  │
            │  └─────────────────────────────────┘  │
            │  ┌─────────────────────────────────┐  │
            │  │  选项 3：NATS JetStream           │  │
            │  │  - NVIDIA Dynamo 采用方案          │  │
            │  │  - 持久化 + 轻量                   │  │
            │  └─────────────────────────────────┘  │
            └─────────────────┬─────────────────────┘
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
              ┌─────────┐ ┌─────────┐ ┌─────────┐
              │ Worker-0│ │ Worker-1│ │ Worker-2│
              │(SGLang) │ │(SGLang) │ │(SGLang) │
              └─────────┘ └─────────┘ └─────────┘
```

**为什么 Worker 推送到中心总线而不是每个 Router 直接订阅 Worker？**

| 方案 | 连接数 | 复杂度 | 一致性 | 扩展性 |
|------|--------|--------|--------|--------|
| 每个 Router 直接订阅所有 Worker（当前 CacheAwareZmqPolicy） | N Routers × M Workers × R dp_ranks | 高 | 最终一致（取决于网络延迟） | 差（连接数爆炸） |
| Worker 推送到中心总线，Router 从总线消费 | M Workers × 1 Bus + N Routers × 1 Bus | 低 | 强一致（同一顺序消费） | 好 |

**实现要点**：
1. Worker 端：将 ZMQ PUB 改为**向 Kafka/Redis/NATS 发布**
2. Router 端：`KvEventIndex` 改为从**中心总线消费**
3. 所有 Router 消费同一事件流 → tree 状态一致
4. **无需主备**：任何 Router 挂掉，其他 Router 的 tree 不受影响

**与 HA 的结合**：
- 如果某个 Router 挂掉，K8s Service 将流量切到其他 Router
- 其他 Router 的 tree 与挂掉的 Router 完全一致 → **无 cache miss**
- 无需 V1~V5 的"主备切换后 tree 预热"问题

### 3.2 方案 B：基于操作日志的 Router 间 Tree 同步

**核心设计**：每个 Router 将本地 tree 的变更操作写入**操作日志**，其他 Router 消费日志并应用。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        操作日志（Append-Only Log）                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  log_entry_1: {type: "Insert", tree_key: "regular::gpt-4",       │  │
│  │               text: "hello", tenant: "http://w1:30000"}          │  │
│  │  log_entry_2: {type: "Insert", tree_key: "regular::gpt-4",       │  │
│  │               text: "hello world", tenant: "http://w1:30000"}    │  │
│  │  log_entry_3: {type: "Remove", tree_key: "regular::gpt-4",       │  │
│  │               tenant: "http://w1:30000"}                         │  │
│  │  ...                                                             │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  存储后端选项：                                                            │
│  - Kafka Topic（per-model partition）                                    │
│  - 共享文件系统（NFS/S3 + 定期 snapshot）                                  │
│  - Raft 共识日志（etcd / TiKV）                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

**事件类型**（复用 sglang 已有的 `TreeOperation`）：

```rust
pub enum TreeOperation {
    Insert(TreeInsertOp { text: String, tenant: String }),
    Remove(TreeRemoveOp { tenant: String }),
}
```

**同步流程**：
1. Router-A 收到请求 → `tree.insert(text, worker_url)` → 同时写入操作日志
2. Router-B/C/D 消费操作日志 → `apply_remote_tree_operation(mesh_key, op)`
3. 所有 Router 的 tree 最终一致

**与方案 A 的差异**：

| 维度 | 方案 A（KV Event Stream） | 方案 B（操作日志同步） |
|------|-------------------------|---------------------|
| **事件来源** | Worker 主动推送 | Router 本地记录 |
| **同步粒度** | Block hash 级别 | Text prefix 级别 |
| **一致性** | 强（worker 是真相来源） | 最终一致（日志传播延迟） |
| **复杂度** | 需要修改 Worker 端 | 纯 Router 端改造 |
| **兼容性** | 需 Worker 支持中心总线 | 兼容现有 Worker |

### 3.3 方案 C：中心化 Tree 存储（Redis/etcd）

**核心设计**：Router 不再本地维护 tree，而是将 tree 存储在**中心化 KV 存储**中。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Redis Cluster / etcd                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Key: "smg:tree:regular:gpt-4"                                    │  │
│  │  Value: 序列化后的 TreeState（或 Merkle tree root）               │  │
│  │                                                                  │  │
│  │  或更细粒度：                                                     │  │
│  │  Key: "smg:tree:regular:gpt-4:node:{block_hash}"                 │  │
│  │  Value: {workers: ["w1", "w2"], children: {...}, last_used: ts} │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**问题**：
1. **读延迟**：每次 `prefix_match` 都要访问 Redis → 增加 RTT
2. **写冲突**：多个 Router 同时 insert → 需要分布式锁或乐观锁
3. **序列化开销**：Tree 结构频繁序列化/反序列化
4. **Redis 内存限制**：Tree 可能很大（每个 worker 的 KV cache 状态）

**结论**：❌ **不推荐**。中心化存储引入了单点瓶颈和额外延迟，违背了 cache-aware 路由的低延迟要求。

### 3.4 方案 D：混合架构 — 本地 Tree + 异步日志同步

**核心设计**：结合方案 A 和方案 B 的优点：

1. **本地 Tree**：每个 Router 本地维护完整的 Tree，路由查询不依赖网络
2. **异步日志同步**：Router 间通过轻量日志流同步 tree 变更
3. **Worker 直接推送**：Worker 仍然通过 ZMQ 或中心总线推送 KV 事件

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Router 实例 A                                   │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────────┐  │
│  │ 本地 Tree        │    │ 操作日志 writer  │    │ Worker Event Consumer│  │
│  │ (DashMap)        │◄───┤ (本地缓冲+批量)   │◄───┤ (ZMQ/Kafka)         │  │
│  └────────┬─────────┘    └────────┬─────────┘    └──────────────────────┘  │
│           │                       │                                         │
│           │                       ▼                                         │
│           │              ┌──────────────────┐                               │
│           │              │  操作日志总线      │                               │
│           │              │ (Kafka/NATS/文件) │                               │
│           │              └────────┬─────────┘                               │
│           │                       │                                         │
│           ▼                       ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 路由查询：完全本地，无网络依赖                                        │  │
│  │ select_worker → prefix_match (本地 Tree) → 返回 worker index         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ 操作日志总线
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Router 实例 B/C/D                               │
│  ┌──────────────────────┐    ┌──────────────────┐                          │
│  │ Worker Event Consumer│    │ 操作日志 reader  │                          │
│  │ (ZMQ/Kafka)         │    │ (异步消费)       │                          │
│  └──────────────────────┘    └────────┬─────────┘                          │
│                                       │                                     │
│                                       ▼                                     │
│                              ┌──────────────────┐                          │
│                              │ 本地 Tree         │                          │
│                              │ (DashMap)        │                          │
│                              └──────────────────┘                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

**这是业界最成熟的方案**：
- NVIDIA Dynamo：NATS Core（本地索引）+ JetStream（持久化）
- llm-d：ZMQ 事件流 + in-memory LRU index
- AIBrix：Consistent hashing ring + 多 tier KV cache

---

## 四、技术可行性评估

### 4.1 方案对比

| 维度 | 方案 A（KV Event Stream 总线） | 方案 B（操作日志同步） | 方案 C（中心化存储） | 方案 D（混合） |
|------|-----------------------------|---------------------|-------------------|--------------|
| **技术成熟度** | ✅ 高（NVIDIA Dynamo/llm-d 已验证） | ✅ 中（类似 CDC） | ⚠️ 低（无先例） | ✅ **最高**（业界主流） |
| **实现复杂度** | 中（需 Worker 改造） | 低（纯 Router 端） | 中（需分布式锁） | **中** |
| **路由延迟** | ✅ 本地 Tree，无额外延迟 | ✅ 本地 Tree | ❌ 增加 RTT | ✅ **本地 Tree** |
| **一致性** | ✅ 强（同一事件流） | ⚠️ 最终一致 | ✅ 强 | ✅ **最终一致** |
| **故障恢复** | ✅ 新 Router 从日志恢复 | ✅ 从日志恢复 | ✅ 从存储恢复 | ✅ **从日志恢复** |
| **扩展性** | ✅ 好 | ✅ 好 | ❌ 中心化瓶颈 | ✅ **好** |
| **运维负担** | 中（需维护 Kafka/NATS） | 低（轻量日志） | 中（需维护 Redis） | **中** |
| **与现有架构兼容性** | 需 Worker 配合 | ✅ 纯 Router 端 | 纯 Router 端 | **需 Worker 配合** |

### 4.2 推荐方案：方案 D（混合架构）

**理由**：
1. **本地 Tree 保证路由性能**：路由查询不依赖网络，与当前 CacheAwarePolicy 性能一致
2. **Worker 直接推送保证准确性**：Tree 基于 worker 真实的 KV cache 状态，而非 Router 的推测
3. **异步日志同步保证一致性**：多个 Router 之间 tree 状态最终一致
4. **业界已验证**：NVIDIA Dynamo、llm-d 均采用类似架构

### 4.3 具体实现路径

**Phase 1：Worker → 中心总线**
- 修改 SGLang Worker 的 `ZmqEventPublisher`，增加向 Kafka/NATS 发布的选项
- 保留现有 ZMQ PUB 作为 fallback

**Phase 2：Router → 消费中心总线**
- 改造 `KvEventIndex`：支持从 Kafka/NATS 消费（而非直接 ZMQ SUB）
- 每个 Router 消费同一 topic，顺序处理事件

**Phase 3：Router 间操作日志同步（可选增强）**
- Router 本地 tree 变更写入操作日志
- 新 Router 启动时从日志恢复 tree 状态
- 实现"Router 无状态化"——tree 可以从日志重建

---

## 五、与主备架构（V1~V5）的关系

### 5.1 V6 不是替代，而是补充

| 问题 | V1~V5（主备架构） | V6（对等无主） |
|------|-----------------|--------------|
| **HA 目标** | 主挂了 → 切到备 | 所有节点对等，无切换 |
| **Cache 树分散** | 切换后 tree 需预热 | 所有节点 tree 一致 |
| **单点故障** | 主是 SPOF | 无 SPOF |
| **扩展性** | 备只能standby | 所有节点同时服务 |
| **实现复杂度** | 中-高（WASM/EnvoyFilter/Operator） | 高（分布式事件流） |
| **与现有系统兼容性** | 较好 | 需 Worker 配合改造 |

### 5.2 两种架构的融合思路

**最实际的方案可能是：V5（WASM failover）+ V6（对等同步）**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Higress Gateway（多副本）                             │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  WASM 插件（V5）                                                      │   │
│  │  - 被动 failover：worker 返回 5xx → 重试到备 worker                    │   │
│  │  - 故障检测：健康检查 + SharedData 状态共享                            │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  SMG Router（每个 Gateway Pod 内嵌）                                   │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │   │
│  │  │  CacheAwarePolicy（本地 Tree）                                    │  │   │
│  │  │  - 路由查询：完全本地                                              │  │   │
│  │  └─────────────────────────────────────────────────────────────────┘  │   │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │   │
│  │  │  KvEventIndex（消费中心总线）                                     │  │   │
│  │  │  - 从 Kafka/NATS 消费 Worker 的 KV 事件                          │  │   │
│  │  │  - 所有 Gateway Pod 消费同一 topic → tree 一致                   │  │   │
│  │  └─────────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
              ┌─────────┐ ┌─────────┐ ┌─────────┐
              │ Worker-0│ │ Worker-1│ │ Worker-2│
              │(SGLang) │ │(SGLang) │ │(SGLang) │
              │         │ │         │ │         │
              │ ┌─────┐ │ │ ┌─────┐ │ │ ┌─────┐ │
              │ │ZMQ/ │ │ │ │ZMQ/ │ │ │ │ZMQ/ │ │
              │ │Kafka│ │ │ │Kafka│ │ │ │Kafka│ │
              │ │PUB  │ │ │ │PUB  │ │ │ │PUB  │ │
              │ └─────┘ │ │ └─────┘ │ │ └─────┘ │
              └─────────┘ └─────────┘ └─────────┘
                    │           │           │
                    └───────────┼───────────┘
                                ▼
                    ┌───────────────────────┐
                    │    Kafka / NATS       │
                    │   (KV Event Bus)      │
                    └───────────────────────┘
```

**核心价值**：
1. **Worker 故障时**：WASM 插件实现被动 failover（V5）
2. **Router 故障时**：其他 Router 接管，tree 状态一致，无 cache miss（V6）
3. **Worker 扩容时**：新 Worker 的 KV 事件自动同步到所有 Router
4. **Router 扩容时**：新 Router 从事件总线恢复 tree 状态

---

## 六、结论

### 6.1 V6 方案的核心结论

> **对等无主架构（V6）在技术上完全可行，且是业界处理分布式 KV cache 路由的主流方向。**
>
> 核心前提：**将 SMG 的 cache_aware 从"每个 Router 独立维护 tree"改为"所有 Router 共享同一事件流"**。
>
> 推荐实现：**方案 D（混合架构）**——本地 Tree + Worker 直接推送 + 异步日志同步。

### 6.2 与 V5 的关系

| 场景 | 方案 |
|------|------|
| Worker 故障 → 需要 failover | V5（WASM 被动 failover） |
| Router 故障 → 需要无感切换 | V6（对等同步，tree 一致） |
| Cache 树分散 → 需要命中率优化 | V6（共享事件流） |
| 快速上线，最小改动 | 先用 V5 |
| 长期优化，追求极致 | V5 + V6 融合 |

### 6.3 落地建议

| 阶段 | 目标 | 工作量 | 依赖 |
|------|------|--------|------|
| **Phase 1** | 引入 Kafka/NATS 作为 KV Event Bus | 2-3 周 | Worker 端增加 Kafka producer |
| **Phase 2** | 改造 KvEventIndex 支持从 Bus 消费 | 1-2 周 | 纯 Router 端改造 |
| **Phase 3** | Router 无状态化：tree 可从日志重建 | 2-3 周 | 操作日志持久化 |
| **Phase 4** | V5 + V6 融合：WASM failover + 对等同步 | 1-2 周 | 前两阶段完成 |

### 6.4 关键风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| Worker 端改造工作量 | 需要修改 SGLang Engine | 使用 ZMQ → Kafka proxy 过渡 |
| 事件总线延迟 | 增加 tree 同步延迟 | 本地 Tree + 异步同步，路由不依赖网络 |
| 事件总线成为瓶颈 | 高 QPS 下 Kafka 压力大 | per-model topic partition |
| 事件丢失/乱序 | tree 状态不一致 | seq 去重 + 持久化消费（durable consumer） |

---

## 七、附录：参考来源

- sglang `cache_aware.rs` — 主版本 cache_aware 策略实现
  https://github.com/sgl-project/sglang/blob/main/sgl-model-gateway/src/policies/cache_aware.rs

- sglang `cache_aware_zmq.rs` — 实验版本基于 ZMQ 事件流的 cache_aware 策略
  https://github.com/sgl-project/sglang/blob/main/experimental/sgl-router/src/policies/cache_aware_zmq.rs

- sglang `kv_events/tree.rs` — HashTree 实现（基于 block hash 的 radix tree）
  https://github.com/sgl-project/sglang/blob/main/experimental/sgl-router/src/policies/kv_events/tree.rs

- sglang `kv_events/index.rs` — KvEventIndex 生命周期管理
  https://github.com/sgl-project/sglang/blob/main/experimental/sgl-router/src/policies/kv_events/index.rs

- sglang `kv_events/wire.rs` — KV 事件 wire format（msgpack）
  https://github.com/sgl-project/sglang/blob/main/experimental/sgl-router/src/policies/kv_events/wire.rs

- NVIDIA Dynamo Router Design — NATS Core + JetStream 多 Router 同步
  https://docs.nvidia.com/dynamo/design-docs/component-design/router-design

- llm-d KV cache aware routing — ZMQ + in-memory index
  https://developers.redhat.com/articles/2025/10/07/master-kv-cache-aware-routing-llm-d-efficient-ai-inference

- AIBrix v0.3.0 — Consistent hashing ring + 多 tier KV cache
  https://aibrix.github.io/posts/2025-05-21-v0.3.0-release/

- LLM Inference Handbook — Prefix-aware routing 综述
  https://bentoml.com/llm/inference-optimization/inference-routing

- GORGO paper — Cross-region LLM load balancing with KV-cache reuse
  https://arxiv.org/html/2602.11688v1
