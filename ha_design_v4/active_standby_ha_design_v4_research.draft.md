# SMG Active-Standby HA 架构 V4 方案 — K8s Operator + EndpointSlice

> 目标：完全避免 EnvoyFilter 的底层 hack，使用**标准 K8s API** 实现主备 failover。通过自研轻量 Operator 持续探测 SMG 健康状态，动态管理 EndpointSlice，让 Higress/Istio 的所有原生能力（DestinationRule 熔断、连接池、TLS 等）全部生效。

---

## 一、V4 方案定位

| 版本 | 方案 | 核心机制 | 主要痛点 |
|------|------|---------|---------|
| V1 | K8s Service + Readiness Probe | kubelet TCP 探测 + readiness 控制 | L4 探测慢（6-9s），不支持多备 |
| V2 | EnvoyFilter + Aggregate Cluster | Envoy 底层 protobuf 配置 | 配置底层 hack，与 Istio 版本耦合 |
| V3 | 父子网关（调研结论：不可行） | Nginx/HAProxy/Higress 子网关 | 现有 L7 代理不支持多级依次 failover |
| **V4** | **K8s Operator + EndpointSlice** | **Operator HTTP 探测 + 动态 EndpointSlice** | **需开发新组件** |

V4 是在 V3 调研发现"父子网关难以实现多级 failover"之后，转向**自研附加组件**方向的方案。

---

## 二、核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Higress Gateway (Parent)                  │
│                                                              │
│  VirtualService: weight 33/33/34 -> Service-1/2/3          │
│  DestinationRule: LEAST_CONN, outlierDetection, tls        │
│  (所有 Istio 原生能力完全生效)                                │
│                                                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │Service-1 │   │Service-2 │   │Service-3 │
        │(标准K8s) │   │(标准K8s) │   │(标准K8s) │
        └────┬─────┘   └────┬─────┘   └────┬─────┘
             │              │              │
             ▼              ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │Endpoint  │   │Endpoint  │   │Endpoint  │
        │Slice-1   │   │Slice-2   │   │Slice-3   │
        │(Operator│   │(Operator│   │(Operator│
        │ 动态管理) │   │ 动态管理) │   │ 动态管理) │
        │ [主]     │   │ [主]     │   │ [主]     │  ← 正常时
        │          │   │          │   │          │
        │ [备A]    │   │ [备A]    │   │ [备A]    │  ← 主挂了时
        │          │   │          │   │          │
        │ [备B]    │   │ [备B]    │   │ [备B]    │  ← 备A也挂了时
        └──────────┘   └──────────┘   └──────────┘
             │              │              │
             ▼              ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │SMG Group1│   │SMG Group2│   │SMG Group3│
        │(Stateful│   │(Stateful│   │(Stateful│
        │  Set)   │   │  Set)   │   │  Set)   │
        │主/备A/备B│   │主/备A/备B│   │主/备A/备B│
        └──────────┘   └──────────┘   └──────────┘

        ┌──────────────────────────────────────────────┐
        │  smg-ha-operator (HA Deployment)             │
        │                                              │
        │  CRD: SMGFailoverConfig                      │
        │    - groups:                                 │
        │      - name: group-1                         │
        │        primary: smg-group-1-0                │
        │        backups: [smg-group-1-1, smg-group-1-2]│
        │        healthCheck:                          │
        │          interval: 1s                        │
        │          timeout: 3s                         │
        │          failureThreshold: 2                 │
        │        failbackStrategy: manual              │
        └──────────────────────────────────────────────┘
```

---

## 三、核心机制

### 3.1 为什么用 EndpointSlice？

K8s 中，Service 的端点列表由 **EndpointSlice** 资源维护：

- kubelet 默认根据 Pod 的 `ready` 状态自动更新 EndpointSlice
- 但我们可以通过 **自定义 Controller** 接管 EndpointSlice 的管理权
- Higress/Istio 通过 watch EndpointSlice 的变化来更新 Envoy 的 upstream endpoints
- 这是一个**完全标准的 K8s 机制**，没有任何 hack

### 3.2 Operator 的行为

```
Operator 持续执行 Reconcile Loop:

  for each group:
    1. 获取该 group 的所有 Pod IP（通过 StatefulSet 或 Pod 标签）
    2. HTTP GET http://<pod-ip>:30002/health
    3. 按优先级选择最健康的端点：primary (0) > backup-1 (1) > backup-2 (2)
    4. 更新 EndpointSlice：只保留选中的活跃端点
    5. Higress 通过 xDS 感知变化，通常在 1-2s 内生效
