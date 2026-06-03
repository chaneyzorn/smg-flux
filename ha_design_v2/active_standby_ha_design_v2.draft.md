# SMG Active-Standby 高可用架构设计 V2 — 解决 L4 延迟与多备扩展缺陷

> 目标：解决 V1 方案（K8s Service + Readiness Probe）的两个核心缺陷：
> 1. L4 探测延迟：Service 的 readiness probe 存在时间窗口，Higress（L7）无法第一时间感知并决策
> 2. 多备扩展性：1主1备设计无法优雅扩展到 1主N备

---

## 一、缺陷根因分析

### 1.1 缺陷1：L4 vs L7 感知断层

```
Client
  │
  ▼
[Higress — L7 网关]
  │ 轮询 3 个 Service
  ├─────────────┬─────────────┬─────────────┐
  ▼             ▼             ▼
[Service-1]  [Service-2]  [Service-3]     ← K8s Service（L4）
  │             │             │
  ▼             ▼             ▼
readiness    readiness    readiness
  probe       probe        probe
  (3s)        (3s)         (3s)
  │             │             │
  ▼             ▼             ▼
主(活跃)      主(活跃)      主(活跃)
备(静默)      备(静默)      备(静默)
```

**问题链条**：
1. SMG-1-主崩溃
2. K8s kubelet 的 readiness probe 需要 **periodSeconds × failureThreshold = 3s × 2 = 6s** 才能确认主已不健康
3. K8s 将主从 Service-1 endpoints 移除
4. Higress DNS 解析 Service-1，发现端点变化（DNS TTL 缓存可能额外增加 5-30s）
5. 窗口期内（~6-30s），Higress 仍可能将请求发到已崩溃的主
6. 即使备在 3s 内已 ready，Higress 也无法第一时间知道

**本质**：故障检测和流量切换被割裂在 **L4（K8s Service）** 和 **L7（Higress）** 两层，L4 的探测周期成为整个链路的瓶颈。

### 1.2 缺陷2：多备的脑裂风险

V1 方案中，备节点的 readiness 逻辑：
```
备探测主不健康 → 备 readiness = 200 → K8s 加入 endpoints
```

如果有 **两个备**（备A 和 备B）：
1. 主挂了
2. 备A 探测到主不健康，readiness = 200
3. 备B 也探测到主不健康，readiness = 200
4. K8s 将备A 和 备B 都加入 endpoints
5. Higress 轮询到两个备，cache_aware 树同时分散到两个备上
6. **脑裂：两个备同时接收请求，cache tree 再次分散**

**本质**：V1 方案依赖 "只有一个备 ready" 的假设，多备会破坏这个假设。

---

## 二、调研过程与关键发现

### 2.1 调研方向

针对两个缺陷，我们调研了以下方向：

| 方向 | 目标 | 结论 |
|------|------|------|
| Higress 原生 failover 能力 | 是否有高层 CRD 支持主备切换 | ❌ 商业版支持，开源版需 EnvoyFilter |
| Istio locality-based failover | 利用 region/zone/subzone 实现优先级 | ❌ 为跨地域设计，不适合单集群主备 |
| EnvoyFilter active health check | 在 Envoy L7 层配置主动健康检查 | ✅ Higress 官方确认可用 |
| Envoy priority + load_assignment | 端点级 priority 实现 failover | ⚠️ 可行但 MERGE 可能与 ServiceEntry 冲突 |
| Envoy aggregate cluster | cluster 级 failover 实现多级 fallback | ✅ 最 clean 的方案，需要 ADD 新 cluster |
| Higress AI Gateway fallback | 利用多模型 failover 泛化到 SMG | ❌ 特定于模型路由，不适用 |
| Custom K8s Controller | 自研 controller 管理 EndpointSlice | ✅ 完全可控，但多一个组件 |

### 2.2 Higress 原生能力调研

Higress 开源版 issue #287 的 maintainer 明确回复：
> "健康检查可以基于 istio envoyfilter 来实现，对应 higress 上的可以 work 的配置"

Higress 商业版对比表显示：
- Active Health Check: ✅
- Passive Health Check (outlier detection): ✅
- Fallback: ✅

但开源版没有封装好的 failover CRD，必须走 EnvoyFilter 底层配置。

### 2.3 Istio Locality-Based Failover 调研

Istio 通过 `DestinationRule.localityLbSetting` + `outlierDetection` 支持基于地域的 failover：

```yaml
trafficPolicy:
  loadBalancer:
    localityLbSetting:
      enabled: true
      failover:
        - from: us-east-1
          to: us-west-2
  outlierDetection:
    consecutive5xxErrors: 3
    interval: 30s
```

**关键限制**：
- `failover.from/to` 是 **region 级别**，不能区分同一 region 内的主备
- locality 由 **K8s 节点标签**（`topology.kubernetes.io/region/zone`）决定，不是 Pod 级别
- 如果将主备硬编码到不同 zone 的节点上，会引入调度复杂度
- **不适合单集群内同一服务的主备切换场景**

### 2.4 EnvoyFilter Active Health Check 验证

