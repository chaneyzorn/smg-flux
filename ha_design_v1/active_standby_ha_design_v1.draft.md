# SMG Active-Standby 高可用架构设计调研报告

> 目标：解决 Higress/K8s 轮询导致 cache-aware 前缀树分散的问题，实现 SMG 主备访问模式，同时保留 Higress 的熔断、健康检查、失败转发能力，并确保 P/D worker 不因单个 SMG 故障而中断服务。

---

## 一、问题根源分析

### 1.1 cache-aware 树分散的本质

```
        Higress (Round Robin)
           │
    ┌──────┼──────┬──────────┐
    ▼      ▼      ▼          ▼
  SMG-1  SMG-2  SMG-3      ...
  (树A)  (树B)  (树C)
    │      │      │
   P/D    P/D    P/D
```

**当前行为**：
- 每个 SMG 实例**独立维护**自己的 `CacheAwarePolicy` 和 `DashMap<String, Arc<Tree>>`
- 请求被轮询到 SMG-1 时，SMG-1 在自己的树中插入 `(prompt, worker_url)`
- 下一个相似请求被轮询到 SMG-2 时，SMG-2 在自己的树中插入，**看不到 SMG-1 的历史**
- 结果：相同前缀的请求散落在不同 SMG 上，每个 worker 的 Radix Cache 命中率下降

**理想行为**：
- 每个 SMG "组"（如 SMG-1-主/备）只有一个活跃实例处理请求
- 所有请求历史集中在该活跃实例的 cache-aware 树中
- 故障切换时，备用实例的树状态与主实例保持同步

### 1.2 约束条件拆解

| 约束 | 含义 |
|------|------|
| **Higress 轮询/随机不具备主备** | 标准 K8s Service 和 Higress Ingress 只做平等负载均衡，没有"优先主、主挂了才走备"的语义 |
| **保留熔断/健康检查/失败转发** | Higress 的 `outlierDetection`、passive health check、retry 能力不能丢失 |
| **P/D 不中断** | SMG 挂了只影响路由层，Prefill/Decode worker 继续运行，备 SMG 接管后重新连上同一组 worker |

---

## 二、代码调研发现

### 2.1 SMG 已有 mesh 同步基础设施

**发送端（已实现）**：
- `cache_aware.rs:452-462`：`select_worker` 中每次插入 tree 后调用 `mesh_sync.sync_tree_operation()`
- `registry.rs:101-105`：worker 添加时同步 policy 状态到 mesh

**接收端（已实现但可能未完全接入）**：
- `cache_aware.rs:273-297`：`apply_remote_tree_operation()` 支持从 mesh 接收 Insert/Remove 操作
- `registry.rs:417-461`：`apply_remote_tree_operation()` 会尝试分发给 model policies、default policy、prefill policy、decode policy
- **关键注释**（`cache_aware.rs:271`）："`PolicyRegistry::apply_remote_tree_operation` (the only forwarder) currently has no in-process callers; the receive path is not yet wired"

**结论**：mesh 同步的**协议层和存储层已经存在**，但**接收端的触发/订阅机制可能还没完全打通**。这意味着如果我们要做主备树同步，大部分基础设施已经具备，只需补全接收端的订阅逻辑。

### 2.2 SMG 已有 router 发现和 HA port 设计意图

`service_discovery.rs:65`：
```rust
router_mesh_port_annotation: "sglang.ai/ha-port".to_string()
```

`service_discovery.rs:283-294`：
```rust
if !config.router_selector.is_empty() {
    info!(
        "Router node discovery enabled | selector: '{}' | mesh port annotation: '{}'",
        router_selector, config.router_mesh_port_annotation
    );
}
```

**结论**：SMG 的 service discovery 已经预留了 **router 节点发现** 和 **HA port**（`sglang.ai/ha-port`）的概念，说明作者在设计之初就考虑了 router 级高可用。这是主备架构的天然锚点。

### 2.3 SMG readiness 端点已支持 PD 模式健康判断

`server.rs:102-144`：
```rust
async fn readiness(State(state): State<Arc<AppState>>) -> Response {
    let workers = state.context.worker_registry.get_all();
    let healthy_workers: Vec<_> = workers.iter().filter(|w| w.is_healthy()).collect();

    let is_ready = match &state.context.router_config.mode {
        RoutingMode::PrefillDecode { .. } => {
            let has_prefill = healthy_workers.iter()
                .any(|w| matches!(w.worker_type(), WorkerType::Prefill { .. }));
            let has_decode = healthy_workers.iter()
                .any(|w| matches!(w.worker_type(), WorkerType::Decode));
            has_prefill && has_decode
        }
        // ...
    };
    // 返回 200 或 503
}
```