```

### 3.3 切换时序

```
t=0s      SMG-1-主崩溃
          /health 返回 503
             │
t=1s      Operator 第 1 次探测失败
             │ (failureThreshold=2，暂不切换)
             │
t=2s      Operator 第 2 次探测失败
          连续失败达到阈值
             │
          Operator 更新 EndpointSlice-1
          移除主，添加备A
             │
          K8s API Server watch 推送
             │
t=2.5s    Higress xDS 推送到达
          流量切换到备A
             │
          客户端无感知（非流式请求自动重试）
```

**切换延迟**：~2-4s（与 V2 的 Envoy active health check 相当）

---

## 四、CRD 设计

### 4.1 SMGFailoverConfig CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: smgfailoverconfigs.smg.ha.example.com
spec:
  group: smg.ha.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                groups:
                  type: array
                  items:
                    type: object
                    properties:
                      name:
                        type: string
                        description: "Group 名称，如 group-1"
                      serviceName:
                        type: string
                        description: "关联的 K8s Service 名称"
                      namespace:
                        type: string
                        default: "default"
                      primary:
                        type: object
                        properties:
                          podName:
                            type: string
                            description: "主 Pod 名称，如 smg-group-1-0"
                          port:
                            type: integer
                            default: 30002
                      backups:
                        type: array
                        items:
                          type: object
                          properties:
                            podName:
                              type: string
                            port:
                              type: integer
                              default: 30002
                      healthCheck:
                        type: object
                        properties:
                          path:
                            type: string
                            default: "/health"
                          scheme:
                            type: string
                            enum: ["http", "https"]
                            default: "http"
                          interval:
                            type: string
                            default: "1s"
                          timeout:
                            type: string
                            default: "3s"
                          failureThreshold:
                            type: integer
                            default: 2
                          successThreshold:
                            type: integer
                            default: 2
                      failbackStrategy:
                        type: string
                        enum: ["manual", "auto", "delayed"]
                        default: "manual"
                      failbackDelay:
                        type: string
                        default: "60s"
                        description: "delayed 回切时的等待时间"
```

### 4.2 用户配置示例

```yaml
apiVersion: smg.ha.example.com/v1
kind: SMGFailoverConfig
metadata:
  name: smg-ha-config
  namespace: default
spec:
  groups:
    - name: group-1
      serviceName: smg-group-1
      primary:
        podName: smg-group-1-0
        port: 30002
      backups:
        - podName: smg-group-1-1
          port: 30002
        - podName: smg-group-1-2
          port: 30002
      healthCheck:
        path: /health
        interval: 1s
        timeout: 3s
        failureThreshold: 2
        successThreshold: 2
      failbackStrategy: manual

    - name: group-2
      serviceName: smg-group-2
      primary:
        podName: smg-group-2-0
        port: 30002
      backups:
        - podName: smg-group-2-1
          port: 30002
      healthCheck:
        path: /health
        interval: 1s
        timeout: 3s
        failureThreshold: 2
      failbackStrategy: manual

    - name: group-3
      serviceName: smg-group-3
      primary:
        podName: smg-group-3-0
        port: 30002
      backups:
        - podName: smg-group-3-1
          port: 30002
        - podName: smg-group-3-2
          port: 30002
      healthCheck:
        path: /health
        interval: 1s
        timeout: 3s
        failureThreshold: 2
      failbackStrategy: manual
```

---

## 五、Operator 核心实现

### 5.1 架构

```
smg-ha-operator (Go / Python)
│
├── Controller Manager
│   └── Reconcile Loop (每 1s 执行一次)
│
├── Health Checker
│   └── HTTP Client Pool (并发探测所有端点)
│
├── EndpointSlice Manager
│   └── Create / Update / Delete EndpointSlice
│
├── Leader Election
│   └── 确保只有一个实例在管理 EndpointSlice
│
└── Metrics Server (Prometheus)
    └── active_endpoint, health_check_failures, failover_events
```

### 5.2 核心代码逻辑

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"

    corev1 "k8s.io/api/core/v1"
    discoveryv1 "k8s.io/api/discovery/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type EndpointInfo struct {
    PodName   string
    Address   string
    Port      int32
    Healthy   bool
    Priority  int // 0=primary, 1=backup-1, 2=backup-2
}

