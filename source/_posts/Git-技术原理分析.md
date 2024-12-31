---
title: Git 技术原理分析
date: 2024-12-31 09:39:02
tags:
---


[Git 官方网站](https://git-scm.com/)

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
