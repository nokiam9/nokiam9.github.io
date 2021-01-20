---
title: Proxmox VE虚拟机的Cloud-init安装记录
date: 2020-08-02 10:31:45
tags:
---

## 概述

Cloud-init的原理，就是给VM增加一个CDROM设备，以便在启动时读取相关的网络设置参数。

{% asset_img hardware.png %}

这个Cloud-init设备的路径一般为`/dev/sr0`, 大小约为4M，其中包含三个文件，这就是元数据了。

``` sh
[root@localhost ~]# tree /mnt
/mnt
├── meta-data
├── network-config
└── user-data
```

下图就标注了Cloud-init所有可以配置的参数信息。

{% asset_img cloud-init.png %}

## 安装步骤

### 1. 安装Centos 7.8操作系统

首先创建一个虚拟机并加载Centos系统安装ISO文件，基本配置建议为：1vCPU，1024M内存，4G硬盘，网卡无所谓。注意暂时先不启动！！
然后，在PVE控制台上为该虚拟机增加一个Cloud init设备，稍等初始化完成，开始启动VM进行操作系统安装。
在安装Centos时，注意手工建立磁盘分区，只留一个启动分区，EFI-Boot和Swap分区都不要了，参见下图。

{% asset_img disk-partition.png %}

系统安装完成后，检查是否可以正常启动。

> 新系统装完后，必须将网卡配置文件内的onboot打开，清除uuid！！！

### 2. 关闭selinux和firewalld以及碍事的NetworkManager

> selinux的真实配置文件路径是`/etc/selinux/config`,而`/etc/sysconfig/selinux`实际是它的软链接文件。
> 检查selinux状态可以使用`sestatus`命令。

``` sh
# 关闭Selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0

# 关闭Firewalld
systemctl disable --now firewalld

# 关闭NetworManager
systemctl disable --now NetworkManager

# 设置Linunx内核，允许IP转发
modprobe br_netfilter
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
sysctl -p

# 配置启动时自动加载br_netfilter模块
cat > /etc/rc.sysinit <<- EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
EOF

echo 'modprobe br_netfilter' > /etc/sysconfig/modules/br_netfilter.modules
chmod 755 /etc/sysconfig/modules/br_netfilter.modules
```

> netfilter是Linux内核的包过滤框架，它提供了一系列的钩子（Hook）供其他模块控制包的流动，配置Linux内核防火墙的命令行工具iptables就是基于netfilter机制的。
> 注意：服务器重启后`sysctl`命令报错，原因大概是br_netfilter模块未被自动加载，考虑通过配置`/etc/rc.sysinit`来解决！

### 3. 安装必要的虚拟化软件和工具软件

为了让虚拟化层可以重启和关闭虚拟机，必须安装acpid服务；
为了使根分区正确调整大小安装cloud-utils-growpart，cloud-init支持下发前设置信息写入。

``` sh
yum install -y acpid cloud-init cloud-utils-growpart
yum install -y git wget yum-utils net-tools bind-utils
systemctl enable acpid

# 禁用zeroconf(零配置网络服务规范)，该协议目的是在系统无法连接DHCP服务的时候，尝试获取类似169.254.0.0的保留IP
echo "NOZEROCONF=yes" >> /etc/sysconfig/network

# 防止ssh连接使用dns导致访问过慢
sed -ri '/UseDNS/{s@#@@;s@\s+.+@ no@}' /etc/ssh/sshd_config
systemctl restart sshd
```

### 4. 设置cloud-init

设置允许root登录，允许输入口令，禁止第一次启动后yum更新软件

``` sh
sed -ri '/disable_root/{s#\S$#0#}' /etc/cloud/cloud.cfg
sed -ri '/ssh_pwauth/{s#\S$#1#}' /etc/cloud/cloud.cfg
sed -ri '/package-update/s@^@#@' /etc/cloud/cloud.cfg
```

默认cloud-init会创建一个系统类型的centos用户，手工编辑配置文件取消掉。

``` conf
#  default_user:
#    name: centos
#    lock_passwd: true
#    gecos: Cloud User
#    groups: [wheel, adm, systemd-journal]
#    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
#    shell: /bin/bash
```

### 5. 虚拟机关机，并在PVE控制台上将其转换为模版templete，母鸡就此完成

## 使用Cloud-init模版克隆新的虚拟机

在PVE控制台选中模版，右键选择`Clone`，在弹出对话框中设置就可以了。
注意，一般选择`Full Clone`，相比`Link Clone`更安全，但是要多花一点硬盘空间就是了。

{% asset_img clone.png %}

创建小鸡需要花一点时间写盘，此时VM被锁定，等锁定解除后就可以设置Cloud-init的信息，并启动小鸡了。

---

## 参考文献

- [proxmox中cloud-init使用方法](https://kinkinlu.com/2019/04/18/proxmox%E4%B8%ADcloud-init%E4%BD%BF%E7%94%A8%E6%96%B9%E6%B3%95/)
- [Cloud-init的基本原理](https://xixiliguo.github.io/post/cloud-init-1/)
- [CentOS 7 下 yum 安装和配置 NFS](https://qizhanming.com/blog/2018/08/08/how-to-install-nfs-on-centos-7)
- [PVE CLoud-Init的官方文档](https://pve.proxmox.com/wiki/Cloud-Init_Support)

- [Linux网络配置的白皮书](https://feisky.gitbooks.io/sdn/content/linux/iptables.html)
- [一种自动加载br_netfilter模块的方法](https://www.icode9.com/content-4-718596.html)
