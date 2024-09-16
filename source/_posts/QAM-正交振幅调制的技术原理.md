---
title: QAM - 正交振幅调制的技术原理
date: 2024-09-15 21:27:22
tags:
---

在信号处理中，通常将传感器从其他物理量转换来的电信号称为基带信号（baseband signal）。基带信号包含了原始信号的所有频谱成分，通常在很低的频率范围内，例如麦克风的电子输出等。

调制（modulation）是一种将一个或多个周期性的载波（carrier wave）混入基带信号的技术，常用于无线电波的传播与通信、利用电话线的数据通信等各方面。解调（demodulation）是调制的逆过程，即在接收端把已搬到给定信道通带内的频谱还原为基带信号的过程。

调制 / 解调技术的目的是为了实现信号频谱的**搬移**，从而有利于信号的传送，并且使频谱资源得到充分利用。例如，天线尺寸为信号的十分之一或更大些，信号才能有效的被辐射。然而，对于语音信号来说，相应的天线尺寸要在几十公里以上！因此就需要调制过程将信号频谱搬移到较高的频率范围。此外，调制 / 解调技术还可以将相同频率范围的信号分别依托于不同频率的载波上，接收机就可以分离出所需的频率信号，不致互相干扰。这也是在同一信道中实现多路复用的基础。

> 载波的频率通常**远高于**基带信号的频率范围。

## 一、调制技术的分类

按照载波信号（被调信号）的类型，可以分为模拟载波（Analog carrier，通常为正弦波），或着数字载波（Digital carrirt，通常为脉冲信号）。

按照基带信号（调制信号）的类型，可以分为模拟调制（Analog data，连续型信息，例如广播电台）与数字调制（Digital data，离散型信息，例如GSM、WiFI等）。

![types-of-modulation](types-of-modulation.png)

## 二、模拟调制方法

在模拟调制中，调制是连续施加在模拟载波之上，以响应模拟信息信号。
最常见的模拟载波是**等幅单频正弦波**，其优势在于它们能够直接表示实际的物理量，可以提供更自然的再现，此外技术实现比较简单成熟，成本较低，因此特别适合无线通信。数学表达式为：
$$C(t)=A \cos( \omega_ct + \phi_0)$$

其中：$A$ 是载波幅度，$\omega_c$ 是载波频率，$\phi_0$ 是载波相位，调制过程中改变的就是这 3 个参数。

- AM（Amplitude Modulation，幅度调制）：载波信号的幅度根据调制信号的瞬时幅度而变化
    ![AM](AM.png)
- FM（Frequency Modulation，频率调制）：载波信号的频率根据调制信号的瞬时幅度而变化
    ![AM+FM](AM+FM.gif)
- PM（Phase Modulation，相位调制）：载波信号的相移根据调制信号的瞬时幅度而变化
    ![PM](PM.gif)

## 三、数字调制方法

在数字调制中，模拟载波信号由离散信号调制。数字调制方法可以被认为是数模（A/D）转换，相应的解调或检测可以被认为是模数转换。载波信号的变化是从有限数量的 M 个替代符号（调制字母表）中选择的。大多数教科书将数字调制方案视为一种数字传输形式，与数据传输同义。

### 1. 二进制数字调制

调制信号为二进制数字信号时，这种调制称为二进制数字调制。
与模拟调制方法类似，数字调制的最基本方法就是调幅、调相和调频。在二进制数字调制中，载波的幅度、频率或相位只有两种变化状态。

- ASK（Amplitude Shift Keying，幅移键控）：也称为 2ASK
    ![ASK](ASK.png)
- FSK（Frequency Shift Keying，频移键控）：也称为 2FSK
    ![FSK](FSK.png)
- PSK（Phase Shift Keying，相移键控）：也称为 2PSK
    ![PSK](PSK.png)

### 2. 多进制数字调制

二进制数字调制中，每一个符号只能表示 0-1 两个数值，为了提高数据传输效率，可以在一个符号内传输更多的比特，从而提高频带利用率，这就是多进制数字调制。
简单来说，就是把原来的 0-1 比特流进行分组，例如两个 bit 为一组映射到一个符号（symbol、码元）上，那么一个符号就有 00, 01, 11, 10 四种调制的取值。

