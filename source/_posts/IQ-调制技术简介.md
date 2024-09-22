---
title: IQ 调制技术简介
date: 2024-09-22 13:57:28
tags:
---

IQ 调制很早就在模拟调制技术中得到应用，最初是为了解决多个信号的合成问题。

传统的信号合成是采用三角函数的**乘法**。
定义：载波信号为$\cos \omega_ct$，基带信号为$s(t)=\cos \omega t$，则调制结果为
$$ s_{RF}(t) = s(t) \cos \omega_ct = \cos \omega t \cos \omega _ct
    = \frac {\cos(\omega_c - \omega)t + \cos(\omega_c + \omega)t} 2
$$

![class](class.png)

测试结果显示 1hz 和 10hz 的两个频率相乘后生成了一个 10-1,10+1 两个频率，过滤其中一个就可以还原出基带信号。然而，两个**对称**的频率脉冲比较接近，剔除不太容易，还浪费了宝贵的频率带宽。

## 一、基本原理

在 IQ 调制中，信号被分解为两个正交的分量独立地携带信息，从而提高数据传输的效率。

- **I（In-phase）分量**：同相分量，与载波信号同相位。
- **Q（Quadrature）分量**：正交分量，与载波信号相位相差90度（或π/2弧度）。

![IQ](IQ.gif)

## 二、调制环节

载波信号和基带信号的调制采用三角函数的**减法**，结果是一个单一的正弦波，频率是 $\omega_c - \omega $ 。

$$ s_{RF}(t) = \cos (\omega_ct - \omega t)
    = \cos\omega_ct \cos \omega t + \sin \omega _ct \sin \omega t
$$

令：$ I(t) = \cos \omega t ,Q(t) = \sin \omega t $ ，则：
$$ s_{RF}(t) = I(t) \cos \omega_ct + Q(t) \sin \omega_ct $$

其中，$\omega_c $ 是载波的角频率，$I(t)$ 和 $Q(t)$ 分别是调制信号的同相和正交分量。

![modern](modern.png)

### 调制的技术实现

1. 载波发生器提供一个 $\cos(\omega_ct)$ 余弦信号，并通过**移相**生成一个 $\sin(\omega_ct)$ 正弦信号；
2. Q 基带信号和正弦载波信号**相乘**，I 基带信号和余弦载波信号**相乘**；
3. 再将两路信号**相加**，最后得到调制结果信号。

![IQ](IQ-struct.jpg)

## 三、解调环节

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

上面的数学结果似乎很复杂，但由于 $2\omega_c$ 频率远高于载波频率 $\omega_c$，使用一个低通滤波器就可以剔除，结果就是纯净的 I 和 Q 基带信号。

### 解调的技术实现

1. 接受端的载波发生器与发送端完全相同，提供一个 $\cos(\omega_ct)$ 余弦信号，并通过**移相**生成一个 $\sin(\omega_ct)$ 正弦信号；
2. 收到的调制信号和余弦载波信号**相乘**，通过低通滤波器清除高频部分，得到纯净的 I 基带信号
3. 收到的调制信号和正弦载波信号**相乘**，通过低通滤波器清除高频部分，得到纯净的 Q 基带信号

![alt text](IQ-A.png)

## 四、数字调制方式

IQ 模式同样可以用于数字调制方式，只不过数字信息要通过数模转换生成和判决。

![alt text](IQ-D.png)

## 五、代码实例

### 发送端代码

