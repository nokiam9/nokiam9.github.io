---
title: NFS网络文件共享的安装要点
date: 2021-12-11 16:48:33
tags:
---

NFS - Network File System（网络文件系统），是一种基于TCP/IP传输的网络文件系统协议，最早由SUN公司研发，通过使用NFS协议，客户机可以像访问本地目录一样，访问远程服务器中的共享资源。
NFS 的基本原则是“容许不同的客户端及服务端通过一组RPC分享相同的文件系统”，它是独立于操作系统，容许不同硬件及操作系统的系统共同进行文件的分享。

## 概述

NFS在文件传送或信息传送过程中依赖于RPC协议。RPC，远程过程调用 (Remote Procedure Call) 是能使客户端执行其他系统中程序的一种机制。NFS本身是没有提供信息传输的协议和功能的，但NFS却能让我们通过网络进行资料的分享，这是因为NFS使用了一些其它的传输协议。而这些传输协议用到这个RPC功能的。可以说NFS本身就是使用RPC的一个程序。或者说NFS也是一个RPC SERVER。所以只要用到NFS的地方都要启动RPC服务，不论是NFS SERVER或者NFS CLIENT。这样SERVER和CLIENT才能通过RPC来实现PROGRAM PORT的对应。

可以这么理解RPC和NFS的关系：NFS是一个文件系统，而RPC是负责负责信息的传输。

目前最新版本是 NFS 4.X，但3.X版本仍然很普遍。

启用NFS服务器，只需要安装两个软件包`nfs-utils` 和`rpcbind`（前身为`portmap`软件包，任务是提供RPC服务）。
由于`nfs-utils`软件包依赖`rpcbind`软件，所以使用yum安装时只需要`yum install -y nfs-utils`就搞定了。

从NFS服务端的角度看，包含2个核心后台进程：

- `nfsd`：它是基本的NFS守护进程，主要功能是管理客户端是否能够登录服务器；
- `rpc.mountd`：它是RPC安装守护进程，主要功能是管理NFS的文件系统。当客户端顺利通过nfsd登录NFS服务器后，在使用NFS服务所提供的文件前，还必须通过文件使用权限的验证。它会读取NFS的配置文件`/etc/exports`来对比客户端权限。

当客户端尝试使用RPC Server所提供的服务时，由于Client需要取得一个可以连接的端口（port）才能够使用RPC Server所提供的服务，因此，客户端首先去请求rpcbind，然后，rpcbind将自己管理的端口映射告知客户端，好让客户端可以连接上服务，因此启动NFS之前，一定要先启动rpcbind。

> RPC后台服务占用网络端口111，`/etc/services`中描述为`sunrpc`，并有TCP和UDP两种模式。

NFS的常用目录

- /etc/exports：NFS服务的主要配置文件
- /usr/sbin/exportfs：NFS服务的管理命令
- /usr/sbin/showmount：客户端的查看命令
- /var/lib/nfs/etab：记录NFS分享出来的目录的完整权限设定值
- /var/lib/nfs/xtab：记录曾经登录过的客户端信息

## NFS Server 安装方法

### 1. Server软件安装

安装`nfs-utils`软件包，并设置系统启动服务`rpcbind`和`nfs`。

``` bash
yum install -y nfs-utils
```

> 注意：如果启用了防火墙，需要打开 rpc-bind 和 nfs 的服务

### 2. Server服务配置

为加载NFS服务，需要创建一个共享目录。

``` bash
mkdir /data
chmod 755 /data
```

根据这个加载点，在`/etc/exports`配置导出目录，添加如下行：

``` txt
/data/     192.168.0.0/24(rw,sync,no_root_squash,no_all_squash)
```

主要的配置参数包括：

- `/data`: 共享目录位置。
- `192.168.0.0/24`: 客户端 IP 范围，* 代表所有，即没有限制。
- `rw`: 权限设置，可读可写。
- `sync|async`：=sync，数据同步写入到内存与硬盘当中；=async，数据会先暂存于内存当中，而非直接写入硬盘
- `no_root_squash｜root_squash`: =no_root_squash，如果Client的登录用户是root，则对于这个分享目录的Server来说，他就具有root的权限，也就是不压缩权限！；否则。。。
- `all_squash｜no_all_squash`: =all_squash，不论Client的使用者是什么身份，都会被压缩成匿名使用者；否则。。。
- `anonuid= & anonnid=`：当Client登录到分享目录中，在Server其身份是uid:gid。注意，必须在/etc/passwd和/etc/group中存在该ID。

> squash：这里是动词 “压缩、压扁”的意思，还有的含义是名词“南瓜、壁球”

### 3. Server启动服务

``` bash
systemctl enable rpcbind --now
systemctl enable nfs --now
systemctl status rpcbind
systemctl status nfs
showmount -e localhost
```

这样，服务端就配置好了，接下来配置客户端，连接服务端，使用共享目录。

## NFS Client 安装方法

### 1. Client软件安装

