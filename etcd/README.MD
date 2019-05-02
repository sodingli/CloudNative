## etcd 安装


### 安装的三种方式
#### 二进制安装


下载二进制文件到 `https://github.com/coreos/etcd/releases` 页面下载最新版本的二进制文件
```bash
wget https://github.com/coreos/etcd/releases/download/v3.1.5/etcd-v3.1.5-linux-amd64.tar.gz
tar -xvf etcd-v3.1.5-linux-amd64.tar.gz
mv etcd-v3.1.5-linux-amd64/etcd* /usr/local/bin
```

#### yum  安装
`yum install etcd`
若使用`yum`安装，默认`etcd`命令将在`/usr/bin`目录下，注意修改下面的`etcd.service`文件中的启动命令地址为`/usr/bin/etcd`









#### 脚本安装

脚本来源于官方文档,修改下安装的路径，就可以了。

install_ectd.sh
```shell
ETCD_VER=v3.3.12

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
ETCDCTL_API=3 /tmp/etcd-download-test/etcdctl version
```

以上三种方式安装完，需要 systemctl 对服务进行管理，需要安装更改配置文件。

### 创建配置文件 
创建 etcd 的 systemd unit 文件

如果用 systemd 管理集群，一般配置文件在这三个地方：
```
/etc/systemd/system/
/usr/lib/systemd/system/
/lib/systemd/system/
```
/lib/systemd/system/ 路径一般是系统的的管理配置，放这里不接受系统监管。



`etcd` 服务一般有两个配置 ，一个是服务配置 `etcd.service` ，一个是变量配置`etcd.conf`。


##### 服务配置etcd.service
一般配置的路径在`/usr/lib/systemd/system/`目录下创建文件`etcd.service`，内容如下。

注意替换IP地址为你自己的etcd集群的主机IP。


```shell
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --data-dir=${ETCD_DATA_DIR}\
  --listen-client-urls ${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
  --advertise-client-urls ${ETCD_ADVERTISE_CLIENT_URLS} \
  --listen-peer-urls ${ETCD_LISTEN_PEER_URLS} \
  --initial-advertise-peer-urls ${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
  --initial-cluster etcd1=http://172.16.97.244:2380,etcd2=https://172.16.97.245:2380,etcd3=https://172.16.97.246:2380 \
  --initial-cluster-token ${ETCD_INITIAL_CLUSTER_TOKEN} \
  --initial-cluster-state new
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

##### 变量和客户端配置文件etcd.conf 
路径`/etc/etcd/etcd.conf`
环境变量的内容如下：
```shell
# [member]
#集群名字
ETCD_NAME=etcd1
ETCD_DATA_DIR="/var/lib/etcd"
#集群地址https,如果https访问需要加证书
ETCD_LISTEN_PEER_URLS="http://172.16.97.244:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.16.97.244:2379"

#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.16.97.244:2380"
#集群验证的token 自定义
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://172.16.97.244:2379"
```


#### 启动服务
初始化服务，并启动配置：

```systemctl daemon-reload && systemctl restart etcd.service && systemctl status etcd.service```

```
如果不成功通过查看异常
```
journalctl -xe
```


#### 测试
```shell
$etcdctl member list
967c7968d162820f: name=etcd1 peerURLs=http://172.16.97.244:2380 clientURLs=http://172.16.97.244:2379 isLeader=true

$etcdctl cluster-health
member 967c7968d162820f is healthy: got healthy result from http://172.16.97.244:2379
cluster is healthy
```



#### 证书版本etcd 集群

确定集群的IP。

|集群|地址||
|-|-|-|
|etcd1|172.20.0.113||
|etcd2|172.20.0.114||
|etcd3|172.20.0.115||

先生成证书，连接k8s 需要生成多个证书。 
看创建证书的章节。

##### 集群配置
更改配置文件：
在`/usr/lib/systemd/system/`目录下创建文件`etcd.service`，内容如下。注意替换IP地址为你自己的etcd集群的主机IP。
```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --peer-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls ${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
  --listen-peer-urls ${ETCD_LISTEN_PEER_URLS} \
  --listen-client-urls ${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
  --advertise-client-urls ${ETCD_ADVERTISE_CLIENT_URLS} \
  --initial-cluster-token ${ETCD_INITIAL_CLUSTER_TOKEN} \
  --initial-cluster infra1=https://172.20.0.113:2380,infra2=https://172.20.0.114:2380,infra3=https://172.20.0.115:2380 \
  --initial-cluster-state new \
  --data-dir=${ETCD_DATA_DIR}
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target


```


环境变量配置文件/etc/etcd/etcd.conf。
```bash
# [member]
ETCD_NAME=infra1
ETCD_DATA_DIR="/var/lib/etcd"

#因为有证书了，所以这里用的https协议，以下都是
ETCD_LISTEN_PEER_URLS="https://172.20.0.113:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.20.0.113:2379"

#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.20.0.113:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.20.0.113:2379"
```
这是172.20.0.113节点的配置，



其他两个etcd节点只要将上面的IP地址改成相应节点的IP地址即可。ETCD_NAME换成对应节点的infra1/2/3。


##### 启动服务
初始化服务，并启动配置：

```
systemctl daemon-reload && systemctl restart etcd.service && systemctl status etcd.service```
```
如果不成功通过查看异常
```
journalctl -xe
```


#### 验证服务

用证书来连接服务集群。
```bash
etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  cluster-health
```

















### etcd 集群管理
- add
- del
- remove

### etcd 启动


### etcd debug

/etcd/etcdctl --debug  member list 查看日志

去/var/log/etcd下看日志

journalctl -u etcd.service --since today 去看日志

 journalctl -xn -u etcd -fl

### etcd 管理
etcdctl 是etcd 服务的命令行工具， 它分为etcdctl v2 和etcdctl v3 版本。确定下要和服务版本的匹配。
```
/opt/etcd/etcdctl  --help
...
VERSION:
   3.0.12
...
```
增加命令：
命令：`set [options] <key> <value>`
例子


```shell 
#/opt/etcd/etcdctl set name "lidashuai"
lidashuai
#/opt/etcd/etcdctl get  name 
lidashuai
#/opt/etcd/etcdctl update  name "superman"
superman
#/opt/etcd/etcdctl get  name 
superman
#当然中间可以加上 -- 线
#/opt/etcd/etcdctl set -- wifename "wanglin"
#支持过期时间 

```


```
get 
#获取里面的值
#/opt/etcd/etcdctl get   name "superman"
superman --sort

delete 
#删除操作
/opt/etcd/etcdctl rm  wifename
```
