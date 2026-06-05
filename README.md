# SMG 高可用方案探索笔记

这里记录了我们（我和 Kimi2.6）在给 sgl-model-gateway（SMG）做高可用设计时，从 V0 到 V6 逐步探索、不断调整思路的过程。每个目录对应一个阶段的思考，下面简述每个方案提出时的动机，以及方案演进中的权衡和取舍。

涉及的开源组件：

- <https://github.com/higress-group/higress>
- <https://github.com/sgl-project/sglang/tree/main/sgl-model-gateway>

> **注意**：调研方案由 Kimi2.6 协助撰写，其中存在未经实证的细节，需仔细斟查。

---

## V0：给定的起点

V0 是一张已经存在的架构蓝图，描述了一个大规模 PD 分离的推理服务部署方式：

```
                         外部请求
                            │
                            ▼
                   ┌─────────────────┐
                   │  Higress Gateway │
                   │  (外层轮询/weight)│
                   └────────┬────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
   │ Group-1     │   │ Group-2     │   │ Group-3     │
   │ (SMG 分片)  │   │ (SMG 分片)  │   │ (SMG 分片)  │
   │             │   │             │   │             │
   │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │
   │ │SMG-1.1  │ │   │ │SMG-2.1  │ │   │ │SMG-3.1  │ │
   │ │SMG-1.2  │ │   │ │SMG-2.2  │ │   │ │SMG-3.2  │ │
   │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │
   │     │       │   │     │       │   │     │       │
   │  ┌──┴──┐    │   │  ┌──┴──┐    │   │  ┌──┴──┐    │
   │  ▼     ▼    │   │  ▼     ▼    │   │  ▼     ▼    │
   │ Prefill Decode│   │ Prefill Decode│   │ Prefill Decode│
   └─────────────┘   └─────────────┘   └─────────────┘
```

Higress 做入口，下面挂多个独立的 SMG 分片，每个分片内部有自己的 P/D worker。图上 SMG-1 标注了"主/备"，但这个标注更多是示意性的——具体主备如何切换、切换过程中 cache tree 的状态如何保证一致，都没有被展开。

这张蓝图成为后续探索的出发点。一个未被回答的细节是：多个 SMG 实例同时接收请求时，各自的 cache_aware 前缀树会记录不同的历史，这个分散问题如何解决？当时的思路是引入主备——让一个组里同时只存在一个活跃实例，这样树就集中在一处。主备切换的实现涉及探测、选主、状态同步等多个具体问题。

→ [V0 调研方案](ha_design_v0/pd_disaggregated_architecture.md)

---

## V1：尝试用 K8s 做主备

注意到 cache tree 分散的问题后，第一个直觉是：如果能让一个 SMG 组里任何时候只有一个实例在干活，树就不会分散了。

K8s 现成的 readiness probe 可以被用于——备节点探测主节点，主挂了备就变成 ready，K8s Service 自动把流量切过去。不需要新增组件或代码，通过纯配置即可实现。

但这里有两个限制：

一是 readiness probe 的结果通过 L4（K8s Service / Endpoints）切换生效，周期乘以失败阈值要 6-9 秒，这期间 Higress 可能还在往已经挂掉的主节点发请求。探测和切换分属两个层面，存在感知断层。

二是这个机制隐含了"一个组里只有一个备"的假设。如果放两个备，两个备同时探测到主挂了、同时变成 ready，那请求又会分散到两个备上，树分散问题重现。我们想扩展到一主多备的时候，这个假设就限制了扩展性。

V1 说明 readiness probe 可以实现基本的流量切换，但也暴露了两个限制：延迟和扩展性。

→ [V1 调研方案](ha_design_v1/active_standby_ha_design_v1.draft.md)

---

## V2：能不能让 L7 自己直接做主备决策？

V1 的问题是探测结果通过 L4（K8s Service / Endpoints）切换生效，存在延迟，L7（Envoy/Higress）无法第一时间感知后端状态变化。那如果让 L7（Envoy/Higress）自己直接感知后端状态、自己决定往哪发，就可以绕过 K8s Service 这层。

