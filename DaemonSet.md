# DaemonSet管理

#### 典型应用场景
Deployment 部署的副本 Pod 会分布在各个 Node 上，每个 Node 都可能运行好几个副本。DaemonSet 的不同之处在于：每个 Node 上最多只能运行一个副本。

DaemonSet 的典型应用场景有：

* 在集群的每个节点上运行存储 Daemon，比如 glusterd 或 ceph。
* 在每个节点上运行日志收集 Daemon，比如 flunentd 或 logstash。
* 在每个节点上运行监控 Daemon，比如 Prometheus Node Exporter 或 collectd。

其实 Kubernetes 自己就在用 DaemonSet 运行系统组件。执行如下命令：
```shell
kubectl get daemonset --namespace=kube-system
```

![bd570425f06c1b9a96cb25d000536b3a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3886)
DaemonSet kube-flannel-ds 和 kube-proxy 分别负责在每个节点上运行 flannel 和 kube-proxy 组件。
![a0da1f86bb35bed7e7cf2bbdc0b60ce3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3887)

因为 flannel 和 kube-proxy 属于系统组件，需要在命令行中通过 --namespace=kube-system 指定 namespace kube-system。如果不指定则只返回默认 namespace default 中的资源。



#### DaemonSet组件分析
详细分析两个 k8s 自己的 DaemonSet：kube-flannel-ds 和 kube-proxy 。
##### kube-flannel-ds

初始化时如何部署flannel网络的?
```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
flannel 的 DaemonSet 就定义在 kube-flannel.yml 中：
![2518d24d39987d1873540dfb0bf757a8.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3888)

注：配置文件的完整内容要复杂些，介于篇幅介绍 DaemonSet，这里只保留了重要的内容。

① DaemonSet 配置文件的语法和结构与 Deployment 几乎完全一样，只是将 kind 设为 DaemonSet。
② hostNetwork 指定 Pod 直接使用的是 Node 的网络，相当于 docker run --network=host。考虑到 flannel 需要为集群提供网络连接，这个要求是合理的。
③ containers 定义了运行 flannel 服务的两个容器。


##### kube-proxy
无法拿到 kube-proxy 的 YAML 文件，可以运行如下命令查看其配置：

kubectl edit daemonset kube-proxy --namespace=kube-system

![31d81531904e6d25f9f8d00ba1ad4fcd.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3889)

同样只保留了重要的信息。

① kind: DaemonSet 指定这是一个 DaemonSet 类型的资源。

② containers 定义了 kube-proxy 的容器。

③ status 是当前 DaemonSet 的运行时状态，这个部分是 kubectl edit特有的。其实 Kubernetes 集群中每个当前运行的资源都可以通过 kubectl edit 查看其配置和运行状态，比如 kubectl edit deployment nginx-pod。

#### 运行自己的DaemonSet
以 Prometheus Node Exporter 为例演示如何运行自己的 DaemonSet。

如果是直接在 Docker 中运行 Node Exporter 容器，命令为：

```shell
docker run -d \
  -v "/proc:/host/proc" \
  -v "/sys:/host/sys" \
  -v "/:/rootfs" \
  --net=host \  
  prom/node-exporter \
  --path.procfs /host/proc \
  --path.sysfs /host/sys \
  --collector.filesystem.ignored-mount-points "^/(sys|proc|dev|host|etc)($|/)"
```
将其转换为 DaemonSet 的 YAML 配置文件 node_exporter.yml：
```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: node-exporter
  labels:
    name: node-exporter
spec:
  template:
    metadata:
      labels:
        name: node-exporter
      annotations:
         prometheus.io/scrape: "true"
         prometheus.io/port: "9100"
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
        - name: node-exporter
          image: prom/node-exporter@sha256:ffc3e0adf58771cb2097aa0b074a6fe68b4925ba75c2e1c41f41ae656eebee11
          securityContext:
            privileged: true
          args:
            - --collector.procfs
            - /host/proc
            - --collector.sysfs
            - /host/sys
            - --collector.filesystem.ignored-mount-points
            - '"^/(sys|proc|dev|host|etc)($|/)"'
          ports:
            - containerPort: 9100
              protocol: TCP
          resources:
            limits:
              cpu: 100m
              memory: 100Mi
            requests:
              cpu: 10m
              memory: 100Mi
          volumeMounts:
            - name: dev
              mountPath: /host/dev
            - name: proc
              mountPath: /host/proc
            - name: sys
              mountPath: /host/sys
            - name: rootfs
              mountPath: /rootfs
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
```
① 直接使用 Host 的网络。
② 设置容器启动命令。
③ 通过 Volume 将 Host 路径 /proc、/sys 和 /sys 映射到容器中。我们将在后面详细讨论 Volume。





