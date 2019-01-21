# helm包管理

#### 为什么需要Helm?
每个成功的软件平台都有一个优秀的打包系统，比如 Debian、Ubuntu 的 apt，Redhat、Centos 的 yum。而 Helm 则是 Kubernetes 上的包管理器。

Helm 到底解决了什么问题？为什么 Kubernetes 需要 Helm？

答案是：Kubernetes 能够很好地组织和编排容器，但它缺少一个更高层次的应用打包工具，而 Helm 就是来干这件事的。

示例: 
比如对于个MySQL服务,Kubernetes需要部署下面这些对象:

1. Service,让外界能够访问MySQL.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-mysql
  labels:
    app: my-mysql
spec: 
  ports:
  - name: mysql
    port: 3306
    targetPort: mysql
  selector:
    app: my-mysql
```
2. 定义secret,设置Mysql的密码
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-mysql
  labels:
    app: my-mysql
type: Opaque
data:
  mysql-root-password: "cm9vdA=="
  mysql-password: "MTIzNDU2"
```
3.PersistentVolumeClaim，为 MySQL 申请持久化存储空间。

```shell
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-mysql
  labels:
    app: my-mysql
spec:
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: "8Gi"
```

4. Deployment，部署 MySQL Pod，并使用上面的这些支持对象。
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-mysql
  labels:
    app: my-mysql
spec:
  template:
    metadata:
      labels:
        app: my-mysql
    sepc:
      containers:
      - name: my-mysql
        image: "mysql:5.6"
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-mysql
              key: mysql-root-password
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: my-mysql
              key: mysql-password
        - name: MYSQL_USER
          value: ""
        - name: MYSQL_DATABASE
          value: ""
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: my-mysql         
```
可以将上面这些配置保存到对象各自的文件中，或者集中写进一个配置文件，然后通过 kubectl apply -f 部署。

到目前为止，Kubernetes 对服务的部署支持得都挺好，如果应用只由一个或几个这样的服务组成，上面的部署方式完全足够了。
但是，如果我们开发的是微服务架构的应用，组成应用的服务可能多达十个甚至几十上百个，这种组织和管理应用的方式就不好使了：
* 很难管理、编辑和维护如此多的服务。每个服务都有若干配置，缺乏一个更高层次的工具将这些配置组织起来。
* 很难将这些服务作为一个整体统一发布。部署人员需要首先理解应用都包含哪些服务，然后按照逻辑顺序依次执行 kubectl apply。即缺少一种工具来定义应用与服务，以及服务与服务之间的依赖关系。
* 不能高效地共享和重用服务。比如两个应用都要用到 MySQL 服务，但配置的参数不一样，这两个应用只能分别拷贝一套标准的 MySQL 配置文件，修改后通过 kubectl apply 部署。也就是说不支持参数化配置和多环境部署。
* 不支持应用级别的版本管理。虽然可以通过 kubectl rollout undo 进行回滚，但这只能针对单个 Deployment，不支持整个应用的回滚。
* 不支持对部署的应用状态进行验证。比如是否能通过预定义的账号访问 MySQL。虽然 Kubernetes 有健康检查，但那是针对单个容器，我们需要应用（服务）级别的健康检查。


Helm 能够解决上面这些问题，Helm 帮助 Kubernetes 成为微服务架构应用理想的部署平台。

#### Helm组成
Helm有两个重要的概念: chart和release
* chart 是创建一个应用的信息集合，包括各种 Kubernetes 对象的配置模板、参数定义、依赖关系、文档说明等。chart 是应用部署的自包含逻辑单元。可以将 chart 想象成 apt、yum 中的软件安装包。
* release 是 chart 的运行实例，代表了一个正在运行的应用。当 chart 被安装到 Kubernetes 集群，就生成一个 release。chart 能够多次安装到同一个集群，每次安装都是一个 release。

Helm 是包管理工具，这里的包就是指的 chart。Helm 能够：
* 查找并使用已经打包为 Kubernetes charts 的流行软件
* 分享您自己的应用作为 Kubernetes charts
* 为 Kubernetes 应用创建可重复执行的构建
* 为您的 Kubernetes 清单文件提供更智能化的管理
* 管理 Helm 软件包的发布


Helm 包含两个组件：Helm 客户端 和 Tiller 服务器。

两个组件:Helm客户端和Trller服务器
![9e9b1fe3f4b634582e696863c7b5cf39.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4044)

Helm 客户端是终端用户使用的命令行工具
* 本地开发chart
* 管理chart仓库
* 与tiller服务器交互
* 在k8s集群上安装chart
* 查看release信息
* 升级卸载release

tiller服务器运行在kubernetes集群中,会处理helm客户端的请求,与k8s api server交互. tiller服务器负载
* 监听helm客户端的请求
* 通过chart构建release
* 在k8s中安装chart,并跟踪release状态
* 通过API server升级或者卸载已有的release
简单来说：Helm 客户端负责管理 chart；Tiller 服务器负责管理 release。


#### helm的安装和部署

1.安装客户端...
下载一个安装脚本并使用bash执行.
```shell
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
helm version
```
目前只能查看到客户端的版本，服务器还没有安装。

helm 有很多子命令和参数，为了提高使用命令行的效率，通常建议安装 helm 的 bash 命令补全脚本，方法如下：
```shell
helm completion bash > .helmrc
echo "source .helmrc" >> .bashrc
. .helmrc
```
![bc84f5592f666e939d87976272221f56.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4045)
tab补全helm命令
2. 安装Tiller服务器
只需要执行helm init
![1e37f65b887c7ddcfa1cd6bfb511c5f8.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4046)
Tiller 本身也是作为容器化应用运行在 Kubernetes Cluster 中的：
```shell
kubectl get service --namespace=kube-system
kubectl get deployment --namespace=kube-system
kubectl get pods --namespace=kube-system 
kubectl get --namespace=kube-system pod tiller-deploy-6f8d4f6c9c-xhd44
```
![edcb51599e0bca4c18a8598205e60fc3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4047)
可以看到Tiller的pod,service,deployment
也可以直接查看到服务器的版本信息了
![fedae6c8115aeb598d8f5769350b5c06.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4048)

#### HELM的使用
安装成功后,可以向yum管理软件包一样管理chart.
apt和yum的软件包存放在仓库中,helm也有自己的仓库.
默认安装时会帮你自动安装两个repo.
```shell
helm repo list

