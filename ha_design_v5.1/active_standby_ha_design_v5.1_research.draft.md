# SMG Active-Standby HA 架构 V5.1 方案调研 — Higress 整体插件化能力深度评估

> 目标：在 V5 仅验证标准 proxy-wasm ABI 能力边界的基础上，进一步深入评估 Higress 生态提供的**整体插件化方案**——包括 `golang-filter`（native Go filter）和 `wasm-go` 框架——能否支持 SMG 主备 failover 的四个核心目标：主动感知拓扑、主动控制请求发送、失败时第一时间重试、后续请求正确路由到备用节点。
>
> 本方案基于对 Higress 官方仓库 `higress/plugins` 目录下源码的深度分析，以及 `ai-proxy`、`ai-load-balancer` 两个生产级插件的参考。

---

## 一、V5 的遗留问题

V5 的调研结论是：标准 proxy-wasm ABI 下，WASM 插件**无法 preemptive 切换 upstream cluster**（主不健康时直接避开），但**可以做被动重试**（主返回 5xx 后向备节点重新发起请求）。

这个结论的推导依据是：
- proxy-wasm/spec issue #16：`set_http_request_header` 修改的 header 只影响 upstream，Envoy Router 看不到
- envoyproxy/envoy issue #28673：`setProperty` 自动前缀 `wasm.`，无法设置 `envoy.router.cluster`

但 V5 的调研存在一个盲区：它只评估了**标准 proxy-wasm ABI** 的能力边界，没有评估 Higress 在标准 ABI 之上**自行扩展的额外能力**。Higress 的 `wasm-go` 框架和 `golang-filter` 机制是否提供了超出标准 ABI 的路由控制能力，是 V5.1 需要回答的核心问题。

---

## 二、Higress 插件体系概览

Higress 提供两种插件开发路线：

| 维度 | `golang-filter`（Native Go Filter） | `wasm-go`（WASM 插件） |
|------|-----------------------------------|----------------------|
| 运行时 | Envoy 进程内原生 Go（CGO） | WebAssembly VM（WASI） |
| 编译产物 | `.so` 共享库 | `.wasm` 字节码 |
| 部署方式 | 随网关镜像发布 / 挂载文件 | `WasmPlugin` CRD + OCI 镜像 |
| Go 标准库 | **完整访问**（goroutines、net/http、SQL 驱动等） | **受限**（无原生 goroutine、无直接 syscall） |
| 热更新 | 困难（需重启 Envoy 或复杂 SO 重载） | 容易（CRD 更新即可） |
| 隔离性 | 低（插件崩溃可能影响 Envoy） | 高（沙箱隔离） |
| 性能 | 高（无 VM 边界开销） | 较低（WASM VM 边界序列化开销） |

两种路线共享同一个核心诉求：在 Envoy HTTP Filter 链中拦截请求/响应，执行自定义逻辑。但它们能调用的 API 和能力边界有显著差异。

---

## 三、golang-filter（Native Go Filter）

`golang-filter` 是 Envoy 官方 Golang HTTP Filter 贡献的 Higress 定制版。Go 代码通过 `-buildmode=c-shared` 编译为 `.so`，Envoy 通过 CGO 调用 Go 的 filter 回调。

### 3.1 核心 API 能力

Filter 实现 `api.StreamFilter` 接口：

```go
type StreamFilter interface {
    StreamDecoderFilter   // DecodeHeaders, DecodeData, DecodeTrailers
    StreamEncoderFilter   // EncodeHeaders, EncodeData, EncodeTrailers
    OnLog(...)
    OnDestroy(DestroyReason)
}
```

**请求/响应操作：**
- `header.Get/Set/Add/Del/Range` — 修改 headers/trailers
- `header.SetMethod/SetHost/SetPath` — 修改伪 headers（`:method`、`:authority`、`:path`）
- `buffer.Set/Append/Prepend/Reset/Bytes` — 修改请求/响应 body
- `callbacks.SendLocalReply(code, body, headers, grpcStatus, details)` — 返回本地响应
- `callbacks.InjectData(data)` — 向流中注入数据（SSE 场景使用）

**路由控制：**
- `callbacks.ClearRouteCache()` — 清除路由缓存，使 Envoy 在后续 filter 中重新评估路由
- `callbacks.RefreshRouteCache()` — 清除并立即重新获取路由
- `header.SetPath()/SetHost()` — 修改 path/authority，影响路由评估

**状态共享：**
- `DynamicMetadata().Set/Get` — 单次请求内跨 filter 共享数据
- `FilterState().SetString/GetString` — 跨请求/连接/FilterChain 共享字符串状态（生命周期可配置）

**关键限制：**
- **无直接 cluster 选择 API**：不能调用 `SetRouteCluster("cluster-b")` 之类的 API，只能通过修改伪 headers + RefreshRouteCache 间接影响路由
- **无内置 async HTTP dispatch**：没有等价于 proxy-wasm `DispatchHttpCall` 的 API。需要用标准 Go `net/http` 在 goroutine 中做阻塞调用
- **FilterState 只能存字符串**：复杂状态需要借助外部存储（Redis）或全局变量

### 3.2 对 failover 的评估

| 能力 | 支持情况 | 说明 |
|------|---------|------|
| 主动健康检查 | ⚠️ 需自研 | 无内置 API，需用 goroutine + `net/http` 实现 |
| preemptive 切换 cluster | ⚠️ 间接 | 只能修改 `:authority`/`:path` + `RefreshRouteCache`，无法直接指定 cluster |
| 被动重试（5xx 后重发） | ⚠️ 需自研 | 无 `DispatchHttpCall`，需在 goroutine 中做阻塞 HTTP 调用 |
| 跨请求状态共享 | ✅ 部分 | `FilterState`（字符串）+ 全局变量 + 外部存储 |
| SSE 流式响应处理 | ✅ | `InjectData` 支持向流中注入数据 |

golang-filter 的优势是性能高、Go 生态完整，但缺点是路由控制能力有限、缺少 async HTTP dispatch、部署灵活性差。对于需要精细控制 upstream 路由的 SMG failover 场景，能力不够充分。

---

## 四、wasm-go 框架深度分析

