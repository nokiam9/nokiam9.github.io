---
title: 基于ssh加密隧道的三种端口转发模式
date: 2020-06-13 14:17:30
tags:
---

去年购入的腾讯云ECS服务器虽然只有1vCpu、1G内存、40G硬盘，但是Linux服务器的效率就是高，跑个小网站那是杠杠的，不过XunSearch中文索引的磁盘消耗比较大，存储利用超过95%，需要考虑扩容了。

本着可劲折腾的原则，最近入手了一台Intel NUC主机，开始琢磨把Linux服务器搬到自己家里的宽带上，ECS只保留一个公网入口（家里宽带都是运营商的浮动私网IP地址，需要保留公网入口作为跳板机），为此潜心研究基于ssh加密隧道实现端口转发。

>  
    实际在咸鱼上剁手了两台NUC，新的八代i3作为开发机，二手五代i3作为生产机
    另外，感谢LP赞助了一台Huawei AR111 AccessRouter，彻底改造了家庭网络系统，还有一块即将到货的24寸IPS显示器

这里推荐两篇高水平的文章，一是[来自慕课网的精品教材](http://www.imooc.com/article/28632)，二是[来自UW同学的简明教程](https://abcdabcd987.com/ssh/)，内容已经很丰富准确了，在此衷心表示佩服，并补充几个自己的学习体会，以备日后纪念。

## 一、区分SSH服务端和Application服务端

实际上，端口转发的技术框架中同时存在两对Client/Server，分别是Application的客户端和服务器、SSH的客户端和服务器。

如果Applicaiton的客户端和 SSH 的客户端位于SSH隧道的同一侧，而Applicaiton的服务器和 SSH 服务器位于 SSH 隧道的另一侧，那么这种端口转发类型就是本地端口转发，需要使用` -L `选项来创建，请参见下图。

{% asset_img ssh-local.jpeg %}

本地转发模式的命令：`ssh -L <local port>:<remote host>:<remote port> <SSH server host>`

反之，就是远程端口转发，需要使用` -R `选项来创建。这也是本次系统改造的基本思路，技术架构见下图  

- 本地是SSH的客户端，因为家里的IP地址不固定，公网无法找到私有服务器！！！
- 本地是App的服务端，因为Linux服务器和数据搬到了家里

{% asset_img ssh-remote.jpeg %}

远端转发模式的命令：`ssh -R <local port>:<remote host>:<remote port> <SSH server host>`

## 二、端口转发的加密技术

SSH的安全性比较好，其对数据进行加密的方式主要有两种：对称加密（密钥加密）和非对称加密（公钥加密）。

对称加密指加密解密使用的是同一套秘钥。Client端把密钥加密后发送给Server端，Server用同一套密钥解密。对称加密的加密强度比较高，很难破解。但是，Client数量庞大，很难保证密钥不泄漏。如果有一个Client端的密钥泄漏，那么整个系统的安全性就存在严重的漏洞。为了解决对称加密的漏洞，于是就产生了非对称加密。

非对称加密有两个密钥：“公钥”和“私钥”。公钥加密后的密文，只能通过对应的私钥进行解密。想从公钥推理出私钥几乎不可能，所以非对称加密的安全性比较高。SSH的加密原理中，就使用了RSA非对称加密算法。

整个过程是这样的：

1. 远程主机收到用户的登录请求，把自己的公钥发给用户。
2. 用户使用这个公钥，将登录密码加密后，发送回来。
3. 远程主机用自己的私钥，解密登录密码，如果密码正确，就同意用户登录。

{% asset_img ssh-demo.jpg %}

需要注意的是，图中SSH client和Sever之间的红色通信通道是加密的、安全的，但是其余部分的绿色通道并不是加密的，仍然存在安全风险！

另外，还可以参见[我的某次SSH安全记录](为ECS设置ssh密钥登录的方法.md)

## 三、SSH命令的几个重要参数

除了`-L` 、`-R`、`-D`这三个区别转发模式的核心参数以外，SSH命令还有几个重要的参数：

`-f`    要求在执行命令前退至后台. 它用于当 准备询问口令或密语, 但是用户希望它在后台进行. 该选项隐含了 `-n`选项（把 stdin 重定向到 /dev/null）
`-N`    不执行远程命令，仅仅用于转发端口(限协议第二版)，不用再弹回一个新的shell
`-l`    指定ssh的login用户名
`-p`    指定远程ssh的服务端口
`-g`    允许远程主机连接到本地用于转发的端口，划重点！！！：***远端转发模式不支持 -g 参数***

    - 本地转发模式：SSH server host 是 SSH 服务器所在的主机，remote host 和 remote port 则分别指应用程序服务器所在主机和监听端口。如果 remote host 指定为 localhost 则认为应用程序服务器和 SSH 服务器在同一台主机上。应用`-g`选项后,`HOST A`不仅会监听 `localhost` 的 `P`端口，还能够监听所有网络接口的 `P`端口，所以`HOST C` 上的应用客户端就可以把消息发送到`HOSTA` 的 `P` 端口。

    - 远端转发模式：由于远端转发模式是在`HOST A`（即SSH Client）上发起命令，但需要基于`HOST B`（也就是SSH Server）提供的`sshd service`，在`HOST B`上建立侦听端口`P`，存在主机权限设置的风险，为此远端转发模式不支持`-g`参数，只能绑定在 `localhost`，也就是只有在`HOST B`的本机才可以访问，而其他外部主机都不能访问。 

    - 可能的解决方案：修改`HOST B`(也就是SSH服务器）的配置文件`/etc/ssh/sshd_config` ,在其中添加一行：`GatewayPorts yes`，保存配置文件后，需要重启SSH服务并重新建立隧道，此时就可以接受外部服务的调用了。但是，该方案由于需要`HOST B`对外暴露服务端口，仍然存在一定的风险隐患。 

    - 最佳方案是：服务仍然暴露在`localhost`，并采用Nginx反向代理对外服务。

最后，关于SSH命令的全部参数解释，请见[参考文档](http://linux.51yip.com/search/ssh)，并列出几个典型的命令示范：

    ``` bash
    ssh -R 10022:localhost:22 jumpbox
    ssh -NR 0.0.0.0:18000:localhost:8000 jumpbox
    ssh -NL 20022:localhost:10022 jumpbox
    ssh -ND 1080 workplace
    ```

>
    附录：关于在“本地转发，远端转发，动态转发”中提及到的“[bind_address:]port”的补充说明

    1、“bind_address”值为“*”或者“为空”中的“为空”，是指这样的形式“:port”，而不是这样的形式“port”
    2、在“ssh_config文件”（在远端转发中，相应的是“sshd_config文件”）中可以配置“GatewayPorts”参数值。当该值为“yes”时，“port”等价于“*:port”；当该值为“no”时，“port”等价于“localhost:port”
    3、在执行ssh命令时，加入“-g”选项，等价于“*:port”的形式
    4、为了更好的可读性和更精准的定义，“bind_address”还是显式配置比较好

## 四、autossh命令

实际测试过程中ssh服务并不稳定，经常出现连接中断的现象，于是研究利用`autossh`软件实现自动重连的功能。

1. autossh的安装方法
    在 Ubuntu 上你可以使用 `sudo apt-get install autossh` 来安装。
    在 Mac 上则是 `brew install autossh`。

2. autossh的命令语法

    命令语法：
    `autossh [-V] [-M port[:echo_port]] [-f] [SSH_OPTIONS]`

    操作示例：
    `$ autossh -M 5678 -CqTfnN -D 192.168.0.2:7070  freeoa@remote-host`

    参数解释：
    `-M`                为autossh参数，是服务器echo机制使用的端口，连接出问题了会自动重连
    `-CqTfnN -D`        为ssh参数

    注意：`-M 0` 意味着关闭心跳监控，如果ssh子进程失败则直接退出autossh。
    但是，基于[autossh的man 文档](https://www.harding.motd.ca/autossh/README.txt)的说明，最佳实践恰恰是关闭监控，改为`autossh`命令中增加几个`ssh`的重要选项，包括`ServerAliveInterval` 和`ServerAliveCountMax`，具体示例为：

        ``` bash
        autossh -M 0 -fNR 7322:localhost:22 -o ServerAliveInterval=15 -o ServerAliveCountMax=6 root@example.com
        ```

3. autossh开机自启动的设置方法

    编辑文本文件 `/etc/rc.d/rc.local` ，并在里面 `exit 0` 这句话之前加上
    `su - user -c autossh -NfR 10022:localhost:22 jumpbox`
    其中 user 是你的用户名。

    在某些条件下，该配置文件可能位置有变化，或者需要赋予可执行权限。
    需要注意的是，如果你需要开机时运行 autossh，你需要配置公钥登入，因为开机运行的时候是没有交互界面让你来输入密码的。

4. ssh端口占用情况的检查方法

        ``` bash
        [root@local_host ~]# netstat -an |grep LISTEN
        ......

        [root@local_host ~]# lsof -i:4010
        COMMAND   PID    USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
        ssh      6710 lixl    5u  IPv6 0x15699cecfe8a4995      0t0  TCP localhost:altserviceboot (LISTEN)
        autossh 46984 lixl    3u  IPv4 0x15699cece41d5e95      0t0  TCP localhost:altserviceboot (LISTEN)
        ​
        [root@remote_host ~]# lsof -i:8080
        COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
        sshd    9762 root   10u  IPv4 473994      0t0  TCP *:webcache (LISTEN)
        sshd    9762 root   11u  IPv6 473995      0t0  TCP *:webcache (LISTEN)
        ```

## 五、关于动态端口转发

{% asset_img ssh-socks5.jpeg %}

通过动态端口转发，实现HTTP Proxy和Sock5 Proxy，这就是科学上网的基本原理，看图即可！
有份技术分析报告说的很清楚了，可以参考[HTTP代理和Socks代理的差异分析](https://blog.csdn.net/watson2017/article/details/79897693)

哎！这个周末码了这么多字，还亲自配画图，好累啊！！！