调研发现 Envoy 的 aggregate cluster 确实能做这件事——可以配多级 fallback，主 cluster 挂了自动降到备 cluster，而且 active health check 的探测周期短于 kubelet 的 readiness probe。一主多备的依次降级也能支持。

但 aggregate cluster 需要通过 EnvoyFilter 往 Envoy 里注入底层 protobuf 配置。这不是 Higress 或 Istio 的标准用法，更像是绕过它们直接操纵 Envoy 内部。配置本身是复杂且底层的——需要直接写 Envoy 的 `ClusterLoadAssignment` protobuf，配置任何路由细节（如超时、重试、header 操作）都要在 protobuf 层面完成，缺乏 VirtualService 那种声明式的简洁性。而且，aggregate cluster 提供的 failover 是粗粒度的 cluster 级切换，无法做到 endpoint 级的精细控制，比如根据自定义健康指标决定切换到哪个具体节点、或者按权重逐步切流。

这会带来几方面的负担：Istio 升级时这些配置可能被覆盖；团队需要熟悉 Envoy 的 xDS 语义才能排查问题；后续想调整路由策略时，改动成本远高于标准 CRD 的方式。

V2 解决了 V1 的技术瓶颈，但引入了配置复杂、底层非标准、缺乏精细控制的问题。这促使我们去寻找更上层、更标准的实现方式。

→ [V2 调研方案](ha_design_v2/active_standby_ha_design_v2.draft.md)

---

## V3：加个中间层行不行？父子网关的尝试

V2 的 EnvoyFilter 需要直接操作 Envoy 的底层配置，与 Istio 的抽象层级脱节。我们设想了一种分层结构：父网关做简单的跨组轮询（标准 VirtualService + weight），每个组前面放一个子网关，子网关负责组内部的主备切换。父网关看到的只是一个"黑盒"入口，不需要知道里面有几主几备。

这个思路在逻辑上清晰，但调研后发现现有代理不具备所需的 failover 语义。Nginx 有 backup upstream，但做不了多级依次 failover；HAProxy 的 backup 也类似；Higress/Envoy 自己同样没有"先尝试 A，A 不行再依次尝试 B、C、D"这种语义。

OpenResty 和 Kong 是例外。OpenResty 的 `balancer_by_lua_block` 可以完整编程控制 upstream 选择，配合 `lua-resty-healthcheck` 能够实现多级 failover。Kong 的商业版也有类似的优先级路由能力。但两者都需要引入新的网关组件生态（Lua 技术栈、独立运维），如果只是用插件实现 failover，不如直接回归 Higress 自身的插件能力（V5.1 方向）。

子网关如果要用现有代理实现一主多备，最终还是得落到类似 V2 的底层机制上，分层并没有解耦问题。

V3 的结论是当前生态下，纯粹靠现有 L7 代理的原生能力无法实现所需的主备语义。要么接受底层配置（V2），要么接受引入新组件。

→ [V3 调研方案](ha_design_v3/active_standby_ha_design_v3_research.draft.md)

---

## V4：Operator + EndpointSlice

沿着 V3 的结论，我们继续探索"加组件"的思路，这一次希望组件能和 K8s/Istio 生态融合，而不是像 V2 那样在底层打补丁。

V4 的设计是一个轻量 Operator：通过 CRD 声明每组的主备关系，Operator 自己跑 HTTP 健康探测，动态增删 EndpointSlice 里的端点。与 V1-V3 不同，Operator 对 failback 策略有完全的控制权——主节点恢复后是否切回、延迟多久切回、是否需人工确认，都可以通过 CRD 字段配置，而不是由 readiness probe 或 Envoy health check 的底层机制自动决定。Higress 看到的永远是标准的 K8s Service + EndpointSlice，所有 Istio 原生能力——熔断、连接池、TLS——全部自动生效，不需要任何 EnvoyFilter。

