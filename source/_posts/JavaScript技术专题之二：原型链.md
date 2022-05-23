---
title: JavaScript技术专题之二：原型链
date: 2022-05-22 22:35:27
tags:
---

## 一、基于类 Vs 基于原型的OOP编程语言

面向对象的 OOP 编程语言（Object-oriented programming），类和实例是最基本概念。

- Class - 类
    类是抽象的，而不是其所描述的对象集合中的任何特定的个体。
    定义了某一对象集合所具有的共同特征，包含了存储数据的结构（Attribute属性）和操纵数据的行为（Method方法）
- Instance - 实例
    是一个类的实例化。例如， Victoria 是 Employee 类的一个实例，表示一个特定的雇员个体。
    实例具有和其父类完全一致的属性，不多也不少。

基于类的编程语言，

基于原型的编程语言，。

大多数的OOP编程语言采用基于类（class-based）的模式，包括Java、Python、C++等，例如Java的开发过程完全基于Class，每个jar包就是一个完整的类定义，而将对象认为是class的实例化，即`对象（Object）== 实例（Instance）`。

- 强调类和实例是两种完全不同的实体，通过类来描述实例对象应该具有哪些状态和行为
- 类是一个模板，对象是一个实例，必须首先定义类，然后才能根据模板完成相应对象的创建工作
- 类与类之间形成了继承、组合等关系

然而，少数语言坚持采用基于原型（prototype-based）的模式，主要是JavaScript、Lua、Perl等，强调程序员应关注一系列对象实例的行为，并将这些对象划分为使用方式相似的原型对象，而不是努力去将实体抽象为难以理解的Class。

- 强调原型对象(prototypical object)，创建一个新对象不是根据“模板”，而是通过“复制”另外一个已经存在的对象（原型）来创建一个新对象
- 任何对象都可以作为另一个对象的原型，从而允许后者共享前者的属性，但只允许单继承
- 强调动态属性和方法，任何对象都可以指定其自身的属性，既可以是创建时也可以在运行时修改

|基于类的（Java）|基于原型的（JavaScript）|
|-|-|
|类和实例是不同的实体,通过类定义来定义类，通过构造器方法来实例化类|所有对象都是实例，无需单独的类定义，通过构造器函数来定义和创建一组对象|
|通过类定义来定义现存类的子类，从而构建对象的层级结构|指定一个对象作为原型，并且与构造函数一起构建对象的层级结构|
|支持多重继承，遵循**类链**继承属性|仅支持单继承，遵循**原型链**继承属性|
|类定义指定类的所有实例的所有属性，不允许运行时动态添加属性|构造器函数或原型指定实例的初始属性集，允许动态地向单个对象或者整个对象集中添加或移除属性|
|通过`new`操作符创建单个对象|相同|

## 二、原型链继承的表现形式

下面，我们来看看JavaScript是如何实现原型链的继承关系的。
对于ECMAScript 5，JS并不支持`Class`关键字，要定义一个类的唯一途径就是定义一个构造函数。

``` js
// 定义Person类的构造函数
function Person(name) {
  // 实例属性
  this.name = name
  // 实例方法
  this.sayName = function() {
    return this.name
  }
}
var person1 = new Person('xyf1')
var person2 = new Person('xyf2')
//true
console.log(person1.constructor === Person)
```

以person1这个实例对象为例，我们来看看原型链的实现原理。

![prototype示例](prototype.png)

- `name`：person1拥有的实例属性
- `sayName`：person1拥有的实例方法
- `[[Prototype]]`：是person1的原型链，它是一个指针（也是一个对象），指向实例的原型对象`Person.[[Prototype]]`

进一步，我们来分析原型对象`Person.[[Prototype]]`的内容：

- `constructor`：就是一个指针，指回该原型的构造函数`Person.constructor`
- `[[Prototype]]`：还是一个原型链，指向该原型的上一层原型`Object.[[Prototype]]`

换一个角度，我们来看看person1实例的表现形式。奇怪的是，我们可以看到`name`属性和`sayName`方法，但是`[[Prototype]]`在哪里呢？

![实例的属性](instance.png)

