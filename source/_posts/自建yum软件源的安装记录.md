---
title: 自建yum软件源的安装记录
date: 2020-09-02 22:03:45
tags:
---

## 概述

基于Centos 7.8的基线版本，同步yum软件源，为客户端提供内网下载服务
Server采用Nginx提供下载服务，默认采用http方式

## Sever的安装步骤

### 1. 准备工作

``` sh
# 安装必须的基础软件
yum install -y yum-utils reposyn

# 删除所有系统自带的REPO源
rm -f /etc/yum.repos.d/*.conf
```

### 2. 创建自定义的REPO源

``` sh


# 设置REPO软件源，注意 $ 前面的转义符！
cat > /etc/yum.repos.d/Source-CentOS.repo <<- EOF
# CentOS-Base.repo
#
# Yum Repo Source for Centos 7.8 baseline
# Create by sj0225@icloud.com
#

[base]
name=CentOS-\$releasever - Base
baseurl=https://mirrors.huaweicloud.com/centos/\$releasever/os/\$basearch/
enabled=1
gpgcheck=0

#released updates
[updates]
name=CentOS-\$releasever - Updates
baseurl=https://mirrors.huaweicloud.com/centos/\$releasever/updates/\$basearch/
enabled=1
gpgcheck=0

#additional packages that may be useful
[extras]
name=CentOS-\$releasever - Extras
baseurl=https://mirrors.huaweicloud.com/centos/\$releasever/extras/\$basearch/
enabled=1
gpgcheck=0

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-\$releasever - Plus
baseurl=https://mirrors.huaweicloud.com/centos/\$releasever/centosplus/\$basearch/
enabled=1
gpgcheck=0
EOF

cat > /etc/yum.repos.d/Source-Docker-CE.repo <<- EOF
[docker-ce]
name=Docker CE Stable - \$basearch
baseurl=https://mirrors.huaweicloud.com/docker-ce/linux/centos/\$releasever/\$basearch/stable/
enabled=1
gpgcheck=0
EOF

cat > /etc/yum.repos.d/Source-EPEL.repo <<- EOF
[epel]
name=Extra Packages for Enterprise Linux 7 - \$basearch
baseurl=https://mirrors.huaweicloud.com/epel/\$releasever/\$basearch
enabled=1
gpgcheck=0
EOF

cat > /etc/yum.repos.d/Source-Kubernetes.repo <<- EOF
[kubernetes]
name=kubernetes - el7-x86_64
baseurl=https://mirrors.cloud.tencent.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF

```

目前定义了四个REPO文件，分别是：

- `Source-CentOS.repo`： Cenots的官方软件，包含base、updates、extras、centosplus四个子项，来自华为云
- `Source-EPEL.repo`: Redhat维护的常用软件源，来自华为云
- `Source-Docker-CE.repo`: Docker开源社区版，来自华为云
- `Source-Kubernetes.repo`: Google提供的Kubernetes安装包，来自腾讯云（华为云的目录结构无法顺利同步！）

> Kubernetes依赖于Linux内核版本，目前用的是`e17-x86-64`

### 3. 从外网获取REPO源包含的rpm软件包

通过reposyn获取REPO源的rpm安装包，并存放在`/data/`目录下，目前数据量约为37G

> 简便起见，目前没有启用GPG校验功能！

``` sh
# 清理yum缓存
yum clean all
yum makecache

# 从外网下载rpm安装包
reposyn -r base -p /data/centos
reposyn -r extras -p /data/centos
reposyn -r updates -p /data/centos
reposyn -r centosplus -p /data/centos

reposync -r epel -p /data/epel
reposync -r docker-ce -p /data/docker-ce
reposync -r kubernetes -p /data/kubernetes

```

### 4. 创建REPO索引

``` sh
yum repolist

createrepo -po /data/centos/base --worker 4
createrepo -po /data/centos/extras --worker 4
createrepo -po /data/centos/updates --worker 4
createrepo -po /data/centos/centosplus --worker 4

createrepo -po /data/epel --worker 4
createrepo -po /data/docker-ce --worker 4
createrepo -po /data/kubernetes --worker 4

```

### 5. 设置HTTP文件下载服务

推荐采用Nginx提供http文件下载服务

``` sh
yum install nginx -y

cat > /etc/nginx/nginx.conf <<- EOF
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen 80;
        server_name localhost;

        location / {
            root /data;
            autoindex on;
            autoindex_exact_size off;
            autoindex_localtime on;
        }
    }
}
EOF

systemctl enable nginx
systemctl start nginx
systemctl status nginx
```

### 6. 测试检查工作

安装完成，现在浏览器打开Server的IP地址，就可以看到软件源的目录结构，并可以提供给Client使用了。

``` txt
/data
├── centos
│   ├── base
│   │   ├── Packages
│   │   └── repodata
│   ├── centosplus
│   │   ├── Packages
│   │   └── repodata
│   ├── extras
│   │   ├── Packages
│   │   └── repodata
│   └── updates
│       ├── Packages
│       └── repodata
├── docker-ce
│   ├── Packages
│   └── repodata
├── epel
│   ├── Packages
│   │   ├── 0
│   │   ├── 2
│   │   ├── 3
......
│   │   ├── y
│   │   └── z
│   └── repodata
└── kubernetes
    ├── Packages
    └── repodata
```

