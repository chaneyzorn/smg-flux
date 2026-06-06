# SMG HA v5.2 调研方案：L7 网关的分布式一致性动态路由决策

> V5.1 的 implementation_design.md 完成了一份基于 `proxy-wasm SharedData` 的 wasm-go 插件设计，但重新审视后发现其前提假设——SharedData 至少覆盖同一组 Gateway 实例——不成立。本文档探讨在此基础上引入跨实例状态同步的可行路径。

V5.1 的 implementation_design.md 完成了一份基于 `proxy-wasm SharedData` 的 wasm-go 插件设计：通过 `RegisterTickFunc` 做主动健康检查、`SetUpstreamOverrideHost` 做 preemptive endpoint 锁定、双阶段重试处理瞬时故障。这份设计的可行性建立在 Higress 自定义扩展（`ai-proxy`、`ai-load-balancer`）已经生产验证的 API 之上。

但 V5.1 的一个前提假设在重新审视时被发现不成立：它假设 `SharedData` 的共享范围至少覆盖同一组 Gateway 实例。proxy-wasm 规范对 SharedData 的 scope 定义为 implementation-defined，而 Higress/Envoy 的实现中它就是**单个 Envoy 进程级别**的。这意味着：

- 不同 Gateway Pod 的 WASM VM 拥有完全独立的 SharedData 命名空间
- Pod 重建（OOM、版本升级、HPA 扩缩容）后 SharedData 丢失
- `ai-proxy` 在多实例下使用 SharedData 做 VM 内 lease election，但 ai-proxy 的目标是**无状态**的 LLM Provider，每个 Pod 独立选择不同 provider 不影响正确性

SMG 的 Router-worker 架构中，cache tree 是**有状态**的，worker 节点只维护自己收到的请求历史。如果不同 Gateway Pod 把同一 Group 的请求路由到不同 worker，tree 的分散是结构性的。这迫使 V5.2 必须回答一个问题：**在多实例 L7 网关环境下，如何实现跨实例的分布式一致性动态路由决策？**

这不是传统意义上的分布式共识（如 Raft/Paxos），而是一种**基于共享状态视图的功能性共识**——所有网关实例观察相同的后端健康状态，运行相同的确定性选择函数，独立计算出相同的路由决策。V5.2 的设计思路是把"共识"从"谁持有锁"的问题转化为"所有参与者是否观察相同状态"的问题。

本文档将 V5.2 落地为可执行的代码设计：可插拔的 `TopologyBackend` 接口、Consul KV 默认实现、本地健康缓存与聚合逻辑、确定性锁定算法、以及完整的 K8s 部署规格。

---

## 1. V5.1 的根本缺陷

### 1.1 跨实例不一致

Higress Gateway 是多副本部署，每个 Pod 是独立的 Envoy 进程 + WASM VM。`proxy-wasm` 的 `SharedData` 是**进程级别**的，不跨 Pod 共享。

```
客户端请求 Group-1
    |
外层负载均衡（K8s Service / ECMP / 云 LB）
    |
Pod-A (WASM VM-A) ----锁定----> SMG-1.1  (cache tree A)
    |
下一个请求 Group-1
    |
外层负载均衡分配到 Pod-B
    |
Pod-B (WASM VM-B) ----锁定----> SMG-1.2  (cache tree B)
```

无论是 K8s Service 的 iptables/IPVS、云厂商 LoadBalancer、BGP ECMP，还是外层 Envoy/Higress 的负载均衡，本质上都是**把请求独立地分发到多个 Gateway Pod**。具体机制不影响结论：同一个 Group 的 cache tree 被分散到多个 worker，违背了 V5.1 "稳定锁定"的设计目标。

### 1.2 跨生命周期丢失

SharedData 在以下场景丢失：
- Envoy OOM 被杀
- Higress 版本升级（Pod 重建）
- HPA 自动扩缩容
- 节点迁移

丢失后新 VM 重新选择 endpoint，cache tree 再次分散。

### 1.3 为什么 ai-proxy 没有这个问题

ai-proxy 的 failover 目标是**无状态**的 LLM Provider。每个 Pod 独立选择不同的 provider 不影响业务正确性。SMG worker 是**有状态**的，cache tree 在 worker 端，跨 Pod 不一致是致命问题。

---

## 2. V5.2 核心设计思想

### 2.1 设计思路：确定性共识

如果**所有 Pod 维护相同的实时健康视图和全局锁定状态**，并且**使用相同的确定性选择算法**，它们会独立得出**相同的锁定决策**——无需分布式锁。

```
相同的输入（同步的健康视图 + 全局 locked 状态） + 相同的算法 = 相同的输出（锁定的 endpoint）
```

### 2.2 发布订阅的作用

Pub/Sub 负责传播两类状态：
1. **健康探测结果**（原始数据）：每个 Pod 独立聚合，得出健康视图
2. **全局 locked endpoint**（计算结果）：所有 Pod 遵循同一 locked，实现跨 Pod 粘滞

锁定决策由算法本地计算，但**粘滞性通过共享的 locked 状态实现**——当前 locked 健康时直接采用，不重新计算。

### 2.3 与 V5.1 的本质区别

| 维度 | V5.1 | V5.2 |
|------|------|------|
| 状态共享范围 | 单个 Envoy 进程内（SharedData） | 跨所有 Gateway Pod |
| 状态共享机制 | 内存共享 | 发布订阅 |
| 锁定决策 | 本地缓存 + SharedData | 本地缓存（由 pub/sub 同步）+ 后端 shared locked |
| 跨 Pod 一致性 | 无 | 确定性共识 |
| 跨生命周期 | 丢失 | 后端持久化 |
| 分布式锁 | 不需要（单进程内 lease） | **不需要**（确定性算法 + CAS 乐观写） |
| 请求路径外部调用 | 0（SharedData 是 host call） | **0**（纯本地内存） |

---

## 3. 数据模型

### 3.1 后端数据模型

后端存储每个 Group 下每个 endpoint 的**原始探测报告列表**。每个 Pod 独立读取这些报告，本地聚合得出健康状态。

**Report 结构**：
```go
type ProbeReport struct {
    Source  string `json:"source"`   // Pod 名称，通过 Downward API 获取
    Healthy bool   `json:"healthy"`  // true = 200, false = 503/timeout
    Ts      int64  `json:"ts"`       // Unix 毫秒级时间戳
}
```

**Pod 身份获取**：
- 通过 K8s Downward API 将 Pod 名称注入环境变量 `HOSTNAME`
- WASM 插件通过 `proxywasm.GetProperty` 读取环境变量（wasm-go 框架通常暴露此能力）
- 若框架不支持，可将 Pod 名称通过 WasmPlugin CRD 的 `defaultConfig` 显式传入（每个 Gateway Pod 使用独立的 WasmPlugin 配置，或共享配置但包含一个 Pod 标识字段）

**健康状态聚合**（每个 Pod 本地执行）：
```go
func aggregateHealth(reports []ProbeReport, now int64, window time.Duration) HealthStatus {
    // 只取时间窗口内的报告（默认 60 秒 = 60000 毫秒，必须 >= failbackCooldown）
    valid := filter(reports, func(r ProbeReport) bool {
        return now - r.Ts <= int64(window.Milliseconds())
    })

    if len(valid) == 0 {
        return UNKNOWN
    }

    // Fail-fast：任一报告不健康 -> 标记为不健康
    for _, r := range valid {
        if !r.Healthy {
            return UNHEALTHY
        }
    }

    // 全部健康 -> 健康
    return HEALTHY
}
```

**Fail-fast 策略 rationale**：
- SMG 的 `/readiness` 是权威健康指标
- 任一 Pod 探测到 503，说明该 endpoint 确实有问题
- 恢复需要**全部活跃 Pod 的最近报告都是健康**，避免 flapping

**Fail-fast 策略的多维度副作用**：

Fail-fast 在快速故障发现上有效，但在以下维度存在值得关注的副作用：

*单 Pod 网络抖动导致全局误判*：假设某个 Gateway Pod 与某个 endpoint 之间存在临时网络抖动（如跨 AZ 链路闪断），该 Pod 探测超时或返回 503，发布 `unhealthy` 报告。其他 Pod 收到后，fail-fast 策略将该 endpoint 全局标记为 unhealthy——即使该 endpoint 本身完全正常，其他所有 Pod 探测都返回 200。误判持续时间 ≈ 该 Pod 的下一次成功探测时间 + Pub/Sub 传播延迟（3-6s）。期间所有流量被不必要地切换到备用 endpoint，造成 cache tree 分散。

*Pod 数量与恢复延迟的正相关*：恢复条件是"全部活跃 Pod 的最近报告都 healthy"。设探测周期为 T，Gateway Pod 数量为 N，每个 Pod 独立探测成功概率为 p。则恢复延迟期望 E ≈ T / p^N。当 p = 0.95（跨 AZ 部署、网络负载较高时）：N = 2 时 E ≈ 3.3s，N = 5 时 E ≈ 3.9s，N = 10 时 E ≈ 4.9s。要求连续 2 次 healthy 才能恢复时，N = 5 的延迟可达 ~8s。

*探测频率与一致性的 trade-off*：缩短探测周期减少延迟但增加负载和误判频率；延长周期则相反。

**缓解方向**：
1. Quorum-based 恢复：不要求"全部 Pod healthy"，而是"超过半数 healthy"（N > 2 时显著降低延迟）
2. 自适应探测频率：healthy 端点低频探测（5s），unhealthy 端点高频探测（1s）
3. 探测路径隔离：探测请求走同节点直接访问，不走 K8s Service LB

**死 Pod 报告处理**：
- 当 Gateway Pod 被缩容或崩溃时，其历史报告仍留在后端
- 聚合时需排除来自"已死亡" Pod 的报告
- 判定方法：若某 Pod 在 2 倍探测周期（如 6s）内未发布新报告，视为 inactive，其历史报告不参与聚合
- 这避免了"僵尸报告"导致 endpoint 长期被误判为 unhealthy

### 3.2 Consul KV 数据模型（默认）

Consul KV 是扁平 key-value 存储，HTTP 原生接口，无需任何代理层。

**Key 结构**（两类）：

```
smg/health/{group}/{endpoint}/{source}   → 原始探测报告
smg/locked/{group}                       → 全局锁定的活跃 endpoint
```

**Health report key** (`smg/health/...`)：
- `group`：SMG Group 名称（如 group-1）
- `endpoint`：endpoint 名称（如 smg-1.1）
- `source`：探测来源 Pod 名称（如 higress-gateway-7d9f4b8c5-x2abc）
- Value：`{"healthy": true, "ts": 1234567890000}`

