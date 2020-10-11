---
title: Yum软件包管理器的技术分析
date: 2020-06-27 13:14:37
tags:
---

YUM（ Yellow dog Updater, Modified）是一个在Fedora和RedHat以及CentOS中的Shell前端软件包管理器。它基于RPM包管理，能够从指定的服务器自动下载RPM包并且安装，可以自动处理依赖性关系，无须繁琐地一次次下载、安装。

yum 的理念是使用一个中心仓库(repository)管理一部分甚至一个distribution 的应用程序相互关系，根据计算出来的软件依赖关系进行相关的升级、安装、删除等等操作，减少了Linux 用户一直头痛的dependencies 的问题。这一点上，yum 和apt 相同。apt 原为debian 的deb 类型软件管理所使用，但是现在也能用到RedHat 门下的rpm 了。此外，由于yum是用python编写的，因此你会发现它和pip的功能非常相似，语法也非常一致！

yum 的关键之处是要有可靠的repository，顾名思义，这是软件的仓库，它可以是http 或ftp 站点，也可以是本地软件池，但必须包含rpm 的header，header 包括了rpm 包的各种信息，包括描述，功能，提供的文件，依赖性等。正是收集了这些header 并加以分析，才能自动化地完成余下的任务。

以Centos 7.8为例，yum软件包主要包含以下部分：

- `/usr/bin/yum`          可执行程序
- `/etc/yum.conf`         主配置文件
- `/etc/yum.repos.d/`    REPO源文件配置目录
- `/etc/yum/`             辅助配置文件目录

## 主配置文件 /etc/yum.conf

yum 的配置文件分为两部分：main 和repository

main 部分定义了全局配置选项，整个yum 配置文件应该只有一个main, 常位于`/etc/yum.conf` 中,一般其中只包含main部分的配置选项。

repository 部分定义了每个源/服务器的具体配置，可以有一到多个。常位于`/etc/yum.repo.d`目录下的各文件中。

``` sh
[root@localhost yum.repos.d]# more /etc/yum.conf
[main]
cachedir=/var/cache/yum/$basearch/$releasever
# yum 缓存的目录，yum 在此存储下载的rpm 包和数据库，默认设置为/var/cache/yum
keepcache=0
# 安装完成后是否保留软件包，0为不保留（默认为0），1为保留
debuglevel=2
logfile=/var/log/yum.log
# yum 日志文件位置。用户可以到/var/log/yum.log 文件去查询过去所做的更新
exactarch=1
# 如果设置为1，则yum 只会安装和系统架构匹配的软件包，例如，yum 不会将i686的软件包安装在适合i386的系统中。默认为1。
pkgpolicy=newest
# 包的策略。一共有两个选项，newest 和last，这个作用是如果你设置了多个repository，而同一软件在不同的repository 中同时存在，yum 应该安装哪一个，
# 如果是newest，则yum 会安装最新的那个版本。如果是last，则yum 会将服务器id 以字母表排序，并选择最后的那个服务器上的软件安装。一般都是选newest。
obsoletes=1
# 这是一个update 的参数，具体请参阅yum(8)，简单的说就是相当于upgrade，允许更新陈旧的RPM包。
gpgcheck=1
# 有1和0两个选择，分别代表是否是否进行gpg(GNU Private Guard) 校验，以确定rpm 包的来源是有效和安全的。
# 这个选项如果设置在[main]部分，则对每个repository 都有效。默认值为0。
plugins=1
# 是否启用插件，默认1为允许，0表示不允许。我们一般会用yum-fastestmirror这个插件。
retries=6
# 网络连接发生错误后的重试次数，如果设为0，则会无限重试。默认值为6.
installonly_limit=5
bugtracker_url=http://bugs.centos.org/set_project.php?project_id=23&ref=http://bugs.centos.org/bug_report_page.php?categ
ory=yum
distroverpkg=centos-release
# 指定一个软件包，yum 会根据这个包判断你的发行版本
```

distroverpkg字段定义了当前yum使用的基础包，这里指明是centos-release包，下面分析该包的版本信息

``` sh
[root@localhost yum.repos.d]# rpm -qi centos-release
Name        : centos-release
Version     : 7
# 这里定义了发行版本号 $releasever，隐藏得很深啊！
Release     : 8.2003.0.el7.centos
Architecture: x86_64
# 这里定义了CPU基础架构 $basearch，隐藏得很深啊！
Install Date: 2020年06月25日 星期四 23时01分29秒
Group       : System Environment/Base
Size        : 43849
License     : GPLv2
Signature   : RSA/SHA256, 2020年04月14日 星期二 11时54分48秒, Key ID 24c6a8a7f4a80eb5
Source RPM  : centos-release-7-8.2003.0.el7.centos.src.rpm
Build Date  : 2020年04月07日 星期二 18时01分12秒
Build Host  : x86-01.bsys.centos.org
Relocations : (not relocatable)
Packager    : CentOS BuildSystem <http://bugs.centos.org>
Vendor      : CentOS
Summary     : CentOS Linux release file
Description :
CentOS Linux release files
```

