# Dashboard

[toc]
#### 配置
默认没有配置,运行下面的指令开启
```shell
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
```
Dashboard 会在 kube-system namespace 中创建自己的 Deployment 和 Service。
![3cbb31766b4213438e9d872519182be4.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4108)

因为 Service 是 ClusterIP 类型，为了方便使用，我们可通过 kubectl --namespace=kube-system edit service kubernetes-dashboard 修改成 NodePort 类型。
![6a23d0e365c8b61a89e9fc7e0a52ddea.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4109)


保存修改，此时已经为 Service 分配了端口31831
![dbfd70f56038d1f95395236053637048.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4110)
通过浏览器访问 Dashboard 
 https://10.0.0.81:31831
 ![87b1f2620fc22c5c6546704482e50422.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4111)
 
##### 登录权限配置


Dashboard 支持 Kubeconfig 和 Token 两种认证方式，为了简化配置，我们通过配置文件 dashboard-admin.yaml 为 Dashboard 默认用户赋予 admin 权限。

创建一个简单的用户,仅作为练习使用:
1. 创建service account
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

```
2. 集群权限设置
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```
3. Bearer Token
需要使用token登录,执行下面的命令行
```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

打印出token,类似于
![3455707752a5a8ade830862355f75a73.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4113)
找到admin-user,复制token

![e9d09de0c5c963ecfa548ac5e9bae9fb.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4115)
添加到令牌,点击登录
![1e5745f8c05f7517e5438f2661c57edc.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4116)


#### 使用场景

1. 部署deployment
用户可以直接输入要部署应用的名字、镜像、副本数等信息；也可以上传 YAML 配置文件。如果是上传配置文件，则可以创建任意类型的资源，不仅仅是 Deployment。

当然也可以文本框输入yaml或者json文件
![89ff3ed04d4c617607961784be35fd2f.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4117)


2. 在线操作pod
![be068534cf4abc046bb62c077e05d4df.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4118)
一为日志 2为yaml删除,编辑等操作
* 编辑yaml
比如点击 View/edit YAML 可直接修改资源的配置，保存后立即生效，其效果与 kubectl edit 一样。
![9a9ffd17c505deb6f6804fa917333c13.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4119)

* 查看资源详细信息
 ![d0143855ece5623423d3f3fb7613c7c3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4120)

* 查看pod日志
在 Pod 及其父资源（DaemonSet、ReplicaSet 等）页面

点击 ![0162bd25df96909ef160bad02327a69b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4122)

按钮，可以查看 Pod 的日志，其效果与 kubectl logs 一样。
![50edc895ef2ad58fe431329e4647a90d.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4121)
Kubernetes Dashboard 界面设计友好，自解释性强，可以看作 GUI 版的 kubectl，更多功能留给大家自己探索。

Kubernetes Dashboard 的安装和使用方法。Dashboard能完成日常管理的大部分工作，可以作为命令行工具 kubectl 的有益补充。