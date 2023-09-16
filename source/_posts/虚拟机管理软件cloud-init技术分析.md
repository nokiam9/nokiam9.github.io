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

## 二、启动流程

cloud-init 对系统的初始化分为四个阶段，分别是：local、init、config、final。
通过命令`systemctl list-unit-files |grep cloud`，可以列出四个阶段对应的 unit 文件。

```console
cloud-config.service                          enabled 
cloud-final.service                           enabled 
cloud-init-local.service                      enabled 
cloud-init.service                            enabled 
cloud-config.target                           static  
cloud-init.target                             static 
```

> .target 是静态定义，一般只有描述服务之间依赖关系的 Unit 段，不包含执行命令

在采用 systemd 系统服务时，启动时会有一个简单的系统状态检查，也被称为 generator stage。
如果满足以下情况，cloud-init 将不在开机时启动：

- `/etc/cloud/cloud-init.disabled` 文件存在时
- 当内核命令发现文件 `/proc/cmdline`包含 `cloud-init=disabled`时

> 当在容器中运行时，内核命令可能会被忽略，但是 cloud-init 会读取`KERNEL_CMDLINE`环境变量

### 1. Local Stage

分析 systemd 配置文件`/lib/systemd/system/cloud-init-local.service`，发现：

- 依赖于 systemd-remount-fs.service，即需要加载 root 文件系统
- 检查是否存在缓存文件目录`/var/lib/cloud`
- 如果存在状态文件`/etc/cloud/cloud-init.disabled`，则禁止启动
- 核心执行代码：`/usr/bin/cloud-init init --local`

Local Stage 作为虚拟机实例启动 cloud-init 的第一个阶段，核心任务就是：查找**本地**数据源，并应用于网络配置！
所谓本地数据源，有以下几种方式：

- datasource：本机的config drive（例如 PVE 的Cloud-init CDROM），或者 Openstack、EC2 提供的云网络配置
- fallback：默认方式，相当于`dhcp on eth0`，即直接通过 DHCP 服务获得网络配置信息
- none：禁用网络。可以通过在`/etc/cloud/cloud.cfg`中，添加内容`network: {config: disabled}`实现

> 所支持的数据源定义位于：`/usr/lib/python2.7/site-packages/cloudinit/settings.py`中的变量`CFG_BUILTIN.datasource_list`

如果是该实例的第一次启动，那么被选中的网络配置会被应用，所有老旧的配置都会会清除。
该阶段需要阻止网络服务启动以及老的配置被应用，这可能带来一些负面的影响，比如 DHCP 服务挂起，或者已经广播了老的 hostname，这可能导致系统进入一个奇怪的状态需要重启网络设备。

#### 与 NetworkManager 的关系

通过源代码分析，在 Local Stage 从 datasource 里读取网络配置信息，处理逻辑是：

- 如果发现使用的是静态地址，cloud-init 就会将 datasource 定义的配置信息写入`/etc/network/interfaces`目录下的配置文件
- 如果发现使用的是 DHCP，cloud-init 并不会创建刷新网卡配置文件，配置ip的工作就交由 NetworkManager 自动获取

从以信息可知，如果创建静态ip的虚拟机，NetworkManager 这个服务必须在 cloudinit-local 之后启动才可正常从配置文件中读取 ip 并配置。而当你在镜像里安装 NetworkManager后，默认情况下它的启动顺序是会在 cloudinit-local 之前的。

> openEuler 的初始网卡是`ens18`，在安装 cloud-init 之后，将被强制改名为`eth0`

#### PVE 的 config drive 网络配置

PVE 的虚拟机模版可以增加一个专用的 Cloud-init CDROM 设备，虚拟机启动 cloud-init 时，将读取`/dev/sr0`的全部文件至缓存，其中
`network-config`就是网络配置文件。

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

disable_root: 1                             # 禁止root登录，默认true，一般需修改！
ssh_pwauth:   0                             # 允许密码登录，默认false，一般需修改！

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
- [Cloud-init 初始化虚拟机配置](https://einverne.github.io/post/2020/03/cloud-init.html)
- [基于 Cloud-init 定制化 PVE 虚拟机](https://gameapp.club/post/2022-07-30-custom-cloud-init-for-pve/)
- [深度解析 OpenStack metadata 服务架构](https://zhuanlan.zhihu.com/p/55078689)
- [CentOS 6.x 如何更改网卡名](https://www.alteeve.com/w/Changing_the_ethX_to_Ethernet_Device_Mapping_in_EL6)