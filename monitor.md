# 日志监控

创建 Kubernetes 集群并部署容器化应用只是第一步。一旦集群运行起来，我们需要确保一起正常，所有必要组件就位并各司其职，有足够的资源满足应用的需求。Kubernetes 是一个复杂系统，运维团队需要有一套工具帮助他们获知集群的实时状态，并为故障排查提供及时和准确的数据支持。

#### Weave Scope
Weave Scope 是 Docker 和 Kubernetes 可视化监控工具。Scope 提供了至上而下的集群基础设施和应用的完整视图，用户可以轻松对分布式的容器化应用进行实时监控和问题诊断。

##### 安装部署Scope
安装 Scope 的方法很简单，执行如下命令：

kubectl apply --namespace weave -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"

部署成功后，有如下相关组件：
* DaemonSet weave-scope-agent，集群每个节点上都会运行的 scope agent 程序，负责收集数据。
* Deployment weave-scope-app，scope 应用，从 agent 获取数据，通过 Web UI 展示并与用户交互。
* Service weave-scope-app，默认是 ClusterIP 类型，为了方便已通过 kubectl edit 修改为 NodePort。
![e361d02127ecf19ea55a78cdfd2bde41.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4123)

##### 使用Scope
记得先修改clusterIP为NodePort,之后
浏览器访问 http://10.0.0.81:30693/，Scope 默认显示当前所有的 Controller（Deployment、DaemonSet 等）。
![6e5f6eee2aab4d851014cd076073987c.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4124)


##### 拓扑结构
Scope 会自动构建应用和集群的逻辑拓扑。比如点击顶部 PODS，会显示所有 Pod 以及 Pod 之间的依赖关系。

![9f590742fd5c8580b7aab8cb009f8be3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4125)
点击 HOSTS，也会显示各个节点之间的关系。

##### 实时资源监控
可以在 Scope 中查看资源的 CPU 和内存使用情况。

支持的资源有 Host、Pod 和 Container。
![d5f6a7511a153b8854bf4091b18148fa.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4126)


##### 在线操作
Scope 还提供了便捷的在线操作功能，比如选中某个 Host，点击 >_ 按钮可以直接在浏览器中打开节点的命令行终端：
![d07799da76f6944130004b684c408044.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4127)


scale up /down
![0632e4fb11051f22200a3bd1defbf2f1.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4128)

可以查看 Pod 的日志：
![b692b71fb07917a9988aeb019324a563.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4129)

可以 attach、restart、stop 容器，以及直接在 Scope 中排查问题：
![8576d5b867577bbcd97305d77d28d3d8.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4130)

##### 强大的搜索功能
Scope 支持关键字搜索和定位资源。

![20d099f2243e5a91bbe746d648d96536.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4131)
还可以进行条件搜索，比如查找和定位 MEMORY > 100M 的 Pod。


...还有些别的功能,可以自行探索

#### kube-prometheus
[官方文档](
https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus)
Prometheus Operator 是 CoreOS 开发的基于 Prometheus 的 Kubernetes 监控方案，也可能是目前功能最全面的开源方案,同时也是官方推荐的.我们先通过截图了解一下它能干什么。

Prometheus Operator 通过 Grafana 展示监控数据，预定义了一系列的 Dashboard：
![67c2b82f4a600b6ee019121e58419a2f.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4133)

可以监控 Kubernetes 集群的整体健康状态：
![33d00ed91f6ec2edba4c153583969006.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4135)

整个集群的资源使用情况：
![3b8de71b60636dea94e20682cd641920.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4136)
![6431b5bc5bbe48b2ace44c64ae950fcc.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4137)
Kubernetes 各个管理组件的状态：

![108755ca4d6b57479675bb094b10ecc7.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4138)
节点的资源使用情况：
![a317fef659e90aeea4fd2fb8cd213b91.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4139)

Deployment的运行状态:
![c32c9f1ca32467a79887dc8e6f693a52.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4140)

Pod的运行状态等
![9c2b4c5cf57c0d277c86359419cadc9e.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4141)

这些 Dashboard 展示了从集群到 Pod 的运行状况，能够帮助用户更好地运维 Kubernetes。而且 Prometheus Operator 迭代非常快，相信会继续开发出更多更好的功能，所以值得我们花些时间学习和实践。

