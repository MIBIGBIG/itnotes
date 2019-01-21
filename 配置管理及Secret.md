# 配置管理及Secret处理

应用启动过程中可能需要一些敏感信息，比如访问数据库的用户名密码或者秘钥。将这些信息直接保存在容器镜像中显然不妥，Kubernetes 提供的解决方案是 Secret。

Secret 会以密文的方式存储数据，避免了直接在配置文件中保存敏感信息。Secret 会以 Volume 的形式被 mount 到 Pod，容器可通过文件的方式使用 Secret 中的敏感数据；此外，容器也可以环境变量的方式使用这些数据。

Secret 可通过命令行或 YAML 创建。比如希望 Secret 中包含如下信息：
用户名 admin
密码 admin123

#### 创建Secret的四种方法
1. 通过 --from-literal 信息条目：
```shell
kubectl create secret generic mysecret --from-literal=username=admin --from-literal=password=admin123
```
每个 --from-literal 对应一个信息条目。

2. 通过 --from-file 文件的方式：
```shell
echo -n admin > ./username
echo -n admin123 > ./password
kubectl create secret generic mysecret --from-file=./username --from-file=./password
```
每个文件内容对应一个信息条目。

3. 通过 --from-env-file：

cat << EOF > env.txt
username=admin
password=admin123
EOF
kubectl create secret generic mysecret --from-env-file=env.txt
文件 env.txt 中每行 Key=Value 对应一个信息条目。

4. 通过 YAML 配置文件：
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
data:
  username: YWRtaW4= 
  password: YWRtaW4xMjM=
```
文件中的敏感数据必须是通过 base64 编码后的结果。

![e15a68a43aa27ef91ebef531c5cff353.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4026)
执行kubectl apply 创建secret
![22b8ca36292d24808d9b129113613ef3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4027)

#### 如何查看secret

可以通过Kubectl get secret 查看存在的secret
通过kubectl describe secret [name]查看key
![05630360de1952b0a1465ca97486039f.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4028)

如果还想查看 Value，可以用 kubectl edit secret mysecret：

![724a61f64c89b49e04b0cf5e56896cae.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4029)
然后通过base64将value反编码过来

![9c5382bf2bd6bb4ed9072887ead257a3.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4030)

#### 挂载文件或者环境变量的方式

##### Volume
Pod 可以通过 Volume 或者环境变量的方式使用 Secret,这里是Volume的方式

Pod 的配置文件如下所示：
```yaml
apiVersion:
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: busybox
    args:
    - /bin/sh
    - -c
    - sleep 10; touch /tmp/healthy; sleep 30000
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    sectet:
      secretName: mysecret
