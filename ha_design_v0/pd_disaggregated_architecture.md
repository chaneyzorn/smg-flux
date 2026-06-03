# 大规模 PD 分离架构请求分发处理架构

## 架构概览

该架构采用**两层路由 + 分片自治**的设计，将大规模请求通过外层网关分发到多个独立的 PD 子集群，每个子集群内部独立完成 Prefill 和 Decode 的协同调度。

```
┌─────────────────────────────────────────────────────────────┐
│                        Higress 入口层                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  gateway-1  │  │  gateway-2  │  │  gateway-3  │  ...    │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
   │ SMG-1 (主)  │      │    SMG-2    │      │    SMG-3    │     ......
   │ SMG-1 (备)  │      │             │      │             │
   └─────────────┘      └─────────────┘      └─────────────┘
        │                     │                     │
   ┌────┴────┐           ┌────┴────┐           ┌────┴────┐
   ▼         ▼           ▼         ▼           ▼         ▼
Prefill-x  Decode-y   Prefill-x' Decode-y'  Prefill-x'' Decode-y''
```

---

## 分层解析

### 第一层：Higress 入口网关（L7 负载均衡）

Higress 作为最外层流量入口，承载所有外部请求的接入和初步调度：

- **流量收敛**：统一 TLS 终结、认证、限流、WAF 等通用网关能力
- **健康感知**：基于下游 SMG 的健康状态动态调整流量分发
- **负载均衡策略**：可按权重、连接数、租户 ID、模型名称等维度分发

> **注意**：此处的 gateway 是通用七层网关，非 sgl-model-gateway（SMG）。

---

### 第二层：SMG 集群（PD 路由层）

每个 **SMG（SGLang Model Gateway）** 是一个独立的 PD 路由控制平面：

| SMG 实例 | 特性说明 |
|---------|---------|
| **SMG-1** | 采用**主备高可用**架构，主节点故障时备用节点接管，保障该分片路由不间断 |
| **SMG-2 / SMG-3** | 独立运行，与 SMG-1 互不依赖 |
| **...** | 可横向扩展更多分片 |

**关键设计原则**：每个 SMG 是**自治的 PD 集群**，内部独立维护：

- Worker 注册表（Worker Registry）
- 前缀路由树（Cache-Aware Policy）
- 健康检查与熔断状态（Circuit Breaker）
- 负载监控与负载均衡策略

不同 SMG 之间**不共享任何状态**。

---

### 第三层：Worker 池（每个 SMG 内部）

每个 SMG 下挂载独立的 Prefill 和 Decode Worker：

```
SMG-1                    SMG-2                    SMG-3
├─ Prefill-1             ├─ Prefill-1             ├─ Prefill-1
├─ Prefill-2             ├─ Prefill-2             ├─ Prefill-2
├─ ......                ├─ ......                ├─ ......
├─ Prefill-x             ├─ Prefill-x'            ├─ Prefill-x''
├─ Decode-1              ├─ Decode-1              ├─ Decode-1
├─ Decode-2              ├─ Decode-2              ├─ Decode-2
├─ ......                ├─ ......                ├─ ......
└─ Decode-y              └─ Decode-y'             └─ Decode-y''
```

**命名约定说明**：
- `x / y`：SMG-1 的 Prefill/Decode worker
- `x' / y'`：SMG-2 的独立 Prefill/Decode worker
- `x'' / y''`：SMG-3 的独立 Prefill/Decode worker

即使名称同为 `Prefill-1`，不同 SMG 下的实例是**完全独立的物理节点**，彼此无共享。

---

## 请求完整流转路径

以一个生成请求为例，数据流向如下：

```
┌────────┐
│ Client │
└───┬────┘
    │ ① 发送请求（含 prompt、model 等）
    ▼
┌─────────────────────────────────────┐
│ ② Higress 按策略选择下游 SMG 分片    │
│    （如：按 model hash、租户 ID、轮询）│
└─────────────┬───────────────────────┘
              │
    ┌─────────┴─────────┐
    ▼                   ▼
┌──────────┐      ┌──────────┐
│ SMG-1(主)│      │  SMG-2   │
└────┬─────┘      └────┬─────┘
     │ ③ cache_aware 选 P worker
     │    （前缀匹配 → 命中已有缓存的 Prefill 节点）
     ▼
┌─────────────┐
│  Prefill-2  │ ④ 执行 Prefill，计算 prefix KV Cache
│  (SGLang)   │
└──────┬──────┘
       │ ⑤ SMG 将 bootstrap 元数据（host/port/room）注入请求
       │    并选中 Decode worker
       ▼
┌─────────────┐
│  Decode-1   │ ⑥ 通过 bootstrap 连接 Prefill-2，拉取 KV Cache
│  (SGLang)   │    继续执行 Decode 自回归生成 token
└──────┬──────┘
       │ ⑦ 返回生成结果
       ▼
┌──────────┐
│  Client  │
└──────────┘
```

---

## 核心设计思想

### 1. 分片自治（Sharding & Autonomy）

