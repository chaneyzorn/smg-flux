# SMG HA v5.1 实现细节设计

本文档基于 V5.1 调研结论，将方案落地为可执行的代码设计。所有接口签名、数据结构和控制流均参考 `sglang` 源码和 `higress/plugins/wasm-go` 框架的实际 API。

---

## 1. 拓扑感知与健康检查

### 1.1 拓扑来源：自主探测，不依赖 K8s/Higress 被动健康状态

**设计决策**：本插件**不**使用 Envoy 的 `GetUpstreamHosts()` 返回的 `health_status` 作为路由决策依据，而是**自主维护**一套健康状态。

**原因**：
- Envoy 的被动健康检查（outlierDetection）基于 5xx 比例，有时间窗口（通常需要数秒到数十秒的统计周期），无法做到亚秒级故障感知
- Envoy 的主动健康检查在 Higress 开源版中配置受限，且探测周期不可精细控制
- 我们需要基于 SMG 的 `/readiness` 语义做探测（PD 模式需要同时有 prefill 和 decode worker），而 Envoy 的 health check 只能做简单的 TCP/HTTP 探测，不理解 SMG 的业务语义

**目标拓扑结构**：

```
                         外部请求
                            │
                            ▼
                   ┌─────────────────┐
                   │  Higress Gateway │
                   │  (外层 weight/   │
                   │   轮询到 Group)  │
                   └────────┬────────┘
                            │
                   ┌────────▼────────┐
                   │  wasm-go Plugin  │
                   │  (按 Group 选择  │
                   │   healthy ep,    │
                   │   SetUpstream    │
                   │   OverrideHost)  │
                   └────────┬────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
   │ Group-1     │   │ Group-2     │   │ Group-3     │
   │ (Envoy      │   │ (Envoy      │   │ (Envoy      │
   │  cluster)   │   │  cluster)   │   │  cluster)   │
   │             │   │             │   │             │
   │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │
   │ │SMG-1.1  │ │   │ │SMG-2.1  │ │   │ │SMG-3.1  │ │
   │ │★locked  │ │   │ │★locked  │ │   │ │★locked  │ │
   │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │
   │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │
   │ │SMG-1.2  │ │   │ │SMG-2.2  │ │   │ │SMG-3.2  │ │
   │ │  standby│ │   │ │  standby│ │   │ │  standby│ │
   │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │
   │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │
   │ │SMG-1.3  │ │   │ │SMG-2.3  │ │   │ │SMG-3.3  │ │
   │ │  standby│ │   │ │  standby│ │   │ │  standby│ │
   │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │
   └─────────────┘   └─────────────┘   └─────────────┘
```

- 外层 Higress 通过 VirtualService weight/轮询将请求路由到某个 Group（Envoy cluster）
- **在请求到达具体 endpoint 之前**，wasm-go Plugin 拦截请求，读取当前 Group 的 endpoint 列表，选择 healthy endpoint 并**稳定锁定**（`SetUpstreamOverrideHost`）
- 所有请求集中到锁定的 endpoint，避免轮询导致的 cache tree 分散
- 每个 Group 内部的 endpoint 完全平等，不存在固定的"主"和"备"

**结论**：拓扑信息来自三个来源：
1. **WasmPlugin CRD 的 `defaultConfig`** — 静态声明每个 Group 的 endpoint 列表和探测参数
2. **周期性 tick health check** — `RegisterTickFunc` 定期向各 endpoint 的 `/readiness` 发送探测，自主判定健康状态
3. **实时请求失败时的即时健康评估** — `OnHttpResponseHeaders` 中检测到 5xx 后，异步触发一次对该 endpoint 的即时 `/readiness` 探测，不等下一个 tick 周期，立即确认并更新状态

### 1.1.2 与 Higress 原生机制的兼容性

**关键问题**：本插件使用 `SetUpstreamOverrideHost` 强制指定 endpoint，这会与 Higress/Envoy 原生的负载均衡、健康检查和熔断机制产生怎样的交互？

**Envoy 处理请求的完整链路**：

```
Client Request
  │
  ▼
VirtualService 路由匹配 → 选定 upstream cluster（Group）
  │
  ▼
Envoy 负载均衡 → 根据 ServiceEntry endpoints 选择默认 endpoint
  │
  ▼
WASM Plugin（OnHttpRequestHeaders）
  ├──→ 插件自主决策 → SetUpstreamOverrideHost(endpoint-X)
  │       （覆盖 Envoy 的负载均衡选择）
  │
  ▼
Envoy 连接管理（对 endpoint-X）
  ├──→ 连接池检查（max_connections / max_pending_requests）
  ├──→ 连接建立 / 复用
  │
  ▼
请求发送到 endpoint-X
  │
  ▼
Envoy 被动健康检查统计（outlierDetection）
  ├──→ 记录 endpoint-X 的 5xx 率
  ├──→ 若 5xx 率超标，将 endpoint-X 从 Envoy 负载均衡池移除
  │       （不影响 SetUpstreamOverrideHost，override 是强制性的）
```

**逐项分析**：

| Higress 原生机制 | 是否仍生效 | 与插件的交互 | 影响评估 |
|-----------------|-----------|-------------|---------|
| **ServiceEntry DNS 解析** | ✅ 生效 | Envoy 仍通过 DNS 解析 endpoints，维护地址列表 | 无影响，插件 override 的地址必须在 ServiceEntry 的 endpoints 中 |
| **Envoy 负载均衡** | ⚠️ 被覆盖 | `SetUpstreamOverrideHost` 强制指定 endpoint，绕过 Envoy 的轮询/加权选择 | 这是设计意图，插件接管了 endpoint 选择 |
| **outlierDetection（被动健康检查）** | ✅ 仍统计 | Envoy 仍统计被 override endpoint 的 5xx 率，可能将其标记为不健康 | 两端健康状态可能不一致：Envoy 认为 unhealthy，但插件自主探测认为 healthy。`SetUpstreamOverrideHost` 是强制性的，不受 Envoy 健康状态限制 |
| **连接池 / 熔断** | ✅ 仍生效 | max_connections、max_pending_requests 等限制仍对被 override 的 endpoint 生效 | 如果连接池已满，请求会失败，插件的被动重试会处理 |
| **主动健康检查（若配置）** | ✅ 仍探测 | Envoy 可能向 endpoints 发送独立的健康探测 | 对插件无直接影响，但会增加 endpoint 的探测负载 |
| **DestinationRule 超时 / 重试** | ✅ 仍生效 | 请求超时、重试次数等仍由 Envoy 控制 | 插件重试与 Envoy 重试是独立的，注意避免重试风暴 |

**核心结论**：

1. **`SetUpstreamOverrideHost` 只覆盖"负载均衡选择"这一步**，后续 Envoy 的连接管理、熔断、被动健康检查统计仍然正常运行。

2. **两端健康状态可能短暂不一致**：Envoy 的 outlierDetection 基于 5xx 比例（需要多个样本、数秒窗口），而插件的自主探测基于 `/readiness` 语义（亚秒级）。可能出现 Envoy 认为 unhealthy、插件认为 healthy，或反之的情况。但由于 `SetUpstreamOverrideHost` 是强制性的，请求始终会发送到插件选定的 endpoint。

3. **推荐做法**：保持插件自主健康检查为**唯一权威**来源，Envoy 的 outlierDetection 仅作为**辅助监控**参考。不在 DestinationRule 中配置复杂的 outlierDetection 参数，避免两端健康状态差异过大造成困惑。

4. **可选增强**：在 `selectHealthyEndpoint` 中，将 Envoy 的 `GetUpstreamHosts()` 返回的 `health_status` 作为**次要参考**（权重较低）。当插件自主探测和 Envoy 健康状态都标记为 unhealthy 时，提高置信度。