`wasm-go` 是 Higress 提供的 WASM Go 插件开发框架，基于 `proxy-wasm-go-sdk`，但在标准 ABI 之上封装了大量高阶能力。这是 V5.1 调研的核心。

### 4.1 标准 proxy-wasm ABI 回顾

标准 ABI 提供的能力：
- HTTP 请求/响应 header/body 读写
- `DispatchHttpCall` — 向指定 cluster 发起异步 HTTP 调用
- `GetSharedData` / `SetSharedData` — 跨请求键值存储
- `SetTickPeriodMilliSeconds` — 定时 tick
- `GetProperty` / `SetProperty` — 读写 Envoy 属性
- `SendHttpResponse` — 向客户端返回本地响应

标准 ABI 的已知限制（V5 已验证）：
- 修改的 header 在 Router 之后执行，**不能影响 routing 决策**
- `SetProperty` 自动前缀 `wasm.`，不能设置 `envoy.router.cluster`

### 4.2 Higress wasm-go 框架的扩展能力

Higress 在标准 ABI 之上，通过自定义 host functions 和 wrapper 封装，提供了以下**超出标准 ABI** 的能力：

#### 4.2.1 `SetUpstreamOverrideHost` — 直接指定 upstream endpoint

```go
proxywasm.SetUpstreamOverrideHost([]byte("10.0.0.1:30002"))
```

这个 API 允许插件**直接覆盖** Envoy 为当前 cluster 选中的 upstream endpoint，将请求固定发送到指定的 IP:port。`ai-load-balancer` 的 `endpoint_metrics`、`global_least_request`、`prefix_cache` 策略都使用了这个 API。

**对 failover 的意义**：在主节点故障时，可以直接将后续请求 override 到备节点的 endpoint，绕过 Envoy 的负载均衡逻辑。

#### 4.2.2 `x-higress-target-cluster` Header — 跨 Cluster 路由切换

```go
proxywasm.ReplaceHttpRequestHeader("x-higress-target-cluster", "smg-group-1-backup")
```

Higress 支持通过 `cluster_header` 路由机制，让 Envoy 根据请求 header 的值来选择 upstream cluster。`ai-load-balancer` 的 `cluster_metrics` 和 `cluster_hash` 策略正是通过这个 header 实现 cluster 级别的路由控制。

**对 failover 的意义**：在需要跨 cluster 切换的场景中（如将流量从一个 SMG Group 整体迁移到另一个 Group），可以通过设置该 header 实现 cluster 级别的 preemptive 切换。但在 SMG 主备架构中，外层 Higress 已通过 VirtualService 选定了 Group（对应一个 cluster），failover 发生在**同一 cluster 内的 endpoint 之间**，因此该能力不是本场景的主要机制。

#### 4.2.3 `RouteCall` — 完全重定向请求

```go
wrapper.RouteCall(method, url, headers, body, callback)
```

在请求 body 阶段（暂停 headers 后），可以完全替换请求的方法、URL、headers、body，并通过当前 route 的目标 cluster 重新发送。

**对 failover 的意义**：在被动重试场景中，可以用 `RouteCall` 将失败请求重新发送到备 endpoint。

#### 4.2.4 `wrapper.HttpCall` / `wrapper.NewClusterClient` — 高阶异步调用

```go
// 通用异步调用
wrapper.HttpCall(cluster, method, url, headers, body, callback, timeout)

// ClusterClient 封装
client := wrapper.NewClusterClient(wrapper.TargetCluster{Cluster: "smg-group-1-backup"})
client.Post(url, headers, body, callback, timeout)
```

框架提供了多种 cluster 类型：
- `RouteCluster{}` — 通过当前 route 的 cluster 发送
- `TargetCluster{Cluster: "name"}` — 指定 cluster 名称发送
- `K8sCluster{ServiceName: "...", Namespace: "...", Port: ...}` — 通过 K8s Service 发送
- `FQDNCluster{Host: "...", Port: ...}` — 通过 FQDN 发送

**对 failover 的意义**：`TargetCluster` 可以直接向备 endpoint 发起 health check 或重试请求。

#### 4.2.5 `RegisterTickFunc` — 高阶定时任务

```go
wrapper.RegisterTickFunc(5000, func() {
    // 每 5 秒执行一次
    // 可做 active health check
})
```

框架内部设置 100ms 基础 tick，根据注册函数的间隔分派执行。`ai-proxy` 的 health check 正是用这个机制实现的。

#### 4.2.6 `GetUpstreamHosts` — 获取 upstream 端点列表及健康状态

```go
hosts, _ := proxywasm.GetUpstreamHosts()
// hosts 包含 address、health_status、metrics 等信息
```

`ai-load-balancer` 的 endpoint 策略使用这个 API 过滤掉 `health_status != "Healthy"` 的端点。

**对 failover 的意义**：可以直接感知到当前 cluster 下所有 endpoint 的健康状态，为主备决策提供数据。

### 4.3 关键结论：preemptive 切换在 Higress wasm-go 中是可行的

V5 的结论——"WASM 无法 preemptive 切换 upstream cluster"——在标准 proxy-wasm ABI 下成立，但在 Higress 的 wasm-go 框架下**不再成立**。Higress 提供了至少三条路径实现 preemptive 切换：

| 路径 | 机制 | 适用场景 |
|------|------|---------|
| **路径 A**：`SetUpstreamOverrideHost` | `OnHttpRequestHeaders` 中根据健康状态直接指定 endpoint IP:port | 主备在同一 cluster 下是不同 endpoint（**本场景推荐**） |
| **路径 B**：`x-higress-target-cluster` header | `OnHttpRequestHeaders` 中设置 header，Envoy `cluster_header` 路由选择备 cluster | 主备是不同 cluster（跨 Group 切换） |
| **路径 C**：`RouteCall` | 完全替换请求并通过指定 cluster 重发 | 被动重试场景 |

这三条路径都需要配合 `SharedData` 来维护跨请求的健康状态，以及 `RegisterTickFunc` 来做主动健康检查。

---

## 五、ai-proxy 插件参考分析

`ai-proxy` 是 Higress 最核心的 AI 网关插件，负责将统一的开源 API 格式转换为各云厂商的私有协议。其内部实现了完整的 **apiToken 级别 failover + health check + retry** 机制，是验证 wasm-go 框架能力的最佳参考。

