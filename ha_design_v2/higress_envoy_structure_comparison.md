# 多 Aggregate Cluster 场景：有无 EnvoyFilter 的结构对比

> 3 个 SMG group，每组 1 主 + 2 备。清晰对比"默认状态"和"V2-B 方案"在 Higress/Envoy 内部的结构差异。

---

## 一、没有 EnvoyFilter（Istio 自动生成）

### 1.1 控制面配置（用户写的 YAML）

```yaml
# VirtualService：Higress 轮询 3 个 group
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: smg-router-vs
spec:
  hosts: [kimi-router-ha-cluster.local]
  gateways: [kimi-router-gateway]
  http:
    - route:
        - destination: { host: smg-group-1.local, port: { number: 30002 } }
          weight: 33
        - destination: { host: smg-group-2.local, port: { number: 30002 } }
          weight: 33
        - destination: { host: smg-group-3.local, port: { number: 30002 } }
          weight: 34
      retries:
        attempts: 2
        perTryTimeout: 30s
        retryOn: gateway-error,connect-failure

---
# ServiceEntry：注册每个 group 的端点
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: smg-group-1
spec:
  hosts: [smg-group-1.local]
  resolution: DNS
  endpoints:
    - address: smg-group-1-0.headless.default.svc.cluster.local
    - address: smg-group-1-1.headless.default.svc.cluster.local
    - address: smg-group-1-2.headless.default.svc.cluster.local

# ServiceEntry smg-group-2、smg-group-3 类似...
```

### 1.2 Envoy 内部结构（Istiod 自动生成）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Envoy 数据面（无 EnvoyFilter）                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Route Table                                                           │  │
│  │                                                                       │  │
│  │   match: prefix /                                                     │  │
│  │   weighted_clusters:                                                  │  │
│  │     - cluster: outbound|30002||smg-group-1.local   <-- 33% weight    │  │
│  │     - cluster: outbound|30002||smg-group-2.local   <-- 33% weight    │  │
│  │     - cluster: outbound|30002||smg-group-3.local   <-- 34% weight    │  │
│  │                                                                       │  │
│  │   retry_policy: { attempts: 2, ... }  <-- 来自 VirtualService        │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Clusters (共 3 个，Istio 自动生成)                                     │  │
│  │                                                                       │  │
│  │ ┌─────────────────────────────────────────┐                          │  │
│  │ │ Cluster: outbound|30002||smg-group-1.local                          │  │
│  │ │ type: STRICT_DNS                                                      │  │
│  │ │ lb_policy: ROUND_ROBIN   <-- 3 个端点平等轮询                        │  │
│  │ │                                                                       │  │
│  │ │ load_assignment:                                                      │  │
│  │ │   endpoints:                                                          │  │
│  │ │     - smg-group-1-0.headless:30002  <-- 主（活跃）                   │  │
│  │ │     - smg-group-1-1.headless:30002  <-- 备A（活跃）                  │  │
│  │ │     - smg-group-1-2.headless:30002  <-- 备B（活跃）                  │  │
│  │ └─────────────────────────────────────────┘                          │  │
│  │                                                                       │  │
│  │ ┌─────────────────────────────────────────┐                          │  │
│  │ │ Cluster: outbound|30002||smg-group-2.local                          │  │
│  │ │ type: STRICT_DNS                                                      │  │
│  │ │ lb_policy: ROUND_ROBIN                                                │  │
│  │ │   endpoints: smg-2-0, smg-2-1, smg-2-2                                │  │
│  │ └─────────────────────────────────────────┘                          │  │
│  │                                                                       │  │
│  │ ┌─────────────────────────────────────────┐                          │  │
│  │ │ Cluster: outbound|30002||smg-group-3.local                          │  │
│  │ │ type: STRICT_DNS                                                      │  │
│  │ │ lb_policy: ROUND_ROBIN                                                │  │
│  │ │   endpoints: smg-3-0, smg-3-1, smg-3-2                                │  │
│  │ └─────────────────────────────────────────┘                          │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 流量分布结果                                                          │  │
│  │                                                                       │  │
│  │  33% --> group-1 cluster --> ROUND_ROBIN --> 主/备A/备B 各 ~11%       │  │
│  │  33% --> group-2 cluster --> ROUND_ROBIN --> 主/备A/备B 各 ~11%       │  │
│  │  34% --> group-3 cluster --> ROUND_ROBIN --> 主/备A/备B 各 ~11.3%     │  │
│  │                                                                       │  │
│  │  结果：9 个 Pod 平等接收流量，cache_aware 树分散到 9 个实例            │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 问题总结（无 EnvoyFilter）

