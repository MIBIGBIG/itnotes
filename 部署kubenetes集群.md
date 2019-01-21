# 部署kubenetes集群

[toc]
### K8S部署
![1ed9fb4bcb94a5d1bef4eb077fea26f5.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3837)

#### 部署准备
系统:CentOS 7.5
内存:2 GB or more of RAM per machine (any less will leave little room for your apps)
CPU: 2 CPUs or more
网络: Full network connectivity between all machines in the cluster (public or private network is fine)
环境要求: 主机名唯一 MAC地址唯一 UUID唯一
SWAP: 禁用交换分区

k8s03: 10.0.0.83
k8s01: 10.0.0.81
k8s04: 10.0.0.84

官方安装文档可以参考 https://kubernetes.io/docs/setup/independent/install-kubeadm/
##### 禁用交换分区
在实际禁用交换空间之前，首先需要查看内存负载程度，然后通过发出以下命令来确定保存交换区域的分区。
```shell
#free -h 
检查交换空间

接下来，发出以下blkid命令 ，查找TYPE=”swap”行以确定交换分区，如下图所示。

#blkid
```
在识别交换分区或文件后，执行以下命令来禁用交换区域。

```shell
[root@docker03 ~]# swapoff /dev/mapper/centos_model-swap

[root@docker03 ~]# swapoff -a
##验证
[root@docker03 ~]# free -h
              total        used        free      shared  buff/cache   available
Mem:           1.8G        216M        1.0G        9.5M        539M        1.4G
Swap:            0B          0B          0B
```
禁用交换分区

为了在Linux中永久禁用交换空间，打开/ etc / fstab文件，搜索交换行并在行的前面添加一个# （hashtag）标记来注释整行，如下图所示。

![6685f5343b9f013babd5f53eb92a2b3b.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3827)
永久禁用交换分区

之后， 重新启动系统以应用新的交换设置，或者在某些情况下发出mount -a命令可能会失效。
```shell
# mount -a
```
系统重启后,再次确认
```shell
# free -h
# blkid 
# lsblk 
```
##### 关闭防火墙,selinux
```shell
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/sysconfig/selinux
```

##### sysctl配置
确保sysctl中net.bridge.bridge-nf-call-iptables的值为1：
```shell
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```


#### 安装docker
所有节点都需要安装 Docker。
```shell
wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
sed -i 's#download.docker.com#mirrors.ustc.edu.cn/docker-ce#g' /etc/yum.repos.d/docker-ce.repo
yum install docker-ce -y
```

#### 安装kubelet、kubeadm 和 kubectl
安装kubernetes的时候，需要安装kubelet, kubeadm等包，但k8s官网给的yum源是packages.cloud.google.com，国内访问不了，此时我们可以使用阿里云的yum仓库镜像。

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

下载K8S组件到所有节点..
```shell

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable kubelet && systemctl start kubelet
```
kubelet 运行在 Cluster 所有节点上，负责启动 Pod 和容器。

kubeadm 用于初始化 Cluster。

kubectl 是 Kubernetes 命令行工具。通过 kubectl 可以部署和管理应用，查看各种资源，创建、删除和更新各种组件。

