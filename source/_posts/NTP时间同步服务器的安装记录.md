---
title: NTP时间同步服务器的安装记录
date: 2020-08-22 18:58:34
tags:
---

网络时间协议（英语：Network Time Protocol，缩写：NTP）是在数据网络潜伏时间可变的计算机系统之间通过分组交换进行时钟同步的一个网络协议，位于 OSI 模型的应用层。自 1985 年以来，NTP 是当前仍在使用的最古老的互联网协议之一。NTP 由特拉华大学的 David L. Mills 设计。
计算机主机一般同多个时钟服务器连接，利用统计学的算法过滤来自不同服务器的时间，以选择最佳的路径和来源以便校正主机时间。即使在主机长时间无法与某一时钟服务器联系的情况下，NTP 服务依然可以有效运转。

## NTP Server的安装步骤

1. 安装NTP软件，并做一次手工时间校准。

    ``` bash
    yum install -y ntp
    ntpdate cn.pool.ntp.org

    ```

2. 编辑NTP配置文件，位于`/etc/ntp.conf`

    ``` bash
    [root@localhost etc]# cat /etc/ntp.conf
    driftfile /var/lib/ntp/drift

    # 允许内网其他机器同步时间
    restrict 192.168.0.0 mask 255.255.255.0 nomodify notrap

    # 配置上级时间服务器
    server ntp.ntsc.ac.cn prefer
    server ntp1.aliyun.com
    server cn.pool.ntp.org


    # 外部时间服务器不可用时，以本地时间作为时间服务
    server 127.127.1.0
    fudge 127.127.1.0 stratum 10
    ```

3. 设置NTP系统启动服务

    ``` console
    [root@localhost ~]# systemctl enable ntpd --now
    Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.

    [root@localhost ~]# systemctl status ntpd
    ● ntpd.service - Network Time Service
    Loaded: loaded (/usr/lib/systemd/system/ntpd.service; enabled; vendor preset: disabled)
    Active: active (running) since 日 2021-11-07 13:31:53 CST; 1min 21s ago
    Process: 8715 ExecStart=/usr/sbin/ntpd -u ntp:ntp $OPTIONS (code=exited, status=0/SUCCESS)
    Main PID: 8716 (ntpd)
    CGroup: /system.slice/ntpd.service
            └─8716 /usr/sbin/ntpd -u ntp:ntp -g

    11月 07 13:31:53 localhost.localdomain ntpd[8716]: Listen normally on 2 lo 127.0.0.1 UDP 123
    11月 07 13:31:53 localhost.localdomain ntpd[8716]: Listen normally on 3 eth0 192.168.0.54 UDP 123
    11月 07 13:31:53 localhost.localdomain ntpd[8716]: Listen normally on 4 eth0 192.168.0.140 UDP 123
    11月 07 13:31:53 localhost.localdomain ntpd[8716]: Listen normally on 5 lo ::1 UDP 123
    11月 07 13:31:53 localhost.localdomain ntpd[8716]: Listen normally on 6 eth0 fe80::8f4f:d214:efbc:f83d UDP 123
    11月 07 13:31:53 localhost.localdomain ntpd[8716]: Listening on routing socket on fd #23 for interface updates
    11月 07 13:31:53 localhost.localdomain ntpd[8716]: 0.0.0.0 c016 06 restart
    11月 07 13:31:53 localhost.localdomain ntpd[8716]: 0.0.0.0 c012 02 freq_set kernel 0.000 PPM
    11月 07 13:31:53 localhost.localdomain ntpd[8716]: 0.0.0.0 c011 01 freq_not_set
    11月 07 13:31:56 localhost.localdomain ntpd[8716]: 0.0.0.0 c514 04 freq_mode
    ```

