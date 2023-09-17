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
- 核心配置文件：`/etc/cloud/cloud.cfg`
- 日志文件位于：`/var/log/cloud-init.log`
- 模版文件目录：`/etc/cloud/templates/`
- 缓存数据位于：`/var/lib/cloud/`
- 系统服务目录：`/usr/lib/systemd/system/`

本文以 centos7.8 + cloud-init 19.4 为例，分析其工作原理和实现方式。

> cloud-init 版本从 0.7.9 突变为 17.1，最新版本为 23.3.1，测试版本为 19.4.0
> 早期版本基于 Python2.7 开发，后来改为 Python3，因此代码安装目录可能有变化

## 二、工作原理

cloud-init 对系统的初始化分为四个阶段，分别是：local、init、config、final。
一般通过 systemd 进行管理，通过命令`systemctl list-unit-files |grep cloud`，可以列出四个阶段对应的 cloud-init-local、cloud-init、cloud-config、cloud-final 等4个 service 文件。

```console
cloud-config.service                          enabled 
cloud-final.service                           enabled 
cloud-init-local.service                      enabled 
cloud-init.service                            enabled 
cloud-config.target                           static  
cloud-init.target                             static 
```

> .target 是静态定义，一般只有描述服务之间依赖关系的 Unit 段，不包含执行命令

systemd 服务启动时会有一个简单的 generator stage，其任务是检查系统状态检查，并读取主配置文件`cloud.cfg`。
如果满足以下情况，cloud-init 将不在开机时启动，本组内其他 service 也遵循该规则：

- `/etc/cloud/cloud-init.disabled` 文件存在时
- 当内核命令发现文件 `/proc/cmdline`包含 `cloud-init=disabled`时

> 当在容器中运行时，内核命令可能会被忽略，但是 cloud-init 会读取`KERNEL_CMDLINE`环境变量

### 1. Local Stage

作为虚拟机实例启动 cloud-init 的第一个阶段，其任务是：查找**本地**数据源，并应用于网络配置！
分析 systemd 配置文件`cloud-init-local.service`，发现：

- 依赖于 systemd-remount-fs.service，即需要加载 root 文件系统
- 检查是否存在缓存文件目录`/var/lib/cloud`
- 核心执行代码：`/usr/bin/cloud-init init --local`

所谓本地数据源，有以下几种方式：

- datasource：本机的config drive（例如 PVE 的Cloud-init CDROM），或者 Openstack、EC2 提供的云网络配置
- fallback：默认方式，相当于`dhcp on eth0`，即直接通过 DHCP 服务获得网络配置信息
- none：禁用网络。可以通过在`/etc/cloud/cloud.cfg`中，添加内容`network: {config: disabled}`实现

> 所支持的数据源定义位于：`/usr/lib/python2.7/site-packages/cloudinit/settings.py`中的变量`CFG_BUILTIN.datasource_list`

如果是该实例的第一次启动，那么被选中的网络配置会被应用，所有老旧的配置都会会清除。
该阶段需要阻止网络服务启动以及老的配置被应用，这可能带来一些负面的影响，比如 DHCP 服务挂起，或者已经广播了老的 hostname，这可能导致系统进入一个奇怪的状态需要重启网络设备。

### 2. Init Stage

在官方文档中，也称为 Network Stage。
分析 systemd 配置文件`cloud-init.service`，发现：

- 依赖于 cloud-init-local.service 和 NetworkManager.service
- 核心执行代码：`/usr/bin/cloud-init init`

此阶段运行核心配置文件中名为`cloud_init_modules下`的所有module，主要包括：

- 文件系统配置：包括 disk_setup 、resizefs、growpart 等
  由于 nfs 等依赖于网络配置，这些模块不能过早启动。
- 主机网络配置：包括 set_hostname、update_hostname、update_etc_hosts等
- 辅助功能实现：包括 migrator、bootcmd、write-files、rsyslog、user-groups、ssh等

### 3. Config Stage

分析 systemd 配置文件`cloud-config.service`和`cloud-config.target`，发现：

- 依赖于 cloud-init-local.service 和 cloud-init.service
- 核心执行代码：`/usr/bin/cloud-init modules --mode=config`

此阶段运行核心配置文件中名为`cloud_config_modules下`的所有module，主要包括：

- 文件系统挂载：包括 mounts 等
- 主机环境配置：包括 locale、timezone、set-password、rh_subscription等
- 系统软件处理：包括 yum-add-repo、package-update-upgrade-install
- 辅助功能实现：包括 runcmd、puppet等

### 4. Final Stage

分析 systemd 配置文件`cloud-final.service`，发现：

