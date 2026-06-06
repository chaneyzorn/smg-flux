# 分布式一致性动态路由决策

> **状态**：调研草案（draft）。本文档脱离 V0-V8 的全部历史方案，仅聚焦于"多实例 L7 网关对有状态后端做一致性动态路由决策"这一核心问题。所有结论均为理论推演，未经实验验证。

---

## 1. 问题定义

### 1.1 什么是"动态路由决策"

路由决策分为两类：

**静态路由**：路由规则在配置时确定，运行时不因后端状态变化而改变。例如：
- 轮询（round robin）：按固定顺序分配
- 一致性哈希（consistent hashing）：按固定哈希函数分配
- 静态权重：按预设比例分配

静态路由的问题：当某个后端故障时，静态路由会继续向故障后端发送请求，直到外部机制（如 EDS 更新）将故障后端从路由表中移除。**移除本身是静态的**——它依赖配置变更，而非实时的健康感知。

**动态路由**：路由决策根据运行时的后端状态实时调整。例如：
- 探测到后端 A 返回 503 → 后续请求不再发往 A
- 探测到后端 B 恢复 → 后续请求可以重新发往 B
- 调整是**实时的**（秒级或亚秒级），不依赖配置推送

动态路由的核心特征是：**路由函数 R(g, req, H) 中，H（健康状态）是运行时变量，而非配置常量。**

### 1.2 什么是"分布式一致性"

在多实例网关环境下，设 G = {g₁, g₂, ..., gₙ}，每个实例独立运行。一致性要求：

对于同一组 s 的任意请求和任意两个实例 gᵢ, gⱼ：
```
R(gᵢ, req, Hᵢ) = R(gⱼ, req, Hⱼ)  当 Hᵢ = Hⱼ 时
```

即：**所有实例在看到相同的健康视图时，必须做出相同的路由决策。**

### 1.3 为什么"动态"与"一致性"缺一不可

如果只追求动态但不追求一致性：
- 每个实例独立探测、独立决策
- 实例 A 认为后端 X 健康，实例 B 认为后端 X 不健康
- 同一 Group 的请求被不同实例路由到不同后端 → cache tree 分散

如果只追求一致性但不追求动态：
- 所有实例通过静态配置达成一致
- 后端故障时，需要等待配置更新（秒级到分钟级）
- 故障窗口期内请求继续发往故障后端

SMG 的场景同时需要两者：
- **动态**：故障切换要快。这里有两层含义：
  - **被动层**：请求失败后的即时重试（毫秒级）——保证当前请求不失败
  - **主动层**：健康探测驱动的路由决策调整（秒级）——保证后续请求路由到正确的后端
- **一致性**：所有实例必须在主动层切换到同一个新后端，否则 cache tree 分散

### 1.4 形式化定义

**分布式一致性动态路由决策**要求存在一个函数：

```
R: (Group, HealthView) → Endpoint
```

满足：
1. **确定性**：相同的 (Group, HealthView) 总是产生相同的 Endpoint
2. **动态性**：HealthView 是运行时变量，随探测结果实时更新
3. **分布式一致性**：所有实例在相同的 HealthView 下调用 R，得到相同结果
4. **收敛性**：当 HealthView 变化时，所有实例在有限时间内更新到新的 HealthView，并应用新的 R 结果

---

## 2. 为什么一致性哈希不够动态

一致性哈希常被提及作为"无需外部协调即可保证一致性"的方案。但它不满足"动态"要求。

### 2.1 一致性哈希的工作方式

- 将后端 endpoint 映射到哈希环上的虚拟节点
- 请求的特征（如 Group ID）哈希到环上的某个点
- 顺时针找到最近的 endpoint

**关键假设**：所有实例使用**相同的 endpoint 列表**构建哈希环。

### 2.2 一致性哈希如何感知故障

一致性哈希本身**不感知健康状态**。它需要外部机制将故障 endpoint 从环中移除：

```
后端 A 故障
  → 健康检查器探测到故障（可能在本实例内，也可能在控制面）
  → 健康检查器更新 endpoint 列表（移除 A）
  → EDS/EndpointSlice 推送更新到所有 Gateway 实例
  → 所有 Gateway 重建哈希环（不含 A）
  → 原来映射到 A 的请求重新分配到 B 或 C
```

**问题**：
1. **故障感知不是实时的**：EDS 更新通常有秒级到分钟级延迟
2. **故障恢复也不实时**：后端恢复后需要重新加入环，同样依赖 EDS 更新
3. **哈希环重建导致大量重分配**：移除 A 不仅影响映射到 A 的请求，还影响 A 附近虚拟节点对应的请求范围
4. **无法处理"亚健康"状态**：endpoint 偶尔超时（50% 成功率），EDS 不会移除它，一致性哈希继续向它发送请求

### 2.3 一致性哈希 vs 动态路由