```go
// 可选：将 Envoy health_status 作为辅助参考
func selectHealthyEndpoint(...) *EndpointConfig {
    // ... 原有逻辑 ...
    
    // 辅助参考：Envoy 的被动健康状态
    upstreamHosts, _ := proxywasm.GetUpstreamHosts()
    envoyHealthy := make(map[string]bool)
    for _, host := range upstreamHosts {
        // host[0] = address, host[1] = metrics（含 health_status）
        if strings.Contains(host[1], "health_status=Healthy") {
            envoyHealthy[host[0]] = true
        }
    }
    
    // 如果插件探测认为 healthy，但 Envoy 认为 unhealthy，记录日志但不改变决策
    // 如果两者都认为 unhealthy，提高不健康置信度
}
```

### 1.1.3 静态拓扑获取：Config 解析

插件启动时，通过 `proxywasm.GetPluginConfiguration()` 读取 WasmPlugin CRD 的 `defaultConfig`，解析为 Go struct。

**Config 结构**：

```go
type PluginConfig struct {
    Groups  []GroupConfig  `json:"groups"`  // 对应 CRD 中的 groups 数组
    Global  GlobalConfig   `json:"global"`  // 全局配置
}

type GroupConfig struct {
    Name             string            `json:"name"`
    Cluster          string            `json:"cluster"`           // Envoy cluster 名称
    Endpoints        []EndpointConfig  `json:"endpoints"`         // endpoint 列表
    HealthCheck      HealthCheckConfig `json:"healthCheck"`
    FailbackStrategy string            `json:"failbackStrategy"`  // auto | manual | sticky
}

type EndpointConfig struct {
    Name     string `json:"name"`     // 如 "smg-1-1"
    Address  string `json:"address"`  // 如 "smg-1-1.default.svc.cluster.local:30002"
    Priority int    `json:"priority"` // 锁定优先级，0 最高
}
```

**解析入口**：参考 `ai-proxy` 的 `parseGlobalConfig` 模式

```go
func parseConfig(json gjson.Result, pluginCtx *wrapper.CommonPluginCtx[PluginConfig]) (PluginConfig, error) {
    var config PluginConfig
    
    // gjson 解析 CRD 配置
    config.Global.CacheTTL = json.Get("global.cacheTTL").Int()
    if config.Global.CacheTTL == 0 {
        config.Global.CacheTTL = 2000 // 默认 2s
    }
    config.Global.LeaseTTL = json.Get("global.leaseTTL").Int()
    if config.Global.LeaseTTL == 0 {
        config.Global.LeaseTTL = 60 // 默认 60s
    }
    
    // 解析 groups 数组
    groups := json.Get("groups").Array()
    for _, g := range groups {
        group := GroupConfig{
            Name:             g.Get("name").String(),
            Cluster:          g.Get("cluster").String(),
            FailbackStrategy: g.Get("failbackStrategy").String(),
        }
        if group.FailbackStrategy == "" {
            group.FailbackStrategy = "sticky"
        }
        
        // 解析 endpoints
        endpoints := g.Get("endpoints").Array()
        for _, ep := range endpoints {
            group.Endpoints = append(group.Endpoints, EndpointConfig{
                Name:     ep.Get("name").String(),
                Address:  ep.Get("address").String(),
                Priority: int(ep.Get("priority").Int()),
            })
        }
        
        // 解析 healthCheck
        hc := g.Get("healthCheck")
        group.HealthCheck = HealthCheckConfig{
            Interval:         hc.Get("interval").Int(),
            Timeout:          hc.Get("timeout").Int(),
            FailureThreshold: hc.Get("failureThreshold").Int(),
            SuccessThreshold: hc.Get("successThreshold").Int(),
            CooldownDuration: hc.Get("cooldownDuration").Int(),
        }
        if group.HealthCheck.Interval == 0 {
            group.HealthCheck.Interval = 3000
        }
        if group.HealthCheck.Timeout == 0 {
            group.HealthCheck.Timeout = 5000
        }
        if group.HealthCheck.FailureThreshold == 0 {
            group.HealthCheck.FailureThreshold = 2
        }
        if group.HealthCheck.SuccessThreshold == 0 {
            group.HealthCheck.SuccessThreshold = 2
        }
        
        config.Groups = append(config.Groups, group)
    }
    
    return config, nil
}
```

**配置注入链路**：

```
WasmPlugin CRD (defaultConfig: {...})
      │
      ▼
Higress Controller → Istio WasmPlugin (pluginConfig)
      │
      ▼
Envoy xDS 下发
      │
      ▼
proxywasm.GetPluginConfiguration() → []byte(JSON)
      │
      ▼
parseConfig() → PluginConfig struct
      │
      ▼
插件内存中持有 config 引用
```

`proxywasm.GetPluginConfiguration()` 在 `OnPluginStart` 阶段被 wasm-go 框架自动调用，解析后的配置通过泛型参数注入到 `CommonPluginCtx[PluginConfig]` 中，后续所有请求处理函数（`onHttpRequestHeaders` 等）均可通过参数访问。

### 1.2 健康检查端点与协议

**探测目标**：SMG 的 `/readiness` 端点（`sgl-model-gateway/src/server.rs:102-144`）

```
GET /readiness HTTP/1.1
Host: <endpoint-address>
```

**响应语义**：
- `200 OK` + body `{"status":"ready","healthy_workers":N,"total_workers":M}` → healthy
- `503 Service Unavailable` + body `{"status":"not ready","reason":"insufficient healthy workers"}` → unhealthy

**PD 模式下的额外检查**：SMG 的 `readiness()` 在 `RoutingMode::PrefillDecode` 模式下会检查 `has_prefill && has_decode`。这意味着即使 SMG 进程存活，如果没有健康的 P/D worker 配对，也会返回 503。这比单纯的 TCP 探测更准确。

**探测实现**：使用 `wrapper.HttpCall` 或 `wrapper.NewClusterClient(wrapper.FQDNCluster{...})` 发送异步 HTTP GET 请求。

```go
client := wrapper.NewClusterClient(wrapper.FQDNCluster{
    Host: endpointHost,  // 如 "smg-1-1.default.svc.cluster.local"
    Port: endpointPort,  // 30002
})
client.Get("/readiness", nil, func(statusCode int, respHeaders http.Header, respBody []byte) {
    // statusCode == 200 → healthy
    // statusCode == 503 → unhealthy
}, uint32(timeoutMs))
```

### 1.3 健康检查策略

**配置结构**：

```go
type HealthCheckConfig struct {
    Interval        int64  // 探测周期，默认 3000ms
    Timeout         int64  // 探测超时，默认 5000ms
    FailureThreshold int64  // 连续失败次数阈值，默认 2
    SuccessThreshold int64  // 连续成功次数阈值，默认 2
    CooldownDuration int64  // 冷却恢复时间（ms），0 表示不启用
}
```

**状态机**：

```
                    ┌─────────────┐
                    │   Unknown   │
                    └──────┬──────┘
                           │ 首次探测成功
                           ▼
                    ┌─────────────┐
       连续 failure │   Healthy   │◄────────┐
       达阈值       └──────┬──────┘         │
                           │                │ 连续 success
                           ▼                │ 达阈值
                    ┌─────────────┐         │
                    │  Unhealthy  │─────────┘
                    └──────┬──────┘
                           │ cooldown 到期
                           │ 或 health check 成功
                           ▼
                    ┌─────────────┐
                    │  Recovering │
                    └─────────────┘
```

**关键规则**：
- 从 Unknown → Healthy：首次探测成功即可（避免启动延迟）
- Healthy → Unhealthy：连续 `FailureThreshold` 次探测失败
- Unhealthy → Healthy：连续 `SuccessThreshold` 次探测成功，或 `CooldownDuration` 到期（若启用）
- 状态翻转时才触发 SharedData 写操作

