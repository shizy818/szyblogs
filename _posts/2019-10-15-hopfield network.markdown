---
title:  “Hopfield神经网络”
date:   2019-10-15 08:00:12 +0800
categories: deeplearning
---

Hopfield网络是一种循环神经网络(recurrent)，从输出到输入有反馈连接。在输入的不断激励下，网络状态不断发生变化；若是一个能收敛的网络，反馈后网络产生的变化越来越小，一直达到稳定状态，则输出稳定。


离散型Hopfield神经网络DHNN(Discrete Hopfield Nerual Network)采用的神经元是二值神经元，输出离散值1和-1分别表示神经元处于激活和关闭状态。定义：

$$s_j$$: 表示神经元j的状态  
$$w_{ij}$$: 突触连接的强度，即神经元之间的权值   
$$\theta_i$$: 神经元的直接输入

![image01]({{site.baseurl}}/image/20191015/hopfield.png)

于是每个神经元的总输入是$$\sum \limits_j w_{ij}s_j + \theta_i$$，系统总能量定义为$$E = - \sum \limits_{i}s_i(\sum \limits_j w_{ij}s_j + \theta_i)$$。

# 连接对称的Hopfield网络

非线性单元的循环神经网络的稳定性：
- 收敛到稳定状态
- 震荡：有限状态之间循环往复
- 混沌：无限多个状态之间变化，但轨迹并不发散到无穷远

John Hopfield发现，如果网络的连接是对称的($$w_{ij} = w_{ji}$$)且神经元无自反馈($$w_{ii}=0$$)，则网络有一个全局能量函数，并且正确的网络更新规则使这个能量函数最终收敛到一个最小值。更新规则为：

$$
s_i(t+1) = \left \{
    \begin{aligned}
    & 1  \qquad if \quad \sum_j w_{ij}s_j(t) + \theta_i > 0 \\
    & -1 \qquad otherwise 
    \end{aligned}
\right.
$$

更新规则的含义即如果一个神经元的总输入大于0，那么神经元置为激活1，从上面总能量定义可知$$E$$变小；而当一个神经元总输入小于0时，神经元状态置为关闭0。

![image02]({{site.baseurl}}/image/20191015/symmetric_hopfield.png)

# Hopfield网络状态变化例子

下面用一个具体的例子，说明Hopfield网络的神经元从某个随机状态开始，以序列化的方式，每次更新一个神经元，最后达到一个能量最小值的状态(此处可能是局部最小）。

1. 初始状态，只有三个神经元被激活，总能量为$$-5 + (-4) + 2 = -7$$

2. 随便挑选右上角橘黄色的神经元更新。总输入为$$-4 \times 1 + 3 \times 1 + 3 \times -1 = -4$$，所以将其关闭，状态-1

    ![image03]({{site.baseurl}}/image/20191015/state1.png)

3. 计算左下角橘黄色神经元总输入$$3 \times 1 + -1 \times -1 = 4$$，保持激活状态1

    ![image04]({{site.baseurl}}/image/20191015/state2.png)

4. 计算中间橘黄色神经元总输入$$-1 \times 1 + 2 \times 1 + 3 \times -1 + -1 \times 1 = -3$$，保持关闭状态-1

    ![image05]({{site.baseurl}}/image/20191015/state3.png)

5. 计算右下角橘黄色神经元总输入$$3 \times -1 + -1 \times -1 = -2$$，改变状态置为关闭-1。此时总能量为$$-5 + (-4) = -11$$

    ![image06]({{site.baseurl}}/image/20191015/state4.png)

5. 计算左上角橘黄色神经元总输入$$3 \times 1 + -4 \times -1 + 2 \times -1= 5$$，保持激活状态1。

    ![image07]({{site.baseurl}}/image/20191015/state5.png)

更新完5个神经元，网络总能量达到一个最小值，网络处于稳定状态。除了序列化的顺序更新每个节点的状态，也可以通过同步的方式来更新收敛（连接矩阵$$W$$非负定对称）。网络最终达到的稳定状态被称为吸引子(attractor)。

![image08]({{site.baseurl}}/image/20191015/state6.png)

# Hopfield网络的应用

Hopfield网络可以用于联想记忆，网络存储若干预先设定的稳定状态，即记忆的东西。若Hopfield网络的拓扑结构和权值矩阵都一定，则稳定态和初始状态有关。若将稳定态视作一个记忆样本，那么初始态朝稳定态收敛的过程就是寻找记忆样本的过程。

![image09]({{site.baseurl}}/image/20191015/application.png)

网络的记忆容量可以理解为联想记忆过程中，网络能够正确记忆的最大样本数，其实就是这个系统的吸引子数目。Hopfield的网络唤醒率为0.138，也就是如果网络有1000个节点，大约能从存储的记忆样本中正确回忆起138个向量样本。

参考：  
[Hopfield Nets and Boltzmann Machine](https://wenku.baidu.com/view/3d16db8364ce0508763231126edb6f1aff0071f5.html)