多个来源验证了通过 EnvoyFilter 配置 `health_checks` 的可行性：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
spec:
  configPatches:
  - applyTo: CLUSTER
    match:
      context: GATEWAY
      cluster:
        service: my-service.local
    patch:
      operation: MERGE
      value:
        health_checks:
        - timeout: 2s
          interval: 2s
          unhealthy_threshold: 1
          http_health_check:
            path: /healthz
```

Higress 官方 issue 确认此配置在 Higress 上可以工作。

### 2.5 Envoy Priority 与 Aggregate Cluster 调研

Envoy 支持两种主备机制：

**机制A：端点级 Priority（`load_assignment.endpoints.priority`）**
- 在同一 cluster 内，端点可以分配不同的 priority（P0/P1/P2）
- Envoy 的 `locality weighted load balancing` 根据每个 priority group 的健康比例动态分配流量
- 当 P0 健康比例 × overprovisioning_factor < 100% 时，流量降级到 P1
- **问题**：EnvoyFilter 的 `MERGE` 操作对 repeated 字段（如 `endpoints`）是**追加**而非替换，可能与 Istio 自动生成的 endpoints 冲突

**机制B：Cluster 级 Aggregate Cluster**
- Envoy 的 `envoy.clusters.aggregate` 类型可以将多个子 cluster 组合成一个 aggregate cluster
- 按顺序尝试子 cluster：primary → backup-1 → backup-2
- 当 primary cluster 的所有端点都不健康时，自动降级到 backup-1 cluster
- **优势**：完全独立于 Istio 的自动生成配置，通过 `ADD` 操作创建新 cluster，无冲突风险
- **劣势**：需要通过 HTTP_ROUTE patch 将流量重定向到 aggregate cluster

### 2.6 Istio 社区 Failover 提案

Istio issue #39445 提出了在 `DestinationRule` 中引入 `failover` 字段：

```yaml
trafficPolicy:
  loadBalancer:
    failover:
      destination:
        host: homepage.sample.svc.cluster.local
        subset: v2
```

该提案计划底层使用 Envoy Aggregate Cluster 实现。但截至当前（2026-06），该 issue 仍处于 open 状态，**尚未落地**。

---

## 三、新方案核心思路

### 3.1 方案名称

**Envoy L7 Active Health Check + Aggregate Cluster Failover**

### 3.2 核心思想

**绕过 K8s Service 的 L4 readiness probe，在 Envoy（Higress 底层）的 L7 层实现主动健康检查 + 聚合集群故障转移。**

```
Client
  │
  ▼
[Higress — Envoy L7]
  │
  ├─ active health check ───────┬──────────────┐
  │  /health every 2s           │              │
  ▼                             ▼              ▼
[SMG-1-主]  ←── primary ──→  [SMG-1-备A] ← backup-1 → [SMG-1-备B] ← backup-2
  100%流量                       0%流量              0%流量

主挂了 → primary cluster 标记为 UNHEALTHY
  → Aggregate Cluster 自动将 100% 流量切换到 backup-1（备A）

备A 也挂了 → 流量自动切换到 backup-2（备B）
```

**关键改变**：
- 不再使用 K8s Service 做分组和负载均衡
- 为每组 SMG 创建一个 Envoy **Aggregate Cluster**，包含 primary 和多个 backup 子 cluster
- 每个子 cluster 配置独立的 **active health check**（HTTP `/health`，2s 间隔）
- Envoy 根据子 cluster 的健康状态自动做 failover
- 切换延迟 = health check interval（可配置为 1-2s）

### 3.3 解决缺陷1：L7 主动探测 vs L4 被动探测

| 维度 | V1 方案（K8s Service） | V2 方案（Envoy Active Health Check） |
|------|----------------------|-------------------------------------|
| 探测层级 | L4（TCP 连接） | **L7（HTTP GET /health）** |
| 探测方向 | K8s kubelet → Pod（被动） | Envoy → Pod（主动） |
| 探测周期 | 3-10s | **1-2s** |
| 失败阈值 | 2-3 次 | **1 次** |
| 切换延迟 | ~6-30s | **~2-4s** |
| 探测精度 | 只能检测进程是否存活 | 能检测 SMG 内部状态（如 worker 是否健康） |

Envoy 的 active health check 在请求到达前就已知道端点是否健康，实现了**真正的 preemptive failover**。

### 3.4 解决缺陷2：Aggregate Cluster 天然支持多备

Envoy 的 aggregate cluster 支持多级 fallback：

```
aggregate cluster: smg-group-1
  ├─ cluster: smg-group-1-primary
  ├─ cluster: smg-group-1-backup-1
  ├─ cluster: smg-group-1-backup-2
  └─ cluster: smg-group-1-backup-3
```

**行为**：
- 正常情况下：primary 健康 → 100% 流量去 primary
- 主挂了：primary 不健康 → 100% 流量去 backup-1
- 备A 也挂了：backup-1 不健康 → 100% 流量去 backup-2
- 以此类推

**只有一个子 cluster 会接收流量**，天然避免了多备脑裂。

---

## 四、方案设计

### 4.1 部署模型

**使用 StatefulSet + Headless Service**：

```yaml
# StatefulSet 保证 Pod 网络标识固定
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: smg-group-1
  namespace: default
