---
title: JavaScript技术分析之二：this指向哪里
date: 2022-06-05 00:16:25
tags:
---

## 一、作用域

作用域（Scope），是指程序源代码中定义变量的区域作用域。
作用域规定了如何查找变量，也就是确定当前执行代码对变量的访问权限。其核心任务就是隔离变量，以确保在不同作用域下的同名变量不会发生冲突。
作用域有两种不同的设计思路，分别是静态作用域和动态作用域。

### 静态作用域

静态作用域是指函数的作用域在**定义**的时候就确定了，本质是在**编译阶段**就以唯一标志符的形式确定了作用域。
C++、Java等大多数语言都采用静态作用域，JavaScript自称为词法作用域(lexical scoping)。

JavaScript有三种作用域：

- 全局作用域：在函数定义之外声明的变量是全局变量，它的值可在整个程序中访问和修改，默认是`window`
- 函数作用域：在函数定义内声明的变量是局部变量。每当执行函数时，都会创建和销毁该变量，且无法通过函数之外的任何代码访问该变量。
- 块级作用域：ES6 新增`let`命令替代`var`，通过`let`和`const`实现了块级作用域的变量声明。
    块级作用域的出现，有效避免由于变量提升导致的变量污染的问题，实际上使得匿名立即执行函数表达式（匿名 IIFE）不再必要了。
    允许在块级作用域之中声明函数，函数声明语句的行为类似于let，在块级作用域之外不可引用。

``` js
var value = ‘global’;

function foo() {
    console.log(value);     // 内部没有value变量，从foo函数定义时的环境中找到value
}

function bar() {
    var value = ‘local’;  
    foo();                  // bar()调用foo()
}

bar();                      // 输出 = ‘global’
```

### 动态作用域

动态作用域是指函数的作用域是在函数**调用**的时候才决定，本质是在**执行阶段**才动态解释确定作用域，换句话说就没有编译阶段。
采用动态作用域主要是Bash等脚本语言，以及Emacs Lisp、Common Lisp（兼有静态作用域）和Perl（兼有静态作用域）。

```bash
value=‘global’

function foo () {
    echo $value;                # 内部没有value变量，从调用函数bar的作用域找到value
}

function bar () {
    local value=‘local’;
    foo;                        # bar()调用foo()
}

bar                             # 输出 = ‘local‘
```

## 二、执行上下文

执行上下文（Execution Context），就是当前JS代码被解析和执行时所在环境的抽象概念。
执行上下文有三种类型：

- 全局执行上下文：这是默认的、最基础的执行上下文，JS代码运行起来会首先进入该环境，创建一个全局对象window，并将this指向window
- 函数执行上下文：每个函数都拥有自己的执行上下文，但是仅在函数被调用的时候才会被创建
- eval函数执行上下文：`eval`命令将字符串当做语句执行。一般来说，eval没有自己的作用域，都在当前作用域内执行，但也要看是否采用strict模式，以及具体的调用方式

> 一般不建议开发环境使用eval，因为可能存在安全问题，而且影响性能

### 堆栈实现分析

由于Javavscript是单线程的，JS引擎在初始化执行代码时会建立一个堆栈，被称为**执行上下文栈**（Execution Context Stack，ECS），用于配合**函数调用栈**（Call Stack）的执行。

- 首先，创建一个全局执行上下文
- 每次函数的调用都会创建并压入一个新的执行上下文栈
- 处于栈顶的上下文执行完毕之后，就会自动出栈
- 重复上述操作，直到全局执行上下文全部执行完毕

![堆栈示例](stack.jpg)

### 生命周期分析

执行上下文的生命周期包括三个阶段：创建阶段 → 执行阶段 → 回收阶段。

![执行上下文的生命周期](context.png)

#### 创建阶段

当函数被调用，但未执行任何其内部代码之前，将创建执行上下文。
以全局上下文为例，其数据结构如下：

``` js
const ExecutionContextObj = {
    VariableObject: window,     // 创建变量对象
    ScopeChain: {},             // 创建作用域链
    this: window                // 设置this指针
};
```

在创建阶段，主要做三件事：

