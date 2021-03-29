---
title:  “Convolutional Neural Networks”
mathjax: true
layout: post
date:   2019-08-15 08:00:12 +0800
categories: deeplearning.ai
---

卷积神经网络(Convolutional Neural Networks)是deeplearning.ai的第四门课程，从这儿开始我觉得基本上算是目前深度学习的主流核心部分了。卷积神经网络最重要的应用是在计算机视觉领域，比如人脸识别，目标检测等等。


# 卷积运算

在下面的例子中，用一个$$6 \times 6 \times 1$$的灰度图像来说明卷积如何进行边缘检测。图像左边一半是10，右边一半是0，所以左边部分看起来是亮色，右边是深色。构造一个$$3 \times 3$$的过滤器(或者称为核)：

$$
\left [
    \begin{matrix}
        1 & 0 & -1 \\
        1 & 0 & -1 \\
        1 & 0 & -1
    \end{matrix}
\right ]
$$

当用这个$$3 \times 3$$的过滤器进行卷积运算的时候，其模式识别为左边是明亮像素，右边是深色像素，所以卷积运算提供了一个方便的方法发现图像中的垂直边缘。

![image01]({{site.baseurl}}/image/20190815/convolution.png)

其他的一些过滤器，主要是增加了中间一行元素的权重，使得结果的鲁棒性高一些。

- Sobel过滤器

    $$
    \left [
        \begin{matrix}
            1 & 0 & -1 \\
            2 & 0 & -2 \\
            1 & 0 & -1
        \end{matrix}
    \right ]
    $$

- Scharr过滤器

    $$
    \left [
        \begin{matrix}
            3 & 0 & -3 \\
            10 & 0 & -10 \\
            3 & 0 & -3
        \end{matrix}
    \right ]
    $$

1. 填充(Padding)

    对于一个$$n \times n$$的图像，用$$f \times f$$的过滤器做卷积，不做填充的输出维度是$$(n-f+1) \times (n-f+1)$$。这样的卷积叫做Valid卷积。每做一次卷积操作，图像就会缩小一次；另外边缘区域的像素点在计算过程中采用较少，意味着丢掉了图像边缘位置的一些信息。

    常用的填充方法叫做Same卷积，填充后输出的大小和输入大小是一样的。用$$p$$表示边缘填充像素个数，简单求解可得$$p=(f-1)/2$$。这也是为什么在计算机视觉中，卷积过滤器维度$$f$$通常是奇数，保证了Same卷积是对称的。

2. 步长(Striding)

    用$$f \times f$$的过滤器卷积一张$$n \times n$$的图像，padding为$$p$$，步长为$$s$$，则输出维度是$$ \lfloor \frac{n+2p-f}{s} + 1 \rfloor \times \lfloor \frac {n+2p-f}{s} + 1 \rfloor$$。其中如果$$\frac{n+2p-f}{s}$$不是一个整数，向下取整。

Andrew Ng特别解释了我一直以来的一个疑惑：**数学上卷积的定义应该包含翻转的动作，比如函数$$f(t)$$和$$g(t)$$的卷积$$f(t)*g(t) = \int_{-\infty}^{\infty} f(\tau)g(t-\tau)d\tau$$。按照机器学习的惯例，我们通常不进行翻转操作。从技术上说，这个操作可能叫做互相关更好。但在大部分的深度学习文献中都把它叫做卷积运算。**

## 三维卷积

对于彩色图片，输入$$height \times width \times channel$$的第三个维度表示RGB颜色通道。对应卷积过滤器的维度为$$f \times f \times channel$$。

# 单层卷积网络

对于卷积神经网络的一层，维度总结如下：

输入： $$n_H^{[l-1]} \times n_W^{[l-1]} \times n_c^{[l-1]}$$  
过滤器($$n_c^{[l]}$$个)： $$f^{[l]} \times f^{[l]} \times n_c^{[l-1]}$$  
激活值$$a^{[l]}$$：$$n_H^{[l]} \times n_W^{[l]} \times n_c^{[l]}$$  
权重参数$$W$$：$$f^{[l]} \times f^{[l]} \times n_c^{[l-1]} \times n_c^{[l]}$$  
偏差参数$$b$$：$$1 \times 1 \times 1 \times n_c^{[l]}$$  
输出： $$n_H^{[l]} \times n_W^{[l]} \times n_c^{[l]}$$

