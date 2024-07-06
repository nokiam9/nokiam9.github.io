---
title: 大模型学习笔记之一：Tensor
date: 2024-06-30 13:25:25
tags:
---

## 一、张量的定义

张量（Tensor）是一个具有统一数据类型的多维数组，本质就是一组有序的数字。

张量的阶数（order）也称为维数（dimensions）、模态（modes）或方式（ways）。一个 N 阶张量是 N 个向量空间元素的张量积，每个向量空间都有自己的坐标系。实际上，引入 tensor 的目的就是把向量、矩阵推向更高的维度.

零阶张量是一个标量（scalar），一阶张量是一个矢量（vector），二阶张量是一个矩阵（matrix），三阶或更高阶的张量叫做高阶张量（tensor）。

![tensor](tensor.png)

在 PyTorch 中，`torch.Tensor`是存储和变换数据的主要工具。如果你之前用过 NumPy，你会发现 Tensor 和 NumPy 的多维数组非常类似。然而，Tensor 提供 GPU 计算和自动求梯度等更多功能，这些使 Tensor 数据类型更加适合深度学习。

## 二、张量的性质

### 张量的范数 - norm

张量的范数是其所有元素平方和的平方根，即:

这类似于矩阵 的 F范数(Frobenius norm).

### 张量的内积 - inner product

```python
import tensorflow as tf

# There can be an arbitrary number of axes (sometimes called "dimensions")
rank_3_tensor = tf.constant([
  [[0, 1, 2, 3, 4],
   [5, 6, 7, 8, 9]],
  [[10, 11, 12, 13, 14],
   [15, 16, 17, 18, 19]],
  [[20, 21, 22, 23, 24],
   [25, 26, 27, 28, 29]],])

print(rank_3_tensor)
```

输出结果如下：

```console
tf.Tensor(
[[[ 0  1  2  3  4]
  [ 5  6  7  8  9]]

 [[10 11 12 13 14]
  [15 16 17 18 19]]

 [[20 21 22 23 24]
  [25 26 27 28 29]]], shape=(3, 2, 5), dtype=int32)
```

## 三、张量的运算

### 基本运算

- torch.add()：加法
- torch.mul()：乘法
- torch.div()：除法
- torch.abs()：tensor内每个元素取绝对值
- torch.round()：tensor内每个元素取整数部分
- torch.frac()：tensor内每个元素取小数部分
- torch.log()：tensor内每个元素取对数
- torch.pow()：tensor内每个元素取幂函数
- torch.exp()：tensor内每个元素取指数
- torch.sigmoid()：tensor内每个元素取sigmoid函数值
- torch.mean()：tensor所有元素的均值
- torch.norm()：tensor所有元素的范数值
- torch.prod()：tensor所有元素积
- torch.sum()：tensor所有元素和
- torch.max()：tensor所有元素最大值
- torch.min()：tensor所有元素最小值

注意：例如add()这类函数一般都是会返回一直tensor值的，但是add_()这类加了下划线的方法是在tensor的原空间处理的，会覆盖之前的值

同样也可以是用以下方式直接处理

加 tensor1 + tensor2；
减 tensor1 - tensor2；
乘 tensor1 * tensor2；
除 tensor1 / tensor2；
内积 tensor1 @ tensor2；
幂运算 tensor1 ** n

### 线性代数运算

- torch.dot()：向量内积运算
- torch.mv()：矩阵与向量的乘法
- torch.mm()：矩阵乘法，仅适用二维矩阵；高维的矩阵乘法，使用 torch.matnul()
- torch.eig()：方阵的特征值和特征向量
- torch.inverse()：方阵的逆
- torch.ger()：两个向量的张量积

#### 连接、分片、变形

- torch.cat()：多个tensor的拼接
- torch.reshape()\torch.view()：返回一个张量，其数据和元素数量与输入相同，但具有指定的形状
- torch.transpose()\torch.t()：指定tensor的两个维度进行转置，torch.t()方法只适用与二维tensor
- torch.squeeze()/unsqueeze()：tensor对于张量中大小为1的维度的压缩与扩张
- torch.permute()：返回维度排列后的原始张量输入的视图。

---

## 参考文献

- [张量的概念及基本运算](https://blog.csdn.net/AIMZZY/article/details/106528350)
- [PyTorch - Tensor的相关概念及操作](https://www.cnblogs.com/young978/p/15678816.html)
- [深入浅出Pytorch - 张量](https://datawhalechina.github.io/thorough-pytorch/%E7%AC%AC%E4%BA%8C%E7%AB%A0/2.1%20%E5%BC%A0%E9%87%8F.html)
- [PyTorch学习笔记(二)：Tensor操作](https://www.jianshu.com/p/314b6cfce1c3)
- [PyTorch基础 - 张量Tensor线性运算（点乘、叉乘）](https://blog.51cto.com/u_11299290/3312822)
- [点积、叉积、内积、外积](https://blog.csdn.net/Dust_Evc/article/details/127502272)