代价是多了一个需要维护的组件：Operator 自身要高可用部署，探测逻辑要处理各种边界情况（网络抖动、探测 goroutine 泄漏、主恢复后是否自动回切等）。作为交换，方案在团队可操作性和生态兼容性上有改善——它是 K8s 原生的、配置是声明式的、运维方式是团队熟悉的范式。

V4 的局限在于没有消除 L4 和 L7 之间的感知断层。Operator 修改 EndpointSlice 属于 L4（网络层端点管理），Higress 作为 L7 网关需要一定时间才能感知到 EndpointSlice 的变化。虽然比 V1 的 readiness probe 更快，但在极端情况下（如 Operator 与 Higress 的 watch 延迟），仍然存在请求被发送到已移除端点的窗口期。探测和路由决策仍然分属两个层面，L7 没有直接做出路由决策。

V4 是一个务实的选择：用组件复杂度换配置标准性和运维友好性，但感知延迟的硬边界仍然存在。

→ [V4 调研方案](ha_design_v4/active_standby_ha_design_v4_research.draft.md)

---

## V5：ai-proxy 给了我们一个启发——WASM 插件能做 failover 吗？

在研究控制面配置的同时，我们注意到 Higress 的 ai-proxy 插件已经在用 WASM 做 AI 网关的扩展了。这让我们产生了一个新的问题：如果不通过控制面配置主备切换，能否在数据面——请求经过 Envoy 时——将失败的请求 transparently 重试到备节点？

V5 验证了 proxy-wasm ABI 的能力边界，结论分为两个方面：

- **preemptive 切换做不到**。WASM Filter 在 Router 之前执行，修改的 header 和 filter state 影响不了 Router 的 cluster 选择决策。主节点不健康时，请求还是会先发到主节点。
- **被动重试可以做**。如果主节点返回了 5xx，WASM 插件可以在 `on_response_headers` 里通过 `dispatch_http_call` 向备节点重新发起请求，然后把备的响应 transparently 返回给客户端。客户端全程看不到错误。

这意味着 WASM 方案不能 preemptive 避开不健康的节点，只能在节点返回 5xx 后 transparently 重试到备节点，客户端全程不感知错误。我们还探讨了 preemptive 增强策略（失败记忆窗口、后台主动探测预热），但这些超出了 WASM ABI 当前能直接支持的范围。

V5 把思路从"控制面怎么配"拓展到了"数据面请求生命周期里能做什么"。它不替代 V1-V4 的控制面方案，而是一个互补的维度。

→ [V5 调研方案](ha_design_v5/active_standby_ha_design_v5_research.draft.md)

---

## V5.1：Higress 插件化能力的再评估

V5 的结论——WASM 插件无法 preemptive 切换 upstream——基于标准 proxy-wasm ABI 的边界分析。写完 V5 后回看，这个结论的调研范围可能过窄：它只覆盖了标准 ABI，没有评估 Higress 在标准 ABI 之上自行扩展的额外能力。

Higress 的插件体系有两条路线：golang-filter（native Go，编译为 .so）和 wasm-go（WebAssembly）。我们读了 Higress 官方仓库的源码，特别是 `ai-proxy` 和 `ai-load-balancer` 两个生产级插件的实现。

读 `ai-load-balancer` 的时候注意到一个 API：`SetUpstreamOverrideHost`。这不是标准 proxy-wasm ABI 里的函数，而是 Higress 自定义扩展的 host call。它允许插件直接覆盖 Envoy 选中的 upstream endpoint，将请求固定发送到指定的 IP:port。V5 中"WASM 无法影响 routing 决策"的结论需要在这个前提下重新审视。

读 `ai-proxy` 的源码，可以看到它已经实现了 token-level health check（lease-based leader election）、failover 和 retry 机制，使用的是 `RegisterTickFunc` + `DispatchHttpCall` + `SharedData` 的组合。Higress 的 wasm-go 框架在以下方面超出了标准 ABI：

- `SetUpstreamOverrideHost`：直接指定 endpoint，实现 preemptive 切换
- `x-higress-target-cluster` header + `cluster_header` 路由：动态切换 cluster
- `RegisterTickFunc`：周期性主动健康检查
- `GetUpstreamHosts`：获取 endpoint 列表及 Envoy 被动健康状态