```
![11555607b555ec67132f0a8e7e290632.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4049)
Helm 安装时已经默认配置好了这两个仓库：stable 和 local。stable 是官方仓库，local 是用户存放自己开发的 chart 的本地仓库。

helm search 会显示 chart 位于哪个仓库，比如 local/cool-chart 和 stable/acs-engine-autoscaler。

用户可以通过 helm repo add 添加更多的仓库，比如企业的私有仓库，仓库的管理和维护方法请参考官网文档 https://docs.helm.sh

与 apt 和 yum 一样，helm 也支持关键字搜索：
```shell
helm search mysql
```
![b9064acc24c2952e609ef17cc9ac88c1.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4050)
包括 DESCRIPTION 在内的所有信息，只要跟关键字匹配，都会显示在结果列表中。
安装也很简单,执行如下命令就可以安装mysql,但如果有下面的错误,通常是因为 Tiller 服务器的权限不足。
```shell
[root@k8s03 ~]# helm install stable/mysql
Error: no available release name found
```
执行如下命令添加权限:
```shell
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
再次执行helm install stable/mysql
![bfb1af7906a2cf976a71663290ee26c3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4051)
输出分为三部分:
① chart 本次部署的描述信息：

NAME 是 release 的名字，因为我们没用 -n 参数指定，Helm 随机生成了一个，这里是 looming-poodle。

NAMESPACE 是 release 部署的 namespace，默认是 default，也可以通过 --namespace 指定。

STATUS 为 DEPLOYED，表示已经将 chart 部署到集群。

② 当前 release 包含的资源：ConfigMap、Service、Deployment、Secret 和 PersistentVolumeClaim，pod其名字都是 looming-poodle-mysql，命名的格式为 ReleasName-ChartName。

③ NOTES 部分显示的是 release 的使用方法。比如如何访问 Service，如何获取数据库密码，以及如何连接数据库等。

通过 kubectl get 可以查看组成 release 的各个对象：
```shell
 kubectl get service looming-poodle-mysql
 kubectl get deployment looming-poodle-mysql
 kubectl get pvc looming-poodle-mysql
 kubectl get configmap looming-poodle-mysql-test
 kubectl get Secret looming-poodle-mysql
```
由于还没有准备PV,所以PVC还是pending状态

