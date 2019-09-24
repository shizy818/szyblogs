---
title:  “Neural Networks and Deep Learning”
date:   2019-08-01 08:00:12 +0800
categories: deeplearning.ai
---

去年在[coursera deeplearning.ai](https://www.coursera.org/specializations/deep-learning)上了吴恩达的深度学习课程。一直想做一个笔记方便以后查询，结果拖了快一年；然后
发现网上已经有各种deeplearning.ai的笔记总结了。好歹还是自己写一下吧，主要是复习。

先说个题外话，吐槽一下AI的学术近亲。Andrew Ng是Micheal Jordon的学生，Pieter Abbeel是Andrew Ng弟子，Sergey Levine是Pieter Abbeel博士后。总结：贵圈很乱。


# 逻辑回归到神经网络

神经网络和深度学习（Neural Networks and Deep Learning）是deeplearning.ai的第一门课程，从逻辑回归(Logistic Regression)引出神经网络(Neural Network)，然后过渡到深度神经网络(Deep Neural Network)。 下图解释了，为什么逻辑回归可以视作一个非常简单的神经网络。

![image01]({{site.baseurl}}/image/20190801/LogReg_kiank.png)

对于每一个样本$$x^{(i)}$$：

$$
\begin{aligned}
&z^{(i)} = w^T x^{(i)} + b \\
&\hat{y}^{(i)} = a = \sigma(z^{(i)}) = \frac {1}{1 + e^{-z^{(i)}}} \\
&L(\hat{y}^{(i)}, y^{(i)}) = -y^{(i)} \log \hat{y}^{(i)} - (1 - y^{(i)}) \log (1 - \hat{y}^{(i)})
\end{aligned}
$$

代价函数是m个样本的交叉熵损失函数(Cross Entropy Loss Function):

$$
\begin{aligned}
&J = \frac{1}{m} \sum_{i=1}^m L(\hat{y}^{(i)}, y^{(i)})
\end{aligned}
$$

梯度下降算法，也可以被视为神经网络的反向传播算法：

$$
\begin{aligned}
\frac{\partial L}{\partial a} &= -\frac{y}{a}+\frac{(1-y)}{(1-a)} \\
\frac{\partial a}{\partial z} &= a \cdot (1-a) \\
\frac{\partial L}{\partial w_i} &= \frac{\partial z}{\partial w_i} \cdot \frac{\partial L}{\partial z} \\
&= x_i \cdot (a - y) \\
\frac{\partial L}{\partial b} &= \frac{\partial z}{\partial b} \cdot \frac{\partial L}{\partial z} \\
&= a - y
\end{aligned}
$$

1. 激活函数（Activation functions）

    逻辑回归中的激活函数是sigmoid函数，在神经网络中，有时候其他激活函数会有更好的效果。

    - tanh: 上下界分别是1和-1，和sigmoid函数相比均值接近0而不是0.5，对于下一层的学习效果更好。但是当z很大或者很小的时候，斜率趋向为0，使得梯度下降的学习速率降低，甚至无法收敛。

    - relu: z大于0时导数为1，解决梯度弥散问题(Gradient Vanishing)。

    - leaky relu: 使得z小于0时导数不为0。

    ![image02]({{site.baseurl}}/image/20190801/activation_func.png)

2. 参数随机初始化

    对于一个神经网络，权重随机初始化非常重要。假设把权重初始化为0， 所有隐藏单元完全对称，无论运行梯度下降多久，隐藏单元仍是相同的函数。

# 深层神经网络

深层神经网络即是隐藏层较多的神经网络。$$w^{[l]}$$记作第$$l$$层的权值，维度$$(n^{[l]}, n^{[l-1]})$$；然后通过激活函数$$g^{[l]}$$作用于$$z^{[l]}$$计算出激活结果记作$$a^{[l]}$$。

![image02]({{site.baseurl}}/image/20190801/dnn.png)

前向传播步骤写成，输入为$$a^{[l-1]}$$，缓存$$z^{[l]}$$：

$$
\begin{aligned}
z^{[l]} &= w^{[l]} \cdot a^{[l-1]} + b^{[l]} \\
a^{[l]} &= g^{[l]}(z^{[l]})
\end{aligned}
$$

反向传播步骤，输入为$$\frac{dL}{da^{[l]}}$$：

$$
\begin{aligned}
\frac{\partial L}{\partial z^{[l]}} &= \frac{\partial L}{\partial a^{[l]}} \cdot {g^{[l]}}'(z^{[l]}) \\
\frac{\partial L}{\partial w^{[l]}} &= \frac{\partial L}{\partial z^{[l]}} \cdot a^{[l-1]} \\
\frac{\partial L}{\partial b^{[l]}} &= \frac{\partial L}{\partial z^{[l]}} \\
\frac{\partial L}{\partial a^{[l-1]}} &= {w^{[l]}}^T \cdot \frac{\partial L}{\partial z^{[l]}}
\end{aligned}
$$

参考：  
[Coursera深度学习教程中文笔记](https://github.com/fengdu78/deeplearning_ai_books)  
[Coursera深度学习课件](https://github.com/stormstone/deeplearning.ai)  
[deeplearning.ai作业](https://github.com/colinback/deeplearning.ai)