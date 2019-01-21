# 网络管理


Kubernetes 作为编排引擎管理着分布在不同节点上的容器和 Pod。Pod、Service、外部组件之间需要一种可靠的方式找到彼此并进行通信，Kubernetes 网络则负责提供这个保障。
主要内容:
* kubenetes网络模型
* 网络方案
* 网络策略


#### 网络管理模型

k8s采用的是基于扁平地址空间的网络模型，集群中的每个 Pod 都有自己的 IP 地址，Pod 之间不需要配置 NAT 就能直接通信。另外，同一个 Pod 中的容器共享 Pod 的 IP，能够通过 localhost 通信。

这种网络模型对应用开发者和管理员相当友好，应用可以非常方便地从传统网络迁移到 Kubernetes。每个 Pod 可被看作是一个个独立的系统，而 Pod 中的容器则可被看做同一系统中的不同进程。

下面讨论在这个网络模型下集群中的各种实体如何通信.

##### pod内容器的通信
当 Pod 被调度到某个节点，Pod 中的所有容器都在这个节点上运行，这些容器共享相同的本地文件系统、IPC 和网络命名空间。

不同 Pod 之间不存在端口冲突的问题，因为每个 Pod 都有自己的 IP 地址。当某个容器使用 localhost 时，意味着使用的是容器所属 Pod 的地址空间。

比如 Pod A 有两个容器 container-A1 和 container-A2，container-A1 在端口 1234 上监听，当 container-A2 连接到 localhost:1234，实际上就是在访问 container-A1。这不会与同一个节点上的 Pod B 冲突，即使 Pod B 中的容器 container-B1 也在监听 1234 端口。

##### POD之间通信
Pod 的 IP 是集群可见的，即集群中的任何其他 Pod 和节点都可以通过 IP 直接与 Pod 通信，这种通信不需要借助任何的网络地址转换、隧道或代理技术。Pod 内部和外部使用的是同一个 IP，这也意味着标准的命名服务和发现机制，比如 DNS 可以直接使用。
##### Pod 与 Service 的通信
Pod 间可以直接通过 IP 地址通信，但前提是 Pod 得知道对方的 IP。在 Kubernetes 集群中， Pod 可能会频繁的销毁和创建，也就是说 Pod 的 IP 不是固定的。为了解决这个问题，Service 提供了访问 Pod 的抽象层。无论后端的 Pod 如何变化，Service 都作为稳定的前端对外提供服务。同时，Service 还提供了高可用和负载均衡功能，Service 负责将请求转发给正确的 Pod。

##### 外部访问
无论是 Pod 的 IP 还是 Service 的 Cluster IP，它们只能在 Kubernetes 集群中可见，对集群之外的世界，这些 IP 都是私有的。

Kubernetes 提供了两种方式让外界能够与 Pod 通信：

* nodePort
Service 通过 Cluster 节点的静态端口对外提供服务。外部可以通过 <NodeIP>:<NodePort> 访问 Service。

* LoadBalancer
Service 利用 cloud provider 提供的 load balancer 对外提供服务，cloud provider 负责将 load balancer 的流量导向 Service。目前支持的 cloud provider 有 GCP、AWS、Azur 等。

##### 网络实现
网络模型有了，如何实现呢？

为了保证网络方案的标准化、扩展性和灵活性，Kubernetes 采用了 Container Networking Interface（CNI）规范。
CNI 是由 CoreOS 提出的容器网络规范，它使用了插件（Plugin）模型创建容器的网络栈。

![29d2e66f23dad44cea1126a277392871.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4094)

CNI 的优点是支持多种容器 runtime，不仅仅是 Docker。CNI 的插件模型支持不同组织和公司开发的第三方插件，这对运维人员来说很有吸引力，可以灵活选择适合的网络方案。

目前已有多种支持 Kubernetes 的网络方案，比如 Flannel、Calico、Canal、Weave Net 等。因为它们都实现了 CNI 规范，用户无论选择哪种方案，得到的网络模型都一样，即每个 Pod 都有独立的 IP，可以直接通信。区别在于不同方案的底层实现不同，有的采用基于 VxLAN 的 Overlay 实现，有的则是 Underlay，性能上有区别,再有就是是否支持 Network Policy。

#### network Policy
Network Policy 是 Kubernetes 的一种资源。Network Policy 通过 Label 选择 Pod，并指定其他 Pod 或外界如何与这些 Pod 通信。

