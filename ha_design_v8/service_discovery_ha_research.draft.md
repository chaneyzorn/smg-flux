# SMG HA V8 方案调研 — 服务发现组件管理拓扑与健康状态

> 目标：评估使用服务注册中心（Consul/Nacos/etcd）替代 Operator（V4）或 WASM 插件（V5.1），通过 Envoy EDS priority 实现主备 failover 的可行性。
>
> 调研范围：Higress McpBridge 能力边界、Envoy EDS priority 机制、服务注册中心健康检查、与 V4/V5.1 的对比。

---

## 一、方案核心思路

```
┌─────────────────────────────────────────────────────────────┐
│                     Higress Gateway                          │
│                                                              │
│  ┌─────────────┐    ┌─────────────────────┐               │
│  │ McpBridge   │───→│ ServiceEntry (xDS)  │───→ Envoy     │
│  │ (watch)     │    │ (EDS with priority) │     clusters  │
│  └──────┬──────┘    └─────────────────────┘               │
│         │                                                    │
│         │ watch / poll                                       │
│         ▼                                                    │
│  ┌─────────────────────────────────────────┐               │
│  │  Service Registry (Nacos / Consul)      │               │
│  │                                          │               │
│  │  • SMG endpoints registered with metadata│               │
│  │  • Health checks (HTTP / TCP)            │               │
│  │  • Weight / priority labels              │               │
│  └─────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────┘
         │                          │
         ▼                          ▼
   ┌──────────┐              ┌──────────┐
   │ SMG-1.1  │              │ SMG-1.2  │
   │ (primary)│              │ (backup) │
   │ priority=0│             │ priority=1│
   └──────────┘              └──────────┘
```

**假设**：如果 SMG 实例注册到服务注册中心，注册中心负责健康检查和拓扑管理，Higress 通过服务发现动态获取健康的 endpoint 列表，Envoy 的 EDS priority 机制实现"主不健康时自动切换到备"，就可以不依赖自定义 Operator 或 WASM 插件。

---

## 二、Higress 服务发现能力（McpBridge）

### 2.1 McpBridge 是什么

McpBridge 是 Higress 的自定义 K8s CRD，用于桥接外部服务注册中心和 Higress/Istio 控制平面。它将注册中心发现的服务转换为 Istio **ServiceEntry**，再通过 xDS 下发给 Envoy。

### 2.2 支持的注册中心

| 注册中心 | 类型值 | 状态 | 备注 |
|----------|--------|------|------|
| Nacos v1/v2/v3 | `nacos` / `nacos2` / `nacos3` | ✅ 支持 | Weight 传播支持 |
| Consul | `consul` | ✅ 支持 | Token 认证 |
| ZooKeeper | `zookeeper` | ✅ 支持 | Dubbo/Spring Cloud 路径 |
| Eureka | `eureka` | ✅ 支持 | |
| Static IPs | `static` | ✅ 支持 | 固定 IP 列表 |
| DNS | `dns` | ✅ 支持 | 域名解析 |
| **etcd** | — | ❌ **不支持** | 非 McpBridge 原生支持类型 |

### 2.3 动态更新延迟

- **默认刷新间隔**：30 秒（Nacos/Consul 均为此默认值）
- **最小可配置间隔**：10 秒（源码中硬编码下限 `DefaultRefreshIntervalLimit = time.Second * 10`）
- **完整更新链路**：Registry → McpBridge watcher → ServiceEntry → Istio Pilot → Envoy xDS
- **总延迟**：注册中心变化到 Envoy 感知，通常 **10–30 秒**

**评估**：SMG 的 failover 需要亚秒级感知，10-30 秒的延迟远高于 V4（~1-2 秒）和 V5.1（亚秒级）。

### 2.4 Weight 与 Priority 支持

- **Nacos**：✅ Weight 可以传播。McpBridge 将 Nacos 的 `Weight`（float64）转换为 Istio `WorkloadEntry.Weight`（uint32）。
- **Consul/ZK/Eureka**：❌ 未找到 Weight 传播的实现。
- **Priority**：❌ **McpBridge 完全不支持**。所有从注册中心发现的 endpoint 都被生成为 flat 的 ServiceEntry，所有 endpoint 的 priority 都是 0。没有机制将注册中心 metadata（如 `datacenter: primary`）映射为 Envoy 的 `LocalityLbEndpoints.priority`。

---

## 三、Envoy EDS Priority 机制

### 3.1 基本原理

Envoy 的 `LocalityLbEndpoints` 支持 `priority` 字段（0 = 最高，N = 更低）。Envoy 使用 **priority load 算法** 结合 **overprovisioning factor**（默认 1.4）决定流量分配：

| P=0 Healthy % | P=0 流量 | P=1 流量 |
|---------------|---------|---------|
| 100% | 100% | 0% |
| 72% | 100% | 0% |
| 71% | ~99% | ~1% |
| 50% | ~70% | ~30% |
| 0% | 0% | 100% |