spec:
  serviceName: smg-group-1-headless
  replicas: 3  # 1 主 + 2 备
  selector:
    matchLabels:
      app: smg-group-1
  template:
    metadata:
      labels:
        app: smg-group-1
    spec:
      containers:
        - name: smg
          image: lmsysorg/sgl-model-gateway:latest
          ports:
            - containerPort: 30002
          args:
            - --pd-disaggregation
            - --prefill http://prefill-group-1:30001 9001
            - --decode http://decode-group-1:30011
            - --policy cache_aware
```

Pod 固定名称：
- `smg-group-1-0` → 主
- `smg-group-1-1` → 备A
- `smg-group-1-2` → 备B

Headless Service 提供固定 FQDN：
- `smg-group-1-0.smg-group-1-headless.default.svc.cluster.local`
- `smg-group-1-1.smg-group-1-headless.default.svc.cluster.local`
- `smg-group-1-2.smg-group-1-headless.default.svc.cluster.local`

### 4.2 核心配置：EnvoyFilter Aggregate Cluster

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: smg-group-1-aggregate
  namespace: higress-system
spec:
  workloadSelector:
    labels:
      app: higress-gateway
  configPatches:
    # ---------------------------------------------------------------
    # 1. Primary Cluster — 主节点
    # ---------------------------------------------------------------
    - applyTo: CLUSTER
      patch:
        operation: ADD
        value:
          name: smg-group-1-primary
          type: STRICT_DNS
          connect_timeout: 5s
          lb_policy: ROUND_ROBIN
          health_checks:
            - timeout: 3s
              interval: 2s
              unhealthy_threshold: 1
              healthy_threshold: 1
              http_health_check:
                path: /health
          load_assignment:
            cluster_name: smg-group-1-primary
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: smg-group-1-0.smg-group-1-headless.default.svc.cluster.local
                          port_value: 30002

    # ---------------------------------------------------------------
    # 2. Backup-1 Cluster — 备A
    # ---------------------------------------------------------------
    - applyTo: CLUSTER
      patch:
        operation: ADD
        value:
          name: smg-group-1-backup-1
          type: STRICT_DNS
          connect_timeout: 5s
          lb_policy: ROUND_ROBIN
          health_checks:
            - timeout: 3s
              interval: 2s
              unhealthy_threshold: 1
              healthy_threshold: 1
              http_health_check:
                path: /health
          load_assignment:
            cluster_name: smg-group-1-backup-1
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: smg-group-1-1.smg-group-1-headless.default.svc.cluster.local
                          port_value: 30002

    # ---------------------------------------------------------------
    # 3. Backup-2 Cluster — 备B（可选，支持更多备继续添加）
    # ---------------------------------------------------------------
    - applyTo: CLUSTER
      patch:
        operation: ADD
        value:
          name: smg-group-1-backup-2
          type: STRICT_DNS
          connect_timeout: 5s
          lb_policy: ROUND_ROBIN
          health_checks:
            - timeout: 3s
              interval: 2s
              unhealthy_threshold: 1
              healthy_threshold: 1
              http_health_check:
                path: /health
          load_assignment:
            cluster_name: smg-group-1-backup-2
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: smg-group-1-2.smg-group-1-headless.default.svc.cluster.local
                          port_value: 30002

    # ---------------------------------------------------------------
    # 4. Aggregate Cluster — 聚合主备
    # ---------------------------------------------------------------
    - applyTo: CLUSTER
      patch:
        operation: ADD
        value:
          name: smg-group-1-aggregate
          connect_timeout: 0.25s
          lb_policy: CLUSTER_PROVIDED
          cluster_type:
            name: envoy.clusters.aggregate
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.clusters.aggregate.v3.ClusterConfig
              clusters:
                - smg-group-1-primary
                - smg-group-1-backup-1
                - smg-group-1-backup-2

    # ---------------------------------------------------------------
    # 5. Route Patch — 将 smg-group-1.local 的流量重定向到 aggregate cluster
    # ---------------------------------------------------------------
    # 注意：必须使用 applyTo: HTTP_ROUTE + MERGE 只改 cluster 字段，
    # 不能使用 ROUTE_CONFIGURATION + MERGE routes，因为 routes 是 repeated
    # 字段，MERGE 会追加而非覆盖，导致原始 route 仍然生效。
    - applyTo: HTTP_ROUTE
      match:
        routeConfiguration:
          vhost:
            name: smg-group-1.local:30002
        route:
          action: ROUTE
      patch:
        operation: MERGE
        value:
          route:
            cluster: smg-group-1-aggregate
```

### 4.3 Higress 层轮询配置

VirtualService 轮询 3 个 group，每个 group 引用独立的 host：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: smg-router-vs
  namespace: higress-system
spec:
  hosts:
    - kimi-router-ha-cluster.local
  gateways:
    - kimi-router-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            host: smg-group-1.local
            port:
              number: 30002
          weight: 33
        - destination:
            host: smg-group-2.local
            port:
              number: 30002
          weight: 33
        - destination:
            host: smg-group-3.local
            port:
              number: 30002
          weight: 34
      retries:
        attempts: 2
        perTryTimeout: 30s
        retryOn: gateway-error,connect-failure,refused-stream
