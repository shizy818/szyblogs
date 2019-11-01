---
title:  “玻尔兹曼机”
date:   2019-10-16 08:00:12 +0800
categories: deeplearning
---

上一篇提到如果Hopfield是无自反馈对称连接网络，则最终会收敛到一个能量最小值，但可能只是局部最小。因此需要一些方法来摆脱局部最小值。随机神经网络就是加入随机噪声，使搜索陷入局部最优时具有一定的“爬山”能力。John Hopfield在1982年提出Hopfield网络；Geoffrey E. Hinton大概在1983年提出玻尔兹曼机。


在Hopfield网络中，原来一个神经元的输入$$\sum \limits_j w_{ij}s_j + \theta_i$$一旦确定，则神经元处于激活或是关闭状态也就确定了。而玻尔兹曼机的每个神经元是1还是0，还需要由一个随机函数确定，如下：

$$
\begin{aligned}
&E = - \sum \limits_{i,j} w_{ij} s_i s_j - \sum \limits_i \theta_i s_i \\
&p(s_i = 1) = \frac {1}{1 + e^{-\Delta E_i / T}} \\
&\Delta E_i = E(s_i = 0) - E(s_i = 1) = \sum \limits_j w_{ij} s_j + \theta_i
\end{aligned}
$$

玻尔兹曼机和Hopfield网络一样，是一个基于能量的模型，并受到了模拟退火算法的影响。温度$$T$$一定时，如果神经元开启比关闭能量低，即$$\Delta E_i$$为正，logistic函数自变量为正，这个神经元为1的概率大于0.5；反之同理。由于每个神经元的开关具有随机性，所以可能跳出局部最小值。

能量差$$\Delta E_i$$一定时，温度越高，logistic函数自变量越靠近0点，即这个神经元为0或者1的随机性越大。温度$$T$$很低时，$$\Delta E_i$$的符号决定开启或关闭。

![image01]({{site.baseurl}}/image/20191016/logistic.png)

# 理解玻尔兹曼机的热平衡

玻尔兹曼机中的能量对应玻尔兹曼机中不同的状态，经过足够长时间的运行，最后达到一个**平稳分布**。每种状态出现的概率是确定的，取决于这个全局状态的能量，而和开始状态无关。

$$
p(v) = \frac {1}{Z} e^{-E(v)} = \frac {e^{-E(v)}}{\sum \limits_u e^{-E(u)}}
$$

**Hopfield网络最终收敛到一个能量最低状态，而玻尔兹曼机的目标是网络稳定后，某个状态能量越小，其出现的概率越大**。神经元直接的连接权重$$w_{ij}$$和每个神经元的偏置$$\theta_i$$，导致了最终不同的概率分布。

玻尔兹曼机有几个神经元，则概率分布的随机变量就是几维的。现在给定一组训练数据，这组数据是从一个未知的分布中采样的。学习一组权值，使得玻尔兹曼机收敛后的平稳分布近似这个未知分布，就是玻尔兹曼机的学习问题。

# 玻尔兹曼机训练过程

我们的目标是让给定训练数据在玻尔兹曼机模型中以最大概率出现。具体而言，给定一个训练样本，代入玻尔兹曼机模型中取代状态向量$$v$$，然后最大化$$P(v)$$。因为有很多训练样本，所以最后就是求$$P(v)$$在这些样本上的期望最大化。

前面提到数据的维度和玻尔兹曼机神经元的个数相同，实际可能仅仅知道一部分神经元取值。因此玻尔兹曼机被分为两部分，可见的叫visible units，不可见的叫hidden units，目标变成了训练网络使得玻尔兹曼机在稳定态时visible units概率分布和样本尽可能接近。

![image02]({{site.baseurl}}/image/20191016/bolzimann.png)

定义$$P(V_\alpha)$$为可见神经元被训练数据决定时，可见神经元状态向量$$V_\alpha$$出现的概率；$$P'(V_\alpha)$$指不受环境影响，达到平稳分布后可见神经元状态向量$$(V_\alpha)$$出现的概率。$$P(V_\alpha)$$和$$w_{ij}$$没有关系；而$$P'(V_\alpha)$$由$$w_{ij}$$确定。用KL散度衡量两个分布的差异：

$$
\begin{aligned}
min: G &= \sum \limits_\alpha P(V_\alpha) ln \frac {P(V_\alpha)}{P'(V_\alpha)} \\
P'(V_\alpha) &= \sum \limits_\beta P'(V_\alpha \land H_\beta) \\
&= \frac {\sum \limits_\beta e^{-E_{\alpha,\beta}/T}} {\sum \limits_{\lambda,\mu} e^{-E_{\lambda,\mu}/T}}
\end{aligned}
$$

上式中$$V_\alpha \land H_\beta$$表示可见神经元状态是$$\alpha$$且隐藏神经元状态是$$\beta$$；$$\lambda,\mu$$表示两者的状态是任意值。求导过程省略，结果如下：

$$
\begin{aligned}
\frac {\partial G}{\partial w_{ij}} &= - \frac{1}{T} (p_{ij} - p'_{ij}) \\
\Delta w_{ij} &= \varepsilon (p_{ij} - p'_{ij})
\end{aligned}
$$

训练迭代：一个是正阶段(Positive Phase)，可见神经元状态被限制为从训练数据分布$$P(V_\alpha)$$采样的二值向量，然后根据运行规则调整隐藏单元，直到达到网络平衡；如此处理完所有训练样本后，统计该连接两端的神经元在热平衡状态时同时激活的频率。另一个是负阶段(Negative Phase)，神经元不受任何限制，自由演变直到达到平衡状态；重复若干次后，统计出两个神经元同时激活的频率。

![image03]({{site.baseurl}}/image/20191016/bm_train.png)

**思考:**玻尔兹曼机应该不适用于监督学习比如联想记忆，输入不同于记忆样本的任意向量，运行网络以一定概率收敛于某个状态。加上的隐藏神经元，相当于改变了Hopfield网络的功能，可以用隐藏神经元的状态来解释输入。所以波兹曼机更适合非监督学习，学习结束后隐藏神经元即代表输入的隐藏特征。

**附:**去年G Hinton在coursera上开了[Neural Networks for Machine Learning](http://www.cs.toronto.edu/~tijmen/csc321/)的课，可惜今年就下线了。有哥们整理了一下[机器学习中的神经网络](http://yuexit.github.io/2016/03/18/NN-for-ML-1of3.html#course-information)笔记。

参考：  
[百度文库 - Hopfield Nets and Boltzmann Machine](https://wenku.baidu.com/view/3d16db8364ce0508763231126edb6f1aff0071f5.html)
