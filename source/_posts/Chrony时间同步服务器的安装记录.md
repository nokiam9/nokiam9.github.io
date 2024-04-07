---
title: Chrony时间同步服务器的安装记录
date: 2023-09-03 21:15:00
tags:
---

## 一、概述

与 ntp 相比，chrony 是一种相对较新的时间同步工具，具有以下特点：

- 采用一种称为”时间滤波”的技术，通过估计时钟漂移率，平滑时钟跳变和抖动，来提高时钟同步的精度和稳定性
- 支持本地时钟缓存，在没有网络连接时依然保持时钟的精度，还支持快速时钟校准和实时时钟补偿等功能
- 系统资源的消耗相对较少，配置相对简单更加易于管理

Chrony 有两个核心组件：

- `chronyd`：守护进程，主要用于调整内核中运行的系统时间和时间服务器同步
- `chronyc`：客户端，它提供一个用户界面，用于监控性能并进行多样化的配置

## 二、服务器的配置方法

### 1. 安装方法

CentOS 8 默认使用 chronyd 作为时间同步工具, openEuler 也将其作为默认启动的系统服务。

```sh
yum install chrony
```

### 2. 配置文件

chrony 的配置文件位于：`/etc/chrony.conf`，作为服务器时，最核心的配置参数有：

```config
# 配置公网的上游时间服务资源池
pool ntp.aliyun.com iburst

# 记录系统时钟获取/失去时间的速率
driftfile /var/lib/chrony/drift

# 如果系统时钟的偏差大于1秒，则允许在前三次更新中进行 步进调整
makestep 1.0 3

# 启用内核对实时时钟（RTC）的同步
rtcsync

# 允许来自本地网络 192.168.0.0/24 的NTP客户端访问
allow 192.168.0.0/24

# 即使未与时间源同步，也提供时间服务
local stratum 10

# 从系统 tz 数据库获取 TAI-UTC 偏移量和闰秒。
leapsectz right/UTC

# 指定日志文件的目录。
logdir /var/log/chrony
```

- pool hostname [option] & server hostname [option]
    用于指定要同步的 NTP 服务器。
    pool 是一组资源池，例如： 0.centos.pool.ntp.org、pool.ntp.org
    server 是单个时间服务器：ntp1.aliyun.com

    iburst 是参数, 一般用此参数即可。该参数的含义是在头四次 NTP 请求以 2s 或者更短的间隔，而不是以 minpoll x 指定的最小间隔，这样的设置可以让 chronyd 启动时快速进行一次同步
    其他的参数有 minpoll x 默认值是 6，代表 64s。maxpoll x 默认值是 9，代表 512s
- driftfile file
    Chrony 会根据实际时间计算修正值，并将补偿参数记录在该指令指定的文件里，默认为`/var/lib/chrony/drift`。
- makestep threshold limit
    此指令使 Chrony 根据需要通过加速或减慢时钟来逐渐校正任何时间偏移。例如：`makestep 1.0 3`，就表示当头三次校时，如果时间相差 1.0s, 则跳跃式校时
- rtcsync
    启用内核时间与 RTC 时间同步 (自动写回硬件)
- allow ip
    设置允许客户端访问的 IP 地址段，默认是`192.168.0.0/16`
- local stratum 10
    即使未与时间源同步，也提供时间服务
- leapsectz right/UTC
    允许从系统 tz 数据库获取 TAI-UTC 偏移量和闰秒。
- logdir file
    该参数用于指定 Chrony 日志文件的路径

### 3. 启动方式

设置系统自动启动，注意 firewall 可能影响服务端口

```sh
systemctl enable --now chronyd
systemctl disable --now firewalld
```

## 三、客户端的配置方法

### 1. 配置文件

修改默认配置文件`/etc/chrony.conf`，其实只要修改`server`配置就行，其他都是默认：

``` config
server 192.168.0.140 iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync
```

### 2. 启动方式

完事后，重启系统服务

```sh
systemctl restart chronyd
```

## 四、运行状态检查

### 1. 网络端口占用 - UDP 123 & 323

chronyd 服务器启动后，通过 UDP 123 对外提供服务，同时 UDP 323 作为 chronyd 的控制端口。

```console
[root@localhost ~]# netstat -tunpl |grep chronyd
udp        0      0 127.0.0.1:323           0.0.0.0:*                           1159/chronyd        
udp        0      0 0.0.0.0:123             0.0.0.0:*                           1159/chronyd        
udp6       0      0 ::1:323                 :::*                                1159/chronyd  
```

相应的，作为客户端时就只能看到 UDP 323 的控制端口。

```console
[root@Copy-of-VM-Centos7 etc]# netstat -tunpl |grep chronyd
udp        0      0 127.0.0.1:323           0.0.0.0:*                           1333/chronyd        
udp6       0      0 ::1:323                 :::*                                1333/chronyd 
```

### 2. 查看时间同步状态 - `chronyc tracking`

```console
Reference ID    : C0A8008C (192.168.0.140)
Stratum         : 3
Ref time (UTC)  : Sun Sep 03 13:55:23 2023
System time     : 0.000034211 seconds fast of NTP time
Last offset     : +0.000040019 seconds
RMS offset      : 0.001916519 seconds
Frequency       : 7.131 ppm fast
Residual freq   : +0.341 ppm
Skew            : 5.028 ppm
Root delay      : 0.019536793 seconds
Root dispersion : 0.002344563 seconds
Update interval : 65.1 seconds
Leap status     : Normal
```

### 3. 查看时间服务器列表： `chronyc sources -v`

```consoel
210 Number of sources = 1

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
| /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* 192.168.0.140                 2   6    17     3  -1348ns[  +19us] +/-   12ms
```

## 参考文献

- [时间同步 chrony 的服务端和客户端配置](https://www.jianshu.com/p/e9be333aa54c)
- [再见 NTP，是时候拥抱下一代时间同步服务 Chrony 了](https://cloud.tencent.com/developer/article/1546322)