- 在码元速率（传码率）相同条件下，M 进制数字传输系统的信息速率是二进制的 $\log_2M$ 倍，从而提高系统的有效性
- 与此等价的，在信息速率相同条件下，M 进制码元宽度是二进制的 $\log_2M$ 倍，因此每个码元的能量增加，而码元速率相应降低，减小了码间串扰的影响，从而提高了传输的可靠性。
- 在接收机输入信噪比相同条件下，多进制数字传输系统的误码率相应增高，同时也增加了发射功率和实现上的复杂性。

在数字调制中，载波参数（幅度，频率和相位）的变化由离散的数字信号决定。数字调制信号只须表示离散的调制状态，这些离散状态在矢量图上称为符号点 (symbol point)，符号点的组合称为星座图(constellation)。

对于相位调制来说，二进制数字调制也称为 BPSK（Binary Phase Shift Keyring），而多进制数字调制依次为 QPSK（Quadrature Phase Shift Keying，也称 4PSK）和 8PSK。

![8PSK](8PSK.png)

从其星座图分析发现：

- 所有符号点平均分布在一个同心圆，圆周半径相等于信号幅度
- 在信号幅度相同（即功率相等）的条件下，进制数 M 增加意味着相邻符号点的距离（相位差异）越小
- 换句话说，更高阶的 PSK 调制技术将导致系统误码率增高

因此，为了进一步提高传输效率，不适合采用高于 8PSK 的数字调制技术，而是改为 ASK + PSK 的联合调制方式！

## 四、IQ 调制

### 1. 数学原理

IQ 调制很早就在模拟调制技术中得到应用，最初是为了解决多个信号的合成问题。

#### 传统的信号合成

传统的信号合成是采用三角函数的**乘法**。
定义：载波信号为$\cos \omega_ct$，基带信号为$s(t)=\cos \omega t$，则调制结果为
$$ s_{RF}(t) = s(t) \cos \omega_ct = \cos \omega t \cos \omega _ct 
    = \frac {\cos(\omega_c - \omega)t + \cos(\omega_c + \omega)t} 2
$$

#### 正交的信号合成

所谓正交合成，就是载波信号和基带信号的计算采用三角函数的**减法**，即
$$ s_{RF}(t) = \cos (\omega_ct - \omega t)
    = \cos\omega_ct \cos \omega t + \sin \omega _ct \sin \omega t
$$

令：$ I = \cos \omega t ,Q = \sin \omega t $ ，则有 $ s_{RF}(t) = I \cos \omega_ct + Q \sin \omega_ct$
在调制环节，其结果是生成了一个单一的正弦波，其频率是 $\omega_ct - \omega t$ 。


在解调环节，分别处理 I 分支和 Q 分支：
对于 I 分支：
$$
\begin {align}
    [I \cos \omega_ct  + Q \sin \omega_ct].2\cos \omega_ct & = I.2 \cos ^2\omega_ct + Q.\sin 2\omega_ct \\\\
    & = I.(1+\cos 2\omega_ct) + Q.\sin 2\omega_ct \\\\
    & = I + I.\cos 2\omega_ct + Q.\sin 2\omega t
\end {align}
$$

对于 Q 分支：
$$
\begin {align}
    [I \cos \omega_ct  + Q \sin \omega_ct].2\sin \omega_ct & = I.\sin 2\omega_ct + Q.2 \sin ^2\omega_ct \\\\
    & = I.\sin 2\omega_ct + Q.(1-\cos 2\omega_ct) \\\\
    & = Q - Q.\cos 2\omega_ct + I.\sin 2\omega_ct
\end {align}
$$

在 IQ 调制中，信号被分解为两个分量独立地携带信息，从而提高数据传输的效率。

- **I（In-phase）分量**：同相分量，与载波信号同相位。
- **Q（Quadrature）分量**：正交分量，与载波信号相位相差90度（或π/2弧度）。

数学表示通常为：
$$ s(t) = I(t) \cos(\omega_c t) - Q(t) \sin(\omega_c t) $$
其中，$\omega_c $ 是载波的角频率，$I(t)$ 和 $Q(t)$ 分别是调制信号的同相和正交分量。
![IQ](IQ.gif)
> BPSK 可以看作是一个特例，其中只有 I 分量携带信息，而 Q 分量 为零。


![IQ](IQ-struct.jpg)
![alt text](IQ-A.png)
![alt text](IQ-D.png)

