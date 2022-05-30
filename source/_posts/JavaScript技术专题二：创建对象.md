---
title: JavaScript技术专题二：创建对象
date: 2022-05-31 00:52:23
tags:
---

## 一、JavaScript的数据类型

JavaScript 语言中类型集合由原始值和对象组成。

### 1. 基本类型

基本类型是直接表示在语言底层的不可变数据，也称为值类型

- String：字符串类型
- Number：数字类型
- Boolean：布尔类型
- Null：空类型
- Undefined：未定义类型
- BigInt：大整数类型
- Symbol：符号类型

### 2. 对象类型

对象类型是一组属性的集合，也称为引用数据类型。

引用数据类型（对象类型）：对象(Object)、数组(Array)、函数(Function)，还有两个特殊的对象：正则（RegExp）和日期（Date）。
JavaScript对每个创建的对象都会设置一个原型，指向它的原型对象。

对象由花括号分隔。在括号内部，对象的属性以名称和值对的形式 (name : value) 来定义。属性由逗号分隔：

`var person={firstname:"John", lastname:"Doe", id:5566};`

当我们用obj.xxx访问一个对象的属性时，JavaScript引擎先在当前对象上查找该属性，如果没有找到，就到其原型对象上找，如果还没有找到，就一直上溯到Object.prototype对象，最后，如果还没有找到，就只能返回undefined。

例如，创建一个Array对象：

var arr = [1, 2, 3];
其原型链是：

``` js
arr ----> Array.prototype ----> Object.prototype ----> null
```

Array.prototype定义了indexOf()、shift()等方法，因此你可以在所有的Array对象上直接调用这些方法。

当我们创建一个函数时：

function foo() {
    return 0;
}

函数也是一个对象，它的原型链是：

foo ----> Function.prototype ----> Object.prototype ----> null
由于Function.prototype定义了apply()等方法，因此，所有函数都可以调用apply()方法。

很容易想到，如果原型链很长，那么访问一个对象的属性就会因为花更多的时间查找而变得更慢，因此要注意不要把原型链搞得太长。

## 一、constructor：构造函数

除了直接用{ ... }创建一个对象外，JavaScript还可以用一种构造函数的方法来创建对象。它的用法是，先定义一个构造函数：

function Student(name) {
    this.name = name;
    this.hello = function () {
        alert('Hello, ' + this.name + '!');
    }
}
你会问，咦，这不是一个普通函数吗？

这确实是一个普通函数，但是在JavaScript中，可以用关键字new来调用这个函数，并返回一个对象：

var xiaoming = new Student('小明');
xiaoming.name; // '小明'
xiaoming.hello(); // Hello, 小明!
注意，如果不写new，这就是一个普通函数，它返回undefined。但是，如果写了new，它就变成了一个构造函数，它绑定的this指向新创建的对象，并默认返回this，也就是说，不需要在最后写return this;。

新创建的xiaoming的原型链是：

xiaoming ----> Student.prototype ----> Object.prototype ----> null
也就是说，xiaoming的原型指向函数Student的原型。如果你又创建了xiaohong、xiaojun，那么这些对象的原型与xiaoming是一样的：

xiaoming ↘
xiaohong -→ Student.prototype ----> Object.prototype ----> null
xiaojun  ↗
用new Student()创建的对象还从原型上获得了一个constructor属性，它指向函数Student本身：

xiaoming.constructor === Student.prototype.constructor; // true
Student.prototype.constructor === Student; // true

Object.getPrototypeOf(xiaoming) === Student.prototype; // true

xiaoming instanceof Student; // true
看晕了吧？用一张图来表示这些乱七八糟的关系就是：

protos

红色箭头是原型链。注意，Student.prototype指向的对象就是xiaoming、xiaohong的原型对象，这个原型对象自己还有个属性constructor，指向Student函数本身。

另外，函数Student恰好有个属性prototype指向xiaoming、xiaohong的原型对象，但是xiaoming、xiaohong这些对象可没有prototype这个属性，不过可以用__proto__这个非标准用法来查看。

现在我们就认为xiaoming、xiaohong这些对象“继承”自Student。

不过还有一个小问题，注意观察：

xiaoming.name; // '小明'
xiaohong.name; // '小红'
xiaoming.hello; // function: Student.hello()
xiaohong.hello; // function: Student.hello()
xiaoming.hello === xiaohong.hello; // false
xiaoming和xiaohong各自的name不同，这是对的，否则我们无法区分谁是谁了。

xiaoming和xiaohong各自的hello是一个函数，但它们是两个不同的函数，虽然函数名称和代码都是相同的！

如果我们通过new Student()创建了很多对象，这些对象的hello函数实际上只需要共享同一个函数就可以了，这样可以节省很多内存。

要让创建的对象共享一个hello函数，根据对象的属性查找原则，我们只要把hello函数移动到xiaoming、xiaohong这些对象共同的原型上就可以了，也就是Student.prototype：

protos2

修改代码如下：

function Student(name) {
    this.name = name;
}

Student.prototype.hello = function () {
    alert('Hello, ' + this.name + '!');
};
用new创建基于原型的JavaScript的对象就是这么简单！

忘记写new怎么办
如果一个函数被定义为用于创建对象的构造函数，但是调用时忘记了写new怎么办？

在strict模式下，this.name = name将报错，因为this绑定为undefined，在非strict模式下，this.name = name不报错，因为this绑定为window，于是无意间创建了全局变量name，并且返回undefined，这个结果更糟糕。

所以，调用构造函数千万不要忘记写new。为了区分普通函数和构造函数，按照约定，构造函数首字母应当大写，而普通函数首字母应当小写，这样，一些语法检查工具如jslint将可以帮你检测到漏写的new。

最后，我们还可以编写一个createStudent()函数，在内部封装所有的new操作。一个常用的编程模式像这样：

function Student(props) {
    this.name = props.name || '匿名'; // 默认值为'匿名'
    this.grade = props.grade || 1; // 默认值为1
}

Student.prototype.hello = function () {
    alert('Hello, ' + this.name + '!');
};

function createStudent(props) {
    return new Student(props || {})
}
这个createStudent()函数有几个巨大的优点：一是不需要new来调用，二是参数非常灵活，可以不传，也可以这么传：

var xiaoming = createStudent({
    name: '小明'
});

xiaoming.grade; // 1
如果创建的对象有很多属性，我们只需要传递需要的某些属性，剩下的属性可以用默认值。由于参数是一个Object，我们无需记忆参数的顺序。如果恰好从JSON拿到了一个对象，就可以直接创建出xiaoming。



## 二、new：创建对象实例

定义了class的构造函数之后，ES5 通过`new`关键字来创建对象实例，其主要步骤参见myNew的伪代码：

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