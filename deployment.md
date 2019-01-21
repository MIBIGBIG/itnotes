# Deployment管理
#### deployment
Kubernetes 通过各种 Controller 来管理 Pod 的生命周期。为了满足不同业务场景，Kubernetes 开发了 Deployment、ReplicaSet、DaemonSet、StatefuleSet、Job 等多种 Controller。

先从例子开始，运行一个 Deployment：
```shell
kubectl run nginx-pod --image=nginx:1.12 --replicas=2
```
上面的命令将部署包含两个副本的 pod nginx-pod，容器的 image 为 nginx:1.12

下面详细分析 Kubernetes 都做了些什么工作。

```shell
[root@k8s03 ~]# kubectl get deployment nginx-pod
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-pod   2/2     2            2           3m12s
```
然后用kubectl describe pod nginx-pod 来分析Kubelet做了什么
```shell
 kubectl describe pod nginx-pod
 kubectl describe deployment nginx-pod
```
![614171ad0d686457f0e77910b59fc11b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3867)
![935b0ac2fe40d2c756510f0788b54d02.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3868)

大部分内容都是自解释的，我们重点看最下面部分。这里告诉我们创建了一个 replica set nginx-pod-5f67d755b9 to 2，Events 是 Deployment 的日志，记录了 ReplicaSet 的启动过程。

通过上面的分析，也验证了 Deployment 通过 ReplicaSet 来管理 Pod 的事实。接着我们将注意力切换到 nginx-pod上，执行 kubectl describe replicaset：

```shell
kubectl get replicaset
```
![3220753ba011b99471ba4ef59919c50c.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3869)
Controlled By 指明此 ReplicaSet 是由 Deployment nginx-deployment 创建。Events 记录了两个副本 Pod 的创建。接着我们来看 Pod，执行 kubectl get pod：
```shell
kubectl get pod
kubectl describe pod [pod-name]
```
![f9fa705fd950f9c99fade9fe5c674d07.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3870)

Controlled By 指明此 Pod 是由 ReplicaSet/nginx-pod-5f67d755b9 创建。Events 记录了 Pod 的启动过程。如果操作失败（比如 image 不存在），也能在这里查看到原因。

##### 创建过程分析
* 用户通过 kubectl 创建 Deployment。
* Deployment 创建 ReplicaSet。
* ReplicaSet 创建 Pod。

![497978c5a5a3f271b2047bf68149a272.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3871)

从上图也可以看出，对象的命名方式是：子对象的名字 = 父对象名字 + 随机字符串或数字。

#### kubernetes创建资源的两种方式

命令 vs 配置文件
Kubernetes 支持两种方式创建资源：
1. 用 kubectl 命令直接创建，比如：
```shell
kubectl run nginx-pod --image=nginx:1.12 --replicas=2
```
在命令行中通过参数指定资源的属性。

2. 通过配置文件和 kubectl apply 创建，要完成前面同样的工作，可执行命令：

kubectl apply -f nginx.yml
nginx.yml 的内容为：

```yml
apiVersion: extensions/v1.1
kind: Deployment
metadata:
  name: nginx-pod
spec:
  replicas: 2
  template:
    metadata:
      label:
        app: web_server
    spec:
      containers:
      - name: nginx
        image: nginx:1.12 
```
资源的属性写在配置文件中，文件格式为 YAML。


下面对这两种方式进行比较。
***
* 基于命令的方式：
简单直观快捷，上手快。
适合临时测试或实验。
***
* 基于配置文件的方式：
1.配置文件描述了 What，即应用最终要达到的状态。
2.配置文件提供了创建资源的模板，能够重复部署。
3.可以像管理代码一样管理部署。
4.适合正式的、跨环境的、规模化部署。
5.要求熟悉配置文件的语法，有一定难度。

kubectl apply 不但能够创建 Kubernetes 资源，也能对资源进行更新，非常方便。不过 Kubernets 还提供了几个类似的命令，例如 kubectl create、kubectl replace、kubectl edit 和 kubectl patch。

为避免造成不必要的困扰，尽量只使用 kubectl apply，
此命令已经能够应对超过 90% 的场景，事半功倍。


#### 如何读懂Deployment YAML
既然要用 YAML 配置文件部署应用，现在就很有必要了解一下 Deployment 的配置格式，其他 Controller（比如 DaemonSet）非常类似。

仍然以nginx-pod为例子,配置文件如下所示:
![f31b5934daa90aa3dfb799b1dfa1c52d.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3872)

① apiVersion 是当前配置格式的版本。
② kind 是要创建的资源类型，这里是 Deployment。
③ metadata 是该资源的元数据，name 是必需的元数据项。
④ spec 部分是该 Deployment 的规格说明。
⑤ replicas 指明副本数量，默认为 1。
⑥ template 定义 Pod 的模板，这是配置文件的重要部分。
⑦ metadata 定义 Pod 的元数据，至少要定义一个 label。label 的 key 和 value 可以任意指定。
⑧ spec 描述 Pod 的规格，此部分定义 Pod 中每一个容器的属性，name 和 image 是必需的。

