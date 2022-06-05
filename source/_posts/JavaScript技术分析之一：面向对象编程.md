---
title: JavaScript技术分析之一：面向对象编程
date: 2022-06-04 16:44:07
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

### 基于类的OOP语言

大多数的OOP编程语言采用基于类（class-based）的模式，也称为经典模式，包括Java、Python、C++等，例如Java的开发过程完全基于Class，每个jar包就是一个完整的类定义，其核心理念是：

- 强调类和实例是两种完全不同的实体，通过类来描述实例对象应该具有哪些状态和行为
- 类是一个模板，对象是一个实例，必须首先定义类，然后才能根据模板完成相应对象的创建工作
- 类与类之间形成了继承、组合等关系

### 基于原型的OOP语言

然而，少数语言坚持采用基于原型（prototype-based）的模式，主要是JavaScript、Lua、Perl等，不区分类和实例的概念，而是通过原型（prototype）来实现面向对象编程。其强调程序员应关注一系列对象实例的行为，并将这些对象划分为使用方式相似的原型对象，而不是努力去将实体抽象为难以理解的Class。

- 强调原型对象(prototypical object)，即不是根据“模板”，而是通过“复制”一个已经存在的对象（原型）来创建另一个新对象
- 任何对象都可以作为另一个对象的原型，从而允许后者共享前者的属性，但只允许单继承
- 强调动态属性和方法，任何对象都可以指定其自身的属性，既可以是创建时也可以在运行时修改

### 对比分析

|基于类的（Java）|基于原型的（JavaScript）|
|-|-|
|类和实例是不同的实体,通过类定义来定义类，通过构造器方法来实例化类|所有对象都是实例，无需单独的类定义，通过构造器函数来定义和创建一组对象|
|通过类定义来定义现存类的子类，从而构建对象的层级结构|指定一个对象作为原型，并且与构造函数一起构建对象的层级结构|
|支持多重继承，遵循**类链**继承属性|仅支持单继承，遵循**原型链**继承属性|
|类定义指定类的所有实例的所有属性，不允许运行时动态添加属性|构造器函数或原型指定实例的初始属性集，允许动态地向单个对象或者整个对象集中添加或移除属性|
|通过`new`操作符创建单个对象|相同|

## 二、原型链的实现原理

早期的JavaScript支持`Object`对象，但根本没有类的概念，也不支持`Class`关键字，实现类定义的唯一途径就是通过函数来**模拟**实现。
以ECMAScript 5为例，定义一个类就等同于定义一个构造函数，实现继承关系就是把一个对象(函数)的原型指向另一个对象(函数)，依次延展从而构成一条原型链。

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

实现原型链继承的关键是`__proto__`（隐式原型）和 `[[prototype]]`（显示原型）,JavaScript正是通过两者的合作实现了原型链，以及对象的继承。

> Every JavaScript object has a second JavaScript object (or null ,but this is rare) associated with it.
> This second object is known as a prototype, and the first object inherits properties from the prototype.

### 隐式原型：`__proto__`

- 每个对象都有的一个属性，指向当前对象的原型对象
- 当访问一个对象的属性时，如果对象内部不存在该属性，那么就通过`__proto__`去原型对象中寻找该属性，并一直循环下去，这就是**原型链**的概念

### 显式原型：`[[prototype]]`

- 函数作为一类特殊对象，在创建时将自动添加`[[prototype]]`属性
- `[[prototype]]`指向一个拥有`constructor`属性的对象（指回该原型的构造函数），如果该函数作为构造函数被调用时（即通过new关键字调用），将自动创建该构造函数的实例
- `[[prototype]]`同样拥有`__proto__`属性，指向上一层的对象原型，用于添加可继承的方法和属性

实际上，JavaScript规定：任意函数对象都必须拥有内置属性`[[Prototype]]`，并据此实现原型链，这也是其被称为**显式原型**的原因。

但是，该规范要求并没有定义**如何实现**以及**如何访问**这个内置属性，而是由各个浏览器自行负责技术实现，大多数浏览器的解决方案都是设置`__proto__`属性用于访问`[[Prototype]]`属性，但这并不是统一标准，这也是其被称为**隐式原型**的原因。

后来，EMCAScript对此进行了进一步规范：

- ES5 定义了`Object.getPrototypeOf`方法，可以获得一个对象的`[[Prototype]]`属性
- ES6 定义了`Object.setPrototypeOf`方法，可以直接修改一个对象的`[[Prototype]]`属性

