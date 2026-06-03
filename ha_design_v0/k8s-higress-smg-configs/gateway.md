# Gateway — Istio 入口网关

## 资源说明

`Gateway` 是 Istio 的入口网关资源，定义了 Envoy 代理的端口、协议和监听的主机名。

这是**方案 B**：Istio 原生流量管理方式，直接操作 Envoy 配置。

## 字段解析

| 字段 | 值 | 说明 |
|------|-----|------|
| `apiVersion` | `networking.istio.io/v1beta1` | Istio 网络 API 版本 |
| `kind` | `Gateway` | 资源类型：网关 |
| `metadata.name` | `kimi-router-gateway` | 网关资源名称 |
| `metadata.namespace` | `higress-system` | 部署在 higress-system 命名空间 |

### spec.selector

| 字段 | 值 | 说明 |
|------|-----|------|
| `istio` | `ingressgateway` | 绑定到带有 `istio=ingressgateway` 标签的 Pod，即 Higress/Istio 网关实例 |

### spec.servers

定义网关监听的服务端口：

| 字段 | 值 | 说明 |
|------|-----|------|
| `port.number` | `80` | 监听端口 |
| `port.name` | `http` | 端口名称 |
| `port.protocol` | `HTTP` | 协议类型 |
| `hosts` | `api.kimi-router.example.com` | 允许访问的主机名，VirtualService 会引用这个 Gateway |

## 与 VirtualService 的关系

`Gateway` 只负责**监听入口流量**，具体的**路由规则**（匹配条件和目标地址）定义在 `VirtualService` 中。

两者通过 `VirtualService.gateways` 字段关联：
- `VirtualService` 引用 `Gateway` 名称，表示"当流量从该 Gateway 进入时，应用以下路由规则"

## 使用建议

- 如果集群已部署 Higress 并作为 Ingress Controller，**推荐优先使用 Ingress 方式**（方案 A），更简洁、更云原生
- 如果需要 Istio 特有的高级流量治理能力（如流量镜像、超时重试、故障注入等），可使用 Gateway + VirtualService 方式（方案 B）
