---
title: NFC技术和产品综述
date: 2021-08-15 18:05:45
tags:
---

## 概述

NFC（`Near Field Communication`，近场通信），也叫做近距离无线通信技术，用于在两个非常接近的设备之间执行数据传输。

2002年，NFC技术由`Philips`和`Sony`两家公司联合推出。
2004年，`Nokia`、`Philips`、`Sony`等公司共同组建了一个非盈利性组织`NFC Forum` ，负责制定NFC相关的技术标准，致力于促进 NFC 技术在消费类电子产品、移动设备和 PC 中的应用，并通过NFC认证测试来保证各厂家的NFC产品符合NFC规范以实现互联互通。

> 2006年，恩智浦半导体公司(NXP Semiconductors)从飞利浦公司中剥离，总部位于荷兰埃因霍温，在全球20多个国家拥有37000名员工（欧洲37%、亚洲37%、大中华区21%、美洲5%），2007年公布的销售额达63亿美元。

NFC的核心技术优势在于：

- **无源支持**：采用感应耦合技术，NFC标签可以从其他NFC设备产生的电磁场获得能量，经济实惠且完全无源，特别适合IoE（物联网）应用场景
- **公开频段**：电磁场频率为 13.56 MHz，该频率是射频频谱高频部分中无需执照的波段，有效距离为4cm左右，目前所支持的数据传输速率有106Kbps、212Kbps和424Kbps三种
- **快速启动**：用户无需事先启动APP（特定场景可能要求安全验证），基于硬件支持即可触发NFC，交互时间只有几分之一秒
- **终端集成**：NFC软件协议栈完全集成到Android（v.4.0以上）、iOS（v.11）中，有助于在多个应用程序中使用NFC，而无需安装任何特定软件或应用

## NFC技术的发展历程

NFC是多种技术路线并行发展的综合体，得益于条形码、磁条卡和智能终端三种技术的共同推动。

{% asset_image nfc-map.png %}

1. 最左边一条为发源于条形码（Barcodes）的无线射频识别技术，即RFID技术路线，最终出现了NFC中的两个重要组件`NFC Tag`（标签）和`NFC Reader`(读卡器)。

2. 最右边一条为磁条卡（Magnetic Strip Cards）技术路线，该路线后来向智能卡（Smart Card）方向发展，最终演化出了NFC使用的Proximity Coupling Smart Card，有效距离在10cm之内，对应的规范为`ISO/IEC 14443(非接触式IC卡标准)`，从RF（Radio Frequency，无线频率）层面定义了如何与不同的非接触式IC卡（其实物可以是NFC Tag、RFID Tag、Smart Cards）交互。

    Vicinity Coupling Smart Card对应的规范为`ISO/IEC 15693`(也称为**疏耦合卡**)，突出优势是**读取距离可高达一米**，同时设计简单使得读取器的生产成本低，在企业门禁卡中得到广泛应用。

    > Close Coupling Smart Card对应的规范是`ISO 10536`，该标准主要发展于1992到1995年间，由于这种卡的成本高，优点少，因此从未在市场上销售。

    粗看上去Smart Card和RFID Tag类似，例如二者都只存储一些数据，而且自身都没有电源组件，但Smart Card在安全性上的要求远比RFID Tag严格。另外，Smart Card上还能运行一些小的嵌入式系统（如Java Card OS）或者应用程序（Applets）以完成更为复杂的工作。

3. 中间一条技术路线为移动终端，它演化出了携带NFC功能的终端设备。随着移动终端越来越智能，NFC和这些设备也融合得更加紧密，使得NFC的应用场景得到了较大的拓展。

NFC涉及到多个标准化组织定义的协议，下图是简化的协议栈结构，下面将有详细描述。

{% asset_image protocol-1.png %}

## NFC的通信模式

从用户角度（即图中的Applications层之上）来看，NFC有三种运行模式（operation mode），分别是

