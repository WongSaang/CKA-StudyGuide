# 集群架构，安装与配置

## 管理基于角色的访问控制（RBAC）

Kubernetes 实现了一个 RBAC 框架，来管理对集群内资源的访问；它是整体身份验证、授权与准入控制框架的一部分。

![img.png](./images/img.png)

为了确定谁（或什么）能访问哪些资源，需要执行活干步骤

### 步骤 1 - 身份验证

第一步是身份人验证，即用户或服务账户如何确定自己的身份。根据来源的不同，使用相应的身份验证模块。身份验证模块从以下几个方面进行身份验证：

* 客户端证书
* 密码
* 明文令牌
* 引导令牌
* JWT 令牌 (服务账户)

所有身份验证通过基于 TLS 的 HTTP 进行处理。

### 步骤 2 - 授权

用户或者服务账户被验证之后，必须对请求进行授权。任何身份验证请求后面都有某种操作请求，操作定义了该请求所作用的对象及具体动作。例如，要列出给定命名空间的 Pod。

只要现有策略授予用户这些权限，任何请求都将被允许执行。

### 步骤 3 和 4 - 准入控制

准入控制模块是可以修改或拒绝请求的软件模块。除了授权模块可访问的属性外，准入控制模块还可以访问正在创建或更新的对象的内容。它们作用于对象的创建、删除、更新或连接（代理）操作，但不处理读取操作。

### `Role` 和 `Rolebindings`

在 Kubernetes 中，实施 RBAC 规则主要涉及两种对象类型—— `role` 和 `rolebindings`：

![img.png](images/roleandrolebindings.png)

`role`（角色）授予对单个命名空间内资源的访问权限。

`rolebinding`（角色绑定）将角色的权限授予单个命名空间内的用户、用户组或者服务账户。

`clusterrole`（集群角色）和 `clusterrolebindings`（集群角色绑定）功能类似，但显然是用于提供对非命名空间资源的访问权限。

`kubectl api-resources --namespaced=false` 可以用来查询非命名空间资源。例如：`node`（节点）、`persistenvolum`（持久卷）、`storageclass`（存储类） 和 `users`（用户）。

`Users`（用户）可以是`serviceaccounts`（服务账户），也可以是`users`（普通用户）。前者通常用于认证应用程序，后者则用于人类用户。

为了测试，以下内容将创建 `namespace`（命名空间）, `serviceaccount`（服务账户）, `role`（角色） 和 `rolebinding`（角色绑定）：

```yaml
apiVersion: v1
kind: Namespace
metadata:
 name: rbac-test
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: rbac-test-sa
 namespace: rbac-test
```

特别重要的是下面内容的格式。
`apiGroup` : 确定要将其应用于哪个API组。
`resources`: 要将其应用于哪些资源类型。
`verbs`: 我们可以对这些对象执行哪些操作（例如创建、删除、监视等）。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
 name: rbac-test-role
 namespace: rbac-test
rules:
 - apiGroups: [""]
   resources: ["pods"]
   verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
 name: rbac-test-rolebinding
 namespace: rbac-test
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: Role
 name: rbac-test-role
subjects:
- kind: ServiceAccount
  name: rbac-test-sa
  namespace: rbac-test
```

然后我们可以使用 kubectl 进行验证。以下命令会返回 yes，因为该服务账号有权限获取 pods 。

```shell
kubectl -n rbac-test --as=system:serviceaccount:rbac-test:rbac-test-sa auth can-i get pods
yes
```

然而对于 `secrets`，它会返回 `no`。 

```shell
kubectl -n rbac-test --as=system:serviceaccount:rbac-test:rbac-test-sa auth can-i get secrets
no
```

## 使用 Kubeadm 安装一个基础集群

kubeadm 是一个用来引导部署 Kubernetes 的实用程序，可以将 Kubernetes 引导部署到若干现有的、原生节点上。
kubeadm 是一个用于在多个现有、普通节点上初始化 Kubernetes 的工具。它负责处理 etcd 集群、Kubernetes 主节点和工作节点，包括启动一个可用的最小 k8s 集群所需的所有组件。

使用 Kubeadm 完成部署后，你将得到一个完全可用、功能齐全的 Kubernetes 集群。

在难度方面，它与 Kelsey Hightower 的 “Kubernetes the hard way” 完全相反。

为了考试，建议您熟悉这两种部署 Kubernetes 的方式。

Kubeadm 是一个命令行工具，它执行以下功能：

* **kubeadm init** 用于引导一个 Kubernetes 控制平面节点
* **kubeadm join** 用于引导一个 Kubernetes 工作节点，并将工作节点加入集群
* **kubeadm upgrade** 用于将 Kubernetes 集群更新到新版本
* **kubeadm config** 如果你的集群是用 kubeadm v1.7.x 或更低版本初始化的, 在使用 **kubeadm upgrade** 升级集群之前，使用这个命令来配置你的集群
* **kubeadm token** 用于管理加入集群的令牌
* **kubeadm reset** 用于撤销 kubeadm init 或 kubeadm join 对当前主机所做的所有更改
* **kubeadm version** 用于打印 kubeadm 的版本
* **kubeadm alpha** 用于访问和测试实验性功能

### Kubeadm - 安装主节点

创建3台 Ubuntu Server 虚拟机用于以下示例

* k8s-cl02-ms01
* k8s-cl02-wk01
* k8s-cl02-wk02

在适当情况下，确保您的节点已安装容器运行时。

在主节点上安装所需的二进制文件

```shell
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