- 依赖于 cloud-config.service
- 核心执行代码：`/usr/bin/cloud-init modules --mode=final`

此阶段运行核心配置文件中名为`cloud_final_modules下`的所有module，主要包括：

- 用户脚本处理：包括 scripts-per-once、scripts-per-boot、scripts-per-instance、scripts-user 等
- 用户登录配置：包括 ssh-authkey-fingerprints、keys-to-console等
- 系统软件处理：包括 yum-add-repo、package-update-upgrade-install
- 辅助功能实现：包括 final-message、phone-home等

### 5. cloud.cfg 示例

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

官方文档提供了配置文件的自定义规则方法，参见[Cloud config examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html)

## 三、PVE 的 config drive 实现

PVE 的虚拟机模版通过增加一个专用的 Cloud-init CDROM 设备实现 config drive。

例如，我们从前台界面配置了 一个虚拟机 VM 511。
![PVE](pve-cloudinit.png)

第一步，将配置信息存储在`/etc/pve/qemu-server/511.conf`，内容是：

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

第二步，PVE 控制台将 VM 配置信息转化为 cloud-init 配置文件，并存储在 CDROM 设备中。

```console
[root@copy-of-vm-centos7 ~]# mount /dev/sr0 /mnt
mount: /dev/sr0 写保护，将以只读方式挂载
[root@copy-of-vm-centos7 ~]# ls -l /mnt
总用量 2
-rw-r--r--. 1 root root  54 9月  17 12:08 meta-data
-rw-r--r--. 1 root root 227 9月  17 12:08 network-config
-rw-r--r--. 1 root root 170 9月  17 12:08 user-data
```

第三步，虚拟机启动 cloud-init 时，将读取`/dev/sr0`的全部文件并加载到缓存。

cloud-init 支持四种配置文件，其中：

- `meta-data`（必须）: 一般是 instance id，唯一的一个机器标识符
- `network-config`: 关于网络如 ip、nameserver、dns等的定义
- `user-data`（可选）: 我们定义的大多数配置都放在这里
- `vendor-data`（可选）: 是供应商（云）定义的如 user-data 类似数据，如果定义为 NoCloud 时该文件不存在

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

## 四、疑难杂症

### 1. cloud-init 与 NetworkManager 的关系

通过源代码分析，在 Local Stage 从 datasource 里读取网络配置信息，处理逻辑是：

- 如果发现使用的是静态地址，cloud-init 就会将 datasource 定义的配置信息写入`/etc/network/interfaces`目录下的配置文件
- 如果发现使用的是 DHCP，cloud-init 并不会创建刷新网卡配置文件，配置ip的工作就交由 NetworkManager 自动获取

从以信息可知，如果创建静态ip的虚拟机，NetworkManager 这个服务必须在 cloudinit-local 之后启动才可正常从配置文件中读取 ip 并配置。而当你在镜像里安装 NetworkManager后，默认情况下它的启动顺序是会在 cloudinit-local 之前的。

### 2. openEuler 的网卡名称被修改

openEuler 初始化安装时，NetworkManager 采用的是 net.ifnames 命名规范，初始网卡被命名为`ens18`。
安装 cloud-init 之后，其强制改为 biosdevname 命名规范，因此默认网卡被改名为`eth0`。
注意，老的网卡配置文件依然存在！具体实现原理参见[CentOS 6.x 如何更改网卡名](http://pythontime.iswbm.com/en/latest/c08/c08_06.html#centos-6-x)

### 3. BCLinux oe21.10 安装 cloud-init 后启动失败，提示错误信息`hosts.redhat`文件不存在

BCLinux oe21.10 提供的 cloud-init 版本，并未建立模版目录`/etc/cloud/templates`，也找不到`hosts.redhat.tmpl`文件。
根本原因是 cloud-init 目前适配了 Centos 和 openEuler 等主流版本，配置信息在 distor 中，但 BCLinux 不再其中。
临时解决办法是，手工建模版目录，并拷贝相应的模版文件，凑合着使吧！！！

---

## 参考文献

- [cloud-init 源码解读](http://pythontime.iswbm.com/en/latest/c08/c08_06.html#)
- [cloud-init 介绍](https://xixiliguo.github.io/linux/cloud-init.html)
- [Cloud-init 初始化虚拟机配置](https://einverne.github.io/post/2020/03/cloud-init.html)
- [基于 Cloud-init 定制化 PVE 虚拟机](https://gameapp.club/post/2022-07-30-custom-cloud-init-for-pve/)
- [深度解析 OpenStack metadata 服务架构](https://zhuanlan.zhihu.com/p/55078689)
