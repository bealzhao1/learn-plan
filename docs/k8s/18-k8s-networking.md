# Kubernetes 网络与服务治理

## 1. K8s 网络模型

### 1.1 基本规则

K8s 网络遵循三条核心规则：
1. **Pod 间通信**：所有 Pod 无需 NAT 即可互相通信
2. **Node 与 Pod 通信**：所有 Node 无需 NAT 即可与 Pod 通信
3. **Pod 自身 IP**：Pod 看到自己的 IP 与其他 Pod 看到的一致

### 1.2 网络层次

```
┌───────────────────────────────────────────────────┐
│ Layer 4: Service (ClusterIP/NodePort/LB)          │
├───────────────────────────────────────────────────┤
│ Layer 3: Pod Network (CNI Plugin)                 │
├───────────────────────────────────────────────────┤
│ Layer 2: Node Network (物理/虚拟网络)              │
└───────────────────────────────────────────────────┘
```

## 2. CNI 插件

### 主流 CNI 对比

| 特性 | Calico | Cilium | Flannel |
|------|--------|--------|---------|
| 数据面 | iptables/eBPF | eBPF | VXLAN/host-gw |
| NetworkPolicy | 完整支持 | 完整支持（L3-L7） | 不支持 |
| 性能 | 高 | 最高 | 中 |
| 加密 | WireGuard | WireGuard/IPsec | 不支持 |
| 可观测性 | 基础 | 强（Hubble） | 无 |
| 复杂度 | 中 | 高 | 低 |
| 适用场景 | 生产通用 | 大规模/安全敏感 | 小型/测试 |

### Calico 网络模式

- **BGP**：Pod IP 直接路由，性能最优，需网络设备支持
- **IPIP**：IP-in-IP 隧道，跨子网场景
- **VXLAN**：L2 over L3 封装，通用性强

## 3. Service 详解

### 3.1 Service 类型

```yaml
# ClusterIP（默认）— 仅集群内部访问
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080

---
# NodePort — 通过节点 IP:Port 暴露
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080    # 范围 30000-32767

---
# LoadBalancer — 云厂商 LB 接入
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 443
    targetPort: 8443
```

### 3.2 Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None      # 关键：设为 None
  selector:
    app: mysql
  ports:
  - port: 3306
```

- 不分配 ClusterIP
- DNS 直接返回所有 Pod IP
- 配合 StatefulSet 使用，每个 Pod 有固定 DNS：`mysql-0.mysql.default.svc.cluster.local`

### 3.3 Service 流量链路

```
Client → kube-proxy (iptables/IPVS) → Endpoint → Pod
```

**iptables 模式**：
- 随机选择后端 Pod
- 规则数量 = O(Services × Endpoints)
- 大规模集群性能下降

**IPVS 模式**：
- 支持多种负载均衡算法（rr/lc/sh 等）
- 性能 O(1)，不随规则数增长
- 推荐大规模集群使用

## 4. Ingress

### 4.1 概念

- 管理集群外部到 Service 的 HTTP/HTTPS 路由
- 需要 Ingress Controller 实现（Nginx/Traefik/Istio Gateway）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
```

### 4.2 Gateway API（新一代）

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
      weight: 90
    - name: api-service-canary
      port: 8080
      weight: 10
```

## 5. NetworkPolicy（网络策略）

### 5.1 默认行为

- 无 NetworkPolicy 时，所有 Pod 间通信不受限制
- 一旦有 NetworkPolicy 选中某 Pod，未明确允许的流量将被拒绝

### 5.2 示例

```yaml
# 只允许前端访问后端的 8080 端口
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# 拒绝所有入站（默认拒绝策略）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}        # 选中所有 Pod
  policyTypes:
  - Ingress
```

## 6. DNS

### CoreDNS

- K8s 默认 DNS 服务
- 为 Service 和 Pod 自动创建 DNS 记录

**DNS 记录格式**：
| 资源类型 | DNS 格式 |
|----------|----------|
| Service | `<svc>.<ns>.svc.cluster.local` |
| Pod | `<pod-ip-dashed>.<ns>.pod.cluster.local` |
| StatefulSet Pod | `<pod-name>.<svc>.<ns>.svc.cluster.local` |

### DNS 策略

```yaml
spec:
  dnsPolicy: ClusterFirst    # 默认，优先集群 DNS
  # None     — 完全自定义
  # Default  — 继承 Node DNS
  # ClusterFirstWithHostNet — hostNetwork Pod 用集群 DNS
```

## 7. Service Mesh（服务网格）

### Istio 架构

```
┌─────────────────────────────────────────────┐
│ Control Plane (istiod)                       │
│  Pilot + Citadel + Galley                   │
└─────────────────────────────────────────────┘
        │ xDS API
        ▼
┌───────────────┐    ┌───────────────┐
│ Pod A         │    │ Pod B         │
│ ┌───────────┐ │    │ ┌───────────┐ │
│ │    App    │ │    │ │    App    │ │
│ └─────┬─────┘ │    │ └─────┬─────┘ │
│ ┌─────┴─────┐ │    │ ┌─────┴─────┐ │
│ │  Envoy    │◄┼────┼►│  Envoy    │ │
│ │ (Sidecar) │ │    │ │ (Sidecar) │ │
│ └───────────┘ │    │ └───────────┘ │
└───────────────┘    └───────────────┘
```

**核心能力**：
- 流量管理（路由、重试、超时、熔断）
- 安全通信（mTLS 自动加密）
- 可观测性（分布式追踪、指标、访问日志）

## 8. 面试高频问题

| 问题 | 核心答案 |
|------|----------|
| Service 和 Ingress 的区别 | Service 是 L4 负载均衡，Ingress 是 L7 HTTP 路由 |
| iptables 和 IPVS 怎么选 | 小集群 iptables 够用，大规模（>1000 Service）用 IPVS |
| Pod 无法访问 Service | 检查 DNS → Endpoint → kube-proxy → NetworkPolicy |
| 跨 Namespace 访问 Service | 使用完整 DNS：`svc.namespace.svc.cluster.local` |
| 为什么需要 Service Mesh | 统一处理服务间通信（重试/熔断/加密/追踪），与业务解耦 |