**Lease-based Leader Election**：

参考 `ai-proxy` 的 `tryAcquireOrRenewLease` 实现，避免多 VM 同时向同一 endpoint 发送 health check 造成压力。

```go
func (p *Plugin) tryAcquireOrRenewLease(vmID string) bool {
    now := time.Now().Unix()
    data, cas, err := proxywasm.GetSharedData(p.leaseKey)
    if err != nil {
        if errors.Is(err, types.ErrorStatusNotFound) {
            return p.setLease(vmID, now, 0)
        }
        return false
    }
    var lease Lease
    json.Unmarshal(data, &lease)
    // 同 VM 续租，或租约过期（>60s）后抢占
    if lease.VMID == vmID || now - lease.Timestamp > 60 {
        return p.setLease(vmID, now, cas)
    }
    return false
}
```

只有获得 lease 的 VM 才执行 health check tick。租约 TTL 60 秒，过期后其他 VM 可以抢占。

### 1.3.1 健康检查启动代码

**Tick 注册**：在 `parseConfig` 完成后注册，每个 Group 注册一个 tick 函数

```go
func parseConfig(json gjson.Result, pluginCtx *wrapper.CommonPluginCtx[PluginConfig]) (PluginConfig, error) {
    config, err := parseConfigInternal(json)
    if err != nil {
        return config, err
    }
    
    // 为每个 Group 注册 health check tick
    for i := range config.Groups {
        group := &config.Groups[i]
        wrapper.RegisterTickFunc(group.HealthCheck.Interval, func() {
            p.runHealthCheck(group)
        })
    }
    
    return config, nil
}
```

**主 health check 函数**：

```go
func (p *Plugin) runHealthCheck(group *GroupConfig) {
    // 1. 尝试获取/续租 leader 身份
    vmID := p.vmID // 从 CommonVmCtx 获取
    if !p.tryAcquireOrRenewLease(vmID) {
        return // 未获得 lease，本次 tick 不执行探测
    }
    
    now := time.Now().UnixMilli()
    
    // 2. 遍历该 Group 的所有 endpoint，逐个发送探测
    for _, ep := range group.Endpoints {
        p.probeEndpoint(group, ep, now)
    }
    
    // 3. cooldown 恢复（如果启用）
    if group.HealthCheck.CooldownDuration > 0 {
        p.checkCooldownRecovery(group, now)
    }
}
```

**单个 endpoint 探测**：

```go
func (p *Plugin) probeEndpoint(group *GroupConfig, ep EndpointConfig, now int64) {
    // 解析 address 为 host 和 port
    host, port := parseAddress(ep.Address)
    
    client := wrapper.NewClusterClient(wrapper.FQDNCluster{
        Host: host,
        Port: port,
    })
    
    // 发送 GET /readiness 探测
    client.Get("/readiness", nil, func(statusCode int, respHeaders http.Header, respBody []byte) {
        // 解析响应
        isHealthy := false
        if statusCode == 200 {
            var resp ReadinessResponse
            if err := json.Unmarshal(respBody, &resp); err == nil {
                isHealthy = resp.Status == "ready"
            }
        }
        
        // 更新本地状态并处理翻转
        p.updateEndpointHealth(group.Name, ep.Name, isHealthy, now)
        
    }, uint32(group.HealthCheck.Timeout))
}
```

**状态更新与翻转检测**：

```go
func (p *Plugin) updateEndpointHealth(groupName, epName string, isHealthy bool, now int64) {
    oldStatus := p.localCache.Get(epName)
    
    // 初始化 unknown 状态
    if oldStatus == nil {
        oldStatus = &HealthStatus{State: "unknown"}
    }
    
    newStatus := &HealthStatus{
        State:      oldStatus.State,
        LastCheck:  now,
        LastChange: oldStatus.LastChange,
        FailCount:  oldStatus.FailCount,
        SuccCount:  oldStatus.SuccCount,
    }
    
    // 更新连续计数器
    if isHealthy {
        newStatus.SuccCount++
        newStatus.FailCount = 0
    } else {
        newStatus.FailCount++
        newStatus.SuccCount = 0
    }
    
    // 状态机转换
    switch oldStatus.State {
    case "healthy", "unknown":
        if newStatus.FailCount >= p.getFailureThreshold(groupName) {
            newStatus.State = "unhealthy"
            newStatus.LastChange = now
            newStatus.FailCount = 0
            p.writeSharedDataHealth(groupName, epName, newStatus)
        }
    case "unhealthy":
        if newStatus.SuccCount >= p.getSuccessThreshold(groupName) {
            newStatus.State = "healthy"
            newStatus.LastChange = now
            newStatus.SuccCount = 0
            p.writeSharedDataHealth(groupName, epName, newStatus)
        }
    }
    
    // 无论是否翻转，都更新本地缓存
    p.localCache.Set(epName, newStatus)
}

func (p *Plugin) writeSharedDataHealth(groupName, epName string, status *HealthStatus) {
    key := fmt.Sprintf("smg-failover:%s:%s:health", groupName, epName)
    data, _ := json.Marshal(status)
    // 健康状态幂等，使用 cas=0 强制写
    proxywasm.SetSharedData(key, data, 0)
}
```

**Cooldown 恢复**：

```go
func (p *Plugin) checkCooldownRecovery(group *GroupConfig, now int64) {
    cooldown := group.HealthCheck.CooldownDuration
    for _, ep := range group.Endpoints {
        status := p.localCache.Get(ep.Name)
        if status == nil || status.State != "unhealthy" {
            continue
        }
        // 如果 cooldown 已过期，直接标记为 healthy
        if now-status.LastChange >= cooldown {
            newStatus := &HealthStatus{
                State:      "healthy",
                LastCheck:  now,
                LastChange: now,
            }
            p.localCache.Set(ep.Name, newStatus)
            p.writeSharedDataHealth(group.Name, ep.Name, newStatus)
        }
    }
}
```

**探测方式说明**：
- 同一 Group 内的多个 endpoint 采用**串行探测**，避免并发请求对 SMG 造成突发压力
- 不同 Group 的 tick 是独立的，可能并发执行，但不同 Group 的 endpoint 通常不重叠，影响有限
- 如果需要并行探测同一 Group 的多个 endpoint，可将 `probeEndpoint` 改为 goroutine 并发，但需考虑 SMG 的探测负载承受能力

### 1.4 健康状态在 SharedData 中的存储格式

```go
// SharedData key: "smg-failover:{group}:{endpoint}:health"
// Value JSON:
type HealthStatus struct {
    State       string `json:"state"`       // "healthy" | "unhealthy" | "unknown"
    LastCheck   int64  `json:"last_check"`  // 上次探测时间戳（Unix ms）
    LastChange  int64  `json:"last_change"` // 上次状态翻转时间戳（Unix ms）
    FailCount   int64  `json:"fail_count"`  // 当前连续失败次数
    SuccCount   int64  `json:"succ_count"`  // 当前连续成功次数
}
```

**序列化**：使用 `json.Marshal` / `json.Unmarshal`，保证可读性和扩展性。

---

## 2. 请求控制与节点管理

### 2.1 请求拦截：OnHttpRequestHeaders

**插件注册**：

```go
func init() {
    wrapper.SetCtx(
        "smg-failover",
        wrapper.ParseConfigWithContext[PluginConfig](parseConfig),
        wrapper.ProcessRequestHeaders[PluginConfig](onHttpRequestHeaders),
        wrapper.ProcessResponseHeaders[PluginConfig](onHttpResponseHeaders),
        wrapper.ProcessRequestBody[PluginConfig](onHttpRequestBody),
        wrapper.WithRebuildAfterRequests[PluginConfig](1000),
    )
}
```

