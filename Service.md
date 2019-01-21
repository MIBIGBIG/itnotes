# Service管理

#### 通过service访问POD
通常不应该期望 Kubernetes Pod 是健壮的，而是要假设 Pod 中的容器很可能因为各种原因发生故障而死掉。Deployment 等 controller 会通过动态创建和销毁 Pod 来保证应用整体的健壮性。换句话说，Pod 是脆弱的，但应用是健壮的。

每个 Pod 都有自己的 IP 地址。当 controller 用新 Pod 替代发生故障的 Pod 时，新 Pod 会分配到新的 IP 地址。这样就产生了一个问题：

如果一组 Pod 对外提供服务（比如 HTTP），它们的 IP 很有可能发生变化，那么客户端如何找到并访问这个服务呢？

Kubernetes 给出的解决方案也是 Service。

##### 创建service

Kubernetes Service 从逻辑上代表了一组 Pod，具体是哪些 Pod 则是由 label 来挑选。Service 有自己 IP，而且这个 IP 是不变的。客户端只需要访问 Service 的 IP，Kubernetes 则负责建立和维护 Service 与 Pod 的映射关系。无论后端 Pod 如何变化，对客户端不会有任何影响，因为 Service 没有变。

来看个例子，创建下面的这个 Deployment：

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: httpd
spec:
  replicas: 3
  template:
    metadata:
      labels:
        run: httpd
    spec:
      containers:
      - name: httpd
        image: httpd
        ports:
        - containerPort: 80
```
启动了三个 Pod，运行 httpd 镜像，label 是 run: httpd，Service 将会用这个 label 来挑选 Pod。
![c6f012ff20521c85ba113f685962b8ba.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3911)
Pod 分配了各自的 IP，这些 IP 只能被 Kubernetes Cluster 中的容器和节点访问。

接下来创建Service,其配置文件如下:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
spec:
  selector:
    run: httpd
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
```
① v1 是 Service 的 apiVersion。
② 指明当前资源的类型为 Service。
③ Service 的名字为 httpd-svc。
④ selector 指明挑选那些 label 为 run: httpd 的 Pod 作为 Service 的后端。
⑤ 将 Service 的 8080 端口映射到 Pod 的 80 端口，使用 TCP 协议。


执行 kubectl apply 创建 Service httpd-svc。
![6ef4b4e7096ae969c43542a92b06bb9c.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3912)

httpd-svc 分配到一个 CLUSTER-IP 10.102.120.255。可以通过该 IP 访问后端的 httpd Pod。
```shell
curl 10.102.120.255:8080

```
![d7d1a5f6f842deabd06382f7efd69c5e.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3913)

根据前面的端口映射，这里要使用 8080 端口。另外，除了我们创建的 httpd-svc，还有一个 Service kubernetes，Cluster 内部通过这个 Service 访问 kubernetes API Server。

通过 kubectl describe 可以查看 httpd-svc 与 Pod 的对应关系。

```shell
kubectl describe service httpd-service
```

![674ea4093c13f89ee61ef7cde47a418a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3914)

Endpoints 罗列了三个 Pod 的 IP 和端口。我们知道 Pod 的 IP 是在容器中配置的，那么 Service 的 Cluster IP 又是配置在哪里的呢？CLUSTER-IP 又是如何映射到 Pod IP 的呢？
答案是iptables

#### service IP 原理
Service Cluster IP 是一个虚拟 IP，是由 Kubernetes 节点上的 iptables 规则管理的。

可以通过 iptables-save 命令打印出当前节点的 iptables 规则，因为输出较多，这里只截取与 httpd-svc Cluster IP 10.102.120.255 相关的信息：

```shell
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.102.120.255/32 -p tcp -m comment --comment "default/httpd-svc: cluster IP" -m tcp --dport 8080 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.102.120.255/32 -p tcp -m comment --comment "default/httpd-svc: cluster IP" -m tcp --dport 8080 -j KUBE-SVC-RL3JAE4GN7VOGDGP
```
这两条规则的含义是：

如果 Cluster 内的 Pod（源地址来自 10.244.0.0/16）要访问 httpd-svc，则允许。

