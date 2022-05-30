---
title: JavaScript技术专题一：基于原型的面向对象编程
date: 2022-05-30 22:19:32
tags:
---

## 一、面向对象编程（Object-oriented programming）

对于面向对象的 OOP 编程语言，最基本的概念就是是：类、实例、对象。

- Class - 类是对象的类型模板
    类是抽象的，而不是其所描述的对象集合中的任何特定的个体。
    类定义了某一对象集合所具有的共同特征，包含了存储数据的结构（Attribute属性）和操纵数据的行为（Method方法）
- Instance - 实例是根据类创建的对象
    对象就是实例，是一个类的实例化。例如， Victoria 是 Employee 类的一个实例，表示一个特定的雇员个体。
    实例具有和其父类完全一致的属性，不多也不少。

### 1. 基于类的OOP语言

大多数的OOP编程语言采用基于类（class-based）的模式，也称为经典模式，包括Java、Python、C++等，例如Java的开发过程完全基于Class，每个jar包就是一个完整的类定义，其核心理念是：

- 强调类和实例是两种完全不同的实体，通过类来描述实例对象应该具有哪些状态和行为
- 类是一个模板，对象是一个实例，必须首先定义类，然后才能根据模板完成相应对象的创建工作
- 类与类之间形成了继承、组合等关系

### 2. 基于原型的OOP语言

然而，少数语言坚持采用基于原型（prototype-based）的模式，主要是JavaScript、Lua、Perl等，不区分类和实例的概念，而是通过原型（prototype）来实现面向对象编程。其强调程序员应关注一系列对象实例的行为，并将这些对象划分为使用方式相似的原型对象，而不是努力去将实体抽象为难以理解的Class。

- 强调原型对象(prototypical object)，即不是根据“模板”，而是通过“复制”一个已经存在的对象（原型）来创建另一个新对象
- 任何对象都可以作为另一个对象的原型，从而允许后者共享前者的属性，但只允许单继承
- 强调动态属性和方法，任何对象都可以指定其自身的属性，既可以是创建时也可以在运行时修改

### 3. 小结

|基于类的（Java）|基于原型的（JavaScript）|
|-|-|
|类和实例是不同的实体,通过类定义来定义类，通过构造器方法来实例化类|所有对象都是实例，无需单独的类定义，通过构造器函数来定义和创建一组对象|
|通过类定义来定义现存类的子类，从而构建对象的层级结构|指定一个对象作为原型，并且与构造函数一起构建对象的层级结构|
|支持多重继承，遵循**类链**继承属性|仅支持单继承，遵循**原型链**继承属性|
|类定义指定类的所有实例的所有属性，不允许运行时动态添加属性|构造器函数或原型指定实例的初始属性集，允许动态地向单个对象或者整个对象集中添加或移除属性|
|通过`new`操作符创建单个对象|相同|

## 二、原型链继承的实现原理

JavaScript的原型链和Java的Class区别就在，它没有“Class”的概念，所有对象都是实例，所谓继承关系不过是把一个对象的原型指向另一个对象而已。
实现原型链继承的关键是`__proto__` 和 `[[prototype]]`，先说结论：

1. `__proto__`是每个对象都有的一个属性，而`[[prototype]]`是函数才会有的属性。
2. `__proto__`指向的是当前对象的原型对象，而`[[prototype]]`指向的，是以当前函数作为构造函数构造出来的对象的原型对象。

> Every JavaScript object has a second JavaScript object (or null ,but this is rare) associated with it. This second object is known as a prototype, and the first object inherits properties from the prototype.

先不解释为什么，请看大图！！！

![原型链的全貌](class5.jpg)

## 三、原型链继承的示例

下面，我们来看看JavaScript是如何实现原型链。
ECMAScript 5并不支持`Class`关键字，要定义一个类的唯一途径就是定义一个构造函数。

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
person1.__proto__ === Person.prototype                  // true
person1.__proto__.__proto__ === Person.prototype        // false
person1.__proto__.__proto__ === Object.prototype        // true
```

## 四、先有鸡还是先有蛋？

![Object和Function的关系](Object-Function3.jpg)

### 1. Object = 万物始祖

在JavaScript的世界里，万物皆对象！
`Object.[[Prototype]]`就是万物的始祖，继承于虚无。
Object是对象的始祖，但也是一个函数。

```js
Object.prototype.__proto__ === null             // true
Object.__proto__ === Function.prototype         // true
```

### 2. Function = 一等公民

在JavaScript的世界里，函数是一等公民！
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

---

### 参考文献

- [JavaScript对象模型设计 - Mozilla官方](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Details_of_the_Object_Model)
- [继承与原型链 - Mozilla官方](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [ES6篇 - class 基本语法](https://wangjintian.com/2021/04/18/ES6%E7%AF%87-class%E5%9F%BA%E6%9C%AC%E8%AF%AD%E6%B3%95/)
- [一文读懂JS中类、原型和继承](https://xieyufei.com/2020/04/10/Js-Class-Inherit.html)
- [基于类 vs 基于原型](https://blog.csdn.net/prehistorical/article/details/53671415)
- [JavaScript基于原型的面向对象编程](https://oychao.github.io/2016/11/28/javascript/21_oop/)
- [JavaScript对象模型与原型体系](https://chenzhuo1024.github.io/tech/js/js-oo.html)
- [js中__proto__和prototype的区别和关系？](https://www.zhihu.com/question/34183746/answer/58155878)
- [JS中的构造函数、原型、原型链](https://segmentfault.com/a/1190000022776150)
- [从__proto__和prototype来深入理解JS对象和原型链](https://github.com/creeperyang/blog/issues/9)
- [JS function 是函数也是对象, 浅谈原型链](https://www.yixuebiancheng.com/article/74630.html)