**Locked key** (`smg/locked/{group}`)：
- `group`：SMG Group 名称
- Value：`{"endpoint": "smg-1.1", "lockedAt": 1234567890000}`
- 由健康探测在发现当前 locked endpoint 故障时通过 CAS 写入
- 所有 Pod 订阅同一 key，通过 blocking query 同步

**操作**：
- Bootstrap：`GET /v1/kv/smg/?recurse=true`（一次性拉取 health + locked）
- 订阅：`GET /v1/kv/smg/?recurse=true&index={lastIndex}&wait=30s`（health 和 locked 的变更共用同一个 blocking query）
- 发布探测报告：`PUT /v1/kv/smg/health/{group}/{endpoint}/{source}`
- CAS 写入 locked：`PUT /v1/kv/smg/locked/{group}?cas={ModifyIndex}`

每个 endpoint 的每个 source 独立为一个 health key。locked 是每个 Group 一个 key。订阅时使用 `?recurse=true` 监听 `/smg/` 前缀，任何 health 或 locked 的变化都会触发 blocking query 返回。blocking query 在 proxy-wasm 的异步模型下完美工作：最长等待 30 秒，响应必定在 0-30 秒内完成，触发 `OnHttpCallResponse` 回调。

### 3.3 Redis 后端数据模型（备选）

如果环境已有 Redis 但没有 Consul，Redis 作为备选方案（通过 Higress 自定义 Redis host 直接访问，不需要 HTTP bridge）：

```
# 每个 Group 一个 Stream，存储最近 100 条探测报告
XADD smg:updates:group-1 MAXLEN ~ 100 * {update_json}

# 每个 Group 一个 locked 键，存储当前全局锁定的 endpoint
SET smg:locked:group-1 "smg-1.1" EX 3600

# Bootstrap: XREVRANGE smg:updates:group-1 - + COUNT 100
#           + GET smg:locked:group-1
# 订阅: 每 3 秒轮询 XREAD STREAMS smg:updates:group-1 {last_id}
#       + GET smg:locked:group-1
```

Redis 的 locked 键使用简单的 `SET` / `GET`，CAS 通过 Lua 脚本实现（`EVAL`）：
```lua
-- CAS 原子更新 locked
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("SET", KEYS[1], ARGV[2])
else
    return nil
end
```

---

## 4. TopologyBackend 接口定义

```go
package backend

import "time"

type HealthStatus int

const (
    UNKNOWN HealthStatus = iota
    HEALTHY
    UNHEALTHY
)

type ProbeReport struct {
    Source  string `json:"source"`
    Healthy bool   `json:"healthy"`
    Ts      int64  `json:"ts"`
}

type HealthUpdate struct {
    Group    string `json:"group"`
    Endpoint string `json:"endpoint"`
    Report   ProbeReport `json:"report"`
}

type LockedUpdate struct {
    Group       string `json:"group"`
    Endpoint    string `json:"endpoint"`
    LockedAt    int64  `json:"lockedAt"`
    ModifyIndex uint64 `json:"modifyIndex"`  // 用于 Consul CAS
}

type BootstrapResult struct {
    Reports map[string]map[string][]ProbeReport  // group → endpoint → reports
    Locked  map[string]*LockedUpdate             // group → locked endpoint
}

// TopologyBackend 可插拔后端接口
type TopologyBackend interface {
    // PublishProbeResult 发布本 Pod 的探测结果
    PublishProbeResult(group, endpoint string, healthy bool, ts int64) error

    // SubscribeHealthUpdates 订阅健康状态增量更新
    SubscribeHealthUpdates(updateCh chan<- HealthUpdate) error

    // SubscribeLockedUpdates 订阅 locked endpoint 增量更新
    SubscribeLockedUpdates(lockedCh chan<- LockedUpdate) error

    // BootstrapFullState 启动时拉取全量状态（health reports + locked endpoints）
    BootstrapFullState() (*BootstrapResult, error)

    // CasWriteLocked CAS 写入当前 Group 的 locked endpoint
    // 返回 (success, currentValue, error)
    // 失败时调用方应读取 currentValue 并采用
    CasWriteLocked(group, endpoint string, expectedIndex uint64) (bool, *LockedUpdate, error)

    // GetLocked 读取指定 Group 的 locked endpoint
    GetLocked(group string) (*LockedUpdate, error)

    Close() error
}

// Config 后端配置
type Config struct {
    BackendType string `json:"backendType"`  // "redis", "consul", "nacos"
    Redis       RedisConfig `json:"redis,omitempty"`
    Consul      ConsulConfig `json:"consul,omitempty"`
    Nacos       NacosConfig `json:"nacos,omitempty"`
}

type RedisConfig struct {
    ServiceName string `json:"serviceName"`
    Namespace   string `json:"namespace"`
    Port        int64  `json:"port"`
    Password    string `json:"password,omitempty"`
}

type ConsulConfig struct {
    Address string `json:"address"`
    Token   string `json:"token,omitempty"`
}

type NacosConfig struct {
    ServerAddr  string `json:"serverAddr"`
    NamespaceId string `json:"namespaceId,omitempty"`
}
```

---

## 5. 后端实现选型与 WASM 可行性

### 5.1 WASM HTTP 能力确认

Higress wasm-go 插件已通过大量生产实践验证出站 HTTP 能力：
- `ai-cache` 连接 Qdrant/Weaviate/Pinecone/Elasticsearch（向量数据库）
- `ai-search` 连接 Google/Bing/Quark/arxiv（搜索引擎）
- `http-call` 扩展演示了访问 K8s Service / Nacos / 静态 IP / DNS 域名
- `ai-proxy` failover 使用 `wrapper.NewClusterClient(wrapper.TargetCluster)` 发 health check 请求

所有后端均通过 `wrapper.HttpClient`（基于 `proxywasm.DispatchHttpCall`）访问，**纯 HTTP，无持久 TCP 连接需求**。

### 5.2 后端对比

| 后端 | 原生协议 | WASM 访问方式 | 是否需要代理 | 实时性 | 推荐度 |
|------|---------|--------------|-------------|--------|--------|
| **Consul KV** | HTTP | 原生 HTTP blocking query (`?index=&wait=`) | 不需要 | 近实时 | **默认推荐** |
| **Nacos** | HTTP | 原生 HTTP long polling | 不需要 | 近实时 | 环境已有 Nacos 时 |
| **Redis** | RESP (TCP) | Higress 自定义 host (`proxy_redis_call`) | 不需要 | 近实时 | 备选 |
| **etcd** | gRPC / HTTP streaming | proxy-wasm 不支持 streaming | — | — | **不可行** |

### 5.3 Consul 原生 HTTP 方案（默认）

Consul KV 提供原生 HTTP API，proxy-wasm 可直接通过 `DispatchHttpCall` 访问，**无需任何代理层**。

**核心机制：HTTP Blocking Query**

```
GET /v1/kv/smg/?recurse=true&index={lastIndex}&wait=30s
```

- `index`：上次读取的 ModifyIndex。自该 index 之后有修改则立即返回新数据（HTTP 200 + `X-Consul-Index` 新值）。
- `wait`：最长等待时间（如 30s）。超时后返回空响应（HTTP 200，`X-Consul-Index` 不变）。
- 响应必定在 0-30 秒内完成并关闭连接，触发 `OnHttpCallResponse`。
- 回调中解析结果，用新的 index 立即递归发起下一个 blocking query。

这与 proxy-wasm 的异步 HTTP 模型完美匹配：不需要 goroutine，不需要持久 TCP 连接，不需要任何代理层。

**数据模型**

```
PUT /v1/kv/smg/health/{group}/{endpoint}/{source}
Value: {"healthy":true,"ts":1234567890000}
```

- 每个 endpoint 的每个 source 独立为一个 key
- 订阅时使用 `?recurse=true` 监听整个 `smg/` 前缀（同时覆盖 health 和 locked）
- Bootstrap 时使用 `GET /v1/kv/smg/?recurse=true`（不加 index，立即返回）

**优势**
- 一个组件（Consul 集群），不需要代理
- 原生 HTTP，代码简洁可控
- blocking query 实时性优于 Redis 短轮询（变化立即推送，无变化仅 30s 一次心跳）
- 支持 session TTL 自动清理失联 Pod 的报告

**部署**
- 环境已有 Consul：直接使用，零额外组件
- 无 Consul：通过 Helm 部署 3 节点 Consul server 集群（Raft 共识）

---

### 5.4 etcd 可行性分析

etcd 是 K8s 集群的核心数据存储，看似"自带"、无需额外部署，但实际不可行，原因如下：

**（1）网络可达性与访问风险**

K8s 的 etcd 通常只在 master 节点监听 `127.0.0.1:2379`，不对外暴露 Service。要让 Gateway Pod 访问 etcd，需要额外的网络打通和 TLS 证书分发。更关键的是，etcd 存储的是 K8s 的完整集群状态（包括 Secrets），直接让业务组件（WASM 插件）访问 etcd 会带来严重的安全边界问题。正确的访问路径应该是通过 K8s API Server，但这又引入了新的限制。

**（2）API 协议：gRPC streaming，proxy-wasm 不支持**

etcd v3 的核心 API 是 gRPC。proxy-wasm 规范仅支持 HTTP/1.1 出站请求，不支持 gRPC。etcd 虽提供 gRPC-gateway 的 HTTP/JSON 接口，但其 Watch 操作（`/v3/watch`）返回的是 chunked transfer-encoding streaming 响应。proxy-wasm 的 `DispatchHttpCall` + `OnHttpCallResponse` 模型要求响应体完整接收后才触发回调——对于永不结束的 Watch stream，`OnHttpCallResponse` 永远不会触发。

**Higress 扩展能力调研**：我们进一步调研了 Higress 是否在 proxy-wasm 之上提供了 streaming HTTP 或 gRPC 的扩展能力。

- **Higress wasm-go 自定义 host 函数**：Higress 的 proxy-wasm-go-sdk fork 增加了若干自定义 host 函数（`proxy_set_upstream_override_host`、`proxy_get_upstream_hosts`、`proxy_redis_init`、`proxy_call_foreign_function` 等），但**没有增加 streaming HTTP client 能力**。`DispatchHttpCall` 仍然是标准 proxy-wasm 行为——等待完整响应后触发回调。
- **Higress Go HTTP filter**：Higress 支持 Envoy 的 Go HTTP filter（`plugins/golang-filter/`），这是原生共享库（非 WASM），支持 inbound 请求/响应的流式处理（`EncodeData`/`DecodeData` 可被多次调用）。但 Go filter 不是 WASM 插件，且**不适合发起独立的出站 streaming 连接到 etcd 或 K8s API Server**。
- **结论**：即使在 Higress 的扩展能力范围内，streaming HTTP/gRPC 出站调用仍然不可行。