type FailoverController struct {
    k8sClient  kubernetes.Interface
    httpClient *http.Client
}

func (c *FailoverController) ReconcileGroup(ctx context.Context, group SMGGroup) error {
    // 1. 解析所有端点的 Pod IP
    endpoints, err := c.resolveEndpoints(ctx, group)
    if err != nil {
        return fmt.Errorf("resolve endpoints: %w", err)
    }

    // 2. 并发 L7 健康检查
    for i := range endpoints {
        endpoints[i].Healthy = c.healthCheck(
            endpoints[i].Address,
            endpoints[i].Port,
            group.HealthCheck.Path,
            group.HealthCheck.Timeout,
        )
    }

    // 3. 选择最优先的健康端点
    activeEndpoint := c.selectActiveEndpoint(endpoints)
    if activeEndpoint == nil {
        return fmt.Errorf("no healthy endpoint for group %s", group.Name)
    }

    // 4. 更新 EndpointSlice
    return c.updateEndpointSlice(ctx, group, activeEndpoint)
}

func (c *FailoverController) resolveEndpoints(ctx context.Context, group SMGGroup) ([]EndpointInfo, error) {
    var endpoints []EndpointInfo

    // 解析主
    primaryPod, err := c.k8sClient.CoreV1().Pods(group.Namespace).Get(ctx, group.Primary.PodName, metav1.GetOptions{})
    if err == nil && primaryPod.Status.PodIP != "" {
        endpoints = append(endpoints, EndpointInfo{
            PodName:  group.Primary.PodName,
            Address:  primaryPod.Status.PodIP,
            Port:     group.Primary.Port,
            Priority: 0,
        })
    }

    // 解析备
    for i, backup := range group.Backups {
        pod, err := c.k8sClient.CoreV1().Pods(group.Namespace).Get(ctx, backup.PodName, metav1.GetOptions{})
        if err == nil && pod.Status.PodIP != "" {
            endpoints = append(endpoints, EndpointInfo{
                PodName:  backup.PodName,
                Address:  pod.Status.PodIP,
                Port:     backup.Port,
                Priority: i + 1,
            })
        }
    }

    return endpoints, nil
}

func (c *FailoverController) healthCheck(address string, port int32, path string, timeout time.Duration) bool {
    url := fmt.Sprintf("http://%s:%d%s", address, port, path)
    client := &http.Client{Timeout: timeout}
    resp, err := client.Get(url)
    if err != nil {
        return false
    }
    defer resp.Body.Close()
    return resp.StatusCode == http.StatusOK
}

func (c *FailoverController) selectActiveEndpoint(endpoints []EndpointInfo) *EndpointInfo {
    // 按优先级排序：primary (0) > backup-1 (1) > backup-2 (2)
    for priority := 0; priority <= 2; priority++ {
        for _, ep := range endpoints {
            if ep.Priority == priority && ep.Healthy {
                return &ep
            }
        }
    }
    return nil
}

