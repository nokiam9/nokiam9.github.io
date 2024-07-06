---
title: 大模型学习笔记之四：Transformer
date: 2024-06-30 13:32:20
tags:
---

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

[Transformer 高级讲解 - Jay Alammar](https://jalammar.github.io/illustrated-transformer/)

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

解码器中的自注意力层的运行方式与编码器中的自注意力层略有不同：

- 在解码器中，自注意力层只允许关注输出序列中较早的位置。这是通过在自注意力计算中的softmax步骤之前屏蔽未来位置（将其设置为）来完成的。-inf
- “编码器-解码器注意力”层的工作方式与多头自注意力类似，只不过它从其下面的层创建其查询矩阵，并从编码器堆栈的输出中获取键和值矩阵。

### 掩码张量

掩代表遮掩，码就是我们张量中的数值，它的尺寸不定，里面一般只有1和0的元素，代表位置被遮掩或者不被遮掩，至于是0位置被遮掩还是1位置被遮掩可以自定义，因此它的作用就是让另外一个张量中的一些数值被遮掩，也可以说被替换, 它的表现形式是一个张量.

掩码张量的作用:

在transformer的编码器中, 掩码张量的主要作用是遮蔽掉源语言中对结果没有意义的字符而产生的注意力值（也就是说遮蔽掉对结果没有意义的字符而将注意力集中在有意义的词上面），比如说，对于一句话:“我爱美丽的中国”,我们要提取句子的主要部分，即"我爱中国",而屏蔽掉对结果没有意义的字符，比如"美丽"这个修饰词，以此提升模型效果和训练速度

在transformer的解码器中, 掩码张量的主要作用在应用attention时，有一些生成的attention张量中的值计算有可能已知了未来信息而得到的，未来信息被看到是因为训练时会把整个输出结果都一次性进行Embedding，但是理论上解码器的的输出却不是一次就能产生最终结果的，而是一次次通过上一次结果综合得出的，因此，未来的信息可能被提前利用. 所以，我们会进行遮掩. 比如在解码器准备生成第一个字符或词汇时，我们其实已经传入了第一个字符以便计算损失，但是我们不希望在生成第一个字符时模型能利用这个信息，因此我们会将其遮掩，同样生成第二个字符或词汇时，模型只能使用第一个字符或词汇信息，第二个字符以及之后的信息都不允许被模型使用.

上文经常提及“词嵌入”，原文的词嵌入是 Encoder 的输入，译文的词嵌入是 Decoder 的输入之一。我们可以用 pytorch 很方便地实现普通的词嵌入，但是这里的词嵌入还包含 Position Embedding。

首先翻译是与词的相对位置紧密相关的，尽管我们在 中将句子中的词按顺序排列，但实际上做矩阵乘法时，这种“顺序”是直接被忽略掉的，在参数看来，这些词就是无序的。我们必须 explicitly 在词嵌入的数值上体现相对位置。

### 注意力机制

![attendtion](attention.png)

- [Transformer架构解析](https://blog.csdn.net/m0_56192771/article/details/118087175)

---

## 参考文献

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