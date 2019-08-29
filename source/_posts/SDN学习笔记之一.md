---
title: SDN学习笔记之一
date: 2019-07-05 11:38:52
tags:
---

## 传统的三层网络架构

{% asset_img core-aggregate-access.png 传统的三层网络架构 %}

特点：

- 每个服务器两个物理网卡，直连到两个置顶交换机做物理高可用

- 汇聚层和接入层走二层交换，和核心层走三层路由

- 所有 OpenStack 网关配置在核心层路由器

- 防火墙和核心路由器直连，做一些安全策略


## 简化的大二层网络架构

{% asset_img spine-leaf.png 简化的大二层网络架构 %}

Spine-Leaf 是 full-mesh 连接，它可以带来如下几个好处：

- 转发路径更短。以图 7 的 Spine-Leaf（两级 Clos 架构）为例，任何两台服务器经过三跳（Leaf1 -> Spine -> Leaf2）就可以到达，延迟更低，并且可保障（可以按跳数精确计算出来）。

- 水平可扩展性更好，任何一层有带宽或性能瓶颈，只需新加一台设备，然后跟另一层的所有设备直连。

- 所有设备都是 active 的，一个设备挂掉之后，影响面比三层模型里挂掉一个设备小得多。

宿主机方面，我们升级到了 10G 和 25G 的网卡。

## SDN的控制平面和数据平面

- 数据平面基于 VxLAN，控制平面基于 MP-BGP EVPN 协议，在设备之间同步控制平面信息。 
- 网关是分布式的，每个 leaf 节点都是网关。
- VxLAN 和 MP-BGP EVPN 都是 RFC 标准协议，更多信息参考 [2]。
- VxLAN 的封装和解封装都在 leaf 完成，leaf 以下是 VLAN 网络，以上是 VxLAN 网络。
- 这套方案在物理上支持真正的租户隔离。

{% asset_img spine-leaf-2.png 大二层网络的整体架构 %}

图 9 是我们软件 + 硬件的网络拓扑：

1）以 leaf 为边界，leaf 以下是 underlay，走 VLAN；上面 overlay，走 VxLAN

2）underlay 由 neutron、OVS 和 neutron OVS agent 控制；overlay 是 CNC 控制

3）Neutron 和 CNC 之间通过 plugin 集成

## SDN控制器的两种模式

{% asset_img SDN控制器的两种模式.png SDN控制器的两种工作模式对比 %}

从控制器是否参与转发设备的的转发控制来看，当前主要有两种控制器类型:

### 弱控制模式

- 弱控制模式下，控制平面基于网络设备自学习，控制器不在转发平面，仅负责配 置下发，实现自动部署。主要解决网络虚拟化，提供适应应用的虚拟网络。

- 弱控制模式的优点是转发控制面下移，减轻和减少对控制器的依赖。

### 强控制模式

- 在强控制模式下，控制器负责整个网络的集中控制，体现SDN集中管理的优势。

- 基于openflow的强控制使得网络具备更多的灵活性和可编程性。除了能够给用 户提供适合应用需要的网络，还可以集成FW等提供安全方案;可以支持混合 Overlay 模型，通过控制器同步主机和拓扑信息, 将各种异构的转发模型同一处理;可以提供基于 openflow 的服务链功能对安全服务进行编排，可以提供更为灵活的网络诊断手段，如虚机仿真和雷达探测等。

用户可能会担心强控制模式下控制器全部故障对网络转发功能的影响，这个影响因素可以通过下述两点来降低和消除:
1. 通过控制器集群增加控制器可靠性，避免单点故障
2. 逃生机制:设备与所有控制器失联后，切换为自转模式，业务不受影响。

---

## 附录1:标准的openstack-provider-network

{% asset_img openstack-provider-network.png 标准的Openstack Provider Network的整体架构 %}


## 附录2:简化的openstack-provider-network

{% asset_img 简化的openstack-provider-network.png 简化的Openstack Provider Network的整体架构 %}