此 nginx.yml 是一个最简单的 Deployment 配置文件

执行 kubectl apply -f nginx.yml：
```shell
[root@k8s03 k8s]# kubectl apply -f nginx.yml
deployment.extensions/nginx-pod created
```
部署成功。同样地，通过 kubectl get 查看 nginx-pod 的各种资源：
```shell
kubectl get deployment
kubectl get replicaset
kubectl get pod -o wide
```

Deployment、ReplicaSet、Pod 都已经就绪。如果要删除这些资源，执行 kubectl delete deployment nginx-deployment 或者 kubectl delete -f nginx.yml。

```shell
[root@k8s03 k8s]# kubectl delete -f nginx.yml

deployment.extensions "nginx-pod" deleted
```
#### deployment扩展和缩减

伸缩（Scale Up/Down）是指在线增加或减少 Pod 的副本数。
Deployment nginx-pod 初始是两个副本。
39e8e6b49f28337cd8b07008bd33970d.png
k8s01 和k8s05上各跑 了一个副本.现在修改nginx.yml,将副本改成5个
3ff9a199487003d8bbf94b3835e642eb.png
再次执行 kubectl apply：
```shell
kubectl apply -f nginx.yml
```

三个新副本被创建并调度到 k8s01和k8s05 上。

出于安全考虑，默认配置下 Kubernetes 不会将 Pod 调度到 Master 节点。如果希望将 k8s-master 也当作 Node 使用，可以执行如下命令：
```shell
kubectl taint node [k8s03] node-role.kubernetes.io/master-
```
如果要恢复master only 状态,执行如下命令:
```shell
kubectl taint node k8s03 node-role.kubernetes.io/master="":NoSchedule
```

接下来修改配置文件，将副本数减少为 3 个，重新执行 kubectl apply：

![519f7e70064e0ec2e3254136d2d8c0be.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3875)

可以看到两个副本被删除，最终保留了 3 个副本。


#### k8s如何failover?
有 3 个 nginx 副本分别运行在 k8s01 和 k8s05 上。现在模拟 k8s01 故障，关闭该节点。
![1c52494405df96fc365054b3401aa66f.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3877)

等待一段时间，Kubernetes 会检查到 k8s01 不可用，将 k8s01 上的 Pod 标记为 Unknown 状态，并在 k8s05 上新创建两个 Pod，维持总副本数为 3。
![e13eb444f190853fba9e1040a290ad7d.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3878)
当 k8s01 恢复后，Unknown 的 Pod 会被删除，不过已经运行的 Pod 不会重新调度回 k8s01。
![8d56836dc10da14606e9bcc87a536ecc.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3879)

删除nginx-pod
```shell
[root@k8s03 k8s]# kubectl delete deployment nginx-pod
deployment.extensions "nginx-pod" deleted
```

#### k8s如何控制pod位置
默认配置下，Scheduler 会将 Pod 调度到所有可用的 Node。不过有些情况我们希望将 Pod 部署到指定的 Node，比如将有大量磁盘 I/O 的 Pod 部署到配置了 SSD 的 Node；或者 Pod 需要 GPU，需要运行在配置了 GPU 的节点上。

Kubernetes 是通过 label 来实现这个功能的。

label 是 key-value 对，各种资源都可以设置 label，灵活添加各种自定义属性。比如执行如下命令标注 k8s01 是配置了 SSD 的节点。

```shell
kubectl label node k8s01 disktype=ssd
```
然后通过 kubectl get node --show-labels 查看节点的 label。

![0e0d844d728e583da92720a182cea5f4.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3880)

disktype=ssd 已经成功添加到 k8s01，除了 disktype，Node 还有几个 Kubernetes 自己维护的 label。

有了 disktype 这个自定义 label，接下来就可以指定将 Pod 部署到 k8s01。编辑 nginx.yml：
![3c55115a06a1890cd29aba770893bbed.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3881)

在 Pod 模板的 spec 里通过 nodeSelector 指定将此 Pod 部署到具有 label disktype=ssd 的 Node 上。

部署 Deployment 并查看 Pod 的运行节点：
![3de41fb8dc08ebf537d09a6bd774ac26.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3882)

```shell
kubectl apply -f nginx.yml
kubectl get pod -o wide
```

全部4个副本都运行在k8s01上,符合预期
要删除label disktype,执行如下命令
```shell
kubectl label node k8s01 disktype-
```
-即删除.

![06e0cf3af4924a833ee8e186a19e3bcc.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3883)
不过此时 Pod 并不会重新部署，依然在 k8s01 上运行。
![bcdfda6a58e34d97e3183b15fa570a97.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3884)

除非在 nginx.yml 中删除 nodeSelector 设置，然后通过 kubectl apply 重新部署。

![95746f20ba109f8b9c0eafe492cc9d2f.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3885)
Kubernetes 会删除之前的 Pod 并调度和运行新的 Pod。

