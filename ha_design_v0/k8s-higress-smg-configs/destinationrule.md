# DestinationRule — 负载均衡与健康检查

## 资源说明

`DestinationRule` 是 Istio 的流量策略资源，用于定义到达特定目标服务后的**负载均衡方式**、**连接池管理**和**异常检测（熔断/驱逐）**策略。

在本场景中，它精细控制 Higress/Istio 如何在 3 个 SMG 分片之间分发请求，并在某个分片故障时自动将其隔离。

## 字段解析

| 字段 | 值 | 说明 |
|------|-----|------|
| `apiVersion` | `networking.istio.io/v1beta1` | Istio 网络 API 版本 |
| `kind` | `DestinationRule` | 资源类型：目标规则 |
| `metadata.name` | `kimi-router-dr` | 目标规则名称 |
| `metadata.namespace` | `higress-system` | 部署在 higress-system 命名空间 |

### spec.host

- `kimi-router-ha-cluster.local`：目标服务域名
- 必须与 `ServiceEntry.spec.hosts` 和 `VirtualService.spec.http[].route[].destination.host` 一致

### spec.trafficPolicy

#### loadBalancer（负载均衡策略）

| 字段 | 值 | 说明 |
|------|-----|------|
| `simple` | `LEAST_CONN` | 最小连接数负载均衡 |

可选策略：

| 策略 | 说明 |
|------|------|
| `ROUND_ROBIN` | 轮询（默认） |
| `LEAST_CONN` | 最小连接数，适合长连接/流式请求 |
| `RANDOM` | 随机 |
| `PASSTHROUGH` | 透传，不做负载均衡 |
| `CONSISTENT_HASH` | 一致性哈希，适合需要会话保持的场景 |

对于 LLM 推理场景（尤其是流式输出），`LEAST_CONN` 通常比 `ROUND_ROBIN` 更合适，因为它会优先将请求分发到当前并发请求较少的 SMG 分片。

#### connectionPool（连接池管理）

##### tcp

| 字段 | 值 | 说明 |
|------|-----|------|
| `maxConnections` | `512` | TCP 最大连接数 |

##### http

| 字段 | 值 | 说明 |
|------|-----|------|
| `http1MaxPendingRequests` | `256` | HTTP/1.1 最大挂起请求数 |
| `maxRequestsPerConnection` | `100` | 每个连接的最大请求数，超过后关闭连接 |

连接池配置可防止某个 SMG 分片因连接过多而被压垮，同时通过限制单连接请求数避免连接长期不释放导致的资源泄漏。

#### outlierDetection（异常检测 / 熔断驱逐）

| 字段 | 值 | 说明 |
|------|-----|------|
| `consecutive5xxErrors` | `3` | 连续 3 次 5xx 错误后，将该端点标记为异常 |
| `interval` | `30s` | 检测周期为 30 秒 |
| `baseEjectionTime` | `30s` | 异常端点被驱逐（隔离）的基础时间为 30 秒 |

异常检测机制：
1. Istio 持续监控后端 3 个 SMG 分片的响应状态
2. 若某个分片在 30 秒内连续返回 3 次 5xx 错误，则将其标记为异常
3. 该异常分片被暂时从负载均衡池中移除（驱逐），流量不再分发到它
4. 经过 `baseEjectionTime` 后，尝试恢复；若仍异常，驱逐时间按指数退避增长

## 生产建议

### 负载均衡策略选择

| 场景 | 推荐策略 |
|------|---------|
| 短请求（非流式） | `ROUND_ROBIN` |
| 流式请求（SSE/gRPC Streaming） | `LEAST_CONN` |
| 需要前缀缓存亲和性 | `CONSISTENT_HASH`（基于 header 或 source IP） |

### 熔断参数调优

| 参数 | 当前值 | 调优建议 |
|------|--------|---------|
| `consecutive5xxErrors` | `3` | 如果 SMG 对偶发错误敏感，可适当增大到 5 |
| `interval` | `30s` | 高频场景可缩短到 10s，更快发现问题 |
| `baseEjectionTime` | `30s` | 如果 SMG 重启慢，可增大到 60s 避免频繁尝试恢复 |

### 与 Higress 的协同

如果通过 `Ingress` 方式（方案 A）接入，Higress 也支持部分流量治理功能。此时：

- Higress 层处理：限流、认证、WAF、重写等
- Istio DestinationRule 层处理：负载均衡、连接池、熔断、重试等

两者互补，形成完整的流量治理体系。