| 问题 | 根因 |
|------|------|
| cache_aware 树分散到 9 个实例 | 3 个 cluster x 3 个端点，全部平权轮询 |
| 主备无差别 | ROUND_ROBIN 不区分主备角色 |
| 故障检测靠 K8s readiness probe | L4 探测，6-9s 延迟 |
| 不支持 1主N备的 failover | 所有端点平等，没有优先级降级机制 |

---

## 二、有 EnvoyFilter（V2-B 方案）

### 2.1 控制面配置变化

用户写的 YAML **新增**了：

```yaml
# 新增：EnvoyFilter（3 组 x 1 个 = 3 个 EnvoyFilter）
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: smg-group-1-aggregate
spec:
  configPatches:
    # patch 1-3: ADD 3 个子 cluster（primary / backup-1 / backup-2）
    # patch 4: ADD 1 个 aggregate cluster
    # patch 5: MERGE route，将 cluster 指向 aggregate
```

VirtualService 和 ServiceEntry **保持不变**。

### 2.2 Envoy 内部结构（有 EnvoyFilter）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Envoy 数据面（有 EnvoyFilter，V2-B）                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Route Table                                                           │  │
│  │                                                                       │  │
│  │   match: prefix /                                                     │  │
│  │   weighted_clusters:                                                  │  │
│  │     - cluster: smg-group-1-aggregate  <-- 33% weight  【已 patch】     │  │
│  │     - cluster: smg-group-2-aggregate  <-- 33% weight  【已 patch】     │  │
│  │     - cluster: smg-group-3-aggregate  <-- 34% weight  【已 patch】     │  │
│  │                                                                       │  │
│  │   retry_policy: { attempts: 2, ... }  <-- 仍来自 VirtualService       │  │
│  │                                                                       │  │
│  │   关键变化：原来指向 Istio 自动生成的 cluster，                       │  │
│  │             现在指向 EnvoyFilter ADD 的 aggregate cluster              │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Clusters (共 12 个)                                                    │  │
│  │                                                                       │  │
│  │ ┌─────────────────────────────────────────────────────────────────┐   │  │
│  │ │ 原有 3 个（Istio 自动生成，仍保留但不再被 route 引用）            │   │  │
│  │ │                                                                 │   │  │
│  │ │   outbound|30002||smg-group-1.local  <-- 保留，但无流量         │   │  │
│  │ │   outbound|30002||smg-group-2.local  <-- 保留，但无流量         │   │  │
│  │ │   outbound|30002||smg-group-3.local  <-- 保留，但无流量         │   │  │
│  │ │                                                                 │   │  │
│  │ └─────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  │ ┌─────────────────────────────────────────────────────────────────┐   │  │
│  │ │ 新增 9 个（EnvoyFilter ADD，实际承载流量）                       │   │  │
│  │ │                                                                 │   │  │
│  │ │  smg-group-1-aggregate  <-- aggregate cluster                   │   │  │
│  │ │    ├── smg-group-1-primary    <-- active HC /health 2s          │   │  │
│  │ │    │      └── endpoint: smg-1-0 (主)                            │   │  │
│  │ │    ├── smg-group-1-backup-1   <-- active HC /health 2s          │   │  │
│  │ │    │      └── endpoint: smg-1-1 (备A)                           │   │  │
│  │ │    └── smg-group-1-backup-2   <-- active HC /health 2s          │   │  │
│  │ │           └── endpoint: smg-1-2 (备B)                           │   │  │
│  │ │                                                                 │   │  │
│  │ │  smg-group-2-aggregate  <-- aggregate cluster                   │   │  │
│  │ │    ├── smg-group-2-primary                                      │   │  │
│  │ │    ├── smg-group-2-backup-1                                     │   │  │
│  │ │    └── smg-group-2-backup-2                                     │   │  │
│  │ │                                                                 │   │  │
│  │ │  smg-group-3-aggregate  <-- aggregate cluster                   │   │  │
│  │ │    ├── smg-group-3-primary                                      │   │  │
│  │ │    ├── smg-group-3-backup-1                                     │   │  │
│  │ │    └── smg-group-3-backup-2                                     │   │  │
│  │ │                                                                 │   │  │
│  │ └─────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 流量分布结果（正常状态）                                              │  │
│  │                                                                       │  │
│  │  33% --> group-1-aggregate --> primary --> smg-1-0 (主) 生效          │  │
│  │  33% --> group-2-aggregate --> primary --> smg-2-0 (主) 生效          │  │
│  │  34% --> group-3-aggregate --> primary --> smg-3-0 (主) 生效          │  │
│  │                                                                       │  │
│  │  结果：3 个主节点各接收 ~33% 流量，cache_aware 树只分布在 3 个主上   │  │
│  │        6 个备节点不接收流量，各自维护独立的空树（随时准备接管）      │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                              │                                              │
│                              ▼                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ 流量分布结果（SMG-1-主 挂了）                                         │  │
│  │                                                                       │  │
│  │  33% --> group-1-aggregate --> primary UNHEALTHY 失效                │  │
│  │                    └── 自动降级 --> backup-1 --> smg-1-1 (备A) 生效   │  │
│  │  33% --> group-2-aggregate --> primary --> smg-2-0 (主) 生效          │  │
│  │  34% --> group-3-aggregate --> primary --> smg-3-0 (主) 生效          │  │
│  │                                                                       │  │
│  │  结果：Higress 仍然轮询 3 个 aggregate cluster                       │  │
│  │        group-1 内部自动 failover 到备A                               │  │
│  │        group-2/group-3 不受影响                                       │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 关键变化点

