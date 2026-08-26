# Kubernetes 安全与运维实践

## 1. RBAC（基于角色的访问控制）

### 1.1 RBAC 模型

```
User/ServiceAccount → RoleBinding → Role → Resources (Verbs)
                   → ClusterRoleBinding → ClusterRole → Cluster Resources
```

### 1.2 核心资源

| 资源 | 作用域 | 说明 |
|------|--------|------|
| Role | Namespace | 定义 namespace 内的权限 |
| ClusterRole | Cluster | 定义集群级别的权限 |
| RoleBinding | Namespace | 将 Role 绑定给用户/组 |
| ClusterRoleBinding | Cluster | 将 ClusterRole 绑定给用户/组 |

### 1.3 示例

```yaml
# 只读权限 Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]

---
# 绑定到 ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 1.4 最小权限原则

- 避免使用 `cluster-admin`
- 精确指定 resources 和 verbs
- 使用 namespace 隔离团队权限
- 定期审计 RBAC 配置

## 2. Pod 安全

### 2.1 SecurityContext

```yaml
spec:
  securityContext:              # Pod 级别
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    securityContext:            # 容器级别
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

### 2.2 Pod Security Standards (PSS)

三个安全级别：

| 级别 | 说明 | 限制 |
|------|------|------|
| Privileged | 无限制 | 无 |
| Baseline | 基本限制 | 禁止特权容器、hostNetwork 等 |
| Restricted | 严格限制 | 必须非 root、只读文件系统、drop ALL capabilities |

```yaml
# Namespace 级别强制执行
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

### 2.3 网络安全

- NetworkPolicy 限制 Pod 间通信
- mTLS（通过 Service Mesh）加密通信
- Ingress TLS 终止

## 3. 监控与可观测性

### 3.1 监控栈

```
┌────────────────────────────────────────────────┐
│ 展示层：Grafana Dashboard                       │
├────────────────────────────────────────────────┤
│ 告警层：Alertmanager → 通知渠道                  │
├────────────────────────────────────────────────┤
│ 查询层：PromQL                                  │
├────────────────────────────────────────────────┤
│ 存储层：Prometheus TSDB / Thanos                │
├────────────────────────────────────────────────┤
│ 采集层：metrics-server / kube-state-metrics     │
│         / node-exporter / cadvisor              │
└────────────────────────────────────────────────┘
```

### 3.2 关键指标

| 维度 | 指标 | 说明 |
|------|------|------|
| 节点 | cpu/memory/disk | 资源使用率 |
| Pod | container_cpu_usage_seconds | 容器 CPU 使用 |
| Pod | container_memory_working_set_bytes | 容器内存 |
| API Server | apiserver_request_total | API 请求量 |
| etcd | etcd_server_has_leader | 是否有 Leader |
| 调度 | scheduler_pending_pods | 等待调度的 Pod |

### 3.3 日志方案

```
应用日志 → stdout/stderr → 容器运行时 → 节点文件
                                         │
                           ┌─────────────┼─────────────┐
                           ▼             ▼             ▼
                    DaemonSet        Sidecar       直接推送
                   (Fluentd/         (每个Pod     (应用内SDK)
                    Fluent Bit)     一个日志容器)
                           │             │             │
                           └─────────────┼─────────────┘
                                         ▼
                              Elasticsearch / Loki
                                         │
                                         ▼
                                Kibana / Grafana
```

## 4. 故障排查

### 4.1 Pod 故障排查流程

```
kubectl get pods → 检查状态
    │
    ├── Pending → kubectl describe pod → 看 Events
    │   ├── 资源不足 → 扩容 Node 或调整 requests
    │   ├── 镜像拉取失败 → 检查 imagePullSecret
    │   └── 调度失败 → 检查 nodeSelector/affinity/taint
    │
    ├── CrashLoopBackOff → kubectl logs pod
    │   ├── 应用错误 → 修复代码
    │   ├── 配置错误 → 检查 ConfigMap/Secret
    │   └── 健康检查失败 → 调整 Probe 参数
    │
    └── Running 但异常 → kubectl exec -it pod -- sh
        ├── 网络不通 → 检查 Service/DNS/NetworkPolicy
        └── 性能问题 → 检查资源限制/日志
