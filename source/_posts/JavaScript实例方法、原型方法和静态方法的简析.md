---
title: JavaScript实例方法、原型方法和静态方法的简析
date: 2022-05-15 16:40:31
tags:
---

## 一、基于类 Vs 基于原型的OOP编程语言

面向对象的 OOP 编程语言（Object-oriented programming），Class和Instance是最基本概念。

- Class - 类
    类是抽象的，而不是其所描述的对象集合中的任何特定的个体。
    定义了某一对象集合所具有的共同特征，包含了存储数据的结构（Attribute属性）和操纵数据的行为（Method方法）
- Instance - 实例
    是一个类的实例化。例如， Victoria 是 Employee 类的一个实例，表示一个特定的雇员个体。
    实例具有和其父类完全一致的属性，不多也不少。

大多数的OOP编程语言采用基于类（class-based）的模式，主要是Java、Python、C++等。例如Java的开发过程完全基于Class，每个jar包就是一个完整的类定义，而将对象认为是class的实例化，即`对象（Object）== 实例（Instance）`。

然而，少数语言坚持采用基于原型（prototype-based）的模式，主要是JavaScript、Lua、Perl等，强调程序员应关注一系列对象实例的行为，并将这些对象划分到最近的使用方式相似的原型对象，而不是努力去将实体抽象为难以理解的Class。

- 强调原型对象(prototypical object)，将原型对象作为一个**模板**，新对象可以从中获得原始属性
- 强调动态属性和方法，任何对象都可以指定其自身的属性，既可以是创建时也可以在运行时修改
- 任何对象都可以作为另一个对象的原型，从而允许后者共享前者的属性，但只允许单继承

|基于类的（Java）|基于原型的（JavaScript）|
|-|-|
|类和实例是不同的实体,通过类定义来定义类，通过构造器方法来实例化类|所有对象都是实例，无需单独的类定义，通过构造器函数来定义和创建一组对象|
|通过类定义来定义现存类的子类，从而构建对象的层级结构|指定一个对象作为原型，并且与构造函数一起构建对象的层级结构|
|支持多重继承，遵循**类链**继承属性|仅支持单继承，遵循**原型链**继承属性|
|类定义指定类的所有实例的所有属性，不允许运行时动态添加属性|构造器函数或原型指定实例的初始属性集，允许动态地向单个对象或者整个对象集中添加或移除属性|
|通过`new`操作符创建单个对象|相同|

然而随着编程理念的不断发展，从ECMAScript 6开始引入了 Class（类）这个作为对象的模板，并提供了更接近Class-based语言的写法。
需要注意的是，ES6 的class可以看作只是一个语法糖，它的绝大部分功能，ES5 都可以做到，新的class写法只是让对象原型的写法更加清晰、更像面向对象编程的语法而已。

## 二、原型链继承的表现形式

下面，我们来看看JavaScript是如何实现原型链的继承关系的。
在ECMAScript 5之前，JS根本就不支持`Class`关键字，定义一个类就等同于定义一个函数。

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

早期的JavaScript规定：任意对象都必须拥有内置属性`[[Prototype]]`，并据此实现原型链,但这是一个标准要求，并没有规定**如何实现**以及**如何访问**这个内置属性，具体实现方式由各个浏览器自行负责。
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

废话不说，先放大图！！！

![原型链的全貌](link.jpeg)

- 在JavaScript的世界里，万物皆对象！
    `Object.[[prototype]]`就是万物的始祖，继承于虚无。

    ```js
    Object.prototype.__proto__ === null                 // true
    ```

- 在JavaScript的世界里，函数是一等公民！
    `Function.[[prototype]]`是函数的原型，也是对象，同样源自于`Object.[[prototype]]`
    `Object.[[prototype]]`的具体实现`Object.__proto__`，指向函数的原型`Function.[[prottype]]`
    `Function.[[prototype]]` 是个特殊的函数对象，它忽略参数总是返回 undefined，且没有 `[[Construct]]`内部方法
    通过`Function.prototype.bind`方法构造出来的函数是个例外，它没有`[[prototype]]`属性

    ```js
    Function.prototype.__proto__ === Object.prototype       // true
    Object.prototype === Function.prototype                 // false
    Object.__proto__ === Function.prototype                 // true
    Object.__proto__ === Function.__proto__                 // true
    ```

有了以上基本概念，我们来看看通过`new` 实例化时，发生了哪些事情：

1. 基于原型对象`Object.[[prototype]]`创建一个空对象,
   由此获得了`Object.[[prototype]]`的全部原型方法和内部方法，包括`isPrototypeOf`、`__defineGetter__`, `__defineSetter__`......
2. 将新对象的`__proto__`指向构造函数Person的原型对象`Person.[[prototype]]`，
   由此获得了Person的构造函数`Person.constructor`