### 5.1 架构定位

`ai-proxy` 处理的是**单一 provider 下的多 apiToken  failover**，而不是多 provider / 多 cluster 的 failover。配置结构：

```json
{
  "providers": [
    { "id": "openai", "type": "openai", "apiTokens": ["sk-1", "sk-2"] }
  ],
  "activeProviderId": "openai"
}
```

每次请求只有一个 `activeProvider` 被选中，failover 发生在该 provider 的多个 `apiTokens` 之间。

### 5.2 Failover 机制（`provider/failover.go`）

**核心流程：**
1. 请求使用某个 apiToken 发送
2. 如果响应状态码匹配配置的错误模式（默认 `4.*`、`5.*`），该 token 的失败计数器 +1
3. 当失败次数达到 `failureThreshold`，该 token 从可用池中移除
4. 后续请求从剩余可用 token 中随机选择
5. 如果所有 token 都不可用，降级为随机选择一个不可用的 token（记录 warning）

**状态存储：** 使用 `proxywasm.SetSharedData` / `GetSharedData` 跨请求共享 token 的可用/不可用状态。

### 5.3 Active Health Check（`provider/failover.go`）

**Lease-based Leader Election：**
- 多个 WASM VM 实例中，只有一个通过 lease 机制获得 health check 执行权
- Lease 有效期 60 秒，过期后重新竞争
- 避免多个 VM 同时向同一个上游发送 health check 造成压力

**Health Check 流程：**
1. `RegisterTickFunc` 注册周期性 tick（默认 5000ms）
2. 获得 lease 的 VM 执行 health check
3. 使用 `wrapper.NewClusterClient(wrapper.TargetCluster{Cluster: healthCheckEndpoint.Cluster})` 向同一 upstream cluster 发送简单 chat completion 请求
4. 如果返回 HTTP 200，成功计数器 +1
5. 当成功次数达到 `successThreshold`，token 重新加入可用池

**CoolDown Recovery：** 作为 health check 的替代方案，token 在不可用后经过 `cooldownDuration` 自动恢复。

### 5.4 Retry 机制（`provider/retry.go`）

**核心流程：**
1. 请求失败后，`OnRequestFailed` 触发 `retryFailedRequest`
2. 将失败 token 从候选列表中移除
3. 从剩余可用 token 中随机选择一个
4. 使用保存的原始请求信息（`CtxRequestHost`、`CtxRequestPath`、`CtxRequestBody`）重构请求
5. 通过 `wrapper.NewClusterClient(wrapper.RouteCluster{})` 发送重试
6. 重试响应：如果 200，转换后返回客户端；如果仍失败，最多重试 `maxRetries` 次（默认 1）

**关键限制：** README 明确说明——"目前仅支持对非流式请求进行重试"。流式（SSE）请求的重试在实现上更复杂，因为响应已经开始发送。

### 5.5 Routing 修改模式（`main.go` + `provider/`）

`ai-proxy` 的 routing 修改遵循以下模式：
1. 各 provider 实现 `TransformRequestHeadersHandler`，设置 `:authority` 为 provider 域名
2. 根据 provider 能力重写 `:path`
3. 调用 `ctx.DisableReroute()` — **防止 header 修改后 Envoy 重新计算路由**

`DisableReroute` 是关键：如果没有这个调用，Envoy 可能在 header 修改后重新评估路由，导致请求被路由到意外的 cluster。

### 5.6 对 SMG failover 的借鉴

| ai-proxy 机制 | SMG failover 对应 |
|--------------|------------------|
| apiToken failover | Worker / endpoint failover |
| `SharedData` 维护可用池 | `SharedData` 维护主备健康状态 |
| Lease-based health check leader | 避免多 VM 重复探测 |
| `RegisterTickFunc` 定时探测 | 主动探测 SMG 主备健康 |
| `RouteCluster` 重试 | 向备 endpoint 重试失败请求 |
| `DisableReroute` | 防止 header 修改后路由重算 |

---

## 六、ai-load-balancer 插件参考分析

`ai-load-balancer` 是 Higress 专门为 LLM 推理场景设计的负载均衡插件，支持多种 AI-aware 调度策略。它对评估 SMG failover 的价值在于：展示了 Higress wasm-go 框架在**cluster 级别**和 **endpoint 级别**的路由控制能力。

### 6.1 两种负载均衡范围

| `lb_type` | 范围 | 控制机制 |
|-----------|------|---------|
| `endpoint` | 单个 pod/endpoint | `proxywasm.SetUpstreamOverrideHost()` 直接覆盖 upstream endpoint |
| `cluster` | 整个 upstream cluster | 修改 `x-higress-target-cluster` header，Envoy `cluster_header` 路由接管 |

### 6.2 Endpoint 级策略

- **`global_least_request`**：Redis 维护全局 inflight 请求计数，Lua 脚本原子读取-最小值-自增，选择 inflight 最少的 endpoint
- **`prefix_cache`**：SHA1 哈希 prompt 内容，Redis 维护前缀树，相同前缀路由到同一 endpoint 以最大化 KV cache 复用；无匹配时 fallback 到 `global_least_request`
- **`endpoint_metrics`**：读取 vLLM Prometheus `/metrics`，解析 `num_requests_running`、`gpu_cache_usage_perc` 等指标，执行 K8s Gateway API Inference Extension 风格的 filter chain 调度

**Health Status 过滤：** `global_least_request`、`prefix_cache`、`endpoint_metrics` 都显式调用 `proxywasm.GetUpstreamHosts()`，过滤掉 `health_status != "Healthy"` 的端点。

### 6.3 Cluster 级策略

- **`cluster_metrics`**：维护各 cluster 的滚动统计（并发请求数、TTFT、总延迟），按 `LeastBusy` / `LeastTotalLatency` / `LeastFirstTokenLatency` 模式选择最优 cluster。失败的请求会被惩罚（记录延迟翻倍）
- **`cluster_hash`**：FNV-1a 哈希 configurable header（默认 `x-mse-consumer`），映射到 100-slot 加权数组，实现一致性哈希路由

### 6.4 对 SMG failover 的借鉴