**（3）K8s API Server 的 Watch 同样不可行**

通过 K8s API Server 的 REST API 访问更安全，但 K8s 的 Watch（`?watch=true`）同样是 chunked streaming 响应，proxy-wasm 同样不支持。轮询 ConfigMap 或 EndpointSlice 虽然技术上可行，但受限于 K8s API rate limit（默认 5 QPS），且需要为 Gateway ServiceAccount 授予写权限（修改 ConfigMap），权限粒度控制复杂，生产环境中不推荐。

**（4）轮询不是 Pub/Sub**

即使退化为轮询（`GET /v3/kv/range`），etcd 也不是为高频读取设计的。health check 场景需要秒级同步，轮询间隔设为 1-3 秒会对 etcd 造成不必要的负载。etcd 的设计目标是存储低频变更的集群配置，而非高频写入的健康探测结果。

**结论**：etcd 不可行。问题不是"有没有 etcd"，而是 proxy-wasm 的 HTTP 能力边界决定了 streaming-based Watch 机制无法工作，而轮询又违背了 etcd 的设计初衷。

---

### 5.5 Consul 可行性分析

Consul KV 的 HTTP blocking query 是 proxy-wasm 环境下最自然的 Pub/Sub 机制，甚至比 Redis Streams 更贴合 WASM 的异步 HTTP 模型。

**（1）Blocking Query 与 proxy-wasm 的天然适配**

```
GET /v1/kv/smg/health/group-1?index=42&wait=30s
```

- `index=42`：上次读取的 ModifyIndex。如果 key 自 index 42 之后有修改，Consul 立即返回新数据（HTTP 200 + X-Consul-Index 新值）。
- `wait=30s`：最长等待 30 秒。如果 30 秒内无修改，Consul 返回空响应（HTTP 200，X-Consul-Index 不变）。
- 响应在 0-30 秒内必定完成并关闭连接，触发 proxy-wasm 的 `OnHttpCallResponse` 回调。
- 回调中解析结果，用新的 index 立即发起下一个 blocking query。

这与 proxy-wasm 的异步模型完美匹配：`DispatchHttpCall` → 等待 0-30s → `OnHttpCallResponse` → 递归。不需要 goroutine，不需要持久 TCP 连接，不需要任何代理层。

**（2）数据模型适配**

Consul KV 是扁平 key-value，可以用前缀组织健康数据：

```
smg/health/group-1/smg-1.1/pod-a  ->  {"healthy":true,"ts":1234567890000}
smg/health/group-1/smg-1.1/pod-b  ->  {"healthy":true,"ts":1234567890000}
smg/health/group-1/smg-1.2/pod-a  ->  {"healthy":false,"ts":1234567890000}
```

每个 endpoint 的每个 source 独立为一个 key。订阅时使用 `?recurse=true` 监听整个 `smg/` 前缀（同时覆盖 health 报告和 locked 状态），任何变化都会触发 blocking query 返回。发布时使用 `PUT /v1/kv/<key>` 写入本 Pod 的探测结果。

**（3）Consul KV 的 TTL 机制**

Consul KV 支持 session-based TTL（`?acquire=<session>` + `Session.TTL`），可以自动清理失联 Pod 的报告。这比 Redis 的 Stream MAXLEN 更精确——不是按数量裁剪，而是按 Pod 活跃度自动过期。

**（4）劣势**

- **部署复杂度**：如果环境没有 Consul，需要部署 3 节点 Consul server 集群。但生产环境的 Redis 同样不能单实例运行（否则是单点故障），至少需要 1 master + 1 replica。值得注意的是，Higress wasm-go 框架通过自定义 host 函数 `proxy_redis_init` / `proxy_redis_call` 提供了 `wrapper.RedisClient` 接口，WASM 插件可以直接通过 RESP 协议访问 Redis，**不需要 HTTP bridge**。因此 Redis 的部署只需要 Redis 本身（通常至少 2 个 Pod），组件数量上比 Consul（3 节点）还少。但 Redis 的订阅机制仍是短轮询（通过 `Command` 执行 `XREAD`，在 callback 中递归），实时性不如 Consul blocking query。
- **写入吞吐量**：Consul KV 的写入性能低于 Redis（基于 Raft 共识的持久化）。但 health check 场景写入频率很低（每个 Pod 每秒几次），不构成瓶颈。
- **并发限制**：Consul 默认限制 512 个并发 blocking query。Gateway 多副本部署时（通常 2-10 个 Pod），每个 Pod 发起 1 个 blocking query（监听前缀），远低于限制。

**（5）Consul vs Redis**

| 维度 | Consul | Redis（Higress 原生） |
|------|--------|----------------------|
| 组件数量 | 1（Consul 集群，3 节点） | 1（Redis，1 master + 1 replica） |
| HTTP 代理 | 不需要 | 不需要（Higress 自定义 Redis host） |
| 运维熟悉度 | 取决于团队 | Redis 更普遍 |
| 部署复杂度 | 中（3 节点集群，官方 Helm） | 低（Redis HA，无自定义代码） |
| Pub/Sub 机制 | HTTP blocking query（变化立即推送） | RESP `XREAD` 短轮询（每 3s） |
| 同步延迟 | ~0ms（变化立即推送） | 0-3s（轮询间隔） |
| 可移植性 | 高（标准 HTTP，任何 proxy-wasm 实现都支持） | 低（绑定 Higress 自定义 Redis host） |
| 环境已有 Consul | 直接使用，零额外组件 | — |
| 无 Consul 环境 | 部署 3 节点集群 | 部署 Redis（通常 1 master + 1 replica） |

**结论**：Consul KV 是 V5.2 的默认推荐后端。原生 HTTP 接口与 proxy-wasm 的异步模型完美适配，blocking query 提供了比 Redis 短轮询更优的实时性和效率，且不需要任何代理层。如果团队环境已有 Consul，直接使用即可；如果还没有 Consul，部署一个 3 节点 Consul server 集群是推荐做法。

Redis 由于 Higress 的自定义 Redis host 支持，不需要 bridge，部署组件数量（通常 1 master + 1 replica）比 Consul（3 节点）还少。但 Redis 的实时性不如 Consul（短轮询 0-3s vs blocking query 变化立即推送），且绑定 Higress 自定义扩展（可移植性差）。如果团队已有 Redis 且对实时性要求不极端，Redis 是一个务实的备选。

---

## 6. ConsulBackend 详细实现

### 6.1 数据结构设计

```go
// ConsulBackend Consul KV 后端实现
type ConsulBackend struct {
    config       ConsulConfig
    httpClient   wrapper.HttpClient
    podName      string              // 本 Pod 名称，作为 Source
    updateCh      chan<- HealthUpdate  // health 订阅通道
    lockedUpdateCh chan<- LockedUpdate  // locked 订阅通道
    lastIndex     uint64               // 上次订阅的 Consul ModifyIndex
    bootstrapped bool                // Bootstrap 是否完成
}

type ConsulConfig struct {
    Address    string `json:"address"`    // e.g. "consul-server.higress-system.svc.cluster.local:8500"
    Token      string `json:"token,omitempty"`
    Datacenter string `json:"datacenter,omitempty"`
}

// ConsulKVEntry Consul KV 响应条目
type ConsulKVEntry struct {
    Key         string `json:"Key"`
    Value       string `json:"Value"`       // base64 编码的字符串
    ModifyIndex uint64 `json:"ModifyIndex"`
}

func NewConsulBackend(cfg ConsulConfig, podName string) (*ConsulBackend, error) {
    // 解析地址
    host, portStr, err := net.SplitHostPort(cfg.Address)
    if err != nil {
        host = cfg.Address
        portStr = "8500"
    }
    port, _ := strconv.Atoi(portStr)
    
    client := wrapper.NewClusterClient(wrapper.K8sCluster{
        ServiceName: host,
        Namespace:   "higress-system", // 从 WasmPlugin 配置读取
        Port:        uint32(port),
    })
    
    return &ConsulBackend{
        config:     cfg,
        httpClient: client,
        podName:    podName,
    }, nil
}
```

### 6.2 PublishProbeResult

发布探测结果是一个简单的 PUT 请求：

```go
func (c *ConsulBackend) PublishProbeResult(group, endpoint string, healthy bool, ts int64) error {
    key := fmt.Sprintf("smg/health/%s/%s/%s", group, endpoint, c.podName)
    report := ProbeReport{
        Source:  c.podName,
        Healthy: healthy,
        Ts:      ts,
    }
    value, _ := json.Marshal(report)
    
    return c.httpClient.Put(
        fmt.Sprintf("/v1/kv/%s", key),
        [][2]string{{"Content-Type", "application/json"}},
        value,
        func(statusCode int, headers http.Header, body []byte) {
            if statusCode != 200 {
                proxywasm.LogErrorf("Consul PUT failed: status=%d key=%s", statusCode, key)
            }
        },
    )
}
```

### 6.3 SubscribeHealthUpdates 与 BootstrapFullState

Consul 的订阅基于 HTTP blocking query，与 proxy-wasm 的异步模型天然适配。

**Bootstrap**：首次获取全量数据并记录初始 index。

