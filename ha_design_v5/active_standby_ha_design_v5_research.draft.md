# SMG Active-Standby HA 架构 V5 方案调研 — Higress WASM 插件实现主备 Failover

> 目标：深入验证 Higress/Envoy WASM 插件能否在**主请求失败后，通过 `dispatch_http_call` 在备 cluster 上重试**，实现客户端无感知的被动 failover。同时验证是否能在请求到达时 preemptive 切换 upstream 目标。此前认为 WASM 无法 preemptive 切走流量，但需详尽验证被动重试路径的可行性。
>
> 本方案为 V3（父子网关）中方案 D 的独立深化调研，从 V3 分离出来做更详尽的技术验证。

---

## 一、WASM 插件方案概述

### 1.1 核心问题

WASM 插件运行在 Envoy HTTP Filter 层，位于 Router 之前。关键问题是：

1. **能否在请求到达时 preemptive 切换 upstream cluster/endpoint？**（即主不健康时，直接将请求路由到备）
2. **能否在请求失败后，在备 cluster 上重试，且对客户端无感知？**（即客户端永远收不到 5xx，只收到备的成功响应）
3. **能否后台持续探测后端健康状态（类似 active health check）？**

### 1.2 预期架构

```
外部请求
   │
   ▼
┌────────────────────────────────────────────┐
│         Higress Gateway                    │
│                                            │
│  VirtualService: /v1/chat/*                │
│  route → smg-group-1-route                 │
│                                            │
│  ┌────────────────────────────────────┐   │
│  │ WASM Plugin (SMG Failover)         │   │
│  │                                    │   │
│  │ 1. on_request_headers:             │   │
│  │    - 无法 preemptive 切换 cluster   │   │
│  │    - Router 已决定发到主 cluster    │   │
│  │                                    │   │
│  │ 2. on_response_headers (主返回 5xx):│   │
│  │    - dispatch_http_call → 备 cluster│   │
│  │    - 备响应成功 → SendHttpResponse  │   │
│  │    - 客户端始终只收到成功响应        │   │
│  │                                    │   │
│  │ 3. tick (health check, 可选):      │   │
│  │    - 定期探测主/备 /health          │   │
│  │    - 更新共享数据                   │   │
│  └────────────────────────────────────┘   │
│                                            │
│  Router → selected cluster                 │
└────────────────────────────────────────────┘
```

---

## 二、proxy-wasm ABI 能力边界验证

### 2.1 修改请求头对 Routing 的影响

**proxy-wasm/spec issue #16** (2020-12-02, 至今 open)：

> 开发者尝试在 WASM 中使用 `cluster_header: region` 路由功能，期望通过 `set_http_request_header("region", "us-central1")` 动态选择 cluster。
>
> **实际结果**：`set_http_request_header` 修改的 header **只影响 upstream**，Envoy 的 Router **看不到**修改后的值，无法影响 routing 决策。

**根本原因**：
- Envoy 的 HTTP Filter 链中，Router 在 WASM Filter 之后执行
- WASM Filter 修改的 header 在 Filter 链继续传播时会被处理，但 Router 已经基于**原始 header** 做了 routing 决策
- 这与 Lua Filter 不同——Lua Filter 在特定执行阶段可以影响 Router

**结论**：❌ **WASM 无法通过修改请求头来改变 Router 的 cluster 选择**。

### 2.2 设置 Filter State 对 Routing 的影响

**envoyproxy/envoy issue #28673** (2023-07-28)：

> `setProperty` 方法在 WASM 中会**自动前缀 `wasm.`**，因此无法设置关键 filter state：
> - `envoy.tcp_proxy.cluster` — tcp_proxy 用它选择 upstream cluster
> - `envoy.router.cluster` — Router 的 cluster 选择

**结论**：❌ **WASM 无法通过 filter state 影响 cluster 选择**。

### 2.3 Higress 扩展 ABI：GetUpstreamHosts

Higress 扩展了 proxy-wasm ABI，提供了 `GetUpstreamHosts()`，可以读取当前 cluster 的 endpoint 列表和健康状态。

**能力**：
- ✅ 读取 upstream host 信息（地址、健康状态、metrics）
- ❌ **只能读，不能写**（无法修改 upstream 选择）

### 2.4 dispatch_http_call 的请求路径与 cluster 配置生效范围

`dispatch_http_call` 是 V5 被动 failover 的核心机制。理解它的内部请求路径，对评估 failover 的可靠性至关重要。

#### 2.4.1 dispatch_http_call 走的是什么路径？

**结论：它走的是 Envoy 的 AsyncClient（异步 HTTP 客户端）路径，不是常规的 downstream → Router 路径。**

证据：
- proxy-wasm-go-sdk issue #438 的调试日志明确显示 `async_client_impl.cc:106`
- Istio issue #31194 的崩溃堆栈显示完整调用链：`Context::httpCall()` → `AsyncClientImpl::send()` → `AsyncStreamImpl` → `Router::Filter::decodeHeaders()`

这意味着 `dispatch_http_call` 内部确实创建了一个 **Router filter 的独立实例** 来处理请求，但这个 Router 实例：
- **不经过** Listener → HTTP Connection Manager → Filter Chain 的完整路径
- 使用**独立的连接池**（不与 downstream 请求共享连接）
- **没有 route 上下文**（不走 route 匹配流程）

#### 2.4.2 cluster 配置哪些生效、哪些不生效？

`dispatch_http_call` 指定 cluster 名称后，Envoy 通过 Cluster Manager 查找该 cluster。cluster 级别的配置**大部分生效**，route 级别的配置**不生效**。

| 配置类型 | 具体配置 | 是否生效？ | 说明 |
|---------|---------|-----------|------|
| **Cluster 级别** | `connect_timeout` | ✅ | 连接超时 |
| | `circuit_breakers`（熔断） | ✅ | 连接数/请求数/重试数限制 |
| | `health_checks`（主动健康检查） | ✅ | 不健康端点被排除 |
| | `outlier_detection`（异常检测） | ✅ | Router filter 实例会报告错误 |
| | `load_assignment` / `lb_policy` | ✅ | 负载均衡选择端点 |
| | `upstream_connection_options` | ✅ | TCP keepalive 等 |
| | `tls_context` / `transport_socket` | ✅ | mTLS 配置 |
| **Route 级别** | `retry_policy` | ❌ | 没有 route 上下文 |
| | `timeout`（per-route） | ❌ | 需通过 `dispatch_http_call` 的 timeout 参数控制 |
| | `rate_limits` | ❌ | 不经过 rate limit filter |
| | `cors` / `fault` 等 | ❌ | 不经过对应 filter |

**关键影响分析**：

1. **熔断（Circuit Breaker）生效是好消息**：备 cluster 如果过载，`dispatch_http_call` 的请求也会被熔断保护，不会无限堆积。
2. **健康检查生效是好消息**：如果备 cluster 的端点被标记为不健康，`dispatch_http_call` 不会把请求发给它们。
3. **连接池独立意味着**：
   - 正常请求和 `dispatch_http_call` 到同一个 cluster 时，连接不共享
   - 这可能导致更多的连接建立开销（但影响通常很小）
   - 熔断阈值是**分别计算**还是**统一计算**？从 Envoy 的实现来看，circuit breaker 是 cluster 级别的计数器，AsyncClient 的请求也会计入该 cluster 的统计，因此应该是**统一计算**
