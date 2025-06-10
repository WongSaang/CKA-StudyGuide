# 工作负载和调度

## 了解部署以及如何执行滚动更新和回滚

Deployment 旨在取代 Replication Controller。它们提供相同的复制功能（通过 Replica Sets）以及推出更改并在必要时回滚的能力。下面是一个示例配置：


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx-frontend
  template:
    metadata:
      labels:
        app: nginx-frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
        ports:
        - containerPort: 80
```

利用 `deployments` 的主要原因是通过一个管理单元——`deployment` 对象，来管理多个相同的 Pod。如果需要进行更改，我们将更改应用到 `deployment` 对象，而不是单独的 Pod。由于 `deployments` 的声明式特性，Kubernetes 会纠正期望状态与运行状态之间的任何差异，并进行相应调整。例如，如果我们手动删除了某些内容。

然后我们可以使用 `kubectl describe deployment nginx-deployment` 来描述它：

### 更新

要更新现有的部署，我们有两个主要选项：

* 滚动更新
* 重建

滚动更新，顾名思义，会将部署中的容器替换为由新镜像创建的容器。

当应用程序支持不同 Pod 混合（即应用程序版本）时，使用滚动更新。这种方法不会导致服务停机，但需要更长时间才能将部署提升到所请求的版本。旧版本和新版本的 Pod 规范将共存，直到它们全部轮换。

重新创建会删除所有现有的 Pod，然后启动新的 Pod。这种方法会导致停机时间，可将其视为一种“爆炸式”方法。

Kubernetes 文档中列出的示例大多是命令式的，但我更喜欢声明式的方法。例如，创建一个新的 yaml 文件并进行所需的更改，在此示例中，nginx 容器的版本被提升。


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx-frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.15
        ports:
        - containerPort: 80
```

然后我们可以应用此文件：`kubectl apply -f updateddeployment.yaml --record=true`

接着执行以下操作：


```shell
kubectl rollout status deployment/nginx-deployment

Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 4 of 5 updated replicas are available...
deployment "nginx-deployment" successfully rolled out
```

我们还可以使用 `kubectl rollout history` 来查看部署的修订历史记录：

```shell
kubectl rollout history deployment/nginx-deployment

deployment.extensions/nginx-deployment
REVISION  CHANGE-CAUSE
1     <none>
2     <none>
4     <none>
5     kubectl apply --filename=updateddeployment.yaml --record=true
```

另外，我们也可以使用命令式方法来实现：

```shell
kubectl --record deployments/nginx-deployment set image deployments/nginx-deployment nginx=nginx:1.9.1

deployment.extensions/nginx-deployment image updated
deployment.extensions/nginx-deployment image updated
```

### 回滚

回滚到上一个版本：

```shell
kubectl rollout undo deployment/nginx-deployment 
```

回滚到特定版本：

```shell
kubectl rollout undo deployment/nginx-deployment --to-revision 5
```

`revision` 的来源：`kubectl rollout history deployment/nginx-deployment`

## 了解如何使用 ConfigMaps 和 Secrets 来配置应用程序

ConfigMaps 是一种将配置与 Pod 清单解耦的方法。显然，第一步是在让 Pod 使用它们之前创建一个 ConfigMap： 

```shell
kubectl create configmap <map-name> <data-source>
```

“Map-name” 是我们为这个特定映射指定的任意名称，而 “data-source” 对应的是存储在 ConfigMap 中的键值对。

```shell
kubectl create configmap vt-cm --from-literal=blog=virtualthoughts.co.uk
```

此时我们可以描述它：

```shell
kubectl describe configmap vt-cm
Name:         vt-cm
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
blog:
----
virtualthoughts.co.uk
```

要在 Pod 中引用此 ConfigMap，我们需要在相应的 YAML 文件中声明它：

ConfigMap 可以作为`volumes`（卷）或`volumes`（环境变量）挂载。以下示例利用了后者。


```yaml
apiVersion: v1
kind: Pod
metadata:
 name: config-test-pod
spec:
 containers:
 - name: test-container
   image: busybox
   command: [ "/bin/sh", "-c", "env" ]
   env:
     - name: BLOG_NAME
       valueFrom:
         configMapKeyRef:
           name: vt-cm
           key: blog
 restartPolicy: Never
```

上述 Pod 将输出环境变量，因此我们可以通过提取 Pod 的日志来验证它是否使用了 ConfigMap：

