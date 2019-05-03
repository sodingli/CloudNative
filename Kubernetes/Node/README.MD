
## 前提
Kubernetes node节点包含如下组件：

* Flanneld：需要先安装。
* Docker ：需要提前安装。
* kubelet：直接用二进制文件安装
* kube-proxy：直接用二进制文件安装


注意：每台 node 上都需要安装 flannel，master 节点上可以不安装

我们再检查一下node节点上，经过前几步操作我们已经创建了如下的证书和配置文件。

``` bash
$ ls /etc/kubernetes/ssl
admin-key.pem  admin.pem  ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  kubernetes-key.pem  kubernetes.pem
$ ls /etc/kubernetes/
apiserver  bootstrap.kubeconfig  config  controller-manager  kubelet  kube-proxy.kubeconfig  proxy  scheduler  ssl  token.csv
```







这几个的关系是，部署完docker --启动flannel
重新改docker 配置启动flanneld， 并启动 kubelet


前面章节已经完成了 docker 和 flanneld 部署。

本章主要是部署  kubelet



## 部署  kubelet

### kubelet 二进制文件
下载最新的kubelet和kube-proxy二进制文件
注意请下载对应的Kubernetes版本的安装包。

```shell

wget https://dl.k8s.io/v1.6.0/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes
tar -xzvf  kubernetes-src.tar.gz
cp -r ./server/bin/{kube-proxy,kubelet} /usr/local/bin/
```



### 创建配置文件

文件 /usr/lib/systemd/system/kubelet.service

```shell
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
#EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/local/bin/kubelet \
            $KUBE_LOGTOSTddDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_ALLOW_PRIV \
            $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target




```

配置 环境变量 /etc/kubernetes/kubelet

```
###
## kubernetes kubelet (minion) config
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=172.16.97.244"
#
## The port for the info server to serve on
#KUBELET_PORT="--port=10250"
#
## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=newcaust"
#
## location of the api-server
## COMMENT THIS ON KUBERNETES 1.8+
#KUBELET_API_SERVER="--api-servers=http://172.20.0.113:8080"
#
## pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=jimmysong/pause-amd64:3.0"
#
## Add your own!
KUBELET_ARGS="--cgroup-driver=systemd --cluster-dns=202.106.0.20 --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig  --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"
```


+ -experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig  是从主节点的 ~/.kube/config 拷贝过来的
+ --cgroup-driver=systemd  跟docker info 里匹配
+ KUBELET_POD_INFRA_CONTAINER 是基础镜像容器，其实该镜像并不运行什么具体进程，可以使用Google的pause镜像gcr.io/google_containers/pause-amd64:3.0，这个镜像只有300多K，或者通过DockerHub下载jimmysong/pause-amd64:3.0。


## 安装和配置kubelet

**kubernets1.8**

相对于kubernetes1.6集群必须进行的配置有：

对于kuberentes1.8集群，必须关闭swap，否则kubelet启动将失败。

修改`/etc/fstab`将，swap系统注释掉。

---

kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper cluster 角色(role)，
然后 kubelet 才能有权限创建认证请求(certificate signing requests)：

``` bash
cd /etc/kubernetes
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

+ `--user=kubelet-bootstrap` 是在 `/etc/kubernetes/token.csv` 文件中指定的用户名，同时也写入了 `/etc/kubernetes/bootstrap.kubeconfig` 文件；

---

kubelet 通过认证后向 kube-apiserver 发送 register node 请求，需要先将 `kubelet-nodes` 用户赋予 `system:node` cluster角色(role) 和 `system:nodes` 组(group)，
然后 kubelet 才能有权限创建节点请求：

``` bash
kubectl create clusterrolebinding kubelet-nodes \
  --clusterrole=system:node \
  --group=system:nodes
```









启动 
```shell

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```


### 通过kublet的TLS证书请求

kubelet 首次启动时向 kube-apiserver 发送证书签名请求，必须通过后 kubernetes 系统才会将该 Node 加入到集群。

查看未授权的 CSR 请求

``` bash
$ kubectl get csr
NAME        AGE       REQUESTOR           CONDITION
csr-2b308   4m        kubelet-bootstrap   Pending
$ kubectl get nodes
No resources found.
```

在master上通过 CSR 请求就可以了。

``` bash
$ kubectl certificate approve csr-2b308
certificatesigningrequest "csr-2b308" approved
$ kubectl get nodes
NAME        STATUS    AGE       VERSION
10.64.3.7   Ready     49m       v1.6.1
```

自动生成了 kubelet kubeconfig 文件和公私钥

``` bash
$ ls -l /etc/kubernetes/kubelet.kubeconfig
-rw------- 1 root root 2284 Apr  7 02:07 /etc/kubernetes/kubelet.kubeconfig
$ ls -l /etc/kubernetes/ssl/kubelet*
-rw-r--r-- 1 root root 1046 Apr  7 02:07 /etc/kubernetes/ssl/kubelet-client.crt
-rw------- 1 root root  227 Apr  7 02:04 /etc/kubernetes/ssl/kubelet-client.key
-rw-r--r-- 1 root root 1103 Apr  7 02:07 /etc/kubernetes/ssl/kubelet.crt
-rw------- 1 root root 1675 Apr  7 02:07 /etc/kubernetes/ssl/kubelet.key
```

假如你更新kubernetes的证书，只要没有更新`token.csv`，当重启kubelet后，该node就会自动加入到kuberentes集群中，而不会重新发送`certificaterequest`，也不需要在master节点上执行`kubectl certificate approve`操作。前提是不要删除node节点上的`/etc/kubernetes/ssl/kubelet*`和`/etc/kubernetes/kubelet.kubeconfig`文件。否则kubelet启动时会提示找不到证书而失败。

**注意：**如果启动kubelet的时候见到证书相关的报错，有个trick可以解决这个问题，可以将master节点上的`~/.kube/config`文件（该文件在[安装kubectl命令行工具](kubectl-installation.md)这一步中将会自动生成）拷贝到node节点的`/etc/kubernetes/kubelet.kubeconfig`位置，这样就不需要通过CSR，当kubelet启动后就会自动加入的集群中。



## 配置 kube-proxy

**安装conntrack**

```bash
yum install -y conntrack-tools
```

**创建 kube-proxy 的service配置文件**

文件路径`/usr/lib/systemd/system/kube-proxy.service`。

```ini
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/kube-proxy \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_MASTER \
	    $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

