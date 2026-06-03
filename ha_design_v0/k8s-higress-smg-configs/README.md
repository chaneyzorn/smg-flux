# K8s Higress + SMG PD 分离架构配置

本目录包含 Higress（Istio/Envoy）体系下，将流量路由到 sgl-model-gateway（SMG）PD 分离集群的完整 Kubernetes 资源配置。

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        外部客户端                              │
│              api.kimi-router.example.com                     │
└────────────────────────┬────────────────────────────────────┘
                         │
              ┌──────────▼──────────┐
              │   Higress Gateway   │  ← Ingress / Gateway
              │   (Istio/Envoy)     │
              └──────────┬──────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
   │ 5p3d-0  │     │ 5p3d-1  │     │ 5p3d-2  │  ← ServiceEntry
   │ (SMG-1) │     │ (SMG-2) │     │ (SMG-3) │
   └────┬────┘     └────┬────┘     └────┬────┘
        │               │               │
   ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
   ▼         ▼     ▼         ▼     ▼         ▼
Prefill-x  Decode-y  ...  ...  ...  ...
```

---

## 文件清单

| 文件 | 类型 | 说明 |
|------|------|------|
| `serviceentry.yaml` + `.md` | **核心必选** | 将 3 个 SMG 分片注册为 Istio 上游集群 |
| `ingress.yaml` + `.md` | **方案 A 必选** | Higress 原生 Ingress 方式，外部流量接入 |
| `gateway.yaml` + `.md` | **方案 B 可选** | Istio Gateway 入口网关 |
| `virtualservice.yaml` + `.md` | **方案 B 可选** | Istio 路由规则，配合 Gateway 使用 |
| `destinationrule.yaml` + `.md` | **推荐选配** | 负载均衡、连接池、熔断驱逐策略 |

---

## 部署方式

### 方案 A：Higress Ingress（推荐）

适用于以 Higress 作为云原生 API 网关的场景，配置简洁，与 K8s 原生资源无缝集成。

```bash
# 核心 + Ingress
kubectl apply -f serviceentry.yaml
kubectl apply -f ingress.yaml

# 可选：追加负载均衡和熔断策略
kubectl apply -f destinationrule.yaml
```

### 方案 B：Istio Gateway + VirtualService

适用于需要 Istio 特有高级流量治理能力（流量镜像、故障注入、按路径分流等）的场景。

```bash
# 核心 + Gateway + VirtualService
kubectl apply -f serviceentry.yaml
kubectl apply -f gateway.yaml
kubectl apply -f virtualservice.yaml

# 可选：追加负载均衡和熔断策略
kubectl apply -f destinationrule.yaml
```

---

## 关键设计要点

### 1. 分片自治

每个 SMG 分片（`5p3d-0`、`5p3d-1`、`5p3d-2`）是**独立自治的 PD 集群**：
- 各自管理独立的 Prefill / Decode worker 池
- 各自维护独立的 cache_aware 前缀路由树
- 故障隔离：单分片故障不影响其他分片

### 2. 高可用设计

| 层级 | 机制 |
|------|------|
| Higress 网关 | 多实例部署，自身高可用 |
| SMG 路由层 | 3 副本分片，单副本故障自动切流 |
| Worker 层 | 每个 SMG 内部多 P/D worker，单节点故障由 SMG 内部切换 |

### 3. 负载均衡分层

- **Higress 层**：跨 SMG 分片的粗粒度负载均衡（轮询 / 最小连接）
- **SMG 层**：分片内部的细粒度 cache_aware 路由，把相似前缀请求粘到同一 worker

---

## 命名约定

```
kimi-k25-router-5p3d-0.default.svc.cluster.local
│    │    │      │    │
│    │    │      │    └── 副本序号：0, 1, 2
│    │    │      └─────── PD 配置：5 个 Prefill + 3 个 Decode
│    │    └────────────── 角色：router（SMG 路由层）
│    └─────────────────── 模型：Kimi K2.5
└──────────────────────── 业务域：kimi
```

---

## 常见问题

### Q：ServiceEntry 和 K8s Service 有什么区别？

A：`ServiceEntry` 用于将**非 K8s Service 管理的端点**（如外部服务、其他命名空间的服务、虚拟域名）纳入 Istio 流量管理。本场景中，3 个 SMG 分片虽然运行在 `default` namespace，但 Higress 通过 `ServiceEntry` 可以跨 namespace 引用它们，并赋予统一的虚拟域名。

### Q：为什么不直接用一个 K8s Service 做负载均衡？

A：K8s Service 只提供 L4 负载均衡（iptables/IPVS），而 Istio `ServiceEntry` + `DestinationRule` 可以提供 L7 治理能力（HTTP 路由、熔断、重试、按 header 分流等），更适合 SMG 这种需要精细流量控制的场景。

### Q：可以只部署 ServiceEntry，不部署 Ingress 吗？

A：可以，但此时 `kimi-router-ha-cluster.local` 只在网格内部可达。如果客户端在集群外部，必须通过 `Ingress` 或 `Gateway` 暴露入口。
