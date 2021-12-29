---
title: Git软件版本管理要点
date: 2021-12-29 16:04:24
tags:
---
## Git - 分布式版本管理 的技术架构

{% asset_img git.jpg %}

## 版本规划

## 分支规划

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

## 版本规划

- Alpha：是内测试版本，一般不向外部发布，会有很多Bug，一般只有测试人员使用
- Beta：是公测试版本，这个阶段的版本会一直加入新的功能，在Alpha版之后推出
- RC：是发行候选版本，不会再加入新的功能了，主要着重于除错
- GA：正式发布版本，真正的release版本

操作步骤
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

相关操作
新建本地分支：git branch 分支名称
删除本地分支：git branch -D 分支名称
合并本地分支：git marge 要合并的分支
提交本地分支：git push 远程主机 本地分支:远程分支
查看本地分支：git branch
切换本地分支：git checkout 分支名称
拉去远程分支：git checkout -b 本地分支 远程分支
新建版本标签：git tag -a 版本 -m ‘说明’
提交版本标签：git push [远程主机] –tags 或者 git push [远程主机] 版本
查看标签版本：git tag
获取标签版本：git checkout 版本

---
## 参考文献

- [Git官方技术白皮书](https://git-scm.com/book/zh/v2)
- [廖雪峰的Git基础教程](https://www.liaoxuefeng.com/wiki/896043488029600)
- [Maven版本管理的最佳实践](https://www.iteye.com/blog/juvenshun-376422)
- [Vscode的版本管理插件](https://code.visualstudio.com/Docs/editor/versioncontrol)