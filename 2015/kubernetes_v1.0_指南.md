#kubernetes v1.0 指南


操作系统：

CentOS7

```
$ uname -a
Linux ip-172-16-0-18 3.10.0-123.el7.x86_64 #1 SMP Mon Jun 30 12:09:22 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

Docker 版本

```
$ docker version
Client version: 1.7.1
Client API version: 1.19
Package Version (client): docker-1.7.1-108.el7.centos.x86_64
Go version (client): go1.4.2
Git commit (client): 3043001/1.7.1
OS/Arch (client): linux/amd64
Server version: 1.7.1
Server API version: 1.19
Package Version (server): docker-1.7.1-108.el7.centos.x86_64
Go version (server): go1.4.2
Git commit (server): 3043001/1.7.1
OS/Arch (server): linux/amd64
```
kubernetes 版本

```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"0", GitVersion:"v1.0.4", GitCommit:"65d28d5fd12345592405714c81cd03b9c41d41d9", GitTreeState:"clean"}
Server Version: version.Info{Major:"1", Minor:"0", GitVersion:"v1.0.4", GitCommit:"65d28d5fd12345592405714c81cd03b9c41d41d9", GitTreeState:"clean"}
```

###软件依赖

master 主机

etcd

kube-apiserver

kube-controller-manager

kube-scheduler

flanneld （docker 集群网络管理 可选）


minion 主机

kube-proxy

kubelet

flanneld

docker

### 安装指南

本文只介绍 CentOS7的安装方式，其他平台安装请移步 [这里](http://kubernetes.io/v1.0/gs-custom.html)

kubernetes 可以运行在一台机器上也可以运行在多台机器上，本文有两台物理机器 ip 分别是 172.16.0.17（master） 和 172.16.0.18（minion）

master 机器为主服务用来调度 各个minion 机器，最终应用程序容器将运行在 minion 的 Docker 容器中，当然可以在 master 机器上安装 docker 同时让其成为 minion node 机器提供服务。

下面开始具体操作，注意两台机器上都需要操作这些

首先先修改下系统 hosts

```
$ vi /etc/hosts
172.16.0.17   master
172.16.0.18   minion
```

添加一个软件源

```
$ vi /etc/yum.repos.d/virt7-testing.repo
[virt7-testing]
name=virt7-testing
baseurl=http://cbs.centos.org/repos/virt7-testing/x86_64/os/
gpgcheck=0
```
通过 yum 方式安装

yum -y install --enablerepo=virt7-testing kubernetes

再装个 etcd 官方文档推荐的是0.4.6这个最新的版本没有试过，这个 etcd 就是做服务发现的类zookeeper，也是 Go 语言写的

yum install http://cbs.centos.org/kojifiles/packages/etcd/0.4.6/7.el7.centos/x86_64/etcd-0.4.6-7.el7.centos.x86_64.rpm
yum -y install --enablerepo=virt7-testing kubernetes

然后在/etc/kubernetes/目录下就能看到配置文件啦，这个时候建议用官方最新的二进制包替换下 yum 源安装的二进制文件,因为 yum 源可能不是最新的可以在 github 上下载最新的[点击这里](https://github.com/kubernetes/kubernetes/releases/latest)或者[release版列表](https://github.com/kubernetes/kubernetes/releases)

然后解压里面的kubernetes/server/kubernetes-server-linux-amd64.tar.gz 把二进制文件替换到/usr/bin 下就好

通过 yum 方式安装后会在/etc/kubernetes目下生成配置文件，在/usr/lib/systemd/system 目录下生成 systemctl 的 Unit 文件 （类似服务配置文件）

然后开始修改/etc/kubernetes/下的配置文件了

先生成一个 key

```
openssl genrsa -out /tmp/serviceaccount.key 2048
```

===

/etc/kubernetes/apiserver

注意: 这里的master 与 minion机器的apiserver 配置的区别在 KUBE_API_PORT="--port=8080"
minion 注释掉就好,仅 master 监听


```
###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# Port minions listen on
# KUBELET_PORT="--kubelet_port=10250"

# Comma separated list of nodes in the etcd cluster 注意：etcd 这儿使用4001端口,请确保 master 机器的 etcd 监听的是此端口
KUBE_ETCD_SERVERS="--etcd_servers=http://master:4001"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

KUBE_MASTER="--master=http://master:8080"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"

# Add your own! (这儿就是刚刚生成的 key)
KUBE_API_ARGS="--service_account_key_file=/tmp/serviceaccount.key"
```

===
蓝后是
/etc/kubernetes/config
这块 master 与 minion 都一样

```
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow_privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://master:8080"
```

===

接着
/etc/kubernetes/controller-manager

master 机器改就行了

```
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--node-monitor-grace-period=10s --pod-eviction-timeout=10s --service_account_private_key_file=/tmp/serviceaccount.key"
```

接着
/etc/kubernetes/kubelet

master

```
###
# kubernetes kubelet (minion) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
# KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname_override=master"

# location of the api-server
KUBELET_API_SERVER="--api_servers=http://master:8080"

# Add your own!
KUBELET_ARGS=""
```

minion

```
###
# kubernetes kubelet (minion) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"

# The port for the info server to serve on
# KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname_override=minion"

# location of the api-server
KUBELET_API_SERVER="--api_servers=http://master:8080"

# Add your own!
KUBELET_ARGS=""
```

这里面的KUBELET_HOSTNAME 就是后面的 node 的名称，注意这个名称似乎要在 hosts 文件种定义好

===
就这么多其他的不用改，然后依次启动服务即可


master 机器上启动


```
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

systemctl 是 centos7 的一个新增的玩意儿
它实际上将 service 和 chkconfig 这两个命令组合到一起

systemctl restart  重启某个服务
systemctl enable 将某个服务设未开机自动运行
systemctl status 查看某个服务运行状态 (如果启动失败这儿能看到一些错误日志)

ps: systemctl 服务的单元文件在 /usr/lib/systemd/system 依赖文件在/etc/systemd/system

通过 yum 方式安装完kubernetes 之后默认已经创建好了这些服务直接启动就好 : )

一定要按照顺序启动,因为kubernetes 依赖 etcd 所以 etcd 得先起来,建议一个一个手动启动方便定位错误

起来之后就能用 kubectl了

```
[root@ip-172-16-0-17 home]# kubectl version
Client Version: version.Info{Major:"1", Minor:"0", GitVersion:"v1.0.4", GitCommit:"65d28d5fd12345592405714c81cd03b9c41d41d9", GitTreeState:"clean"}
Server Version: version.Info{Major:"1", Minor:"0", GitVersion:"v1.0.4", GitCommit:"65d28d5fd12345592405714c81cd03b9c41d41d9", GitTreeState:"clean"}
```


然后在启动节点机器,可以在 master 机器和minion 机器都启用这样的话 master 同时承担折 minion 的任务


```
for SERVICES in kube-proxy kubelet docker; do
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES
done
```

此时在 master 机器上用

```
[root@ip-172-16-0-17 home]# kubectl get nodes
NAME      LABELS                          STATUS
master    kubernetes.io/hostname=master   Ready
minion    kubernetes.io/hostname=minion   Ready
```

其他机器用

```
kubectl -s http://master:8080
```
指定 master主机就行

此时kubernetes 就搭建完毕了，可以用官方的 example pods 测试下 容器是否正常运行

```
 kubectl create -f docs/user-guide/walkthrough/pod-nginx-with-label.yaml
```

这个 docs 就是之前下载的kubernetes.tar.gz

这篇先到这里，后面在介绍用 flanneld 优化docker的覆盖网络