早期JavaScript规定：任意对象都必须拥有内置属性`[[Prototype]]`，并据此实现原型链，但这是规范要求，并没有说明**如何实现**以及**如何访问**这个内置属性，而是由各个浏览器自行负责技术实现。
实际上，大多数浏览器都支持通过`__proto__`来访问`[[Prototype]]`属性，但这并不是统一标准。后来，EMCAScript对此进行了进一步规范，包括：

- ES5 定义了`Object.getPrototypeOf`方法，可以获得一个对象的`[[Prototype]]`属性
- ES6 定义了`Object.setPrototypeOf`方法，可以直接修改一个对象的`[[Prototype]]`属性

``` js
Object.getPrototypeOf(person1) === Person.prototype     // true
person1.__proto__ === Person.prototype                  //true
person1.__proto__.__proto__ === Person.prototype        // false
person1.__proto__.__proto__ === Object.prototype        // true
```

## 三、原型链继承的实现原理

实现原型链继承的关键是`__proto__` 和 `[[prototype]]`，先说结论：

1. `__proto__`是每个对象都有的一个属性，而`[[prototype]]`是函数才会有的属性。
2. `__proto__`指向的是当前对象的原型对象，而`[[prototype]]`指向的，是以当前函数作为构造函数构造出来的对象的原型对象。

> Every JavaScript object has a second JavaScript object (or null ,but this is rare) associated with it. This second object is known as a prototype, and the first object inherits properties from the prototype.

先不解释为什么，请看大图！！！

![原型链的全貌](class5.jpg)

### 1. new：创建对象实例

定义了class的构造函数之后，ES5 通过`new`关键字来创建对象实例，其主要步骤参见myNew的伪代码：

``` js
function myNew(Func, ...param) {
    // 1. 创建一个空对象{}
    var obj = new Object();
    // 2. 设置新对象的原型链指向obj
    obj.__proto__ = Func.prototype;
    // 3. 将新对象作为`this`指向obj，并调用其构造函数
    // 注意call()调用，并支持构造函数的传参
    var result = Func.call(obj, ...param);
    // 4. 将新对象作为返回值
    // 注意：需要判断Func的返回值类型：如果是值类型，返回obj；如果是引用类型，就返回这个引用类型的对象    
    return typeof result === 'object'          
        || typeof result === 'function' ? result : obj
}
```

以上步骤完成后，新对象实例就与其原型`Person`再无联系，这个时候即使其原型`Person`后续增加了成员属性，都不再影响已经实例化的新对象了。

``` js
person1.__proto__ === Person.prototype                  // true
person1.__proto__.__proto__ === Object.prototype        // true

person1.constructor === Person.prototype.constructor    // true
person1.constructor === Person.__proto__.constructor    // false
```

### 2. Object 和 Function：先有鸡还是先有蛋？

![Object和Function的关系](Object-Function3.jpg)

- 在JavaScript的世界里，万物皆对象！
    `Object.[[Prototype]]`就是万物的始祖，继承于虚无。
    Object是对象的始祖，但也是一个函数。

    ```js
    Object.prototype.__proto__ === null             // true
    Object.__proto__ === Function.prototype         // true
    ```

- 在JavaScript的世界里，函数是一等公民！
    `Function.[[Prototype]]`是函数的原型，也是对象，同样源自于`Object.[[Prototype]]`
    `Object.[[Prototype]]`的具体实现`Object.__proto__`，指向函数的原型`Function.[[Prottype]]`
    `Function.[[Prototype]]` 是个特殊的函数对象，它忽略参数总是返回 undefined，且没有 `[[Construct]]`内部方法
    通过`Function.prototype.bind`方法构造出来的函数是个例外，它没有`[[Prototype]]`属性

    ```js
    Function.prototype.__proto__ === Object.prototype       // true
    Object.prototype === Function.prototype                 // false
    Object.__proto__ === Function.prototype                 // true
    Object.__proto__ === Function.__proto__                 // true
    ```

