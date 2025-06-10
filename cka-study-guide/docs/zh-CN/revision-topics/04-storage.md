# 存储

## 理解存储类和持久卷

`StorageClass` 为管理员提供了一种描述所提供存储“类别”的方式。不同的类别可能对应不同的服务质量等级、备份策略，或由集群管理员决定的任意策略。Kubernetes 本身并不关心这些类别具体代表什么。在其他存储系统中，这一概念有时被称为“配置文件（profiles）”。

下面是一个 StorageClass 的示例：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

关键声明如下：

* `provisioner` : 决定使用哪个卷插件。通常对应一个云服务商及其提供的特定存储服务。本例中为 AWS Elastic Block Store。

* `parameters` : 描述该存储类别在底层 provisioner 上的特性。本例中， `type` 为 `gp2`，在 AWS 术语中表示通用型 SSD。其他类型包括 `IO1`（预配置 IOPS）、`ST1`（吞吐优化型）和 `STC`（冷存储）。

单独的 `storageclass` 对象仅定义了被调用时要创建什么存储。它本身不会做任何事情。

`persistentvolume` 对象可用于从 `storageclass` 请求存储，通常作为 pod 清单的一部分。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: standard
```

## 理解卷的模式、访问模式和回收策略

### 卷模式

仅有两种:

* `block` - 作为原始块设备挂载到 Pod 上，*没有*文件系统。Pod 或应用需要能够处理原始块设备。以这种方式挂载可以获得更好的性能，但会增加复杂性。

* `filesystem` - 挂载到 Pod 的文件系统中的某个目录下。如果卷由没有文件系统的块设备支持，Kubernetes 会自动创建一个文件系统。与 `block` 设备相比，这种方式兼容性最高，但性能略低。

### 访问模式

有三种选项:

* `ReadWriteOnce` – 该卷可以被单个节点以读写方式挂载
* `ReadOnlyMany` – 该卷可以被多个节点以只读方式挂载
* `ReadWriteMany` – 该卷可以被多个节点以读写方式挂载

## 理解持久卷申领（PVC）

`PersistentVolume` 可以理解为由管理员预先分配的存储，相当于预分配。

`PersistentVolumeClaim` 以理解为用户或工作负载请求的存储。

当用户创建 PVC 请求存储时，Kubernetes 会尝试将该 PVC 与预分配的 PV 匹配。如果找到匹配项，PVC 会绑定到 PV，用户就可以开始使用这块预分配的存储。

## 了解如何为应用配置持久化存储

在不使用 storageclass 的情况下，需要以下几个步骤：

创建 PV:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

创建工作负载以利用该存储：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

如果使用 storageclass，则不需要 persistentvolume 对象，只需创建 persistentvolumeclaim 即可。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: myStorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```
