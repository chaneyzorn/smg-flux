# VirtualService — Istio 路由规则

## 资源说明

`VirtualService` 是 Istio 的路由规则资源，定义了流量进入网格后的匹配条件和转发目标。

这是**方案 B** 的配套资源，与 `Gateway` 配合使用，实现 Istio 原生的七层流量管理。

## 字段解析

| 字段 | 值 | 说明 |
|------|-----|------|
| `apiVersion` | `networking.istio.io/v1beta1` | Istio 网络 API 版本 |
| `kind` | `VirtualService` | 资源类型：虚拟服务 |
| `metadata.name` | `kimi-router-vs` | 虚拟服务名称 |
| `metadata.namespace` | `higress-system` | 部署在 higress-system 命名空间 |

### spec.hosts

- `api.kimi-router.example.com`：该 VirtualService 处理的主机名
- 必须与 `Gateway.servers[].hosts` 中定义的主机名一致

### spec.gateways

- `kimi-router-gateway`：引用 `Gateway` 资源名称
- 表示"当流量从 `kimi-router-gateway` 进入时，应用本路由规则"

### spec.http

定义 HTTP 路由规则列表：

#### match（匹配条件）

| 字段 | 值 | 说明 |
|------|-----|------|
| `uri.prefix` | `/` | URI 前缀匹配，匹配所有路径 |

#### route（路由目标）

| 字段 | 值 | 说明 |
|------|-----|------|
| `destination.host` | `kimi-router-ha-cluster.local` | 目标主机，即 ServiceEntry 中定义的虚拟域名 |
| `destination.port.number` | `30002` | 目标端口 |

## 扩展能力

VirtualService 支持丰富的流量治理能力，可根据需要追加：

### 超时控制

```yaml
http:
  - route:
      - destination:
          host: kimi-router-ha-cluster.local
    timeout: 30s
```

### 重试策略

```yaml
http:
  - route:
      - destination:
          host: kimi-router-ha-cluster.local
    retries:
      attempts: 3
      perTryTimeout: 10s
      retryOn: gateway-error,connect-failure,refused-stream
```

### 流量镜像

```yaml
http:
  - route:
      - destination:
          host: kimi-router-ha-cluster.local
    mirror:
      host: kimi-router-ha-cluster.local
    mirrorPercentage:
      value: 10.0
```

### 按路径分流

```yaml
http:
  - match:
      - uri:
          prefix: /v1/chat
    route:
      - destination:
          host: kimi-router-ha-cluster.local
  - match:
      - uri:
          prefix: /v1/embeddings
    route:
      - destination:
          host: kimi-embedding-router-ha-cluster.local
```

## 使用建议

- 配合 `Gateway` 使用时，确保 `spec.gateways` 正确引用 Gateway 名称
- 如果同时存在 `Ingress` 和 `VirtualService`，注意避免路由冲突
- 生产环境建议通过 `DestinationRule` 补充负载均衡和健康检查策略