| 维度 | 一致性哈希 | 动态路由（如 V5.2） |
|------|-----------|-------------------|
| 健康感知 | 依赖外部 EDS 更新 | 实例自主探测，实时感知 |
| 主动切换速度 | 秒级~分钟级（EDS 延迟） | 1-3s（探测周期 + 传播） |
| 被动切换速度 | 不支持 | 毫秒级（请求失败后的即时重试） |
| 亚健康处理 | 无法处理 | 可以基于成功率阈值决策 |
| 状态同步需求 | 需要同步 endpoint 列表 | 需要同步健康状态 |
| 一致性保证 | 列表相同时一致 | 健康视图相同时一致 |

一致性哈希的"一致性"是**静态配置层面的一致性**，不是**动态状态层面的一致性**。它无法响应运行时的健康状态变化，因此不满足 SMG HA 场景的需求。

---

## 3. 方向一：基于实时健康视图的确定性路由

### 3.1 核心思想

每个网关实例：
1. **自主探测**：独立向后端发送健康探测（如 HTTP GET /readiness）
2. **聚合视图**：收集所有实例的探测结果，形成全局健康视图
3. **确定性选择**：运行确定性函数 R(Group, HealthView) → Endpoint
4. **实时调整**：当 HealthView 变化时，R 的输出自动变化

### 3.2 健康视图的同步方式

**方式 A：Pub/Sub（V5.2 方向）**
- 每个实例探测后发布结果到共享后端（Consul KV / Redis Stream / Nacos）
- 其他实例订阅更新，维护本地 HealthView
- 请求路径读取本地 HealthView，调用 R

**后端延迟特性对比**：
- **Consul KV**（默认）：HTTP blocking query，数据变化时立即推送（~0ms），无变化时最长等待 30s。传播延迟最低。
- **Redis Streams**：RESP `XREAD` 短轮询（每 3s），传播延迟 0-3s。通过 Higress 自定义 Redis host 直接访问，不需要 HTTP 代理。
- **Nacos**：HTTP long polling，传播延迟接近 Consul，但需已有 Nacos 环境。

**方式 B：查询-更新**
- 每个实例将探测结果写入共享存储
- 请求路径查询共享存储获取最新 HealthView
- 增加 1-5ms 延迟，但实现更简单

**方式 C：P2P 传播**
- 实例间直接交换探测结果（HTTP gossip）
- 无需外部存储，但实现复杂

### 3.3 确定性选择函数 R

```
R(Group, HealthView):
  candidates = Group.Endpoints sorted by priority
  for ep in candidates:
    if HealthView[ep] == HEALTHY:
      return ep
  return candidates[0]  // fail-open
```

**关键**：所有实例使用相同的 candidates 排序和相同的 HealthView 判定逻辑，因此 R 的输出必然相同。

### 3.4 动态性体现

- **T0**：HealthView = {A: HEALTHY, B: HEALTHY, C: HEALTHY} → R 返回 A
- **T1**：实例 x 探测 A 返回 503，发布更新 → HealthView 变为 {A: UNHEALTHY, ...}
- **T2**：所有实例收到更新，R 返回 B
- **T3**：A 恢复，探测返回 200，发布更新 → HealthView 恢复
- **T4**：R 重新返回 A（如果策略允许自动回切）

整个过程在 1-2 个探测周期内完成（Consul 后端因 blocking query 变化立即推送，接近 1 个周期；Redis 后端因短轮询延迟，约 1-2 个周期），无需配置更新。

### 3.5 优势与劣势

**优势**：
- 真正的动态：实时响应健康状态变化
- 真正的一致：确定性函数保证相同输入相同输出
- 请求路径零延迟（本地缓存）或低延迟（查询模式）

**劣势**：
- 需要外部存储或 P2P 机制同步 HealthView
- 实例间 HealthView 有短暂不一致窗口（最终一致性）

### 3.6 Fail-fast 策略的多维度副作用

方向一（以及 V5.2 实现）采用 **fail-fast** 聚合策略：任一 Pod 报告 unhealthy → 全局标记 unhealthy；恢复则需**全部活跃 Pod 的最近报告都 healthy**。这个策略在快速故障发现上有效，但在多维度上存在值得关注的副作用。

**维度一：单 Pod 网络抖动导致全局误判**

假设某个 Gateway Pod 与某个 SMG endpoint 之间存在临时网络抖动（如跨可用区链路闪断、K8s CNI 瞬时丢包）。该 Pod 的探测请求超时或返回 503，于是发布 `unhealthy` 报告。其他 Pod 收到后，fail-fast 策略将该 endpoint 全局标记为 unhealthy，即使：
- 该 endpoint 本身完全正常
- 其他所有 Pod 探测都返回 200
- 客户端直接访问该 endpoint 也能成功

**误判的持续时间** = 该 Pod 的下一次成功探测时间 + Pub/Sub 传播延迟。
- Consul 后端：blocking query 变化立即推送，传播延迟 ≈ 0，误判持续约 **3s**（一个探测周期）。
- Redis 后端：短轮询每 3s，传播延迟 0-3s，误判可能持续 **3-6s**。

