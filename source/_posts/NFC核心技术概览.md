---
title: NFC核心技术概览
date: 2020-12-30 17:20:41
tags:
---

## NFC简介

NFC(近场通信，Near Field Communication）是一种短距高频的无线电技术，由非接触式射频识别(RFID)演变而来。
NFC工作频率为13.56Hz，通常只有在距离不超过4厘米时才能启动连接，其传输速度有106 Kbit/秒、212 Kbit/秒或者424 Kbit/秒三种。
NFC有3种工作模式：读卡器模式、点对点模式、卡模拟模式。

- 读取器/写入器模式（Reader/writer mode）：NFC设备产生射频场从外部采用相同标准的NFC标签中读写数据，支持 NFC 设备读取和/或写入被动 NFC 标签和贴纸。
- 点对点模式（P2P mode）：支持 NFC 设备与其他 NFC 对等设备交换数据，Android Beam 使用的就是此操作模式。
- 卡模拟模式（Card emulation mode）：读卡器是主动设备，产生射频场；NFC设备为被动设备，模拟一张符合NFC标准的非接触式卡片与读卡器进行交互。

{% asset_img 3.png NFC的三种工作模式 %}

## 基于Android的NFC终端

Android 4.4版本开始，通过HCE(host-based card emulation，基于主机的卡模拟)，方式提供NFC功能支持。

NFC终端主要包括非接触性前端CLF(也叫NFC控制器)、天线(Antenna)、安全模块(Secure Element,SE)三个主要部件，其中，非接前端、天线一般都集成在手机终端中，而安全模块可根据情况存放在不同的位置。

{% asset_img 2.jpg 典型的NFC终端架构 %}

- CLF：非接触性前端，也称为NFC控制器，其功能包括射频信号的调制解调，非接触通信的协议处理。
    非接触前端一方面连接射频天线，实现13.56MHz信号的发送与接收，另一方面与安全模块通信。在CLF中提供了识读接口、P2P接口、卡模拟接口，分别对应上面所说的三种工作模式。
- 天线，通常集成在终端内部，与非接前端相连，实现13.56MHz射频信号的发送与接收。
- 安全模块SE，主要功能是实现应用和数据的安全存储，对外提供安全运算服务，它是卡模拟的核心。
    安全模块还通过非接前端与外部读写设备进行通信，实现数据存储及交易过程的安全性。

根据安全模块存放的位置不同，NFC可分为不同的实现方案。

### 基于硬件的虚拟卡模式(Virtual Card Mode)

在虚拟卡模式下，需要提供安全模块SE(Secure Elemen)，SE提供对敏感信息的安全存储和对交易事务提供一个安全的执行环境。NFC芯片作为非接触通讯前端，将从外部读写器接收到的命令转发到SE，然后由SE处理，并通过NFC控制器回复。

{% asset_img SE.png 基于安全芯片SE的方式 %}

### 基于软件的主机卡模式(Host Card Mode)

在主机卡模式下，不需要提供SE，而是由在手机中运行的一个应用或云端的服务器完成SE的功能，此时NFC芯片接收到的数据由操作系统或发送至手机中的应用，或通过移动网络发送至云端的服务器来完成交互。两种方式的特点都是绕过了手机内置的SE的限制。

那么，如何通过HCE技术在手机上实现NFC卡模拟呢？首先要创建一个处理交易事务的HCE 服务，Android4.4为HCE服务提供了一个非常方便的基类，我们可以通过继承基类来实现自己的HCE服务。如果要开发已存在的NFC系统，我们只需要在 HCE 服务中实现NFC 读卡器期望的应用层协议。反之如果要开发自己的新的NFC 系统，我们就需要定义自己的协议和APDU 序列。一般而言我们应该保证数据交换时使用很少的APDU包数量和很小的数据量，这样用户就不必花很长时间将手机放在NFC 读卡器上。

HCE 技术只是实现了将NFC 读卡器的数据送至操作系统的HCE 服务或者将回复数据返回给NFC 读卡器，而对于数据的处理和敏感信息的存储则没有具体实现细，所以说到底HCE 技术是模拟NFC 和SE 通信的协议和实现。但是HCE 并没有实现SE，只是用NFC 与SE 通信的方式告诉NFC 读卡器后面有SE的支持，从而以虚拟SE 的方式完成NFC 业务的安全保证。既然没有SE，那么HCE 用什么来充当SE 呢，解决方案要么是本地软件的模拟，要么是云端服务器的模拟。负责安全的SE如何通过本地化的软件或者远程的云端实现，并且能够保障安全性，需要HCE厂商自己考虑和实现。

{% asset_img HCE.png 基于主机的模拟卡方式 %}

> 超级SIM卡是基于SIM SE芯片的技术方案，因此属于**基于硬件的虚拟卡模式**，但同时也可以为HCE提供支持。

### 双模（Dual Mode）

{% asset_img dual-mode.png 双模方式 %}

### 对比分析

{% asset_img hce-se.png 对比分析 %}

## HCE的协议栈

为支持NFC射频卡，HCE主要实现了两个ISO协议，分别是硬件标准`ISO/IEC 14443`和应用协议`ISO/IEC 7816`。

{% asset_img protocol-stack.png NFC的核心协议栈 %}

### ISO/IEC 14443

Android 4.4 支持基于`NFC-Forum ISO-DEP`规范（基于`ISO/IEC 14443-4`）的模拟卡，要求仅使用 Nfc-A (`ISO/IEC 14443-3 Type A`) 技术模拟 ISO-DEP，但也可以支持 Nfc-B (`ISO/IEC 14443-4 Type B`) 技术。

该标准包含四个部分：

- `ISO/IEC14443-1`:制定了有关非接触卡的物理特性；
- `ISO/IEC14443-2`:制定了有关射频功率及信号界面的特性；
- `ISO/IEC14443-3`:则为非接触卡的初始化及防冲突机制；
- `ISO/IEC14443-4`:位有关的交易协定。

ISO/IEC14443-3 定义了 TYPE A、TYPEB 两种卡型（与飞利浦的 Mifare 标准兼容），均通过13.56Mhz的射频载波传送信号，此外索尼公司开发了FeliCa 标准，也成为TYPE F。
不同TYPE的主要的区别在于信号发送的载波调制深度、二进制数编码方式存在差异。
此外，防冲突机制的原理也完全不同，TYPE A是基于 BIT 冲突检测协议，TYPE B则是通过字节、帧及命令完成防冲突。

### SWP单线协议(Single Wire Protocol)

手机与普通非接触IC卡最大的不同体现在拥有网络功能和人机交互两部分，因此，NFC手机可以从事传统非接触IC所不能完成的丰富业务，如空中充值、余额查询。所有这些业务均需要一个技术前提即需要一个标准的SIM卡访问接口，能够使得应用客户端访问SIM卡并与SIM卡中的applet进行通信。具体讲，需要在手机中支持三个标准：

1. SIM Alliance Open Mobile API：为应用客户端提供与SIM卡通信的通道

2. Global Platform/GSMA：Secure Element Access Control：授权应用客户端访问SIM卡中对应的applet

3. Modem：需完全支持3GPP 27.007标准，支持打开SIM卡逻辑通道，并能够在逻辑通信上真正实现APDU的透传

{% asset_img swp.jpg 对比分析 %}

SWP(Single Wire Protocol)是采用C6引脚的单线连接方案。在SWP方案中，接口界面包括三根线：VCC(C1)、GND(C5)和SWP(C6)，其中SWP一根信号线上基于电压和负载调制原理实现全双工通讯，这样可以实现SIM卡在ISO7816界面定义下同时支持7816和SWP两个接口，并预留了扩展第三个高速(USB)接口的引脚。支持SWP的SIM卡必须同时支持ISO和SWP两个协议栈，需要SIM的COS是多任务的OS系统，并且这两部分需要独立管理的，ISO界面的RST信号不能对SWP部分产生影响。

　　SWP是在一根单线上实现全双工通讯，定义了S1和S2两个方向的信号， SWP传输的波特率可以从106KBPS最高上升至2MBPS。从SWP的定义看，SWP方案同时满足ISO7816、NFC和大容量高速接口，并且是全双工通讯，可以实现较高波特率。SWP系统地定义了从物理层、链路层到应用层的多层协议，并已经上升成为ETSI的标准，正在争取成为ISO的标准，目前得到业界较多的支持。从另一个角度看，SWP方案要求SIM卡和NFC模拟前端芯片同时重新设计，涉及的面比较广，市场推进的难度较大。另外，NFC应用非常关注掉电模式下的应用，SWP的S2负载调制通讯方式带来接口的功耗损失，对掉电模式下的性能有不利影响。

### ISO/IEC 7816

Android处理应用协议数据单元 (APDU)遵循的是`ISO/IEC 7816-4`规范。

Android系统上的HCE技术是通过系统服务实现的(HCE服务)。使用服务的一大优势是它可以一直在后台运行而不需要有用户界面。这个特点就使得HCE技术非常适合像会员卡、交通卡、门禁卡这类的交易，当用户使用时无需打开程序，只需要将手机放到NFC读卡器的识别范围内，交易就会在后台进行。当然如果有必要的话，用户也可以打开UI界面。这时的手机和普通的智能卡片已经没有区别了。

#### 服务选择AID

交易中我们有一个重要问题需要解决，当用户将手机放到NFC读卡器的识别范围时,Android系统需要知道读卡器真正想要和哪个HCE服务交互，这样它才能将接收到的数据发送至相应的服务。ISO/IEC 7816-4规范正是解决服务选择的问题，它定义了一种通过应用程序ID(AID)来选择相应服务的方法。

一个AID占16位，如果手机模拟的是一个已经存在的NFC读卡设施，那么这些NFC读卡设施会去寻找那些经公共注册而广为人知的AID(类似于端口号)。像Visa卡和万事达卡等这些智能卡可以注册 AID号作为他们专用的识别标志。反之，如果要为自己的新的读卡设施部署NFC应用，你就需要注册自己的AID。AID注册过程在ISO/IEC 7816-5规格中定义，为防止和其他的Android程序冲突，Google建议AID号按此规格中推荐的注册。

当用户将设备接近 NFC 读写器时，Android 系统需要知道 NFC 读写器实际上想要与哪个 HCE 服务对话。这就是`ISO/IEC 7816-4`规范的来源：它定义了一种以应用程序 ID（AID）为中心的选择应用程序的方法。

AID 由 16 个字节组成。如果您正在为现有的 NFC 读写器基础设施模拟卡片，这些读者正在寻找的 AID 通常是众所周知的和公开注册的（例如，支付网络的 AID，如 Visa 和 MasterCard）。

如果您想为自己的应用程序部署新的读取器基础设施，则需要注册您自己的 AID。AID 的注册过程在`ISO/IEC 7816-5`规范中定义。谷歌建议，如果您正在为 Android 部署 HCE 应用程序，可以按照 7816-5 注册一个 AID，它可避免与其他应用程序发生冲突。

#### AID组

在某些情况下，HCE 服务可能需要注册多个 AID 才能实现某个应用程序，并且需要确保它是所有这些 AID 的默认处理程序（与另一服务的组中的某些 AID 相反）。

AID 组是一系列被视为属于共同的操作系统 AID 。对于一个 AID 组中的 AID，Android 可以保证以下一项：

组中的所有 AID 都被路由至此 HCE 服务。
组中没有任何 AID 被路由至此 HCE 服务（例如，服务请求组中的一个或多个 AID，而用户优先选择另一服务）。
换句话说，不存在中间状态，其中该组中的一些 AID 可以被路由到一个 HCE 服务，而另一些可以路由到另一个 HCE 服务。

#### AID组和类别

每个 AID 组都与一个类别关联。这使得 Android 可以按类别将 HCE 服务分组，并且反过来允许用户在类别级别上设置默认值而不是 AID 级别。通常，避免在应用程序中任何面向用户的部分中提到 AID：它们对普通用户没有任何意义。

Android 4.4 支持两种类别：`CATEGORY_PAYMENT`（覆盖行业标准支付应用）和`CATEGORY_OTHER`（对应于所有其它 HCE 应用）。

注意：在任何给定时间，在系统中只能启用CATEGORY_PAYMENT类别中的一个 AID 组。通常，这将是一个应用程序，了解主要的信用卡支付协议，可以在任何商家工作。

对于仅在一个商家（例如储值卡）工作的闭环支付应用，您应该使用CATEGORY_OTHER。该类别中的 AID 组可以总是活动的，并且在必要时可以在 AID 选择期间由 NFC 读写器给予优先级。

## 超级SIM卡的通信接口

通信接口指的是 SIM 卡与外部终端设备进行通信的接口，应支持 ISO7816 和 SWP 两种接口。

- ISO7816 接口是 SIM 卡与外部终端设备进行通信的接触式 I/O 接口，遵循 ETSI 102.221 的要求。
- SWP 接口是 SIM 卡与外部非接触终端设备进行通信，实现近场通信相关业务 的物理接口。
    超级 SIM 卡支持 SWP 协议，遵循 ETSI TS 102.613 的要求。支持卡 模拟模式、读卡器模式，可选支持点对点传输模式。

移动终端若支持 NFC 功能，则应支持 SWP 接口，与超级 SIM 卡协同实现刷卡 操作，为用户提供基于非接触感应的线下应用场景。

### 应用层的技术标准

NFC手机可以从事传统非接触IC所不能完成的丰富业务，如空中充值、余额查询。所有这些业务均需要一个技术前提即需要一个标准的SIM卡访问接口，能够使得应用客户端访问SIM卡并与SIM卡中的applet进行通信。具体讲，需要在手机中支持三个标准：

1. SIM Alliance Open Mobile API：为应用客户端提供与SIM卡通信的通道

2. Global Platform/GSMA：Secure Element Access Control：授权应用客户端访问SIM卡中对应的applet

3. Modem：需完全支持3GPP 27.007标准，支持打开SIM卡逻辑通道，并能够在逻辑通信上真正实现APDU的透传

---

## 附录一：SIM卡的技术标准

SIM卡是一个装有微处理器的芯片卡，它的内部有5个模块，并且每个模块都对应一个功能：、

- 微处理器CPU（8位）
- 程序存储器ROM（3--8kbit）
- 工作存储器RAM（6--16kbit）
- 数据存储器EEPROM（128--256kbit）
- 串行通信单元。

{% asset_img sim.jpeg 对比分析 %}

这5个模块被胶封在SIM卡铜制接口后与普通IC卡封装方式相同。这五个模块必须集成在一块集成电路中，否则其安全性会受到威胁。因为，芯片间的连线可能成为非法存取和盗用SIM卡的重要线索。

SIM卡同手机连接时至少需要5条连接线（通常编程口未定义）

- 数据I/O口（Data）
- 复位（RST）
- 接地端（GND）
- 电源（Vcc）
- 时钟（CLK）

如上图所示。

SIM卡的供电分为5V（1998年前发行）、5V与3V兼容、3V、1.8V等，当然这些卡必须与相应的移动电话机配合使用，即移动电话机产生的SIM卡供电电压与该SIM卡所需的电压相匹配。卡电路中的电源VCC、地GND是卡电路工作的必要条件。卡电源用万用表就可以检测到。SIM卡插入移动电话机后，电源端口提供电源给SIM卡内各模块。

## 附录二：主流NFC硬件厂商和芯片型号

|射频前端芯片|读卡器/NFC芯片|卡芯片|
|:---:|:---:|:---:|
|德州仪器：TI TRF7970A|恩智浦：NXP PN532|复旦微电子：FMSH FM1208|
|复旦微电子：FMSH FM11NC08S|恩智浦：NXP PN7150|华翼微电子 HYm4616A1/2/3/4/5/6|
|-|意法半导体：ST CR95HF|华翼微电子： HYm4616A7|
|-|华大电子：HED CIE72D01|-|

## 参考资料

- [NFC-SWP终端架构与标准](https://blog.csdn.net/icycityone/article/details/17358357)
- [Android的NFC官方文档](https://developer.android.com/guide/topics/connectivity/nfc/hce?hl=zh-cn)
- [NFC SWP移动支付解决方案技术分析](https://www.mpaypass.com.cn/news/201307/12110718.html)
- [近距离通信的SWP方案及其在SIM卡的实现](http://tech.rfidworld.com.cn/2010_07/04cd42c1fd6aac1d.html)
- [SIM卡详解](https://blog.csdn.net/xiaoxik/article/details/82156455)
- [关于HCE的NFC支付研究报告及其安全性探讨](http://www.jiajuhf.com/zxxw_8/42705634.html)
- [基于主机的卡模拟概览](http://article.iotxfd.cn/RFID/Host-based%20card%20emulation)
- [HCE基础知识普及](https://blog.csdn.net/wwww1988600/article/details/69523369)
- [NFC之 Type A 与 TYpe B 卡区别](https://blog.csdn.net/liwei16611/article/details/85209361)
- [超级SIM卡的技术白皮书](http://www.cmricloud.com/pdf/07/1.pdf)
- [基于HCE移动支付研究报告](https://mp.weixin.qq.com/s?src=3&timestamp=1625732247&ver=1&signature=YhDcKm20OjT1SqXPV4ZZjLRQtlP42pVugJP77ZqfP6lSnDV7-d-WYWFpxgd-qXkSJ7EwF-g7TpH2pu5MDifsfvGJsEF1yY9jmRZa*elztII6P9xrvmw53XvWBsp-ztpwDYuS4VXwrXgHrA4p4NpNaQ==)
- [NFC-SWP连接方案在SIM卡中的实现方法](http://www.nfcin.com.cn/news/201403/11110054.html)

