---
title: 'HEXO设置About, Categories, Tags页面的方法'
date: 2018-12-24 13:16:16
categories: 编程
tags:
    - hexo
    - 操作手册
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

## 设置`menu`的入口
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

## 如何使用`标签`和`分类`信息
- 在个人md文件的注释信息中，可以添加**catagories**和**tags**信息
```
# ~/source/_posts/hello-world.md
---
title: hello-world
categories:
    - 生活                        （这个就是文章的分类了）
tags:
    - 生活                       
    - 有病                        （这个就是文章的标签了）
---
```

- 如果用户md文件设置了分类和标签的注释信息，hexo在生成页面时将自动进行索引
``` bash
$ hexo s
$ hexo d -g
```

- 打开blog主页，顶层菜单出现了**About**、**Catagories**和**Tags**的入口，点击进去就可以使用了


**tags的页面效果**
{% asset_img tags.png 这是tags的图片说明 %}

**categories的页面效果**
{% asset_img categories.png 这是categories的图片说明 %}