**Context Key 常量定义**（用于在请求生命周期内传递状态）：

```go
const (
    ctxLockedEndpoint = "smg-failover:locked-endpoint"   // 当前锁定的 endpoint name
    ctxLockedAddress  = "smg-failover:locked-address"    // 当前锁定的 endpoint address
    ctxClusterName    = "smg-failover:cluster-name"      // 当前请求的 Envoy cluster 名称
    ctxRequestHost    = "smg-failover:request-host"      // 原始请求的 host（用于重试重构）
    ctxRequestPath    = "smg-failover:request-path"      // 原始请求的 path（用于重试重构）
    ctxRequestMethod  = "smg-failover:request-method"    // 原始请求的 method（用于重试重构）
    ctxRequestBody    = "smg-failover:request-body"      // 原始请求的 body（用于重试重构）
    ctxIsStreaming    = "smg-failover:is-streaming"      // 是否为 SSE 流式请求
    ctxRetryCount     = "smg-failover:retry-count"       // 当前请求的重试次数
)
```

> 这些 key 存储在 `wrapper.HttpContext` 的 per-request context 中（通过 `ctx.SetContext` / `ctx.GetContext`），每个请求独立，不会跨请求共享。如果需要跨请求共享状态，使用 `SharedData`。

**路由决策流程**：

```go
func onHttpRequestHeaders(ctx wrapper.HttpContext, config PluginConfig) types.Action {
    ctx.DisableReroute() // 防止 header 修改后 Envoy 重新计算路由
    
    // 1. 获取当前请求的 cluster 名称
    clusterNameBytes, _ := proxywasm.GetProperty([]string{"cluster_name"})
    clusterName := string(clusterNameBytes)
    
    // 2. 查找该 cluster 对应的 Group 配置
    group := config.GetGroupByCluster(clusterName)
    if group == nil {
        return types.ActionContinue // 未配置的 Group，走默认路由
    }
    
    // 3. 读取当前锁定的 endpoint（来自上一个请求）
    currentLocked := ctx.GetStringContext(ctxLockedEndpoint, "")
    
    // 4. 从本地缓存读取各 endpoint 健康状态（优先，零 host call），传入当前锁定节点
    selectedEndpoint := selectHealthyEndpoint(group, p.localCache, currentLocked)
    
    if selectedEndpoint != nil {
        // 5. 锁定到选中的 endpoint
        proxywasm.SetUpstreamOverrideHost([]byte(selectedEndpoint.Address))
        
        // 6. 记录当前锁定的 endpoint，用于后续请求保持锁定
        ctx.SetContext(ctxLockedEndpoint, selectedEndpoint.Name)
        ctx.SetContext(ctxLockedAddress, selectedEndpoint.Address)
    }
    
    return types.ActionContinue
}
```

### 2.2 节点选择逻辑：selectHealthyEndpoint

```go
func selectHealthyEndpoint(
    group *GroupConfig,
    cache *LocalHealthCache,
    currentLocked string,  // 当前锁定的 endpoint name，可能为空
) *EndpointConfig {
    // 1. 如果有当前锁定节点，优先检查其是否仍 healthy
    if currentLocked != "" {
        currentEp := group.GetEndpointByName(currentLocked)
        if currentEp != nil {
            status := cache.Get(currentLocked)
            if status == nil || status.IsExpired(cacheTTL) {
                status = fetchFromSharedData(group.Name, currentLocked)
                cache.Set(currentLocked, status)
            }
            if status.State == "healthy" {
                // 当前锁定节点仍 healthy
                if group.FailbackStrategy != "auto" {
                    // sticky 或 manual：保持当前锁定，不主动切回更高优先级节点
                    return currentEp
                }
                // auto：继续检查是否有更高优先级的 healthy 节点，有则切回
            }
        }
    }
    
    // 2. 按优先级顺序遍历选择第一个 healthy endpoint
    for _, ep := range group.Endpoints {
        // 如果是 auto 策略且当前锁定节点仍 healthy，跳过比当前锁定优先级低的节点
        if currentLocked != "" && group.FailbackStrategy == "auto" {
            currentEp := group.GetEndpointByName(currentLocked)
            if currentEp != nil && ep.Priority > currentEp.Priority && currentEp.Name != currentLocked {
                // 当前锁定节点 healthy 且优先级更高，跳过低优先级节点
                continue
            }
        }
        
        status := cache.Get(ep.Name)
        if status == nil || status.IsExpired(cacheTTL) {
            status = fetchFromSharedData(group.Name, ep.Name)
            cache.Set(ep.Name, status)
        }
        if status.State == "healthy" {
            return ep
        }
    }
    // 无 healthy endpoint，仍返回优先级最高的 endpoint（硬着头皮锁定）
    // 理由：
    // 1. 保持 cache tree 集中，恢复后无需重新预热
    // 2. 请求触发即时 health check，可能比 tick 更快发现恢复
    // 3. SMG 503 不转发到 worker，不会造成底层压力
    // 4. Envoy 轮询多个故障节点没有任何收益
    if len(group.Endpoints) > 0 {
        return group.Endpoints[0]
    }
    return nil
}
```

**本地缓存设计**：

```go
type LocalHealthCache struct {
    mu     sync.RWMutex
    states map[string]*CachedHealthStatus
}

type CachedHealthStatus struct {
    Status    *HealthStatus
    FetchedAt time.Time
}

func (c *CachedHealthCache) Get(name string) *HealthStatus {
    c.mu.RLock()
    defer c.mu.RUnlock()
    cached := c.states[name]
    if cached == nil || time.Since(cached.FetchedAt) > cacheTTL {
        return nil // 触发回源
    }
    return cached.Status
}
```

`cacheTTL` 建议 1-2 秒。这意味着正常请求路径上**零 SharedData host call**。

### 2.3 节点剔除

当 health check 连续失败达到 `FailureThreshold` 时：

```go
func (p *Plugin) markUnhealthy(groupName, endpointName string) {
    // 1. 更新本地缓存
    p.localCache.Set(endpointName, &HealthStatus{State: "unhealthy"})
    
    // 2. 状态翻转，写 SharedData（供其他 VM 感知）
    key := fmt.Sprintf("smg-failover:%s:%s:health", groupName, endpointName)
    data, _ := json.Marshal(&HealthStatus{
        State:      "unhealthy",
        LastChange: time.Now().UnixMilli(),
    })
    proxywasm.SetSharedData(key, data, 0) // cas=0 强制写，健康状态不要求强一致
}
```

**注意**：这里使用 `cas=0` 强制写。健康状态的最终一致性是可以接受的——即使两个 VM 同时写，结果都是 "unhealthy"，不存在冲突。

### 2.4 节点恢复与回切控制

**恢复检测**：health check 连续成功 `SuccessThreshold` 次后，状态从 `unhealthy` → `healthy`。

**回切策略**（`failbackStrategy`）：

| 策略 | 行为 |
|------|------|
| `auto` | 节点恢复后自动重新纳入可选池。下一个请求到达时，按优先级可能重新锁定到恢复的节点 |
| `manual` | 节点恢复后标记为 `recovering`，需人工确认（或额外的确认机制）后才纳入可选池 |
| `sticky` | 节点恢复后纳入可选池，但**不主动切回**。只有当前锁定节点故障时才可能选到恢复的节点 |

推荐默认策略为 `sticky`：避免频繁切换导致 cache tree 颠簸。切换是有成本的——新锁定的节点需要重新预热 prefix cache。

---

## 3. 失败重试策略

### 3.1 重试触发条件

在 `OnHttpResponseHeaders` 中检查响应状态码：

