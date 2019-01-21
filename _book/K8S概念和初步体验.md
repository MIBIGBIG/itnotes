[toc]
### 一个简单的k8s集群


据说 Google 的数据中心里运行着超过 20 亿个容器，而且 Google 十年前就开始使用容器技术。

最初，Google 开发了一个叫 Borg 的系统（现在命令为 Omega）来调度如此庞大数量的容器和工作负载。在积累了这么多年的经验后，Google 决定重写这个容器管理系统，并将其贡献到开源社区，让全世界都能受益。

这个项目就是 Kubernetes。简单的讲，Kubernetes 是 Google Omega 的开源版本。

从 2014 年第一个版本发布以来，Kubernetes 迅速获得开源社区的追捧，包括 Red Hat、VMware、Canonical 在内的很多有影响力的公司加入到开发和推广的阵营。目前 Kubernetes 已经成为发展最快、市场占有率最高的容器编排引擎产品。

Kubernetes 一直在快速地开发和迭代。
我们会讨论 Kubernetes 重要的概念和架构，学习 Kubernetes 如何编排容器，包括优化资源利用、高可用、滚动更新、网络插件、服务发现、监控、数据管理、日志管理等。

#### 最小可用系统搭建
按照一贯的学习思路，我们会在最短时间内搭建起一个可用系统，这样就能够尽快建立起对学习对象的感性认识。先把玩起来，快速了解基本概念、功能和使用场景。

越是门槛高的知识，就越需要有这么一个最小可用系统。如果直接上来就学习理论知识和概念，很容易从入门到放弃。

当然，要搭建这么一个可运行的系统通常也不会太容易，不过很幸运，Kubernetes 官网已经为大家准备好了现成的最小可用系统。

kubernetes.io 开发了一个交互式教程，通过 Web 浏览器就能使用预先部署好的一个 kubernetes 集群，快速体验 kubernetes 的功能和应用场景，下面我就带着大家去玩一下。