数学上，BPSK信号可以表示为：
$$ s(t) = \cos(\omega_c t + \pi b(t)) $$
其中，$b(t)$ 是一个二进制数据序列，取值为0或1，对应于信号的相位变化。

### 从IQ到BPSK

在IQ调制中，如果将Q分量设置为0，并且根据I分量的值改变相位，就可以实现BPSK。具体来说：

- 当 $ I(t) = 1 $ 时，信号为 $ \cos(\omega_c t) $。
- 当 $ I(t) = -1 $ 时，信号为 $ cos(\omega_c t + \pi) $。

### 应用和优势

使用IQ调制可以轻松地扩展到更复杂的调制方案，如QPSK（四相位调制）、QAM（正交幅度调制）等，这些调制方案可以提供更高的数据传输速率。而BPSK作为一种简单的调制方式，由于其实现简单和对信道要求较低，常用于低速率或高干扰环境中的通信系统。

总结来说，BPSK是IQ调制的一个特殊情况，它利用了IQ调制框架中的同相分量来传输信息，而忽略了正交分量。这种关系使得从BPSK到更复杂的IQ调制方案的过渡变得自然而直接。

## QAM（Quadrature Amplitude Modulation，正交幅度调制）

正交幅度调制 （QAM） 是现代电信中广泛用于传输信息的数字调制方法系列和模拟调制方法的相关系列的名称。它通过使用幅移键控 （ASK） 数字调制方案或幅度调制 （AM） 模拟调制方案来改变（调制）两个载波的幅度，从而传送两个模拟消息信号或两个数字位流。两个载波的频率相同，并且彼此异相 90°，这种情况称为正交或正交。传输的信号是通过将两个载波相加来创建的。在接收器处，由于它们的正交性，两个波可以相干分离（解调）。

相位调制（模拟 PM）和相移键控（数字 PSK）可以看作是 QAM 的一种特殊情况，其中传输信号的幅度是恒定的，但其相位会发生变化。这也可以扩展到频率调制 （FM） 和频移键控 （FSK），因为这些可以被视为相位调制的特例。

QAM 广泛用作数字通信系统的调制方案，例如在 802.11 Wi-Fi 标准中。通过设置合适的星座图大小，QAM 可以实现任意高频谱效率，仅受通信信道的噪声水平和线性度限制。随着比特率的增加，QAM 正在光纤系统中使用;QAM16 和 QAM64 可以用三通道干涉仪进行光学仿真。

### 数学表达

在 QAM 中，载波的振幅和相位同时受到基带信号控制，因此一个码元可以表示为：
$$S_k(t) = A_k \cos (\omega_0t + \theta_k) = A_k \cos \theta_k \cos \omega_0t - A_k \sin \theta_k \sin \omega_0t$$
其中，$k$ 是整数，$A_k$ 和 $\theta_k$ 分别可以取多个离散值，载波频率为 $\omega_0$
令 $X_k = A_k \cos \theta_k, Y_k = -A_k \sin \theta_k$，则信号调制结果为：
$$ S_k(t) = X_k \cos \omega_ot + Y_k \sin \omega_0t$$

在接收器处，相干解调器将接收到的信号分别与余弦和正弦信号相乘，以产生接收到的 I（t） 和 Q（t） 估计值。例如：
低通滤波 r（t） 去除高频项（包含 4πfct），只留下 I（t） 项。此滤波信号不受 Q（t） 的影响，表明同相分量可以独立于正交分量接收。同样，我们可以将 sc（t） 乘以正弦波，然后用低通滤波器提取 Q（t）。



### 正交调幅法

用两路正交的 4ASK 信号叠加，形成 16QAM 信号

### 复合相移法

用两路独立的 4PSK 信号叠加，形成 16QAM 信号。但实际很少采用。

![QAM](QAM.png)

### APSK（Amplitude and Phase-Shift Keyring，幅频联合键控）

幅度和相移键控 （APSK） 是一种数字调制方案，它通过调制载波的幅度和相位来传输数据。换句话说，它结合了幅度偏移键控 （ASK） 和相移键控 （PSK）。与单独的 ASK 或 PSK 相比，这允许在给定的调制阶数和信噪比下降低误码率，但代价是复杂性更高。[1]