```go
func onHttpResponseHeaders(ctx wrapper.HttpContext, config PluginConfig) types.Action {
    statusCode, _ := strconv.Atoi(proxywasm.GetHttpResponseHeader(":status"))
    
    if statusCode >= 500 {
        lockedEndpoint := ctx.GetStringContext(ctxLockedEndpoint, "")
        lockedAddress := ctx.GetStringContext(ctxLockedAddress, "")
        retryCount := ctx.GetIntContext(ctxRetryCount, 0)
        
        group := config.GetGroupByCluster(ctx.GetStringContext(ctxClusterName, ""))
        
        // 第1次重试：仍在原 endpoint（排除瞬时抖动）
        if retryCount == 0 && group != nil {
            ctx.SetContext(ctxRetryCount, 1)
            sendRetryToEndpoint(ctx, lockedAddress, config, func(retryStatus int) {
                if retryStatus == 200 {
                    // 原 endpoint 重试成功，标记为瞬时故障
                    return
                }
                // 原 endpoint 仍失败，尝试转移到下一个 endpoint
                tryNextEndpoint(ctx, group, lockedEndpoint, config)
            })
            return types.ActionPause
        }
        
        // 第2次及以后：直接转移到下一个 endpoint
        if retryCount >= 1 && group != nil {
            tryNextEndpoint(ctx, group, lockedEndpoint, config)
            return types.ActionPause
        }
    }
    
    return types.ActionContinue
}
```

### 3.2 请求状态保存

在 `OnHttpRequestHeaders` 和 `OnHttpRequestBody` 中保存原始请求信息：

```go
func onHttpRequestHeaders(ctx wrapper.HttpContext, config PluginConfig) types.Action {
    ctx.SetContext(ctxRequestHost, ctx.Host())
    ctx.SetContext(ctxRequestPath, ctx.Path())
    ctx.SetContext(ctxRequestMethod, ctx.Method())
    // headers 已在 Envoy 中，重试时重新读取
    return types.ActionContinue
}

func onHttpRequestBody(ctx wrapper.HttpContext, config PluginConfig, body []byte) types.Action {
    ctx.SetContext(ctxRequestBody, body)
    return types.ActionContinue
}
```

### 3.3 重试执行流程

**重试策略**：先在同 endpoint 重试一次（排除瞬时网络抖动），若仍失败再转移到下一个 endpoint。

```
请求发送到 endpoint-A → 返回 5xx
  │
  ├──→ 第1次重试：仍在 endpoint-A（可能是瞬时故障）
  │      │
  │      ├──→ 成功 → 返回响应给客户端
  │      │
  │      └──→ 仍失败
  │             │
  │             ▼
  │         第2次重试：转移到 endpoint-B
  │             │
  │             ├──→ 成功 → 返回响应给客户端
  │             │
  │             └──→ 仍失败 → 返回原始错误
  │
```

**代码实现**：

```go
func onHttpResponseHeaders(ctx wrapper.HttpContext, config PluginConfig) types.Action {
    statusCode, _ := strconv.Atoi(proxywasm.GetHttpResponseHeader(":status"))
    
    if statusCode >= 500 {
        lockedEndpoint := ctx.GetStringContext(ctxLockedEndpoint, "")
        lockedAddress := ctx.GetStringContext(ctxLockedAddress, "")
        retryCount := ctx.GetIntContext(ctxRetryCount, 0)
        
        group := config.GetGroupByCluster(ctx.GetStringContext(ctxClusterName, ""))
        
        // 第1次重试：仍在原 endpoint（排除瞬时抖动）
        if retryCount == 0 && group != nil {
            ctx.SetContext(ctxRetryCount, 1)
            sendRetryToEndpoint(ctx, lockedAddress, config, func(retryStatus int) {
                if retryStatus == 200 {
                    // 原 endpoint 重试成功，无需切换
                    return
                }
                // 原 endpoint 仍失败，尝试转移到下一个 endpoint
                tryNextEndpoint(ctx, group, lockedEndpoint, config)
            })
            return types.ActionPause
        }
        
        // 第2次及以后：直接转移到下一个 endpoint
        if retryCount >= 1 && group != nil {
            tryNextEndpoint(ctx, group, lockedEndpoint, config)
            return types.ActionPause
        }
    }
    
    return types.ActionContinue
}

func sendRetryToEndpoint(
    ctx wrapper.HttpContext,
    address string,
    config PluginConfig,
    onComplete func(statusCode int),
) {
    host, port := parseAddress(address)
    
    client := wrapper.NewClusterClient(wrapper.FQDNCluster{
        Host: host,
        Port: port,
    })
    
    body := ctx.GetByteSliceContext(ctxRequestBody, []byte{})
    method := ctx.GetStringContext(ctxRequestMethod, "POST")
    path := ctx.GetStringContext(ctxRequestPath, "")
    
    client.Call(method, path, nil, body, func(statusCode int, respHeaders http.Header, respBody []byte) {
        if statusCode == 200 {
            proxywasm.SendHttpResponse(statusCode,
                headersToPairs(respHeaders), respBody, -1)
        }
        onComplete(statusCode)
    }, uint32(config.RetryTimeoutMs))
}

func tryNextEndpoint(
    ctx wrapper.HttpContext,
    group *GroupConfig,
    currentLocked string,
    config PluginConfig,
) {
    nextEndpoint := selectNextHealthyEndpoint(group, currentLocked, p.localCache)
    if nextEndpoint == nil {
        proxywasm.ResumeHttpResponse()
        return
    }
    
    sendRetryToEndpoint(ctx, nextEndpoint.Address, config, func(statusCode int) {
        if statusCode != 200 {
            proxywasm.ResumeHttpResponse()
        }
    })
}
```

### 3.4 SSE 流式请求的特殊处理

**限制**：与 `ai-proxy` 相同，流式响应一旦开始发送，无法"替换"为另一个响应。

**处理策略**：

| 场景 | 行为 |
|------|------|
| 首 token 前返回 5xx | 可以完整重试到备 endpoint。`SendHttpResponse` 返回备的响应 |
| 已经开始发送 token 后断开 | 无法重试。客户端会收到截断的流。需在客户端实现重连逻辑 |
| 连接建立后无响应（超时） | Envoy 层面连接超时，插件在 `OnHttpStreamDone` 中检测到异常，标记 endpoint 不健康 |

**SSE 请求的额外上下文保存**：

```go
func onHttpRequestHeaders(ctx wrapper.HttpContext, config PluginConfig) types.Action {
    // 检测 SSE 请求
    if strings.Contains(ctx.GetHttpRequestHeader("accept"), "text/event-stream") ||
       ctx.GetHttpRequestHeader("x-stream") == "true" {
        ctx.SetContext(ctxIsStreaming, true)
    }
    return types.ActionContinue
}
```

对于 SSE 请求，重试策略更保守：仅在首响应前（`OnHttpResponseHeaders` 阶段）尝试重试，一旦进入 `OnHttpStreamingResponseBody` 阶段，不再触发重试。

---

## 4. 后续请求正确路由

### 4.1 锁定持续性

`SetUpstreamOverrideHost` 的锁定是**请求级**的，不是持久的。每个新请求到达时，都需要重新执行 `onHttpRequestHeaders` 中的选择逻辑。

但配合本地缓存后，这个选择过程的开销极低（零 host call，纯内存查找）。因此锁定效果是通过**每个请求都重新评估并选择同一 endpoint** 来实现的，而不是通过一个持久的状态。

### 4.2 失效切换的完整链路