helm使用方法非常类似yum,用helm管理k8s对于熟悉yum的来说非常方便
如
```shell
helm ls     列出release
helm del looming-poodle   删除应用
helm install  安装
helm fetch 从repo下载到本地
helm create 创建新chart
helm history 历史记录
helm inspect 检查chart
......

```
还有一些,就不一一罗列了,可以使用 helm help 来获取


#### chart 
chart 是 Helm 的应用打包格式。chart 由一系列文件组成，这些文件描述了 Kubernetes 部署应用时所需要的资源，比如 Service、Deployment、PersistentVolumeClaim、Secret、ConfigMap 等。

单个的 chart 可以非常简单，只用于部署一个服务，比如 Memcached；chart 也可以很复杂，部署整个应用，比如包含 HTTP Servers、 Database、消息中间件、cache 等。

chart 将这些文件放置在预定义的目录结构中，通常整个 chart 被打成 tar 包，而且标注上版本信息，便于 Helm 部署。

下面我们将详细讨论 chart 的目录结构以及包含的各类文件。
##### chart目录结构
以 MySQL chart 为例。一旦安装了某个 chart，我们就可以在 ~/.helm/cache/archive 中找到 chart 的 tar 包。
![2f137982db0551ebe03f8822bc4f6ead.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4052)
解压后,可以看到对应的目录结构
![956ece7b32947d6c0a26cc07537b7641.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4053)

目录名为chart的名字(不带版本信息),这里是mysql.
1. Chart.yaml
![acfd11520660d4fda9424801b9f9096e.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4054)
YAML文件,描述概要信息
name 和version是必填项,其他可选.
2.README.md
做开发的应该都比较熟悉这个了,使用文档,可选.
3.license
文本文件，描述 chart 的许可信息，此文件为可选。
4. requirements.yaml
chart可能依赖其他的chart,可以通过requestments指定.在安装过程中，依赖的 chart 也会被一起安装。
5.values.yaml
chart 支持在安装的时根据参数进行定制化配置，而 values.yaml 则提供了这些配置参数的默认值。
![56a916aa56478b9a231923412f73f235.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4055)
6.OWNER
用户权限
7.templates 目录 
各类 Kubernetes 资源的配置模板都放置在这里。Helm 会将 values.yaml 中的参数值注入到模板中生成标准的 YAML 配置文件。

模板是 chart 最重要的部分，也是 Helm 最强大的地方。模板增加了应用部署的灵活性，能够适用不同的环境.
>templates/NOTES.txt 
chart 的简易使用文档，chart 安装成功后会显示此文档内容。
与模板一样,可以在note.txt中插入配置参数,
![ff418599795873396127887f7b4fcc40.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4056)

##### chart template
Helm 通过模板创建 Kubernetes 能够理解的 YAML 格式的资源配置文件，

以 templates/secrets.yaml 为例：
![50242451f400564671920e1a9b6f0b87.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4060)

从结构看，文件的内容非常像 Secret 配置，只是大部分属性值变成了{{ xxx }}。这些 {{ xxx }} 实际上是模板的语法。Helm 采用了 Go 语言的模板来编写 chart。Go 模板非常强大，支持变量、对象、函数、流控制等功能。下面我们通过解析 templates/secrets.yaml 模板
```yaml
① \\{{ template "mysql.fullname" . }} 
```
定义 Secret 的 name。
关键字 template 的作用是引用一个子模板 mysql.fullname。这个子模板是在 templates/_helpers.tpl 文件中定义的。
![52d04effc1800157d7ff1a4fd85f0e8d.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4058)
这个定义还是很复杂的，因为它用到了模板语言中的对象、函数、流控制等概念。
不熟悉go的童鞋可能看不懂,不过暂时不是重点.

这里的重点是: 
<pre>如果存在一些信息多个模板都会用到，则可在 templates/_helpers.tpl 中将其定义为子模板，然后通过 templates 函数引用。</pre>

这里 mysql.fullname 是由 release 与 chart 二者名字拼接组成。

根据 chart 的最佳实践，所有资源的名称都应该保持一致，对于我们这个 chart，无论 Secret 还是 Deployment、PersistentVolumeClaim、Service，它们的名字都是子模板 mysql.fullname 的值。

② Chart 和 Release 是 Helm 预定义的对象，每个对象都有自己的属性，可以在模板中使用。如果使用下面命令安装 chart：
<pre>
helm install stable/mysql -n my
</pre>
那么：
```yaml
{{ .Chart.Name }} 的值为 mysql。
{{ .Chart.Version }} 的值为 0.3.0。
{{ .Release.Name }} 的值为 my。
{{ .Release.Service }} 始终取值为 Tiller。
{{ template "mysql.fullname" . }} 计算结果为 my-mysql。
```

