---
title: Kubernetes集群的安装记录
date: 2020-06-25 22:55:25
tags:
---

## Step 0. Kubernets集群安装的准备工作

本次安装测试的基础硬件是Intel NUC5i3 & NUC8i3，基准软件是：

- Centos 7.8（启动速度快、版本稳定性好）
- Docker-ce 19.03.12
- Kubernetes 1.18.2 (v1.18.4拉取国内镜像不成功！！！)
- Calico 3.8.9
- Nginx-ingress 1.5.3

本Kubernetes集群包含了2个Node，分别是：

- 1个Master节点：Hostname = master1，IPADDR = 192.168.0.132
- 1个Worker节点：Hostname = worker1，IPADDR = 192.168.0.130

2台已安装Linux的服务器，分别作为Master和Worker节点。

- 确认服务器支持并已开启硬件虚拟化支持（NUC各个型号均默认支持）
- 刚刚完成安装的Centos尚未配置网络，需要激活网卡并确认可以访问公网，具体方法请参见[Centos 7.8安装步骤](centos-install.md)

## Step 1. 修改操作系统配置

- 关闭SElinux和Firewalld服务
  
    SELinux是一个安全体系结构，它通过LSM(Linux Security Modules)框架被集成到Linux Kernel 2.6.x中。

    SELinux提供了一种灵活的强制访问控制(MAC)系统，且内嵌于Linux Kernel中。SELinux定义了系统中每个【用户】、【进程】、【应用】和【文件】的访问和转变的权限，然后它使用一个安全策略来控制这些实体(用户、进程、应用和文件)之间的交互，安全策略指定如何严格或宽松地进行检查

``` shell
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
setenforce 0
systemctl disable firewalld
systemctl stop firewalld
```

- 关闭swap服务

``` shell
swapoff -a
sed -i '/swap/d' /etc/fstab
free
```

- 修改Linux内核参数以启用bridge功能

    br_netfilter通过和linux bridge功能联动，以实现透明防火墙功能，具体方法是在Bridge层的通过执行Netfilter钩子来实现IP报文过滤。

    sysctl命令用于运行时配置内核参数，这些参数位于`/proc/sys`目录下，可以用`sysctl`来设置或重新设置联网功能，如IP转发、IP碎片去除以及源路由检查等

``` shell
modprobe br_netfilter

echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf

sysctl -p
```

- 安装几个基础的工具软件
  
``` shell
yum -y install yum-utils lvm2 device-mapper-persistent-data nfs-utils xfsprogs wget net-tools
```

## Step 2. 安装Docker服务

- 如果已安装过Docker的旧版本，需要全部卸载，指令是：

   `yum -y remove docker-client docker-client-latest docker-common docker-latest docker-logrotate docker-latest-logrotate docker-selinux docker-engine-selinux docker-engine`

- 添加阿里云的yum源设置，并安装Docker服务，本次Docker安装的stable版本号为`19.03.12`

``` shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce docker-ce-cli containerd.io

systemctl enable docker
systemctl start docker
```

- 确认Docker服务启动正常后，还需要设置国内的image镜像源

    `exec-opts`：**必须的**！目的是修改Docker Cgroup Driver为systemd，如果不修改则在后续添加Worker节点时可能会遇到`detected cgroupfs as ths Docker driver.xx`”`的报错信息，导致Docker镜像库配置为本地源，而无法正常拉取image。

    `registry-mirrors`：设置了几个国内可用的镜像源，可以根据实现情况调整

``` shell
cat > /etc/docker/daemon.json <<EOF
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m"
    },
    "storage-driver": "overlay2",
    "registry-mirrors":[
        "https://kfwkfulq.mirror.aliyuncs.com",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.docker-cn.com"
    ]
}
EOF
```

- 再次启动Docker，确认配置已修改

``` shell
systemctl daemon-reload
systemctl restart docker
docker info
```

## Step 3. 安装Kubernets软件包

- 如果已安装过Docker的旧版本，需要删除，指令是：`yum -y remove kubelet kubadm kubctl`

- 在Yum仓库源的配置目录`/etc/yum.repos.d/`中，新增`kubernetes.repo`并指向阿里云的国内镜像站点

``` shell
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

- 安装Kubernets软件包，本次安装的stable版本号为`1.18.4`，其中：

    kubelet：此组件是运行在每一个集群节点上的代理程序。它确保 Pod 中的容器处于运行状态。Kubelet 通过多种途径获得 PodSpec 定义，并确保 PodSpec 定义中所描述的容器处于运行和健康的状态。Kubelet不管理不是通过 Kubernetes 创建的容器。

    kubeadm：Kubernetes的安装工具。

    kubectl：作为Kubernetes的客户端CLI工具，可以让用户通过命令行的方式对Kubernetes集群进行操作。**Master节点必备，Worker节点可以选装**

``` shell
yum install kublet-1.18.2 kubeadm-1.18.2 kubectl-1.18.2 -y
# yum安装指定版本的软件，查看版本信息的方法是：yum list kubelet --showduplicates |expand

systemctl enable kubelet
systemctl start kubelet
```

