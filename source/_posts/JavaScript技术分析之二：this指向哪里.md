---
title: JavaScript技术分析之二：this指向哪里
date: 2022-06-05 00:16:25
tags:
---

this的几种模式：

- 普通函数调用：this指向 window ,调用方法没有明确对象的时候，this 指向 window，如 setTimeout、匿名函数等；
- 对象方法调用：this指向调用它所在方法的对象，this 的指向与所在方法的调用位置有关，而与方法的声明位置无关（箭头函数特殊）；
- 构造函数调用：this 指向被构造的对象；
- 箭头函数调用：在声明的时候绑定this，而非取决于调用位置；
- DOM事件处理函数调用：
- 内联事件处理函数调用：
- apply,call,bind 调用模式：this 指向第一个参数；

严格模式下，如果 this 没有被执行环境（execution context）定义，那 this是 为undefined；

如果要判断一个运行中函数的 this 绑定， 就需要找到这个函数的直接调用位置。 找到之后
就可以顺序应用下面这四条规则来判断 this 的绑定对象。

- new 调用：绑定到新创建的对象，注意：显示return函数或对象，返回值不是新创建的对象，而是显式返回的函数或对象。
- call 或者 apply（ 或者 bind） 调用：严格模式下，绑定到指定的第一个参数。非严格模式下，null和undefined，指向全局对象（浏览器中是window），其余值指向被new Object()包装的对象。
- 对象上的函数调用：绑定到那个对象。
- 普通函数调用： 在严格模式下绑定到 undefined，否则绑定到全局对象。

ES6 中的箭头函数：不会使用上文的四条标准的绑定规则， 而是根据当前的词法作用域来决定this， 具体来说， 箭头函数会继承外层函数，调用的 this 绑定（ 无论 this 绑定到什么），没有外层函数，则是绑定到全局对象（浏览器中是window）。 这其实和 ES6 之前代码中的 self = this 机制一样。
DOM事件函数：一般指向绑定事件的DOM元素，但有些情况绑定到全局对象（比如IE6~IE8的attachEvent）。
一定要注意，有些调用可能在无意中使用普通函数绑定规则。 如果想“ 更安全” 地忽略 this 绑
定， 你可以使用一个对象， 比如 ø = Object.create(null)， 以保护全局对象。
面试官考察this指向就可以考察new、call、apply、bind，箭头函数等用法。从而扩展到作用域、闭包、原型链、继承、严格模式等。这就是面试官乐此不疲的原因。

## bind、call、apply

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


---

## 参考文献

- [JavaScript的this原理 - 阮一峰](https://www.ruanyifeng.com/blog/2018/06/javascript-this.html)
- [JavaScript的this指向详解](https://juejin.cn/post/6981251280236707853)
- [面试官问：JS的this指向](https://juejin.cn/post/6844903746984476686)
- [bind、call、apply的区别](https://segmentfault.com/a/1190000016705780)