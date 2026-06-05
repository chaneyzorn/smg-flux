# SMG Active-Standby HA 架构 V3 方案调研 — 父子网关方向

> 目标：探索使用 **L7 父子网关** 架构替代 EnvoyFilter，用更高层、更原生的方式实现主备 failover。父网关负责跨 group 轮询，子网关负责 group 内部的主备切换。
> 
> 调研范围包括：引入独立 L7 代理（Nginx/HAProxy）作为子网关、复用 Higress 自身能力作为子网关的各种可行路径。

---

## 一、父子网关架构概述

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         外部客户端请求                                         │
│              api.kimi-router.example.com                                     │
└───────────────────────────┬─────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────────────┐
│                      父网关（Parent Gateway）                                  │
│                         Higress / Envoy                                      │
│                                                                              │
│   职责：                                                                      │
│   - 跨 group 轮询（weight 33/33/34）                                         │
│   - 熔断、失败重试、超时控制                                                  │
│   - TLS 终止、认证、限流                                                      │
│   - 不感知 group 内部主备，只看到"group 入口是否可用"                         │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │ Route Table                                                           │  │
│   │   weighted_clusters:                                                  │  │
│   │     - cluster: child-gateway-1   ←── 33%                             │  │
│   │     - cluster: child-gateway-2   ←── 33%                             │  │
│   │     - cluster: child-gateway-3   ←── 34%                             │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
└─────────────┬───────────────┬────────────────┬──────────────────────────────┘
              │               │                │
              ▼               ▼                ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   子网关-1       │  │   子网关-2       │  │   子网关-3       │
│  (Child GW)     │  │  (Child GW)     │  │  (Child GW)     │
│                 │  │                 │  │                 │
│  职责：          │  │  职责：          │  │  职责：          │
│  - group 内部    │  │  - group 内部    │  │  - group 内部    │
│    主备 failover │  │    主备 failover │  │    主备 failover │
│  - 父网关看到    │  │  - 父网关看到    │  │  - 父网关看到    │
│    的是"黑盒"    │  │    的是"黑盒"    │  │    的是"黑盒"    │
└───────┬─────────┘  └───────┬─────────┘  └───────┬─────────┘
        │                    │                    │
        ▼                    ▼                    ▼
   ┌──────────┐        ┌──────────┐        ┌──────────┐
   │ SMG-1-主 │        │ SMG-2-主 │        │ SMG-3-主 │
   │ SMG-1-备A│        │ SMG-2-备A│        │ SMG-3-备A│
   │ SMG-1-备B│        │ SMG-2-备B│        │ SMG-3-备B│
   └──────────┘        └──────────┘        └──────────┘
```

**核心诉求**：
1. 父网关配置极简（标准 VirtualService + weight），无 EnvoyFilter
2. 子网关封装主备逻辑，对父网关透明
3. 子网关本身也尽量不使用 EnvoyFilter 等底层 hack

---

## 二、方案 A：Nginx 作为子网关

### 2.1 架构

每个 group 前面部署一个 Nginx Pod（或 sidecar），Nginx 配置 `backup` upstream 实现主备。

```
父网关 (Higress)
  │
  ├──→ group-1-nginx-svc:30002
  │       └── Nginx-1 Pod
  │            ├── 主: smg-group-1-0:30002
  │            ├── 备A: smg-group-1-1:30002 (backup)
  │            └── 备B: smg-group-1-2:30002 (backup)
  │
  ├──→ group-2-nginx-svc:30002
  └──→ group-3-nginx-svc:30002
```

### 2.2 Nginx 配置

```nginx
upstream smg_group_1 {
    server smg-group-1-0.headless:30002 weight=100;
    server smg-group-1-1.headless:30002 backup;
    server smg-group-1-2.headless:30002 backup;
    
    # active health check（需要 nginx_upstream_check_module 或 Nginx Plus）
    check interval=2000 rise=1 fall=2 timeout=3000 type=http;
    check_http_send "GET /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx;
}

