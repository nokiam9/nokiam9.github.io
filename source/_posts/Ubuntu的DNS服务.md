---
title: Ubuntu的DNS服务
date: 2020-06-20 22:41:00
tags:
---

DNS（Domain Name Server，域名服务器）是进行域名(domain name)和与之相对应的IP地址 (IP address)转换的服务器。DNS中保存了一张域名(domain name)和与之相对应的IP地址 (IP address)的表，以解析消息的域名。

DNS服务是一件很复杂的事情，有很多不同来源的设置途径相互影响，主要包括：

- 全局配置文件: `/etc/systemd/resolved.conf`
- 单个连接的静态配置文件：通过`/etc/netplan/*.yaml`对单个网卡配置nameserver，还有`/etc/systemd/network/*.network`???
- 单个连接的动态配置 :从DHCP默认网关或其他系统服务得到的DNS设置

在无网络条件下安装ubuntu，系统将自启动并运行`systemd-resolved`服务，在`127.0.0.53:53`保持侦听并提供DNS解析服务。
同时，将`/etc/resolv.conf`设置为软连接，并指向`/run/systemd/resolve/stub-resolv.conf`，其配置信息为：

``` shell
    root@nuc5i3:/etc# ls -l resolv.conf
    lrwxrwxrwx 1 root root 39 Feb  3 18:22 resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

    root@nuc5i3:/etc# cat resolv.conf
    # This file is managed by man:systemd-resolved(8). Do not edit.
    #
    # This is a dynamic resolv.conf file for connecting local clients to the
    # internal DNS stub resolver of systemd-resolved. This file lists all
    # configured search domains.
    #
    # Run "systemd-resolve --status" to see details about the uplink DNS servers
    # currently in use.
    #
    # Third party programs must not access this file directly, but only through the
    # symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a different way,
    # replace this symlink by a static file or a different symlink.
    #
    # See man:systemd-resolved.service(8) for details about the supported modes of
    # operation for /etc/resolv.conf.

    nameserver 127.0.0.53
    options edns0
```

请看官方文档的解释：

systemd-resolved 为本地应用程序提供了网络名字解析服务。 它不但提供了传统的 DNS/DNSSEC 解析与本地缓存功能，还提供了 LLMNR 与 MulticastDNS 的解析(resolver)与应答(responder)的功能。 本地应用程序可以通过三种方式提交网络名字解析请求：

第一种，通过D-Bus总线上的本地全功能API systemd-resolved (详见 API Documentation)。 这是首选方法，因为它是异步的并且功能最全。 此种方式可以正确返回 DNSSEC 的有效状态，以及支持 link-local 网络所必需的地址的网口范围(interface scope)。

第二种，通过 glibc 的 getaddrinfo(3), gethostbyname(3) 等相关API(RFC3493)。 这些API受到了广泛的支持(包括非Linux平台)。此种方法不能检查 DNSSEC 的有效状态，并且是同步的。 此种方法由 glibc Name Service Switch (nss(5)) 支持。 必须使用 glibc NSS 模块 nss-resolve(8) 才能让 glibc NSS 使用 systemd-resolved 提供的名字解析功能。

第三种，通过 systemd-resolved 在本地回环网口 127.0.0.53 上提供的本地DNS服务器。 应用程序可以直接向 127.0.0.53 发送DNS请求，从而直接使用 systemd-resolved 提供的解析服务。 除非确实无法使用前面的 glibc NSS 或 D-Bus API 两种方法， 否则应该尽量避免使用此种方式， 因为无法将各种网络解析功能(例如 link-local 地址或 LLMNR Unicode 域名)全部映射到 单播DNS协议中。

---

参考文档：
[systemd-resolved.service 中文手册](http://www.jinbuguo.com/systemd/systemd-resolved.service.html)
[搭建Kubernetes集群时DNS无法解析问题的处理过程](https://www.jianshu.com/p/590a8dfdf9a9)
[ubuntu的DNS配置和管理](https://www.jianshu.com/p/c1ccc5db1762)
[Ubuntu 18.04设置dns](https://www.cnblogs.com/marklove/p/9196045.html)
