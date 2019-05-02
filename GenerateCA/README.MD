创建证书


### K8S集群用到的证书说明


基于 cfssl 快速生成证书文件。 

整个k8s 集群需要这么几个证书和私钥

生成的 CA 证书和秘钥文件如下：

**公共证书**

* ca-key.pem
* ca.pem

**集群证书**

* kubernetes-key.pem
* kubernetes.pem

**代理证书**

* kube-proxy.pem
* kube-proxy-key.pem

**管理证书**

* admin.pem
* admin-key.pem


使用证书的组件如下：
|服务|需要证书|证书梳理|备注|
|-|-|-|-|-|
|kube-controller-manager|ca-key.pem、ca.pem|2|管理需要主根证书|
|kube-apiserver|ca.pem、kubernetes-key.pem、kubernetes.pem；|3|根证书+集群证书|
|kube-proxy| ca.pem、kube-proxy-key.pem、kube-proxy.pem；|3|根证书+代理证书|
|kubelet|ca.pem|1|根证书|
|etcd|ca.pem、kubernetes-key.pem、kubernetes.pem|3|根证书+集群证书|
|kubectl|ca.pem、admin-key.pem、admin.pem；|3|根证书+管理证书|



证书创建一次，在其它节点拷贝过去就可以了。



### 安装证书创建工具CFSSL
两种安装方式
#### 方式一：直接使用二进制源码包安装
```shell
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

export PATH=/usr/local/bin:$PATH
```
#### 方式二：使用go命令安装

我们的系统中安装了Go1.7.5，如果没有安装，可以去看go环境安装 ，使用以下命令安装更快捷：
```bash
$ go get -u github.com/cloudflare/cfssl/cmd/...
$ echo $GOPATH
/usr/local
$ls /usr/local/bin/cfssl*
cfssl cfssl-bundle cfssl-certinfo cfssljson cfssl-newkey cfssl-scan
```
在$GOPATH/bin目录下得到以cfssl开头的几个命令。


### 创建TLS证书和秘钥

#### 创建公共 CA证书配置

**创建 CA 配置文件**

``` bash
mkdir /root/ssl
cd /root/ssl
cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
# 根据config.json文件的格式创建如下的ca-config.json文件
# 过期时间设置成了 87600h
#新生成了ca-config.json
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```
字段说明

+ `ca-config.json`：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
+ `signing`：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 `CA=TRUE`；
+ `server auth`：表示client可以用该 CA 对server提供的证书进行验证；
+ `client auth`：表示server可以用该CA对client提供的证书进行验证；




**创建 CA 证书签名请求**

创建 `ca-csr.json`  文件，内容如下：

``` json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ],
    "ca": {
       "expiry": "87600h"
    }
}
```

+ "CN"：`Common Name`，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
+ "O"：`Organization`，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；




**生成 CA 证书和私钥**

``` bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
$ ls ca* 
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```





#### 创建公共 kubernetes 证书

创建 kubernetes 证书签名请求文件 `kubernetes-csr.json`：

``` json
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "172.20.0.112",
      "172.20.0.113",
      "172.20.0.114",
      "172.20.0.115",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

+ 如果 hosts 字段不为空则需要指定授权使用该证书的 **IP 或域名列表**，由于该证书后续被 `etcd` 集群和 `kubernetes master` 集群使用，所以上面分别指定了
    + **`etcd`集群**
    + **`kubernetes master`集群的主机IP**
    + **`kubernetes` 服务的服务 IP**（一般是 `kube-apiserver` 指定的 `service-cluster-ip-range` 网段的第一个IP，如 10.254.0.1）。

    
   ```
      "127.0.0.1", #本机iP
      "172.20.0.112", #etcd 集群ip1
      "172.20.0.113", #etcd 集群ip2
      "172.20.0.114", #etcd 集群ip2
      "172.20.0.115", # master 主机IP
      "10.254.0.1",   #虚拟service IP
   ```
    
   
    
+ 这是最小化安装的kubernetes集群，包括一个私有镜像仓库，三个节点的kubernetes集群，以上物理节点的IP也可以更换为主机名。

**生成 kubernetes 证书和私钥**

``` bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
$ ls kubernetes*
kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
```

第二中方式，不用编辑json 文件，直接在命令行上指定相关参数：

``` bash
echo '{"CN":"kubernetes","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes -hostname="127.0.0.1,172.20.0.112,172.20.0.113,172.20.0.114,172.20.0.115,kubernetes,kubernetes.default" - | cfssljson -bare kubernetes
```



#### 创建admin 证书

创建 admin 证书签名请求文件 `admin-csr.json`：

``` json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