**关键特性**：
- P=0 的 healthy % 降到约 71.4% 以下时，Envoy 开始向 P=1 切流
- P=0 全部不健康时，100% 流量切到 P=1
- P=0 恢复后，流量自动回切

### 3.2 能否映射到主备语义

**可以映射**：
- `priority: 0` = Primary / Active
- `priority: 1` = Backup / Standby

**配置示例**（原生 Envoy EDS）：

```yaml
load_assignment:
  cluster_name: smg-group-1
  endpoints:
  - priority: 0
    locality:
      zone: primary
    lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: 10.0.1.10
            port_value: 8080
  - priority: 1
    locality:
      zone: backup
    lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: 10.0.1.11
            port_value: 8080
```

**必要条件**：
1. 必须配置 active health check 或 outlier detection，否则 Envoy 认为所有 endpoint healthy，不会 failover
2. P=1（backup）必须有足够容量承受 100% 流量

### 3.3 Higress/Istio 如何暴露 EDS Priority

Istio 的 `DestinationRule` 支持 `localityLbSetting.failover`，但它是**基于地域（region/zone）的 failover**，不是基于 endpoint 的优先级：

```yaml
trafficPolicy:
  loadBalancer:
    localityLbSetting:
      enabled: true
      failover:
        - from: primary-zone
          to: backup-zone
```

**限制**：这要求 endpoint 具有不同的 `locality` 标签，不能用于同一 Group 内的"IP-A → IP-B"这种严格主备场景。

**结论**：Istio/Higress 的高层 CRD 不直接支持基于 EDS priority 的主备 failover。要实现这个，必须直接操作 Envoy 的 EDS 配置（如 EnvoyFilter 或自定义 xDS 控制平面）。

---

## 四、服务注册中心的健康检查能力

### 4.1 Consul

| 维度 | 能力 |
|------|------|
| 健康检查类型 | HTTP、TCP、gRPC、Script、TTL、Docker |
| Watch 机制 | Blocking queries（long-polling，`?index=` + `?wait=`） |
| 主动/被动 | 主动（agent 执行检查）+ 被动（客户端上报） |
| 语义 | 只有 healthy/unhealthy，无 active-standby |
| KV Store | 支持，可用于分布式锁，但不是注册中心原生语义 |

### 4.2 Nacos

| 维度 | 能力 |
|------|------|
| 健康检查类型 | TCP、HTTP、MySQL、用户自定义；agent 模式 + server 模式 |
| Watch 机制 | Push-based subscription（命名服务订阅） |
| Weight | ✅ 原生支持 per-instance weight（float64） |
| 语义 | 只有 healthy/unhealthy，无 active-standby |
| Metadata | 丰富，可附加 `region`、`zone`、`datacenter` 等标签 |

### 4.3 etcd

| 维度 | 能力 |
|------|------|
| 健康检查类型 | TTL-based leases（客户端必须 renew） |
| Watch 机制 | 原生 watch API，revision-based resumption |
| 语义 | 无服务概念，只有 key-value + lease；可用于 leader election |
| Higress 支持 | ❌ McpBridge 不支持 etcd 作为注册中心 |

### 4.4 关键结论

**Consul、Nacos、etcd 都不原生支持 "active-standby" 语义。** 它们只区分 healthy/unhealthy。要实现主备 failover，必须在**代理层**（Envoy priority）或**消费层**（应用逻辑）实现。

---

## 五、与现有方案的对比

| 维度 | V4 (Operator+EndpointSlice) | V5.1 (WASM Plugin) | V8 (Service Discovery) |
|------|------------------------------|-------------------|------------------------|
| **核心组件** | 轻量 Operator | wasm-go 插件 | 外部注册中心 + McpBridge |
| **健康检查** | Operator HTTP 探测（亚秒级） | `RegisterTickFunc` + `/readiness` | 注册中心检查（依赖类型） |
| **故障感知延迟** | ~1-2 秒（探测 + K8s watch） | 亚秒级（tick 周期） | **10-30 秒**（刷新间隔） |
| **Failover 语义** | CRD 自定义（切谁、何时切回） | 插件逻辑（`SetUpstreamOverrideHost`） | Envoy priority（如果配置） |
| **Priority 支持** | ❌ 无（flat EndpointSlice） | ❌ 无（插件 override host） | ❌ **McpBridge 不支持** |
| **Weight 支持** | ❌ 无 | ❌ 无 | ✅ Nacos 支持 |
| **SMG `/readiness` 探测** | ✅ HTTP 探测 `/readiness` | ✅ HTTP 探测 `/readiness` | ✅ HTTP 探测 `/readiness`（但刷新延迟 10-30s） |
| **K8s 原生** | ✅（EndpointSlice + CRD） | ✅（WasmPlugin CRD） | ⚠️（McpBridge CRD + 外部注册中心） |
| **运维复杂度** | Operator 生命周期 | 插件生命周期 | **注册中心集群生命周期** |
| **额外依赖** | 无 | 无 | Consul/Nacos 集群 |

---

## 六、可行性分析

### 6.1 为什么服务发现方案当前不可行