func (c *FailoverController) updateEndpointSlice(ctx context.Context, group SMGGroup, active *EndpointInfo) error {
    sliceName := group.ServiceName + "-ha"

    slice, err := c.k8sClient.DiscoveryV1().EndpointSlices(group.Namespace).Get(ctx, sliceName, metav1.GetOptions{})

    portName := "http"
    portProtocol := corev1.ProtocolTCP

    newSlice := &discoveryv1.EndpointSlice{
        ObjectMeta: metav1.ObjectMeta{
            Name:      sliceName,
            Namespace: group.Namespace,
            Labels: map[string]string{
                discoveryv1.LabelServiceName: group.ServiceName,
                "managed-by":                "smg-ha-operator",
            },
            OwnerReferences: []metav1.OwnerReference{
                {
                    APIVersion: "v1",
                    Kind:       "Service",
                    Name:       group.ServiceName,
                },
            },
        },
        AddressType: discoveryv1.AddressTypeIPv4,
        Ports: []discoveryv1.EndpointPort{{
            Name:     &portName,
            Port:     &active.Port,
            Protocol: &portProtocol,
        }},
        Endpoints: []discoveryv1.Endpoint{{
            Addresses: []string{active.Address},
            Conditions: discoveryv1.EndpointConditions{
                Ready: boolPtr(true),
            },
            TargetRef: &corev1.ObjectReference{
                Kind:      "Pod",
                Name:      active.PodName,
                Namespace: group.Namespace,
            },
        }},
    }

    if errors.IsNotFound(err) {
        _, err = c.k8sClient.DiscoveryV1().EndpointSlices(group.Namespace).Create(ctx, newSlice, metav1.CreateOptions{})
    } else if err != nil {
        return err
    } else {
        newSlice.ResourceVersion = slice.ResourceVersion
        _, err = c.k8sClient.DiscoveryV1().EndpointSlices(group.Namespace).Update(ctx, newSlice, metav1.UpdateOptions{})
    }

    return err
}
```

### 5.3 关键设计决策

**决策 1：创建新的 EndpointSlice 还是修改现有的？**

- **方案 A**：创建新的 EndpointSlice（如 `smg-group-1-ha`），Service 的 `spec.selector` 为空，避免 K8s 自动生成 EndpointSlice
  - 优点：完全隔离，无冲突
  - 缺点：需要确保 Higress 使用 Operator 管理的 EndpointSlice

- **方案 B**：修改 K8s 自动生成的 EndpointSlice
  - 优点：Service 配置标准，无需修改
  - 缺点：可能与 K8s EndpointSlice controller 产生竞争

- **推荐**：方案 A。让 Service 的 `selector` 指向一个永远不匹配的标签（如 `smg-ha-managed: "true"`），这样 K8s 不会自动生成 EndpointSlice。Operator 创建并管理独立的 EndpointSlice，打上 `kubernetes.io/service-name: smg-group-1` 标签，Higress 会自动关联。

**决策 2：Pod IP vs Pod FQDN**

- EndpointSlice 中使用 **Pod IP**（`pod.Status.PodIP`）
- 优点：直接连接，无 DNS 解析开销
- 缺点：Pod 重启后 IP 会变，Operator 需要在下一个 reconcile loop 中更新
-  mitigation：Operator 的 reconcile loop 周期 1s，Pod 重启后很快会更新

**决策 3：Leader Election**

- Operator 部署为 **Deployment with 3 replicas**
- 使用 K8s 的 **leader election** 机制（通过 ConfigMap 或 Lease）
- 只有一个实例是 leader，负责管理 EndpointSlice
- 其他实例 standby，leader 挂了后自动选举新 leader

---

## 六、与 V1/V2 的核心对比

| 维度 | V1 (K8s Readiness) | V2 (EnvoyFilter) | **V4 (Operator)** |
|------|-------------------|------------------|-------------------|
| **主备切换机制** | K8s readiness probe | Envoy active HC | **Operator HTTP 探测** |
| **探测层级** | L4 (TCP) | L7 (HTTP) | **L7 (HTTP)** |
| **切换延迟** | ~6-9s | ~2-4s | **~2-4s** |
| **多备支持** | ❌（脑裂风险） | ✅ | **✅** |
| **配置方式** | K8s 原生 | EnvoyFilter (底层) | **K8s CRD (声明式)** |
| **Istio 耦合** | 无 | 高 | **无** |
| **DestinationRule** | 生效 | 部分失效 | **完全生效** |
| **新增组件** | 无 | 无 | **Operator (需开发)** |
| **运维复杂度** | 低 | 中 | **中** |
| **生产就绪度** | 高 | 中 | **高（开发后）** |
| **调试方式** | `kubectl get endpoints` | `istioctl proxy-config` | **`kubectl get endpointslices`** |

---

## 七、K8s 资源清单

### 7.1 Service（标准 K8s）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: smg-group-1
  namespace: default
  labels:
    app: smg-group-1
    smg-ha-managed: "true"  # Operator 管理的标记
spec:
  type: ClusterIP
  selector:
    app: smg-group-1
    smg-ha-managed: "true"  # 永远不匹配，阻止 K8s 自动生成 EndpointSlice
  ports:
    - name: http
      port: 30002
      targetPort: 30002
      protocol: TCP
```

### 7.2 StatefulSet（标准 K8s）

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: smg-group-1
  namespace: default
spec:
  serviceName: smg-group-1-headless
  replicas: 3
  selector:
    matchLabels:
      app: smg-group-1
  template:
    metadata:
      labels:
        app: smg-group-1
    spec:
      containers:
        - name: smg
          image: lmsysorg/sgl-model-gateway:latest
          ports:
            - containerPort: 30002
          args:
            - --pd-disaggregation
            - --prefill http://prefill-group-1:30001 9001
            - --decode http://decode-group-1:30011
            - --policy cache_aware