### 通过`new`创建对象

EMCAScript 5提供了`new`关键字，用于创建对象实例，其主要步骤参见myNew的伪代码：

``` js
function myNew(Fn, ...param) {
    // 1. 创建一个空对象{}
    var obj = new Object();
    // 2. 设置新对象的原型链指向obj
    obj.__proto__ = Fn.prototype;
    // 3. 将新对象作为`this`指向obj，并调用其构造函数
    // 注意call()调用，并支持构造函数的传参
    var result = Fn.call(obj, ...param);
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

### 原型链的示例

以person1为例，我们来看看原型链是如何实现的：

``` txt
person1 ---> Person.Prototype ---> Object.Prototype ---> null
```

![原型链的全貌](class5.jpg)

![prototype示例](prototype.png)

- `name`：person1拥有的实例属性
- `sayName`：person1拥有的实例方法
- `[[Prototype]]`：是person1的原型链，指向实例的原型对象`Person.Prototype`

进一步，我们来分析原型对象`Person.Prototype`的内容：

- `constructor`：就是一个指针，指回该原型的构造函数`Person.constructor`
- `[[Prototype]]`：还是一个原型链，指向该原型的上一层原型`Object.[[Prototype]]`

换一个角度，我们来看看person1实例的表现形式。奇怪的是，我们可以看到`name`属性和`sayName`方法，但是`[[Prototype]]`在哪里呢？

![实例的属性](instance.png)

``` js
Object.getPrototypeOf(person1) === Person.prototype     // true
preson1.prototype                                       // undefined
person1.__proto__ === Person.prototype                  // true
person1.__proto__.__proto__ === Person.prototype        // false
person1.__proto__.__proto__ === Object.prototype        // true
```

## 三、原型继承（ES5）

基于类的面向对象编程中，类继承(Inherit)是非常重要的特征，其本质是子类对父类定义的扩展。

### 实例在前，继承在后

由于ES5不支持`Class`关键字，这就意味着根本不存在类定义，那如何实现类定义的扩展呢？还是老办法，通过函数来**模拟**。
解决方案：先创造一个独立的子类的实例对象，然后再将父类的方法添加到这个对象上面，即：**实例在前，继承在后**。

具体来说，原型继承有两个核心问题要解决：一是如何将父类的实例属性传递给子类，也就是构造函数继承问题；二是如何将父类原型的方法传递给子类？也就是原型链继承问题。

### 原型链继承方案

核心思想：子类的原型指向父类的一个实例

```js
Child.prototype = new Parent();
```

优点：由于方法定义在父类的原型上，可以直接复用父类构造函数中的方法
缺点：创建子类实例的时候，不能传参数；由于子类实例共享了父类构造函数的引用属性，不同子类实例的引用属性可能互相污染

### 构造函数继承方案

核心思想：借用父类的构造函数来增强子类实例，等于是复制父类的实例属性给子类

``` js
function Child(name) {
    Parent.call(this,name);
}
```

优点：创建子类实例，可以向父类构造函数传参数；不同子类实例之间独立，引用属性不存在污染
缺点：子类实例无法继承父类原型的方法

### 完美组合方案

由于两种基本方法都存在缺陷，随后提出了组合继承、寄生继承等不同优化方案，最后整合为本完美组合的方案，这也是ES5的推荐方案。

![原型继承](extends-es5.png)

- 采用组合继承方案，将发生两次父类构造函数的调用，一是子类构造函数的`Parent.call(this,name,like)`，二是创建父类实例`Child.prototype = new Parent()`，虽然内容一致不会报错，但是冗余代码影响执行效率，也不优美！
- 采用优化的组合继承方案，通过`Child.prototype = Parent.prototype`替换`Child.prototype = new Parent()`，虽然不再产生二次构造函数调用，但是子类的原型被强制指向父类的原型，造成实质上子类原型和父类原型是同一个，必须手工进行修正
- 完美组合方案进行了进一步优化，解决方案是通过`Object.create()`方法通过一个桥接的空函数，实现父类原型和子类原型的分离，具体参见附录三

### 原型继承的示例

```js
function Parent(name) {
    this.name = name; // 实例基本属性 (该属性，强调私有，不共享)
    this.arr = [1]; // (该属性，强调私有)
}

Parent.prototype.say = function() { // --- 将需要复用、共享的方法定义在父类原型上 
    console.log('hello')
}

