---
title:  “Improving Deep Neural Networks”
mathjax: true
layout: post
date:   2019-08-05 08:00:12 +0800
categories: deeplearning.ai
---

改善深层神经网络(Improving Deep Neural Network)是deeplearning.ai的第二门课程，主要内容包括正则化，优化以及超参数调试。


# 过拟合

提到过拟合，先复习一下偏差和方差的概念。

- 高偏差(High Bias)： 模型不能很好的拟合数据集，称之为“欠拟合”(underfitting)
- 高方差(High variance)： 模型过度拟合数据，称之为“过拟合”(overfitting)

解决深度学习中的过拟合问题，一个是正则化，另一个是准备更多的数据。

1. L2正则化

    L1（
    $$||w||_1 = \sum\limits_{j=1}^n |w_j|$$
    ）正则化的特点是使得参数$$w$$稀疏，也就是说$$w$$向量中很多0。然而在训练神经网络模型时，人们越来越倾向使用L2正则化（
    $$||w||_2^2 = \sum\limits_{j=1}^n w_j^2$$
    ）。带L2正则化的损失函数定义为：

    $$
    \begin{aligned}
    &J = \frac{1}{m} \sum_{i=1}^m L(\hat{y}^{(i)}, y^{(i)}) + \frac{\lambda}{2m} \sum_{l=1}^L ||w^{[l]}||_F^2
    \end{aligned}
    $$

    梯度下降算法对$$w^{[l]}$$的修正公式为：

    $$
    w^{[l]} := (1- \frac {\alpha \lambda}{m}) w^{[l]} - \alpha \frac {\partial J} {\partial w^{[l]}}
    $$

    对于正则化的直观理解就是：当$$\lambda$$增加到足够大，$$w$$会接近于0，等同于屏蔽了这些隐藏单元，深层神经网络被简化为一个很小的网络，于是不容易发生过度拟合。

2. Dropout正则化

    另外一个实用的正则化方法随机失活(Dropout)：在深度神经网络的每一层，以一定的概率随机消除一些节点，然后得到一个节点更少，规模更小的节点，通过反向传播算法进行训练。

    ![image01]({{site.baseurl}}/image/20190805/dropout.png)

    需要注意的一点是，消除节点的节点会造成$$a^{[l]}$$期望值的偏移，因此在计算$$z^{[l]} = w^{[l]} \cdot a^{[l-1]} + b^{[l]}$$时，需要除以keep-prob概率值来修正期望值。在验证和预测阶段，dropout函数是被忽略的，因此也不需要补偿。

3. 数据扩增 (Data Augmentation)

    以图片训练数据为例，可以通过水平翻转，模糊，裁剪，旋转等方法来增加训练集。

4. Early Stopping

    训练迭代过程中，训练误差通常都是下降的。而验证集误差通常会先呈下降趋势，然后在某个节点处开始上升。Early Stopping即是在此处停止训练。

# 加速方法

深度神经网络的训练通常需要花很长时间，如何加速成为深度学习一个广泛讨论的问题。

1. 归一化输入（零均值，归一化方差）

    ![image02]({{site.baseurl}}/image/20190805/normalization.png)

    为什么归一化输入可以加快训练呢？如果不进行归一化处理，代价函数细长狭窄。如上图左所示，从当前的初始位置运行梯度下降法，必须使用一个很小的学习步长，经过多次迭代才能找到最小值。如果函数是一个圆形球状轮廓，不论从哪个位置开始，梯度下降都能使用较大步长找到最小值。

2. Mini-batch梯度下降

    把训练集分割为小一点的子集训练，这些子集被取名为mini-batch；1 epoch遍历了一次整个训练集。使用batch梯度下降法时，每次迭代你都需要历遍整个训练集，可以预期每次迭代成本都会下降，所以如果成本函数$$J$$是迭代次数的一个函数，它应该会随着每次迭代而减少，如果在某次迭代中增加了，那肯定出了问题，也许你的学习率太大。使用mini-batch梯度下降法，则并不是每次迭代都是下降的。你很可能会看到这样的结果，走向朝下，但有更多的噪声。

