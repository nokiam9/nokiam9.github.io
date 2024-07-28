---
title: 大模型学习笔记之四：Transformer
date: 2024-07-10 13:32:20
tags:
---

[Transformer 高级讲解 - Jay Alammar](https://jalammar.github.io/illustrated-transformer/)

## 一、概述

Transformer总体架构可分为四个部分：输入、输出、编码器、解码器。

![tr](transformer-c.png)

1. 输入部分：
   - 源文本嵌入层（Inputs Embedding），及其位置编码器（Positional Encoding）
   - 目标文本嵌入层（Outputs Embedding），及其位置编码器（Positional Encoding）
2. 输出部分:
    - 线性层
    - softmax层
3. 编码器部分：
    - 由 N 个编码器层堆叠而成，每个编码器层由 2 个**子层**连接结构组成
    - 第一个子层连接结构包括：一个多头自注意力子层，规范化层，一个残差连接
    - 第二个子层连接结构包括：一个前馈全连接子层，规范化层，一个残差连接
4. 解码器部分:
    - 由 N 个解码器层堆叠而成，每个解码器层由 3 个**子层**连接结构组成
    - 第一个子层连接结构包括：一个**Masked**多头自注意力子层，规范化层，一个残差连接
    - 第二个子层连接结构包括：一个多头注意力子层（**不是自注意力**），规范化层，一个残差连接
    - 第三个子层连接结构包括：一个前馈全连接子层，规范化层，一个残差连接

## 二、源（目标）文本嵌入层

### 掩码张量

掩代表遮掩，码就是我们张量中的数值，它的尺寸不定，里面一般只有1和0的元素，代表位置被遮掩或者不被遮掩，至于是0位置被遮掩还是1位置被遮掩可以自定义，因此它的作用就是让另外一个张量中的一些数值被遮掩，也可以说被替换, 它的表现形式是一个张量.

掩码张量的作用:

在transformer的编码器中, 掩码张量的主要作用是遮蔽掉源语言中对结果没有意义的字符而产生的注意力值（也就是说遮蔽掉对结果没有意义的字符而将注意力集中在有意义的词上面），比如说，对于一句话:“我爱美丽的中国”,我们要提取句子的主要部分，即"我爱中国",而屏蔽掉对结果没有意义的字符，比如"美丽"这个修饰词，以此提升模型效果和训练速度

在transformer的解码器中, 掩码张量的主要作用在应用attention时，有一些生成的attention张量中的值计算有可能已知了未来信息而得到的，未来信息被看到是因为训练时会把整个输出结果都一次性进行Embedding，但是理论上解码器的的输出却不是一次就能产生最终结果的，而是一次次通过上一次结果综合得出的，因此，未来的信息可能被提前利用. 所以，我们会进行遮掩. 比如在解码器准备生成第一个字符或词汇时，我们其实已经传入了第一个字符以便计算损失，但是我们不希望在生成第一个字符时模型能利用这个信息，因此我们会将其遮掩，同样生成第二个字符或词汇时，模型只能使用第一个字符或词汇信息，第二个字符以及之后的信息都不允许被模型使用.

上文经常提及“词嵌入”，原文的词嵌入是 Encoder 的输入，译文的词嵌入是 Decoder 的输入之一。我们可以用 pytorch 很方便地实现普通的词嵌入，但是这里的词嵌入还包含 Position Embedding。

首先翻译是与词的相对位置紧密相关的，尽管我们在 中将句子中的词按顺序排列，但实际上做矩阵乘法时，这种“顺序”是直接被忽略掉的，在参数看来，这些词就是无序的。我们必须 explicitly 在词嵌入的数值上体现相对位置。

## 三、多头注意力子层

解码器中的自注意力层的运行方式与编码器中的自注意力层略有不同：

- 在解码器中，自注意力层只允许关注输出序列中较早的位置。这是通过在自注意力计算中的 softmax 步骤之前屏蔽未来位置（将其设置为）来完成的。-inf
- “编码器-解码器注意力”层的工作方式与多头自注意力类似，只不过它从其下面的层创建其查询矩阵，并从编码器堆栈的输出中获取键和值矩阵。

### 1. （带掩码的）多头注意力组件

![attendtion](attention.png)

词向量已经提供了基于字典表的、单个 token 的客观语义，自注意力机制的目的是提供基于当前上下文的、token 之间的主观语义，并据此对输入向量进行修正。

与基于直线的线形变换相比，矩阵二次型实际上是基于圆锥曲线的高纬变换，因此有着更强的表达能力。

Q 表示查询向量，K 表示关键字向量，V 表示值向量
可以认为，$Q$ 和 $K$ 分别负责**设定语义**和**表达语义**。

![交叉注意力](qkv-2.png)

$$
\begin{align*}
A &= X \cdot W_q \cdot [X \cdot W_k]^T \\\\
    &= X \cdot [W_q \cdot W_k^T] \cdot X^T \\\\
    &= X \cdot W_A \cdot X^T
\end{align*}
$$

词汇表的容量为 $n_{vocab}$，词向量的维度是 $d_{model}$（也称为 d_embed）

输入数据为：

- $T_1$：基准输入数据，形状为 $(length_{ctx1}, d_{model})$
- $T_2$：对照输入数据，形状为 $(length_{ctx2}, d_{model})$
- 如果是自注意力机制，则 $T_2=T_1$

注意力机制的层数是 $n_{layers}$，如果是单头注意力机制：

- 1 个 $W_q$：Query 权重矩阵，形状为$(length_{ctx1}, d_{model})$
- 1 个 $W_k$：Key 权重矩阵，形状为$(length_{ctx2},  d_{model})$
- 1 个 $W_v$：Value 权重矩阵，形状为$(length_{ctx2}, d_{model})$
- 中间结果 $A$ 的形状为 $(length_{ctx1}, length_{ctx2})$

> TODO: 3Blue1Brown 视频视频中说 Value 是 12288 * 12288 的矩阵，分解为 12288 * 128 乘以 128 * 12288 的矩阵乘法，参数分别是 value 和 output，与我的理解有差异！！！

如果是多头注意力机制，则有$d_{head} = d_{model} \div n_{heads}$（也称为 d_query），相应的：

- $n_{heads}$ 个 $W_q$：Query 权重矩阵，形状为$(length_{ctx1}, d_{head})$
- $n_{heads}$ 个 $W_k$：Key 权重矩阵，形状为$(length_{ctx2},  d_{head})$
- $n_{heads}$ 个 $W_v$：Value 权重矩阵，形状为$(length_{ctx2}, d_{head})$

输出数据的形状为 $(length_{ctx1}, d_{model})$，也就是与输入数据的形状完全系统。

![GPT-3](gpt3.png)

### 2. 规范化组件

### 3. 残差连接组件

## 四、前馈全联接子层

GPT-3 的前馈全联接子层包含 2 个串联的线性组件，分别是 Up-projection 和 Down—projection，主要参数是：

- Up-projection：输出数据的形状为 $(length_{ctx1}, d_{model})$，包含 $n_{neurons}$ 个神经元，输出数据的形状为 $(length_{ctx1}, n_{neurons})$
- Down—projection：输入就是 Up-projection 的输出，包含 $d_{model}$ 个神经元，输出数据的形状为 $(length_{ctx1}, d_{model})$

> TODO: LLM张老师说，神经元的数量 = 词向量维度 * 4，为什么？
> 以及，QKV 的参数矩阵 ，还要加上一个 W-output矩阵，为什么？

## 五、输出部分

---

## 附录一：Gemma 7B 对比分析

以 Gemma 7B 为例，分析其配置文件

![arch](arch.jpg)

与最近大语言模型的研究趋势一致，我们的网络也基于 Transformer 架构（Vaswani 等，2017）。 但做了很多改进，也借鉴了其他模型（例如 PaLM）中的一些技巧。

2.2.1 改进
以下是与原始架构的主要差异，

预归一化（Pre-normalization）：受 GPT3 启发
为了提高训练稳定性，我们对每个 Transformer sub-layer 的输入进行归一化，而不是对输出进行归一化。 这里使用由 Zhang 和 Sennrich（2019）提出的 RMSNorm 归一化函数。

SwiGLU 激活函数：受 PaLM 启发
用 SwiGLU 激活函数替换 ReLU 非线性，该函数由 Shazeer（2020）提出，目的是提升性能。 但我们使用的维度是 2/3 * 4d，而不是 PaLM 中的 4d。

旋转嵌入（Rotary Embeddings）：受 GPTNeo 启发
去掉了绝对位置嵌入（absolute positional embeddings），并在每个网络层中添加旋转位置嵌入（rotary positional embeddings，RoPE）。 RoPE 由 Su 等（2021）提出。

```python
# 词汇表的大小：The number of tokens in the vocabulary.
vocab_size: int = 256000
# The maximum sequence length that this model might ever be used with.
max_position_embeddings: int = 8192
# 28 个隐藏层：The number of blocks in the model.
num_hidden_layers: int = 28
# 16 个注意力 head ：The number of attention heads used in the attention layers of the model.
num_attention_heads: int = 16
# The number of key-value heads for implementing attention.
num_key_value_heads: int = 16
# The hidden size of the model.
hidden_size: int = 3072
# The dimension of the MLP representations.
intermediate_size: int = 24576
# The number of head dimensions.
head_dim: int = 256
# The epsilon used by the rms normalization layers.
rms_norm_eps: float = 1e-6
# The dtype of the weights.
dtype: str = 'bfloat16'
# Whether a quantized version of the model is used.
quant: bool = False
# The path to the model tokenizer.
tokenizer: Optional[str] = 'tokenizer/tokenizer.model'
```

- 词汇表的容量：256K
- 隐藏层的数量：28
- 隐藏层的维度：3072
- 自注意力 head 的数量：16

![VS](MQA.png)

MHA（Multi-head Attention）是标准的多头注意力机制，包含 H 个Query、Key 和 Value 矩阵。所有注意力头的 Key 和 Value 矩阵权重不共享。

MQA（Multi-Query Attention，多查询注意力）是 19 年提出的一种 Attention 机制，其能够在保证模型效果的同时加快 decoder 生成 token 的速度。MQA 是将 head 中的 key 和 value 矩阵抽出来单独存为一份共享参数，而 query 则是依旧保留在原来的 head 中，每个 head 有一份自己独有的 query 参数。

GQA（Grouped-Query Attentio，分组查询注意力）是一种针对 Transformer 模型中 Multi-head Attention 的改进方法，旨在提高模型的运算速度，同时保持预测质量。在标准的Multi-head Attention中，每个注意力头都是独立计算的，这导致了计算和存储需求较高。分组查询注意力通过将查询头分组，让每组共享一个键头和值头，从而减少计算和存储需求。

MQA 和 MHA 主要是在计算 K 和 V 的过程中有计算量的差异，由于训练阶段由于数据是并行的，这种差异整体不明显，而在推理阶段，在 memery cache的基础上，MQA 的推理速度有明显提升，同时也更省内存。

---

## Transformer 架构解析





- [Transformer架构解析](https://blog.csdn.net/m0_56192771/article/details/118087175)

---

## 附录一：残差网络（ResNet）

Residual Network（残差网络，ResNet）是一种深度卷积神经网络（CNN）架构，由 Kaiming He（何凯明）在 2015 年提出，核心思想是通过引入“残差学习”（residual learning）来解决深度神经网络训练中的退化问题（degradation problem）。

在传统的深度神经网络中，随着网络层数的增加，理论上网络的表示能力应该更强，但实际上，过深的网络往往难以训练，性能反而不如层数较少的网络。这种现象被称为“退化问题”，即随着网络深度的增加，网络的准确率不再提升，甚至下降。
![error](resnet0.png)

ResNet通过引入“跳跃连接”（skip connections）或“捷径连接”（shortcut connections）来解决这个问题。在ResNet中，输入不仅传递给当前层，还直接传递到后面的层，跳过一些中间层。这样，后面的层可以直接学习到输入与输出之间的残差（即差异），而不是学习到未处理的输入。这种设计允许网络学习到恒等映射（identity mapping），即输出与输入相同，从而使得网络可以通过更简单的路径来学习到正确的映射关系。

![res](resnet.png)

在 Transformer 模型中，残差网络的使用主要是为了解决自注意力机制（self-attention）带来的问题。Transformer模型完全基于注意力机制，没有卷积层，但其结构本质上也是深度网络。在 Transformer 中，每个编码器（encoder）和解码器（decoder）层都包含自注意力和前馈网络，这些层的参数量非常大，网络深度也很容易变得很深。使用残差连接可以帮助Transformer模型更有效地训练深层网络。在 Transformer 的自注意力层中，输入通过自注意力和前馈网络后，与原始输入相加，形成残差连接。这种设计使得网络即使在增加更多层数时，也能保持较好的性能，避免了退化问题。

参考[https://zh.d2l.ai/chapter_convolutional-modern/resnet.html](https://zh.d2l.ai/chapter_convolutional-modern/resnet.html)

```python
import torch
from torch import nn
from torch.nn import functional as F
from d2l import torch as d2l


class Residual(nn.Module):  #@save
    def __init__(self, input_channels, num_channels,
                 use_1x1conv=False, strides=1):
        super().__init__()
        self.conv1 = nn.Conv2d(input_channels, num_channels,
                               kernel_size=3, padding=1, stride=strides)
        self.conv2 = nn.Conv2d(num_channels, num_channels,
                               kernel_size=3, padding=1)
        if use_1x1conv:
            self.conv3 = nn.Conv2d(input_channels, num_channels,
                                   kernel_size=1, stride=strides)
        else:
            self.conv3 = None
        self.bn1 = nn.BatchNorm2d(num_channels)
        self.bn2 = nn.BatchNorm2d(num_channels)

    def forward(self, X):
        Y = F.relu(self.bn1(self.conv1(X)))
        Y = self.bn2(self.conv2(Y))
        if self.conv3:
            X = self.conv3(X)
        Y += X
        return F.relu(Y)
```

---

## 参考文献

- [Attention Is All You Need](https://arxiv.org/pdf/1706.03762)

### 精品视频

- [认识Transformer架构和代码实现（合集）- 浙大教授](https://www.bilibili.com/video/BV1sW4y1J7cL/?spm_id_from=333.999.0.0&vd_source=735a6376f6214c7b974a1074096ba0fa)
- [Deep Learning 合集 - 3Blue1Brown](https://space.bilibili.com/88461692/channel/seriesdetail?sid=1528929)

### 官方文档

- [Gemma: Open Models Based on Gemini Research and Technology](gemma-report.pdf)
- [Gemma Pytorch - Github](https://github.com/google/gemma_pytorch)

### 技术解读

- [Gemma模型论文详解](https://blog.csdn.net/qinduohao333/article/details/136264993)
- [LLM常见问题（Attention 优化部分）](https://juejin.cn/post/7310061802464264242)

### LLAMA 解读

- [LLaMA: Open and Efficient Foundation Language Models](2302.13971v1.pdf)
- [上面论文的中文翻译](https://arthurchiao.art/blog/llama-paper-zh/)
- [深入解析 LLaMA 如何改进 Transformer 的底层结构](https://www.cnblogs.com/huaweiyun/p/17881295.html)