```python
# NumPy 是一个数学函数库，支持维度数组与矩阵运算
import numpy as np

# Scipy 是基于 Numpy 的高级科学计算库，支持线性代数、积分、微分、快速傅里叶变换、信号处理和图像处理
import scipy.signal as sig

# Pyplot 是 Matplotlib 的子库，提供了和 MATLAB 类似的绘图 API
import matplotlib.pyplot as plt         

t_max = 1.5                                 # 计算 2 秒
fs = 60                                     # 采样周期 1/60
fc = 1                                      # 基础频率为 1 Hz               

times = np.arange(0, t_max, 1/fs)           # 采样的时间序列，90个元素的数组
phases = 2 * np.pi * fc * times             # 采样的相位序列，90个元素的数组
new_ticks = np.arange(0, t_max, 1/8)        # 每个周期有 8 个刻度 
new_lables = new_ticks * fs                 # 计算每个刻度的时间序列号

def signal_I(I):
    return I * np.cos(phases)

def signal_Q(Q):
    return Q * np.sin(phases)

def signal_IQ(I,Q):
    return signal_I(I) + signal_Q(Q)

def test_send():
    I=3
    Q=4

    # plot 是标准的坐标图，参数是 X、Y、Label
    # 分别绘制 I、Q、IQ 三条曲线
    plt.plot(times, signal_I(I), label="signal_I(" +  str(I) + ")" )      
    plt.plot(times, signal_Q(Q), label="signal_I(" +  str(Q) + ")" )    
    plt.plot(times, signal_IQ(I,Q), label="signal_IQ(" +  str(I) + "," + str(Q)+ ")" ) 

    plt.legend()                                        # 图例的位置，默认自适应
    plt.grid()                                          # 显示网格线
    plt.xticks(ticks=new_ticks, labels=new_lables)      # X 轴的刻度 ticks & 标签 label 
    plt.show()                                          # 开始屏幕显示

test_send()
```

![send](send.png)

### 接收端代码

```python
import numpy as np
import scipy.signal as sig
import matplotlib.pyplot as plt

t_max = 1.5
fs = 360
fc = 10

times = np.arange(0, t_max, 1/fs)
phases = 2 * np.pi * fc * times
new_tickets = np.arange(0, t_max, 1/8)
new_lables = new_tickets * fs

def signal_I(I):
    return I * np.cos(phases)

def signal_Q(Q):
    return Q * np.sin(phases)

def signal_IQ(I,Q):
    return signal_I(I) + signal_Q(Q)

def test_receive():
    cut_off = 2
    # 使用窗口法的 FIR 滤波器，[滤波器的长度，截止频率]
    b = sig.firwin(51, cut_off/fs/2)     
    # 生成 IQ 调制信号       
    signal = signal_IQ(1,2)                    

    plt.plot(times, signal, label='signal_IQ(1,2)')
    # 使用 IIR 或 FIR 滤波器沿一维筛选数据，[分子向量，分母向量，输入序列]
    plt.plot(times, sig.lfilter(b, 1, 2 * signal * np.cos(phases)), label='I')
    plt.plot(times, sig.lfilter(b, 1, 2 * signal * np.sin(phases)), label='Q')
    
    plt.grid()
    plt.xticks(ticks=new_tickets, labels=new_lables)
    plt.legend()
    plt.show()

test_receive()
```

![receive](receive.png)

---

## 附录一：时域和频域

时域（time domain）：是信号在时间轴随时间变化的总体概括。是真实世界存在的域，可以通过示波器来看。
频域（frequency domain）：描述频率变化和幅度变化的关系，是把时域波形的表达式做傅立叶等变化得到复频域的表达式。是一个存在于数学定义中的域，可以通过频谱仪来看。

![图示](sp.gif)

从时域到频域，转换方法就是傅里叶变换，有三种类型：

- FI（Fourier Integral，傅里叶积分）：用于将时域内的理想数学表达式变换为频域表示，是将时域时间轴从负无穷到正无穷积分，得出从零到正无穷上连续的频域函数。
- DFT（Discrete Fourier Transform，离散傅里叶变换）：是傅里叶变换在时域和频域上都呈离散的形式，将信号的时域采样变换为其 DTFT（Discrete-time Fourier Transform，离散时间傅里叶变换）的频域采样。
    离散傅里叶变换不需要积分计算，只需要通过求和就可以实现转换。