3. 动量梯度下降(Momentum)

    ![image02]({{site.baseurl}}/image/20190805/momentum.png)

    以上图为例，动量梯度下降旨在抑制梯度的锯齿下降。下面公式中$$v_{dw}$$和$$v_{db}$$表示微分的指数加权平均。在几轮迭代后，纵轴方向的摆动变小且趋近于0，横轴方向运动加快。

    $$
    \begin{aligned}
    v_{dw} &= \beta v_{dw} + (1 - \beta) dw \\
    v_{db} &= \beta v_{db} + (1 - \beta) db \\
    w &:= w - \alpha v_{dw} \\
    b &:= b - \alpha v_{db}
    \end{aligned}
    $$

4. RSMprop(Root Mean Squre prop)

    RSMprop算法用$$w^2$$和$$b^2$$分别取代了$$w$$和$$b$$。假设$$b$$代表纵轴方向，$$s_{db}$$较大，因此纵轴方向的运动减缓。

    $$
    \begin{aligned}
    s_{dw} &= \beta v_{dw} + (1 - \beta) dw^2 \\
    s_{db} &= \beta v_{db} + (1 - \beta) db^2 \\
    w &:= w - \alpha \frac {dw}{\sqrt{s_{dw}}} \\
    b &:= b - \alpha \frac {db}{\sqrt{s_{db}}}
    \end{aligned}
    $$

5. Adam

    将Momentum和RSMprop结合起来就是Adam算法，$$t$$表示迭代次数。

    $$
    \begin{aligned}
    v_{dw} &= \beta_1 v_{dw} + (1 - \beta_1) dw \\
    v_{db} &= \beta_1 v_{db} + (1 - \beta_1) db \\
    s_{dw} &= \beta_2 s_{dw} + (1 - \beta_2) dw^2 \\
    s_{db} &= \beta_2 s_{db} + (1 - \beta_2) db^2 \\
    v_{dw}^c &= \frac {v_{dw}}{1 - \beta_1^t} \\
    v_{db}^c &= \frac {v_{db}}{1 - \beta_1^t} \\
    s_{dw}^c &= \frac {s_{dw}}{1 - \beta_2^t} \\
    s_{db}^c &= \frac {s_{db}}{1 - \beta_2^t} \\
    w &:= w - \alpha \frac {v_{dw}^c}{\sqrt{s_{dw}^c} + \epsilon} \\
    b &:= b - \alpha \frac {v_{db}^c}{\sqrt{s_{db}^c} + \epsilon} \\
    \end{aligned}
    $$

6. Batch归一化

    在深度神经网络中，可以将每一层中的$$z^{[l]}$$做归一化处理，以便使$$w^{[l+1]}$$和$$b^{[l+1]}$$的训练更有效率。在前向/后项传播算法中，用$$\tilde{z}^{(i)}$$取代$$z^{(i)}$$。通过学习参数$$\gamma^{[l]}$$和$$\beta^{[l]}$$，可以得到特定均值和方差的$$z^{[l]}$$分布。

    $$
    \begin{aligned}
    \mu &= \frac {1}{m} \sum\limits_i^m z^{(i)} \\
    \sigma^2 &= \frac {1}{m} \sum\limits_i^m (z^{(i)} - \mu)^2 \\
    z_{norm}^{(i)} &= \frac {z^{(i)} - \mu}{\sqrt{\sigma^2 + \epsilon}} \\
    \tilde{z}^{(i)} &= \gamma z_{norm}^{(i)} + \beta
    \end{aligned}
    $$

    Batch归一化解决最重要的一个问题是协变量转移(Covariate Shift)。例如，对于第三层参数$$w^{[3]}$$和$$b^{[3]}$$的训练，我们将前一层的输出$$a^{[2]}$$作为输入。当参数$$w^{[1]},b^{[1]},w^{[2]},b^{[2]}$$改变，$$a^{[2]}$$值的分布也会发生变化；这会影响到后面隐藏层参数的训练。

    Batch归一化做的，是它减少了输入值分布变化。它减弱了前层参数的作用与后层参数的作用之间的联系，使得网络每层都可以自己学习，稍稍独立于其它层，这有助于加速整个网络的学习。

参考：  
[梯度下降法小结](https://www.cnblogs.com/huangyc/p/9801261.html#_label3_3)