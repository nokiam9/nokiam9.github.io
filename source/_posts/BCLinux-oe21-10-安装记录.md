---
title: BCLinux oe21.10 安装记录
date: 2023-08-17 23:47:40
tags:
---

## 一、概述

移动云的软件源站点是：[https://mirrors.cmecloud.cn](https://mirrors.cmecloud.cn)，提供了一些基础的软件，包括BCLinux。
BCLinux的镜像地址为：[https://mirrors.cmecloud.cn/bclinux/](https://mirrors.cmecloud.cn/bclinux/)
> `mirrors.bclinux.org`是同源域名，yum repolist 内部是这个域名

### 1. 版本规划

作为Linux的发行版，BCLinux 有两个完全不同的技术路线，早期的版本是基于 Centos 定制化，包括：

- V7: 基于 Centos 7，目前包含 v7.8，仅提供x86架构
- V8: 基于 Centos 8，目前包含 v8.2、v8.2、v8.6，仅提供 x86 架构

随着 Centos 停服日期的迫近，以及自主可控操作系统的要求，BCLinux 转向了 OpenEuler，包括：

- oe1: 基于 OpenEuler 20.12，仅提供ARM64架构
- oe21.10: 基于 OpenEuler 21.10 和 21.10U3，提供 x86 和 ARM64 架构
- oe22.10: 基于 OpenEuler 22.10 和 22.10U1，提供 x86 和 ARM64 架构

目前 BCLinux 的版本生命周期的规划如下：
![版本计划](LTS.png)

### 2. 与 open Euler 的关系

尽管目前的 BCLinux 是基于 [openEuler](https://www.openeuler.org/zh/) 的定制化版本，但也存在明显的差异。
![openEuler](openEuler.png)

- BCLinux oe22.10 和 oe22.10 都是基于 OpenEuler 的非公开发行版本
- BCLinux 仅提供 x86_64 和 AArch64 架构，但 openEuler 还提供了 ARM32、RISC-V、LoongArch64、Power 和 SW64 架构。

## 二、安装方法

本次测试的基准版本是 BCLinux oe21.10 的 x86-64 架构。

### 1. 准备工作

1. 通过官方站点[https://mirrors.cmecloud.cn/bclinux/oe21.10/ISO/x86_64/release/](https://mirrors.cmecloud.cn/bclinux/oe21.10/ISO/x86_64/release/)下载ISO安装盘，有基础版本和全量版本，以及两个后续的补丁版本。
2. 新建一个虚拟机，挂载ISO安装盘，建议内存至少 1GB，硬盘至少 10GB。
   启动后，根据安装向导提示信息，选择中国上海时区，设置root密码（8位以上，至少3种类型字符）、硬盘分区默认即可，最小化安装方式。
3. 安装完成后启动虚拟机，如果 console 成功登录即为正常。

### 2. 环境设置

1. 虚拟机添加一个 cloudinit 类型的 CDROM 设备，用于后续管理个性化配置数据。
   此时，可以顺便卸载ISO安装盘的 CDROM 设备。
2. 卸载BCLinux的软件许可，否则执行 YUM 等命令时提示要求购买License。

    ```bash
    rpm -evh `rpm -qa |grep bclinux-license`
    ```

3. 默认安装后网卡尚未启用，需要手动调整。
   在`/etc/sysconfig/network-scripts/`目录中找到网卡配置文件`ifcfg-ens18`（注意不是常见的`eth0`），删除`UUID`，并设置`ONBOOT=yes`。

4. reboot重启虚拟机，此时通过`ip a`可以发现IP地址已启用，联网成功。
   > 建议此时将虚拟机转换为模版，再 clone 一个虚拟机用于后续安装，以避免误操作又来一次冗长的安装过程。

### 3. 基线版本配置

1. 系统软件安装

    ``` bash
    # 关闭Firewalld
    systemctl disable --now firewalld

    # 安装虚拟化软件
    yum install -y acpid cloud-init cloud-utils-growpart qemu-guest-agent
    systemctl enable acpid
    systemctl start qemu-guest-agent

    # 禁用zeroconf(零配置网络服务规范)，该协议目的是在系统无法连接DHCP服务的时候，尝试获取类似169.254.0.0的保留IP
    echo "NOZEROCONF=yes" >> /etc/sysconfig/network

    # 防止ssh连接使用dns导致访问过慢
    sed -ri '/UseDNS/{s@#@@;s@\s+.+@ no@}' /etc/ssh/sshd_config
    systemctl restart sshd
    ```

    - 系统安装已默认关闭 selinux， 但仍然启用 firewalld 系统服务
    - 系统安装已不再关闭 NetworkManager，因为 Centos8 已经不再使用 networkd.service 系统服务
    - cloud-init 是核心的虚拟机管理软件，acpid 用于控制虚拟机的电源设备以便宿主机执行关机命令，cloud-utils-growpart 用于调整虚拟机的分区设置，qemu-guest-agent 用于虚拟机接受宿主机的命令并反馈结果，后续PVE管理界面可以直接显示IP地址信息。
    - BCLinux 已经预置了 git 、net-tools 等常用工具，但没有yum-utils软件包，而是自带 dnf 管理器 yum-config-manager

2. 手工调整cloudinit配置文件`/etc/cloud/cloud.cfg`
   当前cloud-init的版本是 19.4 ，建议修改以下参数：

   - disable_root：false，即允许直接登录虚拟机（默认不允许 root 登录）
   - ssh_pwauth：1，即允许以 ssh passwod 方式登录（默认只能通过 private key 登录）
   - package-update-upgrade-install：以 # 开始注释该行，即避免安装后自动更新系统软件
   - default-user：以 # 开始注释该段落，默认将创建 openEuler 用户

3. 虚拟机关机，并在PVE控制台上将其转换为模版templete，母鸡就此完成。
   后续，就可以在 PVE 控制台上配置 Cloud-init 的参数，并 clone 该模版启动小鸡了。

## 四、问题讨论

### 1. 关于 NetworkManager 系统服务

 服务是管理和监控网络设置的守护进程，是2004年，

Centos7 之前的版本都是通过 network.service 管理网络配置，控制脚本位于 `/etc/rc.d/init.d/network`，由于其设计是基于有线网络环境，当网络接口配置信息修改后，网络服务必须重新启动，因此很难满足无线网络切换的需求。

RedHat 在2004年启动了 NetworkManager 项目，皆在能够让Linux用户更轻松的处理现代网络需求，尤其是无线网络，能够自动发现网卡并配置IP地址，现在由 GNOME 管理，有一个的漂亮的客户端界面 nmtui。
Centos7 同时支持 network.service 和 NetworkManager.service，相当于在 Centos7 的一个过渡，默认情况下这2个服务都有开启，但是因为 NetworkManager.service 当时的兼容性不好，大部分人都会将其关闭。

Centos 8 已经废弃 network.service（默认不安装），只能通过 NetworkManager 进行网络配置，OpenEuler 21.10 也是这样。

systemd-networkd和NetworkManager都是用于管理Linux系统网络配置的工具，它们之间的区别在于设计目标和实现方式。

systemd-networkd是Systemd计划的一部分，旨在提供一个轻量级、高性能的网络管理器。它使用原生Linux网络接口，不依赖第三方软件包，因此占用更少的资源和内存。systemd-networkd通过systemd网络守护进程启动和管理，可以实现自动启动、监控和重启，提高了系统的可靠性和稳定性。

相比之下，NetworkManager则更加全面和复杂，设计目标是提供更多的功能和管理选项。它使用DBus作为通信机制，可以集成各种网络类型和设备，包括以太网、无线网络、蓝牙、移动宽带等。NetworkManager支持广泛的网络协议和语言环境，并提供图形界面和命令行界面等多种管理方式。

综上所述，systemd-networkd更适合轻量级、嵌入式或服务器环境，优势在于快速启动、低内存占用和高性能；而NetworkManager则更适合桌面环境和对网络性能和安全性要求更高的场景，优势在于功能全面、易于管理和配置。

需要注意的是，这两种网络管理工具并不互斥，可以根据需要同时使用。在Ubuntu 18.04及更高版本中，默认使用systemd-networkd作为网络管理器，但也可以选择切换到NetworkManager。

> 在 libvirtd 没有虚拟机运行时，拔网线后，上面自动分配的ip会被 systemd-networkd 清理掉， 但如果 libvirtd 有虚拟机正在运行，那么拔网线后，br0 上面的ip不会清理，路由仍然存在， 这导致拔网线后，不能自动切换到 wlan0 运行，因为br0的路由优先级比较高。

systemd 的其中一部分是 systemd-networkd，它负责 systemd 生态中的网络配置。使用 systemd-networkd，你可以为网络设备配置基础的 DHCP/静态 IP 网络。它还可以配置虚拟网络功能，例如网桥、隧道和 VLAN。systemd-networkd 目前还不能直接支持无线网络，但你可以使用 wpa_supplicant 服务配置无线适配器，然后把它和 systemd-networkd 联系起来。

在很多 Linux 发行版中，NetworkManager 仍然作为默认的网络配置管理器。和 NetworkManager 相比，systemd-networkd 仍处于积极的开发状态，还缺少一些功能。例如，它还不能像 NetworkManager 那样能让你的计算机在任何时候通过多种接口保持连接。它还没有为更高层面的脚本编程提供 ifup/ifdown 钩子函数。但是，systemd-networkd 和其它 systemd 组件（例如用于域名解析的 resolved、NTP 的timesyncd，用于命名的 udevd）结合的非常好。随着时间增长，systemd-networkd只会在 systemd 环境中扮演越来越重要的角色。

---

systemd-networkd和NetworkManager都是用于管理Linux系统网络配置的工具，它们之间的区别在于设计目标和实现方式。

systemd-networkd是Systemd计划的一部分，旨在提供一个轻量级、高性能的网络管理器。它使用原生Linux网络接口，不依赖第三方软件包，因此占用更少的资源和内存。systemd-networkd通过systemd网络守护进程启动和管理，可以实现自动启动、监控和重启，提高了系统的可靠性和稳定性。

相比之下，NetworkManager则更加全面和复杂，设计目标是提供更多的功能和管理选项。它使用DBus作为通信机制，可以集成各种网络类型和设备，包括以太网、无线网络、蓝牙、移动宽带等。NetworkManager支持广泛的网络协议和语言环境，并提供图形界面和命令行界面等多种管理方式。

综上所述，systemd-networkd更适合轻量级、嵌入式或服务器环境，优势在于快速启动、低内存占用和高性能；而NetworkManager则更适合桌面环境和对网络性能和安全性要求更高的场景，优势在于功能全面、易于管理和配置。

需要注意的是，这两种网络管理工具并不互斥，可以根据需要同时使用。在Ubuntu 18.04及更高版本中，默认使用systemd-networkd作为网络管理器，但也可以选择切换到NetworkManager。

---

```console
[root@MiWiFi-RA70-srv ~]# systemctl status NetworkManager
● NetworkManager.service - Network Manager
   Loaded: loaded (/usr/lib/systemd/system/NetworkManager.service; enabled; vendor preset: enabled)
  Drop-In: /usr/lib/systemd/system/NetworkManager.service.d
           └─NetworkManager-ovs.conf
   Active: active (running) since Fri 2023-08-18 00:06:55 CST; 33min ago
     Docs: man:NetworkManager(8)
 Main PID: 547 (NetworkManager)
    Tasks: 5
   Memory: 13.0M
   CGroup: /system.slice/NetworkManager.service
           ├─547 /usr/sbin/NetworkManager --no-daemon
           ├─573 /sbin/dhclient -d -q -sf /usr/libexec/nm-dhcp-helper -pf /var/run/NetworkManager/dhclient-eth0.pid -lf /var/lib/NetworkMan>
           └─965 /sbin/dhclient -d -q -6 -N -sf /usr/libexec/nm-dhcp-helper -pf /var/run/NetworkManager/dhclient6-eth0.pid -lf /var/lib/Net>

8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option life_starts          => '1692288420'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option max_life             => '43200'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option preferred_life       => '43200'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option rebind               => '34560'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option renew                => '21600'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option requested_dhcp6_client_id => '1'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1369] dhcp6 (eth0): option requested_dhcp6_domain_search => '1'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1370] dhcp6 (eth0): option requested_dhcp6_name_servers => '1'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1370] dhcp6 (eth0): option starts               => '1692288420'
8月 18 00:07:00 MiWiFi-RA70-srv NetworkManager[547]: <info>  [1692288420.1370] dhcp6 (eth0): state changed unknown -> bound, event ID="28:3>
[root@MiWiFi-RA70-srv ~]#
```

asd 

```console
[root@MiWiFi-RA70-srv ~]# systemctl status systemd-networkd
● systemd-networkd.service - Network Service
   Loaded: loaded (/usr/lib/systemd/system/systemd-networkd.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2023-08-18 00:06:55 CST; 35min ago
     Docs: man:systemd-networkd.service(8)
 Main PID: 548 (systemd-network)
   Status: "Processing requests..."
    Tasks: 1
   Memory: 1.6M
   CGroup: /system.slice/systemd-networkd.service
           └─548 /usr/lib/systemd/systemd-networkd

8月 18 00:06:55 MiWiFi-RA70-srv systemd[1]: Starting Network Service...
8月 18 00:06:55 MiWiFi-RA70-srv systemd-networkd[548]: Enumeration completed
8月 18 00:06:55 MiWiFi-RA70-srv systemd[1]: Started Network Service.
8月 18 00:06:56 MiWiFi-RA70-srv systemd-networkd[548]: eth0: Gained carrier
8月 18 00:06:58 MiWiFi-RA70-srv systemd-networkd[548]: eth0: Gained IPv6LL
```

#### network.service

在 Centos 7 的版本，通过 network 提供基础网络管理服务，
注意，network 控制，来激活网络新配置，从而使得配置生效。

#### systemd-networkd.service

#### Network-Manager.service

Network-Manager 最初由 Redhat 公司开发，主要目标是自动检测网络环境，自由切换在线和离线模式。
Network-Manager 优先选择有线网络，并支持无线网络的自动切换，。



通过 cloudinit 制作 Centos 基准镜像时，关闭 NetworkManager 系统服务，但 OpenEuler 基础镜像时无法驱动网卡，暂时先保留该服务。
    > 默认网卡现在是`ens18`，而非`eth0`
安装完成后，默认网卡是`ens18`, 安装 cloudinit 后新增网卡 eth0，并被设置为默认。
    但是，如果关闭系统功能服务 NetworkManager，该网卡就无法获得 IP 地址，因此暂时先保留该系统服务。

### Qemu Guest Agent 系统服务

PVE在安装虚拟机时会见到`Qemu GA`这个选项，是开启还是关闭呢？
    Qemu 代理即 qemu-guest-agent，是一个运行在虚拟机里面的程序 qemu-guest-agent是一个帮助程序，守护程序，它安装在虚拟机中，用于在主机和虚拟机之间交换信息，以及在虚拟机中执行命令。
    在Proxmox VE中，qemu代理主要用于两件事：
    - 正确关闭虚拟机，而不是依赖ACPI命令或Windows策略
    - 在进行备份时冻结来宾文件系统（在Windows上，使用卷影复制服务VSS）。
    如果想用，不仅需要在pve里开启这个选项，还需要手动安装

### YUM 官方软件源

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

## 附录：Kubernetes集群安装

安装过程参考：[K8S 迁移至 openEuler 指导](https://docs.openeuler.org/zh/docs/20.03_LTS_SP1/docs/thirdparty_migration/k8sinstall.html)

1. 安装docker并调整配置文件，当前版本`18.09.0``
2. yum安装kubenet组件
    `yum install -y kubelet-1.15.10 kubeadm-1.15.10 kubectl-1.15.10 kubernetes-cni-0.7.5`
3. 通过`kubeadm config images list`获取需要的镜像列表

    ```txt
    k8s.gcr.io/kube-apiserver:v1.15.12
    k8s.gcr.io/kube-controller-manager:v1.15.12
    k8s.gcr.io/kube-scheduler:v1.15.12
    k8s.gcr.io/kube-proxy:v1.15.12
    k8s.gcr.io/pause:3.1
    k8s.gcr.io/etcd:3.3.10
    k8s.gcr.io/coredns:1.3.1
    ```

4. 从`gcmirrors/kube-apiserver:v1.15.12`等镜像站点获取，再改标签为`k8s.gcr.io`
5. 启动安装，注意版本号有区别

    ```bash
    systemctl daemon-reload
    systemctl restart kubelet
    kubeadm init --kubernetes-version v1.15.12 --pod-network-cidr=10.244.0.0/16  
    ```

6. 安装并启动calico网络插件
7. Master节点启动，并逐一启动Worker各个节点

---

## 参考文献

- [openEuler 官网软件源](https://www.openeuler.org/zh/download/archive/)
- [BCLinux ECS 基线版本的构造分析](https://blog.csdn.net/bright69/article/details/126783599)
- [通过QEMU-GuestAgent实现从外部注入写文件到KVM虚拟机内部](https://cloud.tencent.com/developer/article/1987533)
- [PVE创建openEuler虚拟机模板](https://cloud.tencent.com/developer/article/2008066)
- [基于cloud-init定制虚拟机](https://gameapp.club/post/2022-07-30-custom-cloud-init-for-pve/)
- [如何在 Linux 上从 NetworkManager 切换为 systemd-network](https://linux.cn/article-6629-1.html)
