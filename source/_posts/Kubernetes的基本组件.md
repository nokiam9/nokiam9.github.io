---
title: Kubernetes的基本组件
date: 2019-04-01 15:12:29
tags:
---

Kubernetes 是一个跨主机集群的 开源的容器调度平台，它可以自动化应用容器的部署、扩展和操作，提供以容器为中心的基础架构。
Kubernetes集群主要包括一个Master组件，和多个Node组件。

{% asset_img k8s-basic.png Kubernetes的技术架构 %}

# 一、Master组件
Master 组件提供的集群控制。Master 组件对集群做出全局性决策(例如：调度)，以及检测和响应集群事件(副本控制器的replicas字段不满足时,启动新的副本)。
Master 组件可以在集群中的任何节点上运行。然而，为了简单起见，设置脚本通常会启动同一个虚拟机上所有 Master 组件，并且不会在此虚拟机上运行用户容器。

## 1、核心数据库（etcd）
etcd 用于 Kubernetes 的后端存储，负责保存所有集群的状态信息。

## 2、接口服务器（kube-apiserver）
kube-apiserver对外暴露了Kubernetes API。它是的 Kubernetes 前端控制层。它被设计为水平扩展，即通过部署更多实例来缩放。请参阅构建高可用性群集.

## 3、运行控制器（kube-controller-manager）
kube-controller-manager运行控制器，它们是处理集群中常规任务的后台线程。逻辑上，每个控制器是一个单独的进程，但为了降低复杂性，它们都被编译成独立的可执行文件，并在单个进程中运行。
这些控制器包括:
- 节点控制器: 当节点移除时，负责注意和响应。
- 副本控制器: 负责维护系统中每个副本控制器对象正确数量的 Pod。
- 端点控制器: 填充 端点(Endpoints) 对象(即连接 Services & Pods)。
- 服务帐户和令牌控制器: 为新的命名空间创建默认帐户和 API 访问令牌.

## 4.任务调度器（kube-scheduler）
kube-scheduler负责Pod的调度，监视没有分配节点的新创建的Pod，按照预定的调度策略将Pod调度到相应的Node上。

# 二、Node组件
节点组件在每个节点上运行，维护运行的 Pod 并提供 Kubernetes 运行时环境。

## 1、kubelet
kubelet是主要的节点代理，它监测已分配给其节点的 Pod(通过 apiserver 或通过本地配置文件)，负责维护容器的生命周期（创建、启动、监控、重启、销毁等），同时也负责 Volume（CSI）和网络（CNI）的管理；同时与Master节点协作，实现集群管理的基本功能。
主要提供如下功能:
- 挂载 Pod 所需要的数据卷(Volume)。
- 下载 Pod 的 secrets。
- 通过 Docker 运行(或通过 rkt)运行 Pod 的容器。
- 周期性的对容器生命周期进行探测。
- 如果需要，通过创建 镜像 Pod（Mirror Pod） 将 Pod 的状态报告回系统的其余部分。
- 将节点的状态报告回系统的其余部分。

## 2、kube-proxy
kube-proxy负责通信和负载均衡，通过维护主机上的网络规则并执行连接转发，实现了Kubernetes服务抽象。

## 3、docker
Docker用于运行容器。也支持rkt运行容器等作为 Docker 的试验性替代方案。

# 三、addons插件
插件是实现集群功能的 Pod 和 Service。 Pods 可以通过 Deployments，ReplicationControllers 管理。
插件对象本身是受命名空间限制的，被创建于`kube-system`命名空间。
Addon管理器用于创建和维护附加资源.

## 1、域名解析服务（DNS）
虽然其他插件并不是必需的，但所有 Kubernetes 集群都应该具有Cluster DNS，许多示例依赖于它。
Cluster DNS 是一个 DNS 服务器，和您部署环境中的其他 DNS 服务器一起工作，为 Kubernetes 服务提供DNS记录。
Kubernetes 启动的容器自动将 DNS 服务器包含在 DNS 搜索中。

## 2、监控界面（Dashboard）
dashboard 提供了集群状态的只读概述。有关更多信息，请参阅使用HTTP代理访问 Kubernetes API

## 3、外部负载均衡器（Ingress） 
Ingress Controller 为服务提供外网入口

## 4、容器资源监控（Heapster）
容器资源监控将关于容器的一些常见的时间序列度量值保存到一个集中的数据库中，并提供用于浏览这些数据的界面。

## 5、集群层面日志
集群层面日志 机制负责将容器的日志数据保存到一个集中的日志存储中，该存储能够提供搜索和浏览接口。

## 6、包管理器（Helm）
Helm这个东西其实早有耳闻，但是一直没有用在生产环境，而且现在对这货的评价也是褒贬不一。正好最近需要再次部署一套测试环境，对于单体服务，部署一套测试环境我相信还是非常快的，但是对于微服务架构的应用，要部署一套新的环境，就有点折磨人了，微服务越多、你就会越绝望的。虽然我们线上和测试环境已经都迁移到了kubernetes环境，但是每个微服务也得维护一套yaml文件，而且每个环境下的配置文件也不太一样，部署一套新的环境成本是真的很高。如果我们能使用类似于yum的工具来安装我们的应用的话是不是就很爽歪歪了啊？Helm就相当于kubernetes环境下的yum包管理工具。

---

# 附录一：Kubernetes的基础镜像列表
```
# 最基础的根容器
k8s.gcr.io/pause-amd64:3.1    

# etcd核心数据库
k8s.gcr.io/etcd-amd64:3.1.12   

# Master组件，当前稳定版本v1.10.11，也是Kubernetes的主版本号
k8s.gcr.io/kube-apiserver-amd64:v1.10.11
k8s.gcr.io/kube-controller-manager-amd64:v1.10.11
k8s.gcr.io/kube-scheduler-amd64:v1.10.11

# Node节点的通讯组件
k8s.gcr.io/kube-proxy-amd64:v1.10.11

# Node节点的Docker连接器
docker/kube-compose-controller:v0.4.12
docker/kube-compose-api-server:v0.4.12

# Kubernetes集群的DNS插件，当前稳定版本v1.1.14
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:1.14.8
k8s.gcr.io/k8s-dns-sidecar-amd64:1.14.8
k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.8

# Kubernetes集群的Dashboard插件，当前稳定版本v1.8.3
k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
```
---

# 附录二：云控制器管理器-(cloud-controller-manager)
cloud-controller-manager 是用于与底层云提供商交互的控制器。云控制器管理器可执行组件是 Kubernetes v1.6 版本中引入的 Alpha 功能。
cloud-controller-manager 仅运行云提供商特定的控制器循环。您必须在 kube-controller-manager 中禁用这些控制器循环，您可以通过在启动 kube-controller-manager 时将 --cloud-provider 标志设置为external来禁用控制器循环。
cloud-controller-manager 允许云供应商代码和 Kubernetes 核心彼此独立发展，在以前的版本中，Kubernetes 核心代码依赖于云提供商特定的功能代码。在未来的版本中，云供应商的特定代码应由云供应商自己维护，并与运行 Kubernetes 的云控制器管理器相关联。
以下控制器具有云提供商依赖关系:
- 节点控制器: 用于检查云提供商以确定节点是否在云中停止响应后被删除
- 路由控制器: 用于在底层云基础架构中设置路由
- 服务控制器: 用于创建，更新和删除云提供商负载平衡器
- 数据卷控制器: 用于创建，附加和装载卷，并与云提供商进行交互以协调卷