期间所有流量被不必要地切换到备用 endpoint，造成 cache tree 分散。

**维度二：Pod 数量与恢复延迟的正相关**

Fail-fast 的恢复条件是"全部活跃 Pod 的最近报告都 healthy"。设：
- 探测周期为 T
- Gateway Pod 数量为 N
- 每个 Pod 独立探测，成功概率为 p（p ≈ 1，但非 100%）

则一次完整健康评估需要**所有 N 个 Pod 在最近的 T 秒内都发布 healthy 报告**。任意一个 Pod 因网络抖动、GC 暂停、探测超时等原因发布 unhealthy，恢复就被推迟至少一个探测周期。

恢复延迟的期望：
```
E[recovery] ≈ T / p^N   （简化模型）
```

当 p = 0.99（99% 探测成功率），T = 3s 时：
- N = 2：E ≈ 3 / 0.99² ≈ 3.06s
- N = 3：E ≈ 3 / 0.99³ ≈ 3.09s
- N = 5：E ≈ 3 / 0.99⁵ ≈ 3.15s
- N = 10：E ≈ 3 / 0.99¹⁰ ≈ 3.31s

看起来增幅不大，但当 p 下降到 0.95（95% 成功率，如跨 AZ 部署、网络负载较高时）：
- N = 2：E ≈ 3 / 0.95² ≈ 3.32s
- N = 5：E ≈ 3 / 0.95⁵ ≈ 3.87s
- N = 10：E ≈ 3 / 0.95¹⁰ ≈ 4.89s

更关键的是**尾延迟**：所有 N 个 Pod 连续 K 次探测都 healthy 的概率是 p^(N×K)。要求连续 2 次 healthy 才能恢复时：
- N = 5, p = 0.95, K = 2：E ≈ 6 / 0.95¹⁰ ≈ 9.78s

这意味着：**Gateway 副本数越多、网络环境越复杂，fail-fast 策略的恢复延迟越长**。

**维度三：探测频率与一致性的 trade-off**

缩短探测周期 T 可以减少恢复延迟，但代价是：
- 后端探测负载增加（N 个 Pod × M 个 endpoint × 1/T 频率）
- Pub/Sub 后端写入 QPS 增加（Consul 的 Raft 写入成本略高于 Redis，但 health check 频率很低，不构成瓶颈）
- Consul 读取端不受探测频率影响（blocking query 是事件驱动，不是轮询）
- 误判的频率也增加（因为探测次数多了，碰到瞬态故障的概率也高了）

 lengthening T 则相反：减少负载和误判频率，但延长了故障发现和恢复时间。

**缓解方向**：

1. **Quorum-based 恢复**：不要求"全部 Pod healthy"，而是要求"超过半数 Pod healthy"。这在 N > 2 时显著降低恢复延迟，代价是容忍少数 Pod 的观测差异。
2. **自适应探测频率**：对疑似 healthy 的 endpoint 降低探测频率（如 5s），对疑似 unhealthy 的 endpoint 提高频率（如 1s），平衡负载与响应速度。
3. **探测路径隔离**：确保探测请求走与业务请求不同的网络路径（如同节点直接访问，不走 K8s Service 负载均衡），减少网络抖动导致的误判。

---

## 4. 方向二：基于控制面动态推送的状态同步

### 4.1 核心思想

不依赖网关实例自主探测和同步，而是由控制面（Higress Controller 或自定义 Operator）统一探测后端健康状态，计算路由决策，通过 xDS 动态推送给所有 Gateway 实例。

### 4.2 实现路径

**路径 A：动态 EDS（Endpoint Discovery Service）**
- 控制面探测所有后端 `/readiness`
- 只将 HEALTHY 的 endpoint 放入 EDS
- 通过 xDS 推送给 Gateway
- Gateway 的 Envoy 使用任意负载均衡策略（如 round robin）在 healthy endpoints 中选择

**问题**：如果要求"所有请求集中到同一个 endpoint"，EDS 中只能放一个 endpoint。这意味着控制面需要决定"哪个 endpoint 是主"——这本身就是动态路由决策。

**路径 B：动态 Route + 自定义 Header**
- 控制面计算每个 Group 当前应路由的 endpoint
- 通过 xDS 更新 Route 配置，注入 `x-higress-target-endpoint` header
- Gateway 的 WASM 插件读取 header，使用 `SetUpstreamOverrideHost`

**问题**：xDS 推送有秒级延迟，不满足主动层的秒级故障切换要求。更重要的是，控制面推送无法提供请求级别的被动重试——请求失败后必须由数据面自行处理。

**路径 C：自定义 xDS 资源**
- 扩展 Envoy 的 xDS 协议，增加 `HealthView` 资源类型
- 控制面推送 HealthView 到 Gateway
- WASM 插件订阅 HealthView 更新，本地调用 R

这与方向一的 Pub/Sub 类似，只是协议从 Redis/Consul 换成了 xDS。

### 4.3 与方向一的对比