```shell
kubectl logs config-test-pod | grep "BLOG_NAME="
...
BLOG_NAME=virtualthoughts.co.uk
...
```

## 了解如何扩展应用程序

断添加更多的单个 Pod 并不是一种可持续的应用程序扩展模式。为了促进大规模应用的部署，我们需要利用更高级的构造，如 ReplicaSets 或 Deployments。如前所述，`deployments` 为我们提供了一个单一的管理单元来管理底层的 Pod。我们可以通过扩展 `deployment` 对象来增加 Pod 的数量。

例如，如果部署了以下内容：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: nginx-deployment
spec:
 replicas: 5
 template:
   metadata:
     labels:
       app: nginx-frontend
   spec:
     containers:
     - name: nginx
       image: nginx:1.14
       ports:
       - containerPort: 80
```

如果我们想要扩展它，我们可以简单地修改 yaml 文件并通过修改“replicas”字段来扩展/缩小部署，或者在运行时修改它：


```shell
kubectl scale deployment nginx-deployment --replicas 10
```

## 理解用于创建健壮、自愈型应用部署的基础组件

Deployment 通过使用协调循环来检查已部署的 Pod 数量是否与清单中定义的一致，从而实现这一点。在底层，Deployment 利用 ReplicaSet 来实现这一功能，而 ReplicaSet 主要负责这一特性。

StatefulSet 与 Deployment 类似，例如它们都管理一系列 Pod 的部署和扩展。然而，除了 Deployment 的功能外，StatefulSet 还对 Pod 的顺序和唯一性提供了保证。StatefulSet 为每个 Pod 保持一个固定的身份。这些 Pod 虽然由相同的规范创建，但并不是可以互换的：每个 Pod 都有一个持久的标识符，在重新调度时也会保持不变。

StatefulSet 对于需要以下一种或多种特性的应用程序非常有价值。

* 稳定且唯一的网络标识符。
* 稳定且持久的存储。
* 有序且平滑的部署与扩展。
* 有序且自动化的滚动更新。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
 name: nginx-statefulset
spec:
 selector:
   matchLabels:
     app: vt-nginx
 serviceName: "nginx"
 replicas: 2
 template:
   metadata:
     labels:
       app: vt-nginx
   spec:
     containers:
     - name: vt-nginx
       image: nginx:1.7.9
       ports:
       - containerPort: 80
```

## 理解资源限制如何影响 Pod 调度

在命名空间级别，我们可以定义资源限制。这可以对资源进行约束，特别适用于多租户环境，并提供了一种机制，防止 Pod 消耗超出允许范围的资源，从而避免对整个环境产生不利影响。

我们可以定义以下内容：

命名空间的默认内存 / CPU **requests & limits**（请求与限制）

命名空间的最小和最大内存 / CPU **constraints**（约束）

命名空间的内存 / CPU **Quotas**（配额）

### 默认请求和限制

如果在具有默认请求/限制值的命名空间中创建容器，并且在清单中没有显式定义这些值，则容器会继承命名空间的这些值。

注意，如果你为容器定义了内存/CPU 限制，但没有定义请求，Kubernetes 会将限制值作为请求值。

### 最小 / 最大约束

如果 Pod 不符合约束设定的范围，它将不会被调度。

### 配额

控制整个 _命名空间_ 内可消耗的 CPU/内存 _总_ 量。

示例: 尝试调度一个请求的内存超过命名空间定义限制的 Pod

创建命名空间：

```shell
kubectl create namespace tenant-mem-limited
```

创建一个用于限制资源的 YAML 清单：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-max-mem
  namespace: tenant-mem-limited
spec:
  limits:
  - max:
      memory: 250Mi
    type: Container
```

将其应用到上述命名空间：

```shell
kubectl apply -f maxmem.yaml
```

创建一个内存请求超出限制的 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: too-much-memory
  namespace: tenant-mem-limited 
spec:
  containers:
  - name: too-much-mem
    image: nginx
    resources:
      requests:
        memory: "300Mi"
```

执行上述操作将产生以下结果：

```shell
The Pod "too-much-memory" is invalid: spec.containers[0].resources.requests: Invalid value: "300Mi": must be less than or equal to memory limit
```

由于我们已将命名空间的 Pod 限制设置为 250MiB，因此请求 300MiB 会失败。

## 了解清单管理和常用模板工具

### Kustomize

