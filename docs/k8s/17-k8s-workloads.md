# Kubernetes 工作负载与部署策略

## 1. 工作负载资源类型

### 1.1 Deployment（无状态应用）

**适用场景**：Web 服务、API 服务、无状态微服务

**核心能力**：
- 声明式副本管理
- 滚动更新与回滚
- 暂停/恢复部署
- 扩缩容

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: myapp:v2.0
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
```

### 1.2 StatefulSet（有状态应用）

**适用场景**：数据库、消息队列、分布式存储

**与 Deployment 的区别**：
| 特性 | Deployment | StatefulSet |
|------|-----------|-------------|
| Pod 名称 | 随机后缀 | 有序编号（0, 1, 2...） |
| 网络标识 | 不固定 | 固定 DNS（`pod-0.svc`） |
| 存储 | 共享 PVC | 每个 Pod 独立 PVC |
| 创建/删除顺序 | 并行 | 有序（顺序创建，逆序删除） |
| 扩容/缩容 | 任意 | 有序 |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 1.3 DaemonSet（守护进程）

**适用场景**：日志收集（Fluentd）、监控（Node Exporter）、网络插件（Calico）

- 确保每个 Node（或符合条件的 Node）运行一个 Pod
- 新 Node 加入自动部署，Node 删除自动清理

### 1.4 Job & CronJob（批处理）

**Job**：运行一次性任务，保证成功完成

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  backoffLimit: 3        # 失败重试次数
  completions: 1         # 需要成功完成的 Pod 数
  parallelism: 1         # 并行度
  template:
    spec:
      containers:
      - name: migrate
        image: migration:v1
      restartPolicy: Never
```

**CronJob**：定时任务

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # 每天凌晨 2 点
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:v1
          restartPolicy: OnFailure
```

## 2. 部署策略

### 2.1 滚动更新（Rolling Update）

**默认策略**，逐步替换旧 Pod：

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # 最多多创建 1 个 Pod
    maxUnavailable: 0    # 不允许不可用（保证流量无损）
```

**执行过程**：
```
v1: [Pod1] [Pod2] [Pod3]
→   [Pod1] [Pod2] [Pod3] [Pod4-v2]    # 创建新版本 Pod
→   [Pod2] [Pod3] [Pod4-v2]           # 删除旧版本 Pod1
→   [Pod2] [Pod3] [Pod4-v2] [Pod5-v2] # 继续创建
→   ...直到全部替换完成
```

### 2.2 蓝绿部署（Blue-Green）

通过切换 Service Selector 实现：

```yaml
# 蓝色环境（当前版本）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
        version: blue
---
# 绿色环境（新版本）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
        version: green
---
# Service 指向蓝色 → 切换到绿色
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: green  # 切换这里
```

### 2.3 金丝雀发布（Canary）

逐步将流量导向新版本：

**方式一**：调整副本数比例
```
v1 Deployment: replicas=9 (90% 流量)
v2 Deployment: replicas=1 (10% 流量)
```

**方式二**：使用 Istio VirtualService 精确控制

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  http:
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90
    - destination:
        host: myapp
        subset: v2
      weight: 10
```

## 3. 健康检查（Probe）

### 三种探针

| 探针 | 作用 | 失败行为 |
|------|------|----------|
| Liveness | 判断容器是否存活 | 重启容器 |
| Readiness | 判断容器是否就绪 | 从 Service Endpoint 移除 |
| Startup | 判断容器是否启动完成 | 阻塞其他探针 |

### 探测方式

- **HTTP GET**：发送 HTTP 请求，2xx/3xx 为成功
- **TCP Socket**：TCP 连接成功即通过
- **Exec**：执行命令，返回码 0 为成功
- **gRPC**：gRPC 健康检查协议

### 最佳实践

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30    # 给应用启动时间
  periodSeconds: 10          # 检查间隔
  failureThreshold: 3        # 连续失败 3 次才重启
  timeoutSeconds: 5          # 超时时间

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

## 4. 资源管理

### 4.1 Resource Requests & Limits

```yaml
resources:
  requests:              # 调度依据（保证能获得的资源）
    cpu: "250m"          # 0.25 核
    memory: "512Mi"
  limits:                # 上限（超限被 OOMKill 或 CPU 节流）
    cpu: "1000m"         # 1 核
    memory: "1Gi"
```

### 4.2 QoS 等级

| 等级 | 条件 | OOM 优先级 |
|------|------|-----------|
| Guaranteed | requests == limits（所有容器） | 最后被杀 |
| Burstable | 至少一个容器设置了 requests | 中间 |
| BestEffort | 没有设置任何 requests/limits | 最先被杀 |

### 4.3 HPA（水平自动扩缩容）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## 5. Pod Disruption Budget (PDB)

保证服务可用性：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2        # 至少保持 2 个 Pod 可用
  # 或 maxUnavailable: 1 # 最多 1 个不可用
  selector:
    matchLabels:
      app: api-server
```

## 6. 面试高频问题

| 问题 | 核心答案 |
|------|----------|
| 滚动更新如何保证零停机 | maxUnavailable=0 + Readiness Probe |
| StatefulSet 为什么需要 Headless Service | 为每个 Pod 提供固定 DNS 记录 |
| 如何实现灰度发布 | 金丝雀（副本比例或 Istio 权重） |
| HPA 的扩容有延迟怎么办 | 预设足够的 minReplicas + KEDA 自定义指标 |
| Pod 被 OOMKill 怎么排查 | kubectl describe pod → 看 Last State → 调整 limits |
