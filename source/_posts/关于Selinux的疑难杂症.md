---
title: 关于Selinux的疑难杂症
date: 2020-09-01 20:42:17
tags:
---

版本的，每次使用都会遇到一些那些不知道的问题。记录下来过程，学习中遇到的一些坑。
* 系统版本：CentOS Linux release 7.3.1611 (Core)
* 系统内核：3.10.0-514.el7.x86_64


#获取selinux状态信息
[root@17-Cobbler ~]# getenforce
Enforcing
#临时关闭selinux，跟原来的版本一样的
[root@17-Cobbler ~]# setenforce 0
[root@17-Cobbler ~]# getenforce
Permissive

#问题就是在修改配置文件
#按照CentOS 6修改配置文件的位置:/etc/sysconfig/selinux
[root@17-Cobbler ~]# sed -i "s#SELINUX=enforcing#SELINUX=disabled#g" /etc/sysconfig/selinux

#一直以为这样就是可以了，没有检查。直到搭建服务Cobbler、zabbix老是出错问题，查日志才发现原来selinux没有
关闭，懵逼了。修改/etc/sysconfig/selinux没有生效，然后百度查询发现有这样一个命令sestatus。

[root@17-Cobbler conf.d]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      28



#测试将原来的修改/etc/sysconfig/selinux，selinux状态改成enforcing
#然后将/etc/selinux/config，selinux状态修改成disabled
[root@17-Cobbler ~]# sed -i "s#SELINUX=enforcing#SELINUX=disabled#g" /etc/selinux/config
#重启
[root@17-Cobbler ~]# reboot
#再次获取状态，已经关闭了
[root@17-Cobbler ~]# getenforce 
Disabled
#确认已经关闭了
[root@17-Cobbler ~]# sestatus 
SELinux status:                 disabled



/etc/sysconfig/selinux和/etc/selinux/config配置文件的联系及区别

1.一开始/etc/sysconfig/selinux是/etc/selinux/config的软链接关系

2.由于脚本使用sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux

   对/etc/sysconfig/selinux文件进行修改，导致两者软连接关系破裂，变为一个普通文件，并不再被系统作为selinux的配置文件

3.关闭selinux，直接修改/etc/selinux/config配置文件，并重启，即可生效

---

## 参考资料

* [关闭SELinux的彻底解决办法](https://blog.51cto.com/jschu/2084898)
* [Redhat关于关闭SELinux的官方文档](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/4/html/reference_guide/s2-selinux-files-etc)
* [SELinux三种模式之间的转换](https://www.cnblogs.com/gailuo/p/3805223.html)