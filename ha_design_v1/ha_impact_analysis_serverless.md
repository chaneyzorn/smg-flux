# SMG 主备方案故障影响分析 — 服务端无感前提

> **前提约束**：故障切换不暴露到客户端层面，尽量由服务端（Higress + SMG）内部处理。客户端只发送一次请求，不关心后端哪个 SMG 处理。
>
> 本文重新评估单轮对话和多轮对话在服务端无感前提下的影响。

---

## 一、服务端无感的技术前提

### 1.1 什么能做到服务端无感

| 场景 | 能否服务端无感 | 技术依赖 |
|------|-------------|---------|
| **非流式请求**，SMG 崩溃前未返回任何响应 | ✅ 可以 | Higress/Envoy 自动重试到健康端点 |
| **非流式请求**，SMG 崩溃前已返回部分响应 | ❌ 不可以 | HTTP 响应已发出，无法撤回 |
| **流式请求（SSE）**，SMG 崩溃前未发出首 token | ✅ 可以 | Higress 超时后重试 |
| **流式请求（SSE）**，SMG 已发出部分 token | ❌ 不可以 | 客户端已收到不完整流，必须重新发起 |

**关键结论**：
- 服务端无感的核心依赖是 **Higress/Envoy 的请求级重试**
- 但一旦任何字节（包括 HTTP header 或 SSE 首 token）已返回给客户端，重试就不可能了
- 流式请求在已经流出一部分后，**必然需要客户端感知**
- 我们的目标是最小化"需要客户端感知"的场景

### 1.2 Higress 重试配置

通过 Istio `VirtualService` 配置 Envoy 级重试：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: kimi-router-vs
  namespace: higress-system
spec:
  hosts:
    - kimi-router-ha-cluster.local
  gateways:
    - kimi-router-gateway
  http:
    - match:
        - uri:
            prefix: /
      route:
        - destination:
            host: kimi-router-ha-cluster.local
            port:
              number: 30002
      retries:
        attempts: 3
        perTryTimeout: 30s
        retryOn: gateway-error,connect-failure,refused-stream,unavailable
      timeout: 120s
```

**重试语义**：
- 请求发到某个端点（如 SMG-1-主），如果连接失败、返回 502/503/504，Envoy **自动重试**到其他健康端点
- 客户端完全无感知，只看到一个正常的延迟（可能稍长）
- 如果所有重试都失败，才返回 503 给客户端

### 1.3 主备方案下的重试行为

**正常状态**：Higress 轮询 3 个 Service（Service-1/2/3），每个 Service 解析到各自的 SMG-主。

**SMG-1-主挂了，备还未 ready**：
- Service-1 的 endpoints 为空（主已移除，备还未加入）
- Higress 将请求发到 Service-1 → DNS 解析不到端点 → 连接失败
- Envoy `retryOn: connect-failure` 触发 → 重试到其他 Service（Service-2 或 Service-3）
- 客户端无感知

**SMG-1-主挂了，备已 ready**：
- Service-1 的 endpoints 现在指向 SMG-1-备
- Higress 将请求发到 Service-1 → SMG-1-备正常处理
- 客户端无感知

**关键问题**：在"备接管"的窗口期（~5-10s），请求会被重试到 Service-2/3。这引入了**跨组请求漂移**。

---

## 二、当前现状（无 HA，Higress 轮询 3 个 SMG）

### 2.1 单轮对话：SMG-1 挂了

**非流式请求**：
1. 请求发到 SMG-1 → SMG-1 崩溃 → TCP 连接断开
2. Higress 未收到任何响应（HTTP header 都没发），满足重试条件
3. Envoy 自动重试到 SMG-2 或 SMG-3
4. SMG-2/3 正常处理，返回响应
5. **客户端完全无感知**（只感知到延迟增加了几十毫秒到几秒）

**流式请求，首 token 未发出**：
1. 同非流式，Higress 可以在首 token 发出前重试
2. 客户端无感知

**流式请求，已发出部分 token**：
1. SMG-1 在流式生成到一半时崩溃
2. 客户端已收到部分 SSE 数据（如 `"data: {"choices":[{"delta":{"content":"Hel"`）
3. Higress 无法重试（响应已部分返回）
4. 客户端 SSE 连接断开，收到不完整的 JSON
5. **客户端必须感知并重新发起请求**