| 维度 | 控制面推送 | 实例自主同步 |
|------|-----------|-------------|
| 探测来源 | 控制面统一探测 | 每个实例独立探测 |
| 状态同步 | xDS 推送 | Pub/Sub 或 P2P |
| 主动切换速度 | 秒级（xDS 延迟） | 1-3s（探测 + 传播） |
| 被动切换速度 | 不支持 | 毫秒级（请求失败后的即时重试） |
| 复杂度 | 高（需改控制面） | 中（插件内实现） |
| 与 Higress 耦合 | 高 | 低 |

**结论**：控制面推送在理论上是可行的，但需要修改 Higress Controller 或自建控制面，与 Higress 深度耦合。对于独立插件方案来说，实例自主同步更务实。

---

## 5. 方向三：基于请求反馈的自适应动态路由

### 5.1 核心思想

不依赖主动健康探测，而是让**请求的实际结果**驱动路由决策的动态调整。每个实例根据请求的成功/失败历史，独立但确定性地调整 endpoint 选择。

### 5.2 实现机制

**失败计数器**：
- 每个实例维护每个 endpoint 的连续失败计数
- 向 endpoint A 发送请求，返回 503 → A 的失败计数 +1
- 向 endpoint A 发送请求，返回 200 → A 的失败计数清零
- 当失败计数超过阈值 N，实例将 A 标记为 UNHEALTHY
- 所有实例使用相同的阈值 N 和相同的 endpoint 排序，因此会在大致相同的时间标记 A 为 UNHEALTHY

**恢复探测**：
- A 被标记为 UNHEALTHY 后，实例以较低频率向 A 发送探测请求
- 连续 M 次成功后，将 A 恢复为 HEALTHY

### 5.3 为什么这不是真正的动态一致性路由

**问题 1：观测差异**
- 实例 A 的请求恰好都命中了 A 的故障窗口，计数器快速增长
- 实例 B 的请求恰好都命中了 A 的正常窗口，计数器增长缓慢
- 两个实例会在不同时间标记 A 为 UNHEALTHY → 不一致

**问题 2：没有状态同步**
- 实例 A 已经切换到 B，但实例 B 还在向 A 发送请求
- 没有机制通知 B "A 已经被标记为 UNHEALTHY"

**问题 3：统计波动**
- 失败计数器受请求模式影响（某些请求更容易触发故障）
- 不同实例的请求序列不同，导致计数器行为不同

### 5.4 改进：带广播的失败反馈

- 实例 A 标记 A 为 UNHEALTHY 时，广播"A 已故障"消息
- 其他实例收到广播后，即使自己还没达到阈值，也跟随标记
- 广播需要 P2P 或外部服务 → 回到了方向一

**结论**：纯请求反馈驱动的方法无法实现严格的分布式一致性。自适应调整可以作为一种**补充机制**（如 V5.1 中的即时 health check），但不能替代显式的状态同步。

---

## 6. 方向四：基于局部观测的共识收敛

### 6.1 核心思想

每个实例只依赖**本地观测**（自己的探测结果 + 收到的广播消息），通过共识算法达成一致。不依赖持久化的外部存储。

### 6.2 实现机制

**Quorum-based 判定**：
- 每个实例独立探测，将结果通过 gossip 广播给所有其他实例
- 判定 endpoint A 的健康状态时，收集所有实例的最近报告
- 如果超过半数实例报告 A 不健康 → A 被标记为 UNHEALTHY
- 如果超过半数实例报告 A 健康 → A 被标记为 HEALTHY
- 否则保持上一状态

**优势**：
- 去中心化，无单点故障
- 网络分区时，分区内的实例可以基于本地 quorum 继续决策

**劣势**：
- gossip 协议的收敛时间不确定
- 需要处理消息乱序、重复、丢失
- 实例数量变化时 quorum 阈值需要调整
- WASM 环境不原生支持 UDP gossip

### 6.3 与方向一的对比

方向一（Pub/Sub + 外部存储）和方向四（P2P gossip）都实现了"多个实例共享健康视图"，但渠道不同：

| 维度 | Pub/Sub + 外部存储 | P2P Gossip |
|------|-------------------|-----------|
| 中心节点 | 有（外部存储） | 无 |
| 消息传播 | 1 跳（发布到存储，其他订阅） | 多跳（gossip 传播） |
| 收敛速度 | 快（1-3s） | 慢（秒级~分钟级，取决于网络拓扑） |
| 实现复杂度 | 中 | 高 |
| 可靠性 | 依赖外部存储可用性 | 依赖网络连通性 |

**结论**：对于 2-3 个 Gateway 副本的小型部署，gossip 的收益不足以抵消复杂度。外部存储方案更可靠、更简单。

---

## 7. 不够"动态"的方案：为什么被排除

以下方案常被讨论，但因其不满足"动态"要求而被排除。

### 7.1 一致性哈希（Consistent Hashing）