```go
func (c *ConsulBackend) SubscribeHealthUpdates(updateCh chan<- HealthUpdate) error {
    c.updateCh = updateCh
    return nil
}

func (c *ConsulBackend) SubscribeLockedUpdates(lockedCh chan<- LockedUpdate) error {
    c.lockedUpdateCh = lockedCh
    return nil
}

func (c *ConsulBackend) BootstrapFullState() (*BootstrapResult, error) {
    result := &BootstrapResult{
        Reports: make(map[string]map[string][]ProbeReport),
        Locked:  make(map[string]*LockedUpdate),
    }
    
    // 异步 Bootstrap：发起 HTTP GET，在 callback 中解析 health + locked
    c.httpClient.Get(
        "/v1/kv/smg/?recurse=true",  // 拉取 /smg/ 前缀下所有 key（health + locked）
        nil,
        func(statusCode int, headers http.Header, body []byte) {
            if statusCode != 200 {
                proxywasm.LogErrorf("Consul Bootstrap failed: status=%d", statusCode)
                return
            }
            
            // 记录 X-Consul-Index
            if idx := headers.Get("X-Consul-Index"); idx != "" {
                c.lastIndex, _ = strconv.ParseUint(idx, 10, 64)
            }
            
            var kvs []ConsulKVEntry
            if err := json.Unmarshal(body, &kvs); err != nil {
                proxywasm.LogErrorf("Consul Bootstrap unmarshal failed: %v", err)
                return
            }
            
            for _, kv := range kvs {
                if strings.HasPrefix(kv.Key, "smg/health/") {
                    c.parseAndBootstrapHealth(kv, result)
                } else if strings.HasPrefix(kv.Key, "smg/locked/") {
                    c.parseAndBootstrapLocked(kv, result)
                }
            }
            
            c.bootstrapped = true
            // Bootstrap 完成后立即启动 blocking query 订阅（监听 /smg/ 前缀，包含 health 和 locked）
            c.startBlockingQuery()
        },
    )
    
    return result, nil
}

func (c *ConsulBackend) parseAndBootstrapHealth(kv ConsulKVEntry, result *BootstrapResult) {
    // key: smg/health/{group}/{endpoint}/{source}
    parts := strings.Split(kv.Key, "/")
    if len(parts) != 5 {
        return
    }
    group := parts[2]
    endpoint := parts[3]
    source := parts[4]
    
    decoded, _ := base64.StdEncoding.DecodeString(kv.Value)
    var report ProbeReport
    json.Unmarshal(decoded, &report)
    report.Source = source
    
    if result.Reports[group] == nil {
        result.Reports[group] = make(map[string][]ProbeReport)
    }
    result.Reports[group][endpoint] = append(result.Reports[group][endpoint], report)
}

func (c *ConsulBackend) parseAndBootstrapLocked(kv ConsulKVEntry, result *BootstrapResult) {
    // key: smg/locked/{group}
    parts := strings.Split(kv.Key, "/")
    if len(parts) != 3 {
        return
    }
    group := parts[2]
    
    decoded, _ := base64.StdEncoding.DecodeString(kv.Value)
    var locked LockedUpdate
    json.Unmarshal(decoded, &locked)
    locked.Group = group
    locked.ModifyIndex = kv.ModifyIndex
    
    result.Locked[group] = &locked
}
```

**订阅循环**：blocking query 递归。

```go
func (c *ConsulBackend) startBlockingQuery() {
    c.doBlockingQuery()
}

func (c *ConsulBackend) doBlockingQuery() {
    // 监听 /smg/ 前缀，同时订阅 health 报告和 locked 状态
    url := fmt.Sprintf("/v1/kv/smg/?recurse=true&index=%d&wait=30s", c.lastIndex)
    
    c.httpClient.Get(url, nil, func(statusCode int, headers http.Header, body []byte) {
        if statusCode != 200 {
            proxywasm.LogErrorf("Consul blocking query failed: status=%d", statusCode)
            // 出错后由 RegisterTickFunc 在 3 秒后重试
            return
        }
        
        if idx := headers.Get("X-Consul-Index"); idx != "" {
            newIndex, _ := strconv.ParseUint(idx, 10, 64)
            if newIndex == c.lastIndex {
                // 无变化，index 未变，递归发起下一个
                c.doBlockingQuery()
                return
            }
            c.lastIndex = newIndex
        }
        
        var kvs []ConsulKVEntry
        if err := json.Unmarshal(body, &kvs); err != nil {
            proxywasm.LogErrorf("Consul blocking query unmarshal failed: %v", err)
            c.doBlockingQuery()
            return
        }
        
        for _, kv := range kvs {
            c.parseAndPush(kv)
        }
        
        // 递归发起下一个 blocking query
        c.doBlockingQuery()
    })
}

func (c *ConsulBackend) parseAndPush(kv ConsulKVEntry) {
    // 区分 health key 和 locked key
    if strings.HasPrefix(kv.Key, "smg/health/") {
        c.parseAndPushHealth(kv)
    } else if strings.HasPrefix(kv.Key, "smg/locked/") {
        c.parseAndPushLocked(kv)
    }
}

func (c *ConsulBackend) parseAndPushHealth(kv ConsulKVEntry) {
    // key: smg/health/{group}/{endpoint}/{source}
    parts := strings.Split(kv.Key, "/")
    if len(parts) != 5 {
        return
    }
    group := parts[2]
    endpoint := parts[3]
    source := parts[4]
    
    decoded, err := base64.StdEncoding.DecodeString(kv.Value)
    if err != nil {
        proxywasm.LogErrorf("Consul base64 decode failed: %v", err)
        return
    }
    
    var report ProbeReport
    if err := json.Unmarshal(decoded, &report); err != nil {
        proxywasm.LogErrorf("Consul JSON unmarshal failed: %v", err)
        return
    }
    report.Source = source
    
    c.updateCh <- HealthUpdate{
        Group:    group,
        Endpoint: endpoint,
        Report:   report,
    }
}

func (c *ConsulBackend) parseAndPushLocked(kv ConsulKVEntry) {
    // key: smg/locked/{group}
    parts := strings.Split(kv.Key, "/")
    if len(parts) != 3 {
        return
    }
    group := parts[2]
    
    decoded, err := base64.StdEncoding.DecodeString(kv.Value)
    if err != nil {
        proxywasm.LogErrorf("Consul locked base64 decode failed: %v", err)
        return
    }
    
    var locked LockedUpdate
    if err := json.Unmarshal(decoded, &locked); err != nil {
        proxywasm.LogErrorf("Consul locked JSON unmarshal failed: %v", err)
        return
    }
    locked.Group = group
    locked.ModifyIndex = kv.ModifyIndex  // 从 Consul KV 元数据中获取
    
    // 推送到 locked update channel（与 health update 共用 channel 或独立 channel）
    c.lockedUpdateCh <- locked
}

// CasWriteLocked 使用 Consul KV CAS 原子更新 locked endpoint
func (c *ConsulBackend) CasWriteLocked(group, endpoint string, expectedIndex uint64) (bool, *LockedUpdate, error) {
    key := fmt.Sprintf("/v1/kv/smg/locked/%s", group)
    value := fmt.Sprintf(`{"endpoint":"%s","lockedAt":%d}`, endpoint, time.Now().UnixMilli())
    url := fmt.Sprintf("%s?cas=%d", key, expectedIndex)
    
    var success bool
    var winner *LockedUpdate
    
    c.httpClient.Put(url, []byte(value), func(statusCode int, headers http.Header, body []byte) {
        // Consul CAS 返回 "true" 或 "false"
        success = strings.TrimSpace(string(body)) == "true"
        
        if !success {
            // CAS 失败，读取当前值
            c.httpClient.Get(key, nil, func(sc int, h http.Header, b []byte) {
                if sc == 200 {
                    var kvs []ConsulKVEntry
                    json.Unmarshal(b, &kvs)
                    if len(kvs) > 0 {
                        decoded, _ := base64.StdEncoding.DecodeString(kvs[0].Value)
                        var lu LockedUpdate
                        json.Unmarshal(decoded, &lu)
                        lu.Group = group
                        lu.ModifyIndex = kvs[0].ModifyIndex
                        winner = &lu
                    }
                }
            })
        }
    })
    
    return success, winner, nil
}

// GetLocked 读取指定 Group 的 locked endpoint
func (c *ConsulBackend) GetLocked(group string) (*LockedUpdate, error) {
    key := fmt.Sprintf("/v1/kv/smg/locked/%s", group)
    var result *LockedUpdate
    
    c.httpClient.Get(key, nil, func(statusCode int, headers http.Header, body []byte) {
        if statusCode != 200 {
            return
        }
        var kvs []ConsulKVEntry
        json.Unmarshal(body, &kvs)
        if len(kvs) > 0 {
            decoded, _ := base64.StdEncoding.DecodeString(kvs[0].Value)
            var lu LockedUpdate
            json.Unmarshal(decoded, &lu)
            lu.Group = group
            lu.ModifyIndex = kvs[0].ModifyIndex
            result = &lu
        }
    })
    
    return result, nil
}
```

**说明**：blocking query 在 proxy-wasm 中通过 `OnHttpCallResponse` 回调递归实现。每次响应完成后立即发起下一个请求，形成"请求 → 等待 → 回调 → 再请求"的循环。不需要 goroutine，不需要持久连接。单个 Gateway Pod 只维持 1 个 pending 的 blocking query（监听 `/smg/` 前缀，同时覆盖 health 和 locked），Consul 默认支持 512 个并发 blocking query，完全够用。

### 6.4 错误恢复

```go
func (c *ConsulBackend) Close() error {
    // 标记停止，阻止新的 blocking query
    return nil
}
```

如果 Consul 暂时不可用：
- `DispatchHttpCall` 返回连接错误
- `OnHttpCallResponse` 触发，statusCode != 200
- 停止递归，等待 RegisterTickFunc 的下一次 tick 重试
- 重试间隔：首次 3 秒，指数退避到 30 秒

这与 V5.2 的 graceful degradation 策略一致：后端不可用时，退化为 V5.1 行为（本地探测 + 本地锁定）。

---

## 7. 本地健康缓存

### 7.1 数据结构

```go
type LocalHealthCache struct {
    mu       sync.RWMutex
    // map[group][endpoint] -> 原始报告列表（按时间排序）
    reports  map[string]map[string][]ProbeReport
    // map[group][endpoint] -> 聚合后的健康状态（缓存）
    status   map[string]map[string]HealthStatus
    // 状态缓存的过期时间
    statusExpiry map[string]map[string]time.Time
    // 聚合时间窗口
    window   time.Duration  // 默认 60s（必须 >= failbackCooldown）

    // map[group] -> 后端同步的 locked endpoint
    locked       map[string]string
    lockedAt     map[string]int64
    lockedIndex  map[string]uint64  // Consul ModifyIndex，用于 CAS
}
```

### 7.2 核心方法

