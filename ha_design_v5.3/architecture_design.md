# V5.3 架构设计：控制面解耦与状态订阅

> **定位**：本文档聚焦架构层面的设计决策，不展开到实现级代码。
>
> **与 V5.2 的关系**：V5.3 是 V5.2 的**架构演进**。V5.2 已经确立了"确定性共识 + Pub/Sub + 全局 locked 状态"的核心思想，并论证了其可行性；V5.3 在此基础上追问一个架构层面的问题——**这些控制面逻辑（探测、聚合、CAS 选举、阻塞查询）是否必须困在 proxy-wasm 沙箱内？** 如果将它们解耦到一个独立协调器中，WASM 仅保留数据面职责（路由决策 + 被动重试），边界能否更清晰、测试能否更充分、协议选择能否更自由？本文档就是这个追问的系统化展开。

---

## 1. 起点：为什么控制面需要从 WASM 中解耦

V5.2 的设计中，WASM 插件同时承担了数据面和控制面的双重职责：

| 层面 | V5.2 WASM 插件内的职责 |
|------|----------------------|
| **数据面** | 请求路径路由决策、被动重试（fail-fast） |
| **控制面** | 主动探测、健康聚合、确定性选举、CAS 写 Consul、阻塞查询订阅 |

这种混杂带来三个结构性问题：

**问题一：proxy-wasm 的约束绑架了控制面设计**
- WASM 只能发 HTTP 请求（`DispatchHttpCall`），不能使用 gRPC streaming、自定义 TCP 协议
- etcd（gRPC streaming）和 ZooKeeper（自定义 TCP）因此不可用
- goroutine 必须在 host call 返回前结束，无法运行真正的后台事件循环
- 控制面逻辑被迫在一个不合适的沙箱内实现

**问题二：控制面与数据面边界模糊**
- 同一个 WASM VM 内既跑流量代理逻辑，又跑状态同步逻辑
- 代码难以独立测试（WASM 测试框架远弱于标准 Go）
- 控制面升级和数据面升级被绑定在一起

**问题三：探测写放大**
- N 个 Gateway Pod × M 个 endpoints 的探测，每个 Pod 独立写 Consul
- 从 Consul 视角，这是 N 倍于必要值的写放大

基于以上观察，一个更干净的架构应该是：

> **控制面逻辑全部解耦到一个独立协调器中。协调器提供订阅机制，WASM 单向订阅变更。对 Consul 的复杂读写全部放到协调器中。WASM 只负责订阅变更，并负责请求路径上的重试。**

这就是 V5.3 的起点。

---

## 2. Higress 生态调研：控制面与数据面的真实边界

在提出具体架构之前，必须先理解 Higress 生态中控制面和数据面的真实能力边界。以下三项调研结论直接决定了 V5.3 的设计空间。

### 2.1 调研一：Higress Gateway Pod 的容器结构

**关键发现**：

- Higress Gateway Deployment 默认只有 **1 个容器**（`higress-gateway`），开启可观测性时扩展至 2 个（注入 `promtail`）
- `higress-gateway` 镜像内同时包含 **Envoy** 和 **pilot-agent** 两个二进制，但它们以**同容器多进程**方式运行，pilot-agent 通过 **UDS** 与 Envoy 通信，负责 Envoy 的生命周期管理和 xDS 代理
- Gateway Pod 带有 `sidecar.istio.io/inject: "false"`，**无动态 sidecar 注入机制**
- 容器列表在 Helm Template（`_pod.tpl`）中**硬编码**，如需添加额外容器，必须**修改 Higress Helm chart 或维护自定义 Deployment YAML**
- Pod 内容器共享同一个 network namespace，**可以通过 localhost 互相访问端口**

**对 V5.3 的影响**：
- 协调器可以作为同一 Pod 内的第二个容器，与 Envoy/WASM 通过 localhost 通信
- 但添加容器需要修改 Helm chart，这是 V5.3 相比 V5.2 的额外运维成本

### 2.2 调研二：WASM → localhost HTTP 可行性

**关键发现**：