> **注意**：此时系统重复启动`kubelet`不成功，原因是`kubeadm`尚未完成初始化，不要管他，后面自然就好了！

## Step 4. 在Master节点设置运行环境，并初始化启动Kubernetes

- 设置Kubernetes运行环境

``` shell
hostnamectl set-hostname master1
# 设置 hostname，保存在 /etc/hostname

export MASTER_IP=192.168.0.132
export APISERVER_NAME=master1
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
# 设置Master节点的IP地址和Hostname，并保存在DNS本地解析配置文件 /etc/hosts

export POD_SUBNET=10.100.0.1/16nets
# 设置Kubernetes 容器组所在的网段，该网段安装完成后，由 kubernetes 创建，事先并不存在于您的物理网络中
```

- Master节点初始化，如果失败，可以输入`kubeadm reset`回滚，观察错误提示并修改配置后重试

```shell
kubeadm init \
    --apiserver-advertise-address 0.0.0.0 \
    --apiserver-bind-port 6443 \
    --cert-dir /etc/kubernetes/pki \
    --control-plane-endpoint master1 \
    --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
    --kubernetes-version 1.18.2 \
    --pod-network-cidr 10.11.0.0/16 \
    --service-cidr 10.20.0.0/16 \
    --service-dns-domain cluster.local \
    --upload-certs
```

>- apiserver-advertise-address：公布API 服务器所正在监听的 IP 地址,指定`0.0.0.0`以使用默认网络接口的地址。**切记只可以是内网IP，不能是外网IP**，如果有多网卡，可以使用此选项指定某个网卡
>- control-plane-endpoint：为控制平面指定一个稳定的 IP 地址或 DNS 名称，指定的 master1 已经在`/etc/hosts`配置解析为本机IP
>- image-repository：选择用于拉取Control-plane的镜像的容器仓库，默认值：`k8s.gcr.io`，**因为GW的原因必须修改为国内源**
>- kubernetes-version：为Control-plane选择一个特定的 Kubernetes 版本， 默认值：`stable-1`，本次指定`1.18.4`
>-
>- node-name: 指定节点的名称,不指定的话为主机hostname，默认可以不指定
>- apiserver-bind-port：API 服务器绑定的端口，默认 `6443`
>- cert-dir： 保存和存储证书的路径，默认值：`/etc/kubernetes/pki`
>- pod-network-cidr: 指定pod的IP地址范围
>- service-cidr: 指定Service的IP地址范围，默认"10.96.0.0/12"
>- service-dns-domain: 为Service另外指定域名，默认"cluster.local"
>- upload-certs: 指定Control-plane 将证书上传到 kubeadm-certs Secret

- 如果Master节点启动成功，可以根据如下屏幕提示生成`$HOME/.kube/config`配置文件，完成后就可以使用`kubectl`了

``` shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# 在$HOME/.kube下生成config文件，保存master的登录信息

sudo chown $(id -u):$(id -g) $HOME/.kube/config
# 用于为普通用户分配kubectl权限
```

- 安装Calico网络插件
  
  集群必须安装网络插件以实现Pod间通信，只需要在Master节点操作，其他Node节点会自动创建相关Pod。
  该配置文件默认采用的Pod的IP地址为192.168.0.0/16，需要修改为集群初始化参数中采用的值，本例中为10.10.0.0/16；

```shell
wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml

sed -i "s#192\.168\.0\.0/16#10\.10\.0\.0/16#" calico.yaml
kubectl apply -f calico.yaml
```

- 监控容器状态，等待所有容器状态处于Running状态

```shell
watch -n 2 kubectl get pods -n kube-system -o wide
```

- 检查Master的状态，看到下面的结果就说明初始化成功了！！！

``` sh
[root@localhost ~]# kubectl get nodes -o wide
NAME      STATUS   ROLES    AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
master1   Ready    master   3m33s   v1.18.4   192.168.0.132   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   docker://19.3.12

[root@localhost ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-75d555c48-v286d   1/1     Running   0          2m36s
kube-system   calico-node-vz9v2                         1/1     Running   0          2m36s
kube-system   coredns-546565776c-crk2r                  1/1     Running   0          3m28s
kube-system   coredns-546565776c-n6kmm                  1/1     Running   0          3m28s
kube-system   etcd-master1                              1/1     Running   0          3m38s
kube-system   kube-apiserver-master1                    1/1     Running   0          3m38s
kube-system   kube-controller-manager-master1           1/1     Running   0          3m38s
kube-system   kube-proxy-bggcp                          1/1     Running   0          3m29s
kube-system   kube-scheduler-master1                    1/1     Running   0          3m38s
```

## Step 5. 安装Kuboard监控面板

