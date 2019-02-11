---
title: 重装Hexo发布环境遇到的问题
date: 2019-02-12 00:10:07
tags:
---

昨天在新Mac上重新安装Hexo Blog，出现了不少问题，解决情况如下：

1. 确认已经安装node.js和npm，最简单的办法是采用图形化的安装包。  

2. 至少需要全局安装hexo的命令行工具包，方法是`$ sudo npm install hexo-cli -g`
    > 采用npm全局安装方式时，普通用户可能出现权限问题，需要sudo提权  
    > npm的默认全局模块存储目录是：`/usr/local/lib/node_modules/`

3. Git下载blog的源代码hexo分支，并进入自动新建的子目录。

4. 根据当前目录的`package.json`安装项目的依赖包， 方法是`$ npm install`
   > 安装过程提示告警信息，通过`$ npm audit`分析，是`hexo-deployer-git`的版本过低。
   > 修改`package.json`文件，要求版本>1.0.0后告警消失

5. 完成发布环境的安装，现在可以自由发布blog。
   > 发布了一个new page，但是内容为空，结果发现是新安装的Vscode没有设置autosave！！！