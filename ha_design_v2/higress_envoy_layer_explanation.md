# Higress/Envoy 体系层级关系详解

> 面向 SMG Active-Standby HA 场景，逐层解释各层的作用、解决的问题、以及配置项的归属。

---

## 一、总体层次图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         第 5 层：客户端请求                                   │
│     curl https://api.example.com/v1/chat/completions                        │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────────────┐
│                    第 4 层：Higress 控制面（K8s CRD）                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │  Ingress   │  │  Gateway   │  │ VirtualSvc │  │ DestinRule │  ...       │
│  │ (入口定义)  │  │ (监听配置)  │  │ (路由规则)  │  │ (流量策略)  │            │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                            │
│  │ServiceEntry│  │EnvoyFilter │  │  Service   │  (K8s 原生)               │
│  │(服务注册)   │  │(底层注入)   │  │ (服务发现)  │                            │
│  └────────────┘  └────────────┘  └────────────┘                            │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │ Istiod 翻译/推送
┌───────────────────────────────▼─────────────────────────────────────────────┐
│                   第 3 层：Envoy 运行时配置（数据面）                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Listener (监听器)                              │  │
│  │   监听 0.0.0.0:8080                                                   │  │
│  │   └── Filter Chain                                                    │  │
│  │       └── HTTP Connection Manager (HCM)                               │  │
│  │           ├── HTTP Filter Chain                                       │  │
│  │           │   ├── auth filter                                         │  │
│  │           │   ├── rate limit filter                                   │  │
│  │           │   └── Router Filter  ←── 核心！路由决策在这里              │  │
│  │           │                                                          │  │
│  │           └── Route Table (RDS)                                      │  │
│  │               ├── Route: match /v1/chat/*                            │  │
│  │               │   ├── retry policy                                   │  │
│  │               │   ├── timeout                                        │  │
│  │               │   └── cluster: smg-group-1-aggregate                  │  │
│  │               └── Route: match /v1/models/*                          │  │
│  │                   └── cluster: model-list-cluster                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      Cluster (上游集群)                                │  │
│  │   ┌─────────────────────┐  ┌─────────────────────┐                   │  │
│  │   │ smg-group-1-agg     │  │ smg-group-2-agg     │  (aggregate)      │  │
│  │   │ (aggregate cluster) │  │ (aggregate cluster) │                   │  │
│  │   │ ├── primary         │  │ ├── primary         │                   │  │
│  │   │ ├── backup-1        │  │ └── backup-1        │                   │  │
│  │   │ └── backup-2        │  │                     │                   │  │
│  │   └─────────────────────┘  └─────────────────────┘                   │  │
│  │   ┌─────────────────────┐                                            │  │
│  │   │ smg-group-1-primary │  (普通 cluster)                            │  │
│  │   │ type: STRICT_DNS     │                                            │  │
│  │   │ health_checks: [...] │                                            │  │
│  │   │ circuit_breakers: [...]                                          │  │
│  │   │ load_assignment:     │                                            │  │
│  │   │   endpoints:         │                                            │  │
│  │   │     - address: smg-1-0.fqdn                                      │  │
│  │   └─────────────────────┘                                            │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────────────┐
│                   第 2 层：Envoy 网络传输层                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     Transport Socket (传输层)                          │  │
│  │   - TCP 连接建立 / TLS 握手 / mTLS 认证                                 │  │
│  │   - TCP keepalive                                                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    Upstream Connection Pool                            │  │
│  │   - HTTP/1.1 连接池 / HTTP/2 多路复用                                  │  │
│  │   - 最大连接数 / 最大并发请求数 / 最大等待请求数                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │ DNS 解析 / 直接连接
┌───────────────────────────────▼─────────────────────────────────────────────┐
│                   第 1 层：上游服务端点（SMG Pod）                             │
│                                                                              │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │
│   │ SMG-1-主      │  │ SMG-2-主      │  │ SMG-3-主      │                     │
│   │ (30002)       │  │ (30002)       │  │ (30002)       │                     │
│   │ cache_aware树 │  │ cache_aware树 │  │ cache_aware树 │                     │
│   └──────────────┘  └──────────────┘  └──────────────┘                     │
│   ┌──────────────┐  ┌──────────────┐                                        │
│   │ SMG-1-备A     │  │ SMG-1-备B     │  (同一 group，不接收流量)            │
│   │ (30002)       │  │ (30002)       │                                        │
│   │ cache_aware树 │  │ cache_aware树 │                                        │
│   └──────────────┘  └──────────────┘                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、各层详解

### 第 1 层：上游服务端点（SMG Pod）

**作用**：实际处理请求的 SMG 实例。

**解决的问题**：模型推理请求的实际执行。

**关键状态**：
- 每个实例独立维护 `cache_aware` 的 radix tree
- `/health` 端点返回 200（健康）或 503（不健康）
- 同一 group 内的主备实例**互不感知**，各自独立运行

**V2-B 方案的变化**：
- 部署方式从 `Deployment` 改为 `StatefulSet` + `Headless Service`
- 不再需要 `readiness probe` 控制角色（由 Envoy 的 active health check 替代）
- 不再需要 `--role=primary/backup` 启动参数

---

### 第 2 层：Envoy 网络传输层

**作用**：管理 Envoy 到上游 Pod 的物理/逻辑连接。

**解决的问题**：
- TCP 连接建立和复用
- TLS/mTLS 加密传输
- 连接数限制和过载保护

**关键概念**：

| 概念 | 说明 | 配置位置 |
|------|------|---------|
| `connect_timeout` | 建立 TCP 连接的超时时间 | EnvoyFilter cluster |
| `transport_socket` | TLS 配置（证书、SNI、ALPN） | EnvoyFilter cluster / DestinationRule |
| `upstream_connection_options` | TCP keepalive 配置 | EnvoyFilter cluster |
| `circuit_breakers` | 连接池熔断（最大连接数、最大并发请求数） | EnvoyFilter cluster / DestinationRule |

**注意**：在 V2-B 方案中，DestinationRule 的 `connectionPool` 配置不会自动生效（因为流量被重定向到了自定义 cluster），如需限制连接数，需在 EnvoyFilter 中显式配置 `circuit_breakers`。

---

### 第 3 层：Envoy 运行时配置（数据面）

这是整个体系的核心，分为 **Listener** 和 **Cluster** 两大子系统。

#### 3.1 Listener（监听器 + 路由）

**作用**：接收客户端请求，匹配路由规则，决定请求发往哪个 cluster。

**解决的问题**：
- "这个请求应该去哪里？"
- 基于 URL、Host、Header 等做路由匹配
- 配置重试、超时、故障注入

**关键组件**：

| 组件 | 作用 | 类比 |
|------|------|------|
| **Listener** | 监听端口（如 8080），接收连接 | Nginx 的 `listen` 指令 |
| **Filter Chain** | 连接上的过滤器链（TLS、HTTP、TCP） | Nginx 的 `server` 块 |
| **HCM** (HTTP Connection Manager) | 将 TCP 连接升级为 HTTP 处理 | Nginx 的 `location` 处理 |
| **Router Filter** | 根据 Route Table 决定 upstream cluster | Nginx 的 `proxy_pass` |
| **Route Table (RDS)** | 路由表：URL pattern → cluster | Nginx 的 `location` 匹配规则 |

**V2-B 方案的关键操作**：
- **ROUTE_CONFIGURATION patch**：修改 Route Table，将 `smg-group-1.local` 的 upstream cluster 从默认的 `outbound|30002||smg-group-1.local` 改为 `smg-group-1-aggregate`

```yaml
# EnvoyFilter ROUTE_CONFIGURATION patch 的本质：
# 修改 Envoy 内部的路由表
route_table:
  - match: { prefix: / }
    # 原来是：
    # cluster: outbound|30002||smg-group-1.local
    # 被 patch 为：
    cluster: smg-group-1-aggregate
    retry_policy: { attempts: 2, ... }   # ← 来自 VirtualService
    timeout: 0s                         # ← 来自 VirtualService
```

#### 3.2 Cluster（上游集群）

**作用**：定义一组上游端点，以及访问这些端点的策略。

**解决的问题**：
- "如何连接到目标服务？"
- 服务发现（端点列表从哪里获取）
- 健康检查（哪些端点可用）
- 负载均衡（在健康端点间如何分配）
- 熔断保护（连接数限制）

**关键概念**：

| 概念 | 说明 | 配置位置 |
|------|------|---------|
| **Cluster Type** | `STRICT_DNS` / `LOGICAL_DNS` / `EDS` / `STATIC` | EnvoyFilter / ServiceEntry |
| **Load Assignment** | 端点列表（IP 或 FQDN） | EnvoyFilter / ServiceEntry |
| **Health Checks** | 主动健康检查配置 | EnvoyFilter (V2-B) / DestinationRule |
| **Outlier Detection** | 被动熔断（基于错误率驱逐） | DestinationRule |
| **Circuit Breakers** | 连接数/并发数限制 | DestinationRule / EnvoyFilter |
| **Load Balancer** | 负载均衡策略（RR / least_request / ring_hash） | DestinationRule / EnvoyFilter |

**Aggregate Cluster 的特殊性**：

```
smg-group-1-aggregate (CLUSTER_PROVIDED)
  │
  ├─ smg-group-1-primary (ROUND_ROBIN)
  │      ├── active health check: /health, 2s
  │      └── endpoints: smg-1-0.fqdn
  │
  ├─ smg-group-1-backup-1 (ROUND_ROBIN)
  │      ├── active health check: /health, 2s
  │      └── endpoints: smg-1-1.fqdn
  │
  └─ smg-group-1-backup-2 (ROUND_ROBIN)
         ├── active health check: /health, 2s
         └── endpoints: smg-1-2.fqdn
```

**Aggregate Cluster 的行为**：
1. 按顺序尝试子 cluster：`primary` → `backup-1` → `backup-2`
2. 每个子 cluster 独立维护自己的 active health check
3. 当 `primary` 的所有端点都不健康时，降级到 `backup-1`
4. 当 `backup-1` 也不健康时，降级到 `backup-2`
5. 所有子 cluster 都不健康时，aggregate cluster 返回 503

---

### 第 4 层：Higress 控制面（K8s CRD）

这是运维人员日常打交道的层面，通过 YAML 声明式配置。

#### 4.1 各 CRD 的作用域

```
┌─────────────────────────────────────────────────────────────┐
│                    CRD 作用域对比                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │  Ingress    │───→│  Gateway    │    │             │    │
│  │  定义入口    │    │  定义监听器  │    │             │    │
│  └─────────────┘    └──────┬──────┘    │             │    │
│                            │            │             │    │
│  ┌─────────────────────────▼──────────┐│             │    │
│  │        VirtualService               ││             │    │
│  │  ┌─────────────┐  ┌─────────────┐  ││             │    │
│  │  │  路由匹配    │  │  重试/超时   │  ││             │    │
│  │  │  URL/Header │  │  故障注入   │  ││             │    │
│  │  │  权重分配   │  │  重写/重定向 │  ││             │    │
│  │  └─────────────┘  └─────────────┘  ││             │    │
│  │           │                         ││             │    │
│  │           ▼                         ││             │    │
│  │  destination: smg-group-1.local     ││             │    │
│  └─────────────────────────────────────┘│             │    │
│                            │            │             │    │
│  ┌─────────────────────────▼──────────┐│             │    │
│  │        DestinationRule              ││             │    │
│  │  ┌─────────────┐  ┌─────────────┐  ││             │    │
│  │  │  负载均衡    │  │  连接池限制  │  ││             │    │
│  │  │  子集划分   │  │  熔断配置   │  ││             │    │
│  │  │  TLS/mTLS   │  │  outlier    │  ││             │    │
│  │  └─────────────┘  └─────────────┘  ││             │    │
│  └─────────────────────────────────────┘│             │    │
│                            │            │             │    │
│  ┌─────────────────────────▼──────────┐│             │    │
│  │        ServiceEntry                 ││             │    │
│  │  ┌─────────────┐  ┌─────────────┐  ││             │    │
│  │  │  端点注册   │  │  服务发现   │  ││             │    │
│  │  │  FQDN/IP   │  │  DNS/STATIC │  ││             │    │
│  │  └─────────────┘  └─────────────┘  ││             │    │
│  └─────────────────────────────────────┘│             │    │
│                            │            │             │    │
│  ┌─────────────────────────▼──────────┐│             │    │
│  │        EnvoyFilter                  ││             │    │
│  │  ┌─────────────┐  ┌─────────────┐  ││             │    │
│  │  │  直接注入    │  │  绕过 Istio │  ││             │    │
│  │  │  Listener   │  │  自动生成   │  ││             │    │
│  │  │  Cluster    │  │  的配置     │  ││             │    │
│  │  │  Route      │  │             │  ││             │    │
│  │  └─────────────┘  └─────────────┘  ││             │    │
│  └─────────────────────────────────────┘│             │    │
│                                         │             │    │
│  ┌─────────────────────────────────────┐│             │    │
│  │        K8s Service / EndpointSlice  ││             │    │
│  │  ┌─────────────┐  ┌─────────────┐  ││             │    │
│  │  │  K8s 原生    │  │  kube-proxy │  ││             │    │
│  │  │  服务发现   │  │  iptables   │  ││             │    │
│  │  └─────────────┘  └─────────────┘  ││             │    │
│  └─────────────────────────────────────┘│             │    │
│                                         │             │    │
└─────────────────────────────────────────┘             │    │
                                                        │    │
        ↑ 这些 CRD 最终都被 Istiod 翻译成 Envoy 配置    │    │
        ↓ 推送给 Higress Gateway Pod 的 Envoy sidecar   │    │
                                                        │    │
┌───────────────────────────────────────────────────────┘    │
│              Envoy 数据面（第 3 层）                          │
│                                                             │
│  Listener ← VirtualService + Ingress + Gateway              │
│  Cluster  ← DestinationRule + ServiceEntry + Service        │
│  Health   ← DestinationRule (outlier) + EnvoyFilter (active)│
│  Custom   ← EnvoyFilter (aggregate cluster, route patch)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 4.2 各 CRD 在 V2-B 方案中的具体作用

| CRD | V2-B 方案中的作用 | 是否必需 | 替代/补充 |
|-----|------------------|---------|----------|
| **Ingress** | 定义外部访问入口，将域名映射到 Gateway | ✅ 是 | 无 |
| **Gateway** | 定义 Higress 监听端口（8080/8443）和协议（HTTP/HTTPS） | ✅ 是 | 无 |
| **VirtualService** | 定义 `/v1/chat/*` 路由到 3 个 group，配置重试（2 次）和超时 | ✅ 是 | 无 |
| **DestinationRule** | **V2-B 中部分失效**（流量被重定向到自定义 cluster） | ⚠️ 可选 | 连接池/熔断需在 EnvoyFilter 中补配 |
| **ServiceEntry** | 让 Istio 识别 `smg-group-*.local` 并生成基础 cluster | ✅ 是 | 提供 host → Envoy cluster 的映射 |
| **EnvoyFilter** | **核心！** 创建 aggregate cluster + active health check + route patch | ✅ 是 | 替代了 DestinationRule 的部分能力 |
| **K8s Service** | V2-B 中**不再用于流量控制**，Headless Service 仅提供 Pod FQDN | ✅ 是（Headless） | 无 |

#### 4.3 为什么需要 EnvoyFilter？

**Istio 的局限**：
- Istio 没有高层 CRD 来配置 "aggregate cluster"（issue #39445 仍在 open）
- Istio 的 active health check 只能通过 EnvoyFilter 暴露
- Istio 的 locality-based failover 依赖 K8s 节点标签，不适合单集群主备

**EnvoyFilter 的本质**：
- 直接操作 Envoy 运行时配置（Listener / Cluster / Route / Filter）
- 绕过 Istio 的自动生成逻辑
- 使用 Envoy 的原生 protobuf 配置

**V2-B 方案中 EnvoyFilter 的三项工作**：

```
EnvoyFilter patch 1: ADD cluster (primary)
  → 创建 smg-group-1-primary cluster
  → 配置 active health check (/health, 2s)
  → 配置 endpoints (smg-1-0.fqdn)

EnvoyFilter patch 2: ADD cluster (aggregate)
  → 创建 smg-group-1-aggregate cluster
  → 配置 cluster_type = envoy.clusters.aggregate
  → 子 cluster 列表: [primary, backup-1, backup-2]

EnvoyFilter patch 3: MERGE route
  → 修改 Route Table
  → 将 smg-group-1.local 的 upstream 从默认 cluster 改为 aggregate cluster
```

---

## 三、V2-B 方案的完整数据流

### 3.1 正常请求

```
客户端请求: POST /v1/chat/completions
    │
    ▼
┌─────────────────────────────────────────┐
│ Higress Gateway (Envoy)                 │
│                                         │
│ 1. Listener:8080 接收请求               │
│    └── Filter Chain (HTTP)              │
│        └── HCM → Router Filter          │
│            └── Route Table 匹配:        │
│                match: prefix /          │
│                cluster: smg-group-1-agg │  ← 33% weight
│                                         │
│ 2. Router 选择 cluster                  │
│    └── smg-group-1-aggregate            │
│        └── 尝试 primary                 │
│            └── HEALTHY ✅               │
│                └── 发送请求到           │
│                    smg-1-0:30002        │
└─────────────────────────────────────────┘
    │
    ▼
SMG-1-主 处理请求，返回响应
```

### 3.2 主挂了，请求走备

```
客户端请求: POST /v1/chat/completions
    │
    ▼
Higress Gateway (Envoy)
    │
    ├── (t=0s) SMG-1-主崩溃
    │
    ├── (t=2s) active health check 检测到主 UNHEALTHY
    │       smg-group-1-primary → UNHEALTHY
    │
    └── (t=2s+) 后续请求
        │
        Router 选择 cluster: smg-group-1-aggregate
        │
        ├── 尝试 primary
        │   └── UNHEALTHY ❌
        │       └── aggregate 自动降级
        │
        ├── 尝试 backup-1
        │   └── HEALTHY ✅
        │       └── 发送请求到
        │           smg-1-1:30002
        │
        └── 请求成功返回
```

### 3.3 配置项在各层的归属

| 能力 | 用户配置的 CRD | Istiod 翻译后的 Envoy 配置 | 生效层级 |
|------|--------------|--------------------------|---------|
| URL 路由匹配 | VirtualService `match` | Route Table `match` | Listener / Route |
| 加权轮询 | VirtualService `weight` | Route `weighted_clusters` | Route |
| 失败重试 | VirtualService `retries` | Route `retry_policy` | Route |
| 请求超时 | VirtualService `timeout` | Route `timeout` | Route |
| 故障注入 | VirtualService `fault` | Route `fault` | Route |
| 负载均衡策略 | DestinationRule `loadBalancer` | Cluster `lb_policy` | Cluster |
| 连接池限制 | DestinationRule `connectionPool` | Cluster `circuit_breakers` | Cluster |
| 被动熔断 | DestinationRule `outlierDetection` | Cluster `outlier_detection` | Cluster |
| 主动健康检查 | **EnvoyFilter** `health_checks` | Cluster `health_checks` | Cluster |
| 端点列表 | ServiceEntry `endpoints` / K8s Service | Cluster `load_assignment` | Cluster |
| mTLS | DestinationRule `trafficPolicy.tls` | Cluster `transport_socket` | Cluster / Transport |
| 自定义 cluster | **EnvoyFilter** `cluster` | 新增 Cluster | Cluster |
| 路由重定向 | **EnvoyFilter** `routeConfiguration` | 修改 Route `cluster` | Route |

---

## 四、常见困惑解答

### Q1: 为什么 VirtualService 配置的 retries 仍然有效，但 DestinationRule 的 connectionPool 可能失效？

**答**：
- **VirtualService** 配置在 **Route 级别**（"请求怎么路由"），与最终选择哪个 cluster 无关，所以始终有效。
- **DestinationRule** 配置在 **Cluster 级别**（"连接这个服务的策略"），绑定到 Istio 自动生成的 `outbound|port||host` cluster。V2-B 方案通过 EnvoyFilter 将流量重定向到了自定义的 `smg-group-1-aggregate` cluster，所以原 cluster 上的 DestinationRule 配置不再生效。

### Q2: ServiceEntry 在 V2-B 方案中还有什么用？

**答**：
- ServiceEntry 告诉 Istio：`smg-group-1.local` 是一个合法的服务 host
- 没有 ServiceEntry，Istio 不会为这个 host 生成任何配置，VirtualService 引用它会报错
- ServiceEntry 中的 `endpoints` 在 V2-B 中仅作为"占位符"，实际端点由 EnvoyFilter 的 `load_assignment` 定义
- ServiceEntry 的 `resolution: DNS` 让 Istio 生成 `STRICT_DNS` 类型的基础 cluster，方便后续 EnvoyFilter patch

### Q3: 为什么不直接在 EnvoyFilter 中定义 Route，而要用 VirtualService？

**答**：
- 可以只用 EnvoyFilter 定义所有东西，但维护成本极高
- VirtualService 提供声明式、高层的流量控制（URL 匹配、重试、超时），这些是日常运维最常改动的
- EnvoyFilter 只用于补充 Istio 不支持的能力（aggregate cluster、active health check）
- 混合使用：VirtualService 负责"常规路由"，EnvoyFilter 负责"高级 failover"

### Q4: EnvoyFilter 的 `MERGE` 和 `ADD` 有什么区别？

**答**：

| 操作 | 语义 | 适用场景 | V2-B 中的使用 |
|------|------|---------|--------------|
| **ADD** | 新增一个资源 | 创建 Istio 不会自动生成的配置 | 创建新的 aggregate cluster、primary cluster、backup cluster |
| **MERGE** | 合并到已有资源 | 修改 Istio 自动生成的配置 | 修改 Route Table，将 cluster 重定向到 aggregate |
| **REPLACE** | 替换整个资源 | 完全覆盖自动生成的配置 | V2-B 中避免使用，风险高 |
| **REMOVE** | 删除资源 | 去掉不需要的自动生成配置 | V2-B 中未使用 |

**MERGE 的陷阱**：对 repeated 字段（如 `endpoints` 数组）是**追加**而非替换。这就是为什么 V2-B 选择 `ADD` 创建全新的 cluster，而不是 `MERGE` 修改 Istio 自动生成的 cluster。

### Q5: 如果 Higress 升级了，EnvoyFilter 会失效吗？

**答**：
- **低风险**：EnvoyFilter 使用的是 Envoy 的原生 protobuf API（如 `envoy.extensions.clusters.aggregate.v3.ClusterConfig`），这些 API 在 Envoy 中非常稳定
- **中风险**：`ROUTE_CONFIGURATION` patch 依赖 route 的内部结构（如 `vhost.name`），如果 Istio 生成 route 的方式变了，patch 可能匹配不到
- **缓解**：使用 `applyTo: VIRTUAL_HOST` + `match.routeConfiguration.vhost.name` 精确匹配，降低误 patch 风险；升级前在测试环境验证

---

## 五、一张图总结：V2-B 方案的"谁管什么"

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           V2-B 方案：各层分工                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  运维人员配置 ←───────────────────────────────────────────────────────────   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  VirtualService: 定义路由、重试、超时、权重                           │   │
│  │  ServiceEntry:   声明服务 host（占位用）                              │   │
│  │  EnvoyFilter:    创建 aggregate cluster + active health check       │   │
│  │  Ingress:        定义外部访问入口                                     │   │
│  │  Gateway:        定义监听端口                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼ Istiod 翻译 + EnvoyFilter patch              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Envoy 运行时配置                                                    │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │ Route Table                                                    │  │   │
│  │  │  match: /v1/chat/*                                            │  │   │
│  │  │  weight: 33% ──→ cluster: smg-group-1-aggregate  ←───────────│  │   │
│  │  │  weight: 33% ──→ cluster: smg-group-2-aggregate               │  │   │
│  │  │  weight: 34% ──→ cluster: smg-group-3-aggregate               │  │   │
│  │  │  retry: attempts=2, timeout=30s  ←──────── 来自 VirtualService │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  │                              │                                      │   │
│  │  ┌───────────────────────────▼───────────────────────────────────┐  │   │
│  │  │ Aggregate Cluster                                                │  │   │
│  │  │  name: smg-group-1-aggregate                                     │  │   │
│  │  │  clusters: [primary, backup-1, backup-2]  ←──── 来自 EnvoyFilter│  │   │
│  │  │  behavior: 按顺序尝试， unhealthy 则降级                          │  │   │
│  │  └───────────────────────────┬───────────────────────────────────┘  │   │
│  │                              │                                      │   │
│  │  ┌───────────────────────────▼───────────────────────────────────┐  │   │
│  │  │ Primary / Backup Cluster                                         │  │   │
│  │  │  name: smg-group-1-primary                                       │  │   │
│  │  │  type: STRICT_DNS                                                │  │   │
│  │  │  health_checks: /health, interval=2s  ←────── 来自 EnvoyFilter  │  │   │
│  │  │  endpoints: smg-group-1-0.fqdn  ←────────── 来自 EnvoyFilter  │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼ DNS 解析 / TCP 连接                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SMG Pods (StatefulSet)                                            │   │
│  │  smg-group-1-0 (主)  ← 接收流量                                     │   │
│  │  smg-group-1-1 (备A) ← 主挂后接收流量                                │   │
│  │  smg-group-1-2 (备B) ← 主+备A 挂后接收流量                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```