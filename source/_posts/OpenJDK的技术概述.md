---
title: OpenJDK的技术概述
date: 2021-11-21 10:35:37
tags:
---

## 前言

1995年，Sun公司正式发布了Java语言，并建立了`JCP（Java Community Process）`开源社区，负责管理Java技术标准。
`JSR（Java Specification Requests）`是JCP发布的技术规范，由拥有投票权的JCP成员集体投票决定。例如，JVM最核心的两份技术标准就是：JSR 924（Java Virtual Machine Specification）和 JSR 202（Java Class File Specification Update）

为了开发和运行Java应用软件，必须为用户提供一个完整的开发套件，这就是`JDK`（Java Development Kit），包含了编译器、软件库和Java虚拟机等核心组件。1999年之后，Sun公司发布了多个版本的JDK产品，主要包括：

- `J2SE`：Java Platform Standard Edition，标准版的Java平台。目标是工作站等标准应用，也是最常见的
- `J2EE`：Java Platform Enterprise Edition，企业版的Java平台。目标是企业级应用，通常是收费的
- `J2ME`：Java Platform Micro Edition，微型版的Java平台。目标是手机、游戏机等移动设备、嵌入式设备，但由于Android的发展已被淘汰
- `Java Card`：广泛运用在SIM卡、提款卡上，以具有安全防护性的方式来执行小型的Java Applet

> 2010年，Oracle收购Sun以后，`J2SE`也被称为`Oralce JDK`，而JCP董事会也受到广泛批评，被称为“Oracle的橡皮图章”。

## JDK & JRE & JVM

Java的技术理念是“**一次编译，到处运行**”，核心特性就是跨平台，必须通过`JRE`(Java Runtime Enviroment)解决平台适配问题。
`JRE`就是运行Java程序所必须环境的集合，面向Java程序的使用者，而不是开发者。因此，如果你仅下载并安装了JRE，那么你的系统只能运行Java程序，但无法进行开发。

