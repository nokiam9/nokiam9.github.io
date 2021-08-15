---
title: NFC技术和产品综述
date: 2021-08-15 18:05:45
tags:
---

## 概述

NFC（`Near Field Communication`，近场通信），也叫做近距离无线通信技术，用于在两个非常接近的设备之间执行数据传输。

- 2002年，NFC技术由`Philips`和`Sony`两家公司联合推出。
- 2004年，`Nokia`、`Philips`、`Sony`等公司共同组建了非盈利性组织`NFC Forum` ，负责制定NFC相关的技术标准，致力于促进 NFC 技术在消费类电子产品、移动设备和 PC 中的应用，并通过NFC认证测试来保证各厂家的NFC产品符合NFC规范以实现互联互通。
- 2006年，恩智浦半导体公司(`NXP` Semiconductors)从`Philips`剥离，总部位于荷兰埃因霍温，在全球20多个国家拥有37000名员工（欧洲37%、亚洲37%、大中华区21%、美洲5%），2007年公布的销售额达63亿美元。

NFC的核心技术优势在于：

- **无源硬件**：采用感应耦合技术，NFC标签可以从其他NFC设备产生的电磁场获得能量，经济实惠且完全无源，特别适合IoE（物联网）应用场景
- **公开频段**：电磁场频率为 13.56 MHz，该频率是射频频谱高频部分中无需执照的波段，有效距离为4cm左右，目前所支持的数据传输速率有106Kbps、212Kbps和424Kbps三种
- **快速响应**：用户无需事先启动APP（特定场景可能要求安全验证），基于硬件支持即可触发NFC，交互时间只有几分之一秒
- **终端集成**：NFC软件协议栈完全集成到Android（v.4.0以上）、iOS（v.11）中，有助于在多个应用程序中使用NFC，而无需安装任何特定软件或应用

## NFC技术的历史

NFC是多种技术路线并行发展的综合体，得益于条形码、磁条卡和智能终端三种技术的共同推动。

{% asset_image nfc-map.png %}

1. 最左边一条为发源于条形码（Barcodes）的无线射频识别技术，即RFID技术路线，最终出现了NFC中的两个重要组件`NFC Tag`（标签）和`NFC Reader`(读卡器)。

2. 最右边一条为磁条卡（Magnetic Strip Cards）技术路线，该路线后来向智能卡（Smart Card）方向发展，最终演化出了NFC使用的Proximity Coupling Smart Card，有效距离在10cm之内，对应的规范为`ISO/IEC 14443(非接触式IC卡标准)`，从RF（Radio Frequency，无线频率）层面定义了如何与不同的非接触式IC卡（其实物可以是NFC Tag、RFID Tag、Smart Cards）交互。

    > - Vicinity Coupling Smart Card对应的规范为`ISO/IEC 15693`(也称为**疏耦合卡**)，特点一是使用场景的温度要求远高于消费级产品（通常是零下40度到零上85度），二是**读取距离可高达一米**，同时设计简单使得读取器的生产成本低，在物流运输、企业门禁卡等场景中得到广泛应用。

    > - Close Coupling Smart Card对应的规范是`ISO 10536`，该标准主要发展于1992到1995年间，由于这种卡的成本高，优点少，因此从未在市场上销售。

    粗看上去Smart Card和RFID Tag类似，例如二者都只存储一些数据，而且自身都没有电源组件，但Smart Card在安全性上的要求远比RFID Tag严格。另外，Smart Card上还能运行一些小的嵌入式系统（如Java Card OS）或者应用程序（Applets）以完成更为复杂的工作。

3. 中间一条技术路线为移动终端，它演化出了携带NFC功能的终端设备。随着移动终端越来越智能，NFC和这些设备也融合得更加紧密，使得NFC的应用场景得到了较大的拓展。

NFC涉及到多个标准化组织定义的协议，下图是简化的协议栈结构，下面将有详细描述。

{% asset_image protocol-1.png %}

## NFC的运行模式

从用户角度（即图中的Applications层之上）来看，NFC有三种运行模式（operation mode），分别是

- Reader/Write模式（简称R/W，和NFC Tag/NFC Reader相关）
- Peer-to-Peer模式（简称P2P，它支持两个NFC设备交互）
- Card Emulation模式（简称CE，它能把携带NFC功能的设备模拟成Smart Card，这样就能实现诸如手机支付、门禁卡之类的功能）

