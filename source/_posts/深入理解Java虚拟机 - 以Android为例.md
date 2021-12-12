---
title: 深入理解Java虚拟机 - 以Android为例
date: 2021-11-14 15:56:01
tags:
---

Java语言的最大优势就是**一次编译，处处运行**的特性，通过`javac`编译器将源代码编译成通用的中间形式——字节码，然后再由`java`解释器逐条将字节码解释为机器码来执行。尽管在性能上，Java远不如C++这类编译型语言，但由于良好的平台移植性而大行其道，其中的关键技术就是**Java虚拟机**。

## 一、什么是JVM

`JVM（Java Virtual Machine）`是一台虚拟的计算机，本质上通过定义一组技术规范，在物理的计算机上模拟实现各种功能。
JVM屏蔽了与具体操作系统平台相关的信息，使Java程序只需生成在Java虚拟机上运行的目标代码（字节码），就可以在多种平台上不加修改地运行。在执行字节码时，实际上最终还是把字节码解释成具体平台上的机器指令执行，属于用户态。
{% asset_img jvm.png %}

Java虚拟机包括一套字节码指令集、一组寄存器、一个栈、一个垃圾回收堆和一个存储方法域。详细资料请参见[jvm原理](https://cloud.tencent.com/developer/article/1556672)。

Java Community Process组织管理所有Java技术标准，主要途径是发布JSR - Java Specification Requests，Java虚拟机也不例外。
JVM涉及的核心技术规范包括：

- 2004年，发布的JVM核心规范[JSR 924 - Java Virtual Machine Specification](https://www.jcp.org/en/jsr/detail?id=924)
- 2006年，为修订Class文件标准而发布的[JSR 202 - Java Class File Specification Update](https://web.archive.org/web/20120226185155/http://www.jcp.org/en/jsr/detail?id=202)
- 完整的Java虚拟机描述，可以参见[JCP发布的JVM蓝皮书](https://web.archive.org/web/20110925050249/http://java.sun.com/docs/books/vmspec/2nd-edition/html/VMSpecTOC.doc.html)

## 二、JVM的运行模式

以最常见的`Oracle JDK`为例，我们可以查看其版本信息如下：

``` console
bogon:~ sj$ java -version
java version "15.0.2" 2021-01-19
Java(TM) SE Runtime Environment (build 15.0.2+7-27)
Java HotSpot(TM) 64-Bit Server VM (build 15.0.2+7-27, mixed mode, sharing)
```

其中，JVM使用的引擎是`HotSpot`，基于`Server`版本，使用`Mixed mode`混合模式。

> Server模式启动采用重量级的虚拟机，对代码进行更多的优化；而Client模式采用轻量级的虚拟机，启动速度更快。
    尽管Server启动慢，但稳定后的运行速度比Client远远要快。
    64位版本默认都采用Server模式。

`Bytecode`被称为字节码，是因为字节码文件由十六进制值组成，而`JVM`以两个十六进制值为一组，即以字节为单位进行读取。在Java中一般是用`javac`命令编译源代码为字节码文件，一个.java文件从编译到运行的示例如图1所示。

{% asset_img java-2.png %}

为了优化Java的性能 ，JVM在解释器之外引入了即时（Just In Time）编译器：当程序运行时，解释器首先发挥作用，代码可以直接执行。随着时间推移，即时编译器逐渐发挥作用，把越来越多的代码编译优化成本地代码，来获取更高的执行效率。解释器这时可以作为编译运行的降级手段，在一些不可靠的编译优化出现问题时，再切换回解释执行，保证程序可以正常运行。

即时编译器极大地提高了Java程序的运行速度，而且跟静态编译相比，即时编译器可以选择性地编译热点代码，省去了很多编译时间，也节省很多的空间。目前，即时编译器已经非常成熟了，在性能层面甚至可以和编译型语言相比。不过在这个领域，大家依然在不断探索如何结合不同的编译方式，使用更加智能的手段来提升程序的运行速度。

## 三、Android的虚拟机技术演进

Apple基于软硬件一体化的商业模式，很自然地选择`Object C`作为iPhone的开发语言，并获得了更为优秀的性能。
反之，由于Google自身并不生产硬件设备，Android必须面对如何适配不同硬件平台的难题，采用Java技术标准几乎是唯一的解决方案。

在分析Android虚拟机之前，我们先明确几个概念：

- `APK` - Android Package: Android应用的安装包，本质是一个zip打包文件，其中多个目录分别存储：DEX执行文件、资源文件和系统配置文件
- `DEX` - Dalvik EXecutable: Android Dalvik的可执行文件，注意，其基于Dalvik字节码，并非Java的标准字节码
- `ART` - Android Runtime: Android 运行时环境

实际上，开发Android应用开发的路径是：**.java文件 -> .class文件 -> .dex文件**。

{% asset_img dex.png %}

开发者基于Java语言开发源代码文件，然后编译成Java字节码文件，再使用转换工具`dx`转换为DEX字节码文件，最后在Dalvik虚拟机中执行（可能是解释方式，也可能编译模式）。

### 1. 早期的Dalvik虚拟机

Dalvik是Google公司自己设计用于Android平台的虚拟机，在Android 5.0之前采用，其基本演进路线是：

- 2008年，Android发布，Dalvik VM的执行引擎是只有解释器的；
- 2010年，Android 2.2发布，Dalvik VM引入了JIT编译器，JIT的引入使得Dalvik的性能提升了3～6倍；
- 2013年，Android 4.4发布，Dalvik和ART并存；

Dalvik VM的设计思想与Java虚拟机相似，但并不遵循JVM规范来实现，存在着明显的差异：

1. 基于寄存器的技术实现，而非基于栈
    DVM是基于寄存器的，它没有基于栈的虚拟机在拷贝数据而使用的大量的出入栈指令，同时指令更紧凑更简洁；
    但是，由于显示指定了操作数，所以基于寄存器的指令会比基于栈的指令要大，但是由于指令数量的减少，总的代码数不会增加多少。
2. 自定义的字节码
    Dalvik 有 218 个操作码，它们与 Java 中的 200 个操作码有本质的不同。
    例如，有十几种操作码用于在堆栈和局部变量列表之间传输数据，而在 Dalvik 中完全没有。
3. Class类加载方式不同
    标准Java应用开发提交的是`.jar`文件，JVM将**动态加载**其中包含的全部`.class`文件，每个.class文件里面包含了该类的常量池、类信息、属性等。
    对于Android，其通过dx工具将所有的`.class`文件合并、精简并转换为单一的`.dex`文件，然后交给DVM执行，既减少了I/O操作，又提高了运行速度。
4. 多实例并发能力
    针对手机终端有限的内存和计算能力，DVM允许同时运行多个虚拟机的实例，并且每一个Dalvik 应用作为一个独立的Linux进程执行，防止在虚拟机崩溃的时候所有程序都被关闭。
    具体来说，DVM由Zygote创建和初始化，每当系统需要创建一个应用程序时，Zygote就会fork自身，快速地创建和初始化一个DVM实例，用于应用程序的运行。

Dalvik VM的主要问题是性能！
早期的纯解释器执行显然速度慢，Android 2.2加入了JIT编译器，采用了解释器+JIT编译的方式，虽然运行性能提升了，可是每次启动应用都需要动态编译生成机器码，这会拖慢应用的启动速度。

### 2. 中期的ART虚拟机

- 2013年，Android 4.4引入了ART VM，改为AOT - Ahead Of Time的编译模式（静态编译）。
    其运行模式是：应用在第一次安装的时候，字节码就会预先编译成机器码，使其成为真正的本地应用；之后打开App的时候，不需要额外的翻译工作，直接使用本地机器码运行。
- 2014年，Android 5.0发布，ART取代了Dalvik成为了VM，同时AOT也成为了唯一的编译模式。

对比Dalvik，ART是安装的时候就编译好了，执行的时候直接运行机器码，开启应用速度明显提高。
> 有一段时间，Huawei手机总是提示**正在进行APP优化**，其实就是编译生成机器码。

但是，ART的缺点也是明显的，一是应用安装的时间长，二是消耗更多的存储空间，因为需要持久化保存机器码。

### 3. 全新的Hybrid模式(Interpreter + JIT + AOT)

2016年，Android N 引入了一种包含编译、解释和 JIT（Just In Time）的混合运行时，以便在安装时间、内存占用、电池消耗和性能之间获得最好的折衷.
这种混合模式仍被简称为 AOT，但是其含义变成“全时段的编译”（All-Of-the-Time compilation）

{% asset_img Android-N.png %}

其主要特征为：

1. 在App第一次启动的时候，系统会直接以Intercept的方式来运行，以便快速启动；
2. 在App第一次启动的同时，启动`Compilation Daemon Service`服务，负责在系统空闲的时候，后台ART针对**热代码**进行静态编译，其他代码不做编译;
3. ART包含了一个新的、更快的JIT解释器，负责动态生成机器码，但并不是进行持久化存储；
4. 代码在执行期间被分析函数调用情况，分析结果保存在`Profile`文件中，以便未来生成AOT，这就是`Profile Guided AOT`；
5. ART采用了深度内联等几种更高级的技巧，对代码进行更深度的优化。

这种混合使用 AOT、解释、JIT 的策略的全部优点如下

- 即使是大应用，安装时间也能缩短到几秒
- 系统升级能更快地安装，因为不再需要优化这一步
- 应用的内存占用更小，有些情况下可以降低 50%
- 改善了性能
- 更低的电池消耗

## 四、结论

1. 与Javascript V8引擎相似，Android虚拟机也走向了混合编译模式。
2. 华为的方舟编译器，宣称抛弃了虚拟机，全部采用机器码，支持Java和C++的混合编译，但又宣称自主设计了统一的中间表示MapleIR，后续可以深入研究与Java的关系？

---

## 参考文献

### 官方网站

- [JCP发布的Java Virtual Machine蓝皮书](https://web.archive.org/web/20110925050249/http://java.sun.com/docs/books/vmspec/2nd-edition/html/VMSpecTOC.doc.html)
- [Android Dalvik 的字节码清单](https://developer.android.com/reference/dalvik/bytecode/Opcodes.html)
- [主流Java Virtual Machine的技术对比 - Wiki主页](https://en.wikipedia.org/wiki/Comparison_of_Java_virtual_machines)
- 《深入理解Java虚拟机：JVM高级特性与最佳实践》 周志明 著

### 技术研究

- [一文让你搞懂各种虚拟机、解释器、JIT和AOT编译器](https://blog.csdn.net/zhongyili_sohu/article/details/106555297)
- [关于Jvm知识看这一篇就够了](https://zhuanlan.zhihu.com/p/34426768)
- [Java即时编译器原理解析及实践 - 美团技术](https://tech.meituan.com/2020/10/22/java-jit-practice-in-meituan.html)
- [浅谈 Android Dex 文件 - 有赞技术](https://tech.youzan.com/qian-tan-android-dexwen-jian/)
- [Android虚拟机的JIT编译器](https://cloud.tencent.com/developer/article/1445764)
- [Dalvik 和 Java 字节码的比较](https://segmentfault.com/a/1190000040710467)
- [一文读懂 DEX 文件格式解析](https://cloud.tencent.com/developer/article/1663852)
- [首次全面深度解密华为方舟编译器](https://bbs.huaweicloud.com/blogs/detail/105435)
