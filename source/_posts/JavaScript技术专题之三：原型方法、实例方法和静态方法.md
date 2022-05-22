---
title: JavaScript技术专题之三：原型方法、实例方法和静态方法
date: 2022-05-22 22:39:11
tags:
---

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
- [ES5/ES6 的继承除了写法以外还有什么区别](https://www.jianshu.com/p/1aa2755171fe)
- [基于类 vs 基于原型](https://blog.csdn.net/prehistorical/article/details/53671415)
- [JavaScript基于原型的面向对象编程](https://oychao.github.io/2016/11/28/javascript/21_oop/)
- [JavaScript对象模型与原型体系](https://chenzhuo1024.github.io/tech/js/js-oo.html)
- [Python - 实例方法、类方法、静态方法](https://zhuanlan.zhihu.com/p/40162669)
- [js中__proto__和prototype的区别和关系？](https://www.zhihu.com/question/34183746/answer/58155878)
- [JS中的构造函数、原型、原型链](https://segmentfault.com/a/1190000022776150)
- [JavaScript, ES6, ES7, ES10 where are we?](https://segmentfault.com/a/1190000021233366)