server {
    listen 30002;
    location / {
        proxy_pass http://smg_group_1;
        proxy_connect_timeout 5s;
        proxy_next_upstream error timeout http_502 http_503;
    }
}
```

### 2.3 Nginx `backup` 行为分析

| 场景 | Nginx 开源版行为 | 是否符合需求 |
|------|-----------------|------------|
| 主健康 | 100% 流量 → 主 | ✅ 符合 |
| 主挂了 | 50% → 备A，50% → 备B（round_robin 在 backup 间分配） | ❌ **不符合** — 多备同时接管，cache_aware 树分散 |
| 主 + 备A 挂了 | 100% → 备B | ✅ 符合 |

**核心问题**：Nginx 开源版的 `backup` 参数不支持**优先级队列**。所有 backup server 在主不可用时**同时激活**，按负载均衡算法分配流量。无法实现"主 → 备A → 备B"的依次降级。

### 2.4 尝试多级 failover

Nginx 开源版没有原生机制支持多级依次 failover。Nginx Plus 商业版支持 `resolve` 和更复杂的 upstream 管理，但仍没有明确的"优先级队列"概念。

可以通过第三方模块如 `nginx_upstream_check_module` + 复杂的 `map` + `if` 规则模拟，但配置晦涩、维护困难，不符合"高层、声明式"的初衷。

**结论**：❌ **不可行**。除非能接受"多备同时接管"的 trade-off，但这会破坏 cache_aware 树的隔离性。

---

## 三、方案 B：HAProxy 作为子网关

### 3.1 架构

与 Nginx 类似，使用 HAProxy 作为子网关。

```
父网关 (Higress)
  │
  ├──→ group-1-haproxy-svc:30002
  │       └── HAProxy-1 Pod
  │            ├── 主: smg-group-1-0:30002
  │            ├── 备A: smg-group-1-1:30002 (backup)
  │            └── 备B: smg-group-1-2:30002 (backup)
```

### 3.2 HAProxy 配置

```haproxy
global
    maxconn 4096

defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    option httpchk GET /health

backend smg_group_1
    default-server check inter 2s rise 1 fall 2
    
    server primary smg-group-1-0:30002
    server backup-1 smg-group-1-1:30002 backup
    server backup-2 smg-group-1-2:30002 backup
```

### 3.3 HAProxy `backup` 行为分析

| 场景 | HAProxy 行为 | 是否符合需求 |
|------|-------------|------------|
| 主健康 | 100% → 主 | ✅ 符合 |
| 主挂了 | 50% → 备A，50% → 备B | ❌ **不符合** — 与 Nginx 相同的问题 |

### 3.4 尝试用 ACL 实现优先级

```haproxy
backend smg_group_1
    option httpchk GET /health
    
    # 尝试使用 ACL 实现优先级路由
    acl primary_up nbsrv(primary) gt 0
    use-server primary if primary_up
    
    acl backup1_up nbsrv(backup-1) gt 0
    use-server backup-1 if backup1_up !primary_up
    
    acl backup2_up nbsrv(backup-2) gt 0
    use-server backup-2 if backup2_up !primary_up !backup1_up
    
    server primary smg-group-1-0:30002 check
    server backup-1 smg-group-1-1:30002 check
    server backup-2 smg-group-1-2:30002 check