```

对应的 ServiceEntry（用于让 Istio 生成基础 cluster，再由 EnvoyFilter patch）：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: smg-group-1
  namespace: higress-system
spec:
  hosts:
    - smg-group-1.local
  location: MESH_INTERNAL
  ports:
    - name: http
      number: 30002
      protocol: HTTP
  resolution: DNS
  endpoints:
    - address: smg-group-1-0.smg-group-1-headless.default.svc.cluster.local
    - address: smg-group-1-1.smg-group-1-headless.default.svc.cluster.local
    - address: smg-group-1-2.smg-group-1-headless.default.svc.cluster.local
```

**注意**：ServiceEntry 中的 endpoints 仅用于让 Istio 识别 `smg-group-1.local` 这个 host 并生成基础 cluster。实际的路由由 EnvoyFilter 中的 aggregate cluster 接管。

### 4.4 故障切换时序

```
t=0s      SMG-1-主崩溃
          ┌─────┐
          │ 主  │ ✗ /health 返回 503
          └──┬──┘
             │
          Envoy active health check
          interval: 2s, unhealthy_threshold: 1
             │
t=2s      Envoy 标记 primary cluster UNHEALTHY
          Aggregate Cluster 降级到 backup-1
          ┌─────┐
          │ 备A │ ← 100% 流量自动切换
          └──┬──┘
             │
          客户端完全无感知（非流式请求自动重试）

t=4s      如果备A 也崩溃
          Envoy 标记 backup-1 UNHEALTHY
          Aggregate Cluster 降级到 backup-2
          ┌─────┐
          │ 备B │ ← 100% 流量自动切换
          └──┬──┘
```

**切换延迟**：~2s（active health check interval）

---

## 五、备选方案对比

### 5.1 方案总览

| 方案 | 名称 | 切换延迟 | 多备支持 | 核心机制 | 复杂度 |
|------|------|---------|---------|---------|--------|
| V1 | K8s Service + Readiness | ~6-9s | ❌ | K8s readiness probe | 低 |
| V2-A | EnvoyFilter Health Check + K8s Service | ~2-4s | ❌ | Envoy active HC + 单端点 Service | 中 |
| **V2-B** | **Envoy Aggregate Cluster（推荐）** | **~2-4s** | **✅** | **Envoy aggregate + active HC** | **中** |
| V2-C | Custom K8s Controller | ~2-3s | ✅ | 自研 controller + EndpointSlice | 中 |

### 5.2 方案 V2-A：轻量级改进（1主1备）

如果不希望使用 aggregate cluster 的复杂度，可以在 V1 的基础上做轻量级改进：

1. 保留 K8s Service-per-group 架构（1主1备）
2. 通过 EnvoyFilter 给每个 Service 的 cluster 添加 active health check
3. 缩短 K8s readiness probe 到 `periodSeconds: 1, failureThreshold: 1`
4. 利用 Envoy 的 outlierDetection 加速不健康端点的驱逐

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: smg-group-1-health-check
  namespace: higress-system
spec:
  configPatches:
  - applyTo: CLUSTER
    match:
      context: GATEWAY
      cluster:
        service: smg-group-1.local
    patch:
      operation: MERGE
      value:
        health_checks:
        - timeout: 3s
          interval: 2s
          unhealthy_threshold: 1
          healthy_threshold: 1
          http_health_check:
            path: /health
```

**优势**：配置简单，无 aggregate cluster 的复杂度
**劣势**：仍依赖 K8s Service 和 readiness probe，不能彻底消除 L4 延迟；不支持多备

### 5.3 方案 V2-C：Custom K8s Controller

如果不希望使用 EnvoyFilter，可以写一个轻量的 K8s controller：

```
┌─────────────────┐
│ Custom Controller│
│ (HA deployment)  │
└────────┬────────┘
         │ HTTP GET /health every 1s
    ┌────┴────┬────────┐
    ▼         ▼        ▼
[SMG-1-主] [SMG-1-备A] [SMG-1-备B]
    │         │        │
    └────┬────┘        │
         ▼             │
   修改 EndpointSlice   │
   (主 healthy → 加入    │
    主，移除所有备)      │
   (主 unhealthy → 移除  │
    主，加入最高优先级备) │
