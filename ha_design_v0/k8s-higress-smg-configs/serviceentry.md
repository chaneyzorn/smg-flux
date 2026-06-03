# ServiceEntry — SMG PD 分离集群服务注册

## 资源说明

`ServiceEntry` 是 Istio 的核心 CRD，用于将**外部服务**或**未自动注册到服务网格的服务**纳入 Istio 的流量管理。

在本场景中，Higress 通过 `ServiceEntry` 将下游的 3 个 SMG 分片注册为网格内部可调用的上游集群，对外暴露统一虚拟域名，实现高可用负载均衡。

## 字段解析

| 字段 | 值 | 说明 |
|------|-----|------|
| `apiVersion` | `networking.istio.io/v1beta1` | Istio 网络 API 版本 |
| `kind` | `ServiceEntry` | 资源类型：服务条目 |
| `metadata.name` | `kimi-router-ha-se` | 资源名称，`kimi-router` 的高可用 ServiceEntry |
| `metadata.namespace` | `higress-system` | 部署在 higress-system 命名空间 |
| `metadata.labels.app` | `kimi-router` | 应用标签 |
| `metadata.labels.model` | `k2.5` | 模型标签，标识服务的是 Kimi K2.5 模型 |
| `metadata.labels.tier` | `router` | 层级标签，标识这是路由层 |

### spec.endpoints

后端端点列表，定义了 3 个独立的 SMG 分片副本：

| 地址 | 含义 |
|------|------|
| `kimi-k25-router-5p3d-0.default.svc.cluster.local` | 分片 0：5 个 Prefill + 3 个 Decode |
| `kimi-k25-router-5p3d-1.default.svc.cluster.local` | 分片 1：5 个 Prefill + 3 个 Decode |
| `kimi-k25-router-5p3d-2.default.svc.cluster.local` | 分片 2：5 个 Prefill + 3 个 Decode |

命名约定：
- `kimi-k25`：服务的是 Kimi K2.5 模型
- `router`：对应 sgl-model-gateway（SMG）路由层
- `5p3d`：**5 个 Prefill worker + 3 个 Decode worker** 的 PD 分离配置
- `-0 / -1 / -2`：3 个副本（Replicas），对应架构图中的 SMG-1、SMG-2、SMG-3
- `default.svc.cluster.local`：这些 SMG 实例运行在 `default` namespace

### spec.hosts

- `kimi-router-ha-cluster.local`：虚拟服务域名（VIP）
- 客户端只需访问这个统一域名，Higress/Istio 会自动将请求负载均衡到 3 个后端实例

### spec.location

- `MESH_INTERNAL`：标识为网格内部服务
- 表示该服务在 Istio 服务网格内部，享受完整的流量治理能力（负载均衡、熔断、重试等）

### spec.ports

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `http` | 端口名称 |
| `number` | `30002` | SMG 的 HTTP 服务端口 |
| `protocol` | `HTTP` | 明文 HTTP（集群内部通信，外层可能有 TLS 终结） |

### spec.resolution

- `DNS`：通过 Kubernetes DNS 解析 `*.svc.cluster.local` 得到 Pod IP
- Istio 会监听 DNS 解析结果，动态更新后端端点列表

## 历史配置

此前曾部署过 4P4D 单实例配置（已注释掉）：

```yaml
# - address: kimi-k25-router-4p4d-0.default.svc.cluster.local
```

保留原因：
- 快速回滚（需要时取消注释即可切回）
- 作为配置变更历史文档留存
