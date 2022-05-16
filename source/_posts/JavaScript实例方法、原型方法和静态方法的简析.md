---
title: JavaScript实例方法、原型方法和静态方法的简析
date: 2022-05-15 16:40:31
tags:
---

## 一、基于类 Vs 基于原型的编程语言

面向对象的 OOP 编程语言（Object-oriented programming），有几个共同的基本概念。

- Class - 类
    定义了某一对象集合所具有的特征性属性（可以将 Java 中的方法和域以及 C++ 中的成员都视作属性）
    类是抽象的，而不是其所描述的对象集合中的任何特定的个体。例如 Employee 类可以用来表示所有雇员的集合。
    类包含了存储数据的结构和操纵数据的行为，一般称之为称为属性（Attribute）和方法（Method）
- Instance - 实例
    是一个类的实例化。例如， Victoria 可以是 Employee 类的一个实例，表示一个特定的雇员个体。实例具有和其父类完全一致的属性，不多也不少。

大多数的OOP编程语言采用基于类（class-based）的模式，比如 Java、C++和Python等，其特点是非常明确将Class和Instance分成两种不同的实体。

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

然而，少数语言坚持采用基于原型（prototype-based）的模式，主要是JavaScript、Lua、Perl等，强调程序员应关注一系列对象实例的行为，并将这些对象划分到最近的使用方式相似的原型对象，而不是努力去将实体抽象为难以理解的Class。其特点是：

- 强调原型对象(prototypical object)的概念，将原型对象作为一个模板，新对象可以从中获得原始的属性
- 强调动态属性和方法，即：任何对象都可以指定其自身的属性，既可以是创建时也可以在运行时创建
- 任何对象都可以作为另一个对象的原型(prototype)，从而允许后者共享前者的属性。
- 通过原型只能实现单继承。

|基于类的（Java）|基于原型的（JavaScript）|
|:-:|:-:|
|类和实例是不同的事物|所有对象均为实例|
|通过类定义来定义类，通过构造器方法来实例化类|通过构造器函数来定义和创建一组对象|
|通过`new`操作符创建单个对象|相同|
|通过类定义来定义现存类的子类，从而构建对象的层级结构|指定一个对象作为原型并且与构造函数一起构建对象的层级结构|
|遵循类链继承属性|遵循原型链继承属性|
|类定义指定类的所有实例的所有属性，不允许运行时动态添加属性|构造器函数或原型指定实例的初始属性集，允许动态地向单个的对象或者整个对象集中添加或移除属性|

因此，在ECMAScript 5之前，JS根本就不支持`Class`关键字，定义一个类就等同于定义一个函数。

``` js
// Person类
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

然而随着编程理念的不断发展，从ECMAScript 6开始引入了 Class（类）这个作为对象的模板，并提供了更接近Class-based语言的写法。
需要注意的是，ES6 的class可以看作只是一个语法糖，它的绝大部分功能，ES5 都可以做到，新的class写法只是让对象原型的写法更加清晰、更像面向对象编程的语法而已。

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