+ 由于 `kube-apiserver` 使用 `RBAC` 对客户端(如 `kubelet`、`kube-proxy`、`Pod`)请求进行授权；所以预定义了一些 `RBAC` 使用的 `RoleBindings`，如 `cluster-admin` 将 Group `system:masters` 与 Role `cluster-admin` 绑定，该 Role 授予了调用`kube-apiserver` 的**所有 API**的权限；

+ O 指定该证书的 Group 为 `system:masters`，
`kubelet` 使用该证书访问 `kube-apiserver` 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 `system:masters`，所以被授予访问所有 API 的权限；

**注意**：这个admin 证书，是将来生成管理员用的kube config 配置文件用的，现在我们一般建议使用RBAC 来对kubernetes 进行角色权限控制， kubernetes 将证书中的CN 字段 作为User， O 字段作为 Group（具体参考[ Kubernetes中的用户与身份认证授权](../guide/authentication.md)中 X509 Client Certs 一段）。 


在搭建完 kubernetes 集群后，我们可以通过命令: `kubectl get clusterrolebinding cluster-admin -o yaml` ,查看到 `clusterrolebinding cluster-admin` 的 subjects 的 kind 是 Group，name 是 `system:masters`。 `roleRef`  对象是 `ClusterRole cluster-admin`。 意思是凡是 `system:masters Group` 的 user 或者 `serviceAccount` 都拥有 `cluster-admin` 的角色。 因此我们在使用 kubectl 命令时候，才拥有整个集群的管理权限。可以使用 `kubectl get clusterrolebinding cluster-admin -o yaml` 来查看。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2019-04-16T08:04:07Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "95"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/cluster-admin
  uid: 3592b921-601e-11e9-a8cb-000c2990b283
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters
```

生成 admin 证书和私钥：

``` bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
$ ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem
```



#### 创建kube-proxy 证书
创建 kube-proxy 证书签名请求文件 `kube-proxy-csr.json`：

``` json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

+ CN 指定该证书的 User 为 `system:kube-proxy`；
+ `kube-apiserver` 预定义的 RoleBinding `system:node-proxier` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限；

生成 kube-proxy 客户端证书和私钥

``` bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
$ ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem
```





### 证书校验
生产的证书需要验证其正确性.

以 kubernetes 证书为例 
##### 第一中方法使用 `opsnssl` 命令
校验证书的pem 文件与`ca.json,kubernetes-csr.json, ca-config.json`  文件内容是否一致。

``` bash
$ openssl x509  -noout -text -in  kubernetes.pem
...
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=BeiJing, L=BeiJing, O=k8s, OU=System, CN=Kubernetes
        Validity
            Not Before: Apr  5 05:36:00 2017 GMT
            Not After : Apr  5 05:36:00 2018 GMT
        Subject: C=CN, ST=BeiJing, L=BeiJing, O=k8s, OU=System, CN=kubernetes
...
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                DD:52:04:43:10:13:A9:29:24:17:3A:0E:D7:14:DB:36:F8:6C:E0:E0
            X509v3 Authority Key Identifier:
                keyid:44:04:3B:60:BD:69:78:14:68:AF:A0:41:13:F6:17:07:13:63:58:CD

            X509v3 Subject Alternative Name:
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.default.svc.cluster.local, IP Address:127.0.0.1, IP Address:172.20.0.112, IP Address:172.20.0.113, IP Address:172.20.0.114, IP Address:172.20.0.115, IP Address:10.254.0.1
...
```

+ 确认 `Issuer` 字段的内容和 `ca-csr.json` 一致；
+ 确认 `Subject` 字段的内容和 `kubernetes-csr.json` 一致；
+ 确认 `X509v3 Subject Alternative Name` 字段的内容和 `kubernetes-csr.json` 一致；
+ 确认 `X509v3 Key Usage、Extended Key Usage` 字段的内容和 `ca-config.json` 中 `kubernetes` profile 一致；

##### 使用 `cfssl-certinfo` 命令

``` bash
$ cfssl-certinfo -cert kubernetes.pem
...
{
  "subject": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "issuer": {
    "common_name": "Kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "Kubernetes"
    ]
  },
  "serial_number": "174360492872423263473151971632292895707129022309",
  "sans": [
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local",
    "127.0.0.1",
    "10.64.3.7",
    "10.254.0.1"
  ],
  "not_before": "2017-04-05T05:36:00Z",
  "not_after": "2018-04-05T05:36:00Z",
  "sigalg": "SHA256WithRSA",
...
```

### 分发证书

将生成的证书和秘钥文件（后缀名为`.pem`）拷贝到所有机器的 `/etc/kubernetes/ssl` 目录下备用；

``` bash
mkdir -p /etc/kubernetes/ssl
cp *.pem /etc/kubernetes/ssl
```








#### 参考

去GitHub 上了解下 https://github.com/cloudflare/cfssl 了解项目。