kube-proxy配置文件`/etc/kubernetes/proxy`。

``` bash
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--bind-address=172.20.0.113 --hostname-override=172.20.0.113 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"
```

+ `--hostname-override` 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则；
+ kube-proxy 根据 `--cluster-cidr` 判断集群内部和外部流量，指定 `--cluster-cidr` 或 `--masquerade-all` 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；
+ `--kubeconfig` 指定的配置文件嵌入了 kube-apiserver 的地址、用户名、证书、秘钥等请求和认证信息；
+ 预定义的 RoleBinding `cluster-admin` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限；

完整 unit 见 [kube-proxy.service](../systemd/kube-proxy.service)

### 启动 kube-proxy

``` bash
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```## 配置 kube-proxy

**安装conntrack**

```bash
yum install -y conntrack-tools
```

**创建 kube-proxy 的service配置文件**

文件路径`/usr/lib/systemd/system/kube-proxy.service`。

```ini
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/kube-proxy \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_MASTER \
	    $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

kube-proxy配置文件`/etc/kubernetes/proxy`。

``` bash
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--bind-address=172.20.0.113 --hostname-override=172.20.0.113 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"
```

+ `--hostname-override` 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则；
+ kube-proxy 根据 `--cluster-cidr` 判断集群内部和外部流量，指定 `--cluster-cidr` 或 `--masquerade-all` 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；
+ `--kubeconfig` 指定的配置文件嵌入了 kube-apiserver 的地址、用户名、证书、秘钥等请求和认证信息；
+ 预定义的 RoleBinding `cluster-admin` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限；

完整 unit 见 [kube-proxy.service](../systemd/kube-proxy.service)

### 启动 kube-proxy

``` bash
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```## 配置 kube-proxy

**安装conntrack**

```bash
yum install -y conntrack-tools
```

**创建 kube-proxy 的service配置文件**

文件路径`/usr/lib/systemd/system/kube-proxy.service`。

```ini
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/local/bin/kube-proxy \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBE_MASTER \
	    $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

kube-proxy配置文件`/etc/kubernetes/proxy`。

``` bash
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--bind-address=172.20.0.113 --hostname-override=172.20.0.113 --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"
```

+ `--hostname-override` 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则；
+ kube-proxy 根据 `--cluster-cidr` 判断集群内部和外部流量，指定 `--cluster-cidr` 或 `--masquerade-all` 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；
+ `--kubeconfig` 指定的配置文件嵌入了 kube-apiserver 的地址、用户名、证书、秘钥等请求和认证信息；
+ 预定义的 RoleBinding `cluster-admin` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限；

完整 unit 见 [kube-proxy.service](../systemd/kube-proxy.service)

### 启动 kube-proxy

``` bash
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```


## 验证测试


查看集群状态 
```kubectl get cs```

我们创建一个nginx的service试一下集群是否可用。
```shell
$ kubectl run nginx --replicas=2 --labels="run=load-balancer-example" --image=nginx  --port=80
deployment "nginx" created
$ kubectl expose deployment nginx --type=NodePort --name=example-service
service "example-service" exposed
$ kubectl describe svc example-service
Name:            example-service
Namespace:        default
Labels:            run=load-balancer-example
Annotations:        <none>
Selector:        run=load-balancer-example
Type:            NodePort
IP:            10.254.62.207
Port:            <unset>    80/TCP
NodePort:        <unset>    32724/TCP
Endpoints:        172.30.60.2:80,172.30.94.2:80
Session Affinity:    None
Events:            <none>
$ curl "10.254.62.207:80"
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

















## 调试模式




直接加参数启动
 ```shell



/usr/local/bin/kubelet --address=172.16.97.244 --allow-privileged=true --anonymous-auth=false --authentication-token-webhook --authorization-mode=Webhook --client-ca-file=/etc/kubernetes/ssl/ca.pem --cluster-dns=10.68.0.2 --cluster-domain=cluster.local.  --fail-swap-on=false --hairpin-mode hairpin-veth --hostname-override="newhost" --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --max-pods=110 --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.1 --register-node=true --root-dir=/var/lib/kubelet --tls-cert-file=/etc/kubernetes/ssl/kubelet.pem --tls-private-key-file=/etc/kubernetes/ssl/kubelet-key.pem --cgroups-per-qos=true --cgroup-driver=systemd --v=2
```