**结论**：SMG 的 `/readiness` 端点已经有成熟的逻辑判断自身是否准备好接收请求。我们可以**扩展这个端点**，让备实例在探测到主健康时返回 503，主不健康时返回 200。

### 2.4 SMG 的 worker 连接是懒加载的

`worker.rs:838-878`：`get_grpc_client()` 使用 `OnceCell` 懒加载 gRPC 客户端。

**结论**：备 SMG 即使配置了相同的 worker 地址，在接管前不会实际建立连接，资源消耗极小。

### 2.5 Higress 支持多级 Failover

从 Higress 官方文档调研：
- Higress 支持 **multi-level Failover**：服务实例级别、服务级别、跨集群级别
- Higress AI Gateway 支持 **multi-model Failover**：主模型失败自动切换到备用模型
- Higress 支持 **Gateway API Inference Extension (GIE)**：支持 priority-based scheduling

**关键结论**：Higress 在 AI 推理场景下已经有丰富的 failover 经验，但**标准 Ingress/ServiceEntry 不直接暴露"单端点级主备"能力**。需要通过更底层的 Envoy 配置或应用层配合实现。

---

## 三、方案对比与推荐

### 方案 A：K8s Service + Readiness Probe 主备切换（推荐）

**核心思想**：不改 Higress，不改 SMG 核心路由，只利用 K8s 原生的 readiness probe + Service endpoints 机制实现主备。

```
Higress
  │ 轮询 3 个 Service
  ├─────────────┬─────────────┬─────────────┐
  ▼             ▼             ▼
Service-1    Service-2    Service-3
  │             │             │
  ▼             ▼             ▼
SMG-1-主     SMG-2-主     SMG-3-主     ← readiness=200, 在 endpoints 中
SMG-1-备     SMG-2-备     SMG-3-备     ← readiness=503, 不在 endpoints 中
```

**正常状态**：
- 备 SMG 的 `/readiness` 返回 503
- K8s 不会将备的 IP 加入 Service Endpoints
- Higress 轮询只看到 3 个主实例

**故障状态（SMG-1-主挂了）**：
- SMG-1-主 readiness 变为失败，K8s 将其从 Service-1 endpoints 移除
- SMG-1-备探测到主不健康，自己的 readiness 变为 200
- K8s 将备加入 Service-1 endpoints
- Higress 继续轮询 3 个 Service，Service-1 现在指向 SMG-1-备

**优点**：
- Higress **零改动**
- K8s **标准机制**，运维友好
- 保留 Higress **熔断、outlier detection、retry** 全部能力
- cache-aware 树天然集中（每组只有一个活跃 SMG）

**缺点**：
- 切换速度受 K8s readiness 周期限制（默认 ~10s）
- 需要 SMG 新增"主备角色识别"和"peer 健康探测"逻辑
- mesh 树同步接收端需要补全

---

### 方案 B：Higress Wasm/Go 插件自定义主备路由

**核心思想**：在 Higress 网关层写一个自定义插件，维护主备映射，做主动主备路由决策。

```
Higress (Wasm Plugin)
  │ 维护主备状态表
  │ SMG-1-主 健康 → 发给 SMG-1-主
  │ SMG-1-主 挂了 → 发给 SMG-1-备
  ├──────────────────────────────────────┐
  ▼                                      ▼
SMG-1-主 / SMG-1-备                   SMG-2-主 ...
```

**优点**：
- SMG **零改动**
- 切换速度快（插件内内存判断，毫秒级）
- 可以灵活定义主备策略

**缺点**：
- 需要**开发 Higress 插件**（Go/Rust/JS）
- 插件本身成为**新的 SPOF**
- 插件状态管理复杂（如何感知 SMG 健康？需要额外的心跳机制）
- 与 Higress 版本耦合

---

### 方案 C：Istio EnvoyFilter Aggregate Cluster

**核心思想**：通过 Istio `EnvoyFilter` 直接修改 Envoy 配置，使用 Envoy 原生的 **aggregate cluster** 实现主备。

Envoy aggregate cluster 允许一个 cluster 由多个子 cluster 组成，每个子 cluster 可以配置不同的 priority。

**优点**：
- 原生 Envoy 能力，无需应用层改动
- 切换速度快

**缺点**：
- Istio CRD **不直接暴露** aggregate cluster
- 需要写复杂的 `EnvoyFilter` YAML，维护成本高
- 与 Istio 版本强耦合，升级风险大
- cache-aware 树同步问题依然存在（SMG 之间还是需要同步状态）

---

### 方案 D：SMG 内部分布式选举 + Higress 轮询透明转发

**核心思想**：所有 SMG 实例都接收请求，但应用层通过分布式锁（如 etcd）选举出一个 leader 处理请求，其他 follower 直接拒绝或转发。

**优点**：
- Higress 和 K8s **零改动**
- 切换可以在应用层快速完成