```go
func (c *LocalHealthCache) UpdateReport(group, endpoint string, report ProbeReport) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if c.reports[group] == nil {
        c.reports[group] = make(map[string][]ProbeReport)
    }

    // 同一 source 只保留最新报告：请求失败时的疑似不健康标记
    // 会被下一次主动探测结果覆盖，避免单次抖动导致长达 window 的误判
    c.reports[group][endpoint] = upsertBySource(c.reports[group][endpoint], report)
    c.reports[group][endpoint] = filterRecent(c.reports[group][endpoint], c.window)

    // 使状态缓存失效
    delete(c.status[group], endpoint)
}

func (c *LocalHealthCache) GetHealthStatus(group, endpoint string) HealthStatus {
    c.mu.RLock()
    if expiry, ok := c.statusExpiry[group][endpoint]; ok && time.Now().Before(expiry) {
        status := c.status[group][endpoint]
        c.mu.RUnlock()
        return status
    }
    c.mu.RUnlock()

    // 缓存未命中，重新聚合
    c.mu.Lock()
    defer c.mu.Unlock()

    reports := c.reports[group][endpoint]
    status := aggregateHealth(reports, time.Now().Unix(), c.window)

    if c.status[group] == nil {
        c.status[group] = make(map[string]HealthStatus)
        c.statusExpiry[group] = make(map[string]time.Time)
    }
    c.status[group][endpoint] = status
    c.statusExpiry[group][endpoint] = time.Now().Add(1 * time.Second)  // 1s TTL

    return status
}

func (c *LocalHealthCache) UpdateLockedEndpoint(group, endpoint string, lockedAt int64, index uint64) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.locked[group] = endpoint
    c.lockedAt[group] = lockedAt
    c.lockedIndex[group] = index
}

func (c *LocalHealthCache) GetLockedEndpoint(group string) (string, int64, uint64) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.locked[group], c.lockedAt[group], c.lockedIndex[group]
}

func (c *LocalHealthCache) LastUnhealthyTime(group, endpoint string) int64 {
    c.mu.RLock()
    defer c.mu.RUnlock()

    reports := c.reports[group][endpoint]
    var last int64 = 0
    for _, r := range reports {
        if !r.Healthy && r.Ts > last {
            last = r.Ts
        }
    }
    return last
}
```

---

## 8. 确定性锁定算法

### 8.1 算法定义

```go
type FailoverPolicy int

const (
    FailoverSequential FailoverPolicy = iota   // 顺序降级：当前 locked 失败后，选下一个
    FailoverPriority FailoverPolicy = iota     // 优先级回退：当前 locked 失败后，选最高优先级 HEALTHY
)

type GroupConfig struct {
    Name               string
    Endpoints          []EndpointConfig       // 按优先级排序
    FailoverPolicy     FailoverPolicy         // 默认 Sequential
    FailbackCooldownMs int64                  // 冷却期（默认 30000）
}

func selectLockedEndpoint(group GroupConfig, cache *LocalHealthCache) string {
    locked, lockedAt, _ := cache.GetLockedEndpoint(group.Name)

    // 1. 如果存在后端同步的 locked，且仍健康，保持粘滞
    if locked != "" {
        status := cache.GetHealthStatus(group.Name, locked)
        if status == HEALTHY {
            return locked
        }
        // locked 已不健康，需要重新选举（见下方）
    }

    // 2. 没有 locked 或 locked 已不健康：按策略选举新的
    return electNewLocked(group, cache, locked)
}

func electNewLocked(group GroupConfig, cache *LocalHealthCache, prevLocked string) string {
    switch group.FailoverPolicy {
    case FailoverSequential:
        return electSequential(group, cache, prevLocked)
    case FailoverPriority:
        return electPriority(group, cache)
    default:
        return electSequential(group, cache, prevLocked)
    }
}

// 顺序降级：从 prevLocked 的下一个开始找 HEALTHY，避免跳回已恢复的高优先级节点
func electSequential(group GroupConfig, cache *LocalHealthCache, prevLocked string) string {
    startIdx := 0
    if prevLocked != "" {
        for i, ep := range group.Endpoints {
            if ep.Name == prevLocked {
                startIdx = i + 1
                break
            }
        }
    }

    // 从 startIdx 往后找第一个 HEALTHY
    for i := startIdx; i < len(group.Endpoints); i++ {
        if cache.GetHealthStatus(group.Name, group.Endpoints[i].Name) == HEALTHY {
            return group.Endpoints[i].Name
        }
    }

    // 后面的全不健康，才从头找（回退到最高优先级 HEALTHY）
    for _, ep := range group.Endpoints {
        if cache.GetHealthStatus(group.Name, ep.Name) == HEALTHY {
            return ep.Name
        }
    }

    return group.Endpoints[0].Name  // fail-open
}

// 优先级回退：始终选最高优先级的 HEALTHY endpoint（传统行为）
func electPriority(group GroupConfig, cache *LocalHealthCache) string {
    for _, ep := range group.Endpoints {
        if cache.GetHealthStatus(group.Name, ep.Name) == HEALTHY {
            return ep.Name
        }
    }
    return group.Endpoints[0].Name  // fail-open
}
```

### 8.2 为什么仍然不需要分布式锁

`locked` 的引入看似增加了"共享可变状态"，但与传统分布式锁有本质区别：

| 维度 | 分布式锁 | V5.2 的 locked CAS |
|------|---------|-------------------|
| 持有时间 | 长期持有（秒到分钟） | 仅在故障切换瞬间写入（毫秒级） |
| 竞争频率 | 高（每次操作前抢锁） | 极低（只有 endpoint 故障时竞争） |
| 失败处理 | 等待重试，可能死锁 | CAS 失败即读取获胜者值，无等待 |
| 状态依赖 | 锁持有者故障导致全局阻塞 | 无持有者概念，任何 Pod 都可触发选举 |

**locked 的写入流程**：

```
Pod-A 探测到 locked=5 失败
    |
    → 计算新 locked（按 FailoverPolicy）
    |
    → CAS Write /smg/locked/group-1
        | 成功 → 新 locked 生效
        | 失败 → 读取获胜者写入的值 → 采用
    |
    → 继续正常工作
```

所有 Pod 都通过 blocking query 订阅 `/smg/locked/{group}`，写入后几秒内全部同步。

**一致性保证**：
- 正常情况下：`locked` 健康 → 所有 Pod 读相同值 → 全部路由到同一 endpoint
- 故障时：多个 Pod 同时 CAS，一个成功，其余采用成功者值 → 全部切换到同一新 endpoint
- 新 Pod Bootstrap：读取后端 `locked` → 与现有 Pod 一致
- 订阅断开：使用本地缓存的 `locked`，最多缓存 TTL（1s）后重新读取

### 8.3 CAS 选举触发点

CAS 写入 `locked` 只在**周期性健康探测**中触发，不在请求路径上触发：

```go
func (p *Plugin) probeEndpoint(group, endpoint, address string, ts int64) {
    p.httpClient.Get(address + "/readiness", nil, func(statusCode int, headers http.Header, body []byte) {
        healthy := statusCode == 200

        // 1. 更新本地缓存
        p.healthCache.UpdateReport(group, endpoint, ProbeReport{...})

        // 2. 发布到后端
        p.backend.PublishProbeResult(group, endpoint, healthy, ts)

        // 3.【新增】如果探测的是当前 locked endpoint，且它不健康了
        locked, _, lockedIdx := p.healthCache.GetLockedEndpoint(group)
        if endpoint == locked && !healthy {
            newLocked := electNewLocked(p.getGroupConfig(group), p.healthCache, locked)
            success, winner, _ := p.backend.CasWriteLocked(group, newLocked, lockedIdx)
            if !success && winner != nil {
                // 采用获胜者的值
                p.healthCache.UpdateLockedEndpoint(group, winner.Endpoint, winner.LockedAt, winner.ModifyIndex)
            } else if success {
                p.healthCache.UpdateLockedEndpoint(group, newLocked, ts, lockedIdx+1)
            }
        }
    })
}
```

**为什么不在请求路径上触发 CAS？**
- 请求路径不能阻塞等待 CAS 结果
- 多个并发请求同时检测到失败会导致 CAS 竞争混乱
- 周期性探测已覆盖故障发现（最多 `HealthCheckInterval` 延迟），被动重试兜底

### 8.4 增强架构支持的策略

引入 `/smg/locked/{group}` 后，V5.2 支持的策略矩阵：

| 策略 | FailoverPolicy | 自动回切 | 说明 |
|------|---------------|---------|------|
| **顺序降级（默认）** | `Sequential` | ❌ 永不自动回切 | 当前 locked 失败后，按优先级顺序找下一个 HEALTHY。高优先级节点恢复后不回切，直到当前 locked 也失败 |
| **优先级回退** | `Priority` | ✅ 故障即回切 | 每次重新选举都选最高优先级的 HEALTHY endpoint。传统行为，适合无状态后端 |
| **人工确认回切** | `Sequential` + 人工改 locked | ❌ 需人工触发 | 运维直接写入 `/smg/locked/{group}` 指定回切目标。所有 Pod 同步切换 |
| **配置粘滞** | `Sequential` + `pinToBackup: true` | ❌ 永不自动回切 | 配置中指定固定 locked endpoint，算法不再自动选举 |

**推荐默认策略**：`Sequential`。它最符合"能不回切就不回切"的原则——一旦流量稳定到某个 endpoint，就保持粘滞，不因高优先级节点恢复而迁移。

**回切方式**：
1. **自动回切**：不支持（ intentionally 设计如此）
2. **冷却期回切**：可作为未来扩展（在 `selectLockedEndpoint` 中检查 `LastUnhealthyTime` + `FailbackCooldownMs`）
3. **人工回切**：运维直接 PUT `/smg/locked/{group}` → 所有 Pod 同步切换

---

## 9. 请求路径流程

```go
func (p *Plugin) OnHttpRequestHeaders(ctx wrapper.HttpContext, config Config) types.Action {
    // 1. 读取 Group 标识（从 header 或 URL path）
    groupName := extractGroupFromRequest(ctx)
    group := p.getGroupConfig(groupName)

    // 2. 纯本地内存计算锁定 endpoint
    //    优先使用后端同步的 locked，实现跨 Pod 粘滞
    lockedEndpoint := selectLockedEndpoint(group, p.healthCache)

    // 3. 覆盖上游路由
    ctx.SetUpstreamOverrideHost(lockedEndpoint)

    // 4. 记录上下文（用于 retry 和状态跟踪）
    ctx.SetContext(ctxLockedEndpoint, lockedEndpoint)
    ctx.SetContext(ctxGroupName, groupName)

    return types.ActionContinue
}
```

**关键点**：
- **零外部调用**：没有 Redis/Consul/Nacos 查询（locked 通过 blocking query 订阅到本地缓存）
- **纯本地内存**：O(n) 复杂度，n = 每个 Group 的 endpoint 数量
- **延迟**：与 V5.1 相同（<1μs）
- **粘滞性**：`locked` 健康时，高优先级节点恢复也不会触发回切