```

**核心逻辑**：
- Controller 持续 HTTP 探测所有 SMG 实例
- 维护一个优先级队列：主 > 备A > 备B
- 动态修改 EndpointSlice，确保 EndpointSlice 中只有一个（最优先的）健康端点
- Higress 通过 Service 发现 EndpointSlice，自动将流量路由到唯一端点

**优势**：
- 纯 K8s 原生 API，无 EnvoyFilter 兼容性风险
- 完全可控的主备逻辑
- 支持 1主N备

**劣势**：
- 需要开发、部署、维护一个 controller
- Controller 本身需要高可用部署
- 增加了系统组件数量

---

## 六、SMG 应用层改动

### 6.1 V1 vs V2-B 的改动对比

| 改动项 | V1 方案 | V2-B 方案（推荐） |
|--------|---------|------------------|
| 主备角色感知 | 需要（--role primary/backup） | **不需要** |
| peer 健康探测 | 需要（备探测主） | **不需要** |
| readiness 控制 | 需要（备返回 503） | **不需要** |
| 多备支持 | 困难（脑裂风险） | **天然支持** |
| `/health` 端点 | 已有 | **已有，建议增强** |
| cache_aware 树 | 每组只有一个活跃实例维护 | **每个实例独立维护** |
| mesh 同步 | Phase 2 需要 | **可选** |
| K8s 部署方式 | Deployment | **StatefulSet** |
| K8s Service | 普通 Service | **Headless Service** |

**核心变化**：V2-B 方案将主备决策从 SMG 应用层**下放到 Envoy 基础设施层**，SMG 实例不再需要关心自己是主还是备。

### 6.2 SMG `/health` 端点增强建议

SMG 现有的 `/health` 端点返回 200 OK。建议增强为根据 worker 健康状态返回 503：

```rust
async fn health(State(state): State<Arc<AppState>>) -> Response {
    let workers = state.context.worker_registry.get_all();
    let has_healthy = workers.iter().any(|w| w.is_healthy());
    
    match &state.context.router_config.mode {
        RoutingMode::PrefillDecode { .. } => {
            let has_prefill = workers.iter().any(|w| matches!(w.worker_type(), WorkerType::Prefill { .. }) && w.is_healthy());
            let has_decode = workers.iter().any(|w| matches!(w.worker_type(), WorkerType::Decode) && w.is_healthy());
            if has_prefill && has_decode {
                (StatusCode::OK, "Healthy").into_response()
            } else {
                (StatusCode::SERVICE_UNAVAILABLE, "Missing prefill or decode workers").into_response()
            }
        }
        _ => {
            if has_healthy {
                (StatusCode::OK, "Healthy").into_response()
            } else {
                (StatusCode::SERVICE_UNAVAILABLE, "No healthy workers").into_response()
            }
        }
    }
}
```

这样 Envoy 的 active health check 能更精确地反映 SMG 的服务能力。

### 6.3 cache_aware 树管理

**V2-B 方案的关键设计**：
- 每个 SMG 实例（主、备A、备B）都独立维护自己的 cache_aware 树
- 但由于 aggregate cluster 保证同一时间只有一个子 cluster 接收流量，树不会分散

**主恢复后的回切问题**：
- 主挂了 → 流量去备A，备A 的 cache_aware 树开始积累历史
- 主恢复 → Envoy active health check 发现主健康
- Aggregate cluster 自动将流量切回 primary
- 但主的 cache_aware 树是空的（或者和备A 不同步）
- 这会导致 cache miss

**解决策略**（三选一）：

**策略A：不回切（推荐）**
- 主恢复后，保持作为备运行（不提升为 primary）
- 除非人工干预或所有备都挂了
- 避免频繁的 cache miss

**策略B：延迟回切**
- 主恢复后，等待一段时间（如 60s）再提升为 primary
- 期间通过 mesh 同步或主动从备拉取 cache_aware 树

**策略C：接受 trade-off**
- 直接回切，接受切换后的 cache miss
- 这与官方文档推荐的"10-20% cache hit 下降"一致
- 最简单，无需额外逻辑

---

## 七、K8s 配置清单

### 7.1 资源清单对比

| K8s 资源 | V1 方案 | V2-A 方案 | **V2-B 方案** | V2-C 方案 |
|---------|---------|-----------|---------------|-----------|
| Service | 3 个 | 3 个 | **0 个** | 3 个 |
| StatefulSet | 0 个 | 0 个 | **3 个** | 3 个 |
| Headless Service | 0 个 | 0 个 | **3 个** | 3 个 |
| ServiceEntry | 1 个 | 1 个 | **3 个** | 0 个 |
| EnvoyFilter | 0 个 | 3 个 | **3 个** | 0 个 |
| VirtualService | 1 个 | 1 个 | **1 个** | 1 个 |
| Ingress | 1 个 | 1 个 | **1 个** | 1 个 |
| Custom Controller | 0 个 | 0 个 | **0 个** | 1 个 |

### 7.2 完整配置示例（V2-B）

见第 4.2 节。每组 SMG 需要一个独立的 EnvoyFilter，3 组共 3 个 EnvoyFilter。

---

## 八、风险与缓解

### 8.1 EnvoyFilter 与 Istio/Higress 版本兼容性

**风险**：EnvoyFilter 是底层注入，Istio/Higress 升级时可能破坏配置。

**缓解**：
- 使用 `ADD` 操作创建新 cluster（而非 `MERGE` 修改自动生成配置），最小化冲突
- aggregate cluster 的 protobuf 类型是 Envoy 原生支持的，跨版本兼容性好
- 在测试环境验证 EnvoyFilter 的兼容性后再上线
- 关注 Higress 社区动态，如果未来推出原生 failover CRD，可平滑迁移

### 8.2 HTTP_ROUTE Patch 稳定性

**风险**：`applyTo: HTTP_ROUTE` 的 patch 依赖 Istio 生成的 route 结构，如果 VirtualService 配置变化导致 route 结构变化，patch 可能匹配不到。

**缓解**：
- 使用 `match.routeConfiguration.vhost.name` + `match.route.action: ROUTE` 精确匹配
- 配置后使用 `istioctl proxy-config routes` 验证 patch 是否生效
- 如果匹配失败，流量会走原始的 Istio 自动生成 cluster（不会导致服务中断，只是主备功能不生效）

### 8.3 Active Health Check 误判

**风险**：短暂的网络抖动或 GC 停顿可能导致 `/health` 返回 503，Envoy 错误地将端点标记为不健康。

**缓解**：
- 适当提高 `unhealthy_threshold`（如 2 次）
- 配置 `healthy_threshold: 2`，确保恢复后需要连续 2 次成功才标记为健康
- 在切换速度和误判率之间权衡（推荐 2s interval + 2 次 threshold = ~4s 切换）

### 8.4 主恢复后的回切抖动

**风险**：aggregate cluster 在主恢复后会自动将流量切回 primary，但 primary 的 cache_aware 树可能为空。

**缓解**：
- **推荐策略C（接受 trade-off）**：直接回切，接受少量 cache miss。这是最简洁的方案，也与官方"10-20% 下降"的推荐一致
- 或者在 EnvoyFilter 中为 backup cluster 配置更高的权重，使主恢复后仍有部分流量去备，直到备也挂了才完全回切

### 8.5 DNS 解析延迟

**风险**：`STRICT_DNS` cluster 需要 DNS 解析 Pod FQDN，可能存在解析延迟或缓存问题。

**缓解**：
- 使用 Headless Service，DNS 直接返回 Pod IP，无额外代理层
- Envoy 的 DNS 解析有内置缓存和刷新机制
- 如果需要，可以配置 `dns_refresh_rate` 和 `dns_lookup_family`

---

## 九、方案选择建议

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 对切换延迟敏感（要求 <5s） | **V2-B** | Envoy active HC + aggregate cluster，~2-4s |
| 需要 1主N备（N>1） | **V2-B 或 V2-C** | 两者都支持多备 |
| 希望最小化 SMG 代码改动 | **V2-B** | 只需标准 `/health`，无需角色感知 |
| 运维团队熟悉 Envoy | **V2-B** | 利用 Envoy 原生能力 |
| 运维团队不熟悉 Envoy，希望 K8s 原生 | **V2-C** | 纯 K8s API，无 EnvoyFilter |
| 对 EnvoyFilter 兼容性有顾虑 | **V2-C 或 V1** | V2-C 无兼容性风险；V1 最简单 |
| 切换延迟要求不高（~10s 可接受），且只需要 1主1备 | **V1** | 最简单，标准 K8s 机制 |
| 需要快速上线，时间紧迫 | **V1 或 V2-A** | V1 最成熟；V2-A 只需加一个 EnvoyFilter |

### 9.1 渐进式演进路径

如果希望逐步演进，可以采用以下路径：

```
Phase 0: V1（当前）
  └── K8s Service + Readiness Probe
      └── 切换延迟 ~6-9s，仅支持 1主1备