**为什么不满足动态**：
- 一致性哈希本身不感知健康状态
- 故障感知依赖外部机制（EDS/EndpointSlice 更新）移除故障节点
- EDS 更新是配置层面的变更，不是运行时的动态响应
- 亚健康 endpoint（偶发超时）不会被移除

**适用场景**：无状态后端、可接受配置级故障切换延迟（分钟级）。

### 7.2 静态权重 + 被动健康检查

**为什么不满足动态**：
- 权重是配置常量，运行时不变
- Envoy 的被动健康检查（outlier detection）基于统计窗口，需要积累足够多的失败才触发驱逐
- 驱逐后权重重新计算仍然依赖配置更新

### 7.3 轮询 / 随机

**为什么不满足一致性**：
- 轮询和随机 inherently 不一致——不同实例的请求序列不同，选择结果自然不同
- 即使后端列表相同，也无法保证同一 Group 的请求集中到同一个 endpoint

---

## 8. 各方向综合对比

| 方向 | 动态性 | 一致性 | 主动切换 | 被动切换 | 额外组件 | 实现复杂度 | 推荐度 |
|------|--------|--------|----------|--------|----------|-----------|--------|
| **实时健康视图 + 确定性路由**（V5.2） | ✅ 实时探测 | ✅ 确定性函数 | 1-3s | 毫秒级 | Redis/Consul | 中 | **首选** |
| **控制面动态推送** | ⚠️ 秒级延迟 | ✅ xDS 保证 | 秒级 | 不支持 | 改 Controller | 高 | 备选 |
| **请求反馈自适应** | ✅ 实时响应 | ❌ 观测差异 | 分钟级 | 分钟级 | 无 | 低 | 不推荐 |
| **P2P Gossip 共识** | ✅ 实时传播 | ✅ Quorum | 秒级~分钟级 | 不支持 | 无 | 极高 | 不推荐 |
| 一致性哈希 | ❌ 配置驱动 | ✅ 列表相同时 | 分钟级 | 不支持 | 无 | 低 | 已排除 |
| 静态权重 | ❌ 配置常量 | ✅ 权重相同时 | 分钟级 | 不支持 | 无 | 低 | 已排除 |

---

## 9. 社区与业界为什么没有直接对标方案

### 9.1 这是一个"夹缝问题"

V5.2 面对的问题之所以在开源社区没有现成方案，是因为它要求**三个条件同时成立**：

1. **网关是多实例无状态的**（不能靠单实例避免共识）
2. **后端是有状态的**（必须稳定路由到同一 endpoint）
3. **故障切换要求亚秒级**（控制面推送太慢）

而业界主流架构通常破坏至少一个前提，从而绕开问题：

| 业界主流方案 | 破坏的前提 | 说明 |
|-------------|-----------|------|
| **后端无状态化** | #2 | 云原生最佳实践。如果 worker 的 cache tree 能共享存储（Redis/分布式 KV），路由就不需要稳定，任何实例都能处理。这是根本解法，但 sglang 目前不支持 |
| **sticky session / 一致性哈希** | #2 或 #3 | Envoy/Kong 支持 `ring_hash`、`maglev`，但输入是**静态 endpoint 列表**，不感知运行时健康状态；endpoint 故障时必须等 EDS 更新（秒到分钟级） |
| **控制面推送（Istio/Consul Connect）** | #3 | 服务网格标准做法：控制面探测 → xDS 推送到 Envoy。但刷新周期通常 10-30s，远慢于 failover 所需的亚秒级 |
| **单网关 Active-Passive** | #1 | 用 Corosync/Pacemaker 做网关层主备，备机实时同步连接状态。但这牺牲了网关的水平扩展能力 |
| **Higress ai-proxy failover** | #1（部分） | 用 `SharedData` + lease 做**单进程内**的主备切换，每个 Pod 独立决策。多 Pod 之间没有共识——这正是 V5.1 的缺陷 |

### 9.2 为什么大厂也不开源这类方案

大厂遇到类似问题时通常选择：
- **自建专用控制平面**：如 Google 的 GFE（Google Front End）有内部实现的优先级路由和实例级健康同步，但属于基础设施核心组件，不开源。
- **在业务层解决**：让 worker 支持多活或共享状态，从根本消除"必须稳定路由"的约束。
- **接受更高延迟**：非核心业务容忍秒级控制面推送，直接使用 Istio/Envoy 原生能力。

### 9.3 学术界的相关工作

最接近的学术工作是 *"Supporting Disconnected Operation of Stateful Services"*（Farnham et al., 2023），它用 Envoy aggregate cluster + shadow mirroring + 外部缓存代理做边缘/云场景的状态同步和故障恢复。但它解决的是**边云之间的状态复制**，不是**多网关实例间的路由共识**，且引入了外部 Postgres 缓存和请求去重代理，复杂度很高。

### 9.4 结论

V5.2 的 **Pub/Sub + 确定性共识 + 有界分叉 fail-fast** 这套组合，在开源网关生态里没有可直接对标的方案。如果它能落地验证，本身可能就是一个值得分享的设计模式。