```
请求 N（锁定到 SMG-1.1）
  │
  ▼
OnHttpRequestHeaders → selectHealthyEndpoint → SMG-1.1 healthy → SetUpstreamOverrideHost(SMG-1.1)
  │
  ▼
请求发送到 SMG-1.1
  │
  ▼
SMG-1.1 返回 503（worker 故障）
  │
  ▼
OnHttpResponseHeaders → 检测到 5xx
  ├──→ 异步触发即时 health check（确认 SMG-1.1 不健康）
  ├──→ 尝试重试到 SMG-1.2（若重试成功，返回备的响应）
  └──→ 更新 SharedData：SMG-1.1 = unhealthy
  │
  ▼
请求 N+1 到达
  │
  ▼
OnHttpRequestHeaders → selectHealthyEndpoint
  ├──→ 本地缓存：SMG-1.1 = unhealthy（或缓存过期回源 SharedData 确认）
  ├──→ SMG-1.2 healthy → SetUpstreamOverrideHost(SMG-1.2)
  └──→ 锁定到 SMG-1.2
```

**关键延迟**：
- 如果请求 N 的失败触发了即时 health check，SharedData 在几毫秒内更新
- 请求 N+1 的 `OnHttpRequestHeaders` 中，本地缓存可能仍在 TTL 内（1-2 秒），显示的仍是旧状态。但缓存过期后会回源 SharedData，读到最新状态
- 如果需要更快的收敛，可以在 `OnHttpResponseHeaders` 的失败处理中**主动刷新本地缓存**：`p.localCache.Invalidate(lockedEndpoint)`

**主动刷新优化**：

```go
func onHttpResponseHeaders(ctx wrapper.HttpContext, config PluginConfig) types.Action {
    statusCode, _ := strconv.Atoi(proxywasm.GetHttpResponseHeader(":status"))
    if statusCode >= 500 {
        lockedEndpoint := ctx.GetStringContext(ctxLockedEndpoint, "")
        // 立即刷新本地缓存，强制下一个请求回源 SharedData
        p.localCache.Invalidate(lockedEndpoint)
        // ... 重试逻辑
    }
    return types.ActionContinue
}
```

这样请求 N+1 到达时，缓存已失效，会立即回源 SharedData，感知到最新的 unhealthy 状态。


---

## 5. SMG 接口对接

### 5.1 /readiness 探测协议

基于 `sgl-model-gateway/src/server.rs:102-144` 的 `readiness()` 实现，探测请求格式：

```
GET /readiness HTTP/1.1
Host: <endpoint-address>:30002
Accept: application/json
```

**响应解析**：

```go
type ReadinessResponse struct {
    Status         string `json:"status"`          // "ready" | "not ready"
    HealthyWorkers int    `json:"healthy_workers"` // healthy worker 数量
    TotalWorkers   int    `json:"total_workers"`   // 总 worker 数量
}

func parseReadinessResponse(body []byte) (bool, error) {
    var resp ReadinessResponse
    if err := json.Unmarshal(body, &resp); err != nil {
        return false, err
    }
    return resp.Status == "ready", nil
}
```

**探测超时**：默认 5 秒。如果超时未响应，视为探测失败。

**PD 模式的特殊语义**：

SMG 的 `readiness()` 在 `RoutingMode::PrefillDecode` 下要求 `has_prefill && has_decode`。这意味着：
- 如果 prefill worker 全部故障，SMG 返回 503
- 如果 decode worker 全部故障，SMG 返回 503
- 只有当 P/D 配对都 healthy 时，SMG 才认为自身 ready

这对 failover 插件是**有利的**——不需要插件自己判断 P/D worker 的分布，只需信任 SMG 的 readiness 语义。

### 5.2 请求转发

SMG 接收的请求是标准的 OpenAI 兼容格式（`sgl-model-gateway/src/server.rs:536-693`）。插件**不修改请求 body**，只通过 `SetUpstreamOverrideHost` 控制请求发送到哪个 endpoint。

**请求路径**：

```
Client → Higress Gateway → WASM Plugin → SetUpstreamOverrideHost → SMG endpoint
                                        ↓
                                   不修改 :authority、:path、body
```

**为什么不需要修改 `:authority`**：

`SetUpstreamOverrideHost` 直接覆盖 Envoy 选中的 upstream endpoint，不需要通过修改 `:authority` 来影响路由。这与 `ai-proxy` 的路由模式不同——`ai-proxy` 需要修改 `:authority` 来路由到不同的云厂商 API，而本插件只需要在同一个 cluster 内切换 endpoint。

**请求 body 的透传**：

```go
func onHttpRequestHeaders(ctx wrapper.HttpContext, config PluginConfig) types.Action {
    ctx.DisableReroute()
    // ... endpoint 选择 ...
    return types.ActionContinue
}

func onHttpRequestBody(ctx wrapper.HttpContext, config PluginConfig, body []byte) types.Action {
    // body 完全透传，不做任何修改
    return types.ActionContinue
}
```

### 5.3 响应处理

**非流式响应**：直接透传。SMG 返回的响应 body 是完整的 JSON，WASM 插件不需要修改。

**流式响应（SSE）**：`sgl-model-gateway/src/routers/openai/responses/streaming.rs` 将 worker 的响应转换为标准 SSE 格式：

```
event: message
data: {"id":"...","object":"chat.completion.chunk","choices":[...]}

event: message
data: {"id":"...","object":"chat.completion.chunk","choices":[...]}

event: message
data: [DONE]

```

WASM 插件在流式场景下只拦截响应头（`OnHttpResponseHeaders`），不拦截响应 body（`OnHttpStreamingResponseBody` 中直接透传）。这避免了插件处理 SSE chunk 的复杂性。

**响应状态码的处理**：

| 状态码 | 行为 |
|--------|------|
| 200 | 正常，重置该 endpoint 的失败计数器 |
| 4xx | 客户端错误，不触发重试（不是 endpoint 故障） |
| 5xx | 服务端错误，触发重试 + 即时 health check |
| 超时（无响应）| Envoy 连接超时，插件在 `OnHttpStreamDone` 中处理 |

**`OnHttpStreamDone` 异常检测**：

```go
func onHttpStreamDone(ctx wrapper.HttpContext, config PluginConfig) {
    lockedEndpoint := ctx.GetStringContext(ctxLockedEndpoint, "")
    if lockedEndpoint == "" {
        return
    }
    
    // 获取上游响应码
    statusBytes, _ := proxywasm.GetProperty([]string{"response_code"})
    if len(statusBytes) == 0 {
        // 无响应码 → 连接异常（超时或断开）
        p.markUnhealthy(config.GetGroupName(), lockedEndpoint)
        p.localCache.Invalidate(lockedEndpoint)
    }
}
```

---

## 6. SharedData 读写策略

### 6.1 分层缓存架构

