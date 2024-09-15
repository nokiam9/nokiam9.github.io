---
title: QAM - 正交振幅调制的技术原理
date: 2024-09-15 21:27:22
tags:
---

调制（modulation）是一种将一个或多个周期性的载波（carrier wave）混入想发送之信号（称为基带，baseband）的技术，常用于无线电波的传播与通信、利用电话线的数据通信等各方面。
解调（demodulation）是调制的逆过程，就是在接收端把已搬到给定信道通带内的频谱还原为基带信号的过程。

调制 / 解调的目的是为了将信号的频谱**搬移**到任意位置，从而有利于信号的传送，并且使频谱资源得到充分利用。例如，天线尺寸为信号的十分之一或更大些，信号才能有效的被辐射。然而，对于语音信号来说，相应的天线尺寸要在几十公里以上，但实际上不可能实现。因此，这时就需要调制过程将信号频谱搬移到较高的频率范围。此外，调制 / 解调技术还可以将相同频率范围的信号分别依托于不同频率的载波上，接收机就可以分离出所需的频率信号，不致互相干扰。这也是在同一信道中实现多路复用的基础。
> 载波的频率（也称为通带，passband）通常**远高于**调制信号的带宽（bandwith，即最高频率与最低频率之间的差值）。

## 一、调制技术的分类

![types-of-modulation](types-of-modulation.png)

按照载波信号（也被称为被调信号）可以分为三类：正弦波调制、脉冲调制与强度调制。调制的载波分别是正弦波，脉冲和光波。

![AM+FM](AM+FM.png)
![PM](PM.png)


依调制信号的不同，可区分为数字调制及模拟调制，这些不同的调制，是以不同的方法，将信号和载波合成的技术。

数字调制，就是研究如何把比特 bk 映射到符号 In 。数字调制的基本方法是调幅、调相和调频。
调幅ASK，幅移键控，英文amplitude shift keying
![ASK](ASK.png)

调频FSK,频移键控，frequency shift keying。
![FSK](FSK.png)

调相PSK,相移键控，phase shift keying
![PSK](PSK.png)






按照基带信号（也被称为调制信号）可以分为两类：模拟调制与数字调制。顾名思义，模拟调制中基带信号是模拟信号，数字调制中基带信号是数字信号。

调制的另一种分类方法是角度调制和幅度调制，其中角度调制包括调频和调相，幅度调制包括调幅AM、双边带调制DSB、单边带调制SSB、残留边带调制VSB和正交幅度调制QAM[^3]。

根据已调信号的频谱结构是否保留了原来消息信号的频谱模样，可以分为线性调制与非线性调制[^5]。幅度调制一般都是线性调制，角度调制都是非线性调制。

$$C(t)=A \cos( \omega_ct + \phi_0)$$
其中：$A$ 是载波幅度，$\omega_c$ 是载波频率，$\phi_0$ 是载波相位。





---

## 参考文献

- [星座图通信原理](https://zhuanlan.zhihu.com/p/594337231)
- [Wave Modulation – analog and digital](https://telecom.altanai.com/tag/qam/)
- [信号调制与解调](https://zhuanlan.zhihu.com/p/517843859)

### 文档下载

- [数字信号的载波传输](数字信号的载波传输.pdf)

### 视频教材

- [QAM 通信原理1 - 大连理工大学](https://www.bilibili.com/video/BV1Xu4y1J77o/?spm_id_from=333.999.0.0&vd_source=735a6376f6214c7b974a1074096ba0fa)
- [QAM 通信原理2 - 大连理工大学](https://www.bilibili.com/video/BV1Ge411C7ee/?spm_id_from=333.788.recommend_more_video.-1&vd_source=735a6376f6214c7b974a1074096ba0fa)
- [正交相移键控 (QPSK) BPSK 和 QPSK QPSK 波形（数字调制技术）](https://www.bilibili.com/video/BV1Zb4y1z7Tj/?spm_id_from=333.999.0.0&vd_source=735a6376f6214c7b974a1074096ba0fa)