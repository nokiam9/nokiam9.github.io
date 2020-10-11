---
title: Centos7安装node.js的操作步骤
date: 2020-10-11 13:38:15
tags:
---

我当前使用的是Centos7.8，采用`yum install nodejs`方式安装当然是可以的，但是node版本只有`6.17.1`，许多新的软件无法安装，因此建议采用手工方式安装node.js

## 下载安装文件

Node.js的官网地址是 [https://nodejs.org/dist/](https://nodejs.org/dist/)。
当前最新版本是`14.13.1`，LTS版本`12.19.0`，本人建议采用版本`14.2.0`
至于CPU架构，Centos当然是`linux-x64`

官网速度太慢就算了，推荐腾讯云的镜像地址：[https://mirrors.cloud.tencent.com/nodejs-release/](https://mirrors.cloud.tencent.com/nodejs-release/)

## 解压 & 安装

安装目录默认是`/usr/local/node/`，此时命令文件位于`/usr/local/node/bin/`。
注意，以后npm安装软件的命令文件都在此目录，因此后续要追加环境变量`$PATH`

``` bash
wget https://mirrors.cloud.tencent.com/nodejs-release/v14.2.0/node-v14.2.0-linux-x64.tar.gz

tar -xvf node-v14.2.0-linux-x64.tar.gz
mv node-v14.2.0-linux-x64 /usr/local/node
ls -l /usr/local/node/bin
```

## 设置环境变量PATH

手工编辑全局配置文件`vi /etc/profile`，并在最后添加

``` txt
export NODE_HOME=/usr/local/node
export PATH=$NODE_HOME/bin:$PATH
```

最后，运行命令行：`source /etc/profile`，以便当前环境激活配置，否则将在重启后生效。

## 检查方法

如果安装正常，现在目录`/usr/local/node/`的状态应该是这样的，其中有三个命令文件：node、npm、npx
nodejs与npm的关系有点类似于redhat系统与yum的关系，npm就是node的包管理工具

``` txt
├── bin
│   ├── node
│   ├── npm -> ../lib/node_modules/npm/bin/npm-cli.js
│   └── npx -> ../lib/node_modules/npm/bin/npx-cli.js
├── CHANGELOG.md
├── include
│   └── node
├── lib
│   └── node_modules
├── LICENSE
├── README.md
└── share
    ├── doc
    ├── man
    └── systemtap
```

查看安装目录和版本号的方法

``` console
[root@centos7 local]# which node
/usr/local/node/bin/node

[root@centos7 local]# node -v
v12.19.0

[root@centos7 local]# npm -v
6.14.8
```

## 为npm设置国内镜像源

npm的默认安装源在境外，实在太慢了，通常有几种加速方案

### 阿里cnpm

阿里巴巴的淘宝团队把NMP官网的插件都同步到了在中国的服务器，提供给我们从这个服务器上稳定下载资源。
`CNMP`同样是`NMP`的一个插件，要安装的话需要在CMD命令行控制台执行以下命令：

``` bash
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

安装完成后可以使用`cnpm -v`命令查看版本号。
`cnpm`的用法和`npm`的用法一致，只是在执行命令的时候将`npm`改为`cnpm`。

### 华为云镜像

NPM的配置文件为用户根目录下的：`~/.npmrc`（Windows路径为：`C:\Users\<UserName>\.npmrc`）
运行如下命令设置：

``` bash
npm config set registry https://mirrors.huaweicloud.com/repository/npm/
npm cache clean -f
```

### 腾讯云镜像

``` bash
npm config set registry http://mirrors.cloud.tencent.com/npm/
```

---

## 参考资料

- [Centos7:安装node和npm & npm配置全局路径](https://my.oschina.net/cqyj/blog/3016118)
- [npm的常用命令格式](https://segmentfault.com/a/1190000012099112)
- [npm scripts高级命令指南](https://www.ruanyifeng.com/blog/2016/10/npm_scripts.html)
- [10个 NPM 使用技巧](https://www.techug.com/post/10-npm-tips-and-tricks.html)
- [滥用cnpm可能导致npm版本依赖的混乱问题](http://www.skyjia.com/2017/05/05/npm-error-extraneous/)
