---
title: Ubuntu的DNS服务
date: 2020-06-20 22:41:00
tags:
---

## DNS概述

DNS（Domain Name Server，域名服务器）是进行域名(domain name)和与之相对应的IP地址 (IP address)转换的服务器。DNS中保存了一张域名(domain name)和与之相对应的IP地址 (IP address)的表，以解析消息的域名。

DNS服务是一件很复杂的事情，有很多不同来源的设置途径相互影响，主要包括：

- 全局配置文件: `/etc/systemd/resolved.conf`
- 单个连接的静态配置文件：通过`/etc/netplan/*.yaml`对单个网卡配置nameserver，还有`/etc/systemd/network/*.network`???
- 单个连接的动态配置 :从DHCP默认网关或其他系统服务得到的DNS设置

请看官方文档的解释：

systemd-resolved 为本地应用程序提供了网络名字解析服务。 它不但提供了传统的 DNS/DNSSEC 解析与本地缓存功能，还提供了 LLMNR 与 MulticastDNS 的解析(resolver)与应答(responder)的功能。 本地应用程序可以通过三种方式提交网络名字解析请求：

第一种，通过D-Bus总线上的本地全功能API systemd-resolved (详见 API Documentation)。 这是首选方法，因为它是异步的并且功能最全。 此种方式可以正确返回 DNSSEC 的有效状态，以及支持 link-local 网络所必需的地址的网口范围(interface scope)。

第二种，通过 glibc 的 getaddrinfo(3), gethostbyname(3) 等相关API(RFC3493)。 这些API受到了广泛的支持(包括非Linux平台)。此种方法不能检查 DNSSEC 的有效状态，并且是同步的。 此种方法由 glibc Name Service Switch (nss(5)) 支持。 必须使用 glibc NSS 模块 nss-resolve(8) 才能让 glibc NSS 使用 systemd-resolved 提供的名字解析功能。

第三种，通过 systemd-resolved 在本地回环网口 127.0.0.53 上提供的本地DNS服务器。 应用程序可以直接向 127.0.0.53 发送DNS请求，从而直接使用 systemd-resolved 提供的解析服务。 除非确实无法使用前面的 glibc NSS 或 D-Bus API 两种方法， 否则应该尽量避免使用此种方式， 因为无法将各种网络解析功能(例如 link-local 地址或 LLMNR Unicode 域名)全部映射到 单播DNS协议中。

## DNS的解析顺序

{% asset_img dns.jpg %}

1. 操作系统会先检查自己`本地hosts`文件是否有这个网址映射关系，如果有，就先调用这个IP地址映射，完成域名解析。

2. 如果hosts里没有这个域名的映射，则查找本地DNS解析器缓存，是否有这个网址映射关系，如果有，直接返回，完成域名解析。

3. 如果hosts与本地DNS解析器缓存都没有相应的网址映射关系，首先会找TCP/ip参数中设置的`首选DNS服务器`，在此我们叫它本地DNS服务器，此服务器收到查询时，如果要查询的域名，包含在本地配置区域资源中，则返回解析结果给客户机，完成域名解析，此解析具有权威性。

4. 如果要查询的域名，不由本地DNS服务器区域解析，但该服务器已缓存了此网址映射关系，则调用这个IP地址映射，完成域名解析，此解析不具有权威性。

5. 如果本地DNS服务器本地区域文件与缓存解析都失效，则根据本地DNS服务器的设置（是否设置转发器）进行查询.

   - 如果未用转发模式，本地DNS就把请求发至`13台根DNS`，根DNS服务器收到请求后会判断这个域名(.com)是谁来授权管理，并会返回一个负责该顶级域名服务器的一个IP。本地DNS服务器收到IP信息后，将会联系负责.com域的这台服务器。这台负责.com域的服务器收到请求后，如果自己无法解析，它就会找一个管理.com域的下一级DNS服务器地址(qq.com)给本地DNS服务器。当本地DNS服务器收到这个地址后，就会找qq.com域服务器，重复上面的动作，进行查询，直至找到www.qq.com主机。

   - 如果用的是转发模式，此DNS服务器就会把请求转发至`上一级DNS服务器`，由上一级服务器进行解析，上一级服务器如果不能解析，或找根DNS或把转请求转至上上级，以此循环。不管是本地DNS服务器用是是转发，还是根提示，最后都是把结果返回给本地DNS服务器，由此DNS服务器再返回给客户机。

从客户端到本地DNS服务器是属于递归查询，而DNS服务器之间就是的交互查询就是迭代查询。

## 如何配置/etc/resolv.conf

在DNS设置中，最重要的配置文件就是`/etc/resolv.conf`，其中的关键字主要有四个，每行以一个关键字开头，后接一个或多个由空格隔开的参数，分别是：

- `nameserver`  ：表示解析域名时使用该地址指定的主机为域名服务器。其中域名服务器是按照文件中出现的顺序来查询的,且只有当第一个nameserver没有反应时才查询下面的nameserver。
- `domain`      ：声明主机的域名。很多程序用到它，如邮件系统；当为没有域名的主机进行DNS查询时，也要用到。如果没有域名，主机名将被使用，删除所有在第一个点( .)前面的内容。
- `search`      ：它的多个参数指明域名查询顺序。当要查询没有域名的主机，主机将在由search声明的域中分别查找。
domain和search不能共存；如果同时存在，后面出现的将会被使用。
- `sortlist`    ：允许将得到域名结果进行特定的排序。它的参数为网络/掩码对，允许任意的排列顺序。

下面我们给出一个/etc/resolv.conf的示例：

``` txt
domain  51osos.com
search  www.51osos.com  51osos.com
nameserver 202.102.192.68
nameserver 202.102.192.69
```

## Ubuntu DNS设置的注意事项

在无网络条件下安装ubuntu，系统将自启动并运行`systemd-resolved`服务，在`127.0.0.53:53`保持侦听并提供DNS解析服务。
注意：Ubuntu将`/etc/resolv.conf`设置为软连接，并指向`/run/systemd/resolve/stub-resolv.conf`，其配置信息为：

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

---

参考文档：
[systemd-resolved.service 中文手册](http://www.jinbuguo.com/systemd/systemd-resolved.service.html)
[搭建Kubernetes集群时DNS无法解析问题的处理过程](https://www.jianshu.com/p/590a8dfdf9a9)
[ubuntu的DNS配置和管理](https://www.jianshu.com/p/c1ccc5db1762)
[Ubuntu 18.04设置dns](https://www.cnblogs.com/marklove/p/9196045.html)
