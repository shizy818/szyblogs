---
title:  “Attention & Transformer”
mathjax: true
layout: post
date:   2019-12-27 08:00:12 +0800
categories: deep learning
---

去年BERT(Bidirectional Encoder Representation from Transformer)在NLP火的一塌糊涂。作为小白的我一脸懵逼，为了不至于跟人聊天的时候被鄙视，还是了解一下啥是Attention和Transformer。


# Seq2Seq

Attention之前的RNN变种应该是Seq2Seq模型(Encoder-Decoder)，对应输入和输出长度不一致的场景，广泛应用于自然语言处理，机器翻译，语音文字识别等等。

![image01]({{site.baseurl}}/image/20191227/seq2seq.jpg)

Seq2Seq结构中，编码器Encoder把所有输入序列编码成一个统一的语义向量Context，然后由解码器Decoder解码。其存在的问题：
1. 输入序列较长时，Context可能无法保留足够多信息
2. 解码时只用到了编码器的最后一个隐藏状态层

# Attention

注意力机制的基本思路就是利用Encoder所有的隐藏层状态$$h_t$$。

![image02]({{site.baseurl}}/image/20191227/attention.jpg)

其中重点就是如何确定权重$$w_{ij}$$，或者在另外的图例中用$$\alpha$$表示。

![image03]({{site.baseurl}}/image/20191227/attention_model.jpg)

1. 首先定义score函数$$f(z_t, h_s)$$可以有三种，**注：公式中$$z_t, h_s$$对应上图中$$z^t, h^s$$**
    * Dot: $$z_t^T h_s$$
    * General: $$z_t^T W_\alpha h_s$$
    * Concat: $$v_\alpha^T tanh(W_\alpha [z_t; h_s])$$

2. 通过Softmax计算权重
    $$
    \alpha_t^s = \frac {e^{f(z_t, h_s)}}{\sum_{s'} e^{f(z_t, h_{s'})}}
    $$

3. 计算所有隐藏层状态$$h_t$$加权之和
    $$
    c_t = \sum \limits_s \alpha_t^s \cdot h_s
    $$

4. 生成的$$c_t$$可以有多种方式添加到网络
    * ADD（直接将$$z_t$$叠加在输出$$y_t$$上：
    $$
    y_t = y_t + z_t
    $$
    * CONCAT(将$$z_t$$拼接在隐藏层$$h_t$$后全连接恢复维度)
    $$
    h_t = W_c \cdot [c_t; h_t]
    $$

    ![image04]({{site.baseurl}}/image/20191227/attention_zt.jpg)

在论文和代码实现中，需要向量化加速计算，于是有矩阵`Q(query), K(key), V(value)`。`query`对应解码器隐藏状态$$z$$，`key`和`value`对应编码器隐藏状态$$h$$。生成状态$$c$$计算如下：

![image05]({{site.baseurl}}/image/20191227/attention_imp.png)

# Transformer

Transformer本质上是一个Encoder-Decoder结构；原文中的Encoders由6层编码器组成，同样Decoders由6层解码器组成。每个编码器包含一个Self-Attention层和前向网络层(Feed Forward Nerual Network)。

Self-Attention允许模型看到句子其他位置的词作为辅助信息，从而更好的编码当前词；计算过程如下图：

![image06]({{site.baseurl}}/image/20191227/self_attention.png)

直观上的效果如下图，类似一个相关矩阵。`The animal`对`it`的权重最大，所以`The animal`的词向量很大程度上被合进`it`的编码中。

![image07]({{site.baseurl}}/image/20191227/self_attention_visualization.png)

Multi-Head Attention相当于h个不同的Self-Attention的集成。每一组Head随机初始化，输入向量被映射到不同的子空间中。当前词在不同组里，和其他不同位置的词发生联系。

![image08]({{site.baseurl}}/image/20191227/multi_head.png)

其他的细节，比如解码器结构和Encoder-Decoder Attention，可以参考下面的链接或者原论文。

参考：  
[The Illustrated Transformer](https://blog.csdn.net/yujianmin1990/article/details/85221271)  
[The Annotated Transformer](https://nlp.seas.harvard.edu/2018/04/03/attention.html)  
[Attention原理和源码解析 - 知乎](https://zhuanlan.zhihu.com/p/43493999)  
[详解Transformer - 知乎](https://zhuanlan.zhihu.com/p/48508221)  
[Attention和Transformer - 知乎](https://zhuanlan.zhihu.com/p/38485843)
[完全解析RNN, Seq2Seq, Attention注意力机制](https://zhuanlan.zhihu.com/p/51383402)