function Child(name,like) {
    // 核心！通过构造函数继承父类实例的属性和方法
    Parent.call(this,name,like) 
    this.like = like;
}

// 核心！通过原型链继承父类原型的属性和方法，并通过构造中间对象隔离子类原型和父类原型
Child.prototype = Object.create(Parent.prototype);

// 关键！修复构造函数指向的代码
Child.prototype.constructor = Child

let boy1 = new Child('小红','apple')
let boy2 = new Child('小明','orange')
let p1 = new Parent('小爸爸')
```

## 四、Class继承（ES6）

### 继承在前，实例在后

尽管ES5可以基本准确地实现类继承，但是需要编写大量代码，而且难以理解，为此ES6正式引入了`Class`关键字，并通过`extends`关键字实现继承。
由于ES6可以支持类定义，其继承机制就调整为：先将父类的属性和方法，加到一个空的对象上面，然后再将该对象作为子类的实例，即：**继承在前，实例在后**。

- 一个类必须有`constructor()`方法，如果没有显式定义，一个空的`constructor()`方法会被默认添加
- 子类必须在`constructor()`方法中调用`super()`，否则新建实例会报错就会报错
- 通过`new`命令生成对象实例时，自动调用其构造方法
- 类的所有方法都定义在类的原型上，在类的内部定义方法不用加`function`关键字

> 子类的构造函数通过继承父类的this对象，并对其进行加工，因此如果不调用`super()`方法，子类就得不到this对象

### 两条继承链

Class 作为构造函数的语法糖，类定义同时拥有`prototype`属性和`__proto__`属性，并由此构造了两条继承链，分别用于属性继承和方法继承。
`extends`关键字不仅可以用来继承类，还可以用来继承原生的构造函数。因此可以在原生数据结构的基础上，定义自己的数据结构

- 子类的__proto__属性：表示构造函数的继承，总是指向父类
- 子类prototype属性的__proto__属性：表示方法的继承，总是指向父类的prototype属性

需要注意的是，ES6的`Class`关键字并未修改底层设计，实际就是一个语法糖，所有功能ES5都可以实现，只是更加清晰和方便了！

![Class继承](extends-es6.png)

### Class继承的示例

```js
class Parent {
    constructor(name) {
       this.name = name;
       this.arr = [1];
    }
    say() {
        console.log('hello')
    }
}

class Child extends Parent {
    constructor(name, like) {
        super(name);
        this.like = like;
    }
}

let boy1 = new Child('小红','apple')
let boy2 = new Child('小明','orange')
let p1 = new Parent('小爸爸')
```

可以通过以下测试检查Class继承的准确性。

``` js
boy1.__proto__ === Child.prototype                  // true  

Child.__proto__ === Parent                          // true
Child.prototype.__proto__ === Parent.prototype      // true