```
┌─────────────────────────────────────────────────────────────┐
│                      请求处理路径                            │
│                                                             │
│  OnHttpRequestHeaders → 读本地缓存 → 命中？                  │
│                              │                              │
│                              ├─ 是 → 零 host call，直接决策  │
│                              │                              │
│                              └─ 否 → GetSharedData 回源      │
│                                         → 更新本地缓存       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    健康检查 Tick 路径                        │
│                                                             │
│  RegisterTickFunc → health check → 状态翻转？               │
│                              │                              │
│                              ├─ 否 → 不写 SharedData         │
│                              │                              │
│                              └─ 是 → SetSharedData           │
│                                         → 更新本地缓存       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 读路径优化

**本地缓存命中时**：

```go
func (c *LocalHealthCache) Get(name string) *HealthStatus {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    cached := c.states[name]
    if cached == nil {
        return nil // miss，触发回源
    }
    if time.Since(cached.FetchedAt) > cacheTTL {
        return nil // 过期，触发回源
    }
    return cached.Status // 命中，零 host call
}
```

**回源 SharedData**（仅在缓存 miss 或过期时）：

```go
func fetchFromSharedData(groupName, endpointName string) *HealthStatus {
    key := fmt.Sprintf("smg-failover:%s:%s:health", groupName, endpointName)
    data, _, err := proxywasm.GetSharedData(key)
    if err != nil {
        return &HealthStatus{State: "unknown"}
    }
    var status HealthStatus
    json.Unmarshal(data, &status)
    return &status
}
```

**性能数据**：
- 本地缓存命中：~50-100ns（纯 Go map 查找）
- GetSharedData host call：~1-5μs（WASM/Envoy 边界穿越）
- 对于 3 个 endpoint 的 Group，缓存命中时读路径开销从 3-15μs 降为 ~150-300ns

### 6.3 写路径优化

**状态翻转检测**：

```go
func (p *Plugin) onHealthCheckResult(groupName, endpointName string, isHealthy bool) {
    oldStatus := p.localCache.Get(endpointName)
    oldHealthy := oldStatus != nil && oldStatus.State == "healthy"
    
    if oldHealthy == isHealthy {
        // 状态未翻转，仅更新本地缓存的时间戳和计数器
        oldStatus.LastCheck = time.Now().UnixMilli()
        if !isHealthy {
            oldStatus.FailCount++
        } else {
            oldStatus.SuccCount++
        }
        return // 不写 SharedData
    }
    
    // 状态翻转，写 SharedData
    newStatus := &HealthStatus{
        State:      map[bool]string{true: "healthy", false: "unhealthy"}[isHealthy],
        LastCheck:  time.Now().UnixMilli(),
        LastChange: time.Now().UnixMilli(),
    }
    
    p.localCache.Set(endpointName, newStatus)
    
    key := fmt.Sprintf("smg-failover:%s:%s:health", groupName, endpointName)
    data, _ := json.Marshal(newStatus)
    proxywasm.SetSharedData(key, data, 0) // cas=0，健康状态不要求 CAS 强一致
}
```

**为什么健康状态使用 CAS=0**

健康状态写操作使用 `cas=0`（无条件覆盖）而非 CAS 重试，基于以下考量：

1. **写者单一**：周期性 tick 中，只有获得 lease 的 VM 执行 health check 并写 SharedData。正常情况下不存在并发写者。

2. **冲突场景有限**：唯一可能的并发写来自"即时 health check"（`OnHttpResponseHeaders` 中 5xx 触发）与周期性 tick 的重叠。这种重叠的概率极低（探测间隔 3s vs 请求处理毫秒级）。

3. **结果可收敛**：即使发生覆盖，下一次 tick（最多 3s 后）会重新探测并写入正确状态。健康状态是"最终一致"的，不需要强一致。

4. **CAS 重试的开销不值得**：健康状态写频率极低（仅在状态翻转时），为这种低概率冲突引入 CAS 重试循环的收益有限。

**但 CAS=0 的风险需要承认**：如果两个 VM 同时探测同一 endpoint 并得到相反结果（一个 healthy、一个 unhealthy），后写的会覆盖先写的，导致短暂的状态不确定。这种不一致窗口最长为一个 tick 周期（如 3s），在工程上可接受。

对于 lease 租约（leader election），必须使用 CAS：

```go
func (p *Plugin) setLease(vmID string, now int64, cas uint32) bool {
    lease := Lease{VMID: vmID, Timestamp: now}
    data, _ := json.Marshal(lease)
    err := proxywasm.SetSharedData(p.leaseKey, data, cas)
    return err == nil
}
```

租约冲突时返回 `types.ErrorStatusCasMismatch`，调用方重试即可。

### 6.4 CAS 冲突处理

```go
const casMaxRetries = 10

func (p *Plugin) updateSharedDataWithRetry(key string, data []byte) error {
    for i := 0; i < casMaxRetries; i++ {
        _, cas, err := proxywasm.GetSharedData(key)
        if err != nil && !errors.Is(err, types.ErrorStatusNotFound) {
            return err
        }
        err = proxywasm.SetSharedData(key, data, cas)
        if err == nil {
            return nil
        }
        if !errors.Is(err, types.ErrorStatusCasMismatch) {
            return err
        }
        // CAS 冲突，重试
    }
    return fmt.Errorf("CAS max retries exceeded")
}
```

参考 `ai-proxy` failover.go 中的 CAS 重试模式。

---

## 7. K8s 资源 Spec

### 7.1 ServiceEntry

每个 SMG Group 定义为一个 ServiceEntry，包含该 Group 的所有 endpoint。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: smg-group-1
  namespace: default
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
    - address: smg-1-2.default.svc.cluster.local
    - address: smg-1-3.default.svc.cluster.local
```

**说明**：
- `resolution: DNS` 表示 Envoy 通过 DNS 解析每个 endpoint 的地址
- 所有 endpoint 属于同一个 Envoy cluster（`smg-group-1.default.svc.cluster.local`）
- 插件通过 `SetUpstreamOverrideHost` 在 cluster 内选择具体 endpoint

### 7.2 VirtualService

外层路由，将流量按 weight 分发到各 Group。

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: smg-gateway-routing
  namespace: default
spec:
  hosts:
    - smg-gateway.example.com
  gateways:
    - higress-gateway
  http:
    - match:
        - uri:
            prefix: /v1/chat/completions
        - uri:
            prefix: /v1/completions
      route:
        - destination:
            host: smg-group-1.default.svc.cluster.local
          weight: 33
        - destination:
            host: smg-group-2.default.svc.cluster.local
          weight: 33
        - destination:
            host: smg-group-3.default.svc.cluster.local
          weight: 34
      timeout: 300s
```

**说明**：
- 外层按 33/33/34 的 weight 轮询分发到三个 Group
- 每个 Group 内部由插件做 endpoint 锁定
- timeout 设置为 300s，覆盖长推理请求

### 7.3 WasmPlugin

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
          cooldownDuration: 0
        failbackStrategy: sticky  # auto | manual | sticky
        
      - name: group-2
        cluster: smg-group-2.default.svc.cluster.local
        endpoints:
          - name: smg-2-1
            address: smg-2-1.default.svc.cluster.local:30002
            priority: 0
          - name: smg-2-2
            address: smg-2-2.default.svc.cluster.local:30002
            priority: 1
        healthCheck:
          interval: 3000
          timeout: 5000
          failureThreshold: 2
          successThreshold: 2
        failbackStrategy: sticky
        
    global:
      cacheTTL: 2000  # 本地缓存过期时间（ms）
      leaseTTL: 60    # leader election 租约 TTL（s）
      maxRetries: 1   # 被动重试最大次数
      retryTimeout: 30000  # 重试超时（ms）
      
  url: oci://higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/smg-failover:1.0.0
  imagePullPolicy: IfNotPresent
```

**配置字段说明**：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `groups` | array | required | Group 列表，每个 Group 对应一个 Envoy cluster |
| `groups[].name` | string | required | Group 标识 |
| `groups[].cluster` | string | required | Envoy cluster 名称（对应 ServiceEntry host） |
| `groups[].endpoints` | array | required | Endpoint 列表 |
| `groups[].endpoints[].name` | string | required | Endpoint 标识 |
| `groups[].endpoints[].address` | string | required | Endpoint 地址（IP:port 或 host:port） |
| `groups[].endpoints[].priority` | int | 0 | 锁定优先级，数字越小优先级越高 |
| `groups[].healthCheck.interval` | int64 | 3000 | 探测周期（ms） |
| `groups[].healthCheck.timeout` | int64 | 5000 | 探测超时（ms） |
| `groups[].healthCheck.failureThreshold` | int64 | 2 | 连续失败阈值 |
| `groups[].healthCheck.successThreshold` | int64 | 2 | 连续成功阈值 |
| `groups[].healthCheck.cooldownDuration` | int64 | 0 | 冷却恢复时间（ms），0 表示不启用 |
| `groups[].failbackStrategy` | string | "sticky" | 恢复后策略：auto / manual / sticky |
| `global.cacheTTL` | int | 2000 | 本地缓存过期时间（ms） |
| `global.leaseTTL` | int | 60 | leader election 租约 TTL（s） |
| `global.maxRetries` | int | 1 | 被动重试最大次数 |
| `global.retryTimeout` | int | 30000 | 重试超时（ms） |

