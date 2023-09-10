---
title: 虚拟机管理软件cloud-init技术分析
date: 2023-09-10 17:05:26
tags:
---

## 一、概述

cloud-init 最初由 ubuntu 的母公司 Canonical 开发，基本设计思路是：

1. 当用户首次创建虚拟机时，将前台设置的主机名，密码或者秘钥等存入后台的元数据服务器 - metadata server
2. 当 cloud-init 随虚拟机启动而运行时，通过 http 协议访问 metadata server，获取这些信息并修改主机配置，从而完成系统的环境初始化。

目前大部分公有云（openstack, AWS, Aliyun）都在使用 cloud-init , 已经成为虚拟机元数据管理和OS系统配置初始化的事实标准。本文以 centos7.8 + cloud-init 19.4 为例，分析其工作原理和实现方式。
如下是一些cloud-init的关键信息：
开源代码： [https://github.com/cloud-init/cloud-init](https://github.com/cloud-init/cloud-init)
官方文档： [https://cloudinit.readthedocs.io/en/latest/](https://cloudinit.readthedocs.io/en/latest/)

配置文件： `/etc/cloud/cloud.cfg`
日志：`/var/log/cloud-init.log`
存放关键数据的目录：`/var/lib/cloud/`

## 二、个人化配置的元数据

云实例根据磁盘映像和实例数据初始化，cloud-init 元数据被虚拟化为一个 CDROM 设备，根目录下有三个配置文件。

- Cloud metadata
- User data (optional)
- Vendor data (optional)

### 元数据配置：meta-data

```yml
instance-id: 5be815eb6375dae4bfdd91471def3f8415521f1d
```

### 网络配置：network-config

```yaml
version: 1
config:
    - type: physical
      name: eth0
      mac_address: 'da:41:a4:42:9c:db'
      subnets:
      - type: static
        address: '192.168.0.223'
        netmask: '255.255.255.0'
        gateway: '192.168.0.8'
    - type: nameserver
      address:
      - '192.168.0.144'
      search:
      - 'caogo.local3'
```

### 用户数据：user-data

```yaml
hostname: cloud-init
manage_etc_hosts: true
fqdn: cloud-init.caogo.local3
chpasswd:
  expire: False
users:
  - default
package_upgrade: true
```

---

## 参考文献

- [cloud-init介绍](https://xixiliguo.github.io/linux/cloud-init.html)
- [深度解析 OpenStack metadata 服务架构](https://zhuanlan.zhihu.com/p/55078689)
- 