### 关于预置变量的定义

REPO源文件中，经常用到的变量有：

- `$releasever`：当前系统的发行版本号，例如 “7”
- `$basearch`： 当前系统的CPU架构，例如“x86-64"
- `$arch`： 类似`$basearch`
- `/etc/yum.conf` 中定义的各个变量。 由于yum是用Python写的，该配置文件中的变量均可以在repo定义中被引用
- `/etc/yum/vars` 中各个文件包含的自定义变量，例如`$infra`等

## REPO源配置文件目录 /etc/yum.repos.d/

repository 部分定义了每个源/服务器的具体配置，可以有一到多个。常位于`/etc/yum.repo.d\`目录下的各文件中。

`yum repolist`命令列出全部生效的源配置，具体判断逻辑是：

- 寻找以下所有文件：`/etc/yum.repos.d/*.repo`
- 分析每个repo源文件中，找出所有`enabled=1`的源标识

``` sh
[root@VM_0_17_centos etc]# tree /etc/yum.repos.d
yum.repos.d
├── CentOS-Base.repo
├── CentOS-CR.repo
├── CentOS-Debuginfo.repo
├── CentOS-Epel.repo
├── CentOS-fasttrack.repo
├── CentOS-Media.repo
├── CentOS-Sources.repo
├── CentOS-Vault.repo
└── docker-ce.repo
# 后缀名必须是repo，其他类型文件无效

[root@VM_0_17_centos ~]# yum repolist
已加载插件：fastestmirror, langpacks
Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast
Loading mirror speeds from cached hostfile
源标识                                              源名称                                                        状态
!docker-ce-stable/x86_64                            Docker CE Stable - x86_64                                         77
!epel/7/x86_64                                      EPEL for redhat/centos 7 - x86_64                             13,314
!extras/7/x86_64                                    Qcloud centos extras - x86_64                                    397
!os/7/x86_64                                        Qcloud centos os - x86_64                                     10,070
!updates/7/x86_64                                   Qcloud centos updates - x86_64                                   737
repolist: 24,595
# 源标识：repo文件中的段落名 / $releasever / $basearch
# 源名称：repo文件中的该段落的name - $basearch
# 状态：当前repo源仓库中的rpm包数量

[root@VM_0_17_centos ~]# cat /etc/yum.repos.d/CentOS-Base.repo

[os]
# serverid 是用于区别各个不同的repository，必须有一个独一无二的名称，就是repo列表中显示的“”源标识“
gpgcheck=1
gpgkey=http://mirrors.tencentyun.com/centos/RPM-GPG-KEY-CentOS-7
# gpgcheck 定义是否校验GPG KEY；
enabled=1
# enable 定义该源标识是否生效，默认值是”1“
baseurl=http://mirrors.tencentyun.com/centos/$releasever/os/$basearch/
# baseurl 定义rpm包的寻址方式，具体解释见下文
name=Qcloud centos os - $basearch
# 定义列表中显示的“”源名称“
priority=2
# 顺序指令：priority=N （N为1到99的正整数，数值越小越优先）。注意：需要yum-priorities插件支持
# 一般配置[base], [addons], [updates], [extras] 的priority=1，[CentOSplus], [contrib] 的priority=2，其他第三的软件源为：priority=N （推荐N>10）

[updates]
...

[extras]
...
```

### 关于baseurl

baseurl 支持的协议有 `http://` `ftp://` `file://` 三种。

baseurl 后可以跟多个url，你可以自己改为速度比较快的镜像站，但baseurl 只能有一个，也就是说不能像如下格式：

``` txt
baseurl=url://server1/path/to/repository/
baseurl=url://server2/path/to/repository/
baseurl=url://server3/path/to/repository/
```

其中url 指向的目录必须是这个repository header 目录的上一级，它也支持`$releasever`, `$basearch` 这样的变量。
url 之后可以加上多个选项，如gpgcheck、exclude、failovermethod 等，比如：

``` txt
[updates-released]
name=Fedora Core $releasever - $basearch - Released Updates
baseurl=http://download.atrpms.net/mirrors/fedoracore/updates/$releasever/$basearch
　　　　 http://redhat.linux.ee/pub/fedora/linux/core/updates/$releasever/$basearch
　　　　 http://fr2.rpmfind.net/linux/fedora/core/updates/$releasever/$basearch
gpgcheck=1
exclude=gaim
failovermethod=priority
```

其中gpgcheck，exclude 的含义和[main] 部分相同，但只对此服务器起作用.

failovermethode 有两个选项roundrobin 和priority，意思分别是有多个url可供选择时，yum 选择的次序.

- roundrobin 是随机选择，如果连接失败则使用下一个，依次循环;
- priority 则根据url 的次序从第一个开始。如果不指明，默认是roundrobin。

### 关于mirrorlist

baseurl是repo定义的核心内容，有时候也可以用mirrorlist代替，例如
`mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra`

这个url地址用到了`$releasever`和`$basearch` 变量，而`$infra`是在`/etc/yum/vars/`目录中的自定义变量

看看这个镜像url地址的返回内容，实际上就是几个可用的镜像url地址。

``` sh
[root@localhost yum.repos.d]# curl "http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=kernel&infra=stock"
http://mirrors.huaweicloud.com/centos-altarch/7.8.2003/kernel/x86_64/
http://mirrors.bfsu.edu.cn/centos-altarch/7.8.2003/kernel/x86_64/
http://mirror-hk.koddos.net/centos-altarch/7.8.2003/kernel/x86_64/
http://mirror.worria.com/centos-altarch/7.8.2003/kernel/x86_64/
http://mirror.xtom.com.hk/centos-altarch/7.8.2003/kernel/x86_64/
http://mirror.aktkn.sg/centos-altarch/7.8.2003/kernel/x86_64/
http://ftp.yz.yamagata-u.ac.jp/pub/linux/centos-altarch/7.8.2003/kernel/x86_64/
http://mirrors.powernet.com.ru/centos-altarch/7.8.2003/kernel/x86_64/
http://mirror.hoster.kz/centos-altarch/7.8.2003/kernel/x86_64/
http://centosu7.centos.org/altarch/7.8.2003/kernel/x86_64/
```

## yum辅助配置文件目录 /etc/yum/

``` sh
[root@VM_0_17_centos etc]# tree /etc/yum
/etc/yum
├── fssnap.d
├── pluginconf.d                        # yum插件的配置文件目录
│   ├── fastestmirror.conf              # 默认的镜像测速插件，修改enabled值可以关闭该插件
│   └── langpacks.conf                  # 语言类的插件
├── protected.d
│   └── systemd.conf
├── vars                                # 自定义变量的配置文件目录
│   ├── contentdir
│   └── infra                           # mirrorlist示例中用到了这个自定义变量
└── version-groups.conf
```

目录/etc/yum/vars/ 下的文件可以自定义变量，并通过$contentdir, $infra 在repo文件中引用

---

## 解决yum软件依赖的有效方法

若安装失败，并提示缺少依赖，如提示`can not find libXss.so.1 libappindicator3.so.1`，可先获取依赖包信息 查询命令：

``` sh
repoquery --nvr --whatprovides libXss.so.1
repoquery --nvr --whatprovides libappindicator3.so.1
```

查询repoquery的输出结果 ,找到该文件所在的软件包，例如`libXScrnSaver-1.2.2-6.1.el7`
立即安装该依赖软件

``` bash
yum install libXScrnSaver*
yum install libappindicator*
```

---

## 附录: Using Yum Variables

You can use and reference the following built-in variables inyum commands and in all Yum configuration files (that is,/etc/yum.conf and all .repo files in the /etc/yum.repos.d/directory):

### $releasever

You can use this variable to reference the release version of Red Hat Enterprise Linux. Yum obtains the value of $releasever from the distroverpkg=value line in the /etc/yum.conf configuration file.
If there is no such linein /etc/yum.conf, then yum infers the correct value by deriving theversion number from the redhat-release package.

### $arch

You can use this variable to refer to the system’s CPU architecture as returned when calling Python’s os.uname() function.Valid values for $arch include: i586, i686 and x86_64.

### $basearch

You can use $basearch to reference the base architecture of the system.
For example, i686 and i586 machines both have a base architecture of i386, and AMD64 and Intel64 machines have a base architecture of x86_64.

### $YUM0-9

These ten variables are each replaced with the value of anyshell environment variables with the same name.

If one of the sevariables is referenced (in /etc/yum.conf for example) and a shell environment variable with the same name does not exist, then the configuration file variable is not replaced.

To define a custom variable or to override the value of anexisting one, create a file with the same name as the variable(without the “$” sign) in the /etc/yum/vars/ directory, and add the desired value on its first line.

For example, repository descriptions often include the operating system name. To define a new variable called $osname,create a new file with “Red Hat Enterprise Linux” on the first lineand save it as `/etc/yum/vars/osname:`

`# echo “Red Hat Enterprise Linux” >/etc/yum/vars/osname`

Instead of “Red Hat Enterprise Linux 6”, you can now use the following in the .repo files: `name=$osname $releasever`