- 在Master节点,安装[Ingress Controller](https://github.com/nginxinc/kubernetes-ingress/blob/v1.5.3/docs/installation.md)

```shell
kubectl apply -f https://kuboard.cn/install-script/v1.16.0/nginx-ingress.yaml
```

- 在Master节点执行，在线下载并启用Kuboard监控工具包（由kuboard.cn提供，不是默认的Dashboard）

``` shell
kubectl apply -f https://kuboard.cn/install-script/kuboard.yaml
kubectl apply -f https://addons.kuboard.cn/metrics-server/0.3.6/metrics-server.yaml
```

- 持续监控pod运行情况，直到如下信息就说明成功了

``` cmd
$ kubectl get pods -l k8s.kuboard.cn/name=kuboard -n kube-system

NAME                       READY   STATUS        RESTARTS   AGE
kuboard-54c9c4f6cb-6lf88   1/1     Running       0          45s
```

- 在第一个Master节点，以admin角色获取登录的Token，当然也可以是普通用户的角色

``` shell
# 获取admin用户的Token
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)

# 获取普通用户的Token
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-viewer | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)
```

> 注意：Mac OS的base64指令有差异，`base64 -d` 更改为 `base64 -D`

- 浏览器打开地址: `http://192.168.0.132:32567`，访问主监控界面，并粘贴上一步的屏幕显示输入Token

{% asset_img dashboard.png %}

---

## Step 6: Worker节点部署（可选）

- 每一个Worker节点都需要完成步骤1-3，设置操作系统，并安装docker、kubertnetes等软件包
  
- 在Worker节点上，配置 APISERVER 的地址信息，实际上就是在`/etc/hosts`中增加如下域名:
  
``` txt
192.168.0.132    master1
```

- 在Master节点上，获得允许Worker节点加入的Token信息，有两种方法：
  
  方式1: 初始化第一个 master 节点时的输出内容中，第25、26行就是用来初始化 worker 节点的命令。该 token 的有效时间为 2 个小时，2小时内可以使用此 token 初始化任意数量的 worker 节点。

  方式2: 如果Master节点的初始化Token已经失效，则使用命令`kubeadm token create --print-join-command`，输出内容就包含该Token和指令样板

- 在Worker节点上，按照上一步骤中Master节点提供的信息，粘贴并执行以下类似的命令

``` sh
kubeadm join master1:6443 --token c4y5zm.rarmxapvrozfslcr \
    -discovery-token-ca-cert-hash sha256:a9e20cf7f87c372b1ed7fe0f43e622b3e62154ff9e1e63312c110b4102417399 \
    --control-plane --certificate-key 27e02e3b6744beccd16aa878891e074aa7ae45b430848fdfc924a9480200de13
```

- 如果Worker节点加入集群重，在Dashboard上将可以看到新的Node节点

    如果失败，请执行`kubeadm reset`重新初始化，并分析错误原因再次尝试

- 初始化第一个 master 节点时的输出内容中，第25、26行就是用来初始化 worker 节点的命令，如下所示：此时请不要执行该命令

## Step 7. 移除Worker节点（参考）

1. 在准备移除的 worker 节点上，执行`kubeadm reset`

2. 在第一个 master 节点上，执行`kubectl delete node worker1`。

    worker节点的名字可执行 `kubectl get nodes` 命令获得

## Step 8. 移除Master节点（参考）

1. 在准备移除的 Master 节点上，执行`kubeadm reset`

2. 如果想要再次启动Master节点，重复Step4、Step5即可

---

## 问题记录

### 1. 关于Docker的Cgroup Driver驱动程序

Cgroups是control groups的缩写，是Linux内核提供的一种可以限制、记录、隔离进程组（process groups）所使用的物理资源（如：cpu,memory,IO等等）的机制。最初由google的工程师提出，后来被整合进Linux内核。Cgroups也是LXC为实现虚拟化所使用的资源管理手段，可以说没有cgroups就没有LXC。

要理解 systemd 与 cgroups 的关系，我们需要先区分 cgroups 的两个方面：层级结构(A)和资源控制(B)。首先 cgroups 是以层级结构组织并标识进程的一种方式，同时它也是在该层级结构上执行资源限制的一种方式。我们简单的把 cgroups 的层级结构称为 A，把 cgrpups 的资源控制能力称为 B。
对于 systemd 来说，A 是必须的，如果没有 A，systemd 将不能很好的工作。而 B 则是可选的，如果你不需要对资源进行控制，那么在编译 Linux 内核时完全可以去掉 B 相关的编译选项。

docker 17.03使用的Cgroup Driver为`cgroupfs`，而kubelet 1.8.1 使用的cgroup driver为`systemd`,这里需要修改docker的Cgroup Driver，和kubelet 保持一致！

### 2. 关于kubelet启动报错

在使用systemctl启动kubelet以后，其日志显示错误信息：`Process: 11794 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255)`

问题分析：该错误说明kubelet尚未启动，原因是Master节点尚未完成初始化

解决办法：不管他！等Master节点init完成后，自然就恢复正常了。

### 3. Master节点启动 kubeadm init 失败

本次安装默认K8s的版本是`1.18.4`，报错信息显示时无法正常拉取默认镜像版本。

解决办法是：强制修改K8s的版本号为`1.18.2`，结果下载成功!!!