- proxy-wasm 的 `DispatchHttpCall` **不能直接写裸 IP**，第一个参数必须是 Envoy 中**已存在的 cluster name**
- Higress 的 `wrapper.HttpClient` 支持通过 **`.static` 固定地址服务**指向 localhost：`StaticIpCluster` 的 `Host` 字段可设为 `127.0.0.1`，生成 `outbound|{port}||{name}.static` cluster
- **多个 Higress 官方插件已有生产先例**：`ai-intent`（访问 `127.0.0.1:80`）、`ai-json-resp`（`localservice.static`）、`oidc`（`keycloak.static` + `service_host: 127.0.0.1:9090`）、`opa`（`opa-server` cluster 指向 `127.0.0.1:8181`）
- 同 Pod localhost 通信的网络层延迟为**亚毫秒级**（内核 loopback），但 WASM hostcall 穿越和 Envoy Async Client 调度将总开销推高到 **< 1ms**，对配置同步类请求完全可接受

**对 V5.3 的影响**：
- WASM → 协调器的通信路径可行，但需要在 Higress 中创建一个 `.static` 固定地址服务（如 `smg-coordinator.static` → `127.0.0.1:8888`）

### 2.3 调研三：Envoy 轻量级状态推送机制

**关键发现**：

- **Envoy RTDS**（Runtime Discovery Service）是 xDS 正式成员，支持通过 xDS 动态推送 key-value，但 proxy-wasm 规范**没有暴露读取 runtime/RTDS 值的 ABI**，WASM 无法使用
- **WASM SharedData** 仅限**单个 Envoy 进程内**共享，不跨 Pod、不跨生命周期；Higress `ai-proxy failover.go` 证实这一点
- **proxy-wasm 没有"外部直接注入状态"的 ABI**：WASM 插件无法被动接收控制面推送的状态，**必须自己发 HTTP 请求**（`DispatchHttpCall`）
- Higress 标准管道（K8s CRD → Istio API → xDS → Envoy）是纯配置推送，**没有专门的"非配置类数据"通道**；xDS 推送延迟为**秒级**（istiod debounce + 生成 + 推送），远慢于 failover 所需的亚秒级

**对 V5.3 的影响**：
- 在 Higress/Envoy 的约束下，**不存在比 WASM 自己发 HTTP 请求更轻量的状态同步机制**
- WASM 长轮询订阅 localhost 协调器，已经是约束下的**最优解**

### 2.4 调研结论汇总

| 维度 | 结论 | 对 V5.3 的约束/机会 |
|------|------|-------------------|
| Gateway Pod 结构 | 可添加容器，但需修改 Helm chart | 协调器可同 Pod 部署，运维成本增加 |
| localhost 通信 | 可行，需 `.static` 服务 | WASM → 协调器走 localhost，延迟 < 1ms |
| 更轻量同步机制 | **不存在** | WASM 长轮询是唯一可行路径 |
| xDS/RTDS | WASM 不可读，延迟秒级 | 不能替代长轮询 |

---

## 3. V5.3 核心设计

### 3.1 核心思想

V5.3 的核心是**控制面与数据面的职责分离**：

| 组件 | 职责 | 约束 |
|------|------|------|
| **协调器（Coordinator）** | 探测、聚合、选举、CAS 写、阻塞查询订阅、HTTP 长轮询服务 | 无 WASM 约束，标准 Go 二进制 |
| **WASM 插件** | 长轮询订阅协调器、本地缓存更新、请求路径路由、被动重试 | proxy-wasm 约束 |

协调器封装所有与 Consul 的复杂交互，WASM 只通过 HTTP 长轮询"订阅"状态变更。这里的"订阅"在 proxy-wasm 约束下的实际实现是**链式长轮询**：WASM 发一个长超时 HTTP 请求，协调器阻塞到状态变化后返回，WASM callback 更新缓存并立即发起下一个请求。

### 3.2 命名与部署位置

控制面组件部署在 **Higress Gateway Pod** 内，与 Envoy/WASM 同 Pod，通过 localhost 通信。

