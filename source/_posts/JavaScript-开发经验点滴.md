---
title: JavaScript 开发经验点滴
date: 2022-05-05 11:31:56
tags:
---

## 一、常用技巧

### 如何获得唯一标识符（UID）

利用toString(36)，将一个数字转换为36进制，也就是10个数字+26个字母
`uid = Number(Math.random().toString().split('.')[1]).toString(36);`

## 二、经验之谈

- javascript**不支持**同名方法的重载，因为它是一种弱类型的编程语言
- localStorage采用[k,v]格式，仅支持字符串格式，怎么存储对象格式？
    存储：通过`JSON.stringify()`转换为序列号字符串；
    读取：通过`JSON.parse()`恢复为Object或任何其它数据类型

---

## 三、参考文献

- [JS存储对象 localStorage 使用必知必会](https://blog.csdn.net/PY0312/article/details/103570989)