③ 这里指定 mysql-root-password 的值，不过使用了 if-else 的流控制，其逻辑为：
如果 .Values.mysqlRootPassword 有值，则对其进行 base64 编码；否则随机生成一个 10 位的字符串并编码。

Values 也是预定义的对象，代表的是 values.yaml 文件。而 .Values.mysqlRootPassword 则是 values.yaml 中定义的 mysqlRootPassword 参数：
![04148404815565e81319e18949b244da.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4059)
因为 mysqlRootPassword 被注释掉了，没有赋值，所以逻辑判断会走 else，即随机生成密码。

randAlphaNum、b64enc、quote 都是 Go 模板语言支持的函数，函数之间可以通过管道 | 连接。{{ randAlphaNum 10 | b64enc | quote }} 的作用是首先随机产生一个长度为 10 的字符串，然后将其 base64 编码，最后两边加上双引号。

templates/secrets.yaml 这个例子展示了 chart 模板主要的功能，我们最大的收获应该是：模板将 chart 参数化了，通过 values.yaml 可以灵活定制应用。

无论多复杂的应用，用户都可以用 Go 模板语言编写出 chart。无非是使用到更多的函数、对象和流控制。对于初学者，我的建议是尽量参考官方的 chart。根据二八定律，这些 chart 已经覆盖了绝大部分情况，而且采用了最佳实践。如何遇到不懂的函数、对象和其他语法，可参考官网文档 https://docs.helm.sh


##### chart实践
1.安装前准备
需要先清楚 chart 的使用方法。这些信息通常记录在 values.yaml 和 README.md 中。除了下载源文件查看，执行 helm inspect values 可能也是一个更方便的方法。
![fd97048a65088b926d2bfc168cd46066.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4062)
输出的实际上是values.yaml的内容,阅读注释就可以知道MySQL chart支持哪些参数，安装之前需要做哪些准备。其中有一部分是关于存储的：
![2488d1b5770e90966d5376845b0daf64.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4063)
chart 定义了一个 PersistentVolumeClaim，申请 8G 的 PersistentVolume。由于我们的当前环境不支持动态供给，所以得预先创建好相应的 PV，其配置文件 mysql-pv.yml 内容为：
![e55510e86ff746c65d1dbb9364e8e585.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4064)
创建pv
![0fbd0bef299d081bd8ef308a596c7a2c.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4065)
2. 安装chart
除了接受 values.yaml 的默认值，我们还可以定制化 chart，比如设置 mysqlRootPassword。

Helm 有两种方式传递配置参数：

* 指定自己的 values 文件。
通常的做法是首先通过 helm inspect values mysql > myvalues.yaml生成 values 文件，然后设置 mysqlRootPassword，之后执行 helm install --values=myvalues.yaml mysql。
* 通过 --set 直接传入参数值，比如：

![be02ef15162d1136a4792863db038684.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4067)

mysqlRootPassword 设置为 a123456 另外，-n 设置 release 为 mydb，各类资源的名称即为mydb-mysql。
修改storage class后指定values文件

通过 helm list 和 helm status 可以查看 chart 的最新状态。
![9e3e9623c6094ca7439a128704719117.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4069)
PVC 已经 Bound，Deployment 也 AVAILABLE。


#### 升级和回滚应用.

release 发布后可以执行 helm upgrade 对其升级，通过 --values 或 --set应用新的配置。比如将当前的 MySQL 版本升级到 5.7.15：

![a4d283e39630cd44ab57f4dc1f49c4ed.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4070)
等待一段时间
![aea982971eeeca83cde26e0230ae0432.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4071)

helm history 可以查看 release 所有的版本。通过 helm rollback 可以回滚到任何版本。
![baa13b88cceb776dc24069fb19327232.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4072)
helm history 可以查看 release 所有的版本。
![9dbdfd83ee8f716e3e84ed924702e620.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4073)
过一会,回滚成功.恢复到到5.7.14
![db3bfdb878164523256822caacdbaaf6.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4074)


#### 开发chart
Kubernetes 给我们提供了大量官方 chart，不过要部署微服务应用，还是需要开发自己的 chart

##### 创建chart
执行 helm create mychart创建 chart mycrt:
![4a2cb61d3f6dc148500598f1dcec018b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4075)

