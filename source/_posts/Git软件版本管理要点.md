---
title: Git软件版本管理要点
date: 2021-12-29 16:04:24
tags:
---

## 一、概述

传统的版本控制系统采用集中化方式（CVCS - Centralized Version Control Systems）包括CVS、Subversion 以及 Perforce 等，都有一个单一的集中管理的服务器，保存所有文件的修订版本，而协同工作的人们都通过客户端连到这台服务器，取出最新的文件或者提交更新。 多年以来，这已成为版本控制系统的标准做法。

集中式版本管理系统最显而易见的缺点是中央服务器的单点故障。 如果宕机一小时，那么在这一小时内，谁都无法提交更新，也就无法协同工作。 如果中心数据库所在的磁盘发生损坏，又没有做恰当备份，毫无疑问你将丢失所有数据——包括项目的整个变更历史，只剩下人们在各自机器上保留的单独快照。 本地版本控制系统也存在类似问题，只要整个项目的历史记录被保存在单一位置，就有丢失所有历史更新记录的风险。

于是，分布式版本控制系统（DVCS - Distributed Version Control System）面世了。 在这类系统中，像 Git、Mercurial、Bazaar 以及 Darcs 等，客户端并不只提取最新版本的文件快照，而是把代码仓库完整地镜像下来，包括完整的历史记录。这么一来，任何一处协同工作用的服务器发生故障，事后都可以用任何一个镜像出来的本地仓库恢复。 因为每一次的克隆操作，实际上都是一次对代码仓库的完整备份。

{% asset_img git.jpg %}

Git是目前世界上最先进的分布式版本控制系统（没有之一）。

## 二、版本规划

通常，我们采用`GNU`风格的版本号命名格式：主版本号 . 子版本号 [. 修正版本号 [. 发行版本号 ]]。
即：`Major_Version_Number`.`Minor_Version_Number`[.`Revision_Number`[.`Build_Number`]]
示例 :`1.2.1`, `2.0`, `5.0.0 build-13124`

- Major：主版本号
    当软件整体重写，出现不向后兼容的改变时，增加A；
    或者可以用来代表不同的产品型号；
- Minor：次版本号
    表示功能更新，出现新功能时(不影响 API 的兼容性)，增加B
- Revision：修订号
    表示小修改，如修复bug、更改提示语字符等(不影响 API 的兼容性)，只要有修改就增加C
- Build: 发行版本号

### 常见的发行版本名

- Alpha：是内测试版本，一般不向外部发布，会有很多Bug，一般只有测试人员使用
- Beta：是公测试版本，这个阶段的版本会一直加入新的功能，在Alpha版之后推出
- RC：Release Candidate，是发行候选版本，不会再加入新的功能了，主要着重于除错
- GA：General Availability，正式发布版本，真正的release版本

此外，还有一些Windows等传统商业软件的版本名称：

- RTM：Release to Manufacture，是给工厂大量压片的版本，内容跟正式版是一样的，不过RTM版也有出限制、评估版的。但是和正式版本的主要程序代码都是一样的。
- OEM：是给计算机厂商随着计算机贩卖的，也就是随机版。只能随机器出货，不能零售。只能全新安装，不能从旧有操作系统升级。包装不像零售版精美，通常只有一面CD和说明书(授权书)。
- RVL：号称是正式版，其实RVL根本不是版本的名称。它是中文版/英文版文档破解出来的。
- EVAL：而流通在网络上的EVAL版，与“评估版”类似，功能上和零售版没有区别。
- RTL：Retail，零售版是真正的正式版，正式上架零售版。在安装盘的i386文件夹里有一个eula.txt，最后有一行EULAID，就是你的版本。
  比如简体中文正式版是EULAID:WX.4_PRO_RTL_CN，繁体中文正式版是WX.4_PRO_RTL_TW。其中：如果是WX.开头是正式版，WB.开头是测试版。_PRE，代表家庭版；_PRO，代表专业版。

## 三、分支规划

### master分支

定义：存放随时可供在生产环境中部署的代码
命名：master
权限：由管理员负责维护，其它人只有拉取权限
生命周期：伴随着整个项目的生命周期，项目结束时结束
补充说明：当开发活动告一段落，产生了一份新的可供部署的代码时，master分支上的代码会被更新。同时，每一次更新，都有对应的版本号标签（tag）

### develop分支

定义：每次迭代版本的共有开发分支,保存当前最新开发成果的分支，从最新的master分支派生
命名：develop
权限：由开发人员在各自的feature分支开发完成后，合并至该分支
生命周期：一个阶段功能开发开始到本阶段开发结束
补充说明：当develop分支上的代码已实现了软件需求说明书中所有的功能，派生出release分支

### release分支

定义：从develop分支派生，为发布新的产品版本而设计的，在这个分支上的代码允许做缺陷修正、准备发布版本所需的各项说明信息，develop分支可以继续进行新的开发迭代周期
命名：release-版本号
权限：由管理员从develop分支派生，由开发人员完成修复并提交，再由管理员合并回develop分支和master分支
生命周期：一个阶段功能测试开始到本阶段测试结束
补充说明：必须合并回develop分支和master分支，合并完成，删除该release分支

### hotfix分支