初始化主节点:

```shell
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
# 如果是中国网络，你可以使用
sudo kubeadm init --image-repository=registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16
```

注意，是否需要传递 `--pod-network` 参数，取决于所选择的 CNI（容器网络接口）。对于 Flannel， 这是必须的。Kubeadm 还会提示您是否有未满足的前置条件。

完成后，将显示:

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.10.80:6443 --token j5nqhd.cnfmnjgc68aato60 \
    --discovery-token-ca-cert-hash sha256:cbc91031c1ffa47bbea83aa1cf65e99821a1f582c4363e1a4408715bfd66bb60 
```

需要注意的一些重要信息：

* Kubeadm 已为您创建了管理员 kubeconfig 文件，建议将其复制到当前登录用户的主目录以方便使用。

* Kubeadm 尚未部署 Pod 网络解决方案。因此，这属于安装后的操作。

* Kubeadm 提供了一个带有令牌的 join 命令，用于添加工作节点。如果需要，我们可以重新生成该令牌。

如果我们执行 `kubectl get nodes` 命令，会看到主节点处于未就绪状态。

```shell
NAME            STATUS     ROLES    AGE     VERSION
k8s-cl02-ms01   NotReady   master   6m20s   v1.20.2
```

根据 kubeadm 的输出，安装一个网络解决方案，例如 Flannel。

为了使 Flannel 正常工作，您必须在运行 kubeadm init 时传递参数 --pod-network-cidr=10.244.0.0/16。

此外，通过运行 `sysctl net.bridge.bridge-nf-call-iptables=1` 将 bridge-nf-call-iptables 设置为 1。

安装 Flannel:

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

几秒钟后，主节点将变为就绪状态。 

```shell
NAME            STATUS   ROLES    AGE   VERSION
k8s-cl02-ms01   Ready    master   10m   v1.20.2

```

### Kubeadm - 安装工作节点

工作节点的安装过程跟主节点类似 —— 唯一不要做的是执行`kubeadm init` 命令，它只在主节点运行。对于工作节点，我们使用`kubeadm join`。

准备工作:

* 安装一个容器运行时
* 安装 Kubeadm 二进制文件（和上面一样）

要将工作节点加入由 kubeadm 创建的集群，我们需要使用 kubeadm join 命令，并使用在主节点上生成的令牌。该令牌会在主节点运行 kubeadm init 后显示出来。然而，如果令牌未被记录或已过期，我们可以在主节点上轻松重新生成：

（在主节点上）

```shell
david@k8s-cl02-ms01:~$ kubeadm token create --print-join-command
kubeadm join 172.16.10.80:6443 --token ht55yv.8lq69q0189xhe2ql     --discovery-token-ca-cert-hash sha256:cbc91031c1ffa47bbea83aa1cf65e99821a1f582c4363e1a4408715bfd66bb60
```

在工作节点上使用此命令（以 root 用户身份运行）

```shell
root@k8s-cl02-wk01:~# kubeadm join 172.16.10.80:6443 --token ht55yv.8lq69q0189xhe2ql     --discovery-token-ca-cert-hash sha256:cbc91031c1ffa47bbea83aa1cf65e99821a1f582c4363e1a4408715bfd66bb60
```

之后会显示确认信息：

```shell
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

要验证，请在主节点上运行 `kubectl get nodes` 命令：