这些发现意味着 V5 的"preemptive 切换不可行"结论只在标准 ABI 的假设下成立。借助 Higress 的自定义扩展，preemptive 切换可以在一个插件内实现：主动健康检查、preemptive endpoint 锁定、失败时被动重试。不需要引入额外组件（对比 V4 的 Operator），也不需要底层配置 hack（对比 V2 的 EnvoyFilter）。

V5.1 还带来了一个概念上的调整。之前 V1-V4 都在"主备"的框架下思考——谁主谁备、怎么切换、怎么切回。V5.1 的实现细节设计让我们看到另一种描述方式：所有 endpoint 是平等的，插件的作用只是基于健康检查选择一个节点并持续锁定，避免轮询导致的 cache tree 分散。不存在固定的"主"和"备"，只有"当前锁定的"和"未锁定的"。

基于这些调研，V5.1 完成了一份详细的实现细节设计文档（`implementation_design.md`），涵盖了拓扑感知、健康检查状态机、请求控制、`SetUpstreamOverrideHost` 锁定、双阶段重试策略（先同 endpoint 重试排除瞬时抖动，再转移）、SharedData 分层缓存优化、K8s 资源 spec 和部署细节。这份设计的可行性建立在 `ai-proxy` 和 `ai-load-balancer` 两个已生产验证的插件之上。

V5.1 的拓扑结构大致如下：

```
                         外部请求
                            │
                            ▼
                   ┌─────────────────┐
                   │  Higress Gateway │
                   │  (外层 weight/   │
                   │   轮询到 Group)  │
                   └────────┬────────┘
                            │
                   ┌────────▼────────┐
                   │  wasm-go Plugin  │
                   │  (按 Group 选择  │
                   │   healthy ep,    │
                   │   SetUpstream    │
                   │   OverrideHost)  │
                   └────────┬────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
   │ Group-1     │   │ Group-2     │   │ Group-3     │
   │ (Envoy      │   │ (Envoy      │   │ (Envoy      │
   │  cluster)   │   │  cluster)   │   │  cluster)   │
   │             │   │             │   │             │
   │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │
   │ │SMG-1.1  │ │   │ │SMG-2.1  │ │   │ │SMG-3.1  │ │
   │ │★locked  │ │   │ │★locked  │ │   │ │★locked  │ │
   │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │
   │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │
   │ │SMG-1.2  │ │   │ │SMG-2.2  │ │   │ │SMG-3.2  │ │
   │ │  standby│ │   │ │  standby│ │   │ │  standby│ │
   │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │
   │ ┌─────────┐ │   │ ┌─────────┐ │   │ ┌─────────┐ │
   │ │SMG-1.3  │ │   │ │SMG-2.3  │ │   │ │SMG-3.3  │ │
   │ │  standby│ │   │ │  standby│ │   │ │  standby│ │
   │ └─────────┘ │   │ └─────────┘ │   │ └─────────┘ │
   └─────────────┘   └─────────────┘   └─────────────┘
```

- 外层 Higress 通过 VirtualService weight/轮询将请求路由到某个 Group（Envoy cluster）
- **在请求到达具体 endpoint 之前**，wasm-go Plugin 拦截请求，读取当前 Group 的 endpoint 列表，选择 healthy endpoint 并**稳定锁定**（`SetUpstreamOverrideHost`）
- 所有请求集中到锁定的 endpoint，避免轮询导致的 cache tree 分散
- 每个 Group 内部的 endpoint 完全平等，不存在固定的"主"和"备"

→ [V5.1 调研方案](ha_design_v5.1/active_standby_ha_design_v5.1_research.draft.md)
→ [V5.1 实现细节设计](ha_design_v5.1/implementation_design.md)

---

## V6：cache tree 同步，另一种思路

回顾 V1-V5，所有方案都默认了一件事：cache tree 分散是不可避免的，所以必须用主备结构来规避它。重新读 sglang 源码时，这个前提本身出现了松动。

