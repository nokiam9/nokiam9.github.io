---
title: 深入理解Java虚拟机 - 以Android为例
date: 2021-11-14 15:56:01
tags:
---

## 一、JVM的基本概念

Java语言的最大优势就是**一次编译，处处运行**的特性，通过`javac`编译器将源代码编译成通用的中间形式——字节码，然后再由`java`解释器逐条将字节码解释为机器码来执行。尽管在性能上，Java远不如C++这类编译型语言，但由于良好的平台移植性而大行其道，其中的关键技术就是**Java虚拟机**。

### 1. 什么是JVM

`JVM（Java Virtual Machine）`是一台虚拟的计算机，本质上通过定义一组技术规范，在物理的计算机上模拟实现各种功能。
JVM屏蔽了与具体操作系统平台相关的信息，使Java程序只需生成在Java虚拟机上运行的字节码文件，就可以在多种平台上不加修改地运行。在执行字节码时，实际上最终还是把字节码解释成具体平台上的机器码并执行。

> `Bytecode`被称为字节码，是因为字节码文件由十六进制值组成，以两个十六进制值为一组，即以字节为单位进行读取(指令码 + 操作数)。
> 在Java中一般是用`javac`命令编译源代码为字节码文件。

{% asset_img jvm.jpeg %}

Java虚拟机包括一套字节码指令集、一组寄存器、一个栈、一个垃圾回收堆和一个存储方法域。详细资料请参见[jvm原理](https://cloud.tencent.com/developer/article/1556672)。

### 2. JVM的运行模式

以最常见的`Oracle JDK`为例，我们可以查看其版本信息如下：

``` console
bogon:~ sj$ java -version
java version "15.0.2" 2021-01-19
Java(TM) SE Runtime Environment (build 15.0.2+7-27)
Java HotSpot(TM) 64-Bit Server VM (build 15.0.2+7-27, mixed mode, sharing)
```

其中，JVM使用的引擎是`HotSpot`，基于`Server`版本，使用`Mixed mode`混合模式。

> Client模式采用轻量级的虚拟机，启动速度快，内存占用小
    Server模式采用重量级的虚拟机，对代码进行更多的优化，启动速度较慢，但运行速度比Client快得多
    64位版本默认采用Server模式。

### 3. JVM的编译模式

<style>
table th:first-of-type {
    width: 10%;
}
table th:nth-of-type(2) {
    width: 30%;
}
table th:nth-of-type(3) {
    width: 30%;
}
table th:nth-of-type(4) {
    width: 30%;
}
</style>

|  |前端编译 - 解释执行    |后端编译 - Just In Time|静态提前编译 - Ahead Of Time|
|:-|:-|:-|:-|
| 基本流程|通过JVM逐句解释执行字节码文件 | 首次采用解释方式运行，并将部分热点字节码编译成本地机器码| 程序运行前，直接把全部源代码编译成本地机器码|
| 优点|有利于实现泛型、内部类等动态语言特性；省去编译时间，启动速度快|执行效率高，支持各种层次的优化| 编译不占用运行时间，程序启动快；本地机器码保存磁盘，不占内存|
| 缺点| 代码运行效率几乎没有优化| 收集监控信息影响程序运行；编译过程使得启动速度变慢；编译机器码占用内存| 由于Java的动态语言特性，影响静态编译代码的质量|
| 典型产品| Oracle javac|HotSpot JVM的C1、C2编译器 | JAOTC、GCJ、Excelsior JET|

Oraclde JDK的默认引擎是`HotSpot`，其命名来自于它的混合模式执行模式（Mix Mode），也就是`Just In Time`模式：
- 起始阶段都是由解释器执行，解释器记录着每个方法的调用次数和循环次数，并以这两个数值为指标判断一个方法的“热度”
- 等到一个方法足够“热”的时候，就启动对该方法的编译。

> `JRockit`是原BEA公司开发的纯编译引擎，没有解释器。后来被Oracle公司收购后，逐步与开源产品OpenJDK融合。
> `Open J9`是IBM Ottawa实验室从`Small Talk`的虚拟机扩展来的一款JVM，后来贡献给 Eclipse 基金会。

### 4. JVM的结构体系

JVM 整体组成可分为以下四个部分：

- 类加载器（ClassLoader）
- 运行时数据区（Runtime Data Area）
- 执行引擎（Execution Engine）
- 本地库接口（Native Interface）

各个组成部分的用途：
1. 程序在执行之前，先要把java源代码转换成字节码（class文件）。
2. 首先，`JVM`需要把字节码文件通过`ClassLoader`，加载到内存中的`Runtime Data Area`
3. 由于字节码文件不能由底层操作系统执行，因此需要通过`Execution Engine` ，将字节码翻译成机器码指令
4. 物理CPU在执行机器码指令时，需要调用`Native Interface`（通常为C或C++语言开发的库函数）

{% asset_img java-2.png %}

## 二、Android的虚拟机技术演进

2008年9月，Android发布，Dalvik VM的执行引擎是只有解释器的；
2010年5月，Android 2.2发布，Dalvik VM引入了JIT编译器，JIT的引入使得Dalvik的性能提升了3～6倍；
2013年10月，Android 4.4发布，Dalvik和ART并存；
2014年10月，Android 5.0发布，ART取代了Dalvik成为了VM，同时AOT也成为了唯一的编译模式；单纯的使用JIT和AOT都是有缺点的，具体看JIT编译和AOT编译比较
2016年8月，Android 7.0发布，JIT编译器回归，形成了AOT/JIT混合编译模式，吸取了两者的优点同时克服了缺点。

### 1. Dalvik虚拟机

Dalvik是Google公司自己设计用于Android平台的虚拟机

Dalvik虚拟机是Google等厂商合作开发的Android移动设备平台的核心组成部分之一

它可以支持已转换为 .dex格式的Java应用程序的运行，.dex格式是专为Dalvik设计的一种压缩格式，适合内存和处理器速度有限的系统

Dalvik经过优化，允许在有限的内存中同时运行多个虚拟机的实例，并且每一个Dalvik 应用作为一个独立的Linux 进程执行

独立的进程可以防止在虚拟机崩溃的时候所有程序都被关闭

Dalvik是执行的时候编译+运行，安装比较快，开启应用比较慢，应用占用空间小

sa

Dalvik虚拟机是Google等厂商合作开发的Android移动设备平台的核心组成部分之一，它可以支持已转换为.dex格式的java应用程序的运行，.dex格式是专为Dalvik设计的一种压缩格式，适合内存和处理器速度有限的系统。
Dalvik经过优化，允许在有限的内存中同时运行多个虚拟机实例，并且每个Dalvik应用做为一个独立的Linux进程执行。独立的进程可以防止在虚拟机崩溃的时候所有程序都被关闭。
Dalvik早期是采用解释器执行dex字节码的，Android 2.2加入了JIT编译器，采用了解释器+JIT编译的方式，虽然性能提升了，可是每次启动应用都得动态编译，效率还是不是很高，Android 4.4引入了ART VM采用AOT编译模式（静态编译），5.0彻底抛弃了Dalvik，7.0采用了AOT+JIT混合编译模式。

DVM（Dalvik VM）与JVM的区别
DVM之所以不是一个JVM，主要原因是DVM并没有遵循JVM规范来实现，主要区别如下：

基于的架构不同
JVM基于栈实现的，意味着需要去栈中读写数据，所需的指令会更多，这样会导致速度慢，对于性能有限的移动设备，显然不是很合适。
DVM是基于寄存器的，它没有基于栈的虚拟机在拷贝数据而使用的大量的出入栈指令，同时指令更紧凑更简洁；但是，由于显示指定了操作数，所以基于寄存器的指令会比基于栈的指令要大，但是由于指令数量的减少，总的代码数不会增加多少。

执行的字节码不同
在Java SE中，Java类会被编译成一个或多个.class文件，打包成jar文件，而后JVM会通过相应的.class文件和jar文件获取字节码；执行顺序为：.java文件 -> .class文件 -> .jar文件。而DVM会用dx工具将所有的.class文件转换为一个.dex文件，然后DVM会从该.dex文件读取指令和数据；执行顺序为：.java文件 -> .class文件 -> .dex文件。
.jar文件里面包含多个.class文件，每个.class文件里面包含了该类的常量池、类信息、属性等等。当JVM加载该.jar文件的时候，会加载里面的所有的.class文件，JVM的这种加载方式很慢，对于内存有限的移动设备并不合适。 而在.apk文件中只包含了一个.dex文件，这个.dex文件里面将所有的.class里面所包含的信息全部整合在一起了，这样再加载就提高了速度。.class文件存在很多的冗余信息，dex工具会去除冗余信息，并把所有的.class文件整合到.dex文件中，减少了I/O操作，提高了类的查找速度。

通过上面的解释，我们知道DVM为了移动设备做了很多优化，这是因为移动设备有内存和处理器速度有限的特点。首先，架构变成了基于寄存器的，相应的指令集也进行了变化，指令个数变少了，而且对内存的读取变少了；然后，对由指令和数据组成的执行程序，也就是字节码，进行了编码和优化。

DVM允许在有限的内存中同时运行多个进程
DVM经过优化，允许在有限的内存中同时运行多个进程。在Android中的每一个应用都运行在一个DVM实例中，每一个DVM实例都运行在一个独立的进程空间。独立的进程可以防止在虚拟机崩溃的时候所有程序都被关闭。

难道JVM是一个进程运行多个应用的吗？

DVM由Zygote创建和初始化
Zygote可以称它为孵化器，它是一个DVM进程，同时它也用来创建和初始化DVM实例。每当系统需要创建一个应用程序时，Zygote就会fork自身，快速地创建和初始化一个DVM实例，用于应用程序的运行。

DVM架构
DVM的源码位于dalvik/目录下，其中dalvik/vm目录下的内容是DVM的具体实现部分，它会被编译成 libdvm.so；dalvik/libdex会被编译成libdex.a静态库，作为dex工具使用；dalvik/dexdump是.dex文件的反编译工具；DVM的可执行程序位于dalvik/dalvikvm中，将会被编译成dalvikvm可执行程序。

DVM的运行时堆
DVM的运行时堆主要由两个Space以及多个辅助数据结构组成，两个Space分别是Zygote Space（Zygote Heap） 和 Allocation Space（Active Heap）。Zygote Space用来管理Zygote进程在启动过程中预加载和创建的各种对象，Zygote Space中不会触发GC，所有进程都共享该区域，比如系统资源。Allocation Space是在Zygote进程fork第一个子进程之前创建的，它是一种私有进程，Zygote进程和fock的子进程在Allocation Space上进行对象分配和释放。

除了这两个Space，还包含以下数据结构：

Card Table： 用于DVM Concurrent GC，当第一次进行垃圾标记后，记录垃圾信息。 Heap Bitmap： 有两个Heap Bitmap，一个用来记录上次GC存活的对象，另一个用来记录这次GC存活的对象。 Mark Stack： DVM的运行时堆使用标记-清除（Mark-Sweep）算法进行GC，不了解标记-清除算法的同学查看Java虚拟机（四）垃圾收集算法这篇文章。Mark Stack就是在GC的标记阶段使用的，它用来遍历存活的对象。

### 2. ART虚拟机

ART即Android Runtime, ART 的机制与 Dalvik 不同

在Dalvik下，应用每次运行的时候，字节码都需要通过即时编译器（just in time ，JIT）转换为机器码，这会拖慢应用的运行效率

而在ART 环境中，应用在第一次安装的时候，字节码就会预先编译成机器码，使其成为真正的本地应用

这个过程叫做预编译（AOT，Ahead-Of-Time）

这样的话，应用的启动(首次)和执行都会变得更加快速

对比Dalvik，ART是安装的时候就编译好了，执行的时候直接就可以运行的，安装慢，开启应用快，占用空间大

AOT是”Ahead Of Time”的缩写，指的就是ART(Anroid RunTime)这种运行方式

前面介绍过，JIT是运行时编译，这样可以对执行次数频繁的dex代码进行编译和优化，减少以后使用时的翻译时间，虽然可以加快Dalvik运行速度，但是还是有弊病，那就是将dex翻译为本地机器码也要占用时间，所以Google在4.4之后推出了ART，用来替换Dalvik

在4.4版本上，两种运行时环境共存，可以相互切换，但是在5.0+，Dalvik虚拟机则被彻底的丢弃，全部采用ART

ART的策略与Dalvik不同，在ART 环境中，应用在第一次安装的时候，字节码就会预先编译成机器码，使其成为真正的本地应用。之后打开App的时候，不需要额外的翻译工作，直接使用本地机器码运行，因此运行速度提高

当然ART与Dalvik相比，还是有缺点的

ART需要应用程序在安装时，就把程序代码转换成机器语言，所以这会消耗掉更多的存储空间，但消耗掉空间的增幅通常不会超过应用代码包大小的20%

由于有了一个转码的过程，所以应用安装时间难免会延长

### 3. AOT/JIT混合编译模式

Android N 引入了一种包含编译、解释和 JIT（Just In Time）的混合运行时，以便在安装时间、内存占用、电池消耗和性能之间获得最好的折衷

Android N 包含了一个混合模式的运行时，应用在安装时不做编译，而是解释字节码，所以可以快速启动。ART 中有一种新的、更快的解释器，通过一种新的 JIT 完成，但是这种 JIT 的信息不是持久化的

取而代之的是，代码在执行期间被分析，分析结果保存起来。然后，当设备空转和充电的时候，ART 会执行针对“热代码”进行的基于分析的编译，其他代码不做编译

为了得到更优的代码，ART 采用了几种技巧包括深度内联

对同一个应用可以编译数次，或者找到变“热”的代码路径或者对已经编译的代码进行新的优化，这取决于分析器在随后的执行中的分析数据

这个步骤仍被简称为 AOT，可以理解为“全时段的编译”（All-Of-the-Time compilation）

这种混合使用 AOT、解释、JIT 的策略的全部优点如下

即使是大应用，安装时间也能缩短到几秒
系统升级能更快地安装，因为不再需要优化这一步
应用的内存占用更小，有些情况下可以降低 50%
改善了性能
更低的电池消耗

## 四、华为的方舟编译器

## 五、JVM的核心技术点

---

## 参考文献

### 官方网站

- [JSE（Java Platform Standard Edition） = JDK + JRE](https://docs.oracle.com/javase/7/docs/)
- [主流JVM的技术对比 - Wiki主页](https://en.wikipedia.org/wiki/Comparison_of_Java_virtual_machines)
- 《深入理解Java虚拟机：JVM高级特性与最佳实践》 周志明 著

### 技术研究

- [经典面试题|讲一讲JVM的组成](https://zhuanlan.zhihu.com/p/61916233)
- [一文让你搞懂各种虚拟机、解释器、JIT和AOT编译器](https://blog.csdn.net/zhongyili_sohu/article/details/106555297)
- [Java即时编译器原理解析及实践 - 美团技术](https://tech.meituan.com/2020/10/22/java-jit-practice-in-meituan.html)
- [关于Jvm知识看这一篇就够了](https://zhuanlan.zhihu.com/p/34426768)