---

## 10. 关键结论

### 10.1 "动态"与"一致性"可以共存，但需要状态同步

实现分布式一致性动态路由的唯一可行路径是：**所有实例共享相同的运行时健康视图，并通过确定性函数做出相同的路由决策。**

健康视图的同步可以通过：
- 外部存储 + Pub/Sub（推荐）
- 外部存储 + 查询（简单但增加延迟）
- P2P Gossip（复杂但去中心化）

### 10.2 状态同步的内容很重要

同步什么？
- **只同步健康状态**（V5.2 方向）：每个实例独立运行 R，最灵活
- **同步路由决策**（控制面推送）：控制面决定 endpoint，实例被动执行，与 Higress 深度耦合
- **同步探测报告**（Pub/Sub 原始数据）：最细粒度，每个实例独立聚合，容错性最好

V5.2 选择同步探测报告（原始数据），每个实例本地聚合，兼顾灵活性和容错性。

### 10.3 最终一致性在工程上足够

严格的一致性（如 Raft）在理论上有优势，但：
- 延迟过高（百毫秒级）
- WASM 环境不可行
- 对于 SMG 的 HA 场景，1-3s 的收敛时间可接受

最终一致性（Pub/Sub + 本地缓存）是工程上的甜点。

---

## 11. 与 V5.2 的关系

本文档从零重新梳理的所有方向，最终收敛到的结论与 V5.2 的设计一致：

- **外部协调层 + 探测报告同步 + 本地确定性路由** 是实现"分布式一致性动态路由决策"的工程最优解
- **Consul KV**（或 Redis Streams / Nacos）作为默认同步后端
- **请求路径零外部调用**，通过本地缓存实现
- **Fail-fast 聚合**处理探测分歧
- **优雅降级**到独立模式

V5.2 的详细设计文档（`active_standby_ha_design_v5.2_research.draft.md`）将上述原则落地为可执行的接口、算法和部署规格。其中引入了**共享的 `locked` endpoint 状态**（通过后端 CAS 写入，所有 Pod 订阅同步），实现了粘滞路由：当前 locked endpoint 健康时保持不动，不因高优先级节点恢复而回切；failover 采用顺序降级策略（从当前位置往后找下一个 HEALTHY），避免 cache tree 反复迁移。

---

## 附录：参考来源

以下源码和文档在本文档调研过程中被直接阅读或引用。所有技术判断均基于这些一手资料推导，未经独立实验验证。

### 一、Higress 源码

1. **ai-proxy failover：SharedData 与 VM lease 实现**
   - 来源：`higress/plugins/wasm-go/extensions/ai-proxy/provider/failover.go`
   - 关键代码：
     - `ctxVmLease = provider + "-" + id + "-vmLease"`（第 131 行）
     - `tryAcquireOrRenewLease(vmID)` 的 CAS 竞争逻辑（第 292-341 行）
     - `generateVMID()` 返回 `uuid.New().String()`（第 343-345 行）
     - `resetSharedData()` 在配置更新时清除所有状态（第 685-691 行）
   - 用途：确认 ai-proxy 的 lease 是 per-VM 的；多 Pod 场景下每个 Pod 独立竞争；SharedData 在 config update 时被显式重置。这些代码直接证明了 SharedData 不跨 Pod 共享的核心判断。

2. **ai-load-balancer：SetUpstreamOverrideHost**
   - 来源：`higress/plugins/wasm-go/extensions/ai-load-balancer/`
   - 关键发现：`SetUpstreamOverrideHost` 是 Higress 自定义的 host call，允许 WASM 插件直接覆盖 Envoy 选中的 upstream endpoint
   - 用途：确认 Higress WASM 插件具备 preemptive 切换 upstream 的能力，这是 V5.2 方案中"确定性路由函数 R"能在请求路径上生效的技术基础。

3. **Higress Gateway 默认副本数**
   - 来源：`higress/helm/higress/README.md` 第 130 行
   - 内容：`gateway.replicas | int | 2 | Number of Higress Gateway pods`
   - 来源：`higress/helm/core/templates/deployment.yaml` 第 28 行
   - 内容：`replicas: {{ .Values.gateway.replicas }}`
   - 用途：确认生产环境默认多实例部署。这是"分布式一致性"问题存在的前提——如果 Gateway 只有单实例，就不存在跨实例不一致问题。

### 二、proxy-wasm 规范

4. **proxy-wasm spec：SharedData scope 定义**
   - 来源：proxy-wasm/spec 文档（通过 Higress 实现行为推断）
   - 内容：SharedData 的 scope 被定义为 implementation-defined；Higress/Envoy 的实现中为单个 Envoy 进程级别
   - 用途：确认 V5.1 的 SharedData 不跨 Pod 共享，不跨生命周期持久化。这是 V5.2 需要引入外部状态同步机制的根本原因。

### 三、Envoy 文档