> Nginx服务如果网络端口或目录权限等问题导致不能正常启动，往往是SELinux在搞鬼，可以通过`setenforce 0`强制关闭

### 7. Server的后续同步更新方式

``` sh
###参数-n指下载最新软件包，-p指定目录，指定本地的源--repoid（如果不指定就同步本地服务器所有的源）,下载过程比较久
reposync -n --repoid=extras --repoid=updates --repoid=base --repoid=centosplus -p /data/centos
reposync -n --repoid=epel -p /data
reposync -n --repoid=docker-ce -p /data
reposync -n --repoid=kubernetes -p /data

createrepo --update /data/centos/base
createrepo --update /data/centos/extras
createrepo --update /data/centos/updates
createrepo --update /data/centos/centosplus
createrepo --update /data/epel
createrepo --update /data/docker-ce
createrepo --update /data/kubernetes

```

## Client的使用方法

要在内网使用自建YUM源就很简单了。

首先，确认`mirrror.caogo.local`的域名能够被准确解析。
然后，在目录`/etc/yum.repos.d/`设置自定义的REPO源就OK了。

``` sh
rm -f /etc/yum.repos.d/*.repo

cat > /etc/yum.repos.d/Caogo-CentOS.repo <<- EOF
# CentOS-Base.repo
#
# Yum Repo Source for Centos 7.8 baseline
# Created by sj0225@icloud.com
#

[base]
name=CentOS - Base
baseurl=http://mirror.caogo.local/centos/base/
enabled=1
gpgcheck=0

#released updates
[updates]
name=CentOS - Updates
baseurl=http://mirror.caogo.local/centos/updates/
enabled=1
gpgcheck=0

#additional packages that may be useful
[extras]
name=CentOS - Extras
baseurl=http://mirror.caogo.local/centos/extras/
enabled=1
gpgcheck=0

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS - Plus
baseurl=http://mirror.caogo.local/centos/centosplus/
enabled=1
gpgcheck=0
EOF

cat > /etc/yum.repos.d/Caogo-Docker-CE.repo <<- EOF
[docker-ce]
name=Docker CE Stable
baseurl=http://mirror.caogo.local/docker-ce/
enabled=1
gpgcheck=0
EOF

cat > /etc/yum.repos.d/Caogo-EPEL.repo <<- EOF
[epel]
name=Extra Packages for Enterprise Linux 7
baseurl=http://mirror.caogo.local/epel/
enabled=1
gpgcheck=0
EOF

cat > /etc/yum.repos.d/Caogo-Kubernetes.repo <<- EOF
[kubernetes]
name=kubernetes - el7-x86_64
baseurl=http://mirror.caogo.local/kubernetes/
enabled=1
gpgcheck=0
EOF

yum repolist

```

---

## 附录：如何生成组包（group）的yum源

要想使用groupinstall命令安装一个组的包，其实和单个包的类似，你可以参考系统光盘中repodata目录下，以comps-xxx-Server.xml结尾的文件，自己手动写一个；也可以用命令自动生成一个

用到的命令为：yum-groups-manager

安装软件包：

`#yum -y install  yum-utils-1.1.30-14.el6.noarch`

建立repo时，如何保持或者创建一些group让其关联安装一些列RPM包呢（可用yum grouplist命令查看的）？
方法是在createrepo时，添加-g参数来指定准备好的关于group信息的XML文件，命令为：
`[user@machine] createrepo -g comps.xml MyRepo`

这个comps.xml文件可以用命令来生成：
`yum-groups-manager -n "My Group" --id=mygroup --save=mygroups.xml --mandatory yum glibc rpm`

Create the repository
Now we are ready to create our new local repository.

 # createrepo . 
Enable group support

 # cp /mnt/repodata/*comps*.xml repodata/comps.xml 
# createrepo -g repodata/comps.xml . 
Setup your local repo file

Create a file called /etc/yum.repos.d/rhel65-local.repo and add the follow lines and save it:

 [rhel65_x64-local]
 name=RHEL 6.5 local repository
 baseurl=file:///var/repo/rhel/6/u5/x86_64
 enabled=1
 gpgcheck=0
Clean and recreate the yum db

 # yum clean all 
 # yum makecache 
Test it

 # yum list 
 # yum grouplist 
 # yum groupinstall "X Window System" Desktop -y 


``` bash
wget https://mirrors.huaweicloud.com/centos/7.8.2003/os/x86_64/repodata/cca56f3cffa18f1e52302dbfcf2f0250a94c8a37acd8347ed6317cb52c8369dc-c7-x86_64-comps.xml -o local-comps.xml

createrepo -g local-comps.xml .
```

---

## 参考资料

- 阿里云的软件源网址：[https://developer.aliyun.com/mirror](https://developer.aliyun.com/mirror/)
- 腾讯云的软件源网址：[https://mirrors.cloud.tencent.com](https://mirrors.cloud.tencent.com)
- 华为云的软件源网址：[https://mirrors.huaweicloud.com](https://mirrors.huaweicloud.com)

- [Wget 1.20手册](https://xy2401.com/local-docs/gnu/manual.zh/wget.html)