> ⚠️ **为什么不叫"Sidecar"**："Sidecar"在 K8s 语境中容易被误解为与业务 Pod（即 SMG Pod）在一起。但 WASM 运行在 Higress Gateway Pod 内，协调器必须与 WASM 同 Pod 才能实现 localhost 通信。为避免歧义，本文档使用 **"SMG Routing Coordinator"（简称"协调器"）** 指代该组件。

### 3.3 部署拓扑

```
┌─────────────────────────────────────────────────────────────────────┐
│                         K8s 集群                                    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Higress Gateway Deployment (3 replicas)         │   │
│  │                                                              │   │
│  │  ┌─────────────────────┐    ┌─────────────────────────┐     │   │
│  │  │   Gateway Pod #1     │    │   Gateway Pod #2         │     │   │
│  │  │  ┌───────────────┐  │    │  ┌───────────────┐      │     │   │
│  │  │  │   Envoy       │  │    │  │   Envoy       │      │     │   │
│  │  │  │  ┌─────────┐  │  │    │  │  ┌─────────┐  │      │     │   │
│  │  │  │  │ WASM    │  │  │    │  │  │ WASM    │  │      │     │   │
│  │  │  │  │ Plugin  │  │  │    │  │  │ Plugin  │  │      │     │   │
│  │  │  │  └────┬────┘  │  │    │  │  └────┬────┘  │      │     │   │
│  │  │  │       │       │  │    │  │       │       │      │     │   │
│  │  │  │  ┌────┴────┐  │  │    │  │  ┌────┴────┐  │      │     │   │
│  │  │  │  │ 协调器  │  │  │    │  │  │ 协调器  │  │      │     │   │
│  │  │  │  │ (同 Pod)│  │  │    │  │  │ (同 Pod)│  │      │     │   │
│  │  │  │  └────┬────┘  │  │    │  │  └────┬────┘  │      │     │   │
│  │  │  └───────┼───────┘  │    │  └───────┼───────┘      │     │   │
│  │  │          │          │    │          │              │     │   │
│  │  │    localhost:8888   │    │    localhost:8888       │     │   │
│  │  └──────────┼──────────┘    └──────────┼──────────────┘     │   │
│  │             │                          │                     │   │
│  └─────────────┼──────────────────────────┼─────────────────────┘   │
│                │                          │                         │
│                └────────────┬─────────────┘                         │
│                             │ HTTP                                  │
│                    ┌────────┴────────┐                              │
│                    │  Consul KV (3)   │                              │
│                    │  smg/health/*   │                              │
│                    │  smg/locked/*   │                              │
│                    └─────────────────┘                              │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              SMG StatefulSet (N groups × 2 replicas)         │   │
│  │                                                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ SMG G1-R1│  │ SMG G1-R2│  │ SMG G2-R1│  │ SMG G2-R2│    │   │
│  │  │ (主)     │  │ (备)     │  │ (主)     │  │ (备)     │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │   │
│  │       ▲              ▲                                       │   │
│  │       └──────────────┘                                       │   │
│  │            协调器探测目标                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.4 架构分层

```
┌────────────────────────────────────────────────────────────┐
│                      控制面 (Control Plane)                  │
│                                                             │
│  ┌─────────────────┐      ┌─────────────────────────────┐  │
│  │   Consul KV     │      │   协调器 (每 Gateway Pod 一个)│  │
│  │  (共享状态存储)  │      │                             │  │
│  │  - health/*     │◄────►│  - 主动探测                  │  │
│  │  - locked/*     │      │  - 健康聚合                  │  │
│  └─────────────────┘      │  - 确定性选举                │  │
│                           │  - CAS 写 locked             │  │
│                           │  - 阻塞查询订阅               │  │
│                           │  - HTTP 长轮询服务            │  │
│                           └─────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                              │
                              │ 长轮询订阅 (localhost HTTP)
                              ▼
┌────────────────────────────────────────────────────────────┐
│                      数据面 (Data Plane)                     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Higress Gateway Pod                     │   │
│  │                                                     │   │
│  │  ┌─────────────┐        ┌─────────────────────────┐ │   │
│  │  │   Envoy     │        │   WASM Plugin           │ │   │
│  │  │             │◄──────►│  - 长轮询订阅协调器      │ │   │
│  │  │             │        │  - 本地 locked 缓存      │ │   │
│  │  │             │        │  - 请求路径路由决策      │ │   │
│  │  │             │        │  - 被动重试 (fail-fast)  │ │   │
│  │  └─────────────┘        └─────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

### 3.5 组件职责

**协调器（SMG Routing Coordinator）**：

| 职责 | 说明 | 触发时机 |
|------|------|---------|
| **Bootstrap** | 启动时从 Consul 全量拉取 `smg/?recurse=true` | 启动时 |
| **阻塞查询订阅** | 维持一个到 Consul 的长连接，订阅 KV 变化 | 持续运行 |
| **主动探测** | 定期 HTTP 探测所有 SMG endpoints | 定时器 |
| **健康聚合** | 将探测结果按时间窗口聚合为 HEALTHY/UNHEALTHY | 每次探测后 |
| **确定性选举** | 当 locked endpoint 不健康时，运行选举算法 | 探测发现故障时 |
| **CAS 写** | 对 `smg/locked/{group}` 执行乐观并发写 | 选举出新 locked 时 |
| **HTTP 服务** | 暴露 `/subscribe/locked` 长轮询端点 | 持续运行 |
| **等待者管理** | 维护每个 group 的订阅等待队列 | 状态变化时 |

**WASM Plugin**：

| 职责 | 说明 | 触发时机 |
|------|------|---------|
| **长轮询订阅** | 对每个 Group 维持一条到协调器的长轮询链 | 插件启动时 |
| **本地缓存更新** | 收到协调器推送后更新 `locked` 缓存 | 长轮询响应时 |
| **请求路径路由** | `selectLockedEndpoint` 纯本地读取 | 每个请求 |
| **被动重试** | 请求失败后本地标记 UNHEALTHY | 响应码 503/超时 |
| **降级处理** | 协调器不可用时回退到静态配置 | 协调器连接失败时 |

**Consul KV**：

| 职责 | 说明 |
|------|------|
| **健康报告存储** | `smg/health/{group}/{endpoint}/{source}` 存储原始探测报告 |
| **Locked 状态存储** | `smg/locked/{group}` 存储全局粘滞 endpoint |
| **CAS 支持** | `?cas={ModifyIndex}` 实现乐观并发控制 |
| **阻塞查询** | `?index=` + `?wait=` 实现变更订阅 |

### 3.6 状态流转

#### 正常状态：无变化

```
┌──────┐     ┌────────┐     ┌────────┐     ┌─────────┐
│ WASM │────►│ 协调器 │────►│ Consul │     │  SMG    │
└──────┘     └────────┘     └────────┘     └─────────┘
   │           │              │
   │ GET /subscribe           │
   │ group=X, timeout=30s    │
   │──────────►│              │
   │           │ 阻塞等待      │
   │           │ 30s 无变化   │
   │ 204 No Content           │
   │◄──────────│              │
   │           │              │
   │ 立即重发  │              │
   │ GET /subscribe           │
   │──────────►│              │
```

#### 故障切换：探测发现故障

```
┌──────┐     ┌────────┐     ┌────────┐     ┌─────────┐
│ WASM │     │ 协调器 │     │ Consul │     │  SMG    │
└──────┘     └────────┘     └────────┘     └─────────┘
   │           │              │              │
   │           │ 探测 SMG-1.1 │              │
   │           │─────────────►│              │
   │           │ 返回 503     │              │
   │           │◄─────────────│              │
   │           │              │              │
   │           │ 聚合：SMG-1.1 UNHEALTHY    │
   │           │ 运行选举算法 → SMG-1.2     │
   │           │              │              │
   │           │ CAS 写 locked=SMG-1.2      │
   │           │─────────────►│              │
   │           │ 成功         │              │
   │           │◄─────────────│              │
   │           │              │              │
   │           │ 唤醒等待者   │              │
   │ {endpoint:SMG-1.2}      │              │
   │◄──────────│              │              │
   │ 更新缓存  │              │              │
   │ 立即重发新订阅            │              │
   │──────────►│              │              │
```

#### 被动重试：WASM 本地 fail-fast

```
┌──────┐     ┌────────┐     ┌────────┐     ┌─────────┐
│ WASM │     │ 协调器 │     │ Consul │     │  SMG    │
└──────┘     └────────┘     └────────┘     └─────────┘
   │           │              │              │
   │ 请求 SMG-1.1 (locked)   │              │
   │───────────────────────────────────────►│
   │           │              │              │
   │           │              │         返回 503
   │◄───────────────────────────────────────│
   │           │              │              │
   │ 本地标记 SMG-1.1 UNHEALTHY             │
   │ 请求 SMG-1.2 (electNewLocked)          │
   │───────────────────────────────────────►│
   │           │              │              │
   │           │ 【注意】此时 Consul 中的      │
   │           │ locked 可能还是 SMG-1.1     │
   │           │ 这是有界分叉，可接受          │
```

---

## 4. 通信机制

### 4.1 WASM → 协调器

proxy-wasm 的 `DispatchHttpCall` 必须通过 Envoy **cluster name** 调用，不能直接写裸 IP。根据协调器的部署位置，可行方式如下：

**方式一：同 Pod 部署（localhost）**

| 机制 | 延迟 | 可行性 | 说明 |
|------|------|--------|------|
| `.static` 服务指向 `127.0.0.1:8888` | < 1ms | ✅ **已确认可行** | Higress 官方技能文档和多个插件（ai-intent、ai-json-resp、oidc、opa）均使用此模式。Higress 自动生成 `outbound|8888||smg-coordinator.static` cluster |

**方式二：DaemonSet 部署（K8s Service）**

| 机制 | 延迟 | 可行性 | 说明 |
|------|------|--------|------|
| `.dns` 服务指向 K8s Service FQDN（如 `smg-coordinator.higress-system.svc.cluster.local`） | ~1ms | ✅ **可行** | `DnsCluster` 的 `Domain` 字段填 Service FQDN，K8s DNS 解析为 ClusterIP，kube-proxy 负载均衡到 DaemonSet Pod。同节点优先，跨节点延迟仍在 K8s 内网范围（< 2ms），对配置同步完全可接受 |

**不可行的方式**：

| 方式 | 不可行原因 |
|------|-----------|
| UDS (Unix Domain Socket) | proxy-wasm `DispatchHttpCall` ABI 只接受 cluster name（TCP endpoint），不支持 UDS path |
| Shared Memory | proxy-wasm `SharedData` 是**进程级**，无法跨进程共享 |
| 文件系统 | WASM 无文件系统访问能力 |
| xDS / RTDS | proxy-wasm **无 ABI 读取** runtime；xDS 推送延迟秒级 |

**结论**：同 Pod 用 `.static` 访问 localhost，DaemonSet 用 `.dns` 访问 K8s Service，两种路径均已确认可行。

### 4.2 协调器 → Consul

协调器是标准 Go 二进制，无 WASM 约束：

| 后端 | 协议 | 可行性 | 说明 |
|------|------|--------|------|
| Consul KV | HTTP + 阻塞查询 | ✅ | V5.2 已验证 |
| Redis | RESP | ✅ | 协调器可以直接用 go-redis |
| etcd | gRPC + Watch streaming | ✅ | 协调器可以直接用 etcd client |
| ZooKeeper | 自定义 TCP | ✅ | 协调器可以直接用 zk client |
| K8s API | HTTP + Watch | ✅ | 协调器可以直接用 client-go |

**关键收益**：协调器不受 proxy-wasm HTTP-only 的约束，协议选择完全开放。

---

## 5. 部署模型对比

### 5.1 模型 A：Sidecar 模式（同 Pod）

```
Gateway Pod:
  ┌─────────┐
  │ Envoy   │
  │ + WASM  │
  └────┬────┘
       │ localhost
  ┌────┴────┐
  │ 协调器  │
  └────┬────┘
       │ HTTP
  ┌────┴────────┐
  │ Consul KV   │
  └─────────────┘
```

| 维度 | 评估 |
|------|------|
| 复杂度 | 中：协调器作为额外容器，但 Higress Gateway Pod 的容器列表在 Helm Template（`_pod.tpl`）中**硬编码**，无动态注入机制。需要**修改 Higress Helm chart 或自定义 Deployment 模板**来添加协调器容器 |
| 通信延迟 | < 1ms（localhost，同 network namespace） |
| **网络同构性** | **最优**：协调器和 Gateway 共享完全相同的网络命名空间，探测路径与请求转发路径完全一致 |
| 故障隔离 | Pod 级：协调器挂了不影响其他 Pod |
| 探测写放大 | 无改善：N 个 Pod = N 个协调器 = N 倍探测源 |
| 资源开销 | 每个 Gateway Pod 多一份协调器内存/CPU |
| 升级粒度 | Pod 级：可以独立升级协调器镜像 |
| WASM 访问方式 | `.static` 服务指向 `127.0.0.1:8888` |

Sidecar 模式的核心优势在于**网络同构性的严格保证**：协调器与 Gateway 共享同一个 network namespace，两者到 SMG endpoints 的探测/转发路径经过完全相同的 CNI 链路、iptables 规则、DNS 解析和路由表。这意味着协调器探测到的网络状况（延迟、丢包、连通性）与 Gateway 实际转发请求时遇到的状况完全一致，健康判断对 Gateway 具有最强的参考价值。这是 DaemonSet 和独立 Deployment 都无法达到的精确度。

### 5.2 模型 B：DaemonSet + K8s Service

```
K8s Node:
  ┌─────────────┐  ┌─────────────┐
  │ Gateway Pod │  │ Gateway Pod │
  │ WASM 插件    │  │ WASM 插件    │
  └──────┬──────┘  └──────┬──────┘
         │                │
         │  .dns cluster  │  smg-coordinator.higress-system.svc.cluster.local
         └───────┬────────┘
                 │
         ┌───────┴───────┐
         │  K8s Service   │
         │ (ClusterIP)    │
         └───────┬───────┘
                 │
         ┌───────┴────────┐
         │ 协调器 DaemonSet│
         │  (每节点一个)   │
         └───────┬────────┘
                 │ HTTP
         ┌───────┴────────┐
         │  Consul KV     │
         └────────────────┘
```

| 维度 | 评估 |
|------|------|
| 复杂度 | 低：DaemonSet + Service 是标准 K8s 资源，**不需要修改 Higress Helm chart** |
| 通信延迟 | ~1ms（K8s 内网，kube-proxy 负载均衡到 DaemonSet Pod） |
| **网络同构性** | **良好**：协调器和 Gateway 在同一节点，共享同一节点的 CNI、iptables、路由表，探测路径与转发路径基本一致 |
| 故障隔离 | 节点级：协调器挂了影响节点上所有 Gateway Pod |
| 探测写放大 | **改善**：每节点 1 个探测源，通常少于每 Pod 1 个 |
| 资源开销 | 每节点一份协调器内存/CPU |
| 升级粒度 | 节点级：DaemonSet rolling update |
| WASM 访问方式 | `.dns` 服务指向 K8s Service FQDN |

### 5.3 模型 C：独立 Deployment（可选）

```
Gateway Pod:              协调器 Deployment:          Consul KV:
  ┌─────────┐              ┌─────────┐
  │ Envoy   │              │ 协调器  │
  │ + WASM  │◄────────────►│ (N 副本)│
  └─────────┘   K8s Service └────┬────┘
                                 │
                          ┌──────┴──────┐
                          │ Consul KV   │
                          └─────────────┘
```

| 维度 | 评估 |
|------|------|
| 复杂度 | 中：额外的 Deployment + Service，**不需要修改 Higress Helm chart** |
| 通信延迟 | ~1-5ms（跨 Pod，经过 K8s Service） |
| **网络同构性** | **差**：协调器 Pod 由调度器分配，可能部署在与 Gateway 完全不同的节点或可用区，探测路径与转发路径不一致 |
| 故障隔离 | 协调器作为独立服务需要自己的 HA 设计（通常 2-3 副本） |
| 探测写放大 | 无改善：N 个协调器副本 = N 倍探测源 |
| 结论 | 不如 DaemonSet：Deployment 副本数与节点数无关，既不保证节点覆盖，也无法减少探测源 |

---

## 6. 关键决策与权衡

### 6.1 V5.2 vs V5.3 的核心差异

| 维度 | V5.2 | V5.3（本提案） |
|------|------|---------------|
| 控制面位置 | WASM 插件内 | 协调器（独立容器） |
| WASM 代码复杂度 | 高（探测+聚合+CAS+订阅+路由+重试） | 低（订阅+路由+重试） |
| 协议限制 | HTTP-only（WASM 约束） | 任意（协调器是标准 Go） |
| 部署组件数 | 1（WASM 插件） | 2（插件 + 协调器 DaemonSet/Service） |
| 控制面/数据面边界 | 模糊 | 清晰 |
| 测试可维护性 | WASM 测试框架弱 | 标准 Go 测试 |
| 请求路径延迟 | 0（纯内存） | 0（纯内存，订阅异步） |
| Higress 侵入性 | 无（纯 WASM 插件） | **无**（DaemonSet + Service 不修改 Higress） |
| 额外 K8s 资源 | 无 | DaemonSet + Service + 可能的 RBAC/ConfigMap |

### 6.2 V5.3 的核心收益

1. **架构边界清晰**：控制面逻辑与数据面逻辑物理分离，各自独立演化
2. **协议自由**：协调器不受 proxy-wasm HTTP-only 约束，未来可切换 etcd、ZooKeeper 等协议
3. **测试充分**：协调器是标准 Go 二进制，UT/IT/Benchmark 覆盖完整
4. **WASM 极简**：WASM 代码量从 ~900 行降到 ~300 行，降低维护负担

### 6.3 V5.3 的核心代价

1. **运维复杂度增加**：需要维护 Higress Helm chart 的修改（添加协调器容器）
2. **额外组件**：每个 Gateway Pod 多了一个进程/容器，资源开销增加
3. **启动时序**：需要处理协调器与 Envoy 的启动顺序问题

---

## 7. 开放问题

### 7.1 高优先级

1. **长轮询链的可靠性**
   - 如果协调器重启，WASM 的长轮询请求会收到 connection reset
   - WASM 能否优雅处理并自动重连？
   - 重连期间的降级策略是什么？

2. **启动时序**
   - 协调器和 Envoy 谁先启动？
   - 如果 WASM 先就绪但协调器尚未完成 Bootstrap，WASM 如何行为？
   - 是否需要 startup probe 来保证协调器先于 Envoy 接收流量？

3. **协调器自身的健康检查**
   - 协调器作为控制面组件，如果自身不健康，WASM 如何感知？
   - 是否需要 K8s liveness/readiness probe？
   - WASM 是否需要探测协调器的健康状态？

### 7.2 中优先级

4. **协调器的 resource limit**
   - 协调器的内存/CPU 需求估算？
   - 阻塞查询连接 + 探测 goroutine 的资源开销？

5. **探测聚合的 Phase 2**
   - 同 Pod 模型无法减少探测写放大，何时评估 DaemonSet？
   - DaemonSet 模式下 WASM 如何发现同节点协调器地址？

### 7.3 低优先级

6. **后端协议扩展**
   - 协调器打开了对 etcd/ZooKeeper 的可能性，是否需要评估？
   - 何时值得为协议切换付出工程成本？

---

## 8. 暂不决策的事项

1. **是否推进 V5.3**：取决于团队对"架构纯度"与"Helm 维护成本"的权衡
2. **协调器的部署模型**：同 Pod vs DaemonSet，Phase 1 默认同 Pod
3. **与 V5.2 的关系**：V5.3 替代 V5.2，还是 V5.2 作为 V5.3 的 fallback

---

## 附录 A：参考来源

### Higress 架构与 Gateway Pod 结构

1. **Higress Gateway 默认容器数量与组成**
   - 来源：Higress Helm chart（`helm/higress/templates/_pod.tpl`）及 `values.yaml`
   - 结论：默认 1 个容器（`higress-gateway`），开启 `global.o11y.enabled` 时扩展至 2 个（注入 `promtail`）

2. **Envoy 与 pilot-agent 的关系**
   - 来源：Higress Makefile（`docker.proxyv2` 构建目标）及官方架构文档
   - 结论：同容器多进程，pilot-agent 通过 UDS 与 Envoy 通信，代理 xDS 请求

3. **Gateway Pod 无动态 sidecar 注入**
   - 来源：Higress Helm template（`_pod.tpl`）及 Pod 标签/Annotation
   - 结论：`sidecar.istio.io/inject: "false"`，容器列表硬编码，无 Mutating Webhook

4. **Pod 内容器共享 network namespace**
   - 来源：K8s 标准行为 + Higress `values.yaml`（`gateway.hostNetwork` 默认 `false`）
   - 结论：同一 Pod 内所有容器共享 network namespace，localhost 通信可行

### WASM → localhost HTTP 可行性

5. **proxy-wasm `DispatchHttpCall` 必须通过 cluster name**
   - 来源：proxy-wasm spec 及 Higress `wasm-go` 框架源码
   - 结论：第一个参数必须是 Envoy 中已存在的 cluster name，不能写裸 IP

6. **Higress `.static` 固定地址服务支持 localhost**
   - 来源：Higress 官方 skill 文档（`.claude/skills/higress-wasm-go-plugin/references/http-client.md`）及 `wrapper.HttpClient` 源码
   - 结论：`StaticIpCluster` 的 `Host` 字段可设为 `127.0.0.1`，生成 `outbound|{port}||{name}.static` cluster

7. **Higress 插件访问 localhost 的生产先例**
   - 来源：`higress/plugins/wasm-go/extensions/ai-intent`、`ai-json-resp`、`replay-protection`、`oidc`、`opa` 等插件 README 及源码
   - 结论：多个官方插件通过 `.static` 服务访问 `127.0.0.1` 或同 Pod 服务

8. **同 Pod localhost 通信性能**
   - 来源：Linux 内核 loopback 机制 + Envoy Async Client 架构分析
   - 结论：网络层亚毫秒，WASM hostcall 和内存拷贝将总开销推高到 < 1ms

### Envoy 轻量级状态推送机制

9. **Envoy RTDS（Runtime Discovery Service）**
   - 来源：Envoy 官方文档（xDS 协议规范）
   - 结论：支持通过 xDS 推送 key-value，但 proxy-wasm **无 ABI 读取**

10. **proxy-wasm 无外部状态注入 ABI**
    - 来源：proxy-wasm spec 及 Higress `ai-proxy failover.go` 源码
    - 结论：WASM 无法被动接收状态推送，必须主动发 HTTP 请求

11. **Higress 标准管道无专门非配置类数据通道**
    - 来源：Higress 架构文档及项目内部 `distributed_consistent_dynamic_routing_research.draft.md`
    - 结论：K8s CRD → Istio API → xDS → Envoy 是纯配置推送，xDS 延迟秒级

### V5.2 相关

12. **V5.2 确定性共识设计**
    - 来源：`ha_design_v5.2/active_standby_ha_design_v5.2_research.draft.md`
    - 结论：V5.2 已确立"确定性共识 + Pub/Sub + 全局 locked 状态"的核心思想

13. **Consul KV HTTP API 与阻塞查询**
    - 来源：https://developer.hashicorp.com/consul/api-docs/kv 及 https://developer.hashicorp.com/consul/api-docs/features/blocking
    - 结论：`?index=` + `?wait=` 机制，响应在数据变化或超时后返回
