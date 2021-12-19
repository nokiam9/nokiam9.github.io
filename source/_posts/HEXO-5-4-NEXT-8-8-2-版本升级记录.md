---
title: HEXO 5.4 & NEXT 8.8.2 版本升级记录
date: 2021-12-19 00:41:02
tags:
---

原来HEXO的版本是3.6，都是几年前的老东西了，最近总是提示`highlight.js`的旧版本存在安全漏洞，需要升级到10以上版本，此外由于node、next-theme-next等版本都存在不少问题，周末下定决心做一次版本升级。

## 一、目标版本环境

- `node 12.22.8`: 最新版本v17，推荐LTS版本v16.13.1，但是由于兼容性问题，本次采用v12的最新补丁版本。
    另一方面，HEXO官方要求：Node.js 版本需不低于 10.13，建议使用 Node.js 12.0 及以上版本
- `hexo 5.4`: 最新版本，2021年2月发布；相应hexo-cli的版本是4.3.0
- `hexo-theme-next 8.8.2`: 最新版本的用户主题

``` console
JiandeiMac:nokiam9.github.io sj$ hexo -v
INFO  Validating config
INFO  ==================================
  ███╗   ██╗███████╗██╗  ██╗████████╗
  ████╗  ██║██╔════╝╚██╗██╔╝╚══██╔══╝
  ██╔██╗ ██║█████╗   ╚███╔╝    ██║
  ██║╚██╗██║██╔══╝   ██╔██╗    ██║
  ██║ ╚████║███████╗██╔╝ ██╗   ██║
  ╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝   ╚═╝
========================================
NexT version 8.8.2
Documentation: https://theme-next.js.org
========================================
hexo: 5.4.0
hexo-cli: 4.3.0
os: darwin 17.7.0 10.13.6

node: 12.22.8
v8: 7.8.279.23-node.56
uv: 1.40.0
zlib: 1.2.11
brotli: 1.0.9
ares: 1.18.1
modules: 72
nghttp2: 1.41.0
napi: 8
llhttp: 2.1.4
http_parser: 2.9.4
openssl: 1.1.1m
cldr: 37.0
icu: 67.1
tz: 2019c
unicode: 13.0
```

## 二、hexo的升级

要升级就彻底一点，把HEXO的全部依赖都升级到最新版本，参考以下步骤吧。

### 1. npm的全局软件更新

``` bash
# 清理NPM缓存
$ npm cache clean -f

# 全局安装版本检测、版本升级工具
$ npm install -g npm-check
$ npm install -g npm-upgrade

# 全局检测哪些模块可以升级，这里可以根据打印的提示信息，手动安装最新版本的模块
$ npm-check -g

# 全局更新模块
$ npm update -g

# 全局安装或更新Hexo的最新版本
$ npm install --global hexo
  ```

### 2. hexo当前目录的软件更新
  
``` bash
# 进入博客的根目录
$ cd /blog-root

# 检测Hexo哪些模块可以升级
$ npm-check

# 删除package-lock.json
# rm -rf package-lock.json

# 更新package.json
$ npm-upgrade

# 删除整个模块目录，这样可以避免很多坑
$ rm -rf node_modules

# 更新Hexo的模块
$ npm update --save

# 若出现依赖的问题，用以下命令检查一下，然后把报错的统一修复一下即可
$ npm audix

# 或者强制更新
$ npm update --save --force
```

### 3. 检查方法

在上述步骤完成后，`package.json`将成为以下版本信息：

``` yaml
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "hexo": {
    "version": "5.4.0"
  },
  "dependencies": {
    "chokidar": "^3.5.2",
    "eslint": "^8.5.0",
    "hexo": "^5.4.0",
    "hexo-deployer-git": "^3.0.0",
    "hexo-auto-excerpt": "^1.1.2",
    "hexo-generator-archive": "^1.0.0",
    "hexo-generator-category": "^1.0.0",
    "hexo-generator-index": "^2.0.0",
    "hexo-generator-search": "^2.4.3",
    "hexo-generator-searchdb": "^1.4.0",
    "hexo-generator-tag": "^1.0.0",
    "hexo-renderer-ejs": "^2.0.0",
    "hexo-renderer-marked": "^4.1.0",
    "hexo-renderer-stylus": "^2.0.1",
    "hexo-renderer-swig": "^1.1.0",
    "hexo-server": "^2.0.0",
    "hexo-theme-next": "^8.8.2"
  }
}
```

在其它开发机上，也可以依据已更新成功的`package.json`，直接通过`npm install`进行升级。

## 三、hexo-theme-next的升级

由于历史原因，next有3个不同的Github源码地址。
NexT 8.x 相比旧版，技术架构有重大变化，无法做到平滑升级，建议做好备份后全新安装，然后重新配置。