以Oracle JDK为例，其技术架构参见[Java SE Platform at a Glance](https://docs.oracle.com/javase/8/docs/index.html)，而JRE仅仅是**不包含开发工具**。

{% asset_img j2se.png%}

### 1. 开发工具（Tools & Tool APIs）

开发工具仅在`JDK`中提供，具体包括编译器javac、解释器交互工具java、打包工具jar等。
由于`JRE`不包含此部分，因此只能用于部署生产环境，无法支持开发。

### 2. 核心类和支持库文件（Java SE API）

- Integration Libraries：集成库文件，包括数据库连接JDBC、CORBA接口IDL、名字和目录服务JNDI等
- Other Base Libraries：其他基础库文件，包括XML解析器、网络接口、JSON序列化、日期时间函数等
- Base Libraries：核心库文件。包括基础类定义、数学计算、反射等动态语言特性、日志等监控功能

此外，由于浏览器技术的快速发展，Java主要被用于后台服务处理，以下组件实际上很少使用，基本被淘汰

- Deployment：用于开发浏览器脚本和插件等，后来出现了替代开源项目IcedTea，但均未成为行业主流
- User Interface Toolkits：GUI工具，但由于浏览器技术的快速发展，java，未成为行业主流

### 3. 虚拟机（Java Virtuanl Machine）

Java虚拟机是JRE的核心引擎，它支持不同的平台，如Intel 32位和64位架构，ARM架构和SPARC等。
Oracle JDK使用`HotSpot`作为默认引擎，用于解释并执行字节码。
BEA公司开发了`JRockit`引擎，特点是使用纯编译的执行引擎，没有解释器。随着被Oracle收购，从JDK 8已经被融合到`HotSpot`。

``` console
bogon:~ sj$ java -version
java version "15.0.2" 2021-01-19
Java(TM) SE Runtime Environment (build 15.0.2+7-27)
Java HotSpot(TM) 64-Bit Server VM (build 15.0.2+7-27, mixed mode, sharing)
```

### 结论：JDK = Java虚拟机 + Runtime类库 + 编译器等开发工具包

{% asset_img JDK-2.png %}

## OpenJDK Community - OpenJDK的诞生

2005年，Sun公司发布了Java语言和Java SE，虽然通过“Sun社群代码许可”（Sun Community Source License），源代码可以免费使用。但是，使用了Java源代码的程序必须遵守Java的编程规范，商业性改编程序必须要得到Sun的许可。

2006年，为了缓解IT行业的巨大压力，Sun公司牵头组建了开源社区`OpenJDK Community`，初始源代码采用GPL v2许可证。

2007年，Sun公司正式发布了开源版的Java开发组件（OpenJDK），作为Java SE 6的开源和免费实现，并根据`GNU GPL`许可证授权，向开发人员开放了创作改编程序所必要的权限，并赋予其在不同许可下发布应用程序的能力。

目前，OpenJDK社区包含了大量Project项目，例如：

- JDK 6-9：基于JSR 270（Java SE 6 Release Contents），构建Jave SE 6的开源实现，7-9版本类似
- OpenJDK：这是最重要的项目，负责构建OpenJDK 10-18版本的开源技术实现。由于自JDK 10开始，Oracle JDK与OpenJDK实现了融合，该项目产出的OpenJDK直接成为了Java SE的官方参考实现，因此不再为每个版本单独设立项目，OpenJDK 18是当前最新的开发版本。
- Graal：源于SUN公司的`Maxine VM`项目，采用高度优化的JIT编译器。2012年从OpenJDK项目中剥离出来，并在JDK 10中纳入实验性功能
- IcedTea：最初是由于OpenJDK不完整（例如个别字体库由于许可证差异而无法提供）而创立的，为社区提供必要的开源工具链及代码库。它有一个基于`./configure`的不同的构建系统，正是由于IcedTea的努力，许多第三方发行版大大减少了使用补丁的数量

`OpenJDK Project` 项目组成员包括Oracle、IBM、Alibaba，Amazon、Azul、Google，Huawei，Intel、Microsoft等几乎所有主流IT公司，使用C++和Java编程语言开发。

没错，从 Java SE 7开始往后的版本，连大名鼎鼎的 Oracle JDK 都是根据 Open JDK 做出来的 (修改了一些功能的实现方式，再打上自己的商标并提供配套服务)。或者应该说自 Java SE 7开始往后的版本，所有的 JDK 都源自于 Open JDK (OpenJDK 与 其他 JDK 的关系就和 Linux 与它的众多发行版是一样一样的)。

## OpenJDK Vs Oracle JDK

`OpenJDK`和`Oracle JDK` 是通过`TCK`认证的同一Java规范的实现。换句话说，`Oracle JDK`是`OpenJDK`的商业发行版（非开源发行版），就如同Chrome和Chrominum的关系。

|项目| Oracle JDK| OpenJDK|
|:-:|:-:|:-:|
|许可证|Oracle公司的商业（非开源）许可证|基于GPL v2许可证的开源授权|
|开发方|Oracle公司，可以使用Java商标| OpenJDK开源社区，不允许使用Java商标|
|性能优化 |根据Sun JDK的开发和实现提供性能 | 提供由Oracle JDK之上的一些供应商开发的高性能|
|发行方式| 仅提供二进制代码| 基于社区源代码，各方自行定制发行版|
|费用| 基于Oracle许可证，高级功能可能收费| 完全开源和免费使用|
|操作系统| Windows，Linux，Solaris，MacOS| **FreeBSD**，Linux，Microsoft Windows，Mac OS X|

简单一点说，`OpenJDK`源代码是`Oracle JDK`的一个子集，只包含最精简的JDK。

- OpenJDK不包含Deployment组件（Browser Plugin、Java Web Start、Java控制面板等），当然这些功能也没人用
- 由于`OpenJDK`采用`GPL`许可证，`Oracle JDK`的一部分源代码（例如JMX的SNMP功能）由于产权的问题无法开园，只能作为Plug可选插件方式提供给OpenJDK编译时使用。而Icedtea则为这些不完整的部分开发了相同功能、但是符合`GPL`许可证的源代码，促使OpenJDK更加完整
- Oracle JDK的大部分源代码来自于Open JDK，但包含一些Oracle尚未开源的技术组件，也有一些来自于第三方授权的技术组件

{% asset_img vs.jpeg %}

## OpenJDK Builds - 二进制发行版

### 1. Oracle OpenJDK

没错！OpenJDK Community只负责产生OpenJDK源代码，并不提供可以直接使用的二进制文件格式。
OpenJDK官网指向的可下载二进制文件的地址，实际是Oracle公司自行编译后提供的软件。就是这个：

``` bash
[root@test ~]# java -version
openjdk version "1.8.0_312"
OpenJDK Runtime Environment (build 1.8.0_312-b07)
OpenJDK 64-Bit Server VM (build 25.312-b07, mixed mode)
```

### 2. Eclipse Open J9

1996年，IBM公司基于Smalltalk VM开发了J9 VM，并广泛应用于其各类自有产品线。
2017年，J9成为Eclipse基金会项目，改名为Open J9，并完全符合 Java JVM 规范。

### 3. Oracle GraalVM

`Graal`起源于Sun公司的Maxine虚拟机项目，也是基于HotSpot VM，可以认为是HotSpot的一个变种。
特点是有独立的JIT编译器、支持AOT提前编译的等，允许在单个程序中自由混合来自任何编程语言的代码等。
Oracle公司现在`GraalVM Enterprise`的名义提供该产品。
第一个正式版本 Graal VM 19.0 于 2019 年 5 月发布。最新版本是 Graal VM 21.0.0，于 2021 年 1 月发布。

### 4. Azul Zulu

Azul Systems公司是一家专门从事 Java 和 JVM 产品的公司。主要提供`OpenJDK` 二进制分发版，并命名为`Zulu`，包含三个版本：

- `Zulu Community`：基于`GPLv2`协议的社区免费版
- `Zulu Enterprise`：商业发行版
- `Zulu Embedded`：为嵌入式、移动和物联网设备使用的版本
- `Zulu PlatForm Prime`：服务器使用的高性能版本，原名`Zing VM`

> 为了适配Apple M1芯片，目前Mackbook Air仅能适配`OpenJDK 64-Bit Server VM Zulu16.28+11-CA (build 16+36, mixed mode)`

``` bash
sj@bogon ~ % java -version
openjdk version "16" 2021-03-16
OpenJDK Runtime Environment Zulu16.28+11-CA (build 16+36)
OpenJDK 64-Bit Server VM Zulu16.28+11-CA (build 16+36, mixed mode)
```

---

## 附录1: TCK - Java技术兼容性认证

虽然人人都可以运用Java语言编程，Sun仍将Java SE的函数库作为预先编译的Java字节码，附带其API接口一并提供给用户，同时通过技术兼容性测试工具包`TCK(Technology Compatibility Kit)`以检查第三方产品是否符合Java的规范要求，以确保对java语言生态的有效控制。

其中，最著名的事件就是`Apache Harmony`开源项目。

2005年，Apache基金会主导了`Apache Harmony`开源项目，目标是以开放源代码方式实现Java SDK，IBM等公司提供了大量代码。
由于一直无法获得TCK授权，2011年10月项目宣布停止。核心原因是Java Community Process规定的`GPL`许可证，与`Apache`许可证不兼容

值得指出的是，Google在`Android`早期开发中，曾经大量使用该项目的源代码，为此长期陷入与Java的专利诉讼，最终决定基于`OpenJDK`，采用`Clean Room`模式自主开发了`Dalvik VM`。

## 附录2：常见的许可证协议

{% asset_img license.jpg %}

---

## 参考文献

### 官方网站

- Java Community Process的官网：[https://www.jcp.org/](https://www.jcp.org/en/home/index)
- OpenJDK Community的官网：[http://openjdk.java.net/](http://openjdk.java.net/)
- OpenJDK Project的Github源码：[https://github.com/openjdk/jdk](https://github.com/openjdk/jdk)

### 技术评论

- [甲骨文诉谷歌Java侵权案 - WiKI](https://zh.wikipedia.org/zh-hans/%E7%94%B2%E9%AA%A8%E6%96%87%E8%AF%89%E8%B0%B7%E6%AD%8CJava%E4%BE%B5%E6%9D%83%E6%A1%88)
- [开源许可证GPL、BSD、MIT、Mozilla、Apache和LGPL的区别](https://zhuanlan.zhihu.com/p/31881162)
- [Oracle与OpenJDK之间的区别](https://juejin.cn/post/6844903811069247496)
