# Ingress — Higress 外部流量接入

## 资源说明

`Ingress` 是 Kubernetes 标准的七层路由资源。在本场景中，Higress 作为 Ingress Controller，将外部域名流量接入到 `ServiceEntry` 定义的虚拟服务。

这是**方案 A（推荐）**：使用 Higress 原生 Ingress 方式，云原生 API 网关模式。

## 字段解析

| 字段 | 值 | 说明 |
|------|-----|------|
| `apiVersion` | `networking.k8s.io/v1` | K8s 网络 API 版本 |
| `kind` | `Ingress` | 资源类型：入口路由 |
| `metadata.name` | `kimi-router-ingress` | Ingress 资源名称 |
| `metadata.namespace` | `higress-system` | 部署在 higress-system 命名空间 |
| `metadata.annotations.kubernetes.io/ingress.class` | `higress` | 指定使用 Higress 作为 Ingress Controller |
| `metadata.annotations.higress.io/destination` | `kimi-router-ha-cluster.local` | **关键注解**：将请求重写到 ServiceEntry 定义的虚拟域名 |

### spec.rules

定义外部访问规则：

| 字段 | 值 | 说明 |
|------|-----|------|
| `host` | `api.kimi-router.example.com` | 外部暴露的域名，客户端实际访问的地址 |
| `path` | `/` | 路径前缀匹配，所有请求都转发 |
| `pathType` | `Prefix` | 前缀匹配模式 |
| `backend.service.name` | `kimi-router-ha-se` | 后端服务名称，指向 ServiceEntry 资源 |
| `backend.service.port.number` | `30002` | 后端服务端口 |

## 可选注解扩展

如需更精细的会话保持或负载均衡策略，可追加以下注解：

```yaml
annotations:
  # 会话保持：基于 Cookie 的粘性会话
  # higress.io/session-affinity: "cookie"

  # 负载均衡策略：轮询 / 最小连接 / 一致性哈希
  # higress.io/loadbalance: "roundrobin"
  # higress.io/loadbalance: "least_conn"
  # higress.io/loadbalance: "consistentHash"
```

> **注意**：`higress.io/destination` 是 Higress 特有的注解，用于将 Ingress 的后端指向 Istio ServiceEntry 定义的虚拟服务，而非传统的 K8s Service。
