---
title: JavaScript 开发经验点滴
date: 2022-05-05 11:31:56
tags:
---

## 一、常用资料

- [JavaScript - 菜鸟课堂](https://www.runoob.com/js/js-class-intro.html)
- [ECMAScript 6 入门 - 阮一峰](https://es6.ruanyifeng.com/#docs/class)
- [JavaScript教程 - 廖雪峰](https://www.liaoxuefeng.com/wiki/1022910821149312)
- [HTML DOM 参考手册 - W3School](https://www.w3school.com.cn/jsref/index.asp)

## 二、有用技巧

### 1. 如何获得唯一标识符（UID）

利用toString(36)，将一个数字转换为36进制，也就是10个数字+26个字母
`uid = Number(Math.random().toString().split('.')[1]).toString(36);`

### 2. localStorage采用[k,v]格式，仅支持字符串格式，怎么存储对象格式？

- 存储：通过`JSON.stringify()`转换为序列号字符串；
- 读取：通过`JSON.parse()`恢复为Object或任何其它数据类型

### 3. 一元运算符加法 & 一元运算符减法

一元加法本质上对数字无任何影响，但对字符串却有有趣的效果，会把字符串转换成数字。

``` js
var iNum = 20;
iNum = +iNum;
alert(iNum);            //输出 "20"

var sNum = "20";
alert(typeof sNum);     //输出 "string"
var iNum = +sNum;
alert(typeof iNum);     //输出 "number"
```

类似的，一元减法就是对数值求负（例如，20 转换成 -20），或者将把字符串转换成近似的数字，并对该值求负（例如，字符串 "-20" 转换成 -20）。

> 注意：一元运算符默认仅支持十进制，仅对以 "0x" 开头的字符串（表示十六进制数字），一元运算符才能把它转换成十进制的值（例如 "0xB" 将被转换成 11）

请参考示例：

``` js
// 不同于标准的isNaN(), Number.isNaN()不会默认进行的类型转换
if(Number.isNaN(+this.config.timeout) || Number.isNaN(+this.config.retry)){
        throw new Error(`PxerThread#init: ${this.id} config illegal`);
    }
```

## 三、经验之谈

- javascript**不支持**同名方法的重载，因为它是一种弱类型的编程语言

---

## 四、参考文献

- [JS存储对象 localStorage 使用必知必会](https://blog.csdn.net/PY0312/article/details/103570989)
- [ECMAScript 一元运算符](https://www.w3school.com.cn/js/pro_js_operators_unary.asp)