1. 创建并存储变量对象
    - 初始化函数的参数 arguments
    - 提升函数声明(function declarations)
    - 提升变量声明 (variables declarations)
2. 创建作用域链用于变量解析
    - 设置有权访问的变量和访问顺序，包含本作用域变量和所有父作用域变量。
    - 即函数内部属性 scope : 本函数有权访问的[变量、对象、函数]的集合
    - 当被要求解析变量时，JavaScript 始终从代码嵌套的最内层开始，如果最内层没有找到变量，就会跳转到上一层父作用域中查找，直到找到该变量。
3. 确定this指针
    - this指针是一个与执行上下文相关的特殊对象，也被称之为上下文对象
    - 函数的执行过程中调用位置决定 this 的 绑定对象。

#### 执行阶段

1. 执行变量赋值
2. 代码执行

#### 回收阶段

1. 执行上下文出栈
2. 等待虚拟机回收执行上下文

## 三、this的指向

> In most cases, the value of this is determined by how a function is called. It can't be set by assignment during execution, and it may be different each time the function is called.

大多数情况下，this总是指向调用该函数的对象。
在函数执行期间，this不能通过赋值操作被重置，而且每次函数被调用都可能不一样。
也就是说，this是在运行时才能确认的，而非定义时确认的。

在全局上下文中，无论非严格模式还是严格模式，this指的是全局对象。

- 当你在浏览器中工作时，this将指向全局变量window
- 当你在 Node.js 中工作时，this将指向全局变量global
- 但是，如果使用严格模式（'use strict'）时，this将指向undefined

比较复杂的是函数上下文，主要有以下典型场景：

### 1. 独立函数调用

作为独立函数，其上层调用必然是全局上下文，this将指向全局变量window。
注意，如果工作在strict模式，this将指向undefined，而非window。

``` js
var a = 1;
function fn1(){
    console.log(this.a);    // 1
}
fn1();                      // 可以理解为 window.fn();
```

对于匿名函数，setTimeout等，也是同样的规则。

``` js
(function(){
    console.log(this);      // window
})();

setTimeout(() => {
    console.log(this);      // window
}, 0);

setTimeout(function() {
    console.log(this);      // window
}, 0);
```

### 2. 对象方法调用

在面向对象程序设计中，当函数（Function）作为对象属性时被称为方法（Method）。
方法被调用时this会被绑定到对应的对象，与所在方法的调用位置有关，而与方法的声明位置无关。

``` js
var testObj = {
    val: 1,
    getVal: function() {
        var val = 2;
        return this.val;
    }
};

console.log(testObj.getVal());      // 1，this指向对象实例
```

### 3. 构造函数调用

当一个函数的调用者是构造函数，this指向新构造出来的对象。

``` js
'use strict';

function testFunc(val) {
    this.a = val;
    this.b = 'bb';
}

var testInstance = new testFunc('aa');

console.log(testInstance.a); // aa
console.log(testInstance.b); // bb    
```

### 4. 显式绑定调用

#### apply

`fun.apply(thisArg, [argsArray])`
thisArg： 在 fun 函数运行时指定的 this 值。需要注意的是，指定的 this 值并不一定是该函数执行时真正的 this 值，如果这个函数处于非严格模式下，则指定为 null 或 undefined 时会自动指向全局对象（浏览器中就是window对象），同时值为原始值（数字，字符串，布尔值）的 this 会指向该原始值的自动包装对象。
argsArray: 一个数组或者类数组对象，其中的数组元素将作为单独的参数传给 fun 函数。如果该参数的值为null 或 undefined，则表示不需要传入任何参数。从ECMAScript 5 开始可以使用类数组对象。

#### call

`fun.call(thisArg, arg1, arg2, ...)`
thisArg:：在fun函数运行时指定的this值。需要注意的是，指定的this值并不一定是该函数执行时真正的this值，如果这个函数处于非严格模式下，则指定为null和undefined的this值会自动指向全局对象(浏览器中就是window对象)，同时值为原始值(数字，字符串，布尔值)的this会指向该原始值的自动包装对象。
arg1, arg2, ... 指定的参数列表。