```

但 `use-server` 在 HAProxy 中的行为有限制：
- 它只能在请求到达时生效，不适合作为 failover 机制
- `nbsrv()` 计算的是 healthy server 数量，不是按优先级选择
- 实际测试表明，这种配置在 HAProxy 中并不稳定

**结论**：⚠️ **勉强可行但配置复杂**。HAProxy 的 ACL 规则可以实现一定程度的控制，但配置晦涩、调试困难，且需要大量的测试验证。没有明显优于 EnvoyFilter 的优势。

---

## 四、方案 B+：OpenResty / Kong 作为子网关

### 4.1 OpenResty 作为子网关

#### 4.1.1 架构与能力边界

OpenResty（Nginx + LuaJIT）的核心优势在于 `balancer_by_lua_block` 阶段提供了对 upstream 选择的完全编程控制。`ngx.balancer` 模块暴露的 API：

| API | 作用 |
|-----|------|
| `balancer.set_current_peer(host, port)` | 设置当前重试的上游 peer |
| `balancer.set_more_tries(count)` | 设置额外重试次数 |
| `balancer.get_last_failure()` | 获取上次失败原因（`"failed"`、`"next"`、状态码） |

这意味着在 `balancer_by_lua_block` 中可以基于健康状态和优先级顺序选择 endpoint，天然支持多级 failover：

```lua
balancer_by_lua_block {
    local balancer = require "ngx.balancer"
    local peers = {
        {host = "smg-group-1-0.headless", port = 30002, priority = 0},  -- 主
        {host = "smg-group-1-1.headless", port = 30002, priority = 1},  -- 备A
        {host = "smg-group-1-2.headless", port = 30002, priority = 2},  -- 备B
    }
    
    local state = ngx.ctx.tries or 0
    local target = peers[state + 1]
    
    ngx.ctx.tries = state + 1
    balancer.set_more_tries(#peers - state - 1)
    balancer.set_current_peer(target.host, target.port)
}
```

#### 4.1.2 健康检查

OpenResty/nginx OSS **不包含原生主动健康检查**。可选方案：

1. **`lua-resty-healthcheck`**（Kong 出品）：通过 `ngx.timer.*` 做后台探测，`lua_shared_dict` 做跨 worker 状态共享，支持 HTTP/HTTPS/TCP 探测和 healthy/unhealthy 阈值。事件通过 `lua-resty-worker-events` 分发。
2. **`lua-resty-upstream`**（hamishforbes）：**原生支持 priority-based pools**，可定义 `primary`（priority=0）和 `dr`（priority=10）池，高优先级池优先使用，低优先级池作为 failover。

#### 4.1.3 行为分析

| 场景 | OpenResty 行为 | 是否符合需求 |
|------|---------------|------------|
| 主健康 | `balancer` 选择 priority=0 的主节点 | ✅ 符合 |
| 主挂了（第一次重试） | `get_last_failure()` 返回失败，`balancer` 选择 priority=1 的备A | ✅ 符合 |
| 主 + 备A 挂了（第二次重试） | 再次失败，`balancer` 选择 priority=2 的备B | ✅ 符合 |

**结论**：✅ **OpenResty 完全可行**。`balancer_by_lua_block` + `lua-resty-healthcheck`/`lua-resty-upstream` 可以完整实现多级 preemptive failover 和主动健康检查，没有架构层面的 blocker。

---

### 4.2 Kong 作为子网关

#### 4.2.1 Kong 的插件阶段限制

Kong 的插件架构**不暴露 `balancer` 阶段**给自定义插件。官方支持的插件阶段：

| 阶段 | 自定义插件可 hook | 与 upstream 选择相关 |
|------|------------------|---------------------|
| `init_worker` | ✅ | 否 |
| `rewrite` | ✅ | 路由前 |
| `access` | ✅ | 转发前 |
| `header_filter` | ✅ | 响应后 |
| `body_filter` | ✅ | 响应体后 |
| `log` | ✅ | 请求完成后 |
| **`balancer`** | ❌ **未暴露** | **Kong 内部选择 target** |

Kong issue [#6241](https://github.com/Kong/kong/issues/6241) 明确记录了社区请求开放 balancer 阶段的讨论，但截至目前仍未向自定义插件开放。

#### 4.2.2 `access` 阶段的绕过方案

在 `access` 阶段，Kong PDK 提供：

| 函数 | 行为 | 健康检查 | 重试 |
|------|------|---------|------|
| `kong.service.set_upstream(name)` | 使用命名的 Upstream 实体做 LB | ✅ Kong 内置 | ✅ Kong 内置 |
| `kong.service.set_target(host, port)` | **完全绕过 LB**；直接连接 host:port | ❌ **丢失** | ❌ **丢失**（需手动设置） |

**关键限制**：使用 `set_target()` 会**丢失 Kong 内置的主动/被动健康检查**和 ring balancer 的重试逻辑。插件必须在外部维护健康状态（如共享字典或外部缓存），并手动实现重试。

#### 4.2.3 Kong 内置的 priority / failover 能力

Kong v3.12+ 引入了 `failover: true` 的 Target 标志：
> "A Target should be used as a failover (backup) Target in case the other, regular targets are unhealthy... The failover Target is only used when all regular Targets are unhealthy."

但这只支持**单级备份**（regular → failover），不支持多级（primary → backup-A → backup-B）。

Kong 的 AI Proxy Advanced 插件（v3.10+）确实有 `priority` 算法：
> "Routes requests to models based on assigned priority groups... falls back to the next group."

但这是**特定于 LLM 路由**的功能，不适用于通用 upstream。

**结论**：⚠️ **Kong 对于通用流量的多级 failover 不实用**。Balancer 阶段不可 hook，`set_target()` 绕过会丢失健康检查，内置 failover 只支持单级。实现完整的多级 failover 需要在 `access` 阶段重建一套健康感知 + 重试系统，复杂度高。

---

### 4.3 引入新组件生态的代价

OpenResty 在技术上完全可行，但代价明显：

| 维度 | 引入 OpenResty/Kong 作为子网关 |
|------|------------------------------|
| **技术栈** | 新增 Lua 生态（OpenResty/Kong）与现有 Higress/Envoy/K8s 技术栈并行 |
| **配置管理** | Nginx conf / Kong declarative config 与 Istio CRD 两套体系 |
| **监控/日志** | 独立的 metrics 格式、日志格式、告警规则 |
| **版本升级** | OpenResty/Kong 的升级节奏与 Higress 独立，需要分别测试兼容性 |
| **高可用** | 子网关本身需要多副本部署、健康检查、故障恢复 |
| **网络跳数** | 父网关 → 子网关 → SMG，增加一跳延迟 |

**核心问题**：如果用插件实现 failover，为什么不直接用 Higress 自身的插件能力？Higress 的 wasm-go 框架同样支持自定义路由逻辑（`SetUpstreamOverrideHost`、`RegisterTickFunc` 等，详见 V5.1 调研）。引入 OpenResty/Kong 只是换了一个地方写插件，没有解决本质问题，反而增加了组件多样性。

**结论**：⚠️ **OpenResty 可行但不推荐，Kong 不推荐**。OpenResty 的 `balancer_by_lua` 能力足够，但引入新的网关组件生态的代价高于收益。如果团队已经在使用 OpenResty/Kong，可以考虑；否则不如回归 Higress 自身的插件能力（V5.1 方向）。

---

## 五、方案 C：独立 Higress 作为子网关

### 5.1 架构

每个 group 部署一个独立的 Higress Gateway 实例作为子网关，子网关内部使用 EnvoyFilter 配置 Aggregate Cluster。

```
父网关 (Higress Parent)
  │ 轮询 3 个子网关
  ├───────┬────────┬────────┐
  ▼       ▼        ▼
[Higress-1] [Higress-2] [Higress-3]  ← 每个是一个独立的 Higress 实例
  │           │           │
  ▼           ▼           ▼
Aggregate   Aggregate   Aggregate
Cluster     Cluster     Cluster
```

### 5.2 分析

**优势**：
- 父网关配置极简（标准 VirtualService + weight）
- 主备逻辑被封装在子网关内部

**劣势**：
- **子网关内部仍然需要 EnvoyFilter**（Aggregate Cluster + Active Health Check）
- 运维复杂度翻倍（4 个 Higress 实例 vs 1 个）
- 每个子网关需要独立的 EnvoyFilter 配置
- 子网关本身也需要高可用（子网关挂了等于整个 group 挂了）
- 引入了额外的网络跳数（父网关 → 子网关 → SMG）

**结论**：❌ **不可行**。只是将 EnvoyFilter 从父网关移动到了子网关，没有解决根本问题，反而增加了复杂度和延迟。

---

## 五、方案 D：复用 Higress 作为子网关（WASM / Go 插件）

> **本方案已独立为 V5 进行深度调研**，详见 `../ha_design_v5/active_standby_ha_design_v5_research.draft.md`。
> 以下为 V3 调研的精简结论。

### 5.1 思路

不引入新的代理组件，利用 Higress 自身的插件能力实现子网关的主备逻辑。

### 6.2 核心发现

经过对 proxy-wasm ABI、Higress ai-proxy 源码（failover.go、retry.go）以及社区 issue 的深入分析：

1. **WASM 无法修改 Router 的 upstream/cluster 选择决策**
   - `set_http_request_header` 修改的 header 对 Router 不可见（proxy-wasm/spec#16）
   - `setProperty` 自动前缀 `wasm.`，无法设置 `envoy.router.cluster`（envoy#28673）

2. **ai-proxy 的 failover 是被动式的**
   - 在同一 cluster 内换 apiToken，不是跨 cluster 切换
   - 重试用 `dispatch_http_call` 实现，成功后 `SendHttpResponse` 直接返回
   - 存在 body 丢失风险（issue #3531）

3. **主动健康检查依赖 Higress 扩展**
   - `RegisterTickFunc` 是 Higress 扩展，非标准 proxy-wasm
   - 只能探测，无法在请求到达时 preemptive 切换 cluster

### 6.3 结论

**WASM/Go 插件无法单独实现 SMG 主备 preemptive failover。**

- 被动 failover 在 SSE 流式场景下不可接受
- 如需完整技术分析，参阅 V5 调研文档

---

## 七、方案 E：复用 Higress 作为子网关（McpBridge + 健康检查 Sidecar）

### 7.1 思路

不引入新的代理组件，而是引入一个**轻量的健康检查 sidecar**，与 Higress 的 McpBridge 配合：

```
┌─────────────────────────────────────────────────────────────┐
│                    Higress Parent Gateway                    │
│                                                              │
│  McpBridge 配置：从本地 Nacos/Consul 读取端点               │
│                                                              │
│  VirtualService: weight 33/33/34 -> 3 个 service            │
│  （每个 service 只从注册中心获取当前活跃端点）                │
└─────────────────────────────┬───────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Nacos    │   │ Nacos    │   │ Nacos    │
        │ service-1│   │ service-2│   │ service-3│
        │ [主]     │   │ [主]     │   │ [主]     │  ← 正常时
        │          │   │          │   │          │
        │ [备A]    │   │ [备A]    │   │ [备A]    │  ← 主挂了时
        └────┬─────┘   └────┬─────┘   └────┬─────┘
             │              │              │
             ▼              ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Health   │   │ Health   │   │ Health   │
        │ Checker  │   │ Checker  │   │ Checker  │
        │ Sidecar  │   │ Sidecar  │   │ Sidecar  │
        │          │   │          │   │          │
        │ 1. HTTP  │   │ 1. HTTP  │   │ 1. HTTP  │
        │    GET   │   │    GET   │   │    GET   │
        │    /health│  │    /health│  │    /health│
        │ 2. 更新   │   │ 2. 更新   │   │ 2. 更新   │
        │    Nacos │   │    Nacos │   │    Nacos │
        │    权重   │   │    权重   │   │    权重   │
        └──────────┘   └──────────┘   └──────────┘
```

### 7.2 核心机制

1. **健康检查 Sidecar**：持续 HTTP 探测 SMG 实例的 `/health`
2. **注册中心**（Nacos/Consul/etcd）：存储端点列表和权重
3. **Higress McpBridge**：从注册中心读取端点，只获取活跃端点

当主不健康时：
- Sidecar 更新 Nacos：主的权重设为 0，备A 的权重设为 100
- Higress McpBridge 感知变化，流量切换到备A

### 7.3 分析

**优势**：
- 父网关使用标准 Higress McpBridge，无需 EnvoyFilter
- 子网关逻辑外移到 Sidecar + 注册中心

**劣势**：
- 引入了**新的依赖**（Nacos/Consul/etcd），增加了系统复杂度
- 需要维护注册中心的高可用
- Higress 开源版的 McpBridge **动态权重调整能力有限**，需要验证
- 健康检查 Sidecar 本身需要高可用
- 本质上和 V4（Operator + EndpointSlice）类似，只是用注册中心替代了 K8s API

**结论**：⚠️ **可行但复杂度不低**。这个方案需要引入注册中心，系统组件增加，维护成本上升。如果团队已经在使用 Nacos/Consul，可以考虑；否则不如直接用 V4 的 Operator + EndpointSlice（无需额外中间件）。

---

## 八、方案 F：复用 Higress 作为子网关（AI Gateway 路由能力）

### 8.1 思路

Higress 的 AI Gateway 支持"多模型 failover"，如：
- 先尝试 GPT-4，失败则 fallback 到 Claude
- 这是模型级的 failover，本质上就是主备切换

如果可以把 SMG 的主备包装成"同一个模型的不同后端"，利用 AI Gateway 的 failover 能力：

```yaml
# Higress AI Gateway 配置（概念性）
aiRoute:
  model: llama3
  upstreams:
    - name: primary
      url: http://smg-group-1-0:30002
      weight: 100
    - name: backup-1
      url: http://smg-group-1-1:30002
      weight: 0
      fallback: true
    - name: backup-2
      url: http://smg-group-1-2:30002
      weight: 0
      fallback: true
```

### 8.2 分析

**问题**：
1. Higress **开源版**的 AI Gateway **不支持**这种多后端 failover 配置
2. 从 Higress issue #1816 的日志来看，AI Gateway 的 failover 是基于 HTTP 响应码的被动检测，且是商业版能力
3. 即使商业版支持，这也是特定于 LLM 路由的功能，不够通用
4. 需要购买 Higress 商业授权

**结论**：❌ **开源版不可用**。

---

## 八、综合对比

| 方案 | 子网关组件 | 是否避免 EnvoyFilter | 多级 failover | 切换延迟 | 额外组件 | 生产就绪度 |
|------|-----------|-------------------|--------------|---------|---------|-----------|
| **A. Nginx 子网关** | Nginx (开源) | ✅ | ❌（多备同时接管） | ~2-4s | 3 Nginx Pod | 高 |
| **B. HAProxy 子网关** | HAProxy | ✅ | ⚠️（需 ACL） | ~2-4s | 3 HAProxy Pod | 中 |
| **B+. OpenResty 子网关** | OpenResty | ✅ | ✅ | ~2-4s | 3 OpenResty Pod | 高 |
| **B+. Kong 子网关** | Kong | ✅ | ⚠️（需绕过 balancer） | ~2-4s | 3 Kong Pod | 低 |
| **C. 独立 Higress 子网关** | Higress | ❌ | ✅ | ~2-4s | 3 Higress 实例 | 低 |
| **D. WASM/Go 插件** | Higress 自身 | ❌ | ❌（无法 preemptive） | - | 无 | 低（不可行） | 详见 V5 |
| **E. McpBridge + Sidecar** | Higress + Nacos | ✅ | ⚠️（需验证） | ~3-5s | 3 Sidecar + Nacos | 中 |
| **F. AI Gateway** | Higress 商业版 | 商业版 | ✅ | ~2-4s | 无 | 高（商业版） |

---

## 九、关键结论

### 9.1 父子网关方向的核心发现

经过对 6 个子方案的深入调研，我们发现：

1. **Nginx/HAProxy 开源版**不支持"多级依次 failover"（主 → 备A → 备B），它们的 `backup` 参数在主挂了之后会**同时启用所有备**
2. **OpenResty**通过 `balancer_by_lua_block` + `lua-resty-healthcheck` 可以完整实现多级 failover，技术可行，但会**引入新的网关组件生态**（Lua 技术栈、独立运维）
3. **Kong**的 balancer 阶段不向自定义插件开放，内置 failover 只支持单级备份，通用多级 failover **不实用**
4. **复用 Higress 自身能力**（WASM/Go 插件、AI Gateway）经 V5 深入验证，WASM 无法修改 Router 的 upstream 选择，开源版能力有限
5. **独立 Higress 子网关**仍然需要 EnvoyFilter，没有解决根本问题
6. **McpBridge + Sidecar** 可行，但引入了注册中心依赖，复杂度不低于 V4 Operator

### 9.2 为什么父子网关方向难以实现

根本原因是：**现有的主流 L7 代理（Nginx、HAProxy、Envoy 开源配置）中，只有 Envoy 的 Aggregate Cluster 和 OpenResty 的 `balancer_by_lua` 原生支持按优先级依次 failover。Envoy 的能力在 Istio/Higress 中没有高层 CRD 暴露；OpenResty 的能力需要引入 Lua 技术栈和独立运维，代价不低。**

如果要避免 EnvoyFilter，就需要：
- 引入新的代理组件（Nginx/HAProxy）→ 但它们开源版不支持
- 或引入额外的控制组件（Sidecar + 注册中心 / Operator）→ 这本质上是 V4 方向

### 9.3 V3 的启示

V3 的调研虽然没有找到完美的"父子网关"方案，但验证了以下结论：

- **Envoy 的 aggregate cluster 是目前唯一原生支持多级 failover 的 L7 代理**
- 任何试图绕过 EnvoyFilter 的方案，要么需要**接受功能降级**（如多备同时接管），要么需要**引入额外的控制组件**
- 如果必须引入额外组件，**直接用标准 K8s API（EndpointSlice）比引入注册中心或新代理更简洁**

这正是 V4（Operator + EndpointSlice）方案的出发点。

---

## 十、从 V3 到 V4 的演进逻辑

```
V3 调研结论：
  ├── 引入独立 L7 代理（Nginx/HAProxy/OpenResty/Kong）
  │     ├── Nginx/HAProxy 开源版不支持多级 failover ❌
  │     ├── OpenResty 可行但引入新组件生态 ⚠️
  │     └── Kong 通用多级 failover 不实用 ❌
  │
  ├── 复用 Higress 自身能力（插件/AI Gateway）
  │     └── 开源版能力有限 ❌
  │
  ├── 复用 Higress + McpBridge + Sidecar
  │     └── 可行但引入注册中心，复杂 ⚠️
  │
  └── 启示：需要额外的控制组件来实现主备逻辑
        └── V4 方案：用标准 K8s API（EndpointSlice）+ Operator
              └── 无需注册中心，无需新代理，纯 K8s 原生 ✅
```

---

## 十一、附录：Nginx backup 参数的行为验证

如果团队对 Nginx 的 backup 行为有疑问，可以用以下配置快速验证：

```nginx
# 验证配置
events {
    worker_connections 1024;
}

http {
    upstream test_backend {
        server 127.0.0.1:8081 weight=100;  # 主（模拟 SMG 主）
        server 127.0.0.1:8082 backup;      # 备A
        server 127.0.0.1:8083 backup;      # 备B
    }

    server {
        listen 8080;
        location / {
            proxy_pass http://test_backend;
        }
    }

    # 模拟后端
    server { listen 8081; location / { return 200 "primary\n"; } }
    server { listen 8082; location / { return 200 "backup-1\n"; } }
    server { listen 8083; location / { return 200 "backup-2\n"; } }
}
```

**测试步骤**：

```bash
# 1. 启动 Nginx
nginx -c /path/to/test.conf

# 2. 正常情况：只返回 primary
curl http://localhost:8080/
# => primary

# 3. 停止主节点（8081）
# 模拟：fuser -k 8081/tcp

# 4. 主挂了后：同时返回 backup-1 和 backup-2（轮询）
curl http://localhost:8080/
# => backup-1
curl http://localhost:8080/
# => backup-2
curl http://localhost:8080/
# => backup-1
```

**结论验证**：Nginx 的 `backup` 在主不可用时，**所有 backup server 同时激活**，按负载均衡算法分配流量。无法实现依次降级。

---

## 十二、附录：参考来源

- Nginx HTTP Upstream Module — `backup` 参数官方文档  
  https://nginx.org/en/docs/http/ngx_http_upstream_module.html#backup

- HAProxy Configuration Manual — `backup` 与 `use-server` 说明  
  https://docs.haproxy.org/2.8/configuration.html

- OpenResty `ngx.balancer` API  
  https://github.com/openresty/lua-resty-core/blob/master/lib/ngx/balancer.md

- `lua-resty-healthcheck` — OpenResty 主动健康检查库  
  https://kong.github.io/lua-resty-healthcheck/

- `lua-resty-upstream` — 支持 priority-based pools 的 upstream 管理库  
  https://opm.openresty.org/package/hamishforbes/lua-resty-upstream/

- Kong Plugin Phases — 官方插件开发阶段说明  
  https://developer.konghq.com/gateway/entities/plugin/

- Kong issue #6241 — balancer 阶段未向自定义插件开放  
  https://github.com/Kong/kong/issues/6241

- Kong `kong.service.set_target()` PDK 文档  
  https://developer.konghq.com/gateway/pdk/reference/kong.service/

- Kong Upstream Targets — `failover` 标志说明（v3.12+）  
  https://developer.konghq.com/gateway/entities/upstream/

- Envoy Aggregate Cluster 官方文档  
  https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/aggregate_cluster

- Istio EnvoyFilter API 参考  
  https://istio.io/latest/docs/reference/config/networking/envoy-filter/

- Higress WASM 插件开发指南  
  https://higress.ai/en/docs/latest/user/wasm-go/

- Higress AI Gateway 路由能力（商业版）  
  https://higress.ai/en/docs/latest/user/ai-gateway/

- Higress McpBridge 服务来源配置  
  https://higress.ai/en/docs/latest/user/mcp-bridge/

- Istio DestinationRule — `outlierDetection` 与熔断配置  
  https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection

- Kubernetes EndpointSlice API  
  https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/

- Istio Issue #39445 — DestinationRule `failover` 字段进展（仍 open）  
  https://github.com/istio/istio/issues/39445