```
① 定义 volume foo，来源为 secret mysecret。

② 将 foo mount 到容器路径 /etc/foo，可指定读写权限为 readOnly。



apply -f mypod.yml后 读取secret
![5152688eb531907be75ea6042d7c1b6e.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4031)

可以看到，Kubernetes 会在指定的路径 /etc/foo 下为每条敏感数据创建一个文件，文件名就是数据条目的 Key，这里是 /etc/foo/username 和 /etc/foo/password，Value 则以明文存放在文件中。

我们也可以自定义存放数据的文件名，比如将配置文件改为：

![88affc2b7154692fb0f4f9b1a14db8a2.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4032)
这时数据将分别存放在 /etc/foo/my-group/my-username 和 /etc/foo/my-group/my-password 中。

以 Volume 方式使用的 Secret 支持动态更新：Secret 更新后，容器中的数据也会更新。

将 password 更新为 12345，base64 编码为 MTIzNDU=
![9fe48cd22e22db103c4eaef24f144bf0.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4034)
更新Secret,几秒后,同步到容器中
![254edc5120451e600c9f8709d18436c6.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4033)

##### 环境变量的方式
通过 Volume 使用 Secret，容器必须从文件读取数据，会稍显麻烦，Kubernetes 还支持通过环境变量使用 Secret。

Pod 配置文件示例:
![4761df2191c8e3e0e1cbd1e35f34ccc1.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4036)

创建pod ,并读取Key
![c352e15966876727c5b9b604cfe89f0a.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4035)

通过环境变量 SECRET_USERNAME 和 SECRET_PASSWORD 成功读取到 Secret 的数据。

需要注意的是，环境变量读取 Secret 很方便，但无法支撑 Secret 动态更新。

Secret 可以为 Pod 提供密码、Token、私钥等敏感数据；对于一些非敏感数据，比如应用的配置信息，则可以用 ConfigMap。

#### ConfigMap管理配置
Secret 可以为 Pod 提供密码、Token、私钥等敏感数据；对于一些非敏感数据，比如应用的配置信息，则可以用 ConfigMap。

ConfigMap 的创建和使用方式与 Secret 非常类似，主要的不同是数据以明文的形式存放。

与 Secret 一样，ConfigMap 也支持四种创建方式：

1. 通过 --from-literal：
```shell
kubectl create configmap myconfigmap --from-literal=config1=xxx --from-literal=config2=yyy
```
每个 --from-literal 对应一个信息条目。

2. 通过 --from-file：
```shell
echo -n xxx > ./config1
echo -n yyy > ./config2
kubectl create configmap myconfigmap --from-file=./config1 --from-file=./config2
```
每个文件内容对应一个信息条目。

3. 通过 --from-env-file：
```shell
cat << EOF > env.txt
config1=xxx
config2=yyy
EOF
kubectl create configmap myconfigmap --from-env-file=env.txt
```
文件 env.txt 中每行 Key=Value 对应一个信息条目。

4. 通过 YAML 配置文件：
```yaml
apiVersin: v1
kind: ConfigMap
metadata:
  name: myconfigmap
data:
  config1: xxx
  config2: yyy
```
文件中的数据直接以铭文输入.
与 Secret 一样，Pod 也可以通过 Volume 或者环境变量的方式使用 Secret。

Volume 方式：
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: mypod
spec:
  containers:
  - name: mypod
    image: busybox
    args:
      - /bin/sh
      - -c 
      - sleep 10; touch /tmp/healthy; sleep 30000
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    configMap:
      name: myconfigmap
```

环境变量方式：
![1011e7c12ddbdff1cbc4b6d1595825ec.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4037)
大多数情况下，配置信息都以文件形式提供，所以在创建 ConfigMap 时通常采用 --from-file 或 YAML 方式，读取 ConfigMap 时通常采用 Volume 方式。
比如给 Pod 传递如何记录日志的配置信息：
![49934efa977177bd9881b6d40d931cb9.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4038)
可以采用 --from-file 形式，则将其保存在文件 logging.conf 中，然后执行命令：
```shell
kubectl create configmap myconfigmap --from-file=./logging.conf
```
如果采用 YAML 配置文件，其内容则为：
![3a2352684b26e366d7dd12014fbd6558.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4039)
注意别漏写了 Key logging.conf 后面的 | 符号。

创建并查看 ConfigMap：
![f6e7d9332dfa57b58c0a062e2a56d018.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4040)
在pod中使用ConfigMap.配置文件为:
![3b5b08ec12e1cb3dfe76335216758cee.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4043)

① 在 volume 中指定存放配置信息的文件相对路径为 myapp/logging.conf。
② 将 volume mount 到容器的 /etc/foo 目录。
创建 Pod 并读取配置信息：
![a1d3943fb6ca3ce716fcc97eb0b83d66.png](evernotecid://8FFE4719-72F7-4332-B58A-CDD367D554D8/appyinxiangcom/18527455/ENResource/p4042)
配置信息已经保存到 /etc/myapp/logging.conf 文件中。与 Secret 一样，Volume 形式的 ConfigMap 也支持动态更新.

#### 小结

如何向 Pod 传递配置信息。如果信息需要加密，可使用 Secret；如果是一般的配置信息，则可使用 ConfigMap。
Secret 和 ConfigMap 支持四种定义方法。Pod 在使用它们时，可以选择 Volume 方式或环境变量方式，不过只有 Volume挂载文件的 方式支持动态更新。