- Reader/Write模式（简称R/W，和NFC Tag/NFC Reader相关）
- Peer-to-Peer模式（简称P2P，它支持两个NFC设备交互）
- Card Emulation模式（简称CE，它能把携带NFC功能的设备模拟成Smart Card，这样就能实现诸如手机支付、门禁卡之类的功能）

{% asset_image nfc-mode.png %}

### 读卡器模式（Reader/Write Mode）

数据在NFC Tag的芯片中，可以简单理解成**刷标签**，本质上就是通过支持NFC的手机或其它电子设备从带有NFC芯片的标签、贴纸、名片等媒介中读写信息，随后它可能根据读取来自部件的信息执行操作。例如，靠近 NFC 标签的 NFC 手机会检索到一个 URL，并链接到相应的网站，NFC手机可以在无需打字的情况下发送 SMS （短消息服务）文本、获得优惠券、启动配对动作、获得电子名片（Vcard）等。

通常NFC标签是不需要外部供电的，当支持NFC的外设向NFC读写数据时，它会发送某种磁场，而这个磁场会自动的向NFC标签供电。

> NFC Tag的数据通常是ReadOnly，无法支持余额更新等业务需求，而且**由于使用了 NFC 论坛定义的消息格式，数据传输是不安全的**。

### 卡模拟模式（Card Emulation Mode）

该模式下，NFC 设备的工作类似于标准的非接触式智能卡。这让它可以用于已有的非接触式智能卡基础设施，实现访问控制、非接触式支付、固件更换或数据传输等操作。
仿真智能卡的 NFC 设备通常工作在被动 NFC 模式，**此时的数据传输是安全的**。

### 点对点模式（Peer-to-Peer Mode）

在点对点 （P2P）中，具有 NFC 功能的设备工作在主动模式。其中的一个设备会发起通信链接。一旦链接建立，设备就会交替地与其他设备通信，并遵守 “ 说前先听 （listen-beforetalk） ” 规则。该通信模式下的数据交换相比其他模式更快，因此可以交换更多的数据。


NFC使用的是无线射频技术。在RF层，与之相关的规范是ISO 18092（NFC Interface and Protocol I，简称NFCIP-1，该规范定义了NFC RF层的工作流程）和ISO 14443 Type A，Type B和Felica。

ISO 14443全称为非接触式IC卡标准，它从RF层面定义了如何与不同的非接触式IC卡（其实物可以是NFC Tag、RFID Tag、Smart Cards）交互。ISO 14443定义了Type A和Type B两种非接触式IC卡。

Type A最早由Philips公司制订（其生产的芯片商标名为MIFARE，现在由从Philips独立出来的NXP公司拥有，目前世界上70%左右的非接触式IC卡都使用了MIFARE芯片，例如北京市的公交卡），
Type B（主要用在法国市场）由其他公司制订，二者最终都成为ISO标准。
Felica（也被称为Type F）由Sony开发，它最终没有成为ISO标准，而是成为日本工业标准JIS X6319-4，所以Felica主要用于日本市场。
Type A、B和F主要区别在于RF层的信号调制解调方法、传输速率及数据编码方式上。关于ISO 14443和Felica之间的区别，请读者阅读参考资料[4]。
RF层之上是Mode Switch，它用于确定对端NFC Device的类型并选择合适的RF层协议与之通信。


## NFC的协议栈



{% asset_image protocol-2.png %}

{% asset_image protocol-3.png %}


## NFC的工作模式

{% asset_image rf.jpg %}

`ISO 18092`协议介绍了P2P通讯中的Active模式和Passive通讯模式，其实`ISO 18092`使用了`ISO 14443`协议和非国际标准的`FELICA`通讯协议，



## NFC Tag的分类

{% asset_image nfc-tag.jpg %}

---

## 参考文献

- [深入理解Android：Wi-Fi、NFC和GPS卷](https://static.kancloud.cn/alex_wsc/android-wifi-nfc-gps)
- [NFC-Forum官方网站的技术文档列表](https://nfc-forum.org/our-work/specification-releases/)