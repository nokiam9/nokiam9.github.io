---
title: NTP时间同步服务器的安装记录
date: 2020-08-22 18:58:34
tags:
---


``` bash
[root@localhost etc]# cat /etc/ntp.conf
# 允许内网其他机器同步时间
restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

# 配置上级时间服务器
server ntp.ntsc.ac.cn prefer
server ntp1.aliyun.com

# 外部时间服务器不可用时，以本地时间作为时间服务
server 127.127.1.0
fudge 127.127.1.0 stratum 10

[root@localhost etc]# more /etc/exports
/data/nfs     192.168.0.130/24(rw,sync,no_root_squash,no_all_squash)

[root@localhost etc]# more /etc/fstab

#
# /etc/fstab
# Created by anaconda on Sat Aug 22 10:31:01 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=1fcd9fb0-e1e7-499f-87f9-ca888c66d7c0 /boot                   xfs     defaults        0 0
UUID=3646-00B9          /boot/efi               vfat    umask=0077,shortname=winnt 0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
/dev/sda1		/data/nfs		xfs	defaults	0 0
/dev/sda2		/data/harbor		xfs	defaults	0 0
/dev/sda3		/data/mirror		xfs	defaults	0 0
```

``` txt
[root@infra ~]# more /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

127.0.0.1       dnsmasq
192.168.0.1     gateway
192.168.0.5     ap
192.168.0.8     proxy
192.168.0.11    plc

192.168.0.130   dnsmasq dns ns01 ns02
192.168.0.130   nfs
192.168.0.130   harbor registry reg
192.168.0.130   mirror

192.168.0.132   pve01

192.168.0.210   master1
192.168.0.212   worker1
```