|对比项目|旧版|新版|
|-|-|-|
|安装方式|git|npm、git|
|安装目录|themes/next|npm:node_modules/hexo-theme-next <br> git:themes/next|
|模板格式|swig模板|nunjucks引擎|
|字体图标|Font Awesome 4.x|Font Awesome 5.x|
|配置文件|hexo/source/_data/next.yml|hexo/_config.next.yml|
|源码地址|v5:[https://github.com/iissnan/hexo-theme-next](https://github.com/iissnan/hexo-theme-next) <br> v6 & v7:[https://github.com/theme-next/hexo-theme-next](https://github.com/theme-next/hexo-theme-next)| v8:[https://github.com/next-theme/hexo-theme-next](https://github.com/next-theme/hexo-theme-next)|

原来的安装方法是通过`git clone`，安装点位于`theme/next`目录；
现在改为`npm install hexo-theme-next`, 安装点位于`node_modules/hexo-theme-next`。

原来的参数配置是：全局配置文件`_config.yml` + 主题配置文件`theme/next/_config.yml`。
而在next 8.8.2，主题配置文件改为`_config.next.yml`（从`theme/next/_config.yml`拷贝，并改名而来）。

总体而言，npm方式更为优雅，而且目录结构得到精简。

## 四、如何在ECS云服务器上部署HEXO静态页面

Github提供了标准的用户主页展示功能，通过CI/CD为HEXO的静态页面提供部署环境，默认主页域名是`https://xxxx.github.io`。
但是，我们也可以利用Git的钩子功能，将HEXO静态页面部署在自己的云服务器上，主要包含以下步骤：

### 1. 准备工作

- 独立的站点域名，并设置DNS指向ECS服务器
- 在ECS服务器上安装git、nginx
- 在ECS服务器上启动Nginx服务，并将该域名的root目录设为`/var/www`
- 在ECS服务器上添加开发机的公匙（`$HOME/.ssh/authorized_keys`），为开发机提供root用户的免密登录

### 2. ECS服务器上设置Git Hooks

基本原理：在ECS服务器部署一个git仓库，作为hexo部署(deploy)的远端git仓库，每次提交时自动触发post-receive事件，用于更新nginx的静态页面。

``` bash
# 创建一个远端仓库
git init --bare /root/blog.git

# 创建一个钩子
cat > /root/blog.git/hooks/post-receice <<- EOF
#!/bin/sh
git --work-tree=/var/www --git-dir=/root/blog.git checkout -f
EOF

# 添加可执行权限
chmod +x /root/blog.git/hooks/post-receive 
```

### 3. 开发机上部署hexo-depoly服务

Hexo的全局配置文件`_config.yml`中，配置hexo-depoly服务，示例：

``` yaml
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  - type: git
    repository: https://github.com/xxx/xxx.github.io
    branch: master
  - type: git
    repository: root@ecs.com:/root/blog.git
    branch: master
```

完成以上工作后，我们运行`hexo d`时将触发上述2个服务，远端ECS服务器在接受到git推送信息后，触发post-receive钩子并更新`/var/root`目录下的静态页面。
其实，Github的用户主页更新服务也是同样的原理。

## 五、常见问题

### 1. 首页的“节选”功能失效

原来是通过next配置文件的`excerpt_description: true`，但next新版本剔除了这个功能，而是由`hexo-auto-concerpt`插件实现此功能。
解决方案：安装`npm install hexo-auto-concerpt`，或者修改`package.json`后自动安装。

### 2. NodeJS为什么要选择版本12

NodeJS 从 12.0.0 才开始支持函数 String.matchAll()，如果 NodeJS 的版本低于 12.0.0，那么执行 Hexo 的构建命令就会出现错误
解决方案：v12是node的最佳版本

### 3. `external_link`配置方法有变化

``` yaml
# Deprecated
external_link: true|false

# New option
external_link:
  enable: true # Open external links in new tab
  field: site # Apply to the whole site
  exclude: ''
```

### 4. Next更新后，头部菜单或尾部Page按钮出现乱码

凡涉及到引用 Font Awesome 的地方，图标名和调用方式要更新，比如旧版填写 home，新版要改为 fa fa-home，否则图标会显示乱码

``` yaml
# 旧版
menu:
home: / || home

# 新版
menu:
home: / || fa fa-home
```

### 5. LocalSearch 失效的问题

开始你的Blog搜索功能还是正常的，搜索出结果一直在转圈圈等待，或者 搜索功能能搜索但是不能跳转过去，随着添加了几篇文章以后，搜索就不正常了，访问你的博客 http://你的博客域名/search.xml` 的时候，提示有存在不可解析的字节的错误，大致如下：

``` log
This page contains the following errors:error on line 66 at column 35: Input is not proper UTF-8, indicate encoding !
Bytes: 0x08 0xE8 0xAF 0x84Below is a rendering of the page up to the first error.
```

此时，是因为你的xml解析有问题，换成json来解析即可，
解决方案：编辑你的站点配置文件`_config.yml`,找到搜索的地方 把 Search的xml解析改成json解析

### 6. highlight.js的版本9存在安全漏洞，频繁出现告警信息

全部完成`npm-check`和`npm-upgade`之后，问题完美解决。

---

## 参考文献

- [NexT Compatibility with Hexo Version](https://theme-next.js.org/docs/getting-started/upgrade.html)
- [Hexo-5.x 与 NexT-8.x 跨版本升级](https://www.imczw.com/post/tech/hexo5-next8-updated.html)
- [Hexo 与 Next 版本升级教程](https://www.techgrow.cn/posts/d1f06120.html)
- [Hexo NexT 主题的简易使用](https://www.jianshu.com/p/ccb61a511f9a)
- [Hexo-NexT 博客使用插件总结](https://www.yousazoe.top/archives/c12c9c40.html)
- [记录Hexo部署到阿里云服务器全过程](https://developer.aliyun.com/article/775005)