正交幅度调制 （QAM） 可以被视为 APSK 的子集，因为所有 QAM 方案都同时调制载波的幅度和相位。传统上，QAM 星座是矩形的，而 APSK 星座是圆形的，但情况并非总是如此。两者之间的区别在于它们的生产;QAM 由两个正交信号产生。与传统 QAM 相比，APSK 的优势在于可能的幅度级别数量较少，因此峰均功率比 （PAPR） 较低。[2]APSK 的低 PAPR 对放大器和信道非线性的弹性使其对卫星通信（包括 DVB-S2）特别有吸引力。[3]

在 APSK 中，一如其名，幅度和相位均被调制。与 QAM 不同的是， 它的星座点分布在 I/Q平面中的同心圆上。 这个概念被引入到卫星系统（射频功率放大器具有非线性特性）。因此需要一个能够容忍 非线性放大的调制方案（包含较少的幅度），以便更轻松地平衡这种非线性。

图 19 对比了16QAM星座图和 16-APSK星座图，其中 16QAM星座图有 3 个幅度，而 16APSK星座图只有 2 个。32QAM星座图有 5 个幅度，而 32-APSK星座图 有 3 个。注意：QAM振铃的间距是不均匀的， 有的振铃间距很窄，从而加剧了非线性补偿的难度。

在光纤中，APSK 可应用到非线性噪声场景中，用于改善对非线性光纤特征的容忍度。当数据速率高达 400 Gbps 及以上时，16QAM星座点间距较大，更易实施且光信噪比性能更佳，因而是首选的方案。

![APSK](APSK.png)

---

## 附录一：数字通信传输

数字通信传输方法主要有两大类：基带（baseband）和通带（passband）。

在基带传输中使用线路编码，从而产生脉冲序列或数字脉冲幅度调制 （PAM） 信号。这通常用于未过滤的电线，例如光纤电缆和短距离铜链路，例如：V.29 （EIA/TIA-232）、V.35、IEEE 802.3、SONET/SDH。

在通带传输中采用数字调制方法，以便在某些带通滤波通道中仅使用有限的频率范围。通带传输通常用于无线通信和带通滤波通道，例如 POTS 线路。它还允许频分多路复用。数字 bitstream 首先转换为等效的 baseband 信号，然后转换为 RF 信号。在接收器侧，解调器用于检测信号并反转调制过程。

用于调制和解调的组合设备就称为 [Modem - modulator &demodulator](https://en.wikipedia.org/wiki/Modem)。

---

## 参考文献

- [Modulation - Wiki](https://en.wikipedia.org/wiki/Modulation)
- [星座图通信原理](https://zhuanlan.zhihu.com/p/594337231)
- [Wave Modulation – analog and digital](https://telecom.altanai.com/tag/qam/)
- [信号调制与解调](https://zhuanlan.zhihu.com/p/517843859)
- [IQ 调制基本理论及解调简述](http://www.spectrumscience.cn/page93?article_id=145)
- [详解IQ调制以及星座图原理](https://blog.csdn.net/weixin_44586473/article/details/104066625)
- [Josh 的学习笔记之数字通信（Part 4——带通调制和解调）](https://blog.csdn.net/weixin_43870101/article/details/106543036)

### 文档下载

- [数字信号的载波传输](数字信号的载波传输.pdf)
- [QAM Demodulation](qam.pdf)

### 视频教材

- [QAM 通信原理1 - 大连理工大学](https://www.bilibili.com/video/BV1Xu4y1J77o/?spm_id_from=333.999.0.0&vd_source=735a6376f6214c7b974a1074096ba0fa)
- [QAM 通信原理2 - 大连理工大学](https://www.bilibili.com/video/BV1Ge411C7ee/?spm_id_from=333.788.recommend_more_video.-1&vd_source=735a6376f6214c7b974a1074096ba0fa)
- [正交相移键控 (QPSK) BPSK 和 QPSK QPSK 波形（数字调制技术）](https://www.bilibili.com/video/BV1Zb4y1z7Tj/?spm_id_from=333.999.0.0&vd_source=735a6376f6214c7b974a1074096ba0fa)
- [IQ信号的理解](https://www.bilibili.com/video/BV1Au4y1d7TQ/?spm_id_from=333.999.0.0&vd_source=735a6376f6214c7b974a1074096ba0fa)