其中：

$$
\begin{aligned}
n_H^{[l]} &= \lfloor \frac{n_H^{[l-1]}+2p^{[l]}-f^{[l]}}{s^{[l]}} + 1 \rfloor \\
n_W^{[l]} &= \lfloor \frac{n_W^{[l-1]}+2p^{[l]}-f^{[l]}}{s^{[l]}} + 1 \rfloor \\
\end{aligned}
$$

一个典型的卷积神经网络有三层，一个是卷积层(Conv)，另外两层分别是池化层(Pool)和全连接层(FC)。池化层用来缩减模型大小，提高计算速度，比如最大池化(Max Pooling)和平均池化(Average Pooling)。目前来说，最大池化比平均池化更常用。全连接层有一个权重矩阵$$W$$和偏差参数$$b$$，一般输出维度比输入维度小很多。神经网络的常见模式是将几个全连接层放在最后一个Softmax层之前。

# 卷积网络实例

- LeNet-5

    ![image02]({{site.baseurl}}/image/20190815/lenet-5.png)

- AlexNet

    AlexNet和LeNet很相似，但是AlexNet网络要大得多。LeNet-5大约有6万个参数，AlexNet包含约6000万个参数。

    ![image03]({{site.baseurl}}/image/20190815/alexnet.png)

- VGG-16

    VGG-16网络中包含16个卷积层和全连接层，总共约1.38亿个参数。VGG结构很简单，都是卷积层后面跟着可以压缩图像大小的池化层。

    ![image04]({{site.baseurl}}/image/20190815/vgg-16.png)

- ResNet

    非常深的神经网络很难训练，因为存在梯度消失和梯度爆炸的问题。ResNet从某一层获取激活，然后反馈给另外一层。其基本单元是一个残差块(Residual Block)。

    ![image05]({{site.baseurl}}/image/20190815/residual_block.png)

    用公式表达为:

    $$
    \begin{aligned}
    z^{[l+1]} &= W^{[l+1]}a^{[l]} + b^{[l+1]}  \\
    a^{[l+1]} &= ReLU(z^{[l+1]}) \\
    z^{[l+2]} &= W^{[l+2]}a^{[l+1]} + b^{[l+2]} \\
    a^{[l+2]} &= ReLU(z^{[l+2]} + a^{[l]})
    \end{aligned}
    $$

    插入$$a^{[l]}$$的时机是在线性激活函数之后，ReLU激活函数之前。构建一个ResNet网络就是通过将这很多的残差块堆积在一起，形成一个很深的神经网络。下图显示了其中的一部分：

    ![image06]({{site.baseurl}}/image/20190815/resnet.png)

- Inception

    在介绍Inception网络之前，先引入$$1 \times 1$$卷积的概念。下图可以理解为对输入36个不同的位置都应用一个全连接层，全连接层的作用是输入32个数字，输出结果是$$6 \times 6 \times 过滤器数量$$。简单的说，$$1 \times 1$$卷积可以用来压缩或保持输入层中的通道数量，甚至是增加通道数量。

    ![image07]({{site.baseurl}}/image/20190815/11conv.png)

    Inception网络的一个想法就是由网络自动确定卷积层中的过滤器类型，或者确定是否需要创建卷积层或池化层。下图是一个Inception模块，输入维度是$$28 \times 28 \times 192$$，输出为$$28 \times 28 \times 256$$。$$3 \times 3$$卷积和$$5 \times 5$$卷积之前的$$1 \times 1$$卷积，目的在于降低计算成本，分别将通道数量从192压缩到96和16。

    ![image08]({{site.baseurl}}/image/20190815/inception_module.png)

    将多个重复的Inception模块组合到一起，就是一个Inception网络。当然Inception还有一些小细节，比如在隐藏层和中间层即输出预测，防止网络发生过拟合。

    ![image09]({{site.baseurl}}/image/20190815/inception.png)