**缺点**：
- 每个请求先到 SMG，再被拒绝，**浪费一次网络往返**
- 需要引入 etcd/Redis 依赖，增加复杂度
- 拒绝请求会产生 5xx，影响 Higress 的 outlier detection 统计

---

## 四、推荐方案：方案 A（K8s Service + Readiness Probe）+ Mesh 同步增强

### 4.1 架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                            Higress Gateway                           │
│                    轮询 3 个上游 Service                              │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Service-1   │   │  Service-2   │   │  Service-3   │
│ (k8s service)│   │ (k8s service)│   │ (k8s service)│
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
   ┌───┴───┐          ┌───┴───┐          ┌───┴───┐
   ▼       ▼          ▼       ▼          ▼       ▼
 主(A)    备          主       备         主       备
 ━━━━
 活跃    静默         活跃     静默       活跃     静默
```

### 4.2 组件职责

| 层级 | 组件 | 主职责 | 备职责 |
|------|------|--------|--------|
| **网关层** | Higress | 轮询 3 个 Service，熔断/健康检查/失败转发 | 同主 |
| **K8s 层** | Service + Endpoints | 根据 readiness 动态维护可用端点 | 同主 |
| **SMG 层** | SMG 主实例 | 处理请求、维护 cache-aware 树、同步树到 mesh | 连接 worker、接收 mesh 树同步、readiness=503 |
| **SMG 层** | SMG 备实例 | 探测主健康、接收 mesh 树同步、readiness=503 | 探测主健康失败 → readiness=200 → 接管 |
| **Worker 层** | P/D Worker | 正常服务推理请求 | 同主 |

### 4.3 故障切换时序

```
t=0     SMG-1-主健康，SMG-1-备静默
        ┌─────┐
        │ 主  │ ← 处理请求，同步树到 mesh
        └──┬──┘
           │ mesh sync
        ┌──┴──┐
        │ 备  │ ← 接收树同步，readiness=503
        └─────┘

t=10s   SMG-1-主崩溃
        ┌─────┐
        │ 主  │ ✗ 进程退出 / 无响应
        └─────┘
        K8s readiness probe 失败 → 从 Service-1 endpoints 移除

        SMG-1-备检测到主失联
        ┌─────┐
        │ 备  │ → readiness 变为 200
        └──┬──┘
           │ K8s 加入 Service-1 endpoints
           ▼
        Higress 开始将流量发到 SMG-1-备

        SMG-1-备的树已和主同步（通过 mesh），无缝接管
```

### 4.4 需要改动的代码清单

#### 4.4.1 SMG 新增启动参数

```rust
// src/config/types.rs 或新增 ha_config.rs
pub struct HAConfig {
    /// 本实例的角色：Primary / Backup
    pub role: SMGRole,
    /// 本实例所属的组 ID（如 1, 2, 3）
    pub group_id: u32,
    /// 对端 SMG 地址（主的备需要知道主在哪，备的主需要知道备在哪）
    pub peer_address: Option<String>,
    /// 备实例探测主的周期（秒）
    pub peer_probe_interval_secs: u64,
    /// 备实例判定主失联的连续失败次数
    pub peer_probe_failure_threshold: u32,
}

pub enum SMGRole {
    Primary,
    Backup,
}
```

#### 4.4.2 扩展 readiness 端点

```rust
// src/server.rs
async fn readiness(State(state): State<Arc<AppState>>) -> Response {
    // 1. 原有的 worker 健康检查
    let base_ready = check_workers_ready(&state).await;
    
    // 2. 新增：HA 角色判断
    if let Some(ha) = &state.context.ha_config {
        match ha.role {
            SMGRole::Primary => {
                // 主实例：只要 worker 健康就 ready
                if base_ready { return ok(); } else { return not_ready(); }
            }
            SMGRole::Backup => {
                // 备实例：worker 健康 **且** 主实例已失联，才 ready
                if base_ready && state.ha_state.peer_unhealthy.load(Ordering::Relaxed) {
                    return ok();
                } else {
                    return not_ready(); // 503
                }
            }
        }
    }
    
    // 非 HA 模式，走原有逻辑
    ...
}
```

#### 4.4.3 新增 peer 健康探测任务

```rust
// src/core/ha_peer_probe.rs
pub async fn start_peer_probe_task(
    peer_address: String,
    interval: Duration,
    failure_threshold: u32,
    ha_state: Arc<HAState>,
) {
    let client = reqwest::Client::new();
    let mut consecutive_failures = 0;
    
    loop {
        tokio::time::sleep(interval).await;
        
        match client.get(format!("{}/health", peer_address)).send().await {
            Ok(res) if res.status().is_success() => {
                consecutive_failures = 0;
                ha_state.peer_unhealthy.store(false, Ordering::Relaxed);
            }
            _ => {
                consecutive_failures += 1;
                if consecutive_failures >= failure_threshold {
                    ha_state.peer_unhealthy.store(true, Ordering::Relaxed);
                }
            }
        }
    }
}
```

#### 4.4.4 补全 mesh 树同步接收端

```rust
// src/policies/registry.rs
// apply_remote_tree_operation 已经存在，需要补全订阅机制

