#### 安装

注意请下载对应的Kubernetes版本的安装包。
```bash
wget https://dl.k8s.io/v1.6.0/kubernetes-client-linux-amd64.tar.gz
tar -xzvf kubernetes-client-linux-amd64.tar.gz
cp kubernetes/client/bin/kube* /usr/bin/
chmod a+x /usr/bin/kube*

```

#### 创建 kubectl kubeconfig 文件

文件默认在`~/.kube/config` 如果没有，可以新建。

``` bash
export KUBE_APISERVER="https://172.16.97.11:6443"
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}
# 设置客户端认证参数
kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem
# 设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin
# 设置默认上下文
kubectl config use-context kubernetes
```

+ `admin.pem` 证书 OU 字段值为 `system:masters`，`kube-apiserver` 预定义的 RoleBinding `cluster-admin` 将 Group `system:masters` 与 Role `cluster-admin` 绑定，该 Role 授予了调用`kube-apiserver` 相关 API 的权限；
+ 生成的 kubeconfig 被保存到 `~/.kube/config` 文件；

**注意：**`~/.kube/config`文件拥有对该集群的最高权限，请妥善保管。







#### 调整自动
```shell
$source <(kubectl completion bash) # setup autocomplete in bash into the current shell, bash-completion package should be installed first.
$echo "source <(kubectl completion bash)" >> ~/.bashrc # add autocomplete permanently to your bash shell.


$alias k=kubectl
$complete -F __start_kubectl k

```


#### 配置
使用 kubectl 的第一步是配置 Kubernetes 集群以及认证方式，包括

* cluster 信息：Kubernetes server 地址
* 用户信息：用户名、密码或密钥
* Context：cluster、用户信息以及 Namespace 的组合

**示例**
```bash
kubectl config set-credentials myself --username=admin --password=secret
kubectl config set-cluster local-server --server=http://localhost:8080
kubectl config set-context default-context --cluster=local-server --user=myself --namespace=default
kubectl config use-context default-context
kubectl config view
```

#### 例子

```bash
kubectl -h 查看子命令列表

kubectl -n namespace

kubectl -f  Labelname

kubectl options 查看全局选项

kubectl <command> --help 查看子命令的帮助

kubectl [command] [PARAMS] -o=<format> 设置输出格式（如 json、yaml、jsonpath 等）

kubectl explain [RESOURCE] 查看资源的定义
```


#### 常用命令格式

* 创建：kubectl run <name> --image=<image> 或者 kubectl create -f manifest.yaml
* 直接创建一个新` kubectl run --generator=run-pod/v1 --image=nginx nginx-app --port=80 --env="DOMAIN=cluster"`
* 查看 kubectl describe 

* 导出配置：kubectl get deployments.apps *deploymentsname*  -o yaml > nginx2_yaml
* 
* 查询：kubectl get deployments. *deploymentsname*   -o json
* 
* 更新 kubectl set 或者 kubectl patch
* `kubectl patch pod nodename -p '{"metadata":{"labels":{"app":"newname"}}}'`
* 
* 删除：kubectl delete <resource> <name> 或者 kubectl delete -f manifest.yaml
* `kubectl  delete pods --field-selector=status.phase=Pending`
* `delete pods --field-selector=status.phase=Pending --grace-period=0 --force=true` #强制杀掉
* 
* 查询 Pod IP：kubectl get pod <pod-name> -o jsonpath='{.status.podIP}'
* 
* 容器内执行命令：kubectl exec -ti <pod-name> sh
* 
* 容器日志：kubectl logs [-f] <pod-name>
* 
* `kubectl  describe pod|node  podname|nodename `
* `kubectl describe pod podname --namespace=spectail-namespace`
* 
* 
* 导出服务：kubectl expose deploy <name> --port=80
* 
* 在线扩容服务:kubectl scale --replicas=3 deployment/*deploymentsname* 
* 

Base64 解码：

```kubectl get secret SECRET -o go-template='{{ .data.KEY | base64decode}}'```
注意，kubectl run 仅支持 Pod、Replication Controller、Deployment、Job 和 CronJob 等几种资源。具体的资源类型是由参数决定的，默认为 Deployment：




![1b32ee60fc156966f57c570a9739bd0a.png](evernotecid://CF28A078-1096-40A0-9ACD-0DAA8CE64AC7/appyinxiangcom/6208230/ENResource/p761)