3. 将新对象作为`this`去调用构造函数`Person.constructor`，
   从而设置新对象的`this.name`实例属性和`this.sayName`实例方法，
   初始化结束。

``` js
person1.__proto__ === Person.prototype                  // true
person1.__proto__.__proto__ === Object.prototype        // true

person1.constructor === Person.prototype.constructor    // true
person1.constructor === Person.__proto__.constructor    // false
```

以上步骤完成后，新对象实例就与其原型`Person`再无联系，这个时候即使其原型`Person`后续增加了成员属性，都不再影响已经实例化的新对象了。


## 二、EMCAScript 6 的原型方法、实例方法和静态方法

```js
// Person类
class Person {
    // 构造函数
    constructor(name) {
        this.name = name;
    }

    // 原型方法
    sayName() {
        return this.name;
    }
}

var person1 = new Person('xyf1')
var person2 = new Person('xyf2')
//true
console.log(person1.constructor === Person)
```

而
那么什么是静态方法呢？我们在使用 jQuery 的时候，往往会使用一些构造函数直接调用，而非通过实例调用的方法。例如 $.each，$.ajax，$.post，$.get 等等方法，这些方法

### 实例方法

构造函数中的this指向的是新创建的实例。因为在this上添加方法，其实是在往新创建的实例上添加属性与方法，所以称之为实例方法。
实例方法只有实例才能访问到，无法通过类直接调用。

### 静态方法

直接挂载到构造函数上面，我们称之 静态方法。
静态方法不能通过实例访问，只能通过构造函数来访问，不能通过实例调用

### 原型方法

通过 prototype 添加的方法，将会挂载到原型对象上，因此称之为 原型方法
原型中的方法实例和构造函数都可以访问到

### 结论

|方法类别|是否可以被构造函数调用|是否可以被实例化对象调用|
|-|-|-|
|静态方法|可以|不可以|
|实例方法|不可以|可以|
|原型方法|不可以|可以|

简而言之，实例方法就是只有实例可以调用，静态方法只有构造函数可以调用，原型方法是实例和构造函数都可以调用，是共享的方法。
像Promise.all和Promise.race这些就是静态方法，Promise.prototype.then这些就是原型方法，new 出来的实例可以调用

![原型方法](prototype.jpg)
![ES6的继承关系](extends-es6.png)

## 三、Python 的类方法、实例方法和静态方法

## 四、Java 的实例方法、类方法和静态方法

``` java
// 主程序入口
public class Main {
    public static void main(String[] args) {
        // 通过new实例化
        Person p = new Person("Xiao Ming", 15);
        // 调用实例方法
        System.out.println(p.getName());
        System.out.println(p.getAge());
    }
}

//  定义Class
class Person {
    // 定义Attribute属性
    private String name;
    private int age;

    // 同名构造函数
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // 定义Method方法
    public String getName() {
        return this.name;
    }

    public int getAge() {
        return this.age;
    }
}
```

此外，Java等语言还衍生出了一种特殊的对象类型，即接口。

- Interface - 接口
    接口不是Class。类描述对象的属性和方法。接口则包含类要实现的方法。
    接口是一个抽象类型，是抽象方法的集合。一个类通过继承接口的方式来继承接口的抽象方法。
    接口无法被实例化，没有构造方法。接口不是被类继承了，而是要被类实现。

``` java
// 定义接口interface，包含2个方法
interface Person {
    void run();
    String getName();
}

// 具体Class负责提供interface方法的实现
class Student implements Person {
    private String name;

    // 构造函数
    public Student(String name) {
        this.name = name;
    }

    // 提供run()方法的实现
    @Override
    public void run() {
        System.out.println(this.name + " run");
    }

    // 提供getName()方法的实现
    @Override
    public String getName() {
        return this.name;
    }
}
```

---

## 附录一：用于判断原型和实例之间关系的Object内置方法

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

---

### 参考文献

- [JavaScript对象模型设计 - Mozilla官方](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Details_of_the_Object_Model)
- [Class 的基本语法](https://es6.ruanyifeng.com/#docs/class)
- [ES6篇 - class 基本语法](https://wangjintian.com/2021/04/18/ES6%E7%AF%87-class%E5%9F%BA%E6%9C%AC%E8%AF%AD%E6%B3%95/)
- [一文读懂JS中类、原型和继承](https://xieyufei.com/2020/04/10/Js-Class-Inherit.html)
- [基于类 vs 基于原型](https://blog.csdn.net/prehistorical/article/details/53671415)
- [JavaScript基于原型的面向对象编程](https://oychao.github.io/2016/11/28/javascript/21_oop/)
- [JavaScript对象模型与原型体系](https://chenzhuo1024.github.io/tech/js/js-oo.html)
- [Python - 实例方法、类方法、静态方法](https://zhuanlan.zhihu.com/p/40162669)
- [js中__proto__和prototype的区别和关系？](https://www.zhihu.com/question/34183746/answer/58155878)