**影响评估**：
- 非流式：✅ 服务端无感（Higress 重试兜底）
- 流式（首 token 前）：✅ 服务端无感
- 流式（已发出 token）：❌ 客户端必须重连

### 2.2 多轮对话：SMG-1 挂了

**场景**：User A 的 1-3 轮对话被路由到 SMG-1。

**第 4 轮非流式请求**：
1. 请求发到 SMG-1 → SMG-1 崩溃
2. Higress 重试到 SMG-2 或 SMG-3
3. SMG-2/3 的 cache_aware 树中没有 User A 的历史
4. SMG-2/3 选到新的 P worker，Radix Cache miss
5. 全量 prefill → 延迟暴增
6. 但**请求最终成功返回**，客户端无感知（除了延迟变长）

**第 4 轮流式请求（已发出部分 token）**：
1. SMG-1 在流式生成到一半时崩溃
2. 客户端收到不完整流，连接断开
3. 客户端重新发起第 4 轮请求
4. 新请求被 Higress 路由到 SMG-2/3
5. SMG-2/3 无历史，全量 prefill
6. 客户端最终收到完整响应
7. **客户端必须感知故障并重新发起**

**多轮对话的核心问题**：
- 即使服务端能"兜底"让请求最终成功，**延迟模型已经崩溃**
- 多轮对话的第 4 轮本应只计算增量 KV（<100ms prefill）
- 但切换后变成了全量 prefill（可能 2-5s）
- 客户端虽然"无感"于故障本身，但会感知到**突然的延迟飙升**
- 如果对话很长（20 轮），这种延迟飙升可能是致命的

---

## 三、官方 Session Affinity（一致性哈希）

### 3.1 方案原理

Higress 基于 user ID 做一致性哈希，让同一用户的请求尽量固定到同一个 SMG：

```
User A ──hash──► SMG-1（固定）
User B ──hash──► SMG-2（固定）
```

### 3.2 单轮对话：SMG-1 挂了

**非流式请求**：
1. User A 的请求发到 SMG-1 → 崩溃
2. Higress 重试 → 一致性哈希环中 SMG-1 被标记为不健康
3. 重试时，一致性哈希将 User A 重新映射到 SMG-2 或 SMG-3
4. SMG-2/3 正常处理单轮请求
5. **客户端无感知**

**问题**：一致性哈希的"固定性"在故障时被打破了。重试逻辑和一致性哈希的目标（固定到同一节点）存在冲突。

### 3.3 多轮对话：SMG-1 挂了

**第 4 轮非流式请求**：
1. User A 的请求原本固定到 SMG-1 → SMG-1 崩溃
2. Higress 重试到 SMG-2（一致性哈希重新映射）
3. SMG-2 无 User A 历史，全量 prefill
4. 请求成功返回，客户端无感知

**第 5 轮及以后**：
- User A 的请求现在固定到 SMG-2
- SMG-2 的 cache_aware 树中逐渐建立 User A 的历史
- 但 SMG-2 选的 P worker 和之前 SMG-1 选的 P worker **不同**
- P worker 的 Radix Cache 中**没有** `"turn1 turn2 turn3"` 的 KV
- 即使 SMG-2 的树开始命中，选到的 P worker 也无法复用 KV

**对比现状**：
- 现状：User A 的请求可能在 SMG-1/2/3 之间漂移，每次飘到新 SMG 都 miss
- Session Affinity：User A 的请求在 SMG-1 挂了之后**固定**到 SMG-2，至少后续轮次不再漂移
- **但核心问题一样**：切换后的 P worker 不同，Radix Cache 无法复用

---

## 四、推荐方案 A：K8s Service + Readiness 主备切换

### 4.1 方案架构（重申）

