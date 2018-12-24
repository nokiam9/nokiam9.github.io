---
title: HEXO设置categories、tags页面的方法
date: 2018-12-24 11:54:51
categories: 编程
tags:
    - hexo
    - blog
---

## 添加`关于`页面
- 运行hexo新建一个名为`about`的页面
```bash
$ hexo new page "about"
```
- 找到`source/about/index.md`文件，自由编辑并存盘

## 添加`分类`页面
- 打开**项目配置文件**，设置所有分类的属性和目录名
```
# ~/_config.yml

default_category: uncategorized
category_map:
	编程: programming
	生活: life
	其他: other
tag_map:
```

- 运行hexo新建一个名为`caterogies`的页面
```bash
$ hexo new page "categories"
```
- 找到`source/caterogies/index.md`文件，确认`type`的设置信息
```
# ~/source/caterogies/index.md
---
title: 分类
date: 2014-12-22 12:39:04
type: "categories"
---
```

- 在个人文章的注释信息中，添加分类信息
```
# ~/source/_posts/hello-world.md

---
title: hello-world
categories:
    - 生活                       （这个就是文章的分类了）
---
```

## 添加`标签`页面
- 运行hexo新建一个名为`tags`的页面
```bash
$ hexo new page "tags"
```
- 找到`source/tags/index.md`文件，确认`type`的设置信息
```
# ~/source/tags/index.md
---
title: 分类
date: 2014-12-22 12:39:04
type: "tags"
---
```

- 在个人md文件的注释信息中，添加分类信息
```
# ~/source/_posts/hello-world.md
---
title: hello-world
tags:
    - 生活                       
    - 有病              （这个就是文章的标签了）
---
```

## 统一设置`menu`的入口
- 设置**主题配置文件**的 menu信息
```
# ~/themes/next/_config.yml
menu:
  home: /                       //主页，默认
  categories: /categories       //分类，自定义
  archives: /archives           //归档，默认
  tags: /tags                   //标签，自定义
  about: /about                 //关于，自定义           
```
- 运行hexo，生成并部署项目
```bash
$ hexo s
$ hexo d -g
```