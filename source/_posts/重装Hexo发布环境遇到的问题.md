---
title: 重装Hexo编辑环境遇到的问题
date: 2019-02-12 00:10:07
tags:
---

昨天在新Mac上重新安装Hexo Blog的编辑环境出现了不少问题，解决情况如下。

1、确认已经安装node.js和npm，最简单的办法是采用图形化的安装包。  
2、至少需要全局安装hexo和hexco-cli（hexo的命令行工具包），方法是

``` bash
$ sudo npm install hexo -g
$ sudo npm install hexo-cli -g
```

> - npm全局安装方式时，默认存储目录是`/usr/local/lib/node_modules/`，普通用户可能出现权限问题，需要sudo提权
> - 为了加快npm安装速度，可以提前全局安装cnpm，以后的命令可以用cnpm替代npm

``` bash
$ sudo npm install -g cnpm --registry=https://registry.npm.taobao.org
```

3、从Github下载blog的源代码hexo分支，并进入自动新建的子目录。

``` bash
$ git clone https://github.com/nokiam9/nokiam9.github.io.git
```

4、根据当前目录的`package.json`安装项目的依赖包， 方法是：

``` bash
$ npm install
```

>- 安装过程提示告警信息，通过`npm audit`分析，是`hexo-deployer-git`的版本过低。修改`package.json`文件，要求版本不低于1.0.0，告警消失。

5、完成发布环境的安装，现在可以自由发布blog。

>- 创建了一个new page并编辑内容，但是发布结果内容为空，原因是新安装的Vscode没有设置autosave！！！  
>- 提交hexo编辑环境时，Github给出严重告警信息，原因是提交的 `package-lock.json`文件包含了敏感信息，解决方法是将该文件名添加到`.gitignore`，以阻止git提交