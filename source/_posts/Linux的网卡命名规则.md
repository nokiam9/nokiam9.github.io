---
title: Linux的网卡命名规则
date: 2020-06-25 23:51:03
tags:
---

## 网卡命名规则

- Scheme 1: 如果从BIOS中能够取到可用的，板载网卡的索引号，则使用这个索引号命名，例如: eno1，如不能则尝试Scheme 2
- Scheme 2: 如果从BIOS中能够取到可以用的，网卡所在的PCI-E热插拔插槽(注：pci槽位号)的索引号，则使用这个索引号命名，例如: ens1，如不能则尝试Scheme 3
- Scheme 3：如果能拿到设备所连接的物理位置（PCI总线号+槽位号？）信息，则使用这个信息命名，例如:enp2s0，如不能则尝试Scheme 5
- Scheme 5：传统的kernel命名方法，例如: eth0，这种命名方法的结果不可预知的，即可能第二块网卡对应eth0，第一块网卡对应eth1。
- Scheme 4 使用网卡的MAC地址来命名，这个方法一般不使用。

## 示例解析

以`enp2s0`为例，分解规则为：`en2-s0`

前两个字符的含义:

- en 以太网 Ethernet
- wl 无线局域网 WLAN
- ww 无线广域网 WWLAN

第三个字符根据设备类型来选择

- o 集成设备索引号
- s 扩展槽的索引号
- x s 基于MAC进行命名
- p s PCI扩展总线

biosdevname和net.ifnames两种命名规范
net.ifnames的命名规范为: 设备类型+设备位置+数字

## 更多示例

eno1 板载网卡

enp0s2 pci网卡

ens33 pci网卡

wlp3s0 PCI无线网卡

wwp0s29f7u2i2 4G modem

wlp0s2f1u4u1 连接在USB Hub上的无线网卡

enx78e7d1ea46da pci网卡