---

## 9.1 被动重试流程（双阶段）

即使所有实例的健康视图一致，请求到达时仍可能遇到以下情况：
- 后端在探测间隙内刚刚故障（上一次探测显示 HEALTHY，但实际已不可用）
- 后端处于"亚健康"状态（偶发 503，但尚未被探测到）
- 网络瞬时抖动导致单次请求失败

此时需要**请求级别的即时重试**来挽救当前请求，而不是等待下一次健康探测周期。

### 9.1.1 重试触发条件

```go
func onHttpResponseHeaders(ctx wrapper.HttpContext, config Config, log log.Log) types.Action {
    statusCode, err := proxywasm.GetHttpResponseHeader(":status")
    if err != nil || len(statusCode) == 0 {
        return types.ActionContinue
    }
    
    code := 0
    fmt.Sscanf(statusCode, "%d", &code)
    if code < 500 {
        return types.ActionContinue
    }
    
    groupName := ctx.GetStringContext(ctxGroupName, "")
    lockedEndpoint := ctx.GetStringContext(ctxLockedEndpoint, "")
    retryCount := ctx.GetIntContext(ctxRetryCount, 0)
    group := config.GetGroup(groupName)
    
    if group == nil || retryCount >= config.MaxRetries {
        return types.ActionContinue  // 超过最大重试次数，返回错误
    }
    
    // 即时探测通过下一次 tick 自然触发，不在此处 spawn goroutine
    
    // 第1阶段：同 endpoint 重试（排除瞬时抖动）
    if retryCount == 0 {
        ctx.SetContext(ctxRetryCount, 1)
        sendRetryToEndpoint(ctx, lockedEndpoint, config, func(retryStatus int) {
            if retryStatus == 200 {
                // 同 endpoint 重试成功，说明是瞬时故障
                proxywasm.ResumeHttpResponse()
                return
            }
            // 同 endpoint 仍失败，进入第2阶段
            tryNextEndpoint(ctx, group, lockedEndpoint, config)
        })
        return types.ActionPause
    }
    
    // 第2阶段：转移到下一个 endpoint
    tryNextEndpoint(ctx, group, lockedEndpoint, config)
    return types.ActionPause
}

// tryNextEndpoint 按 FailoverPolicy 选择下一个 endpoint
func tryNextEndpoint(ctx wrapper.HttpContext, group GroupConfig, currentLocked string, config Config) {
    // 从本地缓存读取最新健康视图（可能已有更新的 locked）
    nextEndpoint := electNewLocked(group, p.healthCache, currentLocked)
    sendRetryToEndpoint(ctx, nextEndpoint, config, func(retryStatus int) {
        if retryStatus == 200 {
            proxywasm.ResumeHttpResponse()
        } else {
            // 重试也失败，返回错误
            proxywasm.SendHttpResponse(503, nil, []byte("all endpoints unavailable"), -1)
        }
    })
}
```

### 9.1.2 失败时的本地状态标记

被动重试只解决"当前请求不失败"，还需要让**后续请求**尽快避开这个 endpoint。但 WASM 插件中**不能启动后台 goroutine**，所以请求路径上无法直接发起一次异步健康探测并立即把结果发布到后端。

能做的只是在请求路径上**立即更新本地 `HealthCache`**，把该 endpoint 标记为疑似不健康：

```go
func onRequestFailure(group, endpoint string) {
    // 1. 请求路径上立即更新本地缓存，标记该 endpoint 为疑似不健康
    //    后续请求在本 Pod 内会立刻避开它
    p.healthCache.UpdateReport(group, endpoint, ProbeReport{
        Source:  p.podName,
        Healthy: false,
        Ts:      time.Now().UnixMilli(),
    })

    // 2. 真正的探测和向后端发布，仍由 RegisterTickFunc 周期任务完成
    //    最多延迟一个 HealthCheckInterval（如 3s）
    //    WASM 中不能 spawn goroutine 来加速这一步
}
```

**关键点**：
- **请求路径上不发布到后端**——只做纯本地内存更新，零外部调用
- **后端同步依赖下一次 tick**——`RegisterTickFunc` 下一次触发时，会把本地报告发布到 Consul/Redis
- **当前 Pod 的后续请求立即受益**——本地缓存更新后，同 Pod 内下一请求就会重新选 endpoint
- **其他 Pod 需要等待 Pub/Sub 传播**——通常 0-3s（Consul blocking query 接近 0）

**如果下一次 tick 探测时 endpoint 已经恢复了呢？**

这是典型场景——单次 5xx 往往只是网络抖动或 GC 暂停，endpoint 在 3s 内已自愈：

- 下一次 tick 的主动探测返回 `HEALTHY`
- `LocalHealthCache.UpdateReport` 按 `Source` 去重：同一 Pod 的新 `HEALTHY` 报告会覆盖之前请求路径上写入的 `UNHEALTHY` 报告
- 本 Pod 在下次探测后就会重新认为该 endpoint 健康，误判时间最多是一个 `HealthCheckInterval`

如果不去重，旧的 `UNHEALTHY` 报告会在窗口内（如 60s）一直触发 fail-fast，导致本 Pod 长达冷却期的误判。因此 `upsertBySource` 是必要的实现细节。

**那么，本 Pod 的临时 UNHEALTHY 标记会不会和订阅来的全局状态分叉？**

会，但这是**有意的、有界的**本地 fail-fast。考虑这个场景：

```
T0    Pod-A 请求失败 → 本地写入 Source=Pod-A, UNHEALTHY
T0+1s Pod-B 主动探测 HEALTHY → 发布到后端
T0+1s Pod-A 订阅到 Pod-B 的 HEALTHY 报告
```

此时 Pod-A 的本地缓存里同时存在：
- `{Source=Pod-A, Healthy=false}`（来自请求路径的本地标记）
- `{Source=Pod-B, Healthy=true}`（来自订阅的全局状态）

按 fail-fast 规则，只要有任一 `UNHEALTHY` 就判定为 `UNHEALTHY`，所以 **Pod-A 会继续认为该 endpoint 不健康**，而 Pod-B（以及收到 Pod-B 报告的其他 Pod）认为它健康。两方的视图暂时分叉了。

这个分叉的持续时间**最多是一个 `HealthCheckInterval`**（如 3s），因为 Pod-A 自己的下一次主动探测会给出 definitive 结论：
- 探测成功 → `HEALTHY` 覆盖本地 `UNHEALTHY` → Pod-A 重新与全局一致
- 探测失败 → Pod-A 确认问题真实存在，向后端发布 `UNHEALTHY` → 全局收敛到一致

**为什么接受这个分叉？**
- 请求失败是第一手观察，比订阅状态更及时
- 短暂的本地保守可以避免后续请求继续打到可能不稳定的 endpoint 上
- 代价只是本 Pod 在最多 3s 内把流量切到备用 endpoint，其他 Pod 仍走主 endpoint；这会导致短暂的 cache tree 分散，但比继续向疑似故障节点发送请求更安全
- 如果希望完全避免这种分叉，可以选择不在请求路径上更新 `HealthCache`，代价是失去这 3s 内的本地自我保护

**设计取舍**：V5.2 选择保留本地标记，但明确把它定义为"有界分叉的本地 fail-fast"，并在 `UpdateReport` 中通过 source 去重把分叉窗口限制在 `HealthCheckInterval` 内。

### 9.1.3 与主动健康检查的关系

| 维度 | 主动健康检查 | 被动重试 |
|------|-------------|---------|
| 触发时机 | 周期性 tick（如每 3s） | 请求返回 5xx 时即时触发 |
| 响应速度 | 秒级 | 毫秒级 |
| 作用范围 | 影响后续所有请求的路由决策 | 挽救当前失败的请求 |
| 探测目标 | 所有 endpoint | 仅当前失败的 endpoint |
| 状态同步 | 发布到后端，影响所有 Pod | 仅更新本地缓存；后端同步由下一次 tick 探测完成 |

**两层防护**：
- **被动层（毫秒级）**：请求失败 → 即时重试 → 当前请求不失败
- **主动层（秒级）**：探测周期 → 状态同步 → 后续请求路由到正确后端

### 9.1.4 为什么 V5.2 仍需要被动重试

分布式一致性动态路由解决了"所有实例对后续请求做出相同决策"的问题，但不能解决"单个请求到达瞬间后端刚好故障"的问题。原因：

1. **探测间隙**：即使探测周期是 3s，两次探测之间有 3s 的盲区。后端可能在盲区内故障。
2. **最终一致性**：Pub/Sub 传播有 1-3s 延迟。Pod-A 已经知道 A 故障，但 Pod-B 的缓存还没更新。
3. **缓存 TTL**：本地缓存有 1-2s TTL，缓存过期前即使后端已故障，实例仍可能路由到该后端。

被动重试是这三类场景的**兜底机制**——即使路由决策"错误"了，请求仍有机会在毫秒级被挽救。

---

## 10. 健康检查流程

### 10.1 周期性探测（每个 Pod 独立执行）

```go
func (p *Plugin) startHealthCheck() {
    // 使用 RegisterTickFunc 进行周期性探测（WASM 中不能使用 goroutine）
    wrapper.RegisterTickFunc(p.config.HealthCheckInterval, func() {
        now := time.Now().UnixMilli()

        for _, group := range p.config.Groups {
            for _, ep := range group.Endpoints {
                // 同步触发探测（HttpClient 本身是异步回调模式）
                p.probeEndpoint(group.Name, ep.Name, ep.Address, now)
            }
        }
    })
}

func (p *Plugin) probeEndpoint(group, endpoint, address string, ts int64) {
    // 发送 HTTP GET {address}/readiness
    // HttpClient 使用 DispatchHttpCall，回调在 host 触发时执行
    p.httpClient.Get(address + "/readiness", nil, func(statusCode int, headers http.Header, body []byte) {
        healthy := statusCode == 200

        // 1. 更新本地缓存
        p.healthCache.UpdateReport(group, endpoint, ProbeReport{
            Source:  p.podName,
            Healthy: healthy,
            Ts:      ts,
        })

        // 2. 发布到后端（通过 HttpClient 异步发送，不阻塞）
        p.backend.PublishProbeResult(group, endpoint, healthy, ts)

        // 3.【新增】如果探测的是当前 locked endpoint，且它不健康了，触发 CAS 重新选举
        locked, _, lockedIdx := p.healthCache.GetLockedEndpoint(group)
        if endpoint == locked && !healthy {
            groupConfig := p.getGroupConfig(group)
            newLocked := electNewLocked(groupConfig, p.healthCache, locked)
            
            success, winner, _ := p.backend.CasWriteLocked(group, newLocked, lockedIdx)
            if !success && winner != nil {
                // CAS 失败：采用获胜者的值
                p.healthCache.UpdateLockedEndpoint(group, winner.Endpoint, winner.LockedAt, winner.ModifyIndex)
            } else if success {
                // CAS 成功：更新本地缓存
                p.healthCache.UpdateLockedEndpoint(group, newLocked, ts, lockedIdx+1)
            }
        }
    })
}
```