关于两者之间的关系，这篇文章说的最详细[【一文读懂JS中类、原型和继承】](https://xieyufei.com/2020/04/10/Js-Class-Inherit.html)
还有一篇较为简短的分析[【JS function 是函数也是对象, 浅谈原型链】](https://www.yixuebiancheng.com/article/74630.html)

## 五、ES6 对于类继承的改进

![ES5的继承链](extends-es5.png)

ES5的继承实质上是先创建子类的实例对象，然后再将父类的方法添加到this上（Parent.apply(this)）.

es6继承

class继承，class之间使用extends关键字

子类必须要再constructor方法中调用super方法，否则新建实例会报错，这是因为子类没有自己的this对象，而是继承父类的this对象，然后对其进行加工，如果不调用super方法，子类就得不到this对象。

### 原型链继承和构造函数继承

- [js继承的5种方法](https://segmentfault.com/a/1190000015216289)
- [一文读懂JS中类、原型和继承](https://xieyufei.com/2020/04/10/Js-Class-Inherit.html)
- [ES5/ES6 的继承除了写法以外还有什么区别](https://www.jianshu.com/p/1aa2755171fe)
- [原型继承和Class继承 - 廖雪峰](https://www.liaoxuefeng.com/wiki/1022910821149312/1023021997355072)
- [JS继承 -> ES6的class和decorator](https://www.jianshu.com/p/59e6dca643ad)
- [ES5/ES6 的继承除了写法以外还有什么区别？ -2](https://www.jianshu.com/p/6726623123a7)

prototype 和 __proto__

一个继承语句同时存在于两条继承链：一条继承属性，一条实现方法的继承

1 class A extends B{}

2 A.__proto__ === B;//继承属性

3 A.prototype.__proto__ == B.prototype;//继承方法

区别：

随着编程理念的不断发展，从ECMAScript 6开始引入了 Class（类）这个作为对象的模板，并提供了更接近Class-based语言的写法。
需要注意的是，ES6 的class可以看作只是一个语法糖，它的绝大部分功能，ES5都可以做到，新的class写法只是让对象原型的写法更加清晰、更像面向对象编程的语法而已。

ES6的继承机制完全不同，实质上是先创建父类的实例对象this（所以必须先调用父类的super()方法），然后再用子类的构造函数修改this。

ES5的继承时通过原型或构造函数机制来实现。

ES6通过class关键字定义类，里面有构造方法，类之间通过extends关键字实现继承。子类必须在constructor方法中调用super方法，否则新建实例报错。因为子类没有自己的this对象，而是继承了父类的this对象，然后对其进行加工。如果不调用super方法，子类得不到this对象。

注意super关键字指代父类的实例，即父类的this对象。

注意：在子类构造函数中，调用super后，才可使用this关键字，否则报错。

![ES6的继承链](extends-es6.png)

---

### 参考文献

- [JavaScript对象模型设计 - Mozilla官方](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Details_of_the_Object_Model)
- [继承与原型链 - Mozilla官方](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [Class 的基本语法](https://es6.ruanyifeng.com/#docs/class)
- [ES6篇 - class 基本语法](https://wangjintian.com/2021/04/18/ES6%E7%AF%87-class%E5%9F%BA%E6%9C%AC%E8%AF%AD%E6%B3%95/)
- [一文读懂JS中类、原型和继承](https://xieyufei.com/2020/04/10/Js-Class-Inherit.html)
- [ES5/ES6 的继承除了写法以外还有什么区别](https://www.jianshu.com/p/1aa2755171fe)
- [基于类 vs 基于原型](https://blog.csdn.net/prehistorical/article/details/53671415)
- [JavaScript基于原型的面向对象编程](https://oychao.github.io/2016/11/28/javascript/21_oop/)
- [JavaScript对象模型与原型体系](https://chenzhuo1024.github.io/tech/js/js-oo.html)
- [Python - 实例方法、类方法、静态方法](https://zhuanlan.zhihu.com/p/40162669)
- [js中__proto__和prototype的区别和关系？](https://www.zhihu.com/question/34183746/answer/58155878)
- [JS中的构造函数、原型、原型链](https://segmentfault.com/a/1190000022776150)
- [从__proto__和prototype来深入理解JS对象和原型链](https://github.com/creeperyang/blog/issues/9)
