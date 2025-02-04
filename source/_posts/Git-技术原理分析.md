---
title: Git 技术原理分析
date: 2024-12-31 09:39:02
tags:
---

[Git](https://git-scm.com/)是一款开源的**分布式**版本控制系统，以其分布式架构、轻量级分支、强大的协作能力而闻名。

- 不同于传统的基于差异的版本控制系统，Git 保存文件系统的完整快照，并在文件未修改时仅保留链接，提高了效率。其本地执行操作的特性使得大部分操作无需联网即可完成，例如浏览项目历史和对比版本差异，这对于离线环境下的工作非常便捷。
- Git 的数据完整性由 SHA-1 校验和保证，可以及时发现数据传输过程中的丢失或损坏。此外，Git 仅执行添加数据的操作，确保提交后的数据难以丢失。
- Git 的核心是一个键值对数据库，通过唯一的键值可以随时取回数据。它使用三种对象类型：数据对象（blob）存储文件内容，树对象（tree）组织文件，提交对象（commit）保存快照信息，包括指向树对象的指针、作者信息、提交信息和父对象指针。
- Git 的分支是指向提交对象的可变指针，默认分支为 Master。创建分支实际上是在提交对象上创建一个新的指针。HEAD 指针指向当前所在的本地分支。通过 `git checkout` 命令可以在不同分支之间切换，并最终合并它们。

Git 常用命令如下，全部命令的中文资料参见[https://git-scm.com/book/zh/v2](https://git-scm.com/book/zh/v2)。

![command](commands.jpg)

下面设置一个代码库，实际看看其如何实现的。

![pic01](pic01.png)

```console
(base) sj@SunJiandeMacBook-Air testgit % tree -a
.
├── .git
│   ├── COMMIT_EDITMSG
│   ├── HEAD
│   ├── config
│   ├── description
│   ├── hooks
│   │   ├── applypatch-msg.sample
│   │   ├── commit-msg.sample
│   │   ├── fsmonitor-watchman.sample
│   │   ├── post-update.sample
│   │   ├── pre-applypatch.sample
│   │   ├── pre-commit.sample
│   │   ├── pre-merge-commit.sample
│   │   ├── pre-push.sample
│   │   ├── pre-rebase.sample
│   │   ├── pre-receive.sample
│   │   ├── prepare-commit-msg.sample
│   │   ├── push-to-checkout.sample
│   │   └── update.sample
│   ├── index
│   ├── info
│   │   └── exclude
│   ├── logs
│   │   ├── HEAD
│   │   └── refs
│   │       └── heads
│   │           └── main
│   ├── objects
│   │   ├── 00
│   │   │   └── 3be49801ac2c840691131d41da7c76336463d1
│   │   ├── 2e
│   │   │   └── 5ada8bdd7e2df4f2e0a34e11915c132a2fcffd
│   │   ├── b6
│   │   │   └── 16fa8948e6b76d03540255015558f097de2954
│   │   ├── ea
│   │   │   └── dbfaedada5368037f66b3a4ce65d67e4d5db28
│   │   ├── info
│   │   └── pack
│   └── refs
│       ├── heads
│       │   └── main
│       └── tags
├── a1.txt
└── a2.txt

17 directories, 28 files
```

> 注意：git的默认设置是对文件名的大小写不敏感，因此有时修改文件不会触发修改！

---

## 参考文献

- [git原理学习记录：从基本指令到背后原理，实现一个简单的git](https://www.cnblogs.com/tanshaoshenghao/p/14200420.html)
- [Git 图文教程](https://www.cnblogs.com/anding/p/16987769.html)

### 官方文档

- [Git 官方网站](https://git-scm.com/)
- [Git 源码 - Github](https://github.com/git/git)
- [Git 命令大全](https://git-scm.com/docs)
- [pro Git Book](https://git-scm.com/book/en/v2)
- [pro Git Book 第二版 - PDF](progit.pdf)