```
Higress
  │ 轮询 3 个 Service
  ├─────────────┬─────────────┬─────────────┐
  ▼             ▼             ▼
Service-1    Service-2    Service-3
  │             │             │
  ▼             ▼             ▼
主(活跃)      主(活跃)      主(活跃)
备(静默)      备(静默)      备(静默)
```

### 4.2 关键设计：Higress 重试策略优化

标准重试配置的问题是：Service-1 挂了后，请求被重试到 Service-2/3，导致**跨组漂移**。

更优的策略是：**优先在同一 Service 内重试，给备接管留出时间**。

但标准 Envoy 重试不支持"优先在同一 Service/主机重试"。一个折中方案是：

```yaml
retries:
  attempts: 3
  perTryTimeout: 30s
  retryOn: gateway-error,connect-failure,refused-stream
```

在"主刚挂、备未 ready"的窗口期：
- 第 1 次尝试：Service-1 → 连接失败
- 第 2 次尝试：Service-2 → 成功（跨组漂移）
- 第 3 次尝试：Service-3 → 如果第 2 次也失败

**窗口期内的请求会被迫跨组漂移**。

### 4.3 缩短窗口期是关键

服务端无感的核心在于**缩短"主挂了到备接管"的窗口期**：

| 参数 | 默认值 | 建议值 | 效果 |
|------|--------|--------|------|
| K8s readiness `periodSeconds` | 10s | 3s | 更快检测主故障 |
| K8s readiness `failureThreshold` | 3 | 2 | 更快标记主为不健康 |
| SMG 备 peer 探测周期 | - | 3s | 备更快发现主挂了 |
| SMG 备 peer 失败阈值 | - | 2 | 备更快提升为 ready |
| Higress DNS TTL | 默认 | 5s | 更快解析到新的 endpoints |

**优化后窗口期**：从 ~30s 缩短到 **~6-9s**。

在这个窗口期内：
- 如果并发请求量不高，可能**没有任何请求落入窗口期**
- 即使有请求落入窗口期，Higress 重试到 Service-2/3，客户端无感知

### 4.4 单轮对话：SMG-1-主 挂了

**非流式请求**：
- 窗口期外（备已 ready）：请求发到 Service-1 → SMG-1-备正常处理 ✅ 无感
- 窗口期内：请求发到 Service-1 → 连接失败 → Higress 重试到 Service-2/3 → 正常处理 ✅ 无感

**流式请求（已发出 token）**：
- SMG-1-主在生成到一半时崩溃
- 客户端收到不完整流 ❌ 必须感知
- **这是所有方案的共同限制，无法避免**

### 4.5 多轮对话：SMG-1-主 挂了

#### Phase 1（无 mesh 同步）

**窗口期外的请求**（备已接管）：
1. 第 4 轮请求发到 Service-1 → SMG-1-备
2. SMG-1-备的 cache_aware 树：**空树**
3. 前缀匹配失败，fallback 选 P worker
4. **关键问题**：备选到哪个 P worker？
   - 如果 SMG-1-主之前将 `"turn1 turn2 turn3"` 路由到 `Prefill-2`
   - 备的 cache_aware 树中没有这个记录
   - 备通过最短队列策略选择当前负载最低的 P worker
   - 如果 `Prefill-2` 当前负载最低 → 选中 `Prefill-2` → Radix Cache 命中 ✅
   - 如果 `Prefill-1` 当前负载最低 → 选中 `Prefill-1` → Radix Cache miss ❌
5. **这是概率性的**

**服务端无感评估**：
- 如果选对 P worker：延迟正常，客户端无感 ✅
- 如果选错 P worker：全量 prefill，延迟暴增 ⚠️ 客户端"有感"（延迟变长）
- 但请求最终成功返回，不需要客户端重试

**窗口期内的请求**：
1. 第 4 轮请求被 Higress 重试到 Service-2/3
2. Service-2/3 的 SMG 没有 User A 的历史
3. 全量 prefill，延迟暴增
4. 但请求成功返回，客户端无感（除了延迟变长）

**第 5 轮及以后**：
- 如果第 4 轮被 SMG-1-备处理：
  - SMG-1-备的 cache_aware 树中现在有了 `"turn1 turn2 turn3 turn4_prompt"` → 某个 P worker
  - 后续轮次如果继续命中该 P worker，延迟逐渐恢复正常
