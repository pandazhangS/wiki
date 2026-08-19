# S3 共享存储方案（多 Pod 共享 PV）

> 状态: 设计稿 v1
> 日期: 2026-08-19
> 范围: 多个 Pod 通过 S3 共享同一份数据（配置、模型、日志等），含动态制备与静态制备两种方式

## 1. 背景与目标

K8s 中多个 Pod 需要共享同一份数据时，常规做法是让多个 Pod 挂载同一个 PV。S3 天然支持多节点并发读写（`ReadWriteMany`），是共享存储的理想后端。

设计目标:

1. **多 Pod 共享**: 多个 Deployment / 多个副本挂载同一份 S3 数据，读写一致。
2. **两种制备方式**: 支持动态制备（StorageClass 自动建 PV）与静态制备（手动建 PV 指向已有 bucket）。
3. **通用 S3 兼容**: 不绑定单一云厂商，支持 AWS S3、MinIO、Ceph RGW、阿里云 OSS 等。

## 2. 前置知识:PV 访问模式与 S3

| AccessMode | 含义 | S3 支持 |
|---|---|---|
| `ReadWriteOnce` (RWO) | 单节点读写 | 不适用（S3 天然多节点） |
| `ReadOnlyMany` (ROX) | 多节点只读 | ✅ |
| `ReadWriteMany` (RWX) | 多节点读写 | ✅（S3 默认） |
| `ReadWriteOncePod` (RWOP) | 单 Pod | 不适用 |

S3 是对象存储，天然支持多节点并发访问，因此 S3 CSI 卷的 `accessModes` 一律为 `ReadWriteMany`。这正是"多个 Pod 共享一个 PV"的完美落地方式。

## 3. 驱动选型

| 驱动 | 说明 | 适用 |
|---|---|---|
| **Mountpoint S3 CSI**（AWS 官方） | 基于 Mountpoint for S3，性能好 | AWS EKS + S3 |
| **csi-s3**（ctrox，rclone 后端） | 通用 S3 兼容，支持 MinIO/OSS/Ceph | 自建 MinIO / 混合云 |

> 本文以通用 **csi-s3** 为例（支持任意 S3 兼容存储），AWS 场景可替换为 Mountpoint S3 CSI，配置结构类似。

## 4. 动态制备（Dynamic Provisioning）

通过 StorageClass 自动创建 PV，PVC 声明即用。适用于 bucket 由驱动自动管理、无需预先存在的场景。

### 4.1 安装驱动

```bash
helm repo add csi-s3 https://ctrox.github.io/csi-s3
helm install csi-s3 csi-s3/csi-s3 --namespace kube-system
```

### 4.2 创建连接 Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-s3-secret
type: Opaque
stringData:
  accessKeyID: "minioadmin"
  secretAccessKey: "minioadmin"
  endpoint: "http://minio:9000"     # MinIO / S3 兼容端点
  region: "us-east-1"
```

### 4.3 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-s3
provisioner: ch.ctrox.csi.s3-driver
parameters:
  csi.storage.k8s.io/provisioner-secret-name: csi-s3-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-publish-secret-name: csi-s3-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: default
  csi.storage.k8s.io/node-publish-secret-name: csi-s3-secret
  csi.storage.k8s.io/node-publish-secret-namespace: default
  mounter: rclone
```

### 4.4 PVC（动态创建）

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-s3-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: csi-s3
  resources:
    requests:
      storage: 10Gi
```

### 4.5 多个 Pod 挂载同一个 PVC

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: nginx
          volumeMounts:
            - name: shared-data
              mountPath: /data
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: shared-s3-pvc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-b
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: app
          image: nginx
          volumeMounts:
            - name: shared-data
              mountPath: /data
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: shared-s3-pvc
```

> 多个 Deployment 的 Pod 全部挂载同一个 `shared-s3-pvc`，读写同一份 S3 数据。

## 5. 静态制备（Static Provisioning）★ 补充

**静态制备**指：**不通过 StorageClass 动态创建 PV**，而是手动创建一个 PV 直接指向**已存在的 S3 bucket**，再由 PVC 绑定到这个 PV。

适用场景:
- bucket 已由运维预先创建（如生产环境 bucket 命名规范、跨环境复用）。
- 需要精确控制 bucket 名、路径前缀、挂载选项。
- 不想让驱动自动建 bucket（动态制备默认会创建 bucket）。