`ai-load-balancer` 证明了 wasm-go 框架可以：
1. **在请求到达时动态选择 upstream endpoint**（`SetUpstreamOverrideHost`）
2. **在请求到达时动态选择 upstream cluster**（`x-higress-target-cluster`）
3. **基于实时健康状态过滤不可用端点**（`GetUpstreamHosts` + `health_status`）
4. **基于 Redis 维护全局状态**（跨 VM、跨请求共享）

这些能力直接对应 SMG failover 的四个核心目标中的三个（主动感知、主动控制、后续请求正确路由）。被动重试可以借鉴 `ai-proxy` 的 `RouteCluster` 模式。

---

## 七、核心目标逐一评估

基于以上对 golang-filter、wasm-go 框架、ai-proxy、ai-load-balancer 的分析，对 SMG failover 四个核心目标逐一评估。

### 7.1 目标一：主动感知主备拓扑

**评估：✅ 可行**

**方案：**
1. 在 wasm-go 插件中配置每个 Group 的 endpoint 优先级列表（通过 `WasmPlugin` CRD 的 `defaultConfig`）
2. 使用 `RegisterTickFunc` 注册周期性 health check（如每 3 秒）
3. 使用 `wrapper.NewClusterClient(wrapper.FQDNCluster{Host: endpointHost, Port: endpointPort})` 或 `wrapper.NewClusterClient(wrapper.K8sCluster{ServiceName: endpointSvc, Namespace: ns, Port: port})` 向各 endpoint 发送 `/health` 探测
4. 使用 `proxywasm.SetSharedData` 将状态翻转后的 endpoint 健康状态写入共享存储
5. 参考 `ai-proxy` 的 lease 机制，避免多 VM 重复探测

**状态设计：**
```
SharedData key: "smg:group-1:ep:smg-1-1:health"
SharedData key: "smg:group-1:ep:smg-1-2:health"
SharedData key: "smg:group-1:ep:smg-1-3:health"
Value: "healthy" / "unhealthy" / "degraded" + timestamp
```

### 7.2 目标二：主动控制请求发送

**评估：✅ 可行（wasm-go 框架）**

**方案 A（Endpoint 级切换，推荐）：**
```go
func onHttpRequestHeaders(ctx wrapper.HttpContext, cv config.PluginConfig) {
    // 获取当前请求的 cluster 名称
    clusterName := getClusterNameFromProperty()
    // 查找该 cluster 对应的 Group 配置
    groupConfig := cv.GetGroupConfig(clusterName)
    
    // 按优先级遍历 endpoint，选择第一个 healthy 的
    for _, ep := range groupConfig.Endpoints {
        health := getSharedData(fmt.Sprintf("smg:%s:ep:%s:health", groupConfig.Name, ep.Name))
        if health == "healthy" {
            // 锁定到该 endpoint
            proxywasm.SetUpstreamOverrideHost([]byte(ep.Address))
            break
        }
    }
}
```

外层 Higress 已通过 VirtualService 选定 Group（对应一个 Envoy cluster），插件在 cluster 内部按优先级选择 healthy endpoint。`ai-load-balancer` 的 `endpoint_metrics`、`global_least_request`、`prefix_cache` 策略均使用了相同的 `SetUpstreamOverrideHost` 机制。

**方案 B（跨 Cluster 级切换）：**
```go
func onHttpRequestHeaders(ctx wrapper.HttpContext, cv config.PluginConfig) {
    primaryHealthy := getSharedData("smg:group-1:primary:health")
    if primaryHealthy != "healthy" {
        // 切换到备 cluster（适用于主备被定义为不同 cluster 的场景）
        proxywasm.ReplaceHttpRequestHeader("x-higress-target-cluster", "smg-group-1-backup")
    }
}
```

适用于主备实例被定义为不同 Envoy cluster 的场景（如每个实例有独立的 ServiceEntry）。在 SMG 主备架构中，外层 Higress 已通过 weight/轮询选定了 Group，因此该方案不是主要路径。

**方案 C（golang-filter 的局限）：**
golang-filter 没有 `SetUpstreamOverrideHost` 或 `x-higress-target-cluster` 机制，只能通过 `SetHost` + `RefreshRouteCache` 间接影响。如果 Higress 路由规则恰好基于 `:authority` 选择 cluster，这种方式可能有效，但耦合度高、不可靠。**不推荐**。

### 7.3 目标三：请求失败时第一时间正确重试

**评估：✅ 可行**

**方案：**
1. 在 `OnHttpResponseHeaders` 中检查 `:status`
2. 如果主返回 5xx，使用 `wrapper.NewClusterClient(wrapper.FQDNCluster{Host: backupHost, Port: backupPort})` 向备 endpoint 发起异步请求
3. 在回调中收到备的响应后，使用 `proxywasm.SendHttpResponse` 将响应返回给客户端
4. 参考 `ai-proxy` 的 retry 机制，保存原始请求信息用于重构

**代码骨架：**
```go
func onHttpResponseHeaders(ctx wrapper.HttpContext, statusCode int) {
    if statusCode >= 500 {
        // 保存的原始请求信息
        host := ctx.GetContext("saved_host")
        path := ctx.GetContext("saved_path")
        body := ctx.GetContext("saved_body")
        headers := ctx.GetContext("saved_headers")
        
        client := wrapper.NewClusterClient(wrapper.FQDNCluster{
            Host: backupHost,
            Port: backupPort,
        })
        client.Post(path, headers, body, func(status, respHeaders, respBody) {
            proxywasm.SendHttpResponse(status, respHeaders, respBody, -1)
        }, 30000)
    }
}
```

**被动重试触发即时健康评估：**
当主节点返回 5xx 时，除了向备 endpoint 重试当前请求，还可以**异步触发一次对主 endpoint 的即时健康检查**：

```go
func onHttpResponseHeaders(ctx wrapper.HttpContext, statusCode int) {
    if statusCode >= 500 {
        // 1. 向备 endpoint 发起异步重试（同上）
        // ...
        
        // 2. 异步触发对主 endpoint 的即时健康评估
        // 不等下一个 tick（可能还有 1-3 秒），立即确认主是否真的不健康
        client := wrapper.NewClusterClient(wrapper.FQDNCluster{
            Host: primaryHost,
            Port: primaryPort,
        })
        client.Post("/health", nil, nil, func(healthStatus, _, _) {
            if healthStatus != 200 {
                // 确认主不健康，立即更新 SharedData
                proxywasm.SetSharedData("smg:group-1:primary:health", []byte("unhealthy"), cas)
                // 同时更新失败计数器
            }
        }, 3000)
    }
}
```