#### bind

`func.bind(thisArg[, arg1[, arg2[, ...]]])`
thisArg 当绑定函数被调用时，该参数会作为原函数运行时的this指向。当使用new 操作符调用绑定函数时，该参数无效。
arg1, arg2, ... 当绑定函数被调用时，这些参数将置于实参之前传递给被绑定的方法。
  
### 5. 箭头函数

箭头函数比较特殊，它没有自己的this。它使用封闭执行上下文(函数或是global)的 this 值。箭头函数始终是匿名的。

ES6规定，箭头函数会继承外层函数，调用的 this 绑定（ 无论 this 绑定到什么），没有外层函数，则是绑定到全局对象（浏览器中是window）。这其实和 ES5 代码中的 `self = this` 机制一样。

``` js
GLOBAL.a = 'global aa';

var testObj = {
    a: 'aa',
    getValArrowFuc: function() {
        var val = (() => this.a);
        return val();
    },                                 
    getVal: function() {
        var self = this;
        var val = function() {
            return self.a;
        };
        return val();
    },
    getValGlobal: function() {
        var val = function() {
            return this.a;
        };
        return val();
    }
};

console.log(testObj.getValArrowFuc());          // aa
console.log(testObj.getVal());                  // aa
console.log(testObj.getValGlobal());            // global aa
```

---

## 附录一：执行上下文的示例分析

以下面的JS程序例子为例。

```js
const scope = 'global scope';

function checkscope() {
    var scope2 = 'local scope';
    return scope2;
}

checkscope();
```

其实际的执行过程如下：

1. checkscope 函数被创建，保存作用域链到内部属性 [[Scopes]]

    ```js
    checkscope.[[Scopes]] = [
        globalContext.VO
    ];
    ```

2. 执行 checkscope 函数，创建 checkscope 函数执行上下文，checkscope 函数执行上下文被压入执行上下文栈

    `ECStack = [checkscopeContext, globalContext];`

3. checkscope 函数并不立刻执行，开始做准备工作，首先复制函数 [[Scopes]] 属性创建作用域链

   ```js
    checkscopeContext = {
        Scopes: checkscope.[[Scopes]],
    }
    ```

4. 用 arguments 创建活动对象，随后初始化活动对象，加入形参、函数声明、变量声明

    ``` js
    checkscopeContext = {
        AO: {
            arguments: {
            length: 0
            },
            scope2: undefined
        }，
        Scopes: checkscope.[[Scopes]],
    }
    ```

5. 将活动对象压入 checkscope 作用域链顶端

    ```js
    checkscopeContext = {
        AO: {
            arguments: {
            length: 0,
            },
            scope2: undefined,
        },
        Scopes: [AO, [[Scopes]]],
    };
    ```

6. 准备工作做完，开始执行函数，随着函数的执行，修改 AO 的属性值

    ```js
    checkscopeContext = {
        AO: {
            arguments: {
            length: 0,
            },
            scope2: 'local scope',
        },
        Scopes: [AO, [[Scopes]]],
    };
    ```

7. 查找到 scope2 的值，返回后函数执行完毕，函数上下文从执行上下文栈中弹出

    `ECStack = [globalContext];`

---

## 参考文献

- [JavaScript Guidebook](https://tsejx.github.io/javascript-guidebook/core-modules/executable-code-and-execution-contexts/execution/execution-context-stack)
- [JavaScript的this原理 - 阮一峰](https://www.ruanyifeng.com/blog/2018/06/javascript-this.html)
- [深入理解 JavaScript 执行上下文和执行栈](https://blog.fundebug.com/2019/03/20/understand-javascript-context-and-stack/)
- [JavaScript的this指向详解](https://juejin.cn/post/6981251280236707853)
- [面试官问：JS的this指向](https://juejin.cn/post/6844903746984476686)
- [bind、call、apply的区别](https://segmentfault.com/a/1190000016705780)
- [Exploring Javascript's eval() Capabilities And Closure Scoping](https://www.bennadel.com/blog/1926-exploring-javascripts-eval-capabilities-and-closure-scoping.htm)