### 5.1 手动创建 PV（指向已有 bucket）

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: shared-s3-pv
spec:
  capacity:
    storage: 100Gi          # 仅作配额声明，S3 实际无上限
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain   # 静态 PV 建议 Retain，避免误删 bucket 数据
  csi:
    driver: ch.ctrox.csi.s3-driver
    volumeHandle: shared-s3-bucket        # 唯一标识，对应 bucket 名
    volumeAttributes:
      bucket: shared-s3-bucket            # 已存在的 bucket 名
      mounter: rclone
      options: "--dir-cache-time 60s"     # 可选挂载选项
    nodePublishSecretRef:
      name: csi-s3-secret                 # 连接凭证
      namespace: default
```

> 关键点:
> - `volumeHandle` 是 PV 的唯一标识，需与 bucket 对应。
> - `volumeAttributes.bucket` 指定**已存在**的 bucket 名，驱动不会自动创建。
> - `persistentVolumeReclaimPolicy: Retain`：静态 PV 删除时保留底层 bucket 数据，防止误删。

### 5.2 创建 PVC 绑定到该 PV

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-s3-pvc
spec:
  accessModes:
    - ReadWriteMany
  volumeName: shared-s3-pv     # 显式指定要绑定的 PV
  resources:
    requests:
      storage: 100Gi
```

> 静态制备时，PVC 通过 `spec.volumeName` 显式绑定到指定 PV，不经过 StorageClass。

### 5.3 多个 Pod 挂载（与动态制备相同）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-a
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: nginx
          volumeMounts:
            - name: shared-data
              mountPath: /data
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: shared-s3-pvc
```

### 5.4 动态 vs 静态制备对比

| 维度 | 动态制备 | 静态制备 |
|---|---|---|
| PV 创建 | StorageClass 自动创建 | 手动创建 |
| bucket 来源 | 驱动自动创建 | 预先存在 |
| 绑定方式 | PVC 声明 storageClass | PVC 显式 `volumeName` |
| 适用场景 | 快速开发、bucket 无规范 | 生产 bucket 规范、跨环境复用 |
| 回收策略 | 默认 Delete | 建议 Retain |
| 控制粒度 | 低 | 高（可精确控制 bucket/路径/选项） |

## 6. 认证方式

### 6.1 静态凭证（Secret，上文已用）

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-s3-secret
type: Opaque
stringData:
  accessKeyID: "minioadmin"
  secretAccessKey: "minioadmin"
  endpoint: "http://minio:9000"
  region: "us-east-1"
```

### 6.2 IRSA（AWS EKS 推荐）

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-csi-role
```

## 7. 注意事项

1. **并发写一致性**: RWX 只是"能同时挂载"，不保证数据一致性。多个 Pod 同时写同一文件需应用层处理（文件锁、分布式锁）。
2. **性能**: S3 对象存储比本地盘慢，且是单点瓶颈；高频小文件读写场景需评估。
3. **rclone 缓存一致性**: csi-s3 基于 rclone，有缓存一致性窗口，多写场景需注意 `--dir-cache-time` 等选项。
4. **bucket 命名**: 静态制备时 bucket 必须已存在，且凭证需有对应读写权限。
5. **回收策略**: 静态 PV 务必用 `Retain`，避免删除 PV 时连带清空 bucket 数据。

## 8. 备选:应用内直接用 S3 SDK

如果只是共享数据、不需要 POSIX 文件系统语义，很多场景直接用 S3 SDK 更简单，无需 CSI 组件:

```python
import boto3
s3 = boto3.client('s3', endpoint_url='http://minio:9000')
s3.upload_file('/local/file', 'my-bucket', 'key')
```

| 方案 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|
| **S3 CSI（动态/静态）** | 文件系统语义、多 Pod 共享、无改代码 | 需装 CSI 驱动、性能一般 | 需要挂载路径、多 Pod 共享 |
| **应用内 SDK** | 最简单、无额外组件 | 无文件系统语义、需改代码 | 对象存储场景、无需挂载 |

## 9. 待确认项

1. 底层 S3 兼容存储是哪种？（AWS S3 / MinIO / Ceph RGW / 阿里云 OSS）——决定驱动选型与 endpoint。
2. 是否需要 POSIX 文件系统语义？（决定用 CSI 还是应用内 SDK）
3. bucket 是否已预先存在？（决定动态还是静态制备）
4. 多 Pod 是否并发写同一文件？（决定是否需要应用层锁）