### 10.2 增量更新接收

**Health 更新**：
```go
func (p *Plugin) handleHealthUpdate(update HealthUpdate) {
    // 忽略本 Pod 自己发布的更新（避免重复处理）
    if update.Report.Source == p.podName {
        return
    }

    // 更新本地缓存
    p.healthCache.UpdateReport(update.Group, update.Endpoint, update.Report)

    log.Debugf("Received health update from %s: group=%s endpoint=%s healthy=%v",
        update.Report.Source, update.Group, update.Endpoint, update.Report.Healthy)
}
```

**Locked 更新**（来自 blocking query 订阅）：
```go
func (p *Plugin) handleLockedUpdate(update LockedUpdate) {
    // 更新本地 locked 缓存
    p.healthCache.UpdateLockedEndpoint(update.Group, update.Endpoint, update.LockedAt, update.ModifyIndex)

    log.Debugf("Received locked update: group=%s endpoint=%s lockedAt=%d",
        update.Group, update.Endpoint, update.LockedAt)
}
```

---

## 11. 跨生命周期恢复

### 11.1 新 Pod 启动流程

```
Pod-D 启动（ Envoy + WASM VM 初始化）
  |
  |-- 1. 读取 WasmPlugin CRD 配置（Group 列表、endpoint 地址、优先级）
  |
  |-- 2. 创建 TopologyBackend（根据 backendType）
  |
  |-- 3. BootstrapFullState()
  |     -> 从后端读取所有 Group 的最近探测报告
  |     -> 读取所有 Group 的 locked endpoint
  |     -> 填充 LocalHealthCache（health + locked）
  |     -> 此时已有完整的健康视图和锁定状态
  |
  |-- 4. SubscribeHealthUpdates() + SubscribeLockedUpdates()
  |     -> 通过 blocking query 订阅 health 和 locked 的增量更新
  |
  |-- 5. 开始接收请求
  |     -> 第一个请求到达时，healthCache 已有数据
  |     -> selectLockedEndpoint 立即做出正确决策（粘滞到 locked）
  |
  |-- 6. startHealthCheck()
  |     -> 开始独立探测，发布结果
  |     -> locked 故障时触发 CAS 重新选举
```

### 11.2 进程重启场景

| 场景 | 行为 |
|------|------|
| Envoy OOM 后重启 | 新 VM Bootstrap 从后端恢复状态，几秒内恢复一致性 |
| Higress 升级 | Pod 滚动更新，新 Pod Bootstrap，旧 Pod 优雅退出 |
| HPA 扩容 | 新 Pod 加入，Bootstrap 后立即可用 |
| HPA 缩容 | 被缩容 Pod 的探测报告因 TTL 过期自动清理 |

---

## 12. 故障模式矩阵

| 场景 | 行为 | 影响 |
|------|------|------|
| **后端暂时不可用** | 使用本地缓存。继续本地探测。缓存有过期时间（30s），过期后状态变 UNKNOWN，保守视为 HEALTHY（避免误杀） | 最多 30s 内可能与其他 Pod 不一致 |
| **新 Pod 启动时后端不可用** | 无法 Bootstrap，回退到 V5.1 行为（独立本地探测 + 独立锁定） | 该 Pod 与其他 Pod 不一致，直到后端恢复 |
| **单 Pod 与后端网络分区** | 该 Pod 只能依赖本地探测。其他 Pod 仍通过后端同步 | 分区 Pod 可能滞后于全局状态 |
| **单 Pod 与部分 endpoint 网络分区** | 该 Pod 报告 UNHEALTHY，通过后端传播。其他 Pod 收到后 fail-fast 标记为 UNHEALTHY | 全局一致地将流量切走，避免单 Pod 误判导致不一致 |
| **后端永久丢失** | 系统退化为 V5.1。每个 Pod 独立探测和锁定 | 功能正常，但失去跨 Pod 一致性保证 |
| **locked endpoint 故障，多 Pod 同时 CAS** | 多个 Pod 同时探测到 locked 失败，同时 CAS 写入新 locked。一个成功，其余采用成功者值 | 短暂竞争（毫秒级），最终所有 Pod 切换到同一 endpoint |
| **locked 写入后订阅延迟** | Pod-A 已更新 locked，Pod-B 的 blocking query 尚未返回 | 最多 Consul 的 `wait` 时间（默认 30s，实际通常 <1s）内 Pod-B 仍路由到旧 locked。被动重试兜底 |
| **locked 被人工修改为已故障 endpoint** | 运维误操作，将 locked 设为 UNHEALTHY 的 endpoint | `selectLockedEndpoint` 检测到 locked 不健康 → 触发重新选举 → 自动修正 |
| **新 Pod 启动，locked 指向已恢复的高优先级节点** | locked 后端值为 1，但 1 曾故障后恢复，当前实际在 5 上运行 | 新 Pod 读取 locked=1，1 是 HEALTHY → 路由到 1。这是**正确行为**——locked 是权威路由决策 |
| **请求路径本地标记与订阅状态分叉** | Pod-A 请求失败后本地标记 UNHEALTHY，但同时订阅到其他 Pod 的 HEALTHY 报告。fail-fast 使 Pod-A 暂时保持 UNHEALTHY，直到本 Pod 下一次主动探测给出 definitive 结论 | 最多一个 `HealthCheckInterval`（如 3s）内 Pod-A 与其他 Pod 视图不一致；本 Pod 流量切到备用 endpoint，其他 Pod 仍走主 endpoint |
| **探测频率不一致** | 部分 Pod 3s 探测，部分 Pod 5s 探测。时间窗口过滤保证只使用最近报告 | 不影响一致性，只影响故障发现速度 |
| **时钟漂移** | Pod 间时钟差几秒。时间窗口过滤（60s）可容忍小幅漂移 | 小幅漂移不影响，大幅漂移可能导致聚合异常 |

---

## 13. K8s 部署配置

### 13.1 Consul Server 集群部署

**推荐方式：Helm（最简单）**

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install consul hashicorp/consul \
  --namespace higress-system \
  --create-namespace \
  --set global.name=consul \
  --set server.replicas=3 \
  --set server.storage=1Gi \
  --set ui.enabled=false \
  --set connectInject.enabled=false \
  --set client.enabled=false
```

- `server.replicas=3`：Raft 共识需要奇数个节点
- `client.enabled=false`：Gateway Pod 直接通过 Service 访问 server 的 HTTP API（8500 端口），不需要 client agent
- `connectInject.enabled=false`：不需要 Consul Connect Service Mesh

**原生 YAML 方式（可选）**

```yaml
# consul-server.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: consul
  namespace: higress-system
spec:
  serviceName: consul
  replicas: 3
  selector:
    matchLabels:
      app: consul
  template:
    metadata:
      labels:
        app: consul
    spec:
      containers:
      - name: consul
        image: hashicorp/consul:1.17
        env:
        - name: POD_IP
          valueFrom: { fieldRef: { fieldPath: status.podIP } }
        args:
          - agent
          - -server
          - -bootstrap-expect=3
          - -data-dir=/consul/data
          - -bind=0.0.0.0
          - -client=0.0.0.0
          - -advertise=$(POD_IP)
          - -retry-join=consul-0.consul.higress-system.svc.cluster.local
          - -retry-join=consul-1.consul.higress-system.svc.cluster.local
          - -retry-join=consul-2.consul.higress-system.svc.cluster.local
          - -datacenter=dc1
        ports:
        - containerPort: 8300
        - containerPort: 8301
        - containerPort: 8302
        - containerPort: 8500
        - containerPort: 8600
        resources:
          requests: { memory: "256Mi", cpu: "100m" }
          limits:   { memory: "512Mi", cpu: "200m" }
        volumeMounts:
        - name: data
          mountPath: /consul/data
  volumeClaimTemplates:
  - metadata: { name: data }
    spec: { accessModes: ["ReadWriteOnce"], resources: { requests: { storage: 1Gi } } }

---
# Headless service：为 StatefulSet Pod 提供稳定的 DNS 名称（consul-0.consul, consul-1.consul, ...）
apiVersion: v1
kind: Service
metadata:
  name: consul
  namespace: higress-system
spec:
  clusterIP: None
  selector:
    app: consul
  ports:
  - port: 8300
    targetPort: 8300
    name: serflan

---
# ClusterIP service：Gateway Pod 通过此 Service 访问 Consul HTTP API
apiVersion: v1
kind: Service
metadata:
  name: consul-server
  namespace: higress-system
spec:
  selector:
    app: consul
  ports:
  - port: 8500
    targetPort: 8500
    name: http
```

**资源需求**：3 个 server 节点，每个 256Mi-512Mi 内存。health check 的 KV 数据量很小（通常 <1MB），资源主要用于 Raft 共识开销。

### 13.2 WasmPlugin CRD 更新

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: smg-ha
  namespace: higress-system
spec:
  url: oci://registry/smg-ha-plugin:v5.2
  phase: AUTHZ
  matchRules:
  - ingress: smg-gateway
  defaultConfig:
    # 后端配置
    backendType: "consul"
    consul:
      address: "consul-server.higress-system.svc.cluster.local:8500"
      # token: ""    # 如有 ACL 启用则填写
      # datacenter: "dc1"

    # 健康检查配置
    healthCheckInterval: 3000       # 3s
    healthCheckTimeout: 2000        # 2s
    probeWindow: 60000              # 60s 聚合窗口（必须 >= failbackCooldown）

    # Group 配置
    groups:
    - name: group-1
      failoverPolicy: "sequential"    # sequential（默认）或 priority
      endpoints:
      - name: smg-1.1
        address: smg-1-1.smg-system.svc.cluster.local:30000
        priority: 100
      - name: smg-1.2
        address: smg-1-2.smg-system.svc.cluster.local:30000
        priority: 90
      - name: smg-1.3
        address: smg-1-3.smg-system.svc.cluster.local:30000
        priority: 80
```

