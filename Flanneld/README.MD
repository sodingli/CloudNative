

所有的node节点都需要安装网络插件才能让所有的Pod加入到同一个局域网中，本文是安装flannel网络插件的参考文档。


## 安装方式

### 直接yum 安装
建议直接使用yum安装flanneld，除非对版本有特殊需求，。

```bash
yum install -y flannel
```


### 二进制安装
可以去这里下载：https://github.com/coreos/flannel/releases/

```shell
wget  https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz

tar zxvf flannel-v0.11.0-linux-amd64.tar.gz
```
解压后，有两个可执行文件，`mk-docker-opts.sh`,`flanneld` 这两个文件。



## 配置文件

需要先创建这4个文件夹：
```shell
mkdir -p /etc/sysconfig/flanneld
mkdir -p /etc/sysconfig/docker-network
mkdir -p /run/flannel/docker
mkdir -p /usr/libexec/flannel/
```
cp -rf mk-docker-opts.sh /usr/libexec/flannel/mk-docker-opts.sh



service配置文件`/usr/lib/systemd/system/flanneld.service`。

```ini
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/flanneld
EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=/usr/bin/flanneld-start \
  -etcd-endpoints=${FLANNEL_ETCD_ENDPOINTS} \
  -etcd-prefix=${FLANNEL_ETCD_PREFIX} \
  $FLANNEL_OPTIONS
ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
```

`/etc/sysconfig/flanneld`配置文件：

```ini
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="https://172.20.0.113:2379,https://172.20.0.114:2379,https://172.20.0.115:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
FLANNEL_OPTIONS="-etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem"
```

如果是多网卡（例如vagrant环境），则需要在FLANNEL_OPTIONS中增加指定的外网出口的网卡，例如-iface=eth2




同时docker 配置文件也需要更改下配置

```inf
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target
Wants=docker-storage-setup.service
Requires=docker-cleanup.timer

[Service]
Type=notify
NotifyAccess=main
EnvironmentFile=-/run/containers/registries.conf
EnvironmentFile=-/run/docker_opts.env
EnvironmentFile=-/run/flannel/subnet.env
EnvironmentFile=-/run/flannel/docker
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
Environment=GOTRACEBACK=crash
Environment=DOCKER_HTTP_HOST_COMPAT=1
Environment=PATH=/usr/libexec/docker:/usr/bin:/usr/sbin
ExecStart=/usr/bin/dockerd-current \
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
          --default-runtime=docker-runc \
          --exec-opt native.cgroupdriver=systemd \
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
          --init-path=/usr/libexec/docker/docker-init-current \
          --seccomp-profile=/etc/docker/seccomp.json \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY \
	  $REGISTRIES
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0
Restart=on-abnormal
KillMode=process

[Install]
WantedBy=multi-user.target
```

这几行 ,加入docker 的环境变量中。
```
EnvironmentFile=-/run/docker_opts.env
EnvironmentFile=-/run/flannel/subnet.env
```
这两个文件是需要用`./mk-docker-opts.sh -i` 自动生成。如果没生成，就得手动生成。



**在etcd中创建网络配置**

执行下面的命令为docker分配IP地址段。

```bash
etcdctl --endpoints=https://172.20.0.113:2379,https://172.20.0.114:2379,https://172.20.0.115:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  mkdir /kube-centos/network
etcdctl --endpoints=https://172.20.0.113:2379,https://172.20.0.114:2379,https://172.20.0.115:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  mk /kube-centos/network/config '{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'
```

如果你要使用`host-gw`模式，可以直接将vxlan改成`host-gw`即可。

## 启动flannel

```bash
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```

应该在 /run/flannel/subnet.env 和/run/docker_opts.env 有链各个文件。

文件内容如下

```shell
 cat /run/flannel/subnet.env
FLANNEL_NETWORK=172.30.0.0/16
FLANNEL_SUBNET=172.30.8.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
# cat /run/docker_opts.env
DOCKER_OPT_BIP="--bip=172.30.8.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=false"
DOCKER_OPT_MTU="--mtu=1450"
```
如果没有 /run/docker_opts.env 文件，
需要 `执行./mk-docker-opts.sh -i` 将会生成如下两个文件环境变量文件，这个文件就是我们刚才配置docker 里的环境变量


设置docker0网桥的IP地址
```shell
source /run/flannel/subnet.env
ifconfig docker0 $FLANNEL_SUBNET
```
这样docker0和flannel网桥会在同一个子网中，如
```shell
[root@localhost .kube]# ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.30.8.1  netmask 255.255.255.0  broadcast 172.30.8.255
        ether 02:42:07:d8:be:e9  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 172.30.8.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::38b9:84ff:fec3:577c  prefixlen 64  scopeid 0x20<link>
        ether 3a:b9:84:c3:57:7c  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0
```
现在就可以重启docker了。

## 验证
现在查询etcd中的内容可以看到：

```bash
$etcdctl --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  ls /kube-centos/network/subnets
/kube-centos/network/subnets/172.30.8.0-24


$etcdctl --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  get /kube-centos/network/config
{ "Network": "172.30.8.0/16", "SubnetLen": 24, "Backend": { "Type": "vxlan" } }


$/etc/ansible/bin/etcdctl   --ca-file=/etc/kubernetes/ssl/ca.pem   --cert-file=/etc/kubernetes/ssl/kubernetes.pem   --key-file=/etc/kubernetes/ssl/kubernetes-key.pem  --endpoints="https://172.16.97.11:2379" get /kube-centos/network/subnets/172.30.8.0-24
{"PublicIP":"172.16.97.244","BackendType":"vxlan","BackendData":{"VtepMAC":"3a:b9:84:c3:57:7c"}}


```



## 本地验证

在本地执行ip addr 会出现一个 flannel 的一个网卡。
```shell
ip addr

3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:07:d8:be:e9 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 3a:b9:84:c3:57:7c brd ff:ff:ff:ff:ff:ff
    inet 172.30.8.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::38b9:84ff:fec3:577c/64 scope link
       valid_lft forever preferred_lft forever
```




## debug

我在运行配置过程中，如论如何都重启失败。我就用命令行启动了，后续可以参考下这样的debug方式。
```shell

#方案一
/usr/bin/flanneld -etcd-endpoints=https://172.16.97.11:2379 -etcd-cafile=/etc/kubernetes/ssl/ca.pem   -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem  -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem  -etcd-prefix=/kube-centos/network  -public-ip=172.16.97.244

#方案二

usr/bin/flanneld --ip-masq -etcd-endpoints=https://172.16.97.11:2379 -etcd-cafile=/etc/kubernetes/ssl/ca.pem   -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem  -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem  -etcd-prefix=/kube-centos/network  -public-ip=172.16.97.244 ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
```

+ endpoint 地址是etcd的主机地址；
+ public-ip 本机 IP。



确定  从etcd 获取到 key 
```
etcdctl get /kube-centos/network/config
```