```shell
NAME            STATUS   ROLES    AGE     VERSION
k8s-cl02-ms01   Ready    master   50m     v1.20.2
k8s-cl02-wk01   Ready    <none>   2m10s   v1.20.2
```

## 管理一个高可用的 Kubernetes 集群

上一节演示了创建一个包含一个主节点和多个工作节点的 K8s 集群——这种方式无法为控制平面提供容错能力。为实现这一目标，有几种拓扑结构可供选择：

### 堆叠式 etcd

![img.png](images/stacked-etcd.png)

* 多个工作节点
* 在负载均衡器后面部署多个控制平面节点
* 控制平面内的嵌入 etcd

注意:


etcd 是基于 [Quorum机制](https://zh.wikipedia.org/wiki/Quorum_(%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F)) 的。因此，如果使用堆叠式控制平面节点和 etcd，则必须使用奇数个节点。

### 外部 etcd

![img.png](images/external-etcd.png)

Notes:

* 多个工作节点
* 在负载均衡器后面部署多个控制平面节点
* Etcd 独立于 K8s 集群运行

注意:

这种设置的优势在于 etcd 和控制平面可以独立扩展和管理。这提供了更大的灵活性，但以增加操作复杂性为代价。

### 评估集群健康状况 

`kubectl get componentstatus` 从 1.20 版本开始已被弃用。一个合适的替代方法是直接探测 API 服务器。例如，在主节点上运行 `curl -k https://localhost:6443/livez?verbose`，它会返回：

```shell
[+]ping ok
[+]log ok
[+]etcd ok
[+]poststarthook/start-kube-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
.....etc
```

存在三个端点——`healthz`、`livez` 和 `readyz`，用于指示 API 服务器的当前状态：

* healthz：健康检查端点，表示 API 服务器的总体健康状况。
* livez：存活检查端点，表示 API 服务器是否正在运行。
* readyz：就绪检查端点，表示 API 服务器是否准备好处理请求。


## 为部署 Kubernetes 集群提供底层基础设施

上述拓扑选择将影响需要提供的底层资源。这些资源的配置方式取决于具体的云服务提供商。一些通用的观察包括：

* 禁用 swap.
* 利用云功能实现高可用性——例如使用多个可用区（AZ）。
* Windows 可用于工作节点，但不能用于控制平面。

## 使用 Kubeadm 对 Kubernetes 集群进行版本升级

首先，安装指定版本的 kubeadm。这将决定它部署的 Kubernetes 版本：

```shell
sudo apt-get update && sudo apt-get install -y kubeadm=1.19.0-00 kubelet=1.19.0-00 kubectl=1.19.0-00 && sudo apt-mark hold kubeadm
```

启动一个 Kubernetes 集群 

```shell
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

添加 CNI (Container Network Interface - 容器网络接口)

```shell
https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

要升级底层 Kubernetes 集群，我们需要先升级 kubeadm。 

升级 kubeadm

```shell
sudo apt-mark unhold kubeadm
sudo apt-get install --only-upgrade kubeadm
```

接下来，我们运行 `plan` 命令进行升级规划——这不会改变集群，但会显示可以进行的更改

```shell
sudo kubeadm upgrade plan

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
kubelet     1 x v1.19.0   v1.20.2

Upgrade to the latest stable version:

COMPONENT                 CURRENT   AVAILABLE
kube-apiserver            v1.19.7   v1.20.2
kube-controller-manager   v1.19.7   v1.20.2
kube-scheduler            v1.19.7   v1.20.2
kube-proxy                v1.19.7   v1.20.2
CoreDNS                   1.7.0     1.7.0
etcd                      3.4.9-1   3.4.13-0

You can now apply the upgrade by executing the following command:

kubeadm upgrade apply v1.20.2
```

**重要提示：** 在此步骤之后，必须手动升级 kubelet。

升级集群:

```shell
kubeadm upgrade apply v1.20.2
```

升级 Kubelet:

```shell
sudo apt-get install --only-upgrade kubelet kubectl
```

## 实施 etcd 备份和恢复

### 备份 etcd

对数据库进行快照备份，然后将其存储在安全位置：

```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db --cacert /etc/kubernetes/pki/etcd/server.crt --cert /etc/kubernetes/pki/etcd/ca.crt --key /etc/kubernetes/pki/etcd/ca.key
```

验证备份：

```shell
sudo ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshot.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 2125d542 |   364069 |        770 |  3.8 MB    |
+----------+----------+------------+------------+
```

### 恢复到 etcd

要执行恢复操作：

```shell
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db
```