这个机制的价值在于：
- 周期性 tick 健康检查存在探测间隔（如 3 秒），在两次 tick 之间主可能已经故障
- 请求失败是一个**事件驱动的信号**——它暗示主节点可能已不健康
- 利用 `DispatchHttpCall` 的异步能力，可以在不阻塞当前请求处理的前提下，立即验证主的健康状态
- 如果验证确认主不健康，下一个请求到达时（可能就在几毫秒后），SharedData 中已经是正确的健康状态，`OnHttpRequestHeaders` 中按优先级选择下一个 healthy endpoint

这相当于把"周期性主动探测"和"事件驱动即时确认"结合起来，缩短了故障感知到路由切换的延迟窗口。

**SSE 流式请求的限制：**
与 `ai-proxy` retry 的限制相同——流式响应一旦开始发送，就无法"替换"为另一个响应。对于 SSE 场景：
- 如果主在发送首 token 前返回 5xx，可以完整重试到备
- 如果主已经开始发送 token 后断开，重试会变得复杂（客户端已收到部分 token）
- 这种情况下，被动重试只能保证"首 token 前的失败"被兜底

### 7.4 目标四：后续请求正确发送到备用节点

**评估：✅ 可行**

这正是 preemptive 切换解决的问题。通过 `SharedData` 维护的 endpoint 健康状态 + `OnHttpRequestHeaders` 中的 endpoint 优先级选择逻辑，后续请求会自动锁定到 healthy endpoint。

**完整流程：**
```
请求 N 到达
  │
  ▼
OnHttpRequestHeaders:
  1. 读取当前 cluster 名称（GetProperty("cluster_name")）
  2. 读取 SharedData 中该 Group 各 endpoint 的健康状态
  3. 按优先级选择第一个 healthy endpoint:
     SMG-1.1 healthy → SetUpstreamOverrideHost(SMG-1.1:30002)
     SMG-1.1 unhealthy, SMG-1.2 healthy → SetUpstreamOverrideHost(SMG-1.2:30002)
     SMG-1.1 unhealthy, SMG-1.2 unhealthy, SMG-1.3 healthy → SetUpstreamOverrideHost(SMG-1.3:30002)
  │
  ▼
请求发送到选中的 endpoint
  │
  ▼
OnHttpResponseHeaders:
  如果 5xx:
    1. 更新 SharedData：该 endpoint 标记为 unhealthy
    2. 异步触发对该 endpoint 的即时健康检查
    3. 尝试向下一优先级 endpoint 异步重试
    4. 重试成功 → SendHttpResponse 返回给客户端
    5. 重试失败 → 返回错误
```

---

## 八、推荐方案设计

### 8.1 方案选择：wasm-go 插件 + `SetUpstreamOverrideHost` 稳定锁定

基于以上评估，推荐采用 **Higress wasm-go 插件** 方案。需要强调的是，本方案不存在传统意义上的"主备角色"——所有节点完全平等，插件仅基于健康检查结果选择一个 healthy endpoint 并**稳定锁定**，将 Group 内全部请求集中到该节点，避免轮询导致的 cache tree 分散。核心机制：

1. **Endpoint 级节点定义**：外层 Higress 通过 VirtualService 将请求路由到对应 Group（Envoy cluster），每个 Group 内配置多个 endpoint（SMG-1.1、SMG-1.2、SMG-1.3）及锁定顺序
2. **WasmPlugin 配置**：通过 CRD 声明每个 Group 的 endpoint 列表、锁定优先级、健康检查参数
3. **主动健康检查**：`RegisterTickFunc` 定期向各 endpoint 发送 `/health` 探测。每个 VM 在内存中维护一份健康状态缓存，只有当探测结果与缓存中的状态发生**翻转**（healthy → unhealthy 或 unhealthy → healthy）时，才调用 `SetSharedData` 更新共享状态。这样可以大幅减少不必要的 CAS 写竞争。
4. **稳定锁定**：`OnHttpRequestHeaders` 中根据 `SharedData` 健康状态，按优先级选择第一个 healthy endpoint，调用 `SetUpstreamOverrideHost` 锁定。后续请求持续锁定到同一 endpoint，直到其不健康
5. **失效切换与重试**：`OnHttpResponseHeaders` 中 5xx 后向下一优先级 endpoint 异步重试，同时异步触发对故障 endpoint 的即时健康评估，确认后更新 `SharedData`，下一次请求重新锁定到新的 healthy endpoint
6. **恢复后重新纳入**：节点恢复后，经过 `successThreshold` 次连续健康探测后重新纳入可选池（参考 ai-proxy 的 recovery 机制）。是否自动切回由 `failbackStrategy` 配置决定

### 8.2 Higress 路由配置

```yaml
# ServiceEntry：定义 Group（对应一个 Envoy cluster）
# cluster 下包含多个 endpoint（主备实例）
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: smg-group-1
spec:
  hosts:
    - smg-group-1.default.svc.cluster.local
  ports:
    - number: 30002
      name: http
      protocol: HTTP
  location: MESH_INTERNAL
  resolution: DNS
  endpoints:
    - address: smg-1-1.default.svc.cluster.local
      labels:
        role: node
    - address: smg-1-2.default.svc.cluster.local
      labels:
        role: node
    - address: smg-1-3.default.svc.cluster.local
      labels:
        role: node
---
# VirtualService：外层路由，按 weight 分发到各 Group
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: smg-routing
spec:
  hosts:
    - smg-gateway.default.svc.cluster.local
  http:
    - route:
        - destination:
            host: smg-group-1.default.svc.cluster.local
          weight: 33
        - destination:
            host: smg-group-2.default.svc.cluster.local
          weight: 33
        - destination:
            host: smg-group-3.default.svc.cluster.local
          weight: 34
---
# WasmPlugin：SMG Failover 插件
# 配置每个 Group 的 endpoint 优先级，插件在 cluster 内部选择 healthy endpoint
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: smg-failover
  namespace: higress-system
spec:
  selector:
    matchLabels:
      higress: higress-system-higress-gateway
  defaultConfig:
    groups:
      - name: group-1
        cluster: smg-group-1.default.svc.cluster.local
        endpoints:
          - name: smg-1-1
            address: smg-1-1.default.svc.cluster.local:30002
            priority: 0
          - name: smg-1-2
            address: smg-1-2.default.svc.cluster.local:30002
            priority: 1
          - name: smg-1-3
            address: smg-1-3.default.svc.cluster.local:30002
            priority: 2
        healthCheck:
          interval: 3000
          timeout: 5000
          failureThreshold: 2
          successThreshold: 2
        failbackStrategy: auto  # auto / manual
  url: oci://registry/smg-failover:1.0.0
```