Object.getPrototypeOf(boy1) === Child.prototype     // true
Object.getPrototypeOf(Child) === Parent             // true
```

---

## 附录一：Object和Function，先有鸡还是先有蛋？

JavaScript语言的数据类型由基本类型和对象类型两类组成。

1. 基本类型：直接表示在语言底层的不可变数据，也称为值类型，包括：

   - String：字符串类型
   - Number：数字类型
   - Boolean：布尔类型
   - Null：空类型
   - Undefined：未定义类型
   - BigInt：大整数类型
   - Symbol：符号类型，表示独一无二的值，ES6引入

2. 对象类型：一种无序的集合数据类型，由若干键值对(Key : Value)组成，用于描述现实世界某个对象的一组属性。

    - Object：对象类型，这是最基本的对象类型
    - Function：函数类型，由Object构造而来，最重要的一等公民
    - 数组（Array）、正则（RegExp）、日期（Date）等都是Object衍生的对象类型

### 万物始祖 - Object

在JavaScript的世界里，万物皆对象！
`Object.[[Prototype]]`就是万物的始祖，继承于虚无。
Object是对象的始祖，但也是一个函数。

```js
Object.prototype.__proto__ === null             // true
Object.__proto__ === Function.prototype         // true
```

### 一等公民 - Function

在JavaScript的世界里，函数是一等公民！
`Function.[[Prototype]]`是函数的原型，也是对象，同样源自于`Object.[[Prototype]]`
`Object.[[Prototype]]`的具体实现`Object.__proto__`，指向函数的原型`Function.[[Prottype]]`
`Function.[[Prototype]]` 是个特殊的函数对象，它忽略参数总是返回 undefined，且没有 `[[Construct]]`内部方法
通过`Function.prototype.bind`方法构造出来的函数是个例外，它没有`[[Prototype]]`属性

![Object和Function的关系](Object-Function3.jpg)

```js
Function.prototype.__proto__ === Object.prototype       // true
Object.prototype === Function.prototype                 // false
Object.__proto__ === Function.prototype                 // true
Object.__proto__ === Function.__proto__                 // true
```

最后总结一下：

- `Object.prototype`是原型链的顶端，`Function.prototype`继承`Object.prototype`而产生
- `Function`、`Object`以及其它构造函数，都是继承`Function.prototype`而产生

## 附录二：判断原型和实例之间关系的Object内置方法

`Object.[[prototype]]`内置了几个方法，用于判断原型和实例的关系，也可以验证以上实现原理。

- `instanceof`：这个操作符用于判断对象和原型之间的关系

    ```js
    person1 instanceof Person               // true
    person1.__proto__ === Person.prototype  // true，两种方式等价
    ```

- `isPrototypeOf`：如果`[[prototype]]`指向调用此方法的对象，那么这个方法就会返回true

    ```js
    Person.prototype.isPrototypeOf(person1) // true
    person1.__proto__ === Person.prototype  // true，两种方式等价
    ```

- `Object.getPrototypeOf`：这个方法返回`[[Prototype]]`的值,可以获取到一个对象的原型

    ``` js
    Object.getPrototypeOf(person1) === Person.prototype // true
    person1.__proto__ === Person.prototype  // true，两种方式等价
    ```

## 附录三：Object.create()方法

2006年，为了完美解决类继承的问题，javascript之父道格拉斯提出，借助一个空函数作为中间对象来实现正确的原型链，示例代码为：

``` js
function object(o) {
    function F() { };   // 定义一个空函数
    F.prototype = o;    // 指定空函数的原型为父类
    return new F();     // 创建并返回该空函数的实例，作为类继承的中间对象
}
```

EMCAScript 5实现了道爷的奇思妙想，并将上述代码封装为Object.create()方法。

`Object.create(proto[, propertiesObject])`

- proto：必填参数，指定新对象的原型对象。注意，如果是null，那新对象就是一个彻底的空对象，没有继承Object.prototype上的任何属性和方法，如hasOwnProperty()、toString()等
- propertiesObject：可选参数，指定要添加到新对象上的可枚举的属性（即其自定义的属性和方法，可用hasOwnProperty()获取的，而不是原型对象上的）的描述符及相应的属性名称

---

### 参考文献

- [JavaScript对象模型设计 - Mozilla官方](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Details_of_the_Object_Model)
- [继承与原型链 - Mozilla官方](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [ES6篇 - class 基本语法](https://wangjintian.com/2021/04/18/ES6%E7%AF%87-class%E5%9F%BA%E6%9C%AC%E8%AF%AD%E6%B3%95/)
- [原型继承和Class继承 - 廖雪峰](https://www.liaoxuefeng.com/wiki/1022910821149312/1023021997355072)
- [一文读懂JS中类、原型和继承](https://xieyufei.com/2020/04/10/Js-Class-Inherit.html)
- [基于类 vs 基于原型](https://blog.csdn.net/prehistorical/article/details/53671415)
- [JavaScript基于原型的面向对象编程](https://oychao.github.io/2016/11/28/javascript/21_oop/)
- [JavaScript对象模型与原型体系](https://chenzhuo1024.github.io/tech/js/js-oo.html)
- [js中__proto__和prototype的区别和关系？](https://www.zhihu.com/question/34183746/answer/58155878)
- [JS中的构造函数、原型、原型链](https://segmentfault.com/a/1190000022776150)
- [从__proto__和prototype来深入理解JS对象和原型链](https://github.com/creeperyang/blog/issues/9)
- [JS function 是函数也是对象, 浅谈原型链](https://www.yixuebiancheng.com/article/74630.html)
- [js继承的5种方法](https://segmentfault.com/a/1190000015216289)
- [一文读懂JS中类、原型和继承](https://xieyufei.com/2020/04/10/Js-Class-Inherit.html)
- [ES5/ES6 的继承除了写法以外还有什么区别](https://www.jianshu.com/p/1aa2755171fe)
- [JS继承 -> ES6的class和decorator](https://www.jianshu.com/p/59e6dca643ad)
- [ES5/ES6 的继承除了写法以外还有什么区别?](https://www.jianshu.com/p/6726623123a7)