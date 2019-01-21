#### K8S 组件

Kubernetes Cluster 由 Master 和 Node 组成，节点上运行着若干 Kubernetes 服务。

##### Master 节点
Master 是 Kubernetes Cluster 的大脑，运行着如下 Daemon 服务：kube-apiserver、kube-scheduler、kube-controller-manager、etcd 和 Pod 网络（例如 flannel）。

![76ff4981789401adc0079f71f0fc6dac.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3840)
1. API Server（kube-apiserver）
***
API Server 提供 HTTP/HTTPS RESTful API，即 Kubernetes API。API Server 是 Kubernetes Cluster 的前端接口，各种客户端工具（CLI 或 UI）以及 Kubernetes 其他组件可以通过它管理 Cluster 的各种资源。

2. Scheduler（kube-scheduler）
***
Scheduler 负责决定将 Pod 放在哪个 Node 上运行。Scheduler 在调度时会充分考虑 Cluster 的拓扑结构，当前各个节点的负载，以及应用对高可用、性能、数据亲和性的需求。

3.Controller Manager（kube-controller-manager）
***
Controller Manager 负责管理 Cluster 各种资源，保证资源处于预期的状态。Controller Manager 由多种 controller 组成，包括 replication controller、endpoints controller、namespace controller、serviceaccounts controller 等。

不同的 controller 管理不同的资源。例如 replication controller 管理 Deployment、StatefulSet、DaemonSet 的生命周期，namespace controller 管理 Namespace 资源。

4.etcd
***

etcd 负责保存 Kubernetes Cluster 的配置信息和各种资源的状态信息。当数据发生变化时，etcd 会快速地通知 Kubernetes 相关组件。

5.Pod 网络
***
Pod 要能够相互通信，Kubernetes Cluster 必须部署 Pod 网络，flannel 是其中一个可选方案。
##### Node节点
Node 是 Pod 运行的地方，Kubernetes 支持 Docker、rkt 等容器 Runtime。 Node上运行的 Kubernetes 组件有 kubelet、kube-proxy 和 Pod 网络（例如 flannel）。
![daae97cf75e5fd79191bdb5811aee9d0.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3841)

1. kubelet
kubelet 是 Node 的 agent，当 Scheduler 确定在某个 Node 上运行 Pod 后，会将 Pod 的具体配置信息（image、volume 等）发送给该节点的 kubelet，kubelet 根据这些信息创建和运行容器，并向 Master 报告运行状态。
2. kube-proxy
service 在逻辑上代表了后端的多个 Pod，外界通过 service 访问 Pod。service 接收到的请求是如何转发到 Pod 的呢？这就是 kube-proxy 要完成的工作。

每个 Node 都会运行 kube-proxy 服务，它负责将访问 service 的 TCP/UPD 数据流转发到后端的容器。如果有多个副本，kube-proxy 会实现负载均衡。
3. Pod 网络

Pod 要能够相互通信，Kubernetes Cluster 必须部署 Pod 网络，flannel 是其中一个可选方案。
##### 完整架构图
![e9d53a11bc09be7b76690077ee78c3b9.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3842)
为什么 k8s-master 上也有 kubelet 和 kube-proxy 呢？

这是因为 Master 上也可以运行应用，即 Master 同时也是一个 Node。

几乎所有的 Kubernetes 组件本身也运行在 Pod 里，执行如下命令：
```shell
kubectl get pod --all-namespaces -o wide
```
![2387c8c0698d0eb05f44d97ecd753024.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3843)

Kubernetes 的系统组件都被放到 kube-system namespace 中。这里有一个 coredns 组件，它为 Cluster 提供 DNS 服务
kube-dns是在执行 kubeadm init 时 ,作为附加组件安装的。
kubelet 是唯一没有以容器形式运行的 Kubernetes 组件,在centOS中通过Systemd运行
![76f6691a386bfb56f7a3d1e4f041fd3b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3844)

#### 示例理解k8s基础架构

为更好地理解 Kubernetes 架构，部署一个应用来演示各个组件之间是如何协作的。

执行命令
```shell
kubectl run httpd-app --image=httpd --replicas=2
```
等待一段时间，应用部署完成。
![43371b441baa41b4efedd89107c82c5c.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3866)
Kubernetes 部署了 deployment httpd-app，有两个副本 Pod，运行k8s01节点.

##### 详细部署过程
① kubectl 发送部署请求到 API Server。
② API Server 通知 Controller Manager 创建一个 deployment 资源。
③ Scheduler 执行调度任务，将两个副本 Pod 分发到 k8s01节点上
④ k8s01  上的 kubelet 在节点上创建并运行 Pod。

补充两点：

* 应用的配置和当前状态信息保存在 etcd 中，执行 kubectl get pod 时 API Server 会从 etcd 中读取这些数据。
* flannel 会为每个 Pod 都分配 IP。因为没有创建 service，目前 kube-proxy 还没参与进来。