4. **Route retry policy 不生效意味着**：WASM 必须自己实现重试逻辑（这正是 V5 的设计——在回调中递归重试）。

#### 2.4.3 已知问题：no healthy upstream 场景下的稳定性

**Istio issue #31194**：当 `dispatch_http_call` 的目标 cluster 变为 "no healthy upstream" 时，可能导致 Envoy **崩溃**（segmentation fault）。

崩溃堆栈显示：`Router::Filter::sendNoHealthyUpstreamResponse()` → `AsyncStreamImpl::sendLocalReply()` → `AsyncRequestImpl::onComplete()` → 空指针解引用。

**状态**：这是一个已知 bug，在较新版本的 Envoy/Istio 中可能已修复，但生产环境使用时需要注意：
- 确保备 cluster 始终有健康的 endpoint（通过 K8s readiness probe）
- 在 WASM 中捕获 `dispatch_http_call` 的返回值，处理 `internal failure` 错误
- 监控 Envoy 进程是否有异常重启

#### 2.4.4 对 V5 方案的影响

| 影响 | 评估 |
|------|------|
| **熔断/健康检查生效** | ✅ V5 的 failover 请求享有和正常请求同等级别的 cluster 保护 |
| **连接池独立** | ⚠️ 额外的连接开销，但可控；不共享连接意味着 failover 请求不受正常请求的拥塞影响 |
| **无 route retry** | ✅ 这正是 V5 的设计——WASM 自己控制重试逻辑 |
| **no healthy upstream 崩溃风险** | ⚠️ 需要确保备 cluster 健康，并做好错误处理 |
| **是否"无感接续"** | ⚠️ 不是"接续"同一个请求，而是**分叉出一个新的独立 HTTP 请求**。对客户端无感知，但内部是完全独立的请求生命周期 |

### 2.5 proxy-wasm ABI 能力总结

| ABI / 方法 | 能力 | 对 Routing 的影响 |
|-----------|------|-----------------|
| `set_http_request_header` | 修改请求头 | ❌ Router 看不到 |
| `setProperty` | 设置属性 | ❌ 前缀 `wasm.`，无法设置 Router 相关属性 |
| `getProperty("cluster_name")` | 读取当前 cluster | ✅ 只读 |
| `GetUpstreamHosts()` (Higress 扩展) | 读取 endpoint 列表 | ✅ 只读 |
| `dispatch_http_call` | 发起异步 HTTP 请求 | ✅ 走 AsyncClient 路径，独立于 downstream Router，可用于重试到另一个 cluster |
| `setTickPeriodMilliseconds` / `RegisterTickFunc` | 定时回调 | ✅ 可做后台探测（Higress 扩展） |
| `getSharedData` / `setSharedData` | 跨 VM 共享数据 | ✅ 可做健康状态共享 |

**核心结论**：
> **WASM 插件没有 ABI 可以直接修改 Router 的 upstream/cluster 选择决策。**
> 但 WASM 可以通过 `dispatch_http_call` 在请求失败后**独立发起新的 HTTP 请求**到另一个 cluster，实现客户端无感知的被动 failover。
> `dispatch_http_call` 走 AsyncClient 路径，享有 cluster 级别的熔断、健康检查、异常检测保护，但使用独立的连接池，且不享有 route 级别的重试策略。

---

## 三、Higress ai-proxy Failover 源码分析

> **本章是 V5 方案的核心调研支撑。**
>
> Higress **ai-proxy 是官方内置的生产级 WASM 插件**，用于 AI 大模型代理，已在线上环境大规模运行。该插件实现了完整的 **failover（故障切换）+ retry（失败重试）** 机制——其核心技术与 V5 方案完全一致：
> - 用 `dispatch_http_call` 在请求失败后发起新的 HTTP 调用
> - 用 `SendHttpResponse` 将重试成功的响应返回给客户端
> - 用 `RegisterTickFunc` + `SharedData` 实现主动健康检查
> - 用 `HeaderStopIteration` 暂停原请求流
>
> **ai-proxy 的 failover/retry 实现证明了：WASM 插件完全可以在生产环境中做请求级的路由控制和失败重试，这不是理论推导，而是已被官方验证的工程实践。**

为了验证 WASM 在实践中的 failover 能力，我们深入分析了 Higress 官方 ai-proxy 插件的 failover 实现。

### 3.1 ai-proxy 的 Failover 机制

ai-proxy 实现了**apiToken 级别**的 failover：
- 配置多个 apiToken
- 某个 token 连续失败 N 次后，标记为不可用
- 从可用列表中移除，后续请求不再使用该 token
- 对不可用 token 做主动健康检查，恢复后加回列表

### 3.2 主动健康检查实现

```go
// failover.go
wrapper.RegisterTickFunc(c.failover.healthCheckInterval, func() {
    // 只有获取到 lease 的 VM 才执行健康检查
    if c.tryAcquireOrRenewLease(vmID) {
        // 对不可用 token 发 HTTP 探测
        healthCheckClient.Post(url, headers, body, callback, timeout)
    }
})
```

**关键点**：
- 使用 `wrapper.RegisterTickFunc`（Higress 扩展，非标准 proxy-wasm）注册定时任务
- 健康检查通过 `healthCheckClient.Post` 即 `dispatch_http_call` 实现
- 通过 `GetSharedData`/`SetSharedData` 在多个 VM 间共享 token 健康状态

### 3.3 被动 Retry 实现

```go
// retry.go
func (c *ProviderConfig) retryFailedRequest(...) error {
    retryClient := createRetryClient() // RouteCluster{}
    return c.sendRetryRequest(ctx, apiName, activeProvider, retryClient, apiTokenInUse, apiTokens)
}

func (c *ProviderConfig) sendRetryRequest(...) error {
    // 换另一个 apiToken
    apiTokenInUse = GetRandomToken(apiTokens)
    
    // 用 dispatch_http_call 重新发请求
    retryClient.Post(url, headers, body, callback, timeout)
}

func createRetryClient() *wrapper.ClusterClient[wrapper.RouteCluster] {
    return wrapper.NewClusterClient(wrapper.RouteCluster{})
}
```

**关键点**：
- `RouteCluster` 的 `ClusterName()` 通过 `proxywasm.GetProperty([]string{"cluster_name"})` 获取**当前请求已匹配的 cluster**
- 重试是在**同一个 cluster 内**换另一个 apiToken，不是跨 cluster 切换
- 重试通过 `dispatch_http_call` 实现，成功后 `proxywasm.SendHttpResponse` 直接返回

### 3.4 ai-proxy 的设计对我们的启示

> **ai-proxy 作为 V5 方案的核心先例**
>
> ai-proxy 是 Higress **官方维护、线上运行的生产级 WASM 插件**。它的 failover + retry 实现直接证明了以下关键技术在生产环境中的可行性：
> 1. ✅ `dispatch_http_call` 可以在请求生命周期内安全地发起新的 HTTP 调用
> 2. ✅ `SendHttpResponse` 可以绕过 Router，将重试成功的响应直接返回给客户端
> 3. ✅ `RegisterTickFunc` + `SharedData` 可以实现跨 VM 的健康状态共享
> 4. ✅ `HeaderStopIteration` 可以暂停原请求流，由 WASM 接管后续处理
> 5. ✅ 被动 failover（失败后重试）是一种**已被官方采用**的设计模式，不是退而求其次的妥协
>
> **这意味着 V5 方案的技术路径不是从零开始的发明，而是基于 Higress 官方已验证的工程实践的延伸和适配。**

