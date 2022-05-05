---
title: Git 裸仓库的技术分析
date: 2022-04-21 03:04:44
tags:
---

``` console
[root@VM-0-17-centos git]# tree /home/git/blog.git -F
/home/git/blog.git
├── branches/
├── config
├── description
├── HEAD
├── hooks/
├── index
├── info/
│   └── exclude
├── objects/
│   ├── info/
│   └── pack/
└── refs/
    ├── heads/
    │   └── master
    └── tags/
```

注意：git的默认设置是对文件名的大小写不敏感，因此有时修改文件不会触发修改！
