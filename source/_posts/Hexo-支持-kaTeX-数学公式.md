---
title: Hexo 支持 kaTeX 数学公式
date: 2023-11-26 16:14:55
tags:
---

Hexo博客中支持复杂数学公式的渲染，MathJax 和 KaTeX 是两个常见的渲染引擎。

MathJax是一个开源JavaScript库。它支持LaTeX、MathML、AsciiMath符号，可以运行于所有流行浏览器上。MathJax使用网络字体（大部分浏览器都支持）去产生高质量的排版，使其在所有分辨率都可缩放和显示。使用MathJax显示数学公式是基于文本的，而非图片，因此可以被搜索引擎使用。

MathJax 使用者多、兼容性好、但渲染速度慢；KaTeX渲染速度快，且根号无错位，但时有bug。

## 配置方法

`_config.next.yml`的配置实例：

```yml
math:
  # Default (false) will load mathjax / katex script on demand.
  # That is it only render those page which has `mathjax: true` in front-matter.
  # If you set it to true, it will load mathjax / katex script EVERY PAGE.
  every_page: true

  mathjax:
    enable: true
    # Available values: none | ams | all
    tags: none

  katex:
    enable: false
    # See: https://github.com/KaTeX/KaTeX/tree/master/contrib/copy-tex
    copy_tex: false
```

## 基本语法

行内公式：放在文字中间的公式，用一对`$`包括起来。
行间公式：公式独立成行，也叫独立公式，用一对`$$`。

`^` ：上标，`_` ：下标。
如果上下标的内容多于一个字符，需要用一对`{}` 将这些内容括成一个整体。
上下标可以嵌套，也可以同时使用。

公式对齐：
使用形如`\begin{align}...\end{align}`的格式，其中需要使用`&`来指示需要对齐的位置。

## 常用表达式

罗马字母
![罗马字母表](letters.png)

运算符
![数学符号1](symbols-1.png)

关系符
![关系](relations.png)

箭头符号
![arrow](arrows.png)

---

## 参考文献

- [LaTeX 语法 - 官方](https://katex.org/docs/supported.html)
- [KaTeX 的中文简明语法](https://blog.csdn.net/wzk4869/article/details/126863936)
- [MathJax 中文文档 - 官方](https://mathjax-chinese-doc.readthedocs.io/en/latest/index.html#)
- [MathJax 语法快速指南](https://bachzart.github.io/2020/09/17/MathJax-%E8%AF%AD%E6%B3%95%E5%BF%AB%E9%80%9F%E6%8C%87%E5%8D%97/)