[Kustomize](https://kubernetes.io/zh-cn/docs/tasks/manage-kubernetes-objects/kustomization/) 是一个用于原生 Kubernetes 清单（YAML 格式）的模板工具。在处理原始 YAML 文件时，通常会有一个目录，里面包含多个用于标识所创建资源的文件。首先，需要有一个包含我们清单的目录：

```shell
/home/david/app/base
total 16
drwxrwxr-x  2 david david 4096 Feb  9 11:44 .
drwxr-xr-x 27 david david 4096 Feb  9 11:44 ..
-rw-rw-r--  1 david david  340 Feb  9 11:09 deployment.yaml
-rw-rw-r--  1 david david  153 Feb  9 11:09 service.yaml
```

这将构成我们的 base——我们将在此基础上通过添加覆盖（overlays）来进行自定义。首先，我们需要一个 kustomize 文件，可以通过 kustomize create --autodetect 创建。

这将在当前目录下生成 kustomization.yaml 文件：

```shell
total 20
drwxrwxr-x  2 david david 4096 Feb  9 11:47 .
drwxr-xr-x 27 david david 4096 Feb  9 11:47 ..
-rw-rw-r--  1 david david  340 Feb  9 11:09 deployment.yaml
-rw-rw-r--  1 david david  108 Feb  9 11:47 kustomization.yaml
-rw-rw-r--  1 david david  153 Feb  9 11:09 service.yaml
```

内容如下：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

### 变体与覆盖层

* 变体（variant）——与 base 配置有所不同的配置
* 覆盖层（overlay）——将多个变体组合在一起

比如说，我们想为不同的环境（生产 prod 和开发 dev）生成基于此配置的清单，但又有额外的自定义。在本例中，我们将创建一个包含在单一覆盖层中的 dev 变体。

```shell
mkdir -p overlays/{dev,prod}
cd overlays/dev 
```

首先创建一个 Kustomization 对象，指定 base（这将生成 `kustomization.yaml` 文件）：

```shell
kustomize create --resources ../../base
```

在本例中，我想将副本数更改为 1，因为这是开发环境。在 `dev` 目录下，新建一个名为 `deployment.yaml` 的文件，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx-deployment
spec:
  replicas: 1
```

`kustomization.yaml` 文件需要修改以包含 `patchesStrategicMerge` 块：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patchesStrategicMerge:
  - deployment.yaml
```

补丁可用于对资源应用不同的自定义。Kustomize 通过 `patchesStrategicMerge` 和 `patchesJson6902` 支持不同的补丁机制。`patchesStrategicMerge` 是一个文件路径列表。

我们可以通过在基础目录下执行以下命令来生成清单并应用到集群：

```shell
kustomize build ./overlay/dev | kubectl apply -f -
```

通过运行此命令，由于我们应用了自定义，部署对象中只会创建 1 个 Pod，而不是 `base` 中定义的数量。我们也可以对生产环境或任意数量的其他环境进行同样的操作。


### Helm

Helm 类似于 Linux 世界中的 apt 或 yum。它实际上是 Kubernetes 的包管理器。Helm 中的“包”被称为 `charts`，您可以使用自己的值对其进行自定义。

考试不太可能要求任何人从头创建一个 Helm 图表，但了解其工作原理是个好主意。

#### Helm 仓库
仓库是存储 Helm 图表的地方。通常，一个仓库会包含多个可供选择的图表。Helm 可以通过 CLI 客户端进行管理，可以通过运行以下命令添加仓库：

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
```

列出仓库中的包：

```shell
helm search repo bitnami
```

从仓库安装包：

```shell
helm install my-release bitnami/mariadb
```

其中 my-release 是一个字符串，用于标识该应用程序的已安装实例。

可以自定义的参数取决于图表的配置方式。对于上述的 MariaDB 图表，它们列在[https://github.com/bitnami/charts/tree/master/bitnami/mariadb/#parameters](https://github.com/bitnami/charts/tree/master/bitnami/mariadb/#parameters)

这些值封装在仓库中对应的 values.yaml 文件中。您可以填充它的一个实例并通过以下方式应用：

```shell
helm install -f https://raw.githubusercontent.com/bitnami/charts/master/bitnami/mariadb/values.yaml my-release bitnami/mariadb
```

或者，可以通过使用 `--set` 来声明变量，例如：

```shell
helm install my-release --set auth.rootPassword=secretpassword bitnami/mariadb
```