| 维度 | 无 EnvoyFilter | 有 EnvoyFilter（V2-B） |
|------|--------------|----------------------|
| **Cluster 总数** | 3 个（Istio 自动生成） | **12 个**（3 个保留 + 9 个新增） |
| **Route 指向** | outbound|30002||smg-group-*.local | **smg-group-*-aggregate** |
| **每组内部策略** | ROUND_ROBIN（3 端点平权） | **Aggregate Cluster（按优先级 failover）** |
| **故障检测方式** | K8s readiness probe（L4，~6-9s） | **Envoy active health check（L7，~2-4s）** |
| **主备是否区分** | 不区分，全部平等 | 区分，primary 优先，backup 降级 |
| **多备支持** | 不支持，所有备同时接收流量 | 支持，只有一个备在故障时接管 |
| **cache_aware 树分布** | 分散到 9 个实例 | 只分布在 3 个活跃实例（正常时 3 个主） |

---

## 三、EnvoyFilter 具体做了什么（逐条拆解）

以 smg-group-1 为例，EnvoyFilter 的 5 个 patch 分别做了什么：

### Patch 1-3：ADD 子 cluster（创建新资源）

```yaml
- applyTo: CLUSTER
  patch:
    operation: ADD
    value:
      name: smg-group-1-primary      # <-- 新名称，不与自动生成 cluster 冲突
      type: STRICT_DNS
      health_checks: [ ... ]          # <-- L7 active health check
      load_assignment:
        endpoints: [smg-1-0.fqdn]     # <-- 只有主节点
```

- smg-group-1-backup-1 同理，endpoint 只有 smg-1-1
- smg-group-1-backup-2 同理，endpoint 只有 smg-1-2

**语义**：在 Envoy 的 cluster 列表中**新增** 3 个独立 cluster。由于名称与 Istio 自动生成的 outbound|30002||smg-group-1.local 不同，**完全安全，无冲突**。

### Patch 4：ADD aggregate cluster（绑定子 cluster）

```yaml
- applyTo: CLUSTER
  patch:
    operation: ADD
    value:
      name: smg-group-1-aggregate
      cluster_type:
        name: envoy.clusters.aggregate
        typed_config:
          clusters:
            - smg-group-1-primary
            - smg-group-1-backup-1
            - smg-group-1-backup-2
```

**语义**：新增一个特殊的 aggregate cluster，它不直接连接端点，而是**按顺序委托给子 cluster**。当 primary 不健康时，自动降级到 backup-1；当 backup-1 也不健康时，降级到 backup-2。

### Patch 5：MERGE route（修正后的精确 patch）

