---
title: BCLinux 8.2 安装记录
date: 2023-08-05 22:35:18
tags:
---

## 一、准备工作

苏研的软件源站点是：[https://mirrors.cmecloud.cn](https://mirrors.cmecloud.cn)，提供了一些基础的软件，包括BCLinux。
BC-Linux的镜像地址为：[https://mirrors.cmecloud.cn/bclinux/](https://mirrors.cmecloud.cn/bclinux/)

目前的详细版本情况如下：
![版本计划](LTS.png)

- V7: 基于 Centos 7，目前包含 v7.8，仅提供x86架构
- V8: 基于 Centos 8，目前包含 v8.2、v8.2、v8.6，仅提供 x86 架构
- oe1: 基于 OpenEular 20.12，仅提供ARM64架构
- oe21.10: 基于 OpenEular 21.10 和 21.10U3，提供 x86 和 ARM64 架构
- oe22.10: 基于 OpenEular 22.10 和 22.10U1，提供 x86 和 ARM64 架构

当前测试的版本是 oe21.10，

> `mirrors.bclinux.org`是同源域名，yum repolist 内部是这个域名

## 二、安装步骤

1. 卸载BCLinux的软件许可，否则执行 YUM 等命令时经常要求购买License

    ```bash
    rpm -evh `rpm -qa |grep bclinux-license`
    ```

    获得如下输出信息：

    ``` console
    Preparing...                          ################################# [100%]
    Cleaning up / removing...
    1:bclinux-license-manager-4.0-1.el8################################# [100%]
    ```

2. cloudinit 的配置文件有所不同，默认用户从 centos 改为 anolis

    ``` config
    default_user:
        name: anolis
        lock_passwd: true
        gecos: Cloud User
        groups: [adm, systemd-journal]
        sudo: ["ALL=(ALL) NOPASSWD:ALL"]
        shell: /bin/bash
    ```

3. 安装完成后，默认网卡是`ens18`, 安装 cloudinit 后新增网卡 eth0，并被设置为默认。
    但是，如果关闭系统功能服务 NetworkManager，该网卡就无法获得 IP 地址，因此暂时先保留该系统服务。

``` bash
yum install -y acpid cloud-init cloud-utils-growpart qemu-guest-agent
systemctl enable acpid
systemctl start qemu-guest-agent
```

## 三、注意事项

1. 通过 cloudinit 制作 Centos 基准镜像时，关闭 NetworkManager 系统服务，但 OpenEular 基础镜像时无法驱动网卡，暂时先保留该服务。
    > 默认网卡现在是`ens18`，而非`eth0`

2. 当前 cloudinit 的版本是 19.4，配置文件`/etc/cloud/cloud.cfg`有修改，默认用户现在是`openeular`

3. PVE在安装虚拟机时会见到`Qemu GA`这个选项，是开启还是关闭呢？
    Qemu 代理即 qemu-guest-agent，是一个运行在虚拟机里面的程序 qemu-guest-agent是一个帮助程序，守护程序，它安装在虚拟机中，用于在主机和虚拟机之间交换信息，以及在虚拟机中执行命令。
    在Proxmox VE中，qemu代理主要用于两件事：
    - 正确关闭虚拟机，而不是依赖ACPI命令或Windows策略
    - 在进行备份时冻结来宾文件系统（在Windows上，使用卷影复制服务VSS）。
    如果想用，不仅需要在pve里开启这个选项，还需要手动安装

``` console
[root@MiWiFi-RA70-srv ~]# ls /etc/yum.repos.d
BCLinux.repo

[root@MiWiFi-RA70-srv ~]# more /etc/yum.repos.d/*
# BCLinux-release.repo
# 
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for BCLinux updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the 
# remarked out baseurl= line instead.
#
#
[baseos]
name=BC-Linux-release - baseos
baseurl=http://mirrors.bclinux.org/bclinux/oe21.10/OS/$basearch/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-BCLinux-For-Euler

[everything]
name=BC-Linux-release - everything
baseurl=http://mirrors.bclinux.org/bclinux/oe21.10/everything/$basearch/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-BCLinux-For-Euler


[update]
name=BC-Linux-release - update
baseurl=http://mirrors.bclinux.org/bclinux/oe21.10/update/$basearch/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-BCLinux-For-Euler

[extras]
name=BC-Linux-release - extras
baseurl=http://mirrors.bclinux.org/bclinux/oe21.10/extras/$basearch/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-BCLinux-For-Euler
```

cloudinit 的原始配置文件

``` txt
[root@MiWiFi-RA70-srv ~]# cat /etc/cloud/cloud.cfg
# The top level settings are used as module
# and system configuration.

# A set of users which may be applied and/or used by various modules
# when a 'default' entry is found it will reference the 'default_user'
# from the distro configuration specified below
users:
   - default

# If this is set, 'root' will not be able to ssh in and they
# will get a message to login instead as the default $user
disable_root: true

mount_default_fields: [~, ~, 'auto', 'defaults,nofail', '0', '2']
resize_rootfs_tmp: /dev
ssh_pwauth:   0

# This will cause the set+update hostname module to not operate (if true)
preserve_hostname: false

# Example datasource config
# datasource:
#    Ec2:
#      metadata_urls: [ 'blah.com' ]
#      timeout: 5 # (defaults to 50 seconds)
#      max_wait: 10 # (defaults to 120 seconds)

# The modules that run in the 'init' stage
cloud_init_modules:
 - migrator
 - seed_random
 - bootcmd
 - write-files
 - growpart
 - resizefs
 - disk_setup
 - mounts
 - set_hostname
 - update_hostname
 - update_etc_hosts
 - ca-certs
 - rsyslog
 - users-groups
 - ssh

# The modules that run in the 'config' stage
cloud_config_modules:
 - ssh-import-id
 - locale
 - set-passwords
 - spacewalk
 - yum-add-repo
 - ntp
 - timezone
 - disable-ec2-metadata
 - runcmd

# The modules that run in the 'final' stage
cloud_final_modules:
 - package-update-upgrade-install
 - puppet
 - chef
 - mcollective
 - salt-minion
 - rightscale_userdata
 - scripts-vendor
 - scripts-per-once
 - scripts-per-boot
 - scripts-per-instance
 - scripts-user
 - ssh-authkey-fingerprints
 - keys-to-console
 - phone-home
 - final-message
 - power-state-change

# System and/or distro specific settings
# (not accessible to handlers/transforms)
system_info:
   # This will affect which distro class gets used
   distro: openEuler
   # Default user name + that default users groups (if added/used)
   default_user:
     name: openEuler
     lock_passwd: True
     gecos: openEuler Cloud User
     groups: [wheel, adm, systemd-journal]
     sudo: ["ALL=(ALL) NOPASSWD:ALL"]
     shell: /bin/bash
   # Other config here will be given to the distro class and/or path classes
   paths:
      cloud_dir: /var/lib/cloud/
      templates_dir: /etc/cloud/templates/
   ssh_svcname: sshd
```

## 四、上层软件栈版本

- nginx: 1.16.1
- python3: 3.7.9
- docker: 18.09.0
- docker-compose: 1.22.0
- openjdk(java): 11.0.12
- nodejs: v12.22.11

``` bash
[root@Copy-of-VM-OpenEular21 yum.repos.d]# docker -v
Docker version 18.09.0, build a8959d5
[root@Copy-of-VM-OpenEular21 yum.repos.d]# docker-compose -v
docker-compose version 1.22.0, build f46880f
[root@Copy-of-VM-OpenEular21 yum.repos.d]# java -version
openjdk version "11.0.12" 2021-07-20
OpenJDK Runtime Environment Bisheng (build 11.0.12+9)
OpenJDK 64-Bit Server VM Bisheng (build 11.0.12+9, mixed mode, sharing)
```

---

## 参考文献

- [openEular - 官方下载](https://www.openeuler.org/zh/download/archive/)
- [BCLinux ECS 基线版本的构造分析](https://blog.csdn.net/bright69/article/details/126783599)
- [通过QEMU-GuestAgent实现从外部注入写文件到KVM虚拟机内部](https://cloud.tencent.com/developer/article/1987533)
- [PVE创建openEuler虚拟机模板](https://cloud.tencent.com/developer/article/2008066)
- [基于cloud-init定制虚拟机](https://gameapp.club/post/2022-07-30-custom-cloud-init-for-pve/)