// 在 mesh handler 或 worker manager 中，当收到远程 tree operation 时调用：
pub fn on_mesh_tree_operation_received(
    &self,
    mesh_key: &str,
    operation: &TreeOperation,
) {
    self.apply_remote_tree_operation(mesh_key, operation);
}
```

需要确认 `smg_mesh` 的 `MeshSyncManager` 是否提供了订阅回调接口，如果没有，需要新增一个后台任务定期从 mesh store 拉取更新。

#### 4.4.5 K8s Deployment 配置调整

每个 SMG 组需要两个 Deployment：

```yaml
# SMG-1 主实例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kimi-k25-router-5p3d-0-primary
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: smg
          args:
            - --group-id=1
            - --role=primary
            - --peer-address=http://kimi-k25-router-5p3d-0-backup:30002
          readinessProbe:
            httpGet:
              path: /readiness
              port: 30002
            periodSeconds: 5
            failureThreshold: 3

---
# SMG-1 备实例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kimi-k25-router-5p3d-0-backup
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: smg
          args:
            - --group-id=1
            - --role=backup
            - --peer-address=http://kimi-k25-router-5p3d-0-primary:30002
            - --peer-probe-interval-secs=5
            - --peer-probe-failure-threshold=3
          readinessProbe:
            httpGet:
              path: /readiness
              port: 30002
            periodSeconds: 5
            failureThreshold: 3
```

#### 4.4.6 K8s Service 配置

```yaml
# 每个组一个 Service，同时选中主和备的 Pod
apiVersion: v1
kind: Service
metadata:
  name: kimi-k25-router-group-1
spec:
  selector:
    app: kimi-k25-router
    group-id: "1"
  ports:
    - port: 30002
      targetPort: 30002
```

> 注意：Service 不区分主/备标签，只按 `group-id` 选择。readiness probe 会自动过滤掉不 ready 的备实例。

### 4.5 Higress 配置调整

Higress 的 `ServiceEntry` 从指向 6 个 Pod FQDN，改为指向 3 个 K8s Service：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: kimi-router-ha-se
  namespace: higress-system
spec:
  endpoints:
    - address: kimi-k25-router-group-1.default.svc.cluster.local
    - address: kimi-k25-router-group-2.default.svc.cluster.local
    - address: kimi-k25-router-group-3.default.svc.cluster.local
  hosts:
    - kimi-router-ha-cluster.local
  location: MESH_INTERNAL
  ports:
    - name: http
      number: 30002
      protocol: HTTP
  resolution: DNS
```

Higress 的熔断和健康检查依然生效：
- `outlierDetection` 监控的是 Service 解析后的端点（即当前活跃的主或备）
- 如果主挂了、备还没ready，该组没有健康端点，Higress 会将该组隔离
- 如果备接管后也不健康，Higress 同样会隔离

---

## 五、风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| **切换延迟** | 主挂了 ~10-15s 内请求失败 | 1. 缩短 readiness probe 周期到 3s；2. Higress retry 配置重试到其他组；3. 客户端实现重试 |
| **树同步延迟** | 备的树和主有 gap | 1. 缩短 mesh gossip 间隔；2. 备接管后允许少量 cache miss（SMG 的 cache_aware 本身就有 miss fallback） |
| **双主（Split Brain）** | 网络分区时主备都活跃 | 1. 使用更严格的探测（如同时探测主和 P/D worker）；2. 备只在主 **连续多次** 探测失败后才提升；3. 可选：引入 etcd 分布式锁做仲裁 |
| **备资源浪费** | 备 SMG 占用 Pod 但不处理请求 | 1. 备的资源可以配小一些（CPU/内存减半）；2. 备的 worker 连接是懒加载的，不占用 worker 连接池 |
| **回切抖动** | 主恢复后，备立刻回到静默，但主还没完全预热 | 1. 配置延迟回切（主恢复后等待 30s 再让备静默）；2. 或不自动回切，人工介入 |

---

## 六、演进路线

### Phase 1：MVP（最小可用）
- SMG 新增 `--role` / `--peer-address` 参数
- 扩展 readiness 端点
- 新增 peer 健康探测后台任务
- K8s Service 从 1 个改为 3 个（按组）
- 不依赖 mesh 树同步（备接管后从空树开始，允许缓存未命中）

### Phase 2：Mesh 同步（优化缓存命中率）
- 补全 mesh 树同步接收端
- 主实时同步 cache_aware 树到 mesh
- 备实时接收并重建树
- 备接管后 cache-aware 树是"热"的