Helm 会帮我们创建目录 mycrt，并生成了各类 chart 文件。这样我们就可以在此基础上开发自己的 chart 了。
新建的 chart 默认包含一个 nginx 应用示例，values.yaml 内容如下：
![b25d37e3313347789288e241dfa0a919.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4076)
开发时建议大家参考官方 chart 中的模板、values.yaml、Chart.yaml，里面包含了大量最佳实践和最常用的函数、流控制.
##### 调试 chart
只要是程序就会有 bug，chart 也不例外。Helm 提供了 debug 的工具：helm lint 和 helm install --dry-run --debug。

helm lint 会检测 chart 的语法，报告错误以及给出建议。

比如我们故意在 values.yaml 的第 8 行漏掉了一个 :，
![d9cbd5f57af90ee819821d287c0ad32c.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4077)
helm lint mychart 会指出这个语法错误。
![37d318ca483ad10d04879b0a4ea859e0.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4078)

mychart 目录被作为参数传递给 helm lint。错误修复后则能通过检测。
![3523f69ef283952feb980ecd926da354.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4079)

helm install --dry-run --debug 会模拟安装 chart，并输出每个模板生成的 YAML 内容。

可以检视这些输出，判断是否与预期相符。

同样，mychart 目录作为参数传递给 helm install --dry-run --debug。
![209df7caccbdaa9607f9f13758656e90.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4080)



#### 安装chart
准备就绪，就可以安装 chart，Helm 支持四种安装方法：

* 安装仓库中的 chart，例如：helm install stable/nginx
* 通过 tar 包安装，例如：helm install ./nginx-1.2.3.tgz
* 通过 chart 本地目录安装，例如：helm install ./nginx
* 通过 URL 安装，例如：helm install https://example.com/charts/nginx-1.2.3.tgz

这里我们使用本地目录安装：
![bc7765bd163ce936c0ee9d159052840b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4081)
当 chart 部署到 Kubernetes 集群，便可以对其进行更为全面的测试。

##### 将chart添加到从仓库
chart 通过测试后可以将其添加到仓库，团队其他成员就能够使用。任何 HTTP Server 都可以用作 chart 仓库，下面演示在 k8s01 上搭建仓库。

1.在k8s01上启动一个nginx容器:
```shell
docker run -d -p 8080:80 -v /var/www/:/usr/share/nginx/html/:ro nginx
```
2. 打包mychart
```shell
helm package mycrt
mkdir myrepo
mv mycrt-0.1.0.tgz myrepo/
```
3. 执行helm repo index生成仓库的index文件
![92c7fd41312b9915d3ccfbebced37892.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4082)
Helm 会扫描 myrepo 目录中的所有 tgz 包并生成 index.yaml。--url指定的是新仓库的访问路径。新生成的 index.yaml 记录了当前仓库中所有 chart 的信息：
![6a5218f1ce27a8f5bfc47435865a9bda.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4083)
当前只有 mycrt 这一个 chart。


4. 将 mycrt-0.1.0.tgz 和 index.yaml 上传到 k8s01 的 /var/www/charts 目录
 ![99d7f07c6f057a0691e0a8860cf196f4.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4087)
 

5. 通过heml repo add 将仓库添加到heml
![0f27ca4b3489832128a679e171c5f4ea.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4088)

仓库命名为 newrepo，Helm 会从仓库下载 index.yaml。

6. 现在可以repo search到mychart了.
![2e12e9189418399972147347c026fbf5.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4086)

7.从新仓库里安装mycrt
![83af9dbdc786548acc3ff9f8de131403.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4091)
8. 如果以后仓库添加了新的 chart，需要用 helm repo update 更新本地的 index。
```shell
helm repo update
```
![97f2f2a3654ba3deade36a9e6b3a2c3b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4092)


#### 小结
Helm 让我们能够像 yum 管理 rpm 包那样安装、部署、升级和删除容器化应用。

Helm 由客户端和 Tiller 服务器组成。客户端负责管理 chart，服务器负责管理 release。

chart 是 Helm 的应用打包格式，它由一组文件和目录构成。其中最重要的是模板，模板中定义了 Kubernetes 各类资源的配置信息，Helm 在部署时通过 values.yaml 实例化模板。

Helm 允许用户开发自己的 chart，并为用户提供了调试工具。用户可以搭建自己的 chart 仓库，在团队中共享 chart。

Helm 帮助用户在 Kubernetes 上高效地运行和管理微服务架构应用，Helm 非常重要。

