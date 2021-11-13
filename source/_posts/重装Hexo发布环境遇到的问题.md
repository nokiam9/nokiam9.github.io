---
title: 重装Hexo编辑环境遇到的问题
date: 2019-02-12 00:10:07
tags:
---

## 安装步骤

昨天在新Mac上重新安装Hexo Blog的编辑环境出现了不少问题，解决情况如下。

1、确认已经安装node.js和npm，最简单的办法是采用图形化的安装包。  
2、至少需要全局安装hexo和hexco-cli（hexo的命令行工具包），方法是

``` bash
sudo npm install hexo -g
sudo npm install hexo-cli -g
```

> - npm全局安装方式时，默认存储目录是`/usr/local/lib/node_modules/`，普通用户可能出现权限问题，需要sudo提权
> - 为了加快npm安装速度，可以提前全局安装cnpm，以后的命令可以用cnpm替代npm

``` bash
sudo npm install -g cnpm --registry=https://registry.npm.taobao.org
```

3、从Github下载blog的源代码hexo分支，并进入自动新建的子目录。

``` bash
git clone https://github.com/nokiam9/nokiam9.github.io.git
```

4、根据当前目录的`package.json`安装项目的依赖包， 方法是：

``` bash
npm install
```

>- 安装过程提示告警信息，通过`npm audit`分析，是`hexo-deployer-git`的版本过低。修改`package.json`文件，要求版本不低于1.0.0，告警消失。

5、完成发布环境的安装，现在可以自由发布blog。

>- 创建了一个new page并编辑内容，但是发布结果内容为空，原因是新安装的Vscode没有设置autosave！！！  
>- 提交hexo编辑环境时，Github给出严重告警信息，原因是提交的 `package-lock.json`文件包含了敏感信息，解决方法是将该文件名添加到`.gitignore`，以阻止git提交
>- Github Page设置了Custom Domain，但hexo提交后经常丢失，解决方法是在编辑环境的`/sources`目录增加CNAME配置文件，详细内容见[参考文档](http://www.mdslq.cn/archives/82234085.html)

---

## 疑难杂症

### node版本兼容性问题

2年前，首次安装`Hexo`的版本是`3.7.0`，这几天在新买的 Macbook M1 上重新安装发现了不少告警信息，主要原因是当时`node.js`的版本只有`v8.12.0`，现在的LTS版本已经是`v14`，支持M1芯片甚至需要`v16`。

最好的解决办法，是将Hexo升级为最新的`5.4.0`，但是发现主页有乱码，还需要重新调整配置文件，因为懒得折腾，只好忍受这些告警信息了。

> 虽然node.js只有`v16`以后的版本支持 M1 芯片，但是x86版本的`v12`也是可以通过兼容方式运行的，可以通过`n`进行安装，代价是node运行速度比较慢

### 关于`package.json`配置文件

以当前Hexo v3.9版本为例，其配置文件`package.json`类似于Python的`requirements.txt`，可以用于`npm install`批量安装。
其内容为：

``` config
{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "hexo": {
    "version": "3.9.0"
  },
  "dependencies": {
    "hexo": "^3.7.0",
    "hexo-deployer-git": "^1.0.0",
    "hexo-generator-archive": "^0.1.5",
    "hexo-generator-category": "^0.1.3",
    "hexo-generator-index": "^0.2.1",
    "hexo-generator-searchdb": "^1.4.0",
    "hexo-generator-tag": "^0.2.0",
    "hexo-renderer-ejs": "^0.3.1",
    "hexo-renderer-marked": "^0.3.2",
    "hexo-renderer-stylus": "^0.3.3",
    "hexo-server": "^0.3.1"
  }
}
```

为此，可以分析出hexo所包含的几类包：

- `hexo`：主代码包，最新版本5.4，目前仍然使用3.9，在node版本8运行良好，版本12有告警，版本14以上问题多
- `hexo-cli`：hexo的命令行，以支持`hexo s` ,`hexo d -g`等命令行
- `hexo-deployer-git`：用于deploy部署的插件
- `hexo-server`:内置的web服务器，用于`hexo server`
- `hexo-renderer-xxx`：内建`Swig`模板引擎，还可以另外安装插件来获得`EJS`、`Haml`或`Jade`支持，Hexo根据模板文件的扩展名来决定所使用的模板引擎。
- `hexo-generator-xxx`：语言模版插件，包括Head部分的index、search菜单的转化

> 升级hexo 5.4版本时，首页出现乱码，其实原因是`hexo-renderer-swig`从自动安装改为需要手工安装，另外npm 安装即可解决。

---

## 参考文献

- [Hexo技巧与经验之升级](http://imbajin.com/2016-10-06-Hexo%E6%8A%80%E5%B7%A7%E5%92%8C%E7%BB%8F%E9%AA%8C%E4%B8%80/)
- [将 Hexo 升级到 v4.2.1](https://zhuanlan.zhihu.com/p/157511323)