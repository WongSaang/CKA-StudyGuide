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

Constantly adding more, individual pods is not a sustainable model for scaling an application. To facilitate applications at scale, we need to leverage higher level constructs such as replicasets or deployments. As mentioned previously, `deployments` provide us with a single administrative unit to manage the underlying pods. We can scale a `deployment` object to increase the number of `pods`.

As an example, if the following is deployed:

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

If we wanted to scale this, we can simply modify the yaml file and scale up/down the deployment by modifying the “replicas” field, or modify it in the fly:

```shell
kubectl scale deployment nginx-deployment --replicas 10
```

## Understand the primitives used to create robust, self-healing, application deployments

Deployments facilitate this by employing a reconciliation loop to check the number of deployed pods matches what’s defined in the manifest. Under the hood, deployments leverage ReplicaSets, which are primarily responsible for this feature.

Stateful Sets are similar to deployments, for example they manage the deployment and scaling of a series of pods. However, in addition to deployments they also provide guarantees about the ordering and uniqueness of Pods. A StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

StatefulSets are valuable for applications that require one or more of the following.

* Stable, unique network identifiers.
* Stable, persistent storage.
* Ordered, graceful deployment and scaling.
* Ordered, automated rolling updates.

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

## Understand how resource limits can affect Pod scheduling

At a namespace level, we can define resource limits. This enables a restriction in resources, especially helpful in multi-tenancy environments and provides a mechanism to prevent pods from consuming more resources than permitted, which may have a detrimental effect on the environment as a whole.

We can define the following:

Default memory / CPU **requests & limits** for a namespace

Minimum and Maximum memory / CPU **constraints** for a namespace

Memory/CPU **Quotas** for a namespace

### Default Requests and Limits

If a container is created in a namespace with a default request/limit value and doesn't explicitly define these in the manifest, it inherits these values from the namespace

Note, if you define a container with a memory/CPU limit, but not a request, Kubernetes will define the limit the same as the request.

### Minimum / Maximum Constraints

If a pod does not meet the range in which the constraints are valued at, it will not be scheduled.

### Quotas

Control the _total_ amount of CPU/memory that can be consumed in the _namespace_ as a whole.

Example: Attempt to schedule a pod that request more memory than defined in the namespace

Create a namespace:

```shell
kubectl create namespace tenant-mem-limited
```

Create a YAML manifest to limit resources:

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

Apply this to the aforementioned namespace:

```shell
kubectl apply -f maxmem.yaml
```

To create a pod with a memory request that exceeds the limit:

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

Executing the above will yield the following result:

```shell
The Pod "too-much-memory" is invalid: spec.containers[0].resources.requests: Invalid value: "300Mi": must be less than or equal to memory limit
```

As we have defined the pod limit of the namespace to 250MiB, a request for 300MiB will fail.

## Awareness of manifest management and common templating tools

### Kustomize

Kustomize is a templating tool for Kubernetes manifests in its native form (Yaml). When working with raw YAML files you will typically have a directory containing several files identifying the resources it creates. To begin, a directory containing our manifests needs to exist:

```shell
/home/david/app/base
total 16
drwxrwxr-x  2 david david 4096 Feb  9 11:44 .
drwxr-xr-x 27 david david 4096 Feb  9 11:44 ..
-rw-rw-r--  1 david david  340 Feb  9 11:09 deployment.yaml
-rw-rw-r--  1 david david  153 Feb  9 11:09 service.yaml
```

This will form our `base` - we will build on this but adding customisations in the form of overlays. First, we need a `kustomize` file. which can be created with `kustomize create --autodetect`

This will create kustomization.yaml in the current directory:

```shell
total 20
drwxrwxr-x  2 david david 4096 Feb  9 11:47 .
drwxr-xr-x 27 david david 4096 Feb  9 11:47 ..
-rw-rw-r--  1 david david  340 Feb  9 11:09 deployment.yaml
-rw-rw-r--  1 david david  108 Feb  9 11:47 kustomization.yaml
-rw-rw-r--  1 david david  153 Feb  9 11:09 service.yaml
```

The contents being:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
```

#### Variants and Overlays

* variant - Divergence in configuration from the `base`
* overlay - Composes variants together

Say, for example, we wanted to generate manifests for different environments (prod and dev) that are based from this config, but have additional customisations. In this example we will create a `dev` variant encapsulated in a single Overlay

```shell
mkdir -p overlays/{dev,prod}
cd overlays/dev 
```

Begin by creating a Kustomization object specifying the base (this will create `kustomization.yaml`) :

```shell
kustomize create --resources ../../base
```

In this example, I want to change the replica count to 1, as it's a dev environment. In the `dev` directory, create a new file `deployment.yaml` containing:

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

The `kustomization.yaml` file needs modifying to include a `patchesStrategicMerge` block:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
patchesStrategicMerge:
  - deployment.yaml
```

Patches can be used to apply different customizations to Resources. Kustomize supports different patching mechanisms through `patchesStrategicMerge` and `patchesJson6902`. `patchesStrategicMerge` is a list of file paths.

We can generate the manifests and apply to the cluster by executing (from the base folder):

```shell
kustomize build ./overlay/dev | kubectl apply -f -
```

By running this, only 1 pod will be created in the deployment object, instead of what's defined in the `base` because of the customisation we've applied. We can do the same with prod, or any arbitrary number of environments.


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
