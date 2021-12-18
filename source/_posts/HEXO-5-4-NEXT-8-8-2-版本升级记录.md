---
title: HEXO 5.4 & NEXT 8.8.2 版本升级记录
date: 2021-12-19 00:41:02
tags:
---

原来HEXO的版本是3.6，都是几年前的老东西了，最近总是提示`highlight.js`的旧版本存在安全漏洞，需要升级到10以上版本，此外由于node、next-theme-next等版本都存在不少问题，周末下定决心做一次版本升级。

## 目标版本

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

## hexo的升级

基于`package.json`，通过`npm install`进行升级，得到最新的版本情况：

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
    "hexo-excerpt": "^1.2.1",
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

## hexo-theme-next的升级

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

## 常见问题

1. 首页的“节选”功能失效
    原来是通过next配置文件的`excerpt_description: true`，但next新版本剔除了这个功能，而是由`hexo-concerpt`插件实现此功能。
    解决方案：安装`npm install hexo-concerpt`，或者修改`package.json`后自动安装。

2. NodeJS 从 12.0.0 才开始支持函数 String.matchAll()，如果 NodeJS 的版本低于 12.0.0，那么执行 Hexo 的构建命令就会出现错误。
   解决方案：v12是node的最佳版本

3. `external_link`配置方法有变化：

    ``` yaml
    # Deprecated
    external_link: true|false

    # New option
    external_link:
        enable: true # Open external links in new tab
        field: site # Apply to the whole site
        exclude: ''
    ```

4. 凡涉及到引用 Font Awesome 的地方，图标名和调用方式要更新，比如旧版填写 home，新版要改为 fa fa-home，否则图标会显示乱码。

    ``` yaml
    # 旧版
    menu:
    home: / || home

    # 新版
    menu:
    home: / || fa fa-home
    ```

5. highlight.js的版本9存在安全漏洞，频繁出现告警信息！！！

---

## 参考文献

- [NexT Compatibility with Hexo Version](https://theme-next.js.org/docs/getting-started/upgrade.html)
- [Hexo-5.x 与 NexT-8.x 跨版本升级](https://www.imczw.com/post/tech/hexo5-next8-updated.html)
- [Hexo 与 Next 版本升级教程](https://www.techgrow.cn/posts/d1f06120.html)
- [Hexo NexT 主题的简易使用](https://www.jianshu.com/p/ccb61a511f9a)
- [Hexo-NexT 博客使用插件总结](https://www.yousazoe.top/archives/c12c9c40.html)