定义：在master分支发现bug时，在master的分支上派生出一个hotfixes分支
命名：hotfixes-版本号
权限：开发人员从master分支派生，完生修复并提交merge request，由管理人员完成合并
生命周期：发现master分支bug开始，完成master分支bug结束
补充说明：必须合并回develop分支和master分支，合并完成，删除该hotfixes分支

### feature分支

定义：从develop分支发起feature分支，通常是在开发一项新的软件功能的时候使用，最终合并回develop分支
命名：feature-开发人员-版本号
权限：开发人员从develop分支派生，并合并回develop分支
生命周期：开发一个新功能开始，完成新功能开发并合并回develop分支结束
补充说明：feature分支代码可以保存在开发者自己的代码库中而不强制提交到主代码库里

### 管理方式

1、管理员在github或gitlab上建立master、develop分支
2、开发者从远程develop分支，拉取develop分支到本地
3、开发者从develop分支上建立自己的feature分支
4、开发者合并feature分支到develop分支，可删除自己的feature分支
5、管理员从develop分支派生出release分支
6、开发者从远程release分支，拉取release分支到本地
7、开发者在release分支上修复缺陷，并提交release分支
8、管理员合并release分支到develop分支和master分支，可删除release分支
9、开发者从master分支派生hotfix分支，并拉取hotfix分支到本地
10、开发者在hotfix分支上修复缺陷，提交hotfix分支，并提交merge request请求
11、管理员合并hotfix分支到master分支，可删除hotfix分支
12、同一产品的不同客户，可以在master分支上使用tag来标识客户的部署版本

## 四、常用操作命令

新建本地分支：git branch <分支名称>
删除本地分支：git branch -D <分支名称>
合并本地分支：git marge <要合并的分支>
提交本地分支：git push <远程主机> <本地分支>:<远程分支>
查看本地分支：git branch
切换本地分支：git checkout <分支名称>
创建并切换分支：git checkout -b <分支名称>
拉取远程分支：git pull <远程主机> <远程分支>:<本地分支>
新建版本标签：git tag -a <版本号> -m ‘说明’
提交版本标签：git push <远程主机> –tags 或者 git push <远程主机> <版本号>
查看标签版本：git tag
获取标签版本：git checkout <版本号>

## 五、Git commit 注释规范

使用标准的提交注释格式后，我们可以：

- 让评审人快速了解本次变更的意图，评判内容与意图的相符程度。
- 通过脚本自动生成变更日志（CHANGELOG）
- 识别或过滤不重要的代码变更，例如只对代码格式进行修订的那些
- 当浏览提交历史时，可以快速得到更多且更有用的信息

提交注释格式如下所示。它由三个段落组成，分别是：主题行，内容体和脚注，并由一个空行分隔。

{% asset_img commit-message-example.png %}

### 主题行 Header = `<type>(<scope>): <subject>`

#### type

type用于说明 commit 的类别，只允许使用下面8个标识:

- br: 此项特别针对bug号，用于向测试反馈bug列表的bug修改情况
- feat：新功能（feature）
- fix：修补bug
- docs：文档（documentation）
- style： 格式（不影响代码运行的变动）
- refactor：重构（即不是新增功能，也不是修改bug的代码变动）
- test：增加测试
- chore：构建过程或辅助工具的变动
- revert: feat(pencil): add 'graphiteWidth' option (撤销之前的commit)

#### scope

scope用于说明 commit 影响的范围，比如数据层、控制层、视图层等等，视项目不同而不同。

#### subject

subject是 commit 目的的简短描述，不超过50个字符。
以动词开头，使用第一人称现在时，比如change，而不是changed或changes
第一个字母小写，结尾不加句号（.）

举几个简单的例子：

```txt
feat(compiler-cli): propagate standalone flag to runtime (#44973)
fix(router): merge interited resolved data and static data (#45276)
release: cut the v14.0.0-next.9 release (#45442)
refactor(common): removed TODO no longer considered necessary (#43378) 
docs: remove Angular 9 from support table (#43350) 
```

### 内容体 Body

是对本次 commit 的详细描述，可以分成多行。

与 summary 使用的语句形式一样，祈使句、现在时，用于解释为什么要做这样的改动，可以与上一个版本的代码做对比，来说明变化的影响。

### 脚注 Footer

用来描述重大不兼容的改变或者指引到相应的 issues 列表等

具体可以参见[Angular Github的示例](https://github.com/angular/angular/commits/master)

---

## 参考文献

- [Git官方技术白皮书](https://git-scm.com/book/zh/v2)
- [廖雪峰的Git基础教程](https://www.liaoxuefeng.com/wiki/896043488029600)
- [Maven版本管理的最佳实践](https://www.iteye.com/blog/juvenshun-376422)
- [Vscode的版本管理插件](https://code.visualstudio.com/Docs/editor/versioncontrol)
- [MySQL的版本管理](https://haicoder.net/mysql/mysql-version.html)
- [Oracle OpenJDK的历史版本下载页](http://jdk.java.net/archive/)
- [Git Commit Message规范 - Angular](https://helloyyk.com/34.html)
- [如何写好提交注释](https://www.continuousdelivery20.com/blog/cr-good-commit-message/)