其他源地址访问 httpd-svc，跳转到规则 KUBE-SVC-RL3JAE4GN7VOGDGP。

KUBE-SVC-RL3JAE4GN7VOGDGP 规则如下：

![f4bcade15db471a855bbbd3afc6304a8.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3915)
* 1/3 的概率跳转到规则 KUBE-SEP-4DQNMIHOZ4LIT6QF。

* 1/3 的概率（剩下 2/3 的一半）跳转到规则 KUBE-SEP-KCBI42HJSRG3Z4WD

* 1/3 的概率跳转到规则 KUBE-SEP-X5UEIL7VSMT4VZUN。

即将请求分别转发到后端的三个 Pod。通过上面的分析，我们得到如下结论：

iptables 将访问 Service 的流量转发到后端 Pod，而且使用类似轮询的负载均衡策略。

另外需要补充一点：Cluster 的每一个节点都配置了相同的 iptables 规则，这样就确保了整个 Cluster 都能够通过 Service 的 Cluster IP 访问 Service。


![57f1046e49d919dccd6a8cec93539208.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3916)

除了直接通过 Cluster IP 访问到 Service，DNS 是更加便捷的方式，


#### 推荐方式: DNS访问Service
在 Cluster 中，除了可以通过 Cluster IP 访问 Service，Kubernetes 还提供了更为方便的 DNS 访问。

CoreDNS 成为 Kubernetes 的默认 DNS 服务器

在 1.11 版本中，开发团队宣布 CoreDNS 已实现基于 DNS 服务发现的普遍可用性。在最新的 1.13 版本中，CoreDNS 正式取代 kuber-dns 成为 Kubernetes 中的默认 DNS 服务器。CoreDNS 是一种通用的、权威的 DNS 服务器，能够提供与 Kubernetes 向下兼容且具备可扩展性的集成能力。由于 CoreDNS 自身单一可执行文件与单一进程的特性，因此 CoreDNS 的活动部件数量会少于之前的 DNS 服务器，且能够通过创建自定义 DNS 条目来支持各类灵活的用例。此外，由于 CoreDNS 采用 Go 语言编写，它具有强大的内存安全性。

CoreDNS 现在是 Kubernetes 1.13 及后续版本推荐的 DNS 解决方案，Kubernetes 已将常用测试基础设施架构切换为默认使用 CoreDNS ，因此，开发团队建议用户也尽快完成切换。KubeDNS 仍将至少支持一个版本，但现在是时候开始规划迁移了。另外，包括 1.11 中 Kubeadm 在内的许多 OSS 安装工具也已经进行了切换。

```shell
[root@k8s03 k8s]# kubectl get deployment --namespace=kube-system
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           3d19h
```
coredns 是一个 DNS 服务器。每当有新的 Service 被创建，coredns 会添加该 Service 的 DNS 记录。Cluster 中的 Pod 可以通过 <SERVICE_NAME>.<NAMESPACE_NAME> 访问 Service。

比如可以用 httpd-svc 访问 Service httpd-svc。
![d1b58f7c0875491c376d634ef66c44a7.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3917)

DNS 服务器是 coredns它本身是部署在 kube-system namespace 中的一个 Service。

如果要访问其他 namespace 中的 Service，就必须带上 namesapce 了。kubectl get namespace 查看已有的 namespace。
![c8650a6e7a6d72fd7b50a440e42582d5.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3918)


在 kube-public 中部署 Service httpd2-svc，配置如下：
```shell
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: httpd2
  namespace: kube-public
spec:
  replicas: 3
  template:
    metadata:
      labels:
        run: httpd2
    spec:
      containers:
      - name: httpd2
        image: httpd
        port:
        - containerPort: 80
        
---
apiVersion: v1
kind: Service
metadata:
  name: httpd2-svc
  namespace: kube-public
spec:
  selector:
    run: httpd2
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
        
```

通过 namespace: kube-public 指定资源所属的 namespace。多个资源可以在一个 YAML 文件中定义，用 --- 分割。执行 kubectl apply 创建资源：
```shell
kubectl apply -f httpd2.yml
```
查看 kube-public 的 Service：
![018e0f2ff56b8e6947ea43d930c69db7.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3919)

