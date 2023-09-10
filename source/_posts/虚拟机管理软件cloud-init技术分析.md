---
title: 虚拟机管理软件cloud-init技术分析
date: 2023-09-10 17:05:26
tags:
---

## 一、概述

cloud-init 最初由 ubuntu 的母公司 Canonical 开发，基本设计思路是：

- 当用户首次创建虚拟机时，将前台设置的主机名，密码或者秘钥等存入后台的元数据服务器 - metadata server
- 当 cloud-init 随虚拟机启动而运行时，通过 http 协议访问 metadata server
- 虚拟机根据预设的元数据信息修改主机配置，从而完成系统的环境初始化

目前大部分公有云（openstack, AWS, Aliyun）都在使用 cloud-init , 已经成为事实的工业标准。
开源代码： [https://github.com/cloud-init/cloud-init](https://github.com/cloud-init/cloud-init)
官方文档： [https://cloudinit.readthedocs.io/en/latest/](https://cloudinit.readthedocs.io/en/latest/)

cloud-init 基于 Python 开发，可以通过`yum install cloud-init`进行安装，此外还有2个辅助软件包，`cloud-utils-growpart`用于磁盘空间自动扩容，`cloud-utils`用于镜像格式转换等。

- 程序入口：`/usr/bin/cloud-init`
- 代码安装目录：`/usr/lib/python2.7/site-packages/cloudinit/`
- 配置文件目录：`/etc/cloud/`，其中：主配置文件是`/etc/cloud/cloud.cfg`
- 日志文件位于：`/var/log/cloud-init.log`
- 缓存数据位于：`/var/lib/cloud/`
- systemd 系统服务目录仍然是`/usr/lib/systemd/system/`，包含了 cloud-init-local、cloud-init、cloud-config、cloud-final 等四个服务

本文以 centos7.8 + cloud-init 19.4 为例，分析其工作原理和实现方式。

> cloud-init 版本从 0.7.9 突变为 17.1，最新版本为 23.3.1，测试版本为 19.4.0
> 早期版本基于 Python2.7 开发，后来改为 Python3，因此代码安装目录可能有变化

## 三、主配置文件

为了实现 instance 定制工作，cloud-init 会主要按 4 个阶段执行任务（事实上还有一个generator阶段只有在systemd管理下才会触发，这里不做叙述）：

- local stage：寻找本地的data source， 并配置本机网络，以便后续获取user data等信息。
  网络配置可以来源于本地的data source， 如果获取不到，会启用dhcp。
  当然，如果在`/etc/cloud/cloud.cfg`定义 `network: {config: disabled}`，将放弃配置网络
- init stage：
- config stage：
- final stage：


```yaml
users:
 - default

disable_root: 1                             # 默认不允许root登录，一般需修改！
ssh_pwauth:   0                             # 默认不允许输入口令，一般需修改！

mount_default_fields: [~, ~, 'auto', 'defaults,nofail,x-systemd.requires=cloud-init.service', '0', '2']
resize_rootfs_tmp: /dev
ssh_deletekeys:   1
ssh_genkeytypes:  ~
syslog_fix_perms: ~
disable_vmware_customization: false

cloud_init_modules:                         # 定义init阶段需要执行的模块
 - disk_setup
 - migrator                                 # 迁移老的cloud-init数据
 - bootcmd                                  # 启动时执行相关命令
 - write-files                              # 根据cloud.cfg的配置写数据到文件里
 - growpart                                 # 扩展分区到硬盘的大小 ，默认对根分区执行。需要调用 growpart ！
 - resizefs                                 # resize文件系统，适配新的大小。默认对根目录执行
 - set_hostname                             # 根据元数据设置主机名
 - update_hostname                          # 更新主机名，适用于当用户自定义主机名时
 - update_etc_hosts                         # 更新 /etc/hosts
 - rsyslog
 - users-groups                             # 根据cloud.cfg的配置创建用户组和用户
 - ssh                                      # 配置sshd

cloud_config_modules:
 - mounts                                   # 加载自定义的磁盘，/etc/fstab ？
 - locale                                   # 设置语言
 - set-passwords
 - rh_subscription
 - yum-add-repo                             # 添加自定义 YUM 源，好像只能加1个？
 - package-update-upgrade-install           # 安装完成后自动升级软件包，一般需关闭！
 - timezone                                 # 设置时区
 - puppet
 - chef
 - salt-minion
 - mcollective
 - disable-ec2-metadata
 - runcmd                                   # 执行自定义的命令行

cloud_final_modules:
 - rightscale_userdata
 - scripts-per-once
 - scripts-per-boot
 - scripts-per-instance
 - scripts-user
 - ssh-authkey-fingerprints
 - keys-to-console
 - phone-home
 - final-message
 - power-state-change

system_info:
  default_user:                             # 自动创建默认用户，可修改！
    name: centos
    lock_passwd: true
    gecos: Cloud User
    groups: [adm, systemd-journal]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
  distro: rhel                              # Linux发行版类型是 redhat
  paths:
    cloud_dir: /var/lib/cloud               # 缓存数据目录
    templates_dir: /etc/cloud/templates     # 模版文件目录
  ssh_svcname: sshd
```

## 二、个人化配置的元数据

### 1. 前台界面的配置信息

以 PVE 为例，前台界面配置了 VM 511。
![PVE](pve-cloudinit.png)

配置信息存储在`/etc/pve/qemu-server/511.conf`，内容是：

```yaml
agent: 1
bootdisk: scsi0
cores: 1
ide2: local-lvm:vm-511-cloudinit,media=cdrom
ipconfig0: ip=192.168.0.223/24,gw=192.168.0.8
memory: 1024
name: cloud-init
nameserver: 192.168.0.144
net0: virtio=DA:41:A4:42:9C:DB,bridge=vmbr0,firewall=1
numa: 0
ostype: l26
scsi0: local-lvm:vm-511-disk-0,size=10G
scsihw: virtio-scsi-pci
searchdomain: caogo.local3
smbios1: uuid=5584c7ed-76f4-4b6d-a6fe-cb10a8b220fc
sockets: 1
vmgenid: 297abc09-fa0f-46b0-b9ad-ae3195110f2a
```

### 2. cloud-init CDROM 设备

根据前台的 VM 配置信息，PVE 控制台将其转化为 cloud-init 配置文件，并存储在 CDROM 设备中。
cloud-init 支持四种配置文件，其中：

- `meta-data`（必须）: 一般是 instance id，唯一的一个机器标识符
- `network-config`: 关于网络如 ip、nameserver、dns等的定义
- `user-data`（可选）: 我们定义的大多数配置都放在这里
- `vendor-data`（可选）: 是供应商（云）定义的如 user-data 类似数据，如果定义为 NoCloud 时该文件不存在


- Cloud metadata
- User data (optional)
- Vendor data (optional)



### 元数据配置文件：meta-data

```yml
# 定义了一个实例instance，并设置了唯一ID
instance-id: 5be815eb6375dae4bfdd91471def3f8415521f1d
```

### 网络配置文件：network-config

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

### 用户数据文件：user-data

```yaml
# 定义了大多数配置信息
hostname: cloud-init
manage_etc_hosts: true
# 完全合格域名 FQDN (Fully Qualified Domain Name)是一个包含了主机名和域名的完整标识符
fqdn: cloud-init.caogo.local3
chpasswd:
  expire: False
users:
  - default
package_upgrade: true
```

---

## 参考文献

- [cloud-init 源码解读](http://pythontime.iswbm.com/en/latest/c08/c08_06.html#centos-6-x)
- [cloud-init 介绍](https://xixiliguo.github.io/linux/cloud-init.html)
- [基于 Cloud-init 定制化 PVE 虚拟机](https://gameapp.club/post/2022-07-30-custom-cloud-init-for-pve/)
- [深度解析 OpenStack metadata 服务架构](https://zhuanlan.zhihu.com/p/55078689)
- [CentOS 6.x 如何更改网卡名](https://www.alteeve.com/w/Changing_the_ethX_to_Ethernet_Device_Mapping_in_EL6)