### 8.3 SharedData 读写策略与性能分析

在 wasm-go 插件架构中，一个关键的设计决策是：**是否需要 SharedData，以及如何使用它**。这涉及到跨 VM 一致性、路由性能、和健康检查收敛速度之间的权衡。

#### 8.3.1 为什么需要 SharedData？

Higress 网关通常运行多个 Envoy Worker 线程，每个 Worker 线程中可能有独立的 WASM VM 实例。不同 VM 之间**没有共享内存**。

如果不使用 SharedData：
- VM-A 探测到 endpoint-1 不健康，更新本地缓存，后续请求切换到 endpoint-2
- VM-B 仍在使用自己的本地缓存，继续向 endpoint-1 发送请求
- 结果是：部分 VM 已切换，部分 VM 仍在发送请求到故障节点

**SharedData 的核心价值是缩小"不一致窗口"**——当一个 VM 发现故障时，其他 VM 能通过 SharedData 尽快感知，而不是等到各自独立探测到故障。

#### 8.3.2 读写时机与调用链路

**读路径（`OnHttpRequestHeaders`）：**
```
请求到达 → GetSharedData("ep:smg-1-1:health") → 若为 healthy → SetUpstreamOverrideHost
              ↓ 若为 unhealthy
         GetSharedData("ep:smg-1-2:health") → ...依优先级遍历
```

每个请求按优先级读取 SharedData，直到找到第一个 healthy endpoint。最坏情况下读取 N 个 key（N = endpoint 数量）。

**写路径有两个来源：**
1. **周期性 tick health check**（`RegisterTickFunc`）：探测完成后，状态翻转时写 SharedData
2. **即时健康评估**（`OnHttpResponseHeaders` 5xx 后）：异步探测完成后，状态翻转时写 SharedData

#### 8.3.3 本地缓存 + 状态翻转写优化

纯粹的 SharedData 读写（无本地缓存）存在明显问题：
- 每个请求都进行 `GetSharedData` host call，增加 WASM/Envoy 边界穿越开销
- 高频 tick（如每秒）下，即使状态未变也反复写 SharedData，CAS 竞争严重

推荐的分层策略：

```
┌─────────────────────────────────────────────┐
│  每个 VM 的本地缓存（Go map，零 host call）   │
│  - key: endpoint name                         │
│  - value: health status + lastUpdateTime      │
└─────────────────────────────────────────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
    读请求路由    周期性 tick    即时健康评估
    (OnHttpRequest)  (RegisterTickFunc) (OnResponseHeaders)
         │            │            │
         ▼            ▼            ▼
    从本地缓存读   health check   health check
         │            │            │
         │            ▼            ▼
         │      状态翻转？ ──否──→ 不写 SharedData
         │         │
         │         是
         │         ▼
         │    写 SharedData
         │         +
         │    更新本地缓存
         │
         ▼
    若本地缓存 miss 或过期
    → 读 SharedData → 回填本地缓存
```

**读优化：**
- 正常路径：从本地缓存读取（零 host call）
- 本地缓存有过期时间（如 1-2 秒），过期后回源 SharedData
- 这样绝大多数请求的 `OnHttpRequestHeaders` 不涉及 host call

**写优化：**
- tick 探测结果与本地缓存对比，**只有状态翻转时才写 SharedData**
- 健康状态稳定期间（99% 的时间），几乎没有 SharedData 写操作
- 状态翻转瞬间（1% 的时间），存在短暂的 CAS 竞争，但冲突概率极低

#### 8.3.4 性能影响评估

| 场景 | 纯 SharedData（无缓存） | 本地缓存 + 状态翻转写 |
|------|----------------------|---------------------|
| 正常请求（healthy，无切换） | 每请求 N 次 GetSharedData host call | 每请求 0 次 host call（本地缓存命中） |
| 故障切换瞬间 | N 次 GetSharedData + SetSharedData CAS 竞争 | N 次 GetSharedData（缓存过期回源）+ 1 次 SetSharedData |
| 稳定故障状态 | 每请求 N 次 GetSharedData | 每请求 0 次 host call |
| tick 期间 | 每 tick N 次 SetSharedData（高频 CAS） | 0 次 SetSharedData（状态未翻转） |

对于 3 个 endpoint 的 Group，本地缓存方案将正常请求的 host call 从 1-3 次降为 0 次，tick 期间的写操作从每 tick 3 次降为几乎 0 次。

#### 8.3.5 SharedData 是否必须？

这是一个值得追问的设计决策。如果每个 VM 完全独立探测、独立缓存、不写 SharedData：

- **最大不一致窗口** = 2 * tickInterval（如 6 秒，tickInterval=3s）
- 这意味着故障发生后，最多 6 秒内仍有部分 VM 向故障节点发送请求
- 对于 SMG 场景，6 秒的延迟是否可接受？

如果 tickInterval 降低到 1 秒，不一致窗口缩短到 2 秒。但探测频率增加，对 endpoint 的压力也增加。

**建议的折中方案（分层同步）：**
1. 每个 VM 维护本地缓存，读路径走本地缓存（零 host call）
2. 状态翻转时写 SharedData（供其他 VM 快速感知）
3. 其他 VM 定期（如每 2-3 秒）从 SharedData 回源，更新本地缓存
4. 这样既有本地缓存的高性能，又有跨 VM 同步的快速收敛

这个方案下，SharedData 不是"唯一数据源"，而是"跨 VM 状态同步通道"，本地缓存才是每个 VM 的权威数据源。

### 8.4 与 V4（Operator + EndpointSlice）的对比