`cache_aware.rs` 里的 tree 分散，根因是 `DashMap<String, Arc<Tree>>` 作为 Router 进程本地的内存结构，每个实例只看到自己收到的请求。与此同时，sglang 代码库里已经预留了 `sync_tree_operation` 的发送端和 `apply_remote_tree_operation` 的接收端骨架，只是接收端的触发/订阅还没完全打通。

这意味着：如果我们把 tree 操作的事件同步做起来，多个 Router 实例完全可以同时接收流量而树不分散。也就不再需要主备结构。这让我们从"怎么切主备"的问题域转向了"怎么同步状态"的问题域。

V6 调研了几种同步拓扑：KV Event Stream 中心总线、Router 间操作日志同步、中心化存储（Redis/etcd）等。参考了 NVIDIA Dynamo 等系统的做法后，我们倾向于一种混合方案——本地 Tree 保持读性能，Worker 推送的 KV 事件流提供最终一致性，Router 间再异步同步操作日志填补间隙。

这个方向如果走通，SMG 的 Router 层可以做到对等扩展，主备切换逻辑不再是必需。它和 V5 的 WASM failover 不在同一层面：V5 处理 worker 节点瞬时故障时的请求保底，V6 处理 Router 层多实例扩展时的 cache 一致性。

主备结构下，一个 Group 内只有一个活跃 Router 维护 cache tree，其余备节点不持有完整树。如果改为对等扩展加同步，每个 Router 实例都需要维护一份完整的 cache tree 数据，内存需求随副本数线性增加。而且无论选择哪种同步拓扑，都需要引入一个高可用的日志发布订阅组件来承载 tree 操作事件流，这本身就是一个需要独立设计和运维的系统。

→ [V6 调研方案](ha_design_v6/active_standby_ha_design_v6_research.draft.md)

---

## 阶段性总结

目前倾向的方案是 V5.1：用 Higress wasm-go 插件实现基于健康检查的 endpoint 稳定锁定，同时支持 preemptive 切换和被动重试，不引入额外组件。这个方向的可行性建立在 `ai-proxy` 和 `ai-load-balancer` 两个已生产验证的插件之上。

V5.1 本身还有待验证：implementation_design.md 中的代码逻辑能否在实际环境中跑通，SharedData 在并发写入时是否正确处理，SSE 流式场景的重试边界是否覆盖完整，这些都需要在后续的开发或实验中逐步验证。

V6 是另一个方向。如果 cache tree 同步能走通，主备结构就不再需要，Router 层可以做到对等扩展。在 V6 成熟之前，V5.1 仍然有必要——即使 cache tree 分散问题最终被同步解决，worker 节点的瞬时故障仍然需要请求保底机制。

---

## 相关文档

- V0 — 给定的起点：PD 分离架构蓝图 → [V0 调研方案](ha_design_v0/pd_disaggregated_architecture.md)
- V1 — K8s readiness probe 做主备 → [V1 调研方案](ha_design_v1/active_standby_ha_design_v1.draft.md)
- V2 — Envoy aggregate cluster 做主备 → [V2 调研方案](ha_design_v2/active_standby_ha_design_v2.draft.md)
- V3 — 父子网关中间层 → [V3 调研方案](ha_design_v3/active_standby_ha_design_v3_research.draft.md)
- V4 — Operator + EndpointSlice → [V4 调研方案](ha_design_v4/active_standby_ha_design_v4_research.draft.md)
- V5 — WASM 标准 ABI 能力边界 → [V5 调研方案](ha_design_v5/active_standby_ha_design_v5_research.draft.md)
- V5.1 — Higress 自定义扩展再评估 → [V5.1 调研方案](ha_design_v5.1/active_standby_ha_design_v5.1_research.draft.md)
- V5.1 — 实现细节设计 → [V5.1 实现细节设计](ha_design_v5.1/implementation_design.md)
- V6 — cache tree 同步 → [V6 调研方案](ha_design_v6/active_standby_ha_design_v6_research.draft.md)