5. **Envoy Active Health Checking**
   - 来源：https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking
   - 内容：Envoy 支持 HTTP、TCP、gRPC 主动健康检查。检查间隔、超时、不健康阈值、健康阈值均可配置。健康状态变化后，Envoy 会更新本地 endpoint 的 health_status，但**不主动通知其他 Envoy 实例**。
   - 用途：确认 Envoy 原生的主动健康检查是 per-instance 的。每个 Envoy 进程独立探测，独立维护健康状态。这与 V5.1 的 SharedData 问题本质相同——健康状态不跨实例共享。

6. **Envoy Outlier Detection（被动健康检查）**
   - 来源：https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier
   - 内容：基于连续 5xx、超时、成功率等统计指标驱逐异常 endpoint。需要积累足够多的样本才触发，有显式的时间窗口（如 `consecutive_5xx` 需要连续 N 次 5xx，`success_rate` 需要统计周期内的成功率）。驱逐是**本地行为**，不影响其他 Envoy 实例。
   - 用途：确认 Envoy 原生的被动健康检查也不够实时（需要积累样本），且同样是 per-instance 的。这排除了"依赖 Envoy 原生健康检查实现动态路由"的可能性。

7. **Envoy Load Balancing Overview**
   - 来源：https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancing
   - 内容：Envoy 支持多种负载均衡策略：round robin、least request、ring hash、maglev、random。所有策略都基于**本地的 endpoint 列表和健康状态**做决策，没有跨实例协调机制。
   - 用途：确认 Envoy 原生的所有负载均衡策略都是 per-instance 的，无法满足"分布式一致性"要求。

8. **Envoy Consistent Hashing**
   - 来源：https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#ring-hash
   - 内容：Envoy 支持 `ring_hash` 和 `maglev` 两种一致性哈希负载均衡策略。哈希环在后端列表变化时重建。一致性哈希的输入是**endpoint 列表**，不是**健康状态**。
   - 用途：确认一致性哈希不感知运行时健康状态，故障响应依赖 EDS 更新（配置层面），不满足"动态"要求。这是本文档排除一致性哈希方案的核心依据。

9. **Envoy xDS Protocol**
   - 来源：https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol
   - 内容：xDS（Discovery Service）是 Envoy 的动态配置协议。配置通过 ADS 推送给 Envoy 实例，推送延迟取决于控制面实现和网络状况，通常为秒级。
   - 用途：评估控制面集中决策（方向四）的可行性。xDS 的秒级延迟决定了它不适合亚秒级故障切换场景。

### 四、Redis 文档

10. **Redis Streams**
    - 来源：https://redis.io/docs/data-types/streams/
    - 内容：`XADD` 追加消息、`XREAD` 阻塞读取、`XREVRANGE` 范围查询、`MAXLEN` 自动 trim。支持多个 consumer group 独立消费。
    - 用途：设计 V5.2 备选后端（Redis Streams）的数据模型和订阅机制。Redis Streams 的持久化和 trim 特性适合作为探测报告的"短期日志存储"。

11. **Redis Keyspace Notifications**
    - 来源：https://redis.io/docs/manual/keyspace-notifications/
    - 内容：Redis 可以发布 key 级别的事件通知（如 `set`、`expire`、`evicted`）。通过 `CONFIG SET notify-keyspace-events` 启用。
    - 用途：评估 Redis 作为 Pub/Sub 后端的另一种实现路径（通过监听 key 变更事件替代 Stream）。

### 五、SMG 源码

12. **sgl-model-gateway cache tree 实现**
    - 来源：`sglang/sgl-model-gateway/src/cache_aware.rs`、`tree.rs`
    - 关键发现（V7 调研）：
      - tree 缺失时 `rand::rng()` 显式随机选择 worker
      - load 相同时依赖 `DashMap` 不确定的迭代顺序
      - tenant 选择时 `DashMap.iter().next()` 顺序不确定
      - 时间戳更新是概率性的（1/8 chance）
      - node split 时的 `contains_key` + `insert` 是非原子操作
    - 用途：确认 SMG worker 是**有状态**的，且 cache tree 的构建存在多个随机/不确定环节。这证明了"路由一致性"是必要条件——即使流量 100% 相同，不同 worker 上的 tree 也会分歧。

### 六、学术论文

13. **Maglev: A Fast and Reliable Software Network Load Balancer**
    - 作者：Daniel E. Eisenbud 等（Google）
    - 发表：NSDI 2016
    - 来源：https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/
    - 内容：Google 自研的软件负载均衡器，使用一致性哈希（maglev hashing）将流量分配到后端。论文详细讨论了后端故障时的重分配策略、连接跟踪、以及如何避免哈希环重建导致的大规模连接迁移。
    - 用途：一致性哈希在工业界的最高水准实现。论文明确指出了动态后端变化（后端加入/移除）时的重分配代价——即使是 maglev 的优化哈希表，也无法避免故障切换时的请求重分配。这印证了本文档"一致性哈希不够动态"的判断。

