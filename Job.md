# Job管理

容器按照持续运行的时间可分为两类：服务类容器和工作类容器。

服务类容器通常持续提供服务，需要一直运行，比如 http server，daemon 等。工作类容器则是一次性任务，比如批处理程序，完成后容器就退出。

Kubernetes 的 Deployment、ReplicaSet 和 DaemonSet 都用于管理服务类容器；对于工作类容器，我们用 Job。

先看一个简单的 Job 配置文件 myjob.yml：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  template:
    metadata:
      name: myjob
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo","hello k8s!"]
      restartPolicy: Never

```
① batch/v1 是当前 Job 的 apiVersion。
② 指明当前资源的类型为 Job。
③ restartPolicy 指定什么情况下需要重启容器。对于 Job，只能设置为 Never 或者 OnFailure。对于其他 controller（比如 Deployment）可以设置为 Always 。


通过 kubectl apply -f myjob.yml 启动 Job。
kubectl get job 查看 Job 的状态：
![57dda8bc56119926f3aad0001706564a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3890)
稍等一会,再次检查
completions为1/1,表示已经按照预期启动了一个Pod,并且已经成功执行.

![2a36ab0fa5cf38cba401fe62ebe9c46c.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3891)
kubectl get pod 查看 Pod 的状态：
![6fda08ba7c5b9343905dcc1ba75e30d8.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3892)

因为 Pod 执行完毕后容器已经退出，需要用 --show-all 才能查看 Completed 状态的 Pod。

kubectl logs 可以查看 Pod 的标准输出：

![d9cbec0d342f8695cdb08a54697b684b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3893)


以上是 Pod 成功执行的情况，如果 Pod 失败了会怎么样呢？

修改myjob.yml,故意引入一个错误:
![51116bef6a74599142a3ffc4cca9f214.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3894)


先删除之前的 Job,修改后重新启动
![56167bae9ed18c5b3516951acf203c72.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3895)
运行新的 Job 并查看状态
![3dfc2e3ae39d2dd4f52d40694ee5565a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3896)
可以看到有多个pod,状态均为不正常,kubectl describe pod查看某个pod的启动日志
![d5e9c9961dc7eed73078434795b23ac8.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3898)
日志显示没有可执行程序，符合我们的预期。

下面解释一个现象：为什么 kubectl get pod 会看到这么多个失败的 Pod？

原因是：当第一个 Pod 启动时，容器失败退出，根据 restartPolicy: Never，此失败容器不会被重启，但 Job DESIRED 的 Pod 是 1，目前 SUCCESSFUL 为 0，不满足，所以 Job controller 会启动新的 Pod，直到 SUCCESSFUL 为 1。对于我们这个例子，SUCCESSFUL 永远也到不了 1，所以 Job controller 会一直创建新的 Pod。为了终止这个行为，只能删除 Job。
![fe84b9b2b44d8fcf896a136adadc13de.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3899)

如果将 restartPolicy 设置为 OnFailure 会怎么样？下面我们实践一下，修改 myjob.yml 后重新启动。
Job 的 completion Pod 数量还是为 0，看看 Pod 的情况：
![9700d16b964d182fc72763b1bdd524e3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3901)
这里只有一个 Pod，不过 RESTARTS 为 2，而且不断增加，说明 OnFailure 生效，容器失败后会自动重启。

#### 并行执行job
有时，我们希望能同时运行多个 Pod，提高 Job 的执行效率。这个可以通过 parallelism 设置。
kubectl delete -f myjob.yml

![a35696e088050a2ce0216cede1103430.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3902)
修改后重新执行
![9fc927112f670b96f1d254992524b5f6.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3903)
Job 一共启动了两个 Pod，而且 AGE 相同，可见是并行运行的。

我们还可以通过 completions 设置 Job 成功完成 Pod 的总数：
![eaec3568fd980118b425a13567f1dada.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3905)

配置的含义是：每次运行两个 Pod，直到总共有 6 个 Pod 成功完成。实践一下：
![84fc022579f77688c7ce359a7b82805a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3904)

completion 为 6，符合预期。如果不指定 completions 和 parallelism，默认值均为 1。

上面的例子只是为了演示 Job 的并行特性，实际用途不大。不过现实中确实存在很多需要并行处理的场景。比如批处理程序，每个副本（Pod）都会从任务池中读取任务并执行，副本越多，执行时间就越短，效率就越高。这种类似的场景都可以用 Job 来实现。

#### 定时执行job
Linux 中有 cron 程序定时执行任务，Kubernetes 的 CronJob 提供了类似的功能，可以定时执行 Job。CronJob 配置文件示例如下：

```yaml
apiVersion: batch/v2alpha1
kind: CronJob
medadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["echo", "Hello k8S!"]
          restartPolicy: OnFailure
```

① batch/v2alpha1 是当前 CronJob 的 apiVersion。

② 指明当前资源的类型为 CronJob。

③ schedule 指定什么时候运行 Job，其格式与 Linux cron 一致。这里 */1 * * * * 的含义是每一分钟启动一次。

④ jobTemplate 定义 Job 的模板，格式与前面 Job 一致。

接下来通过 kubectl apply 创建 CronJob。

```shell
[root@k8s03 k8s]# kubectl apply -f cronjob.yml
error: unable to recognize "cronjob.yml": no matches for kind "CronJob" in version "batch/v2alpha1"
```
失败了。这是因为 Kubernetes 默认没有 enable CronJob 功能，需要在 kube-apiserver 中加入这个功能。方法很简单，修改 kube-apiserver 的配置文件 /etc/kubernetes/manifests/kube-apiserver.yaml：

kube-apiserver 本身也是个 Pod，在启动参数中加上 --runtime-config=batch/v2alpha1=true 即可。
![0caafa61d846784cfe597dcada6a8d68.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3906)

然后重启 kubelet 服务：

systemctl restart kubelet.service
kubelet 会重启 kube-apiserver Pod。通过 kubectl api-versions 确认 kube-apiserver 现在已经支持 batch/v2alpha1：
![a661a3c65fbe37dd6355e7468afc4ae2.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3907)
再次创建CronJob：
```shell
[root@k8s03 k8s]# kubectl apply -f cronjob.yml
cronjob.batch/hello created
```
这次成功了。通过 kubectl get cronjob 查看 CronJob 的状态：

```shell
[root@k8s03 k8s]# kubectl get cronjob
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        41s             5m54s
```
等待几分钟，然后通过 kubectl get jobs 查看 Job 的执行情况：
![68b21056854ba602c727211f24c307f9.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3909)
可以看到每隔一分钟就会启动一个 Job。执行 kubectl logs 可查看某个 Job 的运行日志：
```shell
[root@k8s03 k8s]# kubectl logs hello-1546582860-kzslr
Hello k8S!
```

运行容器化应用是 Kubernetes 最重要的核心功能。为满足不同的业务需要，Kubernetes 提供了多种 Controller，包括 Deployment、DaemonSet、Job、CronJob 等。