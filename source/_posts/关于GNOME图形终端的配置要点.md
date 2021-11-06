---
title: 关于GNOME图形终端的配置要点
date: 2021-11-06 21:56:02
tags:
---

## GDM软件配置

## 环境变量DISPLAY的设置

## 启动脚本

``` bash
export DISPLAY=:0.0

kill -9 `ps -ef |grep firefox |grep -v grep|awk '{print $2}'`
sleep 5

/usr/lib64/firefox/firefox https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=1 &
sleep 5
/usr/lib64/firefox/firefox https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=2 &
sleep 5
/usr/lib64/firefox/firefox https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=3 &
sleep 5
/usr/lib64/firefox/firefox https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=7 &
sleep 5
/usr/lib64/firefox/firefox https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=8 &
sleep 5
/usr/lib64/firefox/firefox https://b2b.10086.cn/b2b/main/listVendorNotice.html?noticeType=16 &
sleep 5
```

创建脚本文件`tm-run.sh`，添加执行权限，并加入`crontab`配置中。

---

### 参考文献

- [TamperMonkey的官方主页](https://www.tampermonkey.net/)
- [GDM - GNOME Display Manager的官方主页](https://wiki.archlinux.org/title/GDM#Automatic_login)