默认情况下，所有 Pod 是非隔离的，即任何来源的网络流量都能够访问 Pod，没有任何限制。当为 Pod 定义了 Network Policy，只有 Policy 允许的流量才能访问 Pod。

不过，不是所有的 Kubernetes 网络方案都支持 Network Policy。比如 Flannel 就不支持，Calico 是支持的。我们接下来将用 Canal 来演示 Network Policy。Canal 这个开源项目很有意思，它用 Flannel 实现 Kubernetes 集群网络，同时又用 Calico 实现 Network Policy。

##### 部署canal
部署 Canal 与部署其他 Kubernetes 网络方案非常类似，都是在执行了 kubeadm init 初始化 Kubernetes 集群之后通过 kubectl apply 安装相应的网络方案。也就是说，没有太好的办法直接切换使用不同的网络方案，基本上只能重新创建集群。

要销毁当前集群，最简单的方法是在每个节点上执行 kubeadm reset。然后就可以按照我们在前面 “部署 Kubernetes Cluster” 的 “初始化 Master” 中的方法初始化集群。


这里有个坑(扒了N多资料) kubeadm reset并不能彻底的清除一些配置文件,所以要手工清除,脚本奉上.
```shell
export KUBECONFIG=/etc/kubernetes/admin.conf && \
rm -rf $HOME/.kube /etc/kubernetes  && \
swapoff -a  
kubeadm init --apiserver-advertise-address 10.0.0.83 --kubernetes-version=v1.13.1 --pod-network-cidr=10.244.0.0/16   && \
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/canal/rbac.yaml  && \
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/canal/canal.yaml
```
部署成功后，可以查看到 Canal 相关组件：
![d4e12d1140845d0b385e0aa7625a92c1.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4095)

Canal 作为 DaemonSet 部署到每个节点，地属于 kube-system 这个 namespace。

#### network policy
我们先部署一个 httpd 应用，其配置文件 httpd2.yaml 为：
![796100dbcd38192c39232e3b281a7de4.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4096)
httpd 有三个副本，通过 NodePort 类型的 Service 对外提供服务。部署应用：

![ed50a0afe658d618bd5333f4327552ee.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4097)
当前没有定义Network policy,验证可以被访问
1. 启动busybox ,ping,可以访问到副本
![c29d82d32d7c85979ae7654bb87e9a87.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4098)
2. 集群节点访问副本
![087c01d13ca657c1c17a6e6397d7e7c0.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4099)
3. 集群外10.0.0.1也可以访问 Service。
![aac9494df5eb3351426434bd15749420.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4100)
##### 创建network policy
1. 创建network policy 文件
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-httpd
spec:
  podSelector:
    matchLabels:
      run: httpd2
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: "true"
    ports:
    - protocol: TCP
      port: 80
```
① 定义将此 Network Policy 中的访问规则应用于 label 为 run: httpd 的 Pod，即 httpd 应用的三个副本 Pod。

② ingress 中定义只有 label 为 access: "true" 的 Pod 才能访问应用。

③ 只能访问 80 端口。

创建Network policy
![3877244d8a7837b9389eda118644dda7.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4101)

验证network policy:
1. busybox不能访问Service
![aa6507fd4da9058b3b9d67bc8d12824a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4102)
如果POD添加了babel access: "true" 就能访问到应用,但是Ping被禁止.
![07dbcd7cafee4c3cfe5992fd474ffcc9.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4103)
2. 集群节点已经不能访问 Service， 也 Ping 不到副本 Pod。

![2bff5b93c405ad6c203ef02d2eb14057.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4104)

3. 集群外也不能再访问service
![bcf03124a8c10c23a9b23afa15d1f7b3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4105)


如果希望让集群节点和集群外（10.0.0.0/24）也能够访问到应用，可以对 Network Policy 做如下修改：
![146aa4e913ea1645e9acbc49a83de2dd.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4106)
应用新的 Network Policy：
```shell
kubectl apply -f policy.yaml
```
现在，集群节点和集群外（10.0.0.0/24）已经能够访问了：
![5e229966893e388ede86fea2e3b4c4a8.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4107)
#### 小结
Kubernetes 采用的是扁平化的网络模型，每个 Pod 都有自己的 IP，并且可以直接通信。

CNI 规范使得 Kubernetes 可以灵活选择多种 Plugin 实现集群网络。

Network Policy 则赋予了 Kubernetes 强大的网络访问控制机制。