14. **Dynamo: Amazon's Highly Available Key-value Store**
    - 作者：Giuseppe DeCandia 等（Amazon）
    - 发表：SOSP 2007
    - 来源：https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
    - 内容：Amazon Dynamo 使用一致性哈希进行数据分片，使用 gossip-based 反熵协议传播成员状态变化。采用"最终一致性"模型，通过 vector clock 解决冲突。
    - 用途：分布式系统中"最终一致性 + gossip 传播"的经典案例。Dynamo 的设计选择（接受短暂不一致以换取高可用）与 V5.2 的方案（Pub/Sub + 最终一致性 + 本地缓存）在哲学上一致。但 Dynamo 面向的是存储系统，其 gossip 收敛时间（分钟级）不适合网关路由的秒级切换需求。

15. **SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol**
    - 作者：Abhinandan Das, Indranil Gupta, Ashish Motivala
    - 发表：2002
    - 来源：https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf
    - 内容：提出了一种可扩展的、弱一致性的组成员协议，使用 infection-style（传染式）消息传播探测失败节点的状态。将故障检测与成员更新解耦，通过 gossip 实现 O(log N) 的传播效率。
    - 用途：P2P gossip 方向的理论基础。SWIM 证明了去中心化的故障检测和状态传播在理论上可行，但其实现复杂度（消息乱序处理、网络分区处理、成员变更同步）在 WASM 网关场景下得不偿失。

16. **CAP Twelve Years Later: How the "Rules" Have Changed**
    - 作者：Eric A. Brewer
    - 发表：IEEE Computer, 2012
    - 来源：https://sites.cs.ucsb.edu/~rich/class/cs293b-cloud/papers/brewer-cap.pdf
    - 内容：CAP 定理的提出者重新审视了这一定理。指出 CAP 中的"一致性"和"可用性"不是二元的，而是连续光谱。实际系统中通常采用"延迟-bound 的一致性"（如最终一致性、因果一致性），而非强一致性。
    - 用途：为本文档接受"最终一致性"提供理论支撑。V5.2 的 Pub/Sub + 本地缓存方案正是"延迟-bound 的一致性"——所有实例在 1-3s 内收敛到相同状态，这个延迟对于 SMG 的 HA 场景是可接受的。

17. **Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web**
    - 作者：David R. Karger, Eric Lehman, Tom Leighton, Rina Panigrahy, Matthew S. Levine, Daniel Lewin
    - 发表：STOC 1997
    - 来源：https://dl.acm.org/doi/10.1145/258533.258660
    - 内容：提出一致性哈希算法，解决分布式缓存中节点增减时的数据迁移问题。核心性质：当节点数量从 N 变化到 N+1 时，只有 O(1/N) 的数据需要重新映射。
    - 用途：一致性哈希的理论基础。本文档排除一致性哈希的依据不是算法本身有缺陷，而是它的**设计目标**与 SMG HA 的需求不匹配——一致性哈希优化的是"节点变化时的最小重分配"，而不是"运行时健康状态的实时响应"。

### 七、Consul 文档

18. **Consul KV HTTP API**
    - 来源：https://developer.hashicorp.com/consul/api-docs/kv
    - 内容：`PUT /v1/kv/{key}` 写入 KV，`GET /v1/kv/{key}?recurse=true` 前缀查询。KV 变更时 ModifyIndex 单调递增。
    - 用途：设计 V5.2 默认后端（Consul KV）的数据模型和发布机制。

19. **Consul Blocking Queries**
    - 来源：https://developer.hashicorp.com/consul/api-docs/features/blocking
    - 内容：通过 `?index={ModifyIndex}` + `?wait={max_duration}` 实现长轮询。数据变化时立即返回（HTTP 200 + 新 index），无变化时超时返回。
    - 用途：确认 Consul blocking query 与 proxy-wasm 异步 HTTP 模型的适配性——响应必定在 0-30s 内完成，触发 `OnHttpCallResponse`，可在回调中递归发起下一个 blocking query。

### 八、外部项目

20. **Higress 自定义 Redis host 函数**
    - 来源：`higress/plugins/wasm-go/extensions/response-cache/cache/redis.go`；`github.com/higress-group/wasm-go/pkg/wrapper/redis_wrapper.go`
    - 内容：Higress 在 proxy-wasm-go-sdk 中增加了 `proxy_redis_init` / `proxy_redis_call` 自定义 host 函数，wasm-go 框架提供 `wrapper.RedisClient` 接口。WASM 插件可以直接通过 RESP 协议访问 Redis，**不需要 HTTP 代理层**。
    - 用途：确认 Redis 作为 V5.2 备选后端时不需要 Go bridge，直接通过原生 RESP 协议通信。

21. **Webdis — Redis HTTP 代理（已废弃）**
    - 来源：https://github.com/nicolasff/webdis
    - 内容：将 Redis 命令暴露为 HTTP REST API 的轻量级代理
    - 用途：在发现 Higress 自定义 Redis host 之前的备选方案。现在已被原生 RESP 访问替代。