**阻碍一：McpBridge 不支持 Priority 映射**

McpBridge 将所有注册中心 endpoint 转换为 flat ServiceEntry，所有 endpoint priority = 0。没有机制将注册中心 metadata（如 `role: primary` / `role: backup`）映射为 Envoy EDS priority。这是最大的架构缺口。

**阻碍二：刷新延迟太高**

最小 10 秒、默认 30 秒的刷新间隔，对于 SMG 的 failover 需求来说太慢。V4 的 Operator 可以做到 1-2 秒，V5.1 可以做到亚秒级。

**阻碍三：刷新延迟太高，且不支持优先级语义**

Consul/Nacos 可以配置 HTTP 健康检查来探测 SMG 的 `/readiness`，返回 503 时标记为 unhealthy——这在 HTTP 层面是可行的，不需要探针"理解"内部的 P/D worker 语义。但问题在于：

- McpBridge 从注册中心同步状态到 Envoy 的**刷新间隔是 10-30 秒**，远慢于 failover 所需的亚秒级感知
- 注册中心只能做 healthy/unhealthy 的**二元判断**，不能表达"primary vs backup"的优先级语义（这和 V4 Operator 可以灵活配置 failback 策略不同）

**阻碍四：需要额外维护注册中心集群**

引入 Consul（3-5 节点 Raft）或 Nacos（MySQL + 多节点）作为基础设施，增加了运维复杂度和故障域。

### 6.2 如果要实现，需要什么

**路径 A：扩展 Higress McpBridge**
- 修改 McpBridge 源码，支持从注册中心 metadata 读取 `priority` 字段
- 将不同 priority 的 endpoint 分组生成带 `LocalityLbEndpoints.priority` 的 EDS
- 缩短刷新间隔到 1 秒以下（需要验证 McpBridge 性能是否支撑）
- 支持自定义健康检查（调用 SMG `/readiness` 而非通用 TCP ping）

**路径 B：自定义 xDS 控制平面**
- 自建一个轻量 xDS server，替代 McpBridge
- 直接 watch 注册中心（或 K8s API）获取 SMG endpoint
- 执行 SMG-specific 健康检查
- 生成带 priority 的 EDS 直接推送给 Envoy
- 这本质上是用"自定义控制平面"替代了 V4 的 Operator，复杂度更高

**路径 C：使用 Istio Locality Failover**
- 将 primary endpoint 标记为 `locality: primary/zone-a`
- 将 backup endpoint 标记为 `locality: backup/zone-b`
- 使用 `DestinationRule.localityLbSetting.failover`
- **限制**：这是地域级 failover，不是严格的主备。如果 primary 和 backup 在同一机房，这种语义不自然。

### 6.3 与 V3E（McpBridge + Sidecar）的关系

V3 的方案 E（McpBridge + 健康检查 Sidecar）本质上也是基于服务发现，但它引入了 Sidecar 来做 SMG-specific 健康检查，并动态修改注册中心权重。V8 和 V3E 的区别在于：

- V3E 侧重点是"用 Sidecar + 注册中心替代 Operator"
- V8 侧重点是"用 Envoy EDS priority 实现原生 failover"

但两者面临相同的问题：McpBridge 不支持 priority，刷新延迟高。

---

## 七、结论

**服务发现组件（Consul/Nacos）管理 SMG 拓扑与健康状态的方案，在当前 Higress 生态下不可行。**

三个核心原因：

1. **McpBridge 不支持 priority 映射**，无法将注册中心 metadata 转换为 Envoy EDS 的 active-standby 语义
2. **刷新延迟 10-30 秒**，远慢于 failover 所需的亚秒级感知
3. **注册中心同步延迟太高**（10-30 秒），且不支持优先级语义

如果团队已经在使用 Consul/Nacos 作为基础设施，并且愿意投入资源扩展 McpBridge 或自建 xDS 控制平面，这个方向长期来看有价值。但对于 SMG 的 immediate HA 需求，V4（Operator）和 V5.1（WASM 插件）仍然是更务实的选择。

---

## 八、参考来源

- Higress McpBridge 配置文档  
  https://higress.ai/en/docs/latest/user/mcp-bridge/

- Higress Nacos 服务发现源码分析  
  https://caishaodong.pages.dev/api-gateway/higress/79731129-higress-nacos-service-discovery-analysis/

- Envoy 服务发现类型概述  
  https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery

- Envoy Priority Levels  
  https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/priority

- Envoy Locality Weighted LB  
  https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight

- Istio Locality Failover  
  https://istio.io/latest/docs/tasks/traffic-management/locality-load-balancing/failover/

- Istio DestinationRule 参考  
  https://istio.io/latest/docs/reference/config/networking/destination-rule/

- Consul Health Checks  
  https://developer.hashicorp.com/consul/docs/discovery/checks

- Nacos 文档  
  https://nacos.io/en/docs/v2.3/what-is-nacos/

- etcd Leases & Watches  
  https://etcd.io/docs/v3.5/learning/api_guarantees/