---

## 14. 与 V5.1 的详细对比

| 维度 | V5.1 | V5.2 |
|------|------|------|
| **核心机制** | SharedData + VM lease | Pub/Sub + 确定性共识 |
| **跨 Pod 一致性** | ❌ 无（每个 Pod 独立） | ✅ 有（同步健康视图） |
| **跨生命周期** | ❌ 丢失（进程重启后重置） | ✅ 持久（后端存储） |
| **请求路径延迟** | ~1μs（本地缓存命中） | ~1μs（本地缓存命中） |
| **请求路径外部调用** | 0 | **0** |
| **主动 Failover 速度** | 1-3s（lease 切换 + SharedData 传播） | **1-3s**（探测周期 + Consul 推送）；最坏情况约 4-5s（探测间隙 3s + 缓存 TTL 1-2s）。Consul blocking query 在数据变化时立即推送，无 Redis 短轮询的 0-3s 延迟 |
| **分布式锁** | 单进程内 lease | **不需要**（CAS 乐观写） |
| **额外组件** | 无 | Consul server 集群（3 节点） |
| **代码复杂度** | 中等（lease、CAS、缓存失效） | 中等（backend 抽象、聚合逻辑、CAS 选举） |
| **部署复杂度** | 低（纯插件） | 中（需部署后端） |
| **粘滞/回切策略** | 无（VM lease 自动切换） | **Sequential 粘滞**（locked 保持，高优先级恢复不回切）+ **人工回切** |
| **优雅降级** | N/A | ✅ 后端故障时自动回退到 V5.1 行为 |
| **适用场景** | 单实例 Gateway | **多实例 Gateway（生产环境标准）** |

---

## 15. 风险与未验证项

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| Consul 集群运维 | 3 节点 Raft 集群需要基本的运维知识 | 使用 Helm 部署，官方提供完善的运维文档 |
| Consul 写入性能 | Raft 共识的写入延迟高于 Redis | health check 写入频率很低（每秒几次），不构成瓶颈 |
| WASM HTTP 并发限制 | 大量 Group 同时发探测请求，可能触发 WASM HTTP 并发限制 | 控制并发数，使用连接池（如果 SDK 支持） |
| 时钟漂移导致聚合异常 | Pod 间时钟差 >10s | NTP 同步是 K8s 基础设施的基本要求 |
| 首次启动 Bootstrap 延迟 | 大量 Group 时全量读取可能耗时 | 异步 Bootstrap，先订阅再补全 |
| 僵尸 KV 积累 | 缩容后失联 Pod 的 key 不会自动删除 | 依靠 LocalHealthCache 的时间窗口过滤（2× probeInterval）自动排除；定期清理脚本可选 |

---

## 16. 演进路线

### Phase 1: Consul 实现（默认）
- 实现 `ConsulBackend`（原生 HTTP blocking query，监听 `/smg/` 前缀同时订阅 health 和 locked）
- 实现 `LocalHealthCache` + 粘滞锁定算法（`selectLockedEndpoint` + `electSequential`/`electPriority`）
- 实现 `CasWriteLocked` + `GetLocked`（Consul KV CAS）
- 实现健康检查 + locked 故障检测 + CAS 重新选举
- 实现增量发布（health reports）+ 增量接收（health + locked）
- 提供 Consul K8s 部署配置（Helm + 原生 YAML）

### Phase 2: Nacos 实现（可选）
- 实现 `NacosBackend`（HTTP long polling）
- 适合已有 Nacos 的环境

### Phase 3: Redis 实现（备选）
- 实现 `RedisBackend`（通过 Higress 自定义 `proxy_redis_call` host 函数，原生 RESP 协议）
- 不需要 HTTP bridge，直接通过 `wrapper.RedisClient` 访问
- 适合已有 Redis 的环境

### Phase 4: 优化
- 批量发布探测结果（减少 HTTP 请求数）
- 增量 Bootstrap（只读取变化的 Group）
- 多 Consul datacenter 支持

---

## 附录：参考来源

以下源码和文档在 V5.2 调研过程中被直接阅读或引用。所有结论均基于这些一手资料推导，未经独立实验验证。

### Higress 源码与配置

1. **Higress Gateway 默认副本数**
   - 来源：`higress/helm/higress/README.md` 第 130 行
   - 内容：`gateway.replicas | int | 2 | Number of Higress Gateway pods`
   - 用途：确认生产环境默认多实例部署

2. **Higress Helm 部署模板**
   - 来源：`higress/helm/core/templates/deployment.yaml` 第 28 行
   - 内容：`replicas: {{ .Values.gateway.replicas }}`
   - 用途：确认 replicas 参数实际控制 Gateway Pod 数量

3. **ai-proxy failover 的 SharedData 与 lease 实现**
   - 来源：`higress/plugins/wasm-go/extensions/ai-proxy/provider/failover.go`
   - 关键代码：
     - `ctxVmLease = provider + "-" + id + "-vmLease"`（第 131 行）
     - `tryAcquireOrRenewLease(vmID)` 的 CAS 竞争逻辑（第 292-341 行）
     - `generateVMID()` 返回 `uuid.New().String()`（第 343-345 行）
     - `resetSharedData()` 在配置更新时清除所有状态（第 685-691 行）
   - 用途：确认 ai-proxy 的 lease 是 per-VM 的，多 Pod 场景下每个 Pod 独立竞争

4. **ai-proxy health check 触发逻辑**
   - 来源：`higress/plugins/wasm-go/extensions/ai-proxy/provider/failover.go` 第 156-220 行
   - 内容：`RegisterTickFunc` 中 lease-holder 执行 health check，通过 `wrapper.NewClusterClient` 发送探测请求
   - 用途：验证 WASM 插件可以发起异步 HTTP 健康探测

5. **http-call 扩展的 HTTP client 用法**
   - 来源：`higress/plugins/wasm-go/extensions/http-call/main.go`
   - 内容：展示了 `wrapper.NewClusterClient` 支持 `K8sCluster`、`StaticIpCluster`、`DnsCluster`、`NacosCluster` 四种服务发现模式
   - 用途：验证 WASM 插件可以通过多种方式访问外部 HTTP 服务

6. **ai-cache 向量数据库连接**
   - 来源：`higress/plugins/wasm-go/extensions/ai-cache/vector/qdrant.go`
   - 内容：使用 `wrapper.NewClusterClient(wrapper.FQDNCluster{FQDN: ..., Host: ..., Port: ...})` 连接 Qdrant
   - 用途：验证 WASM 插件连接外部数据库的模式

7. **ai-search 搜索引擎连接**
   - 来源：`higress/plugins/wasm-go/extensions/ai-search/engine/google/google.go`、`bing/bing.go` 等
   - 内容：通过 `wrapper.HttpClient` 访问 Google/Bing 等外部搜索 API
   - 用途：验证 WASM 插件可以访问公网 HTTP 服务

### proxy-wasm 规范

8. **proxy-wasm spec 对 SharedData scope 的定义**
   - 来源：proxy-wasm/spec 文档（通过 Higress 实现行为推断）
   - 内容：SharedData 的 scope 被定义为 implementation-defined；Higress/Envoy 的实现中为单个 Envoy 进程级别
   - 用途：确认 V5.1 的 SharedData 不跨 Pod 共享

### 外部项目与协议

9. **Consul KV HTTP API**
   - 来源：https://developer.hashicorp.com/consul/api-docs/kv
   - 内容：`PUT /v1/kv/{key}` 写入，`GET /v1/kv/{key}?recurse=true` 前缀读取
   - 用途：设计 Consul KV 数据模型和发布机制

10. **Consul Blocking Queries**
    - 来源：https://developer.hashicorp.com/consul/api-docs/features/blocking
    - 内容：`?index=`（ModifyIndex）+ `?wait=`（最长等待时间）机制，响应在数据变化或超时后返回
    - 用途：设计订阅机制，确认与 proxy-wasm 异步 HTTP 模型的适配性

11. **Consul Sessions**
    - 来源：https://developer.hashicorp.com/consul/docs/dynamic-app-config/sessions
    - 内容：Session TTL、自动释放关联 KV 机制
    - 用途：评估失联 Pod 报告自动清理方案

12. **Higress 自定义 host 函数：proxy_redis_init / proxy_redis_call**
    - 来源：`higress/plugins/wasm-go/extensions/response-cache/cache/redis.go`；`github.com/higress-group/wasm-go/pkg/wrapper/redis_wrapper.go`；`github.com/higress-group/proxy-wasm-go-sdk/proxywasm/hostcall.go`
    - 内容：Higress 在 proxy-wasm-go-sdk fork 中增加了 `proxy_redis_init` 和 `proxy_redis_call` 两个自定义 host 函数。wasm-go 框架据此提供了 `wrapper.RedisClient` 接口，支持 `Init`、`Command`、`Get`、`Set`、`HGet`、`HSet` 等操作，底层通过 RESP 协议直接与 Redis 通信。
    - 用途：确认 WASM 插件可以**直接访问 Redis**，不需要 HTTP bridge。这是 Redis 后端不需要 Go bridge 的关键依据。

13. **Higress 自定义 host 函数清单（无 streaming HTTP 扩展）**
    - 来源：`github.com/higress-group/proxy-wasm-go-sdk/proxywasm/internal/abi_hostcalls.go`
    - 内容：Higress 增加的自定义 host 函数包括 `proxy_set_upstream_override_host`、`proxy_get_upstream_hosts`、`proxy_inject_encoded_data_to_filter_chain`、`proxy_redis_init`、`proxy_call_foreign_function` 等。**没有增加 streaming HTTP client 能力**。
    - 用途：确认 etcd/K8s API Server Watch 所需的 streaming HTTP/gRPC 出站调用在 Higress 扩展能力范围内仍然不可行。

14. **Envoy request_mirror_policies**
    - 来源：Envoy 官方文档 + V7 调研中对 `cache_aware.rs` 和 `tree.rs` 的源码分析
    - 用途：V7 已证伪镜像流量方案，确认 cache tree 分散无法通过流量复制解决

### SMG 源码

13. **sgl-model-gateway cache tree 实现**
    - 来源：`sglang/sgl-model-gateway/src/cache_aware.rs`、`tree.rs`
    - 关键发现（V7 调研）：tree 构建过程中存在 5 个随机/不确定环节（`rand::rng()`、`DashMap` 迭代顺序、概率性时间戳更新、非原子 node split）
    - 用途：确认镜像流量方案不可行，必须依赖路由一致性而非流量复制