Phase 1: V2-A（轻量级改进）
  └── V1 + EnvoyFilter active health check
      └── 切换延迟 ~3-5s，仍仅支持 1主1备
      └── SMG 无需改动

Phase 2: V2-B（完整方案）
  └── Envoy Aggregate Cluster + StatefulSet
      └── 切换延迟 ~2-4s，支持 1主N备
      └── SMG 无需角色感知
      └── 需要更换部署方式为 StatefulSet

Phase 3: V2-B + Mesh Sync（可选）
  └── 在 V2-B 基础上，启用 SMG 的 mesh tree 同步
      └── 主备之间的 cache_aware 树可继承
      └── 消除主恢复后的 cache miss
```

---

## 十、待验证问题

1. **EnvoyFilter ADD aggregate cluster 在 Higress 中的实际效果**：在测试环境部署并验证 failover 行为
2. **HTTP_ROUTE patch 的匹配准确性**：确认 `match.routeConfiguration.vhost.name` 在不同 Istio/Higress 版本下是否能正确匹配到目标 route
3. **Active health check 对 streaming 请求的影响**：确认 health check 连接是否会影响 SSE/gRPC streaming 的长连接池
4. **Aggregate cluster 的多备行为**：1主2备场景下，确认只有一个实例接收流量
5. **Headless Service DNS 解析稳定性**：验证 StatefulSet Pod FQDN 在 Envoy STRICT_DNS 模式下的解析表现
6. **Higress 原生 failover CRD 进展**：跟踪 Higress 社区是否计划推出类似 `FailoverPolicy` 的高层 CRD

---

## 十一、下一步行动

1. **在测试环境验证 V2-B 的 EnvoyFilter 配置**：确认 aggregate cluster + active health check 的 failover 行为符合预期
2. **对比 V1、V2-A、V2-B 的切换延迟**：模拟 SMG 崩溃，测量各方案的 failover 时间
3. **验证多备扩展性**：在 1主2备场景下，确认流量始终只去一个实例
4. **评估 V2-C 的开发成本**：如果 V2-B 的 EnvoyFilter 在测试环境遇到兼容性问题，评估自研 controller 的开发工作量
5. **SMG `/health` 端点增强**：实现基于 worker 健康状态的 503 返回（见 6.2 节）

---

## 附录：参考文档与源码

### A.1 Higress 官方资源

- **higress如何支持健康检查和路由权重、会话保持 · Issue #287**
  - 链接：https://github.com/alibaba/higress/issues/287
  - 说明：Higress maintainer 确认开源版需通过 EnvoyFilter 实现 active health check，并提供可工作的配置示例

- **基于 AI 网关的 LLM 高可用最佳实践**
  - 链接：https://higress.ai/en/blog/基于-ai-网关的-llm-高可用最佳实践
  - 说明：Higress 商业版 AI Gateway 高可用能力对比表（active/passive health check、failover、SSE first packet timeout）

- **Best Practices for High Availability of LLM Based on AI Gateway**
  - 链接：https://www.alibabacloud.com/blog/best-practices-for-high-availability-of-llm-based-on-ai-gateway_602522
  - 说明：阿里云官方博客，同上内容英文版

- **Azure OpenAI api 端点设置 failover 配置的话健康检查会报 401 · Issue #1816**
  - 链接：https://github.com/alibaba/higress/issues/1816
  - 说明：Higress AI Gateway failover 的实际使用案例和问题排查

- **ai-load-balancer基于vllm指标负载均衡 proxywasm.GetUpstreamHosts() · Discussion #3546**
  - 链接：https://github.com/higress-group/higress/discussions/3546
  - 说明：Higress WASM 插件中 `GetUpstreamHosts()` 的使用和 Envoy cluster 配置示例

### A.2 Envoy 官方文档

- **Health checking — envoy 1.39.0-dev documentation**
  - 链接：https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking
  - 说明：Envoy 官方 health checking 架构文档，涵盖 HTTP/gRPC/L4/Redis/Thrift 五种探测方式、per-cluster-member config、passive health check、degraded health 等

- **Aggregate Cluster — envoy 1.39.0-dev documentation**
  - 链接：https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/aggregate_cluster
  - 说明：Envoy 官方 aggregate cluster 文档，说明 failover 行为、circuit breaker 注意事项

- **envoy.extensions.clusters.aggregate.v3.ClusterConfig**
  - 链接：https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/clusters/aggregate/v3/cluster.proto
  - 说明：Aggregate Cluster 的 protobuf 定义，V2-B 方案核心依赖

- **config.core.v3.HealthCheck**
  - 链接：https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/health_check.proto
  - 说明：Envoy health check 配置的完整 protobuf 定义

### A.3 Istio 官方与社区资源

- **Support failover cluster in DestinationRule · Issue #39445**
  - 链接：https://github.com/istio/istio/issues/39445
  - 说明：Istio 社区提案：在 DestinationRule 中引入 `failover` 字段，底层使用 Envoy Aggregate Cluster。截至 2026-06 仍为 open 状态

- **EnvoyFilter cannot guarantee idempotency for route configuration · Issue #38553**
  - 链接：https://github.com/istio/istio/issues/38553
  - 说明：EnvoyFilter 的 idempotency 问题说明，V2-B 方案选择 `ADD` 而非 `MERGE` 的参考依据

- **EnvoyFilter ext_authz googlegrpc disable routing for specific paths · Issue #48507**
  - 链接：https://github.com/istio/istio/issues/48507
  - 说明：EnvoyFilter `applyTo: HTTP_FILTER` patch 的用法参考

### A.4 技术博客与教程

- **How to Configure Envoy Proxy Health Checking**
  - 链接：https://oneuptime.com/blog/post/2026-02-24-configure-envoy-proxy-health-checking/view
  - 说明：Istio 中 outlierDetection（被动）和 EnvoyFilter active health check（主动）的配置教程

- **How to Add Custom Clusters with EnvoyFilter**
  - 链接：https://oneuptime.com/blog/post/2026-02-24-how-to-add-custom-clusters-with-envoyfilter/view
  - 说明：EnvoyFilter `applyTo: CLUSTER` `operation: ADD` 创建自定义 cluster 并配置 health check 的完整示例

- **How to Set Up Priority-Based Load Balancing in Istio**
  - 链接：https://oneuptime.com/blog/post/2026-02-24-how-to-set-up-priority-based-load-balancing-in-istio/view
  - 说明：Istio locality-based priority 的原理和配置，本文调研后认为不适用于单集群主备场景

- **How to Set Up Failover Load Balancing Across Regions in Istio**
  - 链接：https://oneuptime.com/blog/post/2026-02-24-how-to-set-up-failover-load-balancing-across-regions-in-istio/view
  - 说明：Istio 跨 region failover 配置教程

- **How to Handle Cluster Failover in Multi-Cluster Istio**
  - 链接：https://oneuptime.com/blog/post/2026-02-24-how-to-handle-cluster-failover-in-multi-cluster-istio/view
  - 说明：多集群 Istio failover 的测试和监控方法

- **云原生应用负载均衡系列 (2): 入口流量分发、容错与高可用调度**
  - 链接：https://segmentfault.com/a/1190000040279285
  - 说明：详细介绍 Istio 地域感知负载均衡、priority 计算、`overprovisioning factor`（默认 1.4）的算法

- **Configuring Locality failover with Istio**
  - 链接：https://deckhouse.io/products/kubernetes-platform/documentation/v1/user/network/locality_failover_istio.html
  - 说明：Deckhouse 平台的 Istio locality failover 配置指南

- **Priority based Multi-cluster Traffic routing and Failover**
  - 链接：https://docs.tetrate.io/service-bridge/howto/gateway/multi-cluster-traffic-failover-priority
  - 说明：Tetrate Service Bridge 的多集群优先级路由

- **Istio Multi-master Multi-network Locality Failover Woes**
  - 链接：https://serverfault.com/questions/1058227/istio-multi-master-multi-network-locality-failover-woes
  - 说明：实际踩坑：locality failover 中 priority 未正确分配的排查过程

- **不懂envoyfilter也敢说精通istio系列-cluster**
  - 链接：https://www.modb.pro/db/131908
  - 说明：EnvoyFilter `applyTo: CLUSTER` 的完整字段说明，含 `health_checks` 配置示例

- **不懂envoyfilter也敢说精通istio系列-http filter**
  - 链接：https://www.modb.pro/db/152538
  - 说明：EnvoyFilter `applyTo: HTTP_FILTER` 的用法，含 health_check filter 示例

- **不懂envoyfilter也敢说精通istio系列-accesslog**
  - 链接：https://www.modb.pro/db/139099
  - 说明：EnvoyFilter `applyTo: NETWORK_FILTER` access log 配置

- **envoyfilter之cluster**
  - 链接：https://www.gl.sh.cn/2022/04/11/envoyfilter_zhi_cluster.html
  - 说明：EnvoyFilter cluster 配置的中文详解，含 TCP/HTTP health check、socket_options 等

- **搞定Istio主机头重写：EnvoyFilter配置实战指南**
  - 链接：https://blog.csdn.net/gitblog_00958/article/details/151456912
  - 说明：EnvoyFilter 配置实践，含 `priority` 字段和版本兼容性说明

- **Kubernetes上的Service Mesh实践：用EnvoyFilter扩展Istio**
  - 链接：https://cloudnative.jimmysong.io/blog/using-envoyfilter-extend-istio/
  - 说明：Envoy 架构和 filter chain 原理说明

### A.5 SMG 源码引用

以下源码来自本地克隆的 `sglang/sgl-model-gateway/` 仓库：

- **`sglang/sgl-model-gateway/src/policies/cache_aware.rs`**
  - Cache-aware 路由策略，维护 `(pool, model)` 维度的 radix tree；含 `sync_tree_operation`（发送端）和 `apply_remote_tree_operation`（接收端，未完全 wired）

- **`sglang/sgl-model-gateway/src/server.rs`**
  - HTTP server，`/health` 和 `/ready` 端点定义；V1 方案计划扩展 readiness 以支持主备角色判断

- **`sglang/sgl-model-gateway/src/service_discovery.rs`**
  - K8s Pod 服务发现，支持通过 `router_selector` 和 `router_mesh_port_annotation` 自动发现 peer SMG routers

- **`sglang/sgl-model-gateway/src/routers/http/pd_router.rs`**
  - PD 路由器的 dual dispatch 实现，`tokio::join!(prefill_request, decode_request)`；含 `inject_bootstrap_into_value` 注入 bootstrap 元数据

### A.6 前期设计文档

- **`active_standby_ha_design.draft.md`**
  - V1 方案设计文档（K8s Service + Readiness Probe 主备切换）

- **`ha_impact_analysis_serverless.md`**
  - V1 方案的 server-side 透明 failover 影响分析

- **`pd_disaggregated_architecture.md`**
  - 多集群分片 PD 架构文档

- **`k8s-higress-smg-configs/serviceentry.yaml`**
  - 现有 Higress 配置：ServiceEntry 注册 3 个 SMG replicas

- **`k8s-higress-smg-configs/`**
  - 现有完整 Higress 配置目录（含 Ingress、Gateway、VirtualService、DestinationRule 等）

### A.7 关键调研记录

- **Higress 开源版是否原生支持 active health check**
  - 结论：❌ 不支持高层 CRD，需通过 EnvoyFilter 配置

- **Istio locality-based failover 是否适合单集群主备**
  - 结论：❌ 依赖 K8s 节点 region/zone 标签，不适合

- **EnvoyFilter `MERGE` load_assignment 是否安全**
  - 结论：⚠️ 对 repeated 字段是追加语义，可能与 ServiceEntry 冲突

- **EnvoyFilter `ADD` aggregate cluster 是否可行**
  - 结论：✅ 完全独立于 Istio 自动生成配置，无冲突风险

- **Istio DestinationRule `failover` 字段进展**
  - 结论：🔄 Issue #39445 仍 open，尚未落地

- **Envoy aggregate cluster 是否支持多级 failover**
  - 结论：✅ 原生支持，按顺序尝试子 cluster

- **Higress 商业版 failover 能力**
  - 结论：✅ 支持 active+passive HC、SSE first packet timeout、fallback