打开[官网](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
左边就有教程菜单

![58efa53ba3cac436a9e5602747fac465.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3819)
教程会指引大家完成创建 kubernetes 集群、部署应用、访问应用、扩展应用、更新应用等最常见的使用场景，迅速建立感性认识。

创建 Kubernetes 集群
点击教程菜单 1. Create a Cluster -> Interactive Tutorial - Creating a Cluster

左边部分是操作说明，右边是 Terminal，命令终端窗口。

按照操作说明，我们在 Terminal 中执行 minikube start，然后执行 kubectl get nodes，这样就创建好了一个单节点的 kubernetes 集群。

![bbd8b7f33e129b34eaecfaf4fab3a741.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3820)

集群的唯一节点为 minikube，需要注意的是当前执行命令的地方并不是 minikube。我们是在通过 Kubernetes 的命令行工具 kubectl 远程管理集群。

执行 kubectl cluster-info 查看集群信息：
![5d3cf20e430ea01293b78cb6a3fec7bf.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3821)

只有一个节点
### 核心功能
#### 部署应用
执行命令
$ kubectl run kubernetes-bootcamp \
>       --image=docker.io/jocatalin/kubernetes-bootcamp:v1 \
>       --port=8080


```shell
$ kubectl run kubernetes-bootcamp \
>       --image=docker.io/jocatalin/kubernetes-bootcamp:v1 \
>       --port=8080
deployment.apps/kubernetes-bootcamp created
```
这里我们通过 kubectl run 部署了一个应用，命名为 kubernetes-bootcamp。

Docker 镜像通过 --image 指定。

--port 设置应用对外服务的端口。

这里 deployment 是 Kubernetes 的术语，可以理解为应用。

***
Kubernetes 还有一个重要术语 Pod。

Pod 是容器的集合，通常会将紧密相关的一组容器放到一个 Pod 中，同一个 Pod 中的所有容器共享 IP 地址和 Port 空间，也就是说它们在一个 network namespace 中。

Pod 是 Kubernetes 调度的最小单位，同一 Pod 中的容器始终被一起调度。

运行 kubectl get pods 查看当前的 Pod。
![8c9e1f535692a8cb0f062ce4407919dd.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3822)

kubernetes-bootcamp-56cdd766d-...就是应用的pod.


#### 访问应用
默认情况下，所有 Pod 只能在集群内部访问。对于上面这个例子，要访问应用只能直接访问容器的 8080 端口。为了能够从外部访问应用，我们需要将容器的 8080 端口映射到节点的端口。

执行如下命令:
```shell
kubectl expose deployment/kubernetes-bootcamp \
      --type="NodePort" \
      --port 8080
```
![b9c7d068f3b5b3f5145c3060518c2e6b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3823)
执行命令 kubectl get services 可以查看应用被映射到节点的哪个端口。

![97ac125e5c23d1f25c9ab769311976d7.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3824)
这里有两个 service，可以将 service 暂时理解为端口映射，后面我们会详细讨论。

kubernetes 是默认的 service，暂时不用考虑。kubernetes-bootcamp 是我们应用的 service，8080 端口已经映射到 minikube 的 31961 端口，端口号是随机分配的，可以执行如下命令访问应用：
curl minikube:31961
```shell
$ curl minikube:31961

Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-sgqc5 | v=1
```
Scale 应用
默认情况下应用只会运行一个副本，可以通过 kubectl get deployments查看副本数。

```shell
$ kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1         1         1            1           10m
```
执行如下命令将副本数增加到 3 个：

kubectl scale deployments/kubernetes-bootcamp --replicas=3
```shell
$ kubectl scale deployments/kubernetes-bootcamp --replicas=3
deployment.extensions/kubernetes-bootcamp scaled
$ kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   3         3         3            3           11m
```

可以看到pods也变成3个了
```shell
$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
kubernetes-bootcamp-56cdd766d-95b58   1/1       Running   0          50s
kubernetes-bootcamp-56cdd766d-sgqc5   1/1       Running   0          11m
kubernetes-bootcamp-56cdd766d-zqvq5   1/1       Running   0          50s
```
通过 curl 访问应用，可以看到每次请求发送到不同的 Pod，三个副本轮询处理，这样就实现了负载均衡。
```shell
$ for n in `seq 10`; do curl minikube:31961; done
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-zqvq5 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-sgqc5 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-95b58 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-zqvq5 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-zqvq5 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-zqvq5 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-95b58 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-95b58 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-zqvq5 | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-95b58 | v=1
```
要 scale down 也很方便，执行命令：

kubectl scale deployments/kubernetes-bootcamp --replicas=2
```shell
$ kubectl scale deployments/kubernetes-bootcamp --replicas=2
deployment.extensions/kubernetes-bootcamp scaled
$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
kubernetes-bootcamp-56cdd766d-95b58   1/1       Running   0          4m
kubernetes-bootcamp-56cdd766d-sgqc5   1/1       Running   0          15m

$ kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   2         2         2            2           16m
```
其中一个副本被删除了。

#### 滚动更新
当前应用使用的 image 版本为 v1，执行如下命令将其升级到 v2：
$ kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2

```shell
$ kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2

deployment.extensions/kubernetes-bootcamp image updated
```
![61d78c26c01d40267a4b734fa27eaf27.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3825)
通过 kubectl get pods 可以观察滚动更新的过程：v1 的 Pod 被逐个删除，同时启动了新的 v2 Pod。更新完成后访问新版本应用。
```shell
$ for n in `seq 3`; do curl minikube:31961; done

Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-7799cbcb86-lbvh8 | v=2
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-7799cbcb86-qxk49 | v=2
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-7799cbcb86-qxk49 | v=2
```
如果要回退到 v1 版本也很容易，执行 kubectl rollout undo 命令：

kubectl rollout undo deployments/kubernetes-bootcamp

```shell
$ kubectl get pods
NAME                                   READY     STATUS        RESTARTS AGE
kubernetes-bootcamp-56cdd766d-59j9r    1/1       Running       0 22s
kubernetes-bootcamp-56cdd766d-zpcr5    1/1       Running       0 24s
kubernetes-bootcamp-7799cbcb86-lbvh8   1/1       Terminating   0 3m
kubernetes-bootcamp-7799cbcb86-qxk49   1/1       Terminating   0
```
验证版本已经回退到 v1。
curl minikube:31961
```shell
$ for n in `seq 3`; do curl minikube:31961; done
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-59j9r | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-59j9r | v=1
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-56cdd766d-zpcr5 | v=1
```
至此，我们已经通过官网的交互式教程快速体验了 Kubernetes 的功能和使用方法
### 重要概念
在实践之前，必须先学习 Kubernetes 的几个重要概念，它们是组成 Kubernetes 集群的基石。

1. Cluster 
Cluster 是计算、存储和网络资源的集合，Kubernetes 利用这些资源运行各种基于容器的应用。

2. Master 
Master 是 Cluster 的大脑，它的主要职责是调度，即决定将应用放在哪里运行。Master 运行 Linux 操作系统，可以是物理机或者虚拟机。为了实现高可用，可以运行多个 Master。

3. Node 
Node 的职责是运行容器应用。Node 由 Master 管理，Node 负责监控并汇报容器的状态，并根据 Master 的要求管理容器的生命周期。Node 运行在 Linux 操作系统，可以是物理机或者是虚拟机。

在前面交互式教程中我们创建的 Cluster 只有一个主机 minikube，
它既是 Master 也是 Node。
4. Pod
Pod 是 Kubernetes 的最小工作单元。每个 Pod 包含一个或多个容器。Pod 中的容器会作为一个整体被 Master 调度到一个 Node 上运行。

#### Pod介绍
Kubernetes 引入 Pod 主要基于下面两个目的：
* 可管理性
有些容器天生就是需要紧密联系，一起工作。Pod 提供了比容器更高层次的抽象，将它们封装到一个部署单元中。Kubernetes 以 Pod 为最小单位进行调度、扩展、共享资源、管理生命周期。
* 通信和资源共享
Pod 中的所有容器使用同一个网络 namespace，即相同的 IP 地址和 Port 空间。它们可以直接用 localhost 通信。同样的，这些容器可以共享存储，当 Kubernetes 挂载 volume 到 Pod，本质上是将 volume 挂载到 Pod 中的每一个容器。

pods有两种使用方式
* 运行单个容器
one-container-per-Pod 是 Kubernetes 最常见的模型，这种情况下，只是将单个容器简单封装成 Pod。即便是只有一个容器，Kubernetes 管理的也是 Pod 而不是直接管理容器。

* 运行多个容器
但问题在于：哪些容器应该放到一个 Pod 中？ 
答案是：这些容器联系必须 非常紧密，而且需要 直接共享资源。

例子。
下面这个 Pod 包含两个容器：一个 File Puller，一个是 Web Server。

![99dca28578ad769a9645245efd3bc44a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3826)

File Puller 会定期从外部的 Content Manager 中拉取最新的文件，将其存放在共享的 volume 中。Web Server 从 volume 读取文件，响应 Consumer 的请求。

这两个容器是紧密协作的，它们一起为 Consumer 提供最新的数据；同时它们也通过 volume 共享数据。所以放到一个 Pod 是合适的。

再来看一个反例：是否需要将 Tomcat 和 MySQL 放到一个 Pod 中？
Tomcat 从 MySQL 读取数据，它们之间需要协作，但还不至于需要放到一个 Pod 中一起部署，一起启动，一起停止。同时它们是之间通过 JDBC 交换数据，并不是直接共享存储，所以放到各自的 Pod 中更合适。

##### Controller 
Kubernetes 通常不会直接创建 Pod，而是通过 Controller 来管理 Pod 的。Controller 中定义了 Pod 的部署特性，比如有几个副本，在什么样的 Node 上运行等。为了满足不同的业务场景，Kubernetes 提供了多种 Controller，包括 Deployment、ReplicaSet、DaemonSet、StatefuleSet、Job 等，我们逐一讨论。

Kubernetes 通常不会直接创建 Pod，而是通过 Controller 来管理 Pod 的。Controller 中定义了 Pod 的部署特性，比如有几个副本，在什么样的 Node 上运行等。为了满足不同的业务场景，Kubernetes 提供了多种 Controller，包括 Deployment、ReplicaSet、DaemonSet、StatefuleSet、Job 等，我们逐一讨论。

Deployment 是最常用的 Controller，比如前面在线教程中就是通过创建 Deployment 来部署应用的。Deployment 可以管理 Pod 的多个副本，并确保 Pod 按照期望的状态运行。

ReplicaSet 实现了 Pod 的多副本管理。使用 Deployment 时会自动创建 ReplicaSet，也就是说 Deployment 是通过 ReplicaSet 来管理 Pod 的多个副本，我们通常不需要直接使用 ReplicaSet。

DaemonSet 用于每个 Node 最多只运行一个 Pod 副本的场景。正如其名称所揭示的，DaemonSet 通常用于运行 daemon。

StatefuleSet 能够保证 Pod 的每个副本在整个生命周期中名称是不变的。而其他 Controller 不提供这个功能，当某个 Pod 发生故障需要删除并重新启动时，Pod 的名称会发生变化。同时 StatefuleSet 会保证副本按照固定的顺序启动、更新或者删除。

Job 用于运行结束就删除的应用。而其他 Controller 中的 Pod 通常是长期持续运行。

##### Service 
Deployment 可以部署多个副本，每个 Pod 都有自己的 IP，外界如何访问这些副本呢？

通过 Pod 的 IP 吗？
要知道 Pod 很可能会被频繁地销毁和重启，它们的 IP 会发生变化，用 IP 来访问不太现实。
答案是 Service。
Kubernetes Service 定义了外界访问一组特定 Pod 的方式。Service 有自己的 IP 和端口，Service 为 Pod 提供了负载均衡。

Kubernetes 运行容器（Pod）与访问容器（Pod）这两项任务分别由 Controller 和 Service 执行。

#####  Namespace
如果有多个用户或项目组使用同一个 Kubernetes Cluster，如何将他们创建的 Controller、Pod 等资源分开呢？

答案就是 Namespace。
Namespace 可以将一个物理的 Cluster 逻辑上划分成多个虚拟 Cluster，每个 Cluster 就是一个 Namespace。不同 Namespace 里的资源是完全隔离的。

Kubernetes 默认创建了三个 Namespace。
```shell
$ kubectl get namespace
NAME          STATUS    AGE
default       Active    40m
kube-public   Active    40m
kube-system   Active    40m
```
default -- 创建资源时如果不指定，将被放到这个 Namespace 中。

kube-system -- Kubernetes 自己创建的系统资源将放到这个 Namespace 中。

kube-public --- 该空间由系统自动创建并且对所有用户可读性，做为集群公用资源的保留命名空间。