### Phase 3：自动回切 + 智能降级
- 主恢复后，自动探测主状态，延迟回切
- Higress 层支持按组权重动态调整
- 支持跨组降级（如 Group-1 全挂，流量分到 Group-2/3）

---

## 七、总结

| 维度 | 结论 |
|------|------|
| **推荐方案** | 方案 A：K8s Service + Readiness Probe + SMG HA 角色感知 |
| **Higress 改动** | 无，仅调整 ServiceEntry 指向 Service 而非 Pod |
| **K8s 改动** | 中等：3 个 Service + 6 个 Deployment + readiness 配置 |
| **SMG 改动** | 中等：新增 HA 配置、扩展 readiness、新增 peer 探测、mesh 接收端补全 |
| **cache-aware 集中** | ✅ 每组只有一个活跃 SMG |
| **Higress 熔断保留** | ✅ 完全保留 |
| **P/D 不中断** | ✅ worker 独立运行，备接管后重连同一组 worker |
| **切换速度** | ~10s（可调优到 ~5s） |


---

## 八、结合官方文档的新思考（Post-Doc Review）

在撰写本方案后，我查阅了 SGLang 官方文档的 [sgl_model_gateway](https://docs.sglang.io/docs/advanced_features/sgl_model_gateway) 页面，发现了几个与我们的架构设计高度相关的官方立场和能力，需要补充到方案中。

### 8.1 官方对 HA 和 cache tree 分散的明确态度

官方文档 **Production Recommendations → High Availability** 章节明确写道：

> "With multiple replicas, the cache-aware routing policy's radix tree is **not synchronized across replicas**. This means each replica builds its own cache tree, requests from the same user may hit different replicas. Expected cache hit rate reduction: **10-20%**."

并给出了两条建议：

1. **Accept cache efficiency trade-off**（接受 10-20% 的缓存效率损失）
2. **Configure session affinity (optional)**：基于 user ID 或 API key 的一致性哈希，让同一用户的请求尽量落到同一个 replica

**对我们方案的影响**：
- 官方**没有计划**在标准多 replica 部署中实现 cache tree 共享
- 官方明确接受 10-20% 的 cache hit 下降作为 HA 的代价
- Session affinity 只是**缓解**手段，不能根本解决树分散问题
- 这反而**强化了我们主备方案的价值**：主备架构可以让 cache tree 在每组内保持集中，避免这 10-20% 的损失

### 8.2 Session Affinity 的局限性与主备方案的优势对比

| 维度 | 官方 Session Affinity 方案 | 我们的主备方案 |
|------|------------------------|-------------|
| **实现层级** | L4/L7 负载均衡器（如 Higress、Nginx） | K8s Service + SMG 应用层 |
| **核心假设** | "同一用户的请求应路由到同一 replica" | "每组只有一个活跃 SMG，树天然集中" |
| **Cache Hit 保障** | 概率性（哈希碰撞、replica 扩容/缩容都会导致漂移） | 确定性（每组内 100% 集中） |
| **故障切换** | Replica 挂了，用户请求漂移，cache miss | 备自动接管，树保持连续 |
| **新用户首次请求** | 无历史，必然 cache miss | 同左，但后续请求全部集中在同一组 |
| **官方认可度** | 官方文档明确推荐 | 文档未提及，但架构上更优雅 |

**结论**：主备方案比 session affinity **更彻底**地解决了 cache tree 分散问题，且不需要在 Higress 层做复杂的一致性哈希配置。

### 8.3 WASM 扩展对主备方案的价值评估

官方文档显示 SMG 支持 **WASM Middleware**，可以在 `OnRequest` / `OnResponse` 阶段做自定义处理（认证、限流、日志、请求/响应修改）。

但文档同时明确限制了 WASM 的运行环境：

> "WASM middleware runs in a sandboxed environment with memory isolation, **no network/filesystem access**, and configurable resource limits."

**这意味着**：
- ❌ WASM 插件**不能**做 HTTP 请求转发（备 SMG 无法通过 WASM 把请求转发给主 SMG）
- ❌ WASM 插件**不能**访问文件系统或网络来做 peer 健康探测
- ✅ WASM 插件**可以**用于：请求鉴权、限流、日志记录、请求/响应体修改

**对主备方案的结论**：WASM 扩展**不能替代**我们在 SMG 应用层做的 HA 角色感知和 peer 探测，但可以在主备架构之上叠加额外的能力（如在 `OnRequest` 阶段做更精细的限流、在 `OnResponse` 阶段做响应审计）。

### 8.4 SMG 作为通用推理网关的启示

文档展示了 SMG 支持多种部署模式，不限于 PD 分离：

| 模式 | 说明 |
|------|------|
| Regular HTTP Routing | 普通 worker 轮询 |
| gRPC Launch | gRPC worker，最高吞吐 |
| **Prefill-Decode Disaggregation** | PD 分离（我们的场景） |
| OpenAI Backend Proxy | 代理 OpenAI/xAI 等外部 API |
| Multi-Model Inference Gateway (IGW) | 多模型动态路由 |

**启示**：
- 我们的主备方案**不仅适用于 PD 模式**，也适用于 Regular、gRPC、OpenAI Proxy 等其他模式
- 如果未来在 IGW 模式下为多模型部署主备，同样的架构可以复用
- 这意味着 HA 角色的设计应该是**与路由模式解耦**的通用能力

### 8.5 对 Phase 1 方案的修正

结合官方立场，我建议**调整演进路线**：

| 阶段 | 原方案 | 修正后方案 |
|------|--------|-----------|
| **Phase 1（MVP）** | 不依赖 mesh 同步，备从空树开始 | **保持**，这与官方"accept 10-20% trade-off"的哲学一致。备接管后允许 cache miss，SMG 的 cache_aware 本身就有 miss fallback（回退到最短队列） |
| **Phase 2（优化）** | 补全 mesh 接收端，实现树同步 | **降级为可选增强**，而非必做。如果 Phase 1 的 cache miss 率可接受，可以跳过 mesh 同步，降低复杂度 |
| **Phase 3（智能化）** | 自动回切 + 跨组降级 | **保持**，但优先级降低 |

**修正后的最小可行方案（MVP）**：
1. SMG 新增 `--role=primary/backup`、`--peer-address` 参数
2. 扩展 `/readiness`：备探测主健康，主在时返回 503
3. K8s 从 1 个 Service 改为 3 个 Service（按组）
4. **不做 mesh 树同步**（接受官方推荐的 trade-off）
5. Higress ServiceEntry 指向 3 个 Service

这个 MVP 的改动量更小，风险更低，且与官方文档的 HA 建议**方向一致但效果更优**。

### 8.6 新发现：Service Discovery 的 router 发现能力

官方文档的 **Service Discovery** 章节展示了 K8s 自动发现 worker 的能力。结合我们代码调研中发现的 `router_selector` 和 `sglang.ai/ha-port`，可以推断：

SMG 的 service discovery 已经具备**发现其他 router 节点**的能力（通过 `router_selector` 和 `router_mesh_port_annotation`）。这意味着：
- 在 K8s 环境中，SMG 实例可以通过标签自动发现同组的 peer（主发现备、备发现主）
- 可能**不需要手动配置 `--peer-address`**，而是让 service discovery 自动填充
- 这进一步降低了运维复杂度

### 8.7 最终结论更新

| 维度 | 结论（更新后） |
|------|--------------|
| **问题确认** | 官方文档明确承认多 replica 下 cache tree 不共享，接受 10-20% cache hit 下降 |
| **官方推荐方案** | Session affinity（一致性哈希）+ 接受 trade-off |
| **我们的方案优势** | 主备架构比 session affinity 更彻底地解决 tree 分散，且实现复杂度可控 |
| **MVP 复杂度** | **进一步降低**：不做 mesh 同步，只做 readiness 驱动的主备切换 |
| **WASM 价值** | 对主备核心逻辑无直接帮助，但可作为增强层（限流、审计） |
| **Service Discovery** | 可利用 `router_selector` 自动发现 peer，可能无需硬编码 `--peer-address` |


---

## 附录 A：参考链接

### A.1 SGLang 官方文档

- https://docs.sglang.io/docs/advanced_features/sgl_model_gateway — SGLang Model Gateway 官方文档（本文档分析的主要来源）
- https://docs.sglang.io/llms.txt — SGLang 文档索引

### A.2 Higress 官方文档

- https://higress.ai/en/docs/latest/overview/what-is-higress — Higress 概述
- https://higress.ai/en/glossary/failover — Higress Failover 机制说明
- https://github.com/alibaba/higress — Higress 开源仓库

### A.3 Istio / Envoy 参考

- https://istio.io/latest/docs/reference/config/networking/service-entry/ — Istio ServiceEntry 官方文档
- https://istio.io/latest/docs/reference/config/networking/destination-rule/ — Istio DestinationRule 官方文档
- https://istio.io/latest/docs/reference/config/networking/virtual-service/ — Istio VirtualService 官方文档
- https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/priority — Envoy Priority Load Balancing

### A.4 Kubernetes 参考

- https://kubernetes.io/docs/concepts/services-networking/service/ — K8s Service 官方文档
- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/ — K8s Readiness Probe 官方文档

---

## 附录 B：参考代码

以下代码片段均来自当前工作目录下克隆的 `sglang` 源代码（`sglang/sgl-model-gateway/`）。

### B.1 cache_aware 前缀树与 mesh 同步

**文件**：`src/policies/cache_aware.rs`

核心逻辑：select_worker 中前缀匹配后插入 tree，并同步到 mesh。

```rust
// src/policies/cache_aware.rs:448-462
if let Some(idx) = selected_idx {
    tree.insert(text, workers[idx].url());

    // Sync insert operation to mesh if enabled (no-op if mesh is not enabled)
    if let Some(ref mesh_sync) = self.mesh_sync {
        use smg_mesh::tree_ops::TreeInsertOp;
        let op = TreeOperation::Insert(TreeInsertOp {
            text: text.to_string(),
            tenant: workers[idx].url().to_string(),
        });
        let mesh_key = Self::normalize_mesh_model_id(&tree_key);
        if let Err(e) = mesh_sync.sync_tree_operation(mesh_key.to_string(), op) {
            warn!("Failed to sync tree insert operation to mesh: {}", e);
        }
    }
    workers[idx].increment_processed();
    return Some(idx);
}
```

接收端（已实现但触发路径未完全接入）：

```rust
// src/policies/cache_aware.rs:273-297
pub fn apply_remote_tree_operation(&self, mesh_key: &str, operation: &TreeOperation) {
    let tree_key = Self::normalize_mesh_model_id(mesh_key);
    let tree = self
        .trees
        .entry(tree_key.to_string())
        .or_insert_with(|| Arc::new(Tree::new()));

    match operation {
        TreeOperation::Insert(insert_op) => {
            tree.insert(&insert_op.text, &insert_op.tenant);
        }
        TreeOperation::Remove(remove_op) => {
            tree.remove_tenant(&remove_op.tenant);
        }
    }
}
```

### B.2 PolicyRegistry 的 mesh 同步分发

**文件**：`src/policies/registry.rs`

```rust
// src/policies/registry.rs:417-461
pub fn apply_remote_tree_operation(
    &self,
    model_id: &str,
    operation: &smg_mesh::tree_ops::TreeOperation,
) {
    // Try to find the policy for this model
    if let Some(policy) = self.get_policy(model_id) {
        if policy.name() == "cache_aware" {
            if let Some(cache_aware) = policy.as_any().downcast_ref::<CacheAwarePolicy>() {
                cache_aware.apply_remote_tree_operation(model_id, operation);
            }
        }
    }

    // Also check default policy if it's cache-aware
    if self.default_policy.name() == "cache_aware" {
        if let Some(cache_aware) = self.default_policy.as_any().downcast_ref::<CacheAwarePolicy>() {
            cache_aware.apply_remote_tree_operation(model_id, operation);
        }
    }

    // Check prefill and decode policies for PD mode
    if let Some(prefill_policy) = self.prefill_policy.get() {
        if prefill_policy.name() == "cache_aware" {
            if let Some(cache_aware) = prefill_policy.as_any().downcast_ref::<CacheAwarePolicy>() {
                cache_aware.apply_remote_tree_operation(model_id, operation);
            }
        }
    }

    if let Some(decode_policy) = self.decode_policy.get() {
        if decode_policy.name() == "cache_aware" {
            if let Some(cache_aware) = decode_policy.as_any().downcast_ref::<CacheAwarePolicy>() {
                cache_aware.apply_remote_tree_operation(model_id, operation);
            }
        }
    }
}
```

### B.3 readiness 端点（PD 模式健康判断）

**文件**：`src/server.rs`

```rust
// src/server.rs:102-144
async fn readiness(State(state): State<Arc<AppState>>) -> Response {
    let workers = state.context.worker_registry.get_all();
    let healthy_workers: Vec<_> = workers.iter().filter(|w| w.is_healthy()).collect();

    let is_ready = if state.context.router_config.enable_igw {
        !healthy_workers.is_empty()
    } else {
        match &state.context.router_config.mode {
            RoutingMode::PrefillDecode { .. } => {
                let has_prefill = healthy_workers
                    .iter()
                    .any(|w| matches!(w.worker_type(), WorkerType::Prefill { .. }));
                let has_decode = healthy_workers
                    .iter()
                    .any(|w| matches!(w.worker_type(), WorkerType::Decode));
                has_prefill && has_decode
            }
            RoutingMode::Regular { .. } => !healthy_workers.is_empty(),
            RoutingMode::OpenAI { .. } => !healthy_workers.is_empty(),
        }
    };

    if is_ready {
        (StatusCode::OK, Json(json!({"status": "ready", ... }))).into_response()
    } else {
        (StatusCode::SERVICE_UNAVAILABLE, Json(json!({"status": "not ready", ... }))).into_response()
    }
}
```

### B.4 Service Discovery 中的 router 发现与 HA port

**文件**：`src/service_discovery.rs`

```rust
// src/service_discovery.rs:44-48
// Router node discovery for mesh
pub router_selector: HashMap<String, String>,
pub router_mesh_port_annotation: String,

// src/service_discovery.rs:64-65 (默认值)
router_selector: HashMap::new(),
router_mesh_port_annotation: "sglang.ai/ha-port".to_string(),

// src/service_discovery.rs:185-200
let is_router = if let Some(config) = config {
    !config.router_selector.is_empty()
        && Self::matches_selector(pod, &config.router_selector)
} else {
    false
};

let mesh_port = if is_router {
    if let Some(config) = config {
        pod.metadata
            .annotations
            .as_ref()
            .and_then(|annotations| annotations.get(&config.router_mesh_port_annotation))
            .and_then(|port_str| port_str.parse::<u16>().ok())
    } else {
        None
    }
} else {
    None
};
```

### B.5 Worker 懒加载 gRPC 客户端

**文件**：`src/core/worker.rs`

```rust
// src/core/worker.rs:838-878
async fn get_grpc_client(&self) -> WorkerResult<Option<Arc<GrpcClient>>> {
    match self.metadata.connection_mode {
        ConnectionMode::Http => Ok(None),
        ConnectionMode::Grpc { .. } => {
            let client = self
                .grpc_client
                .get_or_try_init(|| async {
                    match GrpcClient::connect(&self.metadata.url, &runtime_str).await {
                        Ok(client) => Ok(Arc::new(client)),
                        Err(e) => Err(WorkerError::ConnectionFailed { ... }),
                    }
                })
                .await?;
            Ok(Some(Arc::clone(client)))
        }
    }
}
```

### B.6 CircuitBreaker 状态机

**文件**：`src/core/circuit_breaker.rs`

```rust
// src/core/circuit_breaker.rs:34-48
const STATE_CLOSED: u8 = 0;
const STATE_OPEN: u8 = 1;
const STATE_HALF_OPEN: u8 = 2;

pub enum CircuitState {
    Closed,    // Normal operation, requests allowed
    Open,      // Circuit is open, requests rejected
    HalfOpen,  // Testing if service has recovered
}
```

### B.7 mesh 管理 HTTP API

**文件**：`src/routers/mesh/handlers.rs`

SMG 提供的 mesh 集群管理 REST API：

- `GET /mesh/cluster` — 获取 mesh 集群状态
- `GET /mesh/health` — mesh 健康检查
- `GET /mesh/workers` — 获取 mesh store 中的 worker 状态
- `GET /mesh/policies` — 获取 mesh store 中的 policy 状态
- `GET/POST /mesh/global_rate_limit` — 全局限流配置
- `POST /mesh/shutdown` — 优雅关闭

### B.8 外部 ServiceEntry YAML（用户生产配置）

**文件**：`k8s-higress-smg-configs/serviceentry.yaml`

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: kimi-router-ha-se
  namespace: higress-system
spec:
  endpoints:
    - address: kimi-k25-router-5p3d-0.default.svc.cluster.local
    - address: kimi-k25-router-5p3d-1.default.svc.cluster.local
    - address: kimi-k25-router-5p3d-2.default.svc.cluster.local
  hosts:
    - kimi-router-ha-cluster.local
  location: MESH_INTERNAL
  ports:
    - name: http
      number: 30002
      protocol: HTTP
  resolution: DNS
```

---

## 附录 C：关键代码文件索引

- `sglang/sgl-model-gateway/src/policies/cache_aware.rs` — cache_aware 策略实现，含 mesh 同步发送/接收
- `sglang/sgl-model-gateway/src/policies/registry.rs` — PolicyRegistry，管理 model→policy 映射，含 mesh 同步分发
- `sglang/sgl-model-gateway/src/policies/tree.rs` — Radix Tree 数据结构实现
- `sglang/sgl-model-gateway/src/core/worker.rs` — Worker trait 及 BasicWorker 实现，含懒加载 gRPC 客户端
- `sglang/sgl-model-gateway/src/core/worker_registry.rs` — WorkerRegistry，含一致性哈希环
- `sglang/sgl-model-gateway/src/core/circuit_breaker.rs` — 熔断器状态机实现
- `sglang/sgl-model-gateway/src/server.rs` — HTTP 服务入口，含 readiness/liveness 端点
- `sglang/sgl-model-gateway/src/service_discovery.rs` — K8s Pod 服务发现，含 router 发现和 HA port
- `sglang/sgl-model-gateway/src/routers/mesh/handlers.rs` — mesh 管理 REST API
- `sglang/sgl-model-gateway/src/routers/http/pd_router.rs` — HTTP PD 路由实现，含 dual dispatch
- `sglang/sgl-model-gateway/src/routers/grpc/pd_router.rs` — gRPC PD 路由实现
- `sglang/sgl-model-gateway/src/config/types.rs` — RouterConfig / RoutingMode / PolicyConfig 类型定义