##### Prometheus架构
因为 Prometheus Operator 是基于 Prometheus 的，我们需要先了解一下 Prometheus。

Prometheus 是一个非常优秀的监控工具。准确的说，应该是监控方案。Prometheus 提供了数据搜集、存储、处理、可视化和告警一套完整的解决方案。Prometheus 的核心架构如下图所示：

![258aa2acee0281c316646e1ee3aa8a54.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4142)

* Prometheus Server
Prometheus Server 负责从 Exporter 拉取和存储监控数据，并提供一套灵活的查询语言（PromQL）供用户使用。

* Exporter
Exporter 负责收集目标对象（host, container...）的性能数据，并通过 HTTP 接口供 Prometheus Server 获取。

* 可视化组件
监控数据的可视化展现对于监控方案至关重要。以前 Prometheus 自己开发了一套工具，不过后来废弃了，因为开源社区出现了更为优秀的产品 Grafana。Grafana 能够与 Prometheus 无缝集成，提供完美的数据展示能力。

* Alertmanager
用户可以定义基于监控数据的告警规则，规则会触发告警。一旦 Alermanager 收到告警，会通过预定义的方式发出告警通知。支持的方式包括 Email、PagerDuty、Webhook 等.

##### Prometheus Operator 架构
Prometheus Operator 的目标是尽可能简化在 Kubernetes 中部署和维护 Prometheus 的工作。其架构如下图所示：
![2eec8684458fd0a4c5cb277cd9baa0b4.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4143)

* Operator
Operator 即 Prometheus Operator，在 Kubernetes 中以 Deployment 运行。其职责是部署和管理 Prometheus Server，根据 ServiceMonitor 动态更新 Prometheus Server 的监控对象。

* Prometheus Server
Prometheus Server 会作为 Kubernetes 应用部署到集群中。为了更好地在 Kubernetes 中管理 Prometheus，CoreOS 的开发人员专门定义了一个命名为 Prometheus 类型的 Kubernetes 定制化资源。我们可以把 Prometheus看作是一种特殊的 Deployment，它的用途就是专门部署 Prometheus Server。

* Service
这里的 Service 就是 Cluster 中的 Service 资源，也是 Prometheus 要监控的对象，在 Prometheus 中叫做 Target。每个监控对象都有一个对应的 Service。比如要监控 Kubernetes Scheduler，就得有一个与 Scheduler 对应的 Service。当然，Kubernetes 集群默认是没有这个 Service 的，Prometheus Operator 会负责创建。

* ServiceMonitor
Operator 能够动态更新 Prometheus 的 Target 列表，ServiceMonitor 就是 Target 的抽象。比如想监控 Kubernetes Scheduler，用户可以创建一个与 Scheduler Service 相映射的 ServiceMonitor 对象。Operator 则会发现这个新的 ServiceMonitor，并将 Scheduler 的 Target 添加到 Prometheus 的监控列表中。
ServiceMonitor 也是 Prometheus Operator 专门开发的一种 Kubernetes 定制化资源类型。

* Alertmanager
除了 Prometheus 和 ServiceMonitor，Alertmanager 是 Operator 开发的第三种 Kubernetes 定制化资源。我们可以把 Alertmanager 看作是一种特殊的 Deployment，它的用途就是专门部署 Alertmanager 组件。
##### prometheus-operator部署
要求: 
Kubernetes 1.10+ with Beta APIs
Helm 2.10+ (For a workaround using an earlier version see below)

1.github地址
https://github.com/helm/charts/tree/master/stable/prometheus-operator
 2.使用helm部署
确保helm环境正常
helm version

3. 安装
```shell
helm install --name my-prom stable/prometheus-operator
```

安装:
![ae43c0e05c7977c99de09b41c294f7b1.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4144)

可能会报错
解决方法:
* 重置helm
```shell
helm reset --force
kubectl create serviceaccount --namespace kube-system tiller

kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}' 
```
* 配置rbac 
kubectl create -f rbac-config.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tiller-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: ""
```
4.检查pod , svc等
```shell
kubectl -n default get pods,svc,deployment,servicemonitor -l "release=my-prom"  
```
![95e44f44a77b0edb7595c0a1fb91173d.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4145)


8.全部起来后，访问service/kube-prometheus-grafana 
可以看到各种监控数据
默认用户名: admin
默认密码:  prom-operator