确保iptables设置
```shell
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

##### 注意
在测试时发现老版本docker驱动不是cgroupfs，导致kubernetes报错。查看驱动版本：
```shell
docker info | grep -i cgroup
```
如果不是需要修改kubernetes配置：
```shell
vi /etc/default/kubelet
KUBELET_EXTRA_ARGS=--cgroup-driver=改为当前驱动
systemctl daemon-reload
systemctl restart kubelet
```
![96fb7cf84ed33601472c7b62a1ff8b34.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3839)

##### 用 kubeadm 创建 Cluster
完整的官方文档可以参考 https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

##### master下载所需镜像
下载服务所需镜像

可以跳过，初始化会自动下载，但是因为有墙，所以需要尽量提前准备好或者设置科学↑net。强烈建议可以首先设置docker代理.
```shell
mkdir /etc/systemd/system/docker.service.d
cat <<EOF > /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:1080"
EOF
systemctl daemon-reload
systemctl restart docker
```
下载镜像

kubeadm config images pull --kubernetes-version v1.13.1
得到镜像
###### 初始化master

此处可能会有些坑,要先准备好http-proxy,不然初始化时docker pull gcr.io和pian.io镜像时会被.墙...
此处强烈👎下墙...按照官网教程来的伤不起啊(被折腾好久)

在 Master 上执行如下命令：
```shell
kubeadm init --apiserver-advertise-address 10.0.0.83 --kubernetes-version=v1.13.1 --pod-network-cidr=10.244.0.0/16
```
--apiserver-advertise-address：api-server服务监听地址，即master的ip

--kubernetes-version：指定要部署的kubernetes版本，此处为1.13.1

--pod-network-cidr：指定pod网络分配地址范围。需要与下一步要使用的插件的配置一致，此处使用Flannel
```shell
[init] Using Kubernetes version: v1.13.1
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 18.09.0. Latest validated version: 18.06
	[WARNING Hostname]: hostname "k8s" could not be reached
	[WARNING Hostname]: hostname "kube-master": lookup kube-master on 192.168.2.1:53: no such host
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kube-master localhost] and IPs [10.0.0.83 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kube-master localhost] and IPs [10.0.0.83 127.0.0.1 ::1]
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kube-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.83]
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 23.502901 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "kube-master" as an annotation
[mark-control-plane] Marking the node kube-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node kube-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: thmy85.7ahn8zezyt6m39yy
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.0.0.83:6443 --token 57p8re.bwy6gsi69cephqrs --discovery-token-ca-cert-hash sha256:7fd3b0d1c70529c1661f50383f184aab80c60f9da4719767321ab4b04418ef48
```

#### 创建配置文件

上一步输出中提示创建配置文件，否则执行kubectl会报错。

按照提示执行

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
再次执行
```shell
[root@k8s03 ~]# kubectl get nodes
NAME    STATUS     ROLES    AGE     VERSION
k8s01   NotReady   <none>   101m    v1.13.1
k8s02   NotReady   <none>   2m34s   v1.13.1
k8s03   NotReady   master   123m    v1.13.1
```

安装Pod网络插件,使pod可以相互通信
```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

#### 节点加入

切换到node服务器,执行初始化master时最后打印的命令
```shell
kubeadm join 10.0.0.83:6443 --token 57p8re.bwy6gsi69cephqrs --discovery-token-ca-cert-hash sha256:7fd3b0d1c70529c1661f50383f184aab80c60f9da4719767321ab4b04418ef48
```
如果当时没有记录下来可以通过master的 kubeadm token list 查看。
token的值
```shell
kubeadm token list
```
--discovery-token-ca-cert-hash 的值
```shell
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

根据提示,我们可以通过kubectl get nodes 查看节点的状态

![b07f3d2cd03483287e644ac6299f7208.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3828)
目前所有节点都是 NotReady，这是因为每个节点都需要启动若干组件，这些组件都是在 Pod 中运行，需要首先从 google 下载镜像，我们可以通过如下命令查看 Pod 的状态：
```shell
kubectl get pod --all-namespaces
```
![761fc39cbb578e14160dc1784c91ba07.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3829)

Pending、ContainerCreating、ImagePullBackOff 都表明 Pod 没有就绪，Running 才是就绪状态。我们可以通过 kubectl describe pod <Pod Name> 查看 Pod 具体情况，比如：
```shell
kubectl describe pod kube-flannel-ds-amd64-rt9s4 --namespace=kube-system
```
![495cab8e5fa8761c418cbe28edceb585.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3830)

为了节省篇幅，这里只截取命令输出的最后部分，可以看到在下载 image 时失败，如果网络质量不好，这种情况是很常见的。我们可以耐心等待，因为 Kubernetes 会重试，我们也可以自己手工执行 docker pull 去下载这个镜像。

等待一段时间，image 都成功下载后，所有 Pod 会处于 Running 状态。

![999482fc8464cc4bb7e381696bba5b85.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p3836)
##### debug NotReady
首先 describe nodes 
```shell
$ kubectl describe nodes
```
这里如果没有问题,SSH 进问题node查看kubelet logs输出
```shell
journalctl -u kubelet
```
这时，所有的节点都已经 Ready，Kubernetes Cluster 创建成功，一切准备就绪。