- FFT（Fast Fourier Transform，快速傅里叶变换）：只应用于时域中数据点个数是 2 的整数次幂的情况（如 256、512、1024 点），使用了快速矩阵代数学的方法，计算速度可以比普通离散傅里叶变换快很多。
    需要注意的是，快速傅里叶变换要求信号是周期重复的，所以需要对原始信号进行相干采样，或在采样后加窗处理。

IFFT（Inverse Fast Fourier Transform，逆快速傅里叶变换）算法是 FFT 的逆过程，它将频域信号转换回时域信号。在数学上，IFFT是FFT的逆运算，用于恢复原始的时域信号。

![傅立叶变换](IFFT+FFT.png)

## 附录二：数字滤波器

数字滤波器是利用计算机程序、专用芯片等软、硬件改变数字信号频谱。如果要处理的是模拟信号可以通过 A/D 在信号形式上进行变换，在利用数字滤波器处理后经过 D/A 恢复为模拟信号。

根据冲激响应的不同，数字滤波器分为有限冲激响应（FIR）滤波器和无限冲激响应（IIR）滤波器。

- FIR 滤波器：冲激响应在有限时间内衰减为零，其输出仅取决于当前和过去的输入信号值。
- IIR 滤波器：冲激响应理论上应会无限持续，其输出不仅取决于当前和过去的输入信号值，也取决于过去的信号输出值。

### FIR 滤波器

FIR（Finite impulse response，有限冲激响应），是冲激响应为有限长度的滤波器，脉冲输入信号的响应会在有限时间内变为零。FIR 滤波器的表达式，也称为离散卷积：
$$ y[n] = b_0x[n] + b_1x[n-1] + b_2x[n-2] + ... = \sum _{i=0}^N b_i \cdot x[n-i] $$
其中:

- $N$ 是滤波器阶数，$N^{th}$ 阶滤波器表示在右边有 $ N+1$ 项
- $b_i$ 是 $N^{th}$ 阶滤波器在第 $i$ 时间的脉冲响应。
- $x[n]$ 是输入信号
- $y[n]$ 是输出信号

### IIR 滤波器

IIR（Infinite Impulse Response，无限冲激响应），是数字滤波器的一种。由于无限冲激响应滤波器中存在反馈回路，因此对于脉冲输入信号的响应是无限延续的。IIR 滤波器的表达式：
$$ y[n] = \sum _{i=0}^P b_ix[n-i] + \sum _{i=1}^Q a_iy[n-i] $$
其中：

- $P$ 是前馈滤波器顺序
- $b_i$ 是前馈滤波器系数
- $Q$ 是反馈滤波器顺序
- $a_i$ 是反馈滤波器系数
- $x[n]$ 是输入信号
- $y[n]$ 是输出信号

---

## 参考文献

- [In-phase and quadrature components - Wiki](https://en.wikipedia.org/wiki/In-phase_and_quadrature_components)
- [IQ 调制基本理论及解调简述](http://www.spectrumscience.cn/page93?article_id=145)
- [详解IQ调制以及星座图原理](https://blog.csdn.net/weixin_44586473/article/details/104066625)
- [Josh 的学习笔记之数字通信（Part 4——带通调制和解调）](https://blog.csdn.net/weixin_43870101/article/details/106543036)
- [常用的信号处理函数 Scipy 之滤波器](https://blog.csdn.net/qq_36002089/article/details/127793378)
- [如何通俗易懂地理解 FIR/IIR 滤波器](https://www.zhihu.com/question/323353814)
- [详解 FIR 滤波器与 IIR 滤波器的具体区别](https://www.jianshu.com/p/c825679ea991)

### 视频教材

- [IQ 信号的理解](https://www.bilibili.com/video/BV1Au4y1d7TQ/?spm_id_from=333.999.0.0&vd_source=735a6376f6214c7b974a1074096ba0fa)
- [QAM 通信原理2 - 大连理工大学](https://www.bilibili.com/video/BV1Ge411C7ee/?spm_id_from=333.788.recommend_more_video.-1&vd_source=735a6376f6214c7b974a1074096ba0fa)
