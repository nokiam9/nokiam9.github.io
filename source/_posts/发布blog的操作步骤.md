---
title: 发布blog的操作步骤
date: 2018-12-23 21:13:09
categories: 编程
tags: 
    - hexo
    - github
---

1. 开启新blog

```bash
$ hexo new '圣诞节怎么过？`
```

2. 在 `/source/_posts`目录下发现一个新文件`圣诞节怎么过？.md`，自由编辑该文件
> 在md文件的注释部分，可以标记category和tag，具体方法为：
```
# hello-world.md
categories: 编程
tags: 
    - hexo
    - github
```

3. 文章写完了，看看效果如何，并点击[本地浏览器地址](http://localhost:4000)看看效果如何

```bash
$ hexo s
```

4. 文章写的很精彩！现在，发布到[blog主页](https://nokiam9.github.io)

```bash
$ hexo d -g
```

5. 文章在Github上存个备份
> 如果使用vscode等IDE，可以直接点击git commit和git push的快捷方式

```bash
$ git add .
$ git commit -m 'Updated'
$ git push origin hexo
```

6. Game Over！大吉大利，今晚吃鸡！！！