| 维度 | V4：Operator + EndpointSlice | V5.1：Higress wasm-go 插件 |
|------|---------------------------|---------------------------|
| 组件数量 | 1 个 Operator Deployment | 0 个额外组件（纯插件） |
| 部署方式 | CRD + Go 程序 | CRD + OCI 镜像 |
| 热更新 | Operator 滚动更新 | WasmPlugin CRD 更新即可 |
| 主动健康检查 | Operator HTTP 探测 | 插件内 RegisterTickFunc |
| 路由控制 | 动态修改 EndpointSlice | `SetUpstreamOverrideHost` 锁定 endpoint |
| 被动重试 | ❌ 无 | ✅ 5xx 后 dispatch 到备 endpoint |
| 恢复后重新纳入 | CRD 字段完全可控 | CRD 字段完全可控（`failbackStrategy`: auto / manual） |
| 已建立 SSE 连接中断处理 | ❌ 无（连接由 Envoy 维护，中断后无恢复机制） | ❌ 无（与 V4 相同，WASM 无法替换已开始的流式响应） |
| 与 Istio 耦合 | 低（标准 K8s API） | 中（依赖 Higress `SetUpstreamOverrideHost` 扩展） |
| 隔离性 | 高（独立进程） | 高（WASM 沙箱） |

V5.1 在**不引入额外组件**的前提下，同时提供了健康感知锁定和被动重试两种能力，是 V1-V4 所有方案中**能力最完整**的一条路径。所有节点平等，插件仅基于健康状态选择并锁定 endpoint，不存在固定主备角色。唯一的局限是 SSE 流式请求的被动重试（与 ai-proxy 的 retry 限制相同）。

---

## 九、遗留问题与下一步

1. **SSE 流式请求的被动重试**：流式响应一旦开始发送，无法透明替换。需要评估 SMG 场景下流式请求在主节点断开后的处理策略（如客户端重连、token 级续传等）。
2. **SharedData 的并发安全**：多 VM 同时读写 SharedData 需要 CAS（Compare-And-Swap）机制。`ai-proxy` 的 lease 机制值得参考。通过"状态翻转才写"的优化，健康状态稳定期间几乎没有写竞争，只有状态翻转瞬间存在 CAS 冲突，竞争开销可控。
3. **Cluster 预热**：当流量切换到备 endpoint 时，Envoy 可能需要时间建立连接池。首请求的延迟可能高于正常路径。可以评估是否需要连接池预热机制。
4. **多 Group 的扩展性**：当前设计针对单个 SMG group。当 group 数量增加时，WasmPlugin 配置的体积和 health check 的并发数需要评估。
5. **与 V6（cache tree 同步）的协同**：V5.1 解决的是 worker 故障时的 failover，V6 解决的是 Router 多实例扩展时的 cache 一致性。两者可以共存：V5.1 的插件同时维护主备健康状态和 cache tree 同步通道的元数据。

---

## 十、结论

V5.1 的核心发现是：**Higress 的 wasm-go 框架通过自定义扩展 ABI，提供了超出标准 proxy-wasm 的路由控制能力**，使得 preemptive 切换 upstream endpoint 成为可能。这与 V5 基于标准 ABI 得出的"preemptive 切换不可行"结论形成了重要修正。

具体而言：
- `SetUpstreamOverrideHost` 可以在请求到达时动态选择 endpoint
- `RegisterTickFunc` + `TargetCluster` 可以实现主动健康检查
- `DispatchHttpCall` + `SendHttpResponse` 可以在锁定节点失效时透明重试
- `SharedData` 可以维护跨请求的节点健康状态

生产级插件 `ai-proxy`（token-level failover + health check + retry）和 `ai-load-balancer`（cluster/endpoint-level dynamic routing）已经验证了这套机制的工程可行性。

**推荐方向**：使用 Higress wasm-go 插件实现 SMG 的 endpoint 健康感知与稳定锁定。所有节点完全平等，插件按配置顺序选择第一个 healthy endpoint 并持续锁定，直到其不健康才重新选择。无需引入额外组件。

---

## 附录：参考来源

本调研报告中所有技术结论均基于对以下源码、文档和 issue 的直接阅读和分析，按主题分类列出。

### A. Higress 官方仓库（本地 clone）

本项目中的 Higress 源码 clone 路径：`higress/`

#### A.1 插件体系总览

- `higress/plugins/README.md`：Higress 插件体系概览：WASM C++/Go 插件介绍、内置插件列表、WasmPlugin CRD 配置示例

#### A.2 golang-filter（Native Go Filter）

- `higress/plugins/golang-filter/README.md`：golang-filter 框架说明：基于 Envoy Golang HTTP Filter，编译为 `.so`，支持请求/响应头修改、body 修改、同步请求处理
- `higress/plugins/golang-filter/main.go`：Filter 注册入口：`envoyHttp.RegisterHttpFilterFactoryAndConfigParser` 注册 `mcp-session` 和 `mcp-server`
- `higress/plugins/golang-filter/mcp-server/filter.go`：StreamFilter 实现示例：`DecodeHeaders`、`DecodeData`、`EncodeHeaders` 回调处理
- `higress/plugins/golang-filter/mcp-session/filter.go`：另一 StreamFilter 实现示例，包含 SSE 响应 body 改写逻辑

> Envoy Golang HTTP Filter 官方源码参考：`github.com/envoyproxy/envoy/tree/main/contrib/golang`

#### A.3 wasm-go 框架与示例

- `higress/plugins/wasm-go/README.md`：wasm-go 插件开发指南：构建方式、WasmPlugin CRD 配置、单元测试、E2E 测试
- `higress/plugins/wasm-go/README_EN.md`：英文版开发指南
- `higress/plugins/wasm-go/examples/custom-log/main.go`：自定义日志/追踪属性示例：`SetUserAttribute`、`WriteUserAttributeToLog`
- `higress/plugins/wasm-go/examples/custom-span-attribute/main.go`：修改请求 header 和 trace span tag 示例
- `higress/plugins/wasm-go/examples/test-foreign-function/main.go`：Foreign Function 调用和响应 header 操作示例