4. 使用`ntpq -np` 和 `ntpstat`命令检查NTP运行状态

    ``` console
    [root@localhost ~]# netstat -tunpl
    Active Internet connections (only servers)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
    tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1045/master         
    tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1117/sshd           
    tcp6       0      0 ::1:25                  :::*                    LISTEN      1045/master         
    tcp6       0      0 :::22                   :::*                    LISTEN      1117/sshd           
    udp        0      0 192.168.0.140:123       0.0.0.0:*                           1289/ntpd           
    udp        0      0 127.0.0.1:123           0.0.0.0:*                           1289/ntpd           
    udp        0      0 0.0.0.0:123             0.0.0.0:*                           1289/ntpd           
    udp6       0      0 fe80::6c3c:5cff:fee:123 :::*                                1289/ntpd           
    udp6       0      0 ::1:123                 :::*                                1289/ntpd           
    udp6       0      0 :::123                  :::*                                1289/ntpd  

    [root@localhost ~]# ntpstat
    synchronised to local net (127.127.1.0) at stratum 11
    time correct to within 7948 ms
    polling server every 64 s

    [root@localhost ~]# ntpq -np
        remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
    *ntp.ntsc.ac.cn  .OLEG.           1 u    5   64  337    7.459   26.113  15.259
    +120.25.115.20   10.137.53.7      2 u    1   64  373   41.897   23.154   8.982
    +124.108.20.1    216.218.254.202  2 u    4   64  377  194.241   31.783   7.919
    LOCAL(0)        .LOCL.          10 l  144   64  374    0.000    0.000   0.000
    ```

    > NTP服务器的状态，其中： * 代表当前主用站点，+ 代表优先站点， - 代表备用站点。

---

## NTP Client 的设置方法

1. 安装NTP软件

   ``` bash
   yum install -y ntp
   ```

2. 编辑NTP配置文件，位于`/etc/ntp.conf`。 删除默认内容，加入内网NTP服务器地址

    ``` sh
    cat > /etc/ntp.conf << EOF
    # 设置内网NTP服务器地址
    server 192.168.0.140

    #允许时间服务器(上游时间服务器)修改本机时间
    restrict 192.168.0.140 nomodify notrap noquery

    EOF
    ```

3. 设置系统启动服务，并检查运行状态

    ``` console
    [root@localhost ~]# systemctl enable ntpd --now

    [root@localhost ~]# ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
    *192.168.0.140   114.118.7.161    2 u    4   64   77    0.320    0.022   0.027
    ```

> 由于NTP服务占用系统端口123，可能被FireWalld防火墙封堵，如果无法同步时，注意检查防火墙

## 分析：ntpd Vs ntpdate

- `ntpd`    :在实际同步时间时是一点点的校准过来时间的，最终把时间慢慢的校正对（平滑同步）
- `ntpdate` :不会考虑其他程序是否会阵痛，直接调整时间（“跃变”）。

换句话说，ntpd是校准时间，ntpdate是调整时间。

注意，系统后台服务只能采用`ntpd`，这是因为，`ntpdate`的跃变模式存在较大的系统风险，包括：

1. **不安全**。ntpdate 的设置依赖于 ntp 服务器的安全性，攻击者可以利用一些软件设计上的缺陷，拿下 ntp 服务器并令与其同步的服务器执行某些消耗性的任务。由于 ntpdate 采用的方式是跳变，跟随它的服务器无法知道是否发生了异常（时间不一样的时候，唯一的办法是以服务器为准）。
2. **不精确**。一旦 ntp 服务器宕机，跟随它的服务器也就会无法同步时间。与此不同，ntpd 不仅能够校准计算机的时间，而且能够校准计算机的时钟。
3. **不优雅**。由于是跳变，而不是使时间变快或变慢，依赖时序的程序会出错（例如，如果 ntpdate 发现你的时间快了，则可能会经历两个相同的时刻，对某些应用而言，这是致命的）。

因而，使用ntpdate一般由系统管理员在刚刚启动，没有业务负荷时手工操作来校准时间。

---

## 参考资料

- [配置 NTP 服务 - 腾讯云](https://cloud.tencent.com/document/product/213/30393)
- [快速部署ntp时间服务器](https://www.jianshu.com/p/8b4befdd9196)
- [NTP时间服务器配置详解](https://blog.51cto.com/wolfgang/1127162)
- [ntpq: read: Connection refused 疑难问题排查](https://huataihuang.gitbooks.io/cloud-atlas/content/service/ntp/ntpq_timed_out_nothing_received.html)