```

### 7.3 Higress 配置（标准 Istio，无需 EnvoyFilter）

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: smg-router-vs
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
            host: smg-group-1.default.svc.cluster.local
            port:
              number: 30002
          weight: 33
        - destination:
            host: smg-group-2.default.svc.cluster.local
            port:
              number: 30002
          weight: 33
        - destination:
            host: smg-group-3.default.svc.cluster.local
            port:
              number: 30002
          weight: 34
      retries:
        attempts: 2
        perTryTimeout: 30s
        retryOn: gateway-error,connect-failure,refused-stream

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: smg-router-dr
  namespace: higress-system
spec:
  host: "*.default.svc.cluster.local"
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 512
      http:
        http1MaxPendingRequests: 256
        maxRequestsPerConnection: 100
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 30s
```

**关键变化**：没有 EnvoyFilter！所有配置都是标准 Istio CRD，DestinationRule 的 `outlierDetection`、`connectionPool`、`loadBalancer` 全部生效。

---

## 八、风险与缓解

### 8.1 Operator 自身高可用

**风险**：Operator 挂了，EndpointSlice 不再更新，主挂了无法自动切换。

**缓解**：
- Operator 部署为 **Deployment with 3 replicas**
- 使用 **leader election**，确保只有一个实例在管理 EndpointSlice
- 如果所有 Operator 副本都挂了，EndpointSlice 保持最后一次更新的状态（主或备继续接收流量），不会导致服务中断
- 建议配置 **PodDisruptionBudget**，确保升级时至少保留 1 个副本

### 8.2 Operator 误判

**风险**：Operator 探测到主返回 503（短暂抖动），错误地切换到备。

**缓解**：
- 配置 `failureThreshold: 2`（连续 2 次不健康才切换）
- 配置 `successThreshold: 2`（主恢复后连续 2 次健康才回切）
- 提供 **手动回切** 的 API/命令，运维人员可以干预
- 记录 failover 事件到日志和 metrics，便于审计

### 8.3 与 K8s Endpoint Controller 冲突

**风险**：K8s 的 EndpointSlice controller 会自动为 Service 创建 EndpointSlice，可能与 Operator 管理的冲突。

**缓解**：
- Service 的 `spec.selector` 使用 **永远不匹配的标签**（如 `smg-ha-managed: "true"`）
- 或者：Service 的 `spec.publishNotReadyAddresses: true` + 空 selector
- Operator 创建的 EndpointSlice 使用 `kubernetes.io/service-name: smg-group-1` 标签，Higress 会自动关联
- 给 Operator 管理的 EndpointSlice 打上 `managed-by: smg-ha-operator` 标签，便于区分

### 8.4 开发成本

**风险**：开发 Operator 需要投入人力。

**缓解**：
- 使用 **Kubebuilder** 或 **Operator SDK** 框架，快速搭建脚手架
  ```bash
  kubebuilder init --domain ha.example.com --repo github.com/example/smg-ha-operator
  kubebuilder create api --group smg --version v1 --kind SMGFailoverConfig
  ```
- 核心逻辑简单（HTTP 探测 + EndpointSlice 更新），估计 **1-2 人周** 可完成 MVP
- 也可以使用 **KOPF** (Python) 等轻量框架，进一步降低开发成本
  ```python
  import kopf
  
  @kopf.on.create('smg.ha.example.com', 'v1', 'smgfailoverconfigs')
  def create_fn(spec, name, namespace, logger, **kwargs):
      # 启动 reconcile loop
      pass
  ```

### 8.5 权限管理

**风险**：Operator 需要较高的 RBAC 权限（修改 EndpointSlice、读取 Pod 等）。