> 核心框架代码不在本仓库中，位于 Go module 依赖 `github.com/higress-group/wasm-go` 的 Go module cache 中。框架源码路径（基于 Go module cache 结构）：
> - `pkg/wrapper/plugin_wrapper.go` — 插件生命周期、tick 分派、rule 匹配、`RouteCall`、body buffering
> - `pkg/wrapper/http_wrapper.go` — `HttpClient` 接口、`ClusterClient`、异步 `HttpCall`
> - `pkg/wrapper/cluster_wrapper.go` — Cluster 抽象类型：`RouteCluster`、`TargetCluster`、`K8sCluster`、`NacosCluster`、`FQDNCluster`
> - `pkg/wrapper/request_wrapper.go` — 请求辅助函数（scheme、host、path、method、websocket 检测）
> - `pkg/wrapper/response_wrapper.go` — `IsResponseFromUpstream()` 辅助函数
> - `pkg/iface/context.go` — `HttpContext` 和 `PluginContext` 接口定义
> - `pkg/wrapper/redis_wrapper.go` — Redis 客户端封装

> proxy-wasm-go-sdk 标准 ABI 参考：`github.com/higress-group/proxy-wasm-go-sdk/proxywasm/hostcall.go`

#### A.4 ai-proxy 插件（生产级 failover + retry 参考）

- `higress/plugins/wasm-go/extensions/ai-proxy/main.go`：插件主入口：请求/响应生命周期钩子、provider 选择、`DisableReroute` 调用、fallback header 处理
- `higress/plugins/wasm-go/extensions/ai-proxy/config/config.go`：插件配置结构：`PluginConfig`（providers 数组、`activeProviderId`）
- `higress/plugins/wasm-go/extensions/ai-proxy/provider/provider.go`：Provider 接口定义、apiToken 选择逻辑、模型映射
- `higress/plugins/wasm-go/extensions/ai-proxy/provider/failover.go`：**核心参考**：apiToken failover 机制、active health check（lease-based leader election）、cooldown recovery、状态码匹配
- `higress/plugins/wasm-go/extensions/ai-proxy/provider/retry.go`：**核心参考**：请求重试机制、`RouteCluster` 重试、非流式请求限制
- `higress/plugins/wasm-go/extensions/ai-proxy/provider/context.go`：Context 文件加载：`FQDNCluster`、`TargetCluster` 使用示例
- `higress/plugins/wasm-go/extensions/ai-proxy/util/http.go`：Header 操作工具函数、fallback header 定义
- `higress/plugins/wasm-go/extensions/ai-proxy/README.md`：ai-proxy 插件文档：配置说明、retry 限制（"目前仅支持对非流式请求进行重试"）

#### A.5 ai-load-balancer 插件（生产级动态路由参考）

- `higress/plugins/wasm-go/extensions/ai-load-balancer/main.go`：插件主入口：配置解析、lb_type 分发
- `higress/plugins/wasm-go/extensions/ai-load-balancer/cluster_hash/lb_policy.go`：Cluster 级一致性哈希：`FNV-1a` 哈希、100-slot 加权数组、`x-higress-target-cluster` header 使用
- `higress/plugins/wasm-go/extensions/ai-load-balancer/cluster_metrics/lb_policy.go`：Cluster 级指标负载均衡：`LeastBusy`、`LeastTotalLatency`、`LeastFirstTokenLatency`、失败惩罚机制
- `higress/plugins/wasm-go/extensions/ai-load-balancer/endpoint_metrics/lb_policy.go`：Endpoint 级指标负载均衡：`GetUpstreamHosts` 健康过滤、vLLM Prometheus 指标解析、Inference Extension filter chain
- `higress/plugins/wasm-go/extensions/ai-load-balancer/endpoint_metrics/backend/vllm/metrics.go`：vLLM `/metrics` 端点数据解析：`num_requests_running`、`gpu_cache_usage_perc` 等
- `higress/plugins/wasm-go/extensions/ai-load-balancer/global_least_request/lb_policy.go`：全局最少请求：Redis Lua 脚本原子操作、`SetUpstreamOverrideHost` 使用
- `higress/plugins/wasm-go/extensions/ai-load-balancer/prefix_cache/lb_policy.go`：Prompt 前缀缓存：Redis 前缀树、`SetUpstreamOverrideHost`、fallback 到 `global_least_request`
- `higress/plugins/wasm-go/extensions/ai-load-balancer/utils/queue.go`：固定大小环形缓冲区（用于 rate limit 统计）
- `higress/plugins/wasm-go/extensions/ai-load-balancer/README.md`：ai-load-balancer 文档：lb_type 说明、策略详解、配置示例
- `higress/plugins/wasm-go/extensions/ai-load-balancer/README_EN.md`：英文版文档

### B. V5 调研中引用的标准 proxy-wasm ABI 限制

- proxy-wasm/spec issue #16：WASM 中 `set_http_request_header` 修改的 header 只影响 upstream，Envoy Router 看不到修改后的值，无法影响 routing 决策
- envoyproxy/envoy issue #28673：WASM 中 `setProperty` 自动前缀 `wasm.`，无法设置 `envoy.tcp_proxy.cluster` 或 `envoy.router.cluster` 等关键 filter state

> 以上两个 issue 的详细分析见 `ha_design_v5/active_standby_ha_design_v5_research.draft.md` 第二章。

### C. 相关系统设计参考

- NVIDIA Dynamo / llm-d：V6 调研中引用的同类系统，其 Router 层事件同步架构为 V6 的混合同步方案提供了工程参考
- sglang `cache_aware.rs` / `tree.rs` / `kv_events/`：V6 调研中分析的 sglang 源码，确认了 cache tree 分散的根因和已有事件流基础设施

### D. 术语对照

- preemptive 切换：`OnHttpRequestHeaders` 中通过 `SetUpstreamOverrideHost` 锁定 healthy endpoint
- 被动重试：`OnHttpResponseHeaders` 中 `DispatchHttpCall` + `SendHttpResponse`
- 主动健康检查：`RegisterTickFunc` + `TargetCluster` HTTP 探测
- SharedData：`proxywasm.GetSharedData` / `SetSharedData`
- Lease-based leader election：`ai-proxy` failover.go 中的 `tryAcquireOrRenewLease` 机制
- Cluster：Envoy upstream cluster，对应 Higress `ServiceEntry` 定义的上游服务
- Endpoint：Cluster 内的具体后端实例（IP:port）