```yaml
- applyTo: HTTP_ROUTE
  match:
    routeConfiguration:
      vhost:
        name: smg-group-1.local:30002
    route:
      action: ROUTE
  patch:
    operation: MERGE
    value:
      route:
        cluster: smg-group-1-aggregate   # <-- 只覆盖 cluster 字段
```

**语义**：
- applyTo: HTTP_ROUTE：匹配到**单个 route**（而不是整个 route list）
- match.routeConfiguration.vhost.name：精确匹配属于 smg-group-1.local 的 virtual host
- match.route.action: ROUTE：只匹配有 route action 的 route（排除 redirect、rewrite 等）
- operation: MERGE + value.route.cluster：对 route 这个 message 字段做 **proto merge**，只覆盖 cluster 子字段
- 结果：route.cluster 从 outbound|30002||smg-group-1.local 变成 smg-group-1-aggregate，其他字段（timeout、retry_policy）完全保留

**为什么不能用 ROUTE_CONFIGURATION + MERGE + routes？**

因为 routes 是 **repeated 字段**，proto merge 语义对 repeated 字段是**追加**，不是覆盖。这会导致：

```
# 错误的 patch 结果
routes:
  - match: { prefix: / }
    route:
      cluster: outbound|30002||smg-group-1.local   # <-- 原始 route，先匹配到
  - match: { prefix: / }
    route:
      cluster: smg-group-1-aggregate                 # <-- patch 追加，永远不会匹配
```

Envoy 路由匹配**从上到下，第一条命中就停止**。所以原始 route 仍然生效，新 route 被忽略。

---

## 四、Cluster 命名空间全景图

```
Envoy Cluster 命名空间
│
├── Istio 自动生成（保留但不再被引用）
│   ├── outbound|30002||smg-group-1.local     <-- 原 group-1 流量目标
│   ├── outbound|30002||smg-group-2.local     <-- 原 group-2 流量目标
│   └── outbound|30002||smg-group-3.local     <-- 原 group-3 流量目标
│
└── EnvoyFilter ADD（新增，实际承载流量）
    │
    ├── smg-group-1-aggregate                  <-- route 指向这里
    │   ├── smg-group-1-primary                <-- aggregate 内部引用
    │   ├── smg-group-1-backup-1               <-- aggregate 内部引用
    │   └── smg-group-1-backup-2               <-- aggregate 内部引用
    │
    ├── smg-group-2-aggregate
    │   ├── smg-group-2-primary
    │   ├── smg-group-2-backup-1
    │   └── smg-group-2-backup-2
    │
    └── smg-group-3-aggregate
        ├── smg-group-3-primary
        ├── smg-group-3-backup-1
        └── smg-group-3-backup-2
```

---

## 五、验证 patch 是否生效的命令

```bash
# 1. 查看所有 cluster，确认新增 9 个自定义 cluster
kubectl exec -it deploy/higress-gateway -n higress-system -c istio-proxy -- \
  curl -s localhost:15000/clusters | grep smg-group

# 2. 查看 route 配置，确认 cluster 已指向 aggregate
kubectl exec -it deploy/higress-gateway -n higress-system -c istio-proxy -- \
  curl -s localhost:15000/config_dump | jq '
    .configs[] |
    select(.["@type"] == "type.googleapis.com/envoy.admin.v3.RoutesConfigDump") |
    .dynamic_route_configs[].route_config.virtual_hosts[] |
    select(.name | contains("smg-group-1")) |
    .routes[].route.cluster'

# 3. 查看 aggregate cluster 的健康状态
kubectl exec -it deploy/higress-gateway -n higress-system -c istio-proxy -- \
  curl -s localhost:15000/clusters | grep -E "smg-group-1-(primary|backup|aggregate)"
```

---

## 六、一句话总结

**没有 EnvoyFilter**：Istio 生成 3 个 cluster，每组 3 个端点平权轮询，无主备概念，cache_aware 树分散到 9 个实例。

**有 EnvoyFilter**：EnvoyFilter 新增 9 个 cluster（3 个 aggregate + 6 个子 cluster），通过精确 patch 将 route 指向 aggregate cluster。每组内部按优先级 failover，同一时间只有 1 个实例接收流量，cache_aware 树只分布在 3 个活跃实例上。Istio 自动生成的 3 个 cluster 仍保留在内存中，但不再被 route 引用，无实际流量。