- 如果第 4 轮被 Service-2/3 处理（窗口期漂移）：
  - 第 5 轮回到 Service-1（备）
  - Service-1 的备仍然没有 User A 的历史（因为第 4 轮是在 Service-2/3 建立的）
  - 继续 cache miss

**这是 Phase 1 最糟糕的情况**：窗口期内的请求漂移到了其他 Service，导致 cache_aware 树"碎片化"。

#### Phase 2（有 mesh 同步）

**窗口期外的请求**：
1. 第 4 轮请求发到 Service-1 → SMG-1-备
2. SMG-1-备的 cache_aware 树：通过 mesh 同步，和主基本一致
3. 前缀匹配命中 `"turn1 turn2 turn3"` → `Prefill-2`
4. `Prefill-2` 的 Radix Cache 命中
5. 只计算增量 KV，延迟正常 ✅

**窗口期内的请求**：
- 即使被重试到 Service-2/3，也只是第 4 轮延迟增加
- 第 5 轮回到 Service-1（备），备的 cache_aware 树是完整的
- 后续轮次正常 ✅

**影响评估（Phase 2）**：
- 窗口期内的 1-2 个请求：可能延迟增加（如果漂移）
- 窗口期后的所有请求：**完全无感**（cache_aware 树无缝继承）
- 即使窗口期内有请求漂移，也只是"一次性的延迟增加"，不会持续

---

## 五、三种方案对比（服务端无感前提）

### 5.1 单轮对话

| 评估维度 | 当前现状 | Session Affinity | 主备 Phase 1 | 主备 Phase 2 |
|---------|---------|-----------------|-------------|-------------|
| **非流式请求** | ✅ Higress 重试兜底，无感 | ✅ 同左 | ✅ 同左 | ✅ 同左 |
| **流式（首 token 前）** | ✅ Higress 超时重试，无感 | ✅ 同左 | ✅ 同左 | ✅ 同左 |
| **流式（已发出 token）** | ❌ 必须客户端重连 | ❌ 同左 | ❌ 同左 | ❌ 同左 |
| **窗口期内请求** | 重试到其他 SMG，正常处理 | 重试到其他 SMG，正常处理 | 重试到其他 Service，正常处理 | 同左 |

**结论**：单轮对话在服务端无感前提下，四种方案表现基本一致。Higress 重试配置是关键。

### 5.2 多轮对话

| 评估维度 | 当前现状 | Session Affinity | 主备 Phase 1 | 主备 Phase 2 |
|---------|---------|-----------------|-------------|-------------|
| **非流式，窗口期外** | 新 SMG 无历史，全量 prefill，延迟暴增 | 新 SMG 无历史，全量 prefill，延迟暴增 | 备选 P worker 靠运气，可能延迟暴增 | ✅ 备继承树的树，延迟正常 |
| **非流式，窗口期内** | 重试到其他 SMG，全量 prefill | 重试到新 SMG，全量 prefill | 重试到其他 Service，全量 prefill | 重试到其他 Service，全量 prefill |
| **流式（已发出 token）** | ❌ 客户端必须重连 | ❌ 同左 | ❌ 同左 | ❌ 同左 |
| **窗口期后恢复速度** | 永远 miss（请求继续漂移） | 逐渐恢复（新 SMG 固定后建树） | 靠运气（如果选对 P worker 则恢复） | ✅ 立即恢复（树已同步） |
| **树碎片化风险** | 高（请求在多个 SMG 漂移） | 中（切换后固定到新 SMG） | 中（窗口期内可能漂移） | 低（备继承主的树） |

**结论**：多轮对话场景下，**主备 Phase 2 是唯一能在服务端无感前提下保持延迟稳定的方案**。

---

## 六、服务端无感的最佳实践配置

### 6.1 Higress 重试配置（必须）

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: kimi-router-vs
  namespace: higress-system