---

## 8. 部署细节

### 8.1 镜像构建

基于 `higress/plugins/wasm-go` 的构建流程：

```dockerfile
# Dockerfile
FROM higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/wasm-go-builder:latest as builder

WORKDIR /workspace
COPY . .
RUN go mod download
RUN make build

FROM scratch
COPY --from=builder /workspace/main.wasm /
```

```makefile
# Makefile
BUILDER_IMAGE ?= higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/wasm-go-builder:latest

build:
	docker run --rm -v $(PWD):/workspace $(BUILDER_IMAGE) \
		go build -o main.wasm -buildmode=c-shared -target=wasm32-wasi

push:
	docker build -t $(REGISTRY)/smg-failover:$(VERSION) .
	docker push $(REGISTRY)/smg-failover:$(VERSION)
```

### 8.2 配置热更新

Higress `WasmPlugin` CRD 支持热更新：修改 CRD 的 `defaultConfig` 后，Higress 控制器会自动将新配置推送到 Envoy，无需重启网关。

**更新流程**：

```bash
# 修改配置
kubectl edit wasmplugin smg-failover -n higress-system

# 或应用新配置
kubectl apply -f smg-failover-config.yaml

# 验证更新
kubectl get wasmplugin smg-failover -n higress-system -o yaml
```

**配置变更的影响**：
- `endpoints` 列表变更：立即生效，下一个请求使用新列表
- `healthCheck` 参数变更：下一个 tick 周期生效
- `failbackStrategy` 变更：立即生效

### 8.3 监控与告警

**Metrics**：利用 wasm-go 框架的 `DefineCounterMetric` / `DefineGaugeMetric` 暴露以下指标：

```go
var (
    failoverCounter = proxywasm.DefineCounterMetric("smg_failover_switch_total")
    retryCounter    = proxywasm.DefineCounterMetric("smg_retry_total")
    healthGauge     = proxywasm.DefineGaugeMetric("smg_endpoint_healthy")
)
```

| 指标名 | 类型 | 标签 | 说明 |
|--------|------|------|------|
| `smg_failover_switch_total` | Counter | group, from_endpoint, to_endpoint | 锁定切换次数 |
| `smg_retry_total` | Counter | group, endpoint, status_code | 重试次数 |
| `smg_health_check_total` | Counter | group, endpoint, result | 健康检查结果 |
| `smg_endpoint_healthy` | Gauge | group, endpoint | 1=healthy, 0=unhealthy |
| `smg_healthstate_cache_hit_ratio` | Gauge | group | 插件本地健康状态缓存命中率 |

**日志**：利用 `proxywasm.LogInfof` / `proxywasm.LogErrorf` 输出关键事件：

```
[info] smg-failover: endpoint smg-1-1 marked unhealthy (consecutive failures: 2)
[info] smg-failover: locked to endpoint smg-1-2 (group=group-1)
[info] smg-failover: retry success from smg-1-1 to smg-1-2 (status=200)
[error] smg-failover: health check failed for smg-1-1: connection timeout
```

**Prometheus 集成**：Higress 网关已内置 Prometheus 指标采集，wasm-go 定义的指标会自动暴露到 `/stats/prometheus` 端点。

### 8.4 多环境部署

**开发环境**：
- 使用 `imagePullPolicy: Always`，方便频繁迭代
- health check 间隔设为 1s，快速验证切换逻辑
- `failbackStrategy: auto`，观察自动回切行为

**生产环境**：
- 使用 `imagePullPolicy: IfNotPresent`，固定版本
- health check 间隔 3-5s，避免探测过于频繁
- `failbackStrategy: sticky`，减少 cache tree 颠簸
- 配置 `maxRetries: 1`，避免重试风暴
- 建议 health check 走独立端口或轻量级端点，避免与业务请求竞争资源

**灰度发布**：
- 通过 `matchRules` 按域名或路由灰度：

```yaml
spec:
  matchRules:
    - domain:
        - "smg-beta.example.com"
      config:
        groups:
          - name: group-1-beta
            # ... beta 配置
```

---

## 9. 关键接口签名速查

### 9.1 wasm-go 框架

```go
// 插件注册
wrapper.SetCtx(pluginName, ...ctxOptions)

// 生命周期钩子
wrapper.ProcessRequestHeaders[PluginConfig](onHttpRequestHeaders)
wrapper.ProcessResponseHeaders[PluginConfig](onHttpResponseHeaders)
wrapper.ProcessRequestBody[PluginConfig](onHttpRequestBody)

// Tick 注册
wrapper.RegisterTickFunc(tickPeriodMs int64, tickFunc func())

// HTTP 调用
wrapper.NewClusterClient(wrapper.FQDNCluster{Host: "...", Port: 30002})
client.Get(path, headers, callback, timeout)
client.Post(path, headers, body, callback, timeout)

// 上下文
ctx.SetContext(key string, value interface{})
ctx.GetContext(key string) interface{}
ctx.GetStringContext(key, defaultValue string) string
ctx.GetByteSliceContext(key string, defaultValue []byte) []byte
ctx.DisableReroute()

// RouteCall（完全替换请求）
ctx.RouteCall(method, url, headers, body, callback)
```

### 9.2 proxy-wasm SDK

```go
// SharedData
proxywasm.GetSharedData(key string) (value []byte, cas uint32, err error)
proxywasm.SetSharedData(key string, data []byte, cas uint32) error

// 属性
proxywasm.GetProperty(path []string) ([]byte, error)

// Upstream 控制
proxywasm.GetUpstreamHosts() ([][2]string, error)
proxywasm.SetUpstreamOverrideHost(address []byte) error

// 响应
proxywasm.SendHttpResponse(statusCode int, headers [][2]string, body []byte, grpcStatus int32) error
proxywasm.ResumeHttpResponse() error

// Tick
proxywasm.SetTickPeriodMilliSeconds(ms uint32) error
```

### 9.3 SMG 端点

```
GET /liveness    → 200 OK, body "OK"
GET /readiness   → 200 OK {"status":"ready",...} 或 503 {"status":"not ready",...}
GET /health      → 200 OK, body "OK"（liveness 的别名）

POST /v1/chat/completions  → OpenAI 兼容格式
POST /v1/completions       → OpenAI 兼容格式
```

---

## 10. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 所有 endpoint 同时故障 | Group 无可用节点 | 仍锁定到优先级最高的 endpoint（保持 cache tree 集中），请求失败触发即时 health check，可能比 tick 更快发现恢复 |
| SharedData 读写延迟 | 状态同步不及时 | 本地缓存 + 缓存失效主动刷新 + 短 TTL |
| 健康检查探测风暴 | endpoint 压力增大 | lease-based leader election，单 VM 执行探测 |
| 频繁切换导致 cache 颠簸 | 命中率下降 | `failbackStrategy: sticky`，减少回切 |
| SSE 流式中断无法重试 | 客户端收到截断响应 | 客户端实现重连 + token 续传逻辑 |
| WASM VM 重建 | 本地缓存丢失 | 缓存丢失后自动回源 SharedData，无状态设计 |
| 插件 panic | 请求处理中断 | wasm-go 框架捕获 panic，FAIL_OPEN 策略继续处理 |
