# Kubernetes 架构与核心概念

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Control Plane                           │
│  ┌──────────┐ ┌──────────────┐ ┌──────────┐ ┌───────────┐  │
│  │ API Server│ │ Controller   │ │ Scheduler│ │   etcd    │  │
│  │  (入口)   │ │ Manager      │ │ (调度器)  │ │ (状态存储) │  │
│  └──────────┘ └──────────────┘ └──────────┘ └───────────┘  │
└─────────────────────────────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│    Node 1    │  │    Node 2    │  │    Node 3    │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │  kubelet │ │  │ │  kubelet │ │  │ │  kubelet │ │
│ │kube-proxy│ │  │ │kube-proxy│ │  │ │kube-proxy│ │
│ │Container │ │  │ │Container │ │  │ │Container │ │
│ │ Runtime  │ │  │ │ Runtime  │ │  │ │ Runtime  │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
└──────────────┘  └──────────────┘  └──────────────┘
```

## 2. Control Plane 组件

### 2.1 API Server (kube-apiserver)

- 集群的统一入口，所有操作都通过 REST API 进行
- 负责认证（Authentication）、授权（Authorization）、准入控制（Admission Control）
- 唯一直接操作 etcd 的组件

**请求处理流程**：
```
kubectl → Authentication → Authorization → Admission → Validation → etcd
```

### 2.2 etcd

- 分布式键值存储，保存集群所有状态数据
- 基于 Raft 协议保证一致性
- 只有 API Server 直接读写 etcd

**关键特性**：
| 特性 | 说明 |
|------|------|
| Watch 机制 | 支持事件监听，组件通过 watch 获取变更 |
| MVCC | 多版本并发控制，支持历史版本查询 |
| Lease | 租约机制，用于临时数据过期 |
| 选举 | 基于 Raft 的 Leader 选举 |

### 2.3 Scheduler (kube-scheduler)

- 监听未调度的 Pod，为其选择最优 Node
- 调度流程分为 **预选（Filtering）** 和 **优选（Scoring）** 两阶段

**调度流程**：
```
新 Pod 创建 → 过滤不满足条件的 Node → 对候选 Node 评分 → 绑定到最优 Node
```

**常见调度策略**：
- `NodeSelector` — 标签匹配
- `NodeAffinity` — 节点亲和性（软/硬约束）
- `PodAffinity/AntiAffinity` — Pod 间亲和/反亲和
- `Taints/Tolerations` — 污点与容忍
- `TopologySpreadConstraints` — 拓扑分散约束

### 2.4 Controller Manager

- 运行各种控制器的管理进程
- 通过控制循环（Reconcile Loop）确保实际状态趋近期望状态

**核心控制器**：
| 控制器 | 职责 |
|--------|------|
| Deployment Controller | 管理 ReplicaSet，实现滚动更新 |
| ReplicaSet Controller | 维护 Pod 副本数 |
| StatefulSet Controller | 管理有状态应用 |
| DaemonSet Controller | 确保每个节点运行一个 Pod |
| Job/CronJob Controller | 管理批处理任务 |
| Node Controller | 监控节点健康状态 |
| Service Controller | 管理 LoadBalancer 类型 Service |

## 3. Node 组件

### 3.1 kubelet

- 每个 Node 上的代理进程
- 负责管理 Pod 的生命周期
- 执行健康检查（Liveness / Readiness / Startup Probe）
- 向 API Server 汇报节点状态

### 3.2 kube-proxy

- 实现 Service 的网络规则
- 支持三种代理模式：
  - **iptables**（默认）：基于 iptables 规则转发
  - **IPVS**：基于虚拟服务器，性能更优
  - **userspace**：早期模式，已不推荐

### 3.3 Container Runtime

- 负责运行容器的底层引擎
- 支持 CRI 标准接口
- 主流实现：containerd、CRI-O

## 4. 核心资源对象

### 4.1 Pod

- K8s 最小调度单元
- 一个 Pod 内可有多个容器，共享网络和存储
- 生命周期：Pending → Running → Succeeded/Failed

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

### 4.2 Deployment

- 管理无状态应用的声明式更新
- 支持滚动更新、回滚、暂停/恢复
- 通过 ReplicaSet 管理 Pod 副本

**滚动更新策略**：
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # 最多额外创建的 Pod 数
    maxUnavailable: 25%  # 最多不可用的 Pod 数
```

### 4.3 Service

- 为一组 Pod 提供稳定的网络入口
- 类型：ClusterIP / NodePort / LoadBalancer / ExternalName

**Service 发现机制**：
- DNS（推荐）：`<service>.<namespace>.svc.cluster.local`
- 环境变量：自动注入同 namespace 的 Service 地址

### 4.4 ConfigMap & Secret

- ConfigMap：管理非敏感配置
- Secret：管理敏感数据（Base64 编码，非加密）
- 挂载方式：环境变量 / Volume 文件

### 4.5 PersistentVolume (PV) & PersistentVolumeClaim (PVC)

- PV：集群级别的存储资源
- PVC：用户的存储请求
- StorageClass：动态供给存储

```
StorageClass → 动态创建 PV → PVC 绑定 PV → Pod 挂载 PVC
```

## 5. 核心设计原则

### 5.1 声明式 API

```
用户声明期望状态 → Controller 对比实际状态 → 执行调和（Reconcile）→ 达到期望状态
```

### 5.2 控制循环（Reconcile Loop）

```go
for {
    desired := getDesiredState()   // 从 API Server 获取期望状态
    current := getCurrentState()   // 获取实际状态
    diff := compare(desired, current)
    if diff != nil {
        reconcile(diff)            // 执行调和操作
    }
}
```

### 5.3 Label & Selector

- Label：键值对标签，附加在资源上
- Selector：基于标签选择资源
- 松耦合的关联机制（Service → Pod、Deployment → ReplicaSet）

## 6. 面试高频问题

| 问题 | 核心答案 |
|------|----------|
| Pod 创建的完整流程 | kubectl → API Server → etcd → Scheduler → kubelet → Container Runtime |
| Pod 重启策略 | Always / OnFailure / Never |
| Service 和 Endpoint 的关系 | Service 通过 Selector 自动关联 Pod 到 Endpoint |
| 如何实现零停机部署 | 滚动更新 + Readiness Probe + PDB |
| etcd 数据丢失怎么办 | 定期快照备份 + 从快照恢复 |
| 为什么不建议在 K8s 上跑数据库 | 网络开销、存储复杂性、运维成本高 |