spec:
  hosts:
    - kimi-router-ha-cluster.local
  http:
    - route:
        - destination:
            host: kimi-router-ha-cluster.local
            port:
              number: 30002
      retries:
        attempts: 3
        perTryTimeout: 30s
        retryOn: gateway-error,connect-failure,refused-stream,unavailable
      timeout: 120s
```

### 6.2 缩短切换窗口期的配置

**SMG 备实例启动参数**：
```bash
sgl-model-gateway \
  --role backup \
  --group-id 1 \
  --peer-address http://smg-1-primary:30002 \
  --peer-probe-interval-secs 3 \
  --peer-probe-failure-threshold 2
```

**K8s Deployment readiness probe**：
```yaml
readinessProbe:
  httpGet:
    path: /readiness
    port: 30002
  periodSeconds: 3
  failureThreshold: 2
  successThreshold: 1
```

**Higress DNS TTL**：
```yaml
# 在 DestinationRule 中配置
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: kimi-router-dr
spec:
  host: kimi-router-ha-cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 512
      http:
        http1MaxPendingRequests: 256
    loadBalancer:
      simple: ROUND_ROBIN
```

Envoy 的 DNS TTL 默认由 ServiceEntry 的 `resolution: DNS` 控制，可通过 `EnvoyFilter` 调整。

### 6.3 流式请求的特殊处理

**服务端能做到的**：
- 在 SSE 流的第一个 `data:` 发出前检测到 SMG 故障 → Higress 重试，客户端无感
- 但一旦发了第一个 token，就无法挽回了

**建议的客户端策略（最小化感知）**：
```javascript
// 客户端伪代码
let buffer = "";
eventSource.onmessage = (event) => {
  buffer += event.data;
};
eventSource.onerror = () => {
  // 流中断，尝试恢复
  if (buffer) {
    // 已收到部分响应，重新发起请求
    // 新请求需要包含 conversation_id，让服务端识别上下文
    reconnectWithConversationId(conversationId, buffer.length);
  }
};
```

但用户明确要求"不要暴露到客户端"，所以对于流式请求，我们只能：
- 在服务端尽量降低 SMG 崩溃的概率（如优雅关闭、预探测）
- 但**完全隐藏流式中断是不可能的**

---

## 七、修正后的最终建议

### 7.1 非流式请求：服务端完全可以无感

只要配置 Higress 重试，所有非流式请求（包括单轮和多轮的非流式调用）都可以在 SMG 故障时自动重试到健康端点，客户端完全无感知。

### 7.2 流式请求：服务端无法完全无感

这是由 SSE/gRPC Streaming 的协议特性决定的。一旦 token 流出，就无法"撤销"或"续传"。

**缓解措施**：
- 缩短切换窗口期（~6s），降低流式请求"正好在生成过程中"碰到故障的概率
- SMG 实现优雅关闭（`SIGTERM` 时停止接受新请求，等待已有请求完成）
- 但这只是降低概率，不是消除

### 7.3 多轮对话的延迟稳定性

| 方案 | 窗口期外延迟 | 窗口期内延迟 | 延迟恢复速度 |
|------|-----------|-----------|-----------|
| 现状 | 永远不稳定 | 不稳定 | 不恢复 |
| Session Affinity | 不稳定 | 不稳定 | 缓慢（需重建树） |
| 主备 Phase 1 | 靠运气 | 不稳定 | 靠运气 |
| **主备 Phase 2** | **稳定** | 短暂不稳定 | **立即恢复** |

### 7.4 最简可行路径（服务端无感）

如果业务以**非流式请求**为主：
- Phase 1 MVP + Higress 重试配置即可
- 备的 cache_aware 树是否同步不影响"请求最终成功"
- 只影响延迟（多轮对话下延迟可能波动）

如果业务以**流式请求**为主：
- 无论如何都需要客户端实现重连逻辑
- Phase 2 可以最小化重连后的延迟（因为备的树是热的）
- 如果对话很长，Phase 2 的价值更大

如果业务以**多轮对话**为主：
- **强烈建议 Phase 2**，否则每次切换都等于"对话重新开始"
- 服务端无感（请求最终成功），但延迟模型崩溃，用户体验差