{% asset_image nfc-mode.png %}

### 读卡器模式（Reader/Write Mode）

数据在NFC Tag的芯片中，可以简单理解成**刷标签**，本质上就是通过支持NFC的手机或其它电子设备从带有NFC芯片的标签、贴纸、名片等媒介中读写信息，随后它可能根据读取来自部件的信息执行操作。例如，靠近 NFC 标签的 NFC 手机会检索到一个 URL，并链接到相应的网站，NFC手机可以在无需打字的情况下发送 SMS （短消息服务）文本、获得优惠券、启动配对动作、获得电子名片（Vcard）等。

{% asset_img rw-mode.jpg %}

对于读卡器模式，NFC Forum定义了4种标准类型的[NFC标签](#NFC标签)（Type 5还不是正式标准），这四种类型NFC Tag的区别在于存储空间大小，数据传输率以及底层使用的协议上，后面有具体的技术细节。

此外，NFC Forum定义了两个通用的数据结构用于在NFC Device之间（包括R/W模式中的NFC Reader和NFC Tag）传递数据。这两个通用数据结构分别是NFC Data Exchange Format（简写为NDEF）以及NFC Record。

> 对于“刷标签”场景，NFC 手机工作在主动模式，而NFC标签是不需要外部供电的，依靠NFC手机发送的电磁场获得能量，**此时数据传输是不安全的**。

### 卡模拟模式（Card Emulation Mode）

数据在支持NFC的手机中，可以简单理解成**刷手机**。本质上就是将支持NFC的手机或其它电子设备当成借记卡、公交卡、门禁卡等IC卡使用。

基本原理是将相应IC卡中的信息凭证封装成数据包存储在支持NFC的外设中，在使用时还需要一个NFC射频器（例如Pos终端、闸机等），将手机靠近NFC射频器，手机就会接收到NFC射频器发过来的信号，在通过一系列复杂的验证后，将IC卡的相应信息传入NFC射频器，最后这些IC卡数据会传入NFC射频器连接的电脑，并进行相应的处理（如电子转帐、开门等操作），这让它可以用于已有的非接触式智能卡基础设施，实现访问控制、非接触式支付、固件更换或数据传输等操作。

{% asset_img ce-mode.jpg %}

该模式下，NFC 手机的工作类似于标准的[非接触式IC卡](#非接触式IC卡)，为此ISO 14443定义了Type A 和 Type B两种技术标准（Type F不是正式标准），后续有详细技术分析。

> 对于“刷手机”场景，NFC 手机工作在被动模式，它只在机具发出的射频场中被动响应，**数据传输是安全的**。

### 点对点模式（Peer-to-Peer Mode）

在点对点 （P2P）中，具有 NFC 功能的设备工作在主动模式。其中的一个设备会发起通信链接。一旦链接建立，设备就会交替地与其他设备通信，并遵守 “ 说前先听 （listen-beforetalk） ” 规则。该通信模式下的数据交换相比其他模式更快，因此可以交换更多的数据。

### 小结

从协议栈的角度，三种工作模式的分析如下：

{% asset_image rf.jpg %}

## 非接触式IC卡

`ISO 14443`全称为非接触式IC卡标准，它从RF层面定义了如何与不同的非接触式IC卡（其实物可以是NFC Tag、RFID Tag、Smart Cards）交互。

该标准包含四个部分：

- ISO/IEC14443-1:   制定了有关非接触卡的物理特性；
- ISO/IEC14443-2:   制定了有关射频功率及信号界面的特性；
- ISO/IEC14443-3:   则为非接触卡的初始化及防冲突机制；
- ISO/IEC14443-4:   位有关的交易协定。

ISO 14443定义了Type A和Type B两种非接触式IC卡。不同TYPE的主要的区别在于信号发送的载波调制深度、二进制数编码方式存在差异。
此外，防冲突机制的原理也完全不同，TYPE A是基于 BIT 冲突检测协议，TYPE B则是通过字节、帧及命令完成防冲突。

### Type A

最早由Philips公司制订（其生产的芯片商标名为MIFARE，现在由从Philips独立出来的NXP公司拥有），目前世界上70%左右的非接触式IC卡都使用了MIFARE芯片，例如北京市的公交卡，特点是采用Modified Miller编码。

> Android 4.4 支持基于NFC-Forum ISO-DEP规范（基于ISO/IEC 14443-4）的模拟卡，要求仅使用 Nfc-A (ISO/IEC 14443-3 Type A) 技术模拟 ISO-DEP，但也可以支持 Nfc-B (ISO/IEC 14443-4 Type B) 技术。

### Type B

由TI公司制订，主要用在法国市场，特点是采用NRZ编码，二者最终都成为ISO标准。

> 第二代居民身份证采用的是Type B标准。

### Type F（FeliCa）

FeliCa是Sony为了非接触式IC卡而开发出来的通信技术，与ISO 14443标准的主要区别是采用曼彻斯特编码。
1998年，FeliCa原本被提案为ISO 14443 type C，但悲剧的是未被采纳，只能成为日本工业标准`JIS X6319-4`。之后，FeliCa和其向后相容方式被标准化为ISO 18092。

> 目前，Felica也被称为Type F，主要应用于日本市场，境外只有香港的“八达通”仍有继续使用。

## NFC标签

NFC 论坛定义了四种类型的 NFC 标签，第五种标签与 NFC-V 技术相关，目前它还不是NFC 论坛规范中的一部分。

|项目|Type 1|Type 2|Type 3|Type 4|Type 5|
|:-:|:-:|:-:|:-:|:-:|:-:|
|对应规范   |ISO 14443 Type A|ISO 14443 Type A |JIS X 6319-4、FELICA|ISO 14443 Type A/B|ISO/IEC 15693|
|常见芯片	|Topaz	|MIFARE	|Felica	|MIFARE-DESFire|NXP的I Code系列，ST的ST25TV系列|
|存储容量	|最大1KB	|最大2KB	|最大1MB	|最大64KB|高达 8 KB|
|读写速率	|106kbps    |106kbps    |212kbps	|106-424kbps|26.48 kbps|
|价格	|低	|低	|高	|中等/高|低|
|安全性	|数字签名保护	|不安全	|数字签名保护	|可选 |N/A|
|说明	|Topaz由Innovision公司推出	|MIFARE由NXP公司推出	|由Sony公司推出，价格比较贵	|这类芯片在出厂时就被配置好是否只读或可读写|上海贝岭等|

从协议栈的角度，下图是 NFC 各类标签所涉及的技术标准。

{% asset_image protocol-2.png %}

## NDEF的数据格式

NFC Forum定义了两个通用的数据结构用于在NFC Device之间（包括R/W模式中的NFC Reader和NFC Tag）传递数据。这两个通用数据结构分别是NFC Data Exchange Format（简写为NDEF）以及NFC Record。

## ISO 18092

`ISO 18092`协议介绍了P2P通讯中的Active模式和Passive通讯模式，其实`ISO 18092`使用了`ISO 14443`协议和非国际标准的`FELICA`通讯协议，

---

## 附录一：NFC的核心技术标准清单

{% asset_image protocol-3.png %}

## 附录二：NFC主流芯片型号

### ISO 14443 Type A

- MF1 IC S20：国内常称为MIFARE Mini，原装芯片厂家为荷兰恩智浦(NXP)，在一卡通方面应用普遍。
- SLE66R35：德国英飞凌（infineon），兼容MF1 IC S50。
- FM11RF08：芯片厂家为上海复旦，兼容MF1 IC S50。
- Mifare Std 1k MF1 IC S50及其兼容卡：原装芯片厂家为荷兰恩智浦(NXP)，在一卡通方面应用普遍。 
- Mifare Std 4k MF1 IC S70及其兼容卡：原装芯片厂家为荷兰恩智浦(NXP)，在一卡通方面应用普遍。　
- Mifare Ultralight MF0 IC U1X：国内常称为U10,芯片厂家为荷兰恩智浦(NXP)，广深高速火车票为典型应用。
- Mifare Ultralight C：原装芯片厂家为荷兰恩智浦（NXP）。
- FM11RF005:芯片厂家为上海复旦,包括FM11RF005SH与FM11RF005M,上海地铁单程票、上海轮渡单程票为典型应用。FM11RF08:芯片厂家为上海复旦
- Mifare DESfire 2k MF3 IC D21：芯片厂家为荷兰恩智浦（NXP），国内常称为MF3 2k。
- Mifare DESfire 4k MF3 IC D41：芯片厂家为荷兰恩智浦（NXP），国内常称为MF3。南京地铁卡为典型应用。
- Mifare DESfire 8k MF3 IC D81：芯片厂家为荷兰恩智浦（NXP），国内常称为MF3 8k。
- Mifare ProX：芯片厂家为荷兰恩智浦（NXP）。不判别容量。　　　　　　　 　
- SHC1102：芯片厂家为上海华虹，上海一卡通为典型应用。
- Advant ATC2048-MP：芯片厂家为瑞士LEGIC。
- MF1 PLUS 2k：芯片厂家为荷兰恩智浦（NXP），国内常称为PLUS S。
- MF1 PLUS 4k：芯片厂家为荷兰恩智浦（NXP），国内常称为PLUS X。
- JEWEL：芯片厂家为英国innovision，国内常称为宝石卡。不读序列号。
- IS23SC4456：芯片厂家为美国ISSI，可兼容MF1 IC S50的CPU卡。
- CPU卡（兼容MF1）：芯片厂家为上海复旦、上海华虹等，可兼容MF1 IC S50的CPU卡。该类也包含FM1208M1及其它类似的芯片卡。
- 纯CPU卡：芯片厂家为上海复旦、美国ISSI等，纯CPU卡。该类也包含FM1208、IS23SC4456中的纯CPU卡及其它类似的芯片卡。
- X82A：芯片厂家为北京华大，CPU卡。

### ISO 14443 Type B

- AT88RF020：芯片厂家为美国爱特梅尔（ATMEL），广州地铁卡为典型应用。
- SR176：芯片厂家为瑞士意法半导体（ST），主要用于防伪识别等。
- SRIX4K：芯片厂家为瑞士意法半导体（ST），主要用于防伪识别等。
- SRT512：芯片厂家为瑞士意法半导体（ST），主要用于防伪识别等。
- ST23YR18：芯片厂家为瑞士意法半导体（ST），CPU卡。
- THR1064：芯片厂家为北京同方，奥运门票为典型应用。
- THR2408：芯片厂家为北京同方，纯CPU卡。
- 第二代居民身份证：芯片厂家为上海华虹、北京同方THR9904、天津大唐和北京华大，第二代身份证为典型应用。

### ISO 15693

- EM4135：芯片厂家为瑞士EM，主要用于票证管理、防伪识别等。
- ICODE SL2 ICS53/ICODE SL2 ICS54：芯片厂家为荷兰恩智浦（NXP），国内常称为ICODE SLI-S，主要用于物流仓储、票证管理等。
- ICODE SL2 ICS20：芯片厂家为荷兰恩智浦（NXP），国内常称为I CODE 2，主要用于物流仓储、票证管理等。
- ICODE SL2 ICS50/ICODE SL2 ICS51：芯片厂家为荷兰恩智浦（NXP），国内常称为ICODE SLI-L，主要用于物流仓储、票证管理等。
- Tag-it HF-1 Plus：芯片厂家为美国德州仪器（TI），国内常称为TI2048，主要用于物流仓储、票证管理等。
- Tag-it HF-1 Standard：芯片厂家为美国德州仪器（TI），国内常称为TI256，主要用于物流仓储、票证管理等。
- BL75R04：芯片厂家为上海贝岭，兼容TI 2048，主要用于物流仓储、票证管理等。
- BL75R05：芯片厂家为上海贝岭，兼容I CODE 2，主要用于物流仓储、票证管理等。
- FM1302N：芯片厂家为上海复旦，兼容I CODE 2，主要用于物流仓储、票证管理等。
- Advant ATC128-MV：芯片厂家为瑞士LEGIC，主要用于一卡通等。
- Advant ATC256-MV：芯片厂家为瑞士LEGIC，主要用于一卡通等。
- Advant ATC1024-MV：芯片厂家为瑞士LEGIC，主要用于一卡通等。
- LRI2K：芯片厂家为瑞士意法半导体（ST）。

---

## 参考文献

- [深入理解Android：Wi-Fi、NFC和GPS卷](https://static.kancloud.cn/alex_wsc/android-wifi-nfc-gps)
- [NFC-Forum官方网站的技术文档列表](https://nfc-forum.org/our-work/specification-releases/)

### 资料下载

- [ISO14443 Overview-v5.ppt](ISO14443 Overview-v5.ppt)
- [NFC技术指南](tn1216-st25-nfc-guide-stmicroelectronics.pdf)
- [ST意法半导体公司的NFC产品说明书](zh.NFC_Solutions_from_ST.pdf)
- [NFC在Linux的技术实现](Near_Field_Communication_with_Linux.pdf)