```

### 4.2 常用排查命令

```bash
# 查看 Pod 详情和事件
kubectl describe pod <name>

# 查看日志（当前 + 上一次）
kubectl logs <pod> --previous
kubectl logs <pod> -c <container>

# 进入容器调试
kubectl exec -it <pod> -- /bin/sh

# 临时调试容器（K8s 1.25+）
kubectl debug -it <pod> --image=busybox --target=<container>

# 查看资源使用
kubectl top pods
kubectl top nodes

# 检查 Endpoint
kubectl get endpoints <service>

# DNS 调试
kubectl run debug --rm -it --image=busybox -- nslookup <service>

# 网络连通性测试
kubectl run debug --rm -it --image=nicolaka/netshoot -- curl <service>:port
```

## 5. 集群升级

### 5.1 升级策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| 原地升级 | 逐节点升级 Control Plane + Node | 小型集群 |
| 蓝绿升级 | 创建新集群，迁移工作负载 | 大型/关键集群 |
| 滚动升级 | 逐批升级节点 | 托管 K8s（EKS/GKE/TKE） |

### 5.2 节点升级流程

```bash
# 1. 标记节点不可调度
kubectl cordon <node>

# 2. 驱逐 Pod（尊重 PDB）
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# 3. 升级 kubelet 和 kube-proxy
# (操作系统级别)

# 4. 重新标记为可调度
kubectl uncordon <node>
```

## 6. GitOps 与 CI/CD

### 6.1 GitOps 流程

```
开发者 Push → Git Repo → ArgoCD/Flux 监听 → 自动同步到集群
                              │
                    对比 Git 声明 vs 集群实际状态
                              │
                    不一致则自动/手动 Reconcile
```

### 6.2 ArgoCD 核心概念

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # 自动清理多余资源
      selfHeal: true     # 自动修复漂移
```

### 6.3 镜像更新策略

| 方案 | 原理 | 优点 |
|------|------|------|
| CI 推送 Manifest | CI 更新 Git 中的镜像 tag | 简单直观 |
| Image Updater | 监听镜像仓库，自动更新 tag | 解耦 CI/CD |
| Kustomize overlay | 不同环境不同覆盖 | 多环境管理 |

## 7. 生产环境 Checklist

### 7.1 资源管理
- [ ] 所有 Pod 设置 requests 和 limits
- [ ] 配置 LimitRange 和 ResourceQuota
- [ ] 启用 HPA/VPA

### 7.2 高可用
- [ ] Control Plane 多副本（3/5 节点）
- [ ] 跨可用区部署（TopologySpreadConstraints）
- [ ] 配置 PDB

### 7.3 安全
- [ ] 启用 RBAC，最小权限原则
- [ ] Pod 以非 root 运行
- [ ] NetworkPolicy 限制通信
- [ ] Secret 加密存储
- [ ] 镜像签名验证

### 7.4 可观测性
- [ ] 监控覆盖（Prometheus + Grafana）
- [ ] 集中日志（Loki/ELK）
- [ ] 分布式追踪（Jaeger/Tempo）
- [ ] 关键告警配置

### 7.5 备份与恢复
- [ ] etcd 定期快照
- [ ] PV 数据备份（Velero）
- [ ] 灾备演练

## 8. 面试高频问题

| 问题 | 核心答案 |
|------|----------|
| RBAC 和 ABAC 的区别 | RBAC 基于角色聚合权限，ABAC 基于属性，RBAC 更灵活易管理 |
| 如何保证集群安全 | RBAC + NetworkPolicy + PSS + Secret 加密 + 审计日志 |
| 如何实现零停机升级集群 | cordon → drain → upgrade → uncordon，配合 PDB |
| GitOps 的优势 | 声明式、可审计、自动检测漂移、Git 作为唯一真相源 |
| Pod OOMKilled 怎么办 | 检查 memory limits → 分析内存泄漏 → 调整 limits 或修复代码 |
| 如何排查 Pod 一直 Pending | describe pod 看 Events → 检查资源/节点/调度约束 |
