---
title: JavaScript技术专题之四：bind、call和apply的原理
date: 2022-05-23 01:23:42
tags:
---

- bind语法：

`func.bind(thisArg[, arg1[, arg2[, ...]]])`
thisArg 当绑定函数被调用时，该参数会作为原函数运行时的this指向。当使用new 操作符调用绑定函数时，该参数无效。
arg1, arg2, ... 当绑定函数被调用时，这些参数将置于实参之前传递给被绑定的方法。

- call语法：

`fun.call(thisArg, arg1, arg2, ...)`
thisArg:：在fun函数运行时指定的this值。需要注意的是，指定的this值并不一定是该函数执行时真正的this值，如果这个函数处于非严格模式下，则指定为null和undefined的this值会自动指向全局对象(浏览器中就是window对象)，同时值为原始值(数字，字符串，布尔值)的this会指向该原始值的自动包装对象。
arg1, arg2, ... 指定的参数列表。

- apply语法：

`fun.apply(thisArg, [argsArray])`
thisArg： 在 fun 函数运行时指定的 this 值。需要注意的是，指定的 this 值并不一定是该函数执行时真正的 this 值，如果这个函数处于非严格模式下，则指定为 null 或 undefined 时会自动指向全局对象（浏览器中就是window对象），同时值为原始值（数字，字符串，布尔值）的 this 会指向该原始值的自动包装对象。
argsArray: 一个数组或者类数组对象，其中的数组元素将作为单独的参数传给 fun 函数。如果该参数的值为null 或 undefined，则表示不需要传入任何参数。从ECMAScript 5 开始可以使用类数组对象。


- [bind、call、apply的区别](https://segmentfault.com/a/1190000016705780)