**被动 failover 本身是一种合理且成熟的设计选择**，其优势在于：
1. **零探测开销**：不需要定期发健康探测请求，减少额外流量
2. **即时响应**：请求失败时立即触发 failover，不会因为探测周期（如 2-5s）而错过切换窗口
3. **客户端无感知**：WASM 拦截 5xx 并在备上重试，客户端永远只收到成功响应
4. **状态一致性**：failover 决策基于**真实用户请求**的结果，而非探测请求的结果（探测和真实请求可能有不同行为，如认证、路由路径差异）

**ai-proxy 与 SMG 主备场景的差异**：“}}”}}cionefunctions.StrReplaceFile:35>{

| ai-proxy 场景 | SMG 主备场景 | 差异分析 |
|-------------|------------|---------|
| 同一 cluster 内换 token | **跨 cluster 切换**（主 cluster → 备 cluster） | ai-proxy 用 `RouteCluster`（获取当前 cluster），SMG 需用 `TargetCluster`（指定备 cluster） |
| 外部 API 调用（LLM SaaS） | 内部服务（SMG Pod） | SMG 是内部服务，延迟更低，更可控 |
| 非流式请求为主 | **SSE 流式响应** | 这是需要重点评估的场景 |

---

## 四、WASM 实现 SMG 主备 Failover 的可能方式

### 4.1 方式一：主失败后重试到备（`dispatch_http_call` 转发）

**思路**：
1. 请求到达时，WASM 不做干预，Router 将请求发到主 cluster
2. 如果主返回 5xx/连接失败，`on_http_response_headers` 中：
   - 用 `dispatch_http_call` 向备 cluster 发同样的请求（带完整 body）
   - 收到响应后，`SendHttpResponse` 把备的响应返回给客户端
   - **客户端始终只收到成功响应，不感知主已失败**

**核心优势**：
- **客户端无感知**：主失败后立即在备上重试，客户端永远收不到 5xx
- **不依赖后台探测状态同步**：无需担心探测周期导致的切换延迟，失败即切换
- **真实请求驱动**：决策基于真实用户请求的响应，比探测请求更可靠
- **无探测开销**：不需要 active health check，减少额外流量
- **多级依次降级**：可在回调中递归重试备A→备B，完全可控

**实现**：
```go
func onHttpRequestBody(ctx wrapper.HttpContext, config MyConfig, body []byte, log logs.Log) types.Action {
    // 缓存请求 body 到 context，供后续 retry 使用
    ctx.SetByteSliceContext(CtxRequestBody, body)
    return types.ActionContinue
}

func onHttpResponseHeaders(ctx wrapper.HttpContext, config MyConfig, log logs.Log) types.Action {
    status, _ := proxywasm.GetHttpResponseHeader(":status")
    if status == "503" || status == "502" || status == "504" {
        // 主挂了，发请求到备
        body := ctx.GetByteSliceContext(CtxRequestBody, []byte(""))
        config.backupClient.Post(url, headers, body, func(sc int, rh http.Header, rb []byte) {
            proxywasm.SendHttpResponse(sc, util.HeaderToSlice(rh), rb, -1)
        }, timeout)
        return types.HeaderStopAllIterationAndWatermark
    }
    return types.HeaderContinue
}
```

**关于 body 传递**：
- `proxywasm.DispatchHttpCall` 的签名包含 `body []byte` 参数，body 可直接传递
- issue #3531（ai-proxy retry 时 body 丢失）是 ai-proxy 插件的 bug，而非 proxy-wasm 的固有限制
- 正确做法：在 `on_http_request_body` 中缓存 body 到 context，retry 时从 context 取出传递

**限制**：
1. **每个失败请求增加一个 RTT**：先尝试主，失败后重试到备。但如果主已完全不可达，TCP 连接失败通常在几百毫秒内返回（取决于 connect_timeout 配置）。
2. **无法 preemptive**：即使主已不健康，新请求仍然先发到主。这是被动 failover 的固有特性，不是 WASM 的缺陷。
3. **连接池独立**：`dispatch_http_call` 使用独立的连接池，无法复用 Router 到备 cluster 的连接。

**结论**：✅ **非流式场景下完全可行**。被动 failover 的设计思路合理，实现路径清晰，客户端无感知。

### 4.2 方式二：主动探测 + Header 修改

**思路**：
1. WASM 用 `RegisterTickFunc` 定期探测主/备健康状态
2. 主不健康时，在 `on_http_request_headers` 中修改 `:authority` 或其他 header
3. 期望 Router 根据修改后的 header 选择备 cluster

**问题**：
- 如 2.1 节所述，Router **看不到** WASM 修改的 header
- 即使修改 `:authority`，Router 的 route 匹配已经完成

**结论**：❌ **不可行**。

### 4.3 方式三：WASM + EnvoyFilter 混合方案

**思路**：
1. 用 **EnvoyFilter** 配置 Aggregate Cluster（同 V2），实现 Router 级别的 preemptive failover
2. 用 **WASM 插件**做更智能的健康检查或业务级决策

**评估**：
- 如果已经使用 EnvoyFilter，WASM 的被动 failover 能力被 Aggregate Cluster 覆盖，价值有限
- 但如果希望**避免 EnvoyFilter**，WASM 单独实现被动 failover 是一个更轻量的替代方案
- WASM 可在请求前后做额外的业务逻辑（如日志、监控、自定义熔断策略）

**结论**：⚠️ **可行但冗余**。如果决定使用 EnvoyFilter，WASM 只做增强；如果决定不用 EnvoyFilter，WASM 单独实现被动 failover 是合理选择。

### 4.4 方式四：WASM 修改 Dynamic Metadata + EnvoyFilter 配合

**思路**：
- WASM 设置 Dynamic Metadata
- EnvoyFilter 中的 `metadata` match 或 Lua filter 读取 metadata 做路由决策

**问题**：
- WASM 的 `setProperty` 前缀 `wasm.`，Dynamic Metadata 键名受限
- EnvoyFilter 的 Lua filter 虽然可以读取 metadata，但这又回到了 EnvoyFilter 方案
- 复杂度比纯 EnvoyFilter 更高

**结论**：❌ **过于复杂，无实际优势**。

### 4.5 为什么不使用 Lua Filter

Envoy 的 HTTP Lua filter 允许在请求/响应流中运行 Lua 脚本，也支持异步 HTTP 调用（`httpCall`）。从能力上看，Lua 似乎也能实现被动 failover。但深入对比后，**Lua 不是 V5 的合适选择**。

**Lua Filter 的能力**：

| 能力 | 支持？ |
|------|--------|
| 检查/修改 headers | ✅ |
| 读取/缓冲 body | ✅ |
| 异步 HTTP 调用（类似 `dispatch_http_call`） | ✅ `request_handle:httpCall()` |
| 直接响应 | ✅ `request_handle:respond()` |
| 修改路由目标（preemptive failover） | ❌ 和 WASM 同样受限 |
| 进程隔离/沙箱 | ❌ 与 Envoy 同进程运行 |

**不选择 Lua 的四个理由**：

**1. 无隔离 = 生产风险**

LuaJIT 与 Envoy 同进程运行，没有 WASM 的 VM 沙箱隔离。Lua 脚本的内存泄漏、死循环或逻辑错误会**直接拖垮整个网关进程**。WASM 插件即使崩溃，也只会影响单个 VM 实例，不会波及 Envoy 本身。

**2. Higress 原生支持弱**

Higress 的插件体系核心是 **WasmPlugin CRD**，Lua 没有同等级别的原生支持。实际使用 Lua 需要通过 **EnvoyFilter** 注入 Lua 脚本——这和 V2 方案使用的是同一种机制，绕开了 Higress 的插件管理基础设施（无版本管理、无热更新、无 OCI 分发）。

**3. 性能更差**

| 机制 | 执行模型 | 性能 | 隔离性 |
|------|---------|------|--------|
| C++ Filter | Native | 最优 | 无 |
| **Lua** | LuaJIT coroutine | 中等 | **无** |
| **WASM** | VM (V8/Wazero) | 中等 | **沙箱** |

Higress 官方明确指出："**Lua 脚本的大量使用对性能影响非常大**"。failover 涉及多级降级、body 缓存、超时控制，这些用 Lua 写比 WASM + Go 更耗性能。

**4. Lua 能做的 WASM 都能做，WASM 能做的 Lua 未必能做**

- 两者都**无法 preemptive**（修改 Router 的 cluster 选择）
- 两者都可以**被动 failover**（异步 HTTP 调用 → 失败后重试）
- 但 WASM 有 Higress 原生 WasmPlugin CRD、沙箱隔离、Go SDK 生态、OCI 热更新——Lua 没有

**结论**：❌ **Lua filter 可以作为 Envoy 的轻量扩展用于 header 操作或日志采集，但不适合作为生产级 failover 机制**。V5 选择 WASM 而非 Lua，是因为 WASM 在隔离性、Higress 原生支持、可维护性上全面优于 Lua。

### 4.6 WASM 能否实现 Preemptive 效果？

前文多次提到 WASM "无法 preemptive"——这是指 WASM **无法修改 Router 的 cluster 选择**，新请求仍然先被 Router 路由到主 cluster。但这不等于 WASM **完全无法避免"每个请求都先失败一次"**。通过健康检查 + `SharedData` 状态共享，WASM 可以实现**类似 preemptive 的效果**。

#### 4.6.1 三种策略对比

| 策略 | 核心机制 | 主挂期间新请求行为 | 实现复杂度 | 适用场景 |
|------|---------|-------------------|-----------|---------|
| **A：纯被动 Failover** | 主返回 5xx 后重试到备 | 每个请求先走主 → 失败 → 重试到备 | 低 | 非流式为主，容忍先失败 |
| **B：失败记忆** | `SharedData` 记录主最近失败时间 | 第一次请求先失败 → 后续 N 秒内直接走备 | 中 | 非流式为主，减少失败延迟 |
| **C：主动探测 + Preemptive** | `RegisterTickFunc` 定期探测主健康 | 主不健康期间所有请求直接 `dispatch_http_call` 到备 | 中-高 | 要求 preemptive，可接受 SSE 实时性 trade-off |

#### 4.6.2 策略 A：纯被动 Failover（文档描述的 V5 基础版）

```go
func onHttpResponseHeaders(ctx, numHeaders int, endOfStream bool) types.Action {
    status := proxywasm.GetHttpResponseHeader(":status")
    if isFailureStatus(status) {
        // 主失败，重试到备
        dispatchToBackup(ctx, requestBody)
        return types.HeaderStopIteration
    }
    return types.Continue
}
```

**优点**：最简单，无健康检查开销，正常请求完全走 Router 不受影响。
**缺点**：主挂期间每个请求都要先失败一次。

#### 4.6.3 策略 B：带"失败记忆"的被动 Failover

```go
func onHttpRequestHeaders(ctx, numHeaders int, endOfStream bool) types.Action {
    // 读取 SharedData 中主最近失败的时间戳
    lastFailureTime, _ := proxywasm.GetSharedData("primary_last_failure")
    
    if time.Since(lastFailureTime) < memoryWindow { // 如 10 秒
        // 最近失败过，直接 dispatch 到备，不先尝试主
        dispatchToBackup(ctx, nil)
        return types.HeaderStopIteration
    }
    
    // 记忆窗口内没有失败记录，正常走 Router → 主
    return types.Continue
}

func onHttpResponseHeaders(ctx, numHeaders int, endOfStream bool) types.Action {
    status := proxywasm.GetHttpResponseHeader(":status")
    if isFailureStatus(status) {
        // 记录失败时间到 SharedData
        proxywasm.SetSharedData("primary_last_failure", nowBytes, 0)
        dispatchToBackup(ctx, requestBody)
        return types.HeaderStopIteration
    }
    return types.Continue
}
```

**优点**：避免每个请求都先失败（第一次除外），实现简单，无定期探测开销。
**缺点**：第一次请求仍然会失败；需要管理"记忆窗口"和恢复探测逻辑。

#### 4.6.4 策略 C：主动健康检查 + Preemptive 路由（V5 增强版）

```go
func onPluginStart(ctx) {
    // 定期探测主健康状态
    wrapper.RegisterTickFunc(healthCheckInterval, func() {
        if isPrimaryHealthy() {
            proxywasm.SetSharedData("primary_healthy", []byte("true"), 0)
        } else {
            proxywasm.SetSharedData("primary_healthy", []byte("false"), 0)
        }
    })
}

func onHttpRequestHeaders(ctx, numHeaders int, endOfStream bool) types.Action {
    healthy, _ := proxywasm.GetSharedData("primary_healthy")
    
    if string(healthy) == "false" {
        // 主不健康，直接 dispatch 到备
        dispatchToBackup(ctx, nil)
        return types.HeaderStopIteration
    }
    
    // 主健康，正常走 Router
    return types.Continue
}
```

**关键实现细节**：
- `HeaderStopIteration` 暂停原请求，Router 不再继续处理
- WASM 发起 `dispatch_http_call` 到备
- 备返回后，`SendHttpResponse` 直接返回响应并结束 stream
- 原请求被"跳过"，不会走到主 cluster

**优点**：真正的 preemptive，主挂期间新请求不会先尝试主。
**缺点**：
- 需要健康检查开销
- 主不健康期间，**所有请求**（包括正常非 failover 请求）都走 `dispatch_http_call` 路径
- SSE 流式请求在 preemptive 期间实时性会丧失（`dispatch_http_call` 缓冲完整响应）

#### 4.6.5 核心结论

> **WASM 虽然无法修改 Router 的 cluster 选择，但可以通过 `HeaderStopIteration` + `dispatch_http_call` + `SendHttpResponse` 的组合，在 HTTP Filter 层"拦截"请求并直接转发到备 cluster，实现类似 preemptive 的效果。**
>
> 三种策略的核心差异不是"能不能"，而是"愿不愿意承受什么 trade-off"：
> - 策略 A：接受"先失败后重试"，换取最简单实现
> - 策略 B：增加失败记忆，减少失败次数，换取中等复杂度
> - 策略 C：增加主动探测，实现真正 preemptive，换取健康检查开销和 SSE 实时性影响

---

## 五、流式请求（SSE）场景客观分析

### 5.1 所有 HA 方案对流式连接的共性限制

**已建立的 SSE 连接中途断开**：

| 方案 | 主在 SSE 传输中途挂了 | 说明 |
|------|---------------------|------|
| **V2 Aggregate Cluster** | ❌ 连接中断 | Aggregate Cluster 在请求开始时选择 cluster，无法在中途切换已建立的连接 |
| **V4 EndpointSlice** | ❌ 连接中断 | EndpointSlice 更新只影响新请求路由，不影响已建立连接 |
| **V5 WASM 被动 Failover** | ❌ 连接中断 | WASM `on_http_response_headers` 不会在流式传输中途触发 |

**核心结论**：已建立的 SSE 连接中途断开是**所有方案的共性限制**，不是 V5 特有的。任何基于 HTTP 请求级的 failover 机制都无法保护已建立的长连接。

### 5.2 新 SSE 请求（主在 headers 阶段失败）

| 方案 | 行为 | 客户端体验 |
|------|------|-----------|
| **V2 Aggregate Cluster** | Router 直接选择备 cluster，正常建立 SSE 连接 | ✅ 正常的实时 SSE 流 |
| **V4 EndpointSlice** | 新请求路由到健康端点，正常建立 SSE 连接 | ✅ 正常的实时 SSE 流 |
| **V5 WASM 被动 Failover** | 主返回 5xx → WASM 重试到备 → `dispatch_http_call` 缓冲完整响应后 `SendHttpResponse` 一次性发送 | ⚠️ 客户端延迟收到所有 SSE 事件（数据完整，但无实时性） |

**V5 的 SSE 新请求行为分析**：
- `dispatch_http_call` 的回调在响应**完成后**才触发，因此备的 SSE 响应会被完整缓冲
- `SendHttpResponse` 一次性发送缓冲的响应，客户端会延迟收到所有事件
- 这是一个**实际的行为差异**，但不是"致命"问题：
  - 客户端仍然能收到完整的数据
  - 对于 SSE，EventSource API 会自动重连，客户端可以在重连时正常恢复
  - 如果业务对实时性要求极高（如实时语音），这是一个需要权衡的点

### 5.3 客观评估

**V5 在 SSE 场景下的定位**：
- 非流式请求：✅ 完全可行，客户端无感知
- 新 SSE 请求（主 headers 失败）：⚠️ 可行，但备响应延迟发送（数据完整，实时性降低）
- 已建立 SSE 连接中断：❌ 所有方案共有，不是 V5 特有

**与其他方案的公平对比**：
- V2/V4 的优势在于 preemptive：新请求直接到备，SSE 正常流式
- V5 的优势在于无需 EnvoyFilter/额外组件：实现更轻量，维护更简单
- 已建立连接中断：所有方案都无法处理

---

## 六、拓扑感知与插件配置

### 6.1 L7 如何感知主备拓扑

WASM 插件本身运行在 HTTP Filter 层，它**不直接感知** K8s 的 Pod 拓扑。但通过以下方式，WASM 可以明确知道主备关系：

#### 方案 A：通过 `pluginConfig` 在 WasmPlugin CRD 中硬编码主备拓扑

这是最直接的方式，在 CRD 中显式声明每个 group 的主备 cluster。

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: smg-failover
  namespace: higress-system
spec:
  selector:
    matchLabels:
      higress: higress-system-higress-gateway
  pluginConfig:
    groups:
      - name: smg-group-1
        primary:
          cluster: "outbound|30002||smg-group-1-primary.default.svc.cluster.local"
          host: "smg-group-1-primary.default.svc.cluster.local"
        backups:
          - cluster: "outbound|30002||smg-group-1-backup-a.default.svc.cluster.local"
            host: "smg-group-1-backup-a.default.svc.cluster.local"
          - cluster: "outbound|30002||smg-group-1-backup-b.default.svc.cluster.local"
            host: "smg-group-1-backup-b.default.svc.cluster.local"
        retryTimeout: 5000
        retryStatuses: ["502", "503", "504"]
      - name: smg-group-2
        primary: ...
        backups: ...
```

**WASM 插件解析配置**：

```go
type FailoverConfig struct {
    Groups []GroupConfig `json:"groups"`
}

type GroupConfig struct {
    Name      string         `json:"name"`
    Primary   EndpointConfig `json:"primary"`
    Backups   []EndpointConfig `json:"backups"`
    RetryTimeout int64       `json:"retryTimeout"`
    RetryStatuses []string   `json:"retryStatuses"`
}

type EndpointConfig struct {
    Cluster string `json:"cluster"`
    Host    string `json:"host"`
}

func parseConfig(json gjson.Result, config *FailoverConfig) error {
    for _, group := range json.Get("groups").Array() {
        gc := GroupConfig{
            Name:         group.Get("name").String(),
            RetryTimeout: group.Get("retryTimeout").Int(),
        }
        gc.Primary.Cluster = group.Get("primary.cluster").String()
        gc.Primary.Host = group.Get("primary.host").String()
        for _, backup := range group.Get("backups").Array() {
            gc.Backups = append(gc.Backups, EndpointConfig{
                Cluster: backup.Get("cluster").String(),
                Host:    backup.Get("host").String(),
            })
        }
        config.Groups = append(config.Groups, gc)
    }
    return nil
}
```

#### 方案 B：通过服务命名约定自动推断主备拓扑

如果 SMG 服务的命名遵循统一约定，WASM 插件可以在**运行时从请求中自动推断**主备关系，无需在 CRD 中硬编码每个 group 的 cluster 列表。

**命名约定示例**：

```
smg-group-{group_id}-{instance_index}
```

| 组件 | 约定 | 示例 |
|-----|------|------|
| Group ID | `smg-group-1`, `smg-group-2` | 标识一个主备组 |
| Instance Index | `0` = 主, `1..N` = 备 | `smg-group-1-0` 是主，`smg-group-1-1` 是备 |

对应 K8s Service：

```yaml
# 主实例
apiVersion: v1
kind: Service
metadata:
  name: smg-group-1-0
spec:
  selector:
    app: smg-group-1
    instance: "0"
  ports:
    - port: 30002

# 备实例 1
apiVersion: v1
kind: Service
metadata:
  name: smg-group-1-1
spec:
  selector:
    app: smg-group-1
    instance: "1"
  ports:
    - port: 30002
```

**WASM 插件运行时推断逻辑**：

```go
// 从 :authority header 获取当前请求的 host
authority, _ := proxywasm.GetHttpRequestHeader(":authority")
// authority = "smg-group-1-0.default.svc.cluster.local"

// 解析 group 和实例序号
parts := strings.Split(authority, ".") // [smg-group-1-0, default, svc, cluster, local]
hostPart := parts[0]                    // smg-group-1-0

// 按约定推断主备
lastDash := strings.LastIndex(hostPart, "-")
baseName := hostPart[:lastDash]         // smg-group-1
indexStr := hostPart[lastDash+1:]       // 0

// 主失败时，依次尝试备
backupIndex := 1
for backupIndex <= maxBackupCount {
    backupHost := fmt.Sprintf("%s-%d.default.svc.cluster.local", baseName, backupIndex)
    backupCluster := fmt.Sprintf("outbound|30002||%s", backupHost)
    // dispatch_http_call(backupCluster, ...)
    backupIndex++
}
```

**CRD 配置（仅配置全局规则，无需硬编码每个 group）**：

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: smg-failover
  namespace: higress-system
spec:
  defaultConfig:
    # 命名约定规则
    namingConvention:
      enabled: true
      primarySuffix: "-0"           # 主的实例序号后缀
      backupRange: [1, 3]           # 备的序号范围 1~3
      servicePort: 30002
      namespace: "default"
    # 可选：限制哪些 host 启用 failover
    hostPrefix: "smg-group-"
    retryTimeout: 5000
    retryStatuses: ["502", "503", "504"]
```

#### 方案对比

| 维度 | 方案 A：CRD 硬编码 | 方案 B：命名约定 |
|------|-------------------|-----------------|
| **配置量** | 每个 group 需单独声明主备 cluster | 一条全局规则，自动适配所有 group |
| **新增 group** | 需修改 CRD 并 apply | 只要命名符合约定，无需修改配置 |
| **灵活性** | 高——每个 group 可独立配置超时、状态码 | 中——全局统一规则，group 间行为一致 |
| **命名约束** | 无——任何命名都可以 | 有——必须严格遵守约定 |
| **可读性** | 高——配置即文档 | 中——需要理解命名约定 |
| **运维复杂度** | 中——N 个 group 需维护 N 份配置 | 低——配置一次，动态适配 |

**推荐**：
- 如果 SMG 服务命名已经统一（或可以统一），**优先使用方案 B**，大幅降低运维负担
- 如果不同 group 的主备策略差异大（如有的 group 有 2 个备、有的有 5 个），或命名无法统一，使用方案 A
- **两种方案可以共存**：CRD 支持 `groups`（硬编码）和 `namingConvention`（自动推断）同时配置，显式配置的 group 优先

### 6.2 Cluster 名称的来源

WASM 插件中使用的 cluster 名称，来源于 Istio/Higress 为 K8s Service 自动生成的 Envoy cluster：

| K8s 资源 | Envoy cluster 名称格式 | 示例 |
|---------|----------------------|------|
| K8s Service | `outbound\|<port>\|\|<svc>.<ns>.svc.cluster.local` | `outbound\|30002\|\|smg-group-1-primary.default.svc.cluster.local` |
| ServiceEntry | `outbound\|<port>\|\|<host>` | `outbound\|30002\|\|smg-group-1-primary.local` |

**获取 cluster 名称的方法**：
1. **通过 Envoy admin API**：`curl http://localhost:15000/clusters | grep smg-group`
2. **通过 proxy-wasm**：`proxywasm.GetProperty([]string{"cluster_name"})` 获取当前请求的 cluster
3. **通过 Higress 控制台**：查看服务列表中的 "upstream" 名称

### 6.3 备 Cluster 的创建

**关键前提**：备 cluster 必须在 Envoy 中已存在，否则 `dispatch_http_call` 会失败。

**确保备 cluster 存在的方法**：

1. **为每个 SMG 备实例创建独立的 K8s Service**：
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: smg-group-1-backup-a
     namespace: default
   spec:
     selector:
       app: smg-group-1
       role: backup-a
     ports:
       - port: 30002
         targetPort: 30002
   ```
   - Istio 会自动为该 Service 创建 Envoy cluster
   - WASM 可以通过 `outbound|30002||smg-group-1-backup-a.default.svc.cluster.local` 访问

2. **使用 ServiceEntry（如果备不在 K8s 中）**：
   ```yaml
   apiVersion: networking.istio.io/v1beta1
   kind: ServiceEntry
   metadata:
     name: smg-group-1-backup-a
     namespace: higress-system
   spec:
     hosts:
       - smg-group-1-backup-a.local
     ports:
       - number: 30002
         protocol: HTTP
     endpoints:
       - address: 10.0.1.101
   ```

### 6.4 路由匹配与 Group 选择

WASM 插件需要知道当前请求属于哪个 group。有几种方式：

**方式 A：通过请求 path 匹配**
```go
func getGroupByPath(path string, config FailoverConfig) *GroupConfig {
    for _, g := range config.Groups {
        if strings.HasPrefix(path, "/v1/chat/"+g.Name) {
            return &g
        }
    }
    return nil
}
```

**方式 B：通过 VirtualService 的 `matchRules` 在 WasmPlugin CRD 中绑定**
```yaml
spec:
  matchRules:
    - ingress:
        - default/smg-group-1-route
      config:
        groupName: smg-group-1
    - ingress:
        - default/smg-group-2-route
      config:
        groupName: smg-group-2
```

**方式 C：通过当前请求的 cluster 名称反推**
```go
clusterName, _ := proxywasm.GetProperty([]string{"cluster_name"})
// clusterName = "outbound|30002||smg-group-1-primary.default.svc.cluster.local"
// 解析出 group = "smg-group-1"
```

---

## 七、K8s 部署运维实践

### 7.1 开发环境准备

```bash
# 1. 初始化项目
mkdir smg-failover-plugin && cd smg-failover-plugin
go mod init smg-failover-plugin

# 2. 下载依赖
go get github.com/higress-group/proxy-wasm-go-sdk@go-1.24
go get github.com/higress-group/wasm-go@main
go get github.com/tidwall/gjson
```

### 7.2 编译

```bash
# 方式 1：使用标准 Go 编译器（Go 1.24+，推荐）
go mod tidy
GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared -o smg-failover.wasm ./

# 方式 2：如果在 Higress 官方插件目录中，使用 Makefile
# PLUGIN_NAME=smg-failover make build
```

### 7.3 打包 OCI 镜像

```dockerfile
# Dockerfile
FROM scratch
COPY smg-failover.wasm ./
```

```bash
# 构建并推送
docker build -t <registry>/higress-plugins/smg-failover:v1.0.0 .
docker push <registry>/higress-plugins/smg-failover:v1.0.0
```

**私有镜像仓库**：
- 在 WasmPlugin CRD 中配置 `imagePullSecret`
- 或在 Higress Gateway Deployment 中挂载 registry credentials

### 7.4 部署 WasmPlugin CRD

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: smg-failover
  namespace: higress-system
spec:
  selector:
    matchLabels:
      higress: higress-system-higress-gateway
  # 执行阶段：AUTHZ（认证之后，路由之前）
  phase: AUTHZ
  # 优先级：确保在业务插件之前执行
  priority: 100
  url: oci://<registry>/higress-plugins/smg-failover:v1.0.0
  imagePullPolicy: IfNotPresent
  pluginConfig:
    groups:
      - name: smg-group-1
        primary:
          cluster: "outbound|30002||smg-group-1-primary.default.svc.cluster.local"
          host: "smg-group-1-primary.default.svc.cluster.local"
        backups:
          - cluster: "outbound|30002||smg-group-1-backup-a.default.svc.cluster.local"
            host: "smg-group-1-backup-a.default.svc.cluster.local"
          - cluster: "outbound|30002||smg-group-1-backup-b.default.svc.cluster.local"
            host: "smg-group-1-backup-b.default.svc.cluster.local"
        retryTimeout: 5000
        retryStatuses: ["502", "503", "504"]
```

```bash
kubectl apply -f smg-failover-wasmplugin.yaml
```

### 7.5 更新与回滚

**更新插件**：
```bash
# 1. 修改代码，重新编译、打包新版本
docker build -t <registry>/higress-plugins/smg-failover:v1.0.1 .
docker push <registry>/higress-plugins/smg-failover:v1.0.1

# 2. 更新 WasmPlugin CRD
kubectl patch wasmplugin smg-failover -n higress-system \
  --type merge -p '{"spec":{"url":"oci://<registry>/higress-plugins/smg-failover:v1.0.1"}}'
```

**热加载特性**：
- Envoy 通过 ECDS（Extension Config Discovery Service）机制加载 WASM 插件
- 更新 WasmPlugin CRD 后，Envoy **无需重启**即可加载新插件
- 已建立的连接**不会中断**，业务流量完全无损

**回滚**：
```bash
kubectl patch wasmplugin smg-failover -n higress-system \
  --type merge -p '{"spec":{"url":"oci://<registry>/higress-plugins/smg-failover:v1.0.0"}}'
```

### 7.6 监控与调试

**查看插件日志**：
```bash
# Higress Gateway 日志中包含 WASM 插件的输出
kubectl logs -n higress-system -l app=higress-gateway -f | grep "smg-failover"
```

**开启 WASM debug 日志**：
```bash
kubectl exec -n higress-system <higress-pod> -c istio-proxy -- \
  curl -XPOST "localhost:15000/logging?wasm=debug"
```

**查看 Envoy 加载的 WASM 插件**：
```bash
kubectl exec -n higress-system <higress-pod> -c istio-proxy -- \
  curl -s "localhost:15000/config_dump" | jq '.configs[] | select(.["@type"]=="type.googleapis.com/envoy.admin.v3.ExtensionsConfig")'
```

**验证 failover 行为**：
```bash
# 1. 正常请求
curl http://api.example.com/v1/chat/completions -d '{"model":"gpt-4"}'

# 2. 模拟主节点故障（kill SMG primary）
kubectl exec -it smg-group-1-primary-0 -- pkill sgl-model-gateway

# 3. 再次请求，观察是否无感知切换到备
# 客户端应收到正常响应，无 5xx 错误
```

### 7.7 生产环境 checklist

| 检查项 | 说明 |
|--------|------|
| **镜像仓库可用性** | 确保 Higress Gateway Pod 能拉取 OCI 镜像（网络连通、认证正确） |
| **备 Service 健康** | 确认备 cluster 对应的 K8s Service 和 EndpointSlice 正常 |
| **Cluster 名称正确** | 通过 Envoy admin `/clusters` 确认 cluster 名称与 pluginConfig 一致 |
| **超时配置合理** | `retryTimeout` 应小于客户端超时，建议 3-5s |
| **重试状态码** | 根据 SMG 的故障模式配置（如 502/503/504，或连接超时） |
| **日志级别** | 生产环境建议 `info` 级别，避免 `debug` 产生过多日志 |
| **资源限制** | WASM 插件内存消耗主要来自 body 缓存，建议设置合理的 VM 内存限制 |

---

## 八、综合结论

### 6.1 WASM 单独实现 SMG 主备 Failover

| 需求 | WASM 能力 | 结论 |
|------|----------|------|
| **Preemptive failover**（主不健康时自动切流量） | ❌ 无法修改 Router 的 cluster 选择 | **不可行** |
| **被动 failover**（主失败后重试到备，客户端无感知）非流式 | ✅ `dispatch_http_call` + `SendHttpResponse` | **可行** |
| **被动 failover**（主失败后重试到备）新 SSE 请求 | ⚠️ 备响应被缓冲后发送 | 可行，实时性降低 |
| **已建立 SSE 连接中断保护** | ❌ 所有方案都无法处理 | **共性限制** |
| **主动健康检查** | ✅ `RegisterTickFunc` + `dispatch_http_call` | 可行，但被动 failover 不需要 |
| **多级依次降级**（主→备A→备B） | ✅ 回调中递归重试 | **可行** |

### 6.2 WASM 在 HA 架构中的定位

**WASM 可以作为 SMG 主备被动 failover 的核心实现**，具有以下价值：

1. **独立的被动 failover 方案**：无需 EnvoyFilter，无需额外组件，纯 WASM 插件实现
2. **客户端无感知**：拦截 5xx，重试到备，返回成功响应
3. **多级依次降级**：Go 代码中完全可控，主→备A→备B 递归重试
4. **业务级增强**：在 failover 前后做自定义逻辑（如监控、日志、熔断策略）
5. **轻量级探测**（可选）：用 `RegisterTickFunc` 做 L7 探测，结果写入 `SharedData`

### 6.3 V5 vs V2 vs V4 公平对比

| 维度 | V2: EnvoyFilter + Aggregate Cluster | V4: Operator + EndpointSlice | V5: WASM 被动 Failover |
|------|------------------------------------|------------------------------|------------------------|
| **核心机制** | Router Aggregate Cluster | Operator HTTP 探测 + 动态 EndpointSlice | WASM `dispatch_http_call` 重试 |
| **Preemptive failover** | ✅ | ✅ | ❌ |
| **被动 failover（客户端无感知）** | ✅ 非流式 | ✅ 非流式 | ✅ 非流式 |
| **新 SSE 请求（主失败）** | ✅ 正常 SSE | ✅ 正常 SSE | ⚠️ 缓冲后发送 |
| **已建立 SSE 中断** | ❌ | ❌ | ❌ |
| **多级依次降级** | ✅ | ✅ | ✅ |
| **需要 EnvoyFilter** | ✅ | ❌ | ❌ |
| **需要额外组件** | ❌ | ✅ Operator | ❌ |
| **探测开销** | 有（Envoy active HC） | 有（Operator 探测） | **无** |
| **切换延迟** | ~2s（探测周期） | ~1-2s（探测+传播） | ~1 RTT（主失败即切换） |
| **实现复杂度** | 中（EnvoyFilter YAML） | 中（开发 Operator） | **低**（Go 插件） |
| **与 Istio 版本耦合** | 高（EnvoyFilter 底层） | **无**（标准 K8s API） | **无**（WASM 插件） |
| **可维护性** | 低（protobuf  hack） | 高（标准 API） | **高**（Go 代码，易调试） |

### 6.4 最终结论

> **WASM 插件无法单独实现 preemptive failover**（这是 proxy-wasm ABI 的设计限制）。
>
> **但 WASM 插件完全可以实现客户端无感知的被动 failover**：主返回 5xx 后，`dispatch_http_call` 到备 cluster 重试，`SendHttpResponse` 返回备的成功响应。实现路径清晰，非流式场景下生产就绪。
>
> **V5 的核心价值**：
> - 无需 EnvoyFilter，与 Istio/Higress 版本完全解耦
> - 无需额外组件（如 Operator），部署简单
> - 无探测开销，失败即切换
> - 客户端无感知，多级降级完全可控
>
> **V5 的固有 trade-off**：
> - 无法 preemptive：每个新请求先尝试主（但主完全不可达时 TCP 连接失败很快返回）
> - SSE 新请求场景：备响应被缓冲后发送，实时性降低（数据完整）
> - 已建立 SSE 连接中断：所有方案共有，不是 V5 特有
>
> **关于 cache miss**：主备切换后 cache 无法命中是所有 HA 方案的共性问题（V2 Aggregate Cluster、V4 EndpointSlice 切换后同样面临），并非 V5 特有缺点。
>
> **建议**：
> - 如果追求**preemptive failover**（新请求直接到备）且能接受 EnvoyFilter：使用 **V2**
> - 如果追求**preemptive failover**且希望纯 K8s 原生：使用 **V4**
> - 如果追求**极简部署**（无 EnvoyFilter、无额外组件）、可接受**被动 failover**：使用 **V5**
> - V5 和 V2/V4 不是互斥的：可以先用 V5 快速上线，后续根据需要迁移到 V2 或 V4
>
> **关于扩展机制的选择**：V5 使用 WASM 而非 Lua Filter，因为 WASM 具有沙箱隔离、Higress 原生 WasmPlugin CRD 支持、Go SDK 生态、OCI 热更新等优势。Lua Filter 虽能实现类似功能，但无进程隔离、需通过 EnvoyFilter 注入、性能更差，不适合生产级 failover（详见 4.5 节）。

### 6.5 V5 方案落地可行性评估

#### 可行性维度分析

| 维度 | 评估 | 说明 |
|------|------|------|
| **技术可行性** | ✅ **高** | proxy-wasm ABI 提供了所有需要的功能（`dispatch_http_call`、`SendHttpResponse`、`SharedData`、`RegisterTickFunc`） |
| **实现复杂度** | **中** | 需要处理 body 缓存、多级降级、超时控制、错误处理，但 Higress Go SDK 已有成熟封装 |
| **生产稳定性** | ⚠️ **需要关注** | 已知 `dispatch_http_call` "no healthy upstream" 崩溃风险（Istio issue #31194）；body 缓存需正确实现 |
| **性能影响** | **低-中** | `dispatch_http_call` 独立连接池有额外开销，但 failover 是低频事件，正常请求走 Router 不受影响 |
| **SSE 实时性** | ⚠️ **有影响** | 纯被动场景仅 failover 请求受影响；preemptive 增强版主挂期间所有 SSE 请求实时性降低 |
| **Higress 集成** | ✅ **成熟** | 原生 WasmPlugin CRD、Go SDK、热更新、OCI 分发，生态完整 |
| **运维复杂度** | **低** | 一个 WASM 插件 + 命名约定或 CRD 配置，无 EnvoyFilter、无额外组件 |

#### 落地路径建议

| 阶段 | 目标 | 推荐策略 | 预计时间 |
|------|------|---------|---------|
| **Phase 1：MVP** | 快速上线，解决"有总比没有好" | **策略 A（纯被动）** | 1-2 周 |
| **Phase 2：增强** | 减少失败延迟，提升体验 | **策略 B（失败记忆）** | +1 周 |
| **Phase 3：演进** | 根据实际数据决定方向 | 若 preemptive 需求强烈 → 策略 C；若 SSE 实时性要求高 → 迁移到 V2/V4 | 按需 |

#### 核心风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| `dispatch_http_call` 崩溃（no healthy upstream） | Envoy 进程崩溃 | 确保备 cluster 始终有健康 endpoint（K8s readiness probe）；WASM 中捕获 `internal failure` 错误；监控 Envoy 进程重启频率 |
| body 丢失（ai-proxy issue #3531） | 重试请求无 body | 在 `on_http_request_body` 中正确缓存 body 到 context；`dispatch_http_call` 时显式设置 body 和 `Content-Length` |
| 多级降级失败（备 A 挂了再试备 B） | 客户端收到 5xx | 回调中递归实现降级；每个级别独立超时；最后级别失败后返回原始错误 |
| SSE 流式实时性 | 主挂期间 SSE 延迟 | 根据业务需求选择策略：纯被动影响最小；若不能接受，考虑 V2/V4 |
| 主恢复后流量切回 | 策略 B/C 需要恢复探测 | 策略 B：定期让请求尝试走主；策略 C：健康检查周期内自动恢复 |

#### 最终可行性结论

> **V5 方案落地可行性：80%+**
>
> - **最小可行版本（策略 A 纯被动）**：技术成熟，实现路径清晰，适合快速上线。核心价值是"无需 EnvoyFilter、无需额外组件、客户端无感知"。
> - **增强版本（策略 B/C）**：在最小版本基础上增加健康检查或失败记忆，1 个月内可完成。
> - **关键前提**：正确实现 body 缓存、妥善处理 `dispatch_http_call` 错误、充分的边界测试（备全挂、多级降级、SSE 场景）。
> - **迁移路径**：V5 和 V2/V4 不是互斥的。可以先用 V5 策略 A 快速解决主备 failover 的有无问题，后续根据实际运行数据和业务需求，决定是否增强为策略 B/C，或迁移到 V2/V4。

---

## 九、附录：参考来源

- proxy-wasm/spec#16 — WASM 无法修改 upstream/cluster 选择（至今 open）
  https://github.com/proxy-wasm/spec/issues/16

- envoyproxy/envoy#28673 — WASM `setProperty` 前缀 `wasm.` 限制，无法设置 filter state
  https://github.com/envoyproxy/envoy/issues/28673

- Higress ai-proxy `failover.go` — 主动健康检查实现（`RegisterTickFunc` + `dispatch_http_call`）
  https://github.com/alibaba/higress/blob/main/plugins/wasm-go/extensions/ai-proxy/provider/failover.go

- Higress ai-proxy `retry.go` — 被动 retry 实现（`RouteCluster` + `dispatch_http_call`）
  https://github.com/alibaba/higress/blob/main/plugins/wasm-go/extensions/ai-proxy/provider/retry.go

- Higress ai-proxy issue #3531 — retry 时 request body 丢失（`Content-Length: 0`）
  https://github.com/alibaba/higress/issues/3531

- Higress wasm-go `cluster_wrapper.go` — `RouteCluster` / `TargetCluster` / `K8sCluster` 定义
  https://github.com/higress-group/wasm-go/blob/main/pkg/wrapper/cluster_wrapper.go

- Higress WASM Go 插件开发指南
  https://higress.ai/en/docs/latest/user/wasm-go/

- Higress 插件 Go SDK 处理流程文档（`HeaderStopIteration` 等状态说明）
  https://higress.ai/en/docs/latest/user/plugins/wasm-dev/wasm16/

- Envoy `ext_proc` (External Processing) Filter 官方文档
  https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_proc_filter

- Envoy WASM Filter 官方文档
  https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/wasm_filter

- Higress AI-Proxy 插件 failover + retryOnFailure 功能验证 issue #2159
  https://github.com/higress-group/higress/issues/2159

- Higress `GetUpstreamHosts()` 扩展 ABI 讨论 #3546
  https://github.com/higress-group/higress/discussions/3546

- proxy-wasm-go-sdk issue #438 — `DispatchHttpCall` 走 AsyncClient 路径（日志显示 `async_client_impl.cc`）
  https://github.com/tetratelabs/proxy-wasm-go-sdk/issues/438

- Istio issue #31194 — WASM `proxy_http_call` 在目标 cluster "no healthy upstream" 时导致 Envoy 崩溃
  https://github.com/istio/istio/issues/31194

- Higress `properties` 包文档 — `GetClusterName`、`GetRouteMetadata` 等
  https://pkg.go.dev/github.com/higress-group/proxy-wasm-go-sdk/properties

- Higress 插件部署与配置文档（WasmPlugin CRD、`pluginConfig`、`matchRules`）
  https://higress.ai/en/docs/latest/plugins/intro/

- Higress WASM Go Plugin 开发 Skill（项目初始化、编译、打包完整流程）
  https://lobehub.com/skills/alibaba-higress-higress-wasm-go-plugin

- Higress 开源 Plugin Server（私有部署场景下替代 OCI 仓库的 HTTP 分发方案）
  https://www.alibabacloud.com/blog/higress-open-source-plugin-server-simplifies-the-challenges-of-private-deployment-for-wasm-plugins_602345

- Higress 实战：30 行代码写一个 Wasm Go 插件
  https://higress.io/blog/higress-gvr7dx_awbbpb_fi4t7317cvughlzh/

- Istio WasmPlugin API 官方文档
  https://istio.io/latest/docs/reference/config/proxy_extensions/wasm-plugin/

- Higress WASM Go 插件本地调试与 `DispatchHttpCall` 使用指南
  https://higress.ai/en/docs/latest/user/wasm-go/