在 busybox Pod 中访问 httpd2-svc：
```shell
kubectl run busybox --rm -it --image=busybox
```
通过namespace A记录访问
![bf368eabb87c4ed52f9bf55d9b49b086.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3921)
官方解释
>service .A records

>“Normal” (not headless) Services are assigned a DNS A record for a name of the form my-svc.my-namespace.svc.cluster.local. This resolves to the cluster IP of the Service.
“Headless” (without a cluster IP) Services are also assigned a DNS A record for a name of the form my-svc.my-namespace.svc.cluster.local. Unlike normal Services, this resolves to the set of IPs of the pods selected by the Service. Clients are expected to consume the set or else use standard round-robin selection from the set.

因为属于不同的 namespace，必须使用 httpd2-svc.kube-public 才能访问到。

Kubernetes 集群内部可以通过 Cluster IP 和 DNS 访问 Service，那么集群外部如何访问呢？

#### 外网访问service
除了 Cluster 内部可以访问 Service，很多情况我们也希望应用的 Service 能够暴露给 Cluster 外部。Kubernetes 提供了多种类型的 Service，默认是 ClusterIP。

* ClusterIP 
Service 通过 Cluster 内部的 IP 对外提供服务，只有 Cluster 内的节点和 Pod 可访问，这是默认的 Service 类型.

* NodePort 
Service 通过 Cluster 节点的静态端口对外提供服务。Cluster 外部可以通过 <NodeIP>:<NodePort> 访问 Service。

* LoadBalancer 
Service 利用 cloud provider 特有的 load balancer 对外提供服务，cloud provider 负责将 load balancer 的流量导向 Service。目前支持的 cloud provider 有 GCP、AWS、Azure 等。

实践NodePort,Service httpd-svc的配置文件修改:
![ff50bad933d87ef2a43db8015718c9a4.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3923)
添加 type: NodePort，重新创建 httpd-svc。
```shell
kubectl apply -f httpd-service
kubectl get service httpd-svc
```
![f88808b1c454c37aca25fa691528d116.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3924)

Kubernetes 依然会为 httpd-svc 分配一个 ClusterIP，不同的是：

EXTERNAL-IP 为 none，表示可通过 Cluster 每个节点自身的 IP 访问 Service。

PORT(S) 为 8080:32312。8080 是 ClusterIP 监听的端口，32312 则是节点上监听的端口。Kubernetes 会从 30000-32767 中分配一个可用的端口，每个节点都会监听此端口并将请求转发给 Service。
下面测试 NodePort 是否正常工作。
![1f1da4bf7e73655314a2f2c907e92584.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3925)

通过三个节点 IP + 32312 端口都能够访问 httpd-svc。

这里探讨一个问题：Kubernetes 是如何将 <NodeIP>:<NodePort> 映射到 Pod 的呢？
与 ClusterIP 一样，也是借助了 iptables。与 ClusterIP 相比，每个节点的 iptables 中都增加了下面两条规则：
![1ddd10c44ec50753ff78f5336759beaf.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3926)
规则的含义是：访问当前节点 32312 端口的请求会应用规则 KUBE-SVC-RL3JAE4GN7VOGDGP，内容为：
```shell
-A KUBE-SVC-RL3JAE4GN7VOGDGP -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-X5UEIL7VSMT4VZUN
-A KUBE-SVC-RL3JAE4GN7VOGDGP -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-KCBI42HJSRG3Z4WD
-A KUBE-SVC-RL3JAE4GN7VOGDGP -j KUBE-SEP-4DQNMIHOZ4LIT6QF
```
其作用就是负载均衡到每一个 Pod。

NodePort 默认是的随机选择，不过我们可以用 nodePort 指定某个特定端口。

![d89c0c83b740f76ea2ccb630ae179de9.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3929)


现在配置文件中就有三个 Port 了：
nodePort 是节点上监听的端口。
port 是 ClusterIP 上监听的端口。
targetPort 是 Pod 监听的端口。

最终，Node 和 ClusterIP 在各自端口上接收到的请求都会通过 iptables 转发到 Pod 的 targetPort。
应用新的 nodePort 并验证：
![04e2256e65daecd10bbfe6388d9da9f7.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3928)

nodePort: 31000 已经生效了。