| 对比项 | 单 SMG 大集群 | 多 SMG 分片架构 |
|-------|-------------|---------------|
| **路由状态规模** | 一个 SMG 维护所有 P/D worker 的前缀树，规模大、匹配开销高 | 每个 SMG 只管理小规模 worker（如 x P + y D），前缀树精简，匹配高效 |
| **故障域** | 单点 SMG 故障影响全部流量 | 单 SMG 故障仅影响该分片，Higress 可快速切流 |
| **扩展瓶颈** | SMG 作为中心路由节点，cache_aware 状态难以水平扩展 | 新增分片即扩展容量，各 SMG 状态独立 |
| **多租户隔离** | 多业务/多模型共用 worker 池，互相抢占缓存 | 每个 SMG 可独立服务特定模型或租户，完全隔离 |

### 2. 两层负载均衡

- **Higress（粗粒度）**：负责跨 SMG 分片的流量分发，不关心 PD 细节
- **SMG（细粒度）**：在分片内部做 cache_aware 路由，把相似前缀请求"粘"到同一 worker，最大化 KV Cache 复用

### 3. Cache-Aware 最大化 KV Cache 命中

每个 SMG 内部独立运行 `cache_aware` 策略：
- 维护 `prefill::model_id` 和 `decode::model_id` 两棵前缀树
- 多轮对话中，相同前缀的请求被固定到同一组 P/D worker
- Worker 本地的 SGLang Radix Cache 持续命中，避免重复 prefix 计算

### 4. 高可用设计

- **SMG 层**：SMG-1 采用主备模式，避免单点路由故障
- **Worker 层**：每个 SMG 内部有多个 P/D worker，单 worker 故障时 SMG 可自动切换到同池其他节点
- **网关层**：Higress 多实例 + 健康检查，自动屏蔽异常 SMG

---

## 适用场景

该架构主要面向以下大规模推理场景：

- **超大规模单模型部署**：单模型需要数百甚至上千张 GPU，单 SMG 无法承载全部 worker 的路由状态
- **多模型混部**：不同模型对缓存策略、资源配比要求不同，分片隔离可避免互相干扰
- **多租户 SaaS 平台**：每个租户分配独立 SMG 分片，保障 SLA 和资源隔离
- **长上下文高频复用场景**：如 RAG、Agent 多轮对话，prefix cache 命中率对成本影响巨大

---

## 总结

该架构的核心是 **"分片自治、两层路由、缓存亲和"**：

1. **Higress** 做粗粒度流量分片，把请求映射到某个 SMG 自治域
2. **每个 SMG** 在独立的小规模 worker 池内做细粒度 cache_aware 调度
3. **Worker 完全隔离**，不同 SMG 的 P/D 节点互不共享，故障和缓存都在分片内闭环

通过这种方式，既保留了 PD 分离带来的推理效率优势，又通过分片解决了单控制平面的扩展瓶颈，实现了超大规模集群下的高吞吐、高可用、高缓存命中率。


---

## K8s 配置清单

上述架构在 Kubernetes 中的具体实现，已拆分到 `k8s-higress-smg-configs/` 目录，YAML 配置与文档注释分离，便于直接部署和维护：

```
k8s-higress-smg-configs/
├── README.md                    # 总览文档：架构说明、部署命令、常见问题
├── serviceentry.yaml            # 核心：3 个 SMG 分片注册为 Istio 上游集群
├── serviceentry.md              # ServiceEntry 字段解析、命名约定、历史配置
├── ingress.yaml                 # 方案 A：Higress 原生 Ingress 外部流量接入
├── ingress.md                   # Ingress 注解说明、可选会话保持扩展
├── gateway.yaml                 # 方案 B：Istio Gateway 入口网关
├── gateway.md                   # Gateway 与 VirtualService 的关系说明
├── virtualservice.yaml          # 方案 B：Istio 路由规则
├── virtualservice.md            # VirtualService 字段解析、扩展能力（超时/重试/镜像/分流）
├── destinationrule.yaml         # 推荐选配：负载均衡、连接池、熔断驱逐
└── destinationrule.md           # 策略选择建议、生产调参指南
```

### 拆分原则

| 文件类型 | 用途 |
|---------|------|
| **`.yaml`** | 纯净 K8s 资源，可直接 `kubectl apply`，无注释干扰 |
| **`.md`** | 同名字段详解、使用建议、可选扩展、生产调优 |

### 部署方式速查

**方案 A（推荐）：Higress Ingress**
```bash
kubectl apply -f k8s-higress-smg-configs/serviceentry.yaml
kubectl apply -f k8s-higress-smg-configs/ingress.yaml
# 可选
kubectl apply -f k8s-higress-smg-configs/destinationrule.yaml
```

**方案 B：Istio Gateway + VirtualService**
```bash
kubectl apply -f k8s-higress-smg-configs/serviceentry.yaml
kubectl apply -f k8s-higress-smg-configs/gateway.yaml
kubectl apply -f k8s-higress-smg-configs/virtualservice.yaml
# 可选
kubectl apply -f k8s-higress-smg-configs/destinationrule.yaml
```