**缓解**：
- 使用最小权限原则，只授予必要的权限
- Operator 的 ServiceAccount 只绑定到 `default` namespace（或指定的 namespace）
- 定期审计 RBAC 配置

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: smg-ha-operator
rules:
  - apiGroups: ["discovery.k8s.io"]
    resources: ["endpointslices"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["smg.ha.example.com"]
    resources: ["smgfailoverconfigs"]
    verbs: ["get", "list", "watch"]
```

---

## 九、验证方法

```bash
# 1. 查看 EndpointSlice，确认只包含活跃端点
kubectl get endpointslices -l managed-by=smg-ha-operator

# 2. 查看某个 group 的活跃端点
kubectl get endpointslices smg-group-1-ha -o yaml

# 3. 模拟主节点故障，观察切换
kubectl exec -it smg-group-1-0 -- pkill sgl-model-gateway

# 4. 观察 EndpointSlice 变化
kubectl get endpointslices smg-group-1-ha -w

# 5. 查看 Operator 日志
kubectl logs -l app=smg-ha-operator -f

# 6. 查看 Prometheus metrics
curl http://smg-ha-operator:8080/metrics | grep smg_failover
```

---

## 十、渐进式路径

```
Phase 0: V1（当前）
  └── K8s Service + Readiness Probe
      └── 切换延迟 ~6-9s，仅支持 1主1备

Phase 1: V4 MVP（短期，1-2 周）
  └── 轻量 Operator（Kubebuilder 或 KOPF）
      └── 只支持基本功能：HTTP 探测 + EndpointSlice 更新
      └── 切换延迟 ~2-4s，支持 1主N备
      └── 无 EnvoyFilter，所有 Istio 原生能力生效

Phase 2: V4 生产版（中期，1 个月）
  └── 完整 Operator
      └── CRD 支持丰富的配置（探测参数、回切策略、告警）
      └── Leader election + HA 部署
      └── Prometheus metrics + Grafana dashboard
      └── 回切策略：manual / auto / delayed
      └── 事件日志：failover 事件持久化

Phase 3: V4 + Mesh Sync（长期，可选）
  └── 在 V4 基础上，启用 SMG 的 mesh tree 同步
      └── 主备之间的 cache_aware 树可继承
      └── 消除主恢复后的 cache miss
```

---

## 十一、结论

**V4（K8s Operator + EndpointSlice）是 V2 方案的最佳替代**。

它在保留 V2 核心能力（~2-4s 切换、1主N备）的同时：
- ✅ 完全避免了 EnvoyFilter 的底层 hack
- ✅ 使用标准 K8s API，与 Istio/Higress 版本解耦
- ✅ 所有 DestinationRule 功能完全生效（熔断、连接池、TLS）
- ✅ 声明式配置，运维友好
- ✅ 调试简单，`kubectl get endpointslices` 即可查看当前状态

**唯一代价**：需要投入 1-2 人周开发一个轻量 Operator。这个投入在长期来看是值得的，因为 Operator 的行为完全可控，且可以平滑演进（添加更多功能如自动回切、Prometheus 监控、告警等）。

**推荐决策路径**：
- 如果**时间紧迫**（1 周内上线）：使用 V2（EnvoyFilter），已知可行
- 如果**追求长期可维护性**（1 个月周期）：使用 V4（Operator），标准 K8s API，无技术债
- 如果**对 EnvoyFilter 有顾虑但仍想快速验证**：先用 V2 上线，同时并行开发 V4 Operator，后续平滑迁移

---

## 十二、附录：参考来源

- Kubernetes EndpointSlice API 官方文档  
  https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/

- Kubernetes EndpointSlice API Reference (discovery.k8s.io/v1)  
  https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoint-slice-v1/

- Kubebuilder 官方文档与快速开始  
  https://book.kubebuilder.io/

- controller-runtime (client-go 封装库) GitHub  
  https://github.com/kubernetes-sigs/controller-runtime

- Istio VirtualService API 参考  
  https://istio.io/latest/docs/reference/config/networking/virtual-service/

- Istio DestinationRule API 参考  
  https://istio.io/latest/docs/reference/config/networking/destination-rule/

- Higress 官方文档  
  https://higress.ai/en/docs/latest/overview/what-is-higress/

- Prometheus Go Client (client_golang)  
  https://github.com/prometheus/client_golang

- Envoy Active Health Checking 官方文档  
  https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking

- Envoy `load_assignment` 与 `health_checks` 配置参考  
  https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint.proto

- Kubernetes Controller 开发最佳实践 (Google)  
  https://cloud.google.com/kubernetes-engine/best-practices/operating#controllers

- Higress AI-Proxy 插件 failover 实现源码 (`failover.go`)  
  https://github.com/alibaba/higress/blob/main/plugins/wasm-go/extensions/ai-proxy/provider/failover.go

- Istio EnvoyFilter API 参考  
  https://istio.io/latest/docs/reference/config/networking/envoy-filter/

- Envoy Aggregate Cluster 官方文档  
  https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/aggregate_cluster
