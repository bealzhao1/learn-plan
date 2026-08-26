# Kubernetes 存储与配置管理

## 1. 存储架构

### 1.1 存储层次

```
┌──────────────────────────────────────────────┐
│ Pod (Volume Mount)                           │
├──────────────────────────────────────────────┤
│ PersistentVolumeClaim (PVC) — 用户申请       │
├──────────────────────────────────────────────┤
│ PersistentVolume (PV) — 集群级存储资源        │
├──────────────────────────────────────────────┤
│ StorageClass — 动态供给策略                   │
├──────────────────────────────────────────────┤
│ CSI Driver — 存储插件接口                     │
├──────────────────────────────────────────────┤
│ 底层存储 (云盘/NFS/Ceph/本地SSD)              │
└──────────────────────────────────────────────┘
```

### 1.2 Volume 类型

| 类型 | 生命周期 | 适用场景 |
|------|----------|----------|
| emptyDir | 随 Pod 销毁 | 临时缓存、同 Pod 容器间共享 |
| hostPath | 随 Node 存在 | 日志收集、DaemonSet |
| PVC | 独立于 Pod | 持久化数据（数据库等） |
| ConfigMap/Secret | 独立于 Pod | 配置注入 |
| projected | 独立于 Pod | 多来源合并挂载 |

## 2. PV & PVC

### 2.1 静态供给

```yaml
# 管理员创建 PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany         # RWX
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 10.0.0.100
    path: /data/share

---
# 用户创建 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
```

### 2.2 动态供给（StorageClass）

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: disk.csi.cloud.com
parameters:
  type: ssd
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer  # 延迟绑定到调度节点
allowVolumeExpansion: true               # 允许扩容

---
# PVC 引用 StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
spec:
  storageClassName: fast-ssd  # 指定 StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### 2.3 访问模式

| 模式 | 缩写 | 说明 |
|------|------|------|
| ReadWriteOnce | RWO | 单节点读写 |
| ReadOnlyMany | ROX | 多节点只读 |
| ReadWriteMany | RWX | 多节点读写 |
| ReadWriteOncePod | RWOP | 单 Pod 读写（K8s 1.27+） |

### 2.4 回收策略

| 策略 | 行为 |
|------|------|
| Retain | PVC 删除后保留 PV 和数据，需手动清理 |
| Delete | PVC 删除后自动删除 PV 和底层存储 |
| Recycle | （已废弃）清空数据后重新可用 |

## 3. CSI（Container Storage Interface）

### 3.1 CSI 架构

```
┌─────────────┐     ┌─────────────────────────────────┐
│ kubelet     │     │ CSI Driver                       │
│             │     │ ┌─────────────┐ ┌─────────────┐ │
│ ┌─────────┐ │     │ │ Node Plugin │ │ Controller  │ │
│ │ Volume  │◄┼─────┼►│ (DaemonSet) │ │ (Deployment)│ │
│ │ Manager │ │     │ └─────────────┘ └─────────────┘ │
│ └─────────┘ │     └─────────────────────────────────┘
└─────────────┘
```

### 3.2 CSI 操作流程

```
CreateVolume → ControllerPublishVolume → NodeStageVolume → NodePublishVolume
                                                                   │
                                                              容器可用
```

## 4. ConfigMap

### 4.1 创建方式

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # 键值对
  database_host: "mysql.default.svc"
  database_port: "3306"
  # 文件内容
  config.yaml: |
    server:
      port: 8080
      mode: production
    logging:
      level: info
      format: json
```

### 4.2 使用方式

```yaml
spec:
  containers:
  - name: app
    env:
    # 方式一：环境变量（单个 key）
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host
    # 方式二：环境变量（所有 key）
    envFrom:
    - configMapRef:
        name: app-config
    # 方式三：Volume 挂载为文件
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/config.yaml
      subPath: config.yaml
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

### 4.3 热更新

- **Volume 挂载**：ConfigMap 更新后，文件会自动更新（约 30-60s）
- **环境变量**：不会自动更新，需重启 Pod
- **subPath 挂载**：不会自动更新

## 5. Secret

### 5.1 Secret 类型

| 类型 | 用途 |
|------|------|
| Opaque | 通用密钥 |
| kubernetes.io/tls | TLS 证书 |
| kubernetes.io/dockerconfigjson | 镜像拉取凭证 |
| kubernetes.io/service-account-token | SA Token |

### 5.2 安全注意事项

- Secret 默认 Base64 编码，**不是加密**
- 启用 etcd 加密（EncryptionConfiguration）
- 使用外部密钥管理（Vault/AWS KMS/Sealed Secrets）
- RBAC 限制 Secret 访问权限
- 避免在 Git 中存储 Secret 明文

```yaml
# etcd 加密配置
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-key>
  - identity: {}
```

## 6. 实战：有状态应用存储方案

### MySQL 主从 + PVC

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:8.0
        command: ["bash", "-c", "... 初始化脚本 ..."]
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d
      volumes:
      - name: config
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: fast-ssd
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 50Gi
```

## 7. 面试高频问题

| 问题 | 核心答案 |
|------|----------|
| PV 和 PVC 的关系 | PV 是存储资源，PVC 是用户的存储申请，通过绑定关联 |
| StorageClass 的作用 | 定义动态供给策略，自动创建 PV |
| ConfigMap 更新 Pod 能感知吗 | Volume 挂载可以（有延迟），环境变量不行 |
| Secret 安全吗 | 默认只是 Base64 编码，需配合 etcd 加密和 RBAC |
| emptyDir 数据会丢失吗 | Pod 销毁即丢失，仅用于临时数据 |
| WaitForFirstConsumer 的好处 | 避免 PV 和 Pod 调度到不同可用区 |