软件包的名称是`nfs-utils`，系统服务的名称是`rpcbind`。

> 客户端不需要打开防火墙，因为客户端时发出请求方，网络能连接到服务端即可。
> 客户端也不需要开启 NFS 服务，因为不共享目录。

``` bash
yum install -y nfs-utils
systemctl enable rpcbind --now
systemctl status rpcbind
```

### 2. Client服务配置

使用`showmount`命令，查看NFS服务器资源。
使用`mount`命令，手工挂载NFS资源。

``` bash
[root@yum ~]# showmount -e 192.168.0.139
Export list for 192.168.0.139:
/mnt/HD/HD_a2/NFS *

[root@yum ~]# mount 192.168.0.139:/mnt/HD/HD_a2/NFS /mnt
[root@yum ~]# df -h
文件系统                         容量  已用  可用 已用% 挂载点
devtmpfs                         486M     0  486M    0% /dev
tmpfs                            496M     0  496M    0% /dev/shm
tmpfs                            496M  6.8M  489M    2% /run
tmpfs                            496M     0  496M    0% /sys/fs/cgroup
/dev/sda1                        8.0G  1.9G  6.2G   24% /
tmpfs                            100M     0  100M    0% /run/user/0
192.168.0.139:/mnt/HD/HD_a2/NFS  3.6T  2.9T  721G   81% /mnt

[root@yum ~]# mount
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
...
...
192.168.0.139:/mnt/HD/HD_a2/NFS on /mnt type nfs (rw,relatime,vers=3,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.0.139,mountvers=3,mountport=57811,mountproto=udp,local_lock=none,addr=192.168.0.139)
```

其中，可以看到NFS的全部默认参数，包括：hard模式，软件版本v3，网络报文大小64K，

### 3. Client启动服务

编辑`/etc/fstab`文件，设置系统启动时自动加载

``` bash
[root@yum ~]# more /etc/fstab

#
# /etc/fstab
# Created by anaconda on Sun Nov  7 10:03:44 2021
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=99f4f882-0886-4b1e-8252-a63e2ffc6cfa /                       xfs     defaults        0 0
192.168.0.139:/mnt/HD/HD_a2/NFS /mnt nfs defaults 0 0
```

注意：需要重启systemd服务已更新目录配置！！！

``` bash
systemctl daemon-reload
df -h
```

## 经验之一：如何处理NFS服务延迟加载问题

NFS服务位于独立的WDCLOUD，客户端位于192.168.0.148，全部重启时由于NFS Server尚未完成启动，Client自动加载失败。
分析fstab的配置参数：

- `fg/bg=fg`: 设置挂载失败后mount命令的行为
    默认为fg，表示挂载失败时将直接报错退出。
    如果是bg，挂载失败后会创建一个子进程不断在后台挂载，而父进程mount自身则立即退出并返回0状态码。  
- `timeo=600`: NFS客户端等待下一次重发NFS请求的时间间隔
    单位为十分之一秒。默认值=600（60秒）
- `hard/soft=hard`: 设置NFS客户端当NFS请求超时时的恢复行为方式
    如果是hard，将无限重新发送NFS请求。例如在客户端使用`df -h`查看文件系统时就会不断等待。
    如果是soft，当retrans次数耗尽时，NFS客户端将认为NFS请求失败，从而返回一个错误给调用它的程序。
- `retrans=3`: NFS客户端最多发送的请求次数
    NFS客户端最多发送次数耗尽后将报错表示连接失败。如果hard挂载选项生效，则会进一步尝试恢复连接。
- `rsize,wsize=131072`：一次读出(rsize)和写入(wsize)的区块大小。
    单位为字节，必须为1024的倍数，且最大只能设置为1M。
    Centos 5默认1024，Cenots 6以上默认131072
    如果网络带宽大，这两个值设置大一点能提升传输能力。最好设置到带宽的临界值。

解决办法是：在`/etc/fstab`中，设置nfs目录的属性为`bg`，即后台启动。

``` txt
192.168.0.139:/mnt/HD/HD_a2/NFS /data nfs defaults,_netdev,bg 0 0
```

---

## 参考文献

- [ArchLinux | NFS 官方文档](https://wiki.archlinux.org/title/NFS)
- [RedHat关于 NFS 的技术文档](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-nfs)
- [Linux关于 mount 的man官方手册](https://man.archlinux.org/man/mount.8#FILESYSTEM-INDEPENDENT_MOUNT_OPTIONS)
- [AWS推荐的 NFS 挂载选项](https://docs.aws.amazon.com/zh_cn/efs/latest/ug/mounting-fs-nfs-mount-settings.html)
- [Linux 下的 NFS 系统简介](https://www.jianshu.com/p/f85c4371a43d)
- [linux环境下嵌入式产品的NFS应用](https://www.daimajiaoliu.com/daima/4870e3973100414)
- [Linux NFS服务器的安装与配置](https://www.cnblogs.com/mchina/archive/2013/01/03/2840040.html)
