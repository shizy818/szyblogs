---
title:  “Spark ML - 神经网络”
date:   2018-08-15 08:00:12 +0800
categories: machine learning
---

作为Deep Learning的基础，了解一下神经网络的反向传播算法。

# **雅可比矩阵 & 海瑟矩阵**

再复习一下雅可比矩阵(Jacobian matrix)和海瑟矩阵(Hessian matrix)。一般情况，参数空间都不是一维的。对目标函数（标量或者0阶张量）求梯度得到一个向量（1阶张量）。


$$
\nabla_\theta g = \left ( \frac{\partial g}{\partial \theta_1}, \frac{\partial g}{\partial \theta_2}, \cdots, \frac{\partial g}{\partial \theta_n}   \right )
$$

如果原来的g是向量$$ g = (g_1, g_2, \cdots, g_m) $$，求导可得雅可比矩阵：

$$
\frac{\partial g}{\partial \theta} = \left [
        \begin{matrix}
            \frac {\partial g_1}{\partial \theta_1} & \frac {\partial g_1}{\partial \theta_2} & \cdots & \frac {\partial g_1}{\partial \theta_n} \\
            \frac {\partial g_2}{\partial \theta_1} & \frac {\partial g_2}{\partial \theta_2} & \cdots & \frac {\partial g_2}{\partial \theta_n} \\
            \vdots & \vdots & \ddots & \vdots \\
            \frac {\partial g_m}{\partial \theta_1} & \frac {\partial g_m}{\partial \theta_2} & \cdots & \frac {\partial g_m}{\partial \theta_n}
        \end{matrix}
    \right ]
$$

如果g是某个函数f的梯度$$ g = \nabla_\theta f = \left ( \frac{\partial f}{\partial x_1}, \frac{\partial f}{\partial x_2}, \cdots, \frac{\partial f}{\partial x_n} \right ) $$，它的雅可比矩阵就是原函数f的海瑟矩阵：

$$
H(f) = \left [
        \begin{matrix}
            \frac {\partial^2 f}{\partial x_1^2} & \frac {\partial^2 f}{\partial x_1 \partial x_2} & \cdots & \frac {\partial^2 f}{\partial x_1 \partial x_n} \\
            \frac {\partial^2 f}{\partial x_2 \partial x_1} & \frac {\partial^2 f}{\partial x_2^2} & \cdots & \frac {\partial^2 f}{\partial x_2 \partial x_n} \\
            \vdots & \vdots & \ddots & \vdots \\
            \frac {\partial^2 f}{\partial x_n \partial x_1} & \frac {\partial^2 f}{\partial x_n \partial x_2} & \cdots & \frac {\partial^2 f}{\partial x_n^2}
        \end{matrix}
    \right ]
$$

# **反向传播算法**

神经网络的基本结构
- 前向预测（Forward）：给定一个输入数据，数据一层层通过$$ f^i $$传递到下一层，最终计算出一个预测结果（Score）
- 反向修正（Backward）：通过损失值计算反馈信号，一层层通过$$ f^i $$的雅可比矩阵传回上一层，最终正确的反映为参数的修正量

整体结构如下图：

![image01]({{site.baseurl}}/image/20180815/nn_struct.png)

用$$ h_i $$记第i层的神经元个数，$$ q $$记为总的层数。最后一层的输出Score记为$$ S = X^q = (s_1, s_2, \cdots, s_{h_q}) $$，损失（Loss）记为$$ l = L(S, y) $$

前向传播信号流，$$ X^i = f^i(X^{i-1}; W^i) $$：

![image02]({{site.baseurl}}/image/20180815/forward.png)

反向传播信号流：

$$
\left \{
    \begin{aligned}
    & \delta X^q = (\frac{\partial l}{\partial s_1}, \frac{\partial l}{\partial s_2}, \cdots, \frac{\partial l}{\partial s_{h_q}}) \\
    & \delta X^i = \delta X^{i+1} \centerdot \frac{\partial X^{i+1}}{\partial X^i} = \delta X^{i+1} \centerdot \left[ \frac{\partial f_j^{i+1}}{\partial X_k^i} \right ]_{jk}  \\
    & \frac{\partial l}{\partial W^i} = \frac{\partial l}{\partial X^i} \centerdot \frac{\partial X^i}{\partial W^i} = \delta X^i \centerdot \left[ \frac{\partial f_j^i}{\partial W_k^i} \right ]_{jk}
    \end{aligned}
\right.
$$

其中$$ \left[ \frac{\partial f_j^{i+1}}{\partial X_k^i} \right ]_{jk} $$是$$ f^{i+1} $$对$$ X^i $$的雅可比矩阵;$$ \left[ \frac{\partial f_j^i}{\partial W_k^i} \right ]_{jk} $$是$$ f^i $$对$$ W^i $$的雅可比矩阵。$$ \frac{\partial l}{\partial W^i} $$得到了参数$$ W^i $$的修正量：

$$
W^i := W^i - \lambda \centerdot \nabla_{W^i}l = W^i - \lambda \centerdot \delta X^i \centerdot \left[ \frac{\partial f_j^i}{\partial W_k^i} \right ]_{jk}
$$

![image03]({{site.baseurl}}/image/20180815/backward.png)

# **具体$$ f^i $$函数**

1. Affine Transform（仿射变换）

    $$ W $$为$$ h_i \times h_{i-1} $$矩阵，$$ X $$为$$ \mathbb R^{h_{i-1}} $$向量，$$ b $$为$$ \mathbb R^{h_i} $$向量，仿射变换定义$$ \mathbb R^{h_{i-1}} \to \mathbb R^{h_i} $$映射：

    $$
    Y = W \centerdot X + b
    $$

    对应于之前的定义

    $$
    f_k = Y_k = W(k,:) \centerdot X + b_k \ (k = 1, 2, \cdots, h_i)
    $$

    $$ Y $$对$$ X $$求导：

    $$
    \frac{\partial Y}{\partial X} = J_X(Y) = W
    $$

    $$ Y $$对$$ W $$求导是一个三阶张量：

    $$
    \begin{aligned}
    &\frac{\partial Y_k}{\partial W(u,v)} = \delta_{ku} \centerdot X_v \\
    &\frac{\partial Y_k}{\partial b(u)} = \delta_{ku} \\
    &\delta_{mn} = \left \{
            \begin{aligned}
            &1 &, \ m=n \\
            &0 &, \ m \neq n
            \end{aligned}
        \right.
    \end{aligned}
    $$

    反向传播时，Affine Layer一方面负责将$$ \delta X $$向更低的层传递:

    $$
    \delta X^{i-1} = \delta X^i \centerdot W^i
    $$

    另一方面计算梯度$$ dW $$更新参数$$ W$$:

    $$
    \begin{aligned}
    &   \begin{aligned}
        \frac{\partial l}{\partial W^i(u,v)} &= \sum_{k} \frac{\partial l}{\partial X_k^i} \frac{\partial X_k^i}{\partial W^i(u,v)} \\
        &= \sum_{k} \delta X_k^i \centerdot (\delta_{k,u} \centerdot X_v^{i-1}) \\
        &= \delta X_u^i \centerdot X_v^{i-1}
        \end{aligned} \\
    &dW^i = (\delta X^i)^T \centerdot X^{i-1}
    \end{aligned}
    $$

    上式的直观意义就是反馈信号$$ \delta X^i $$按照当前层输入的激活度$$ X^{i-1} $$的比例传到该层的参数$$ W^i $$上。计算梯度$$ db $$更新参数$$ b $$:

    $$
    \begin{aligned}
    &   \begin{aligned}
        \frac{\partial l}{\partial b_k^i} &= \sum_{u} \frac{\partial l}{\partial X_u^i} \frac{\partial X_u^i}{\partial b_k^i} \\
        &= \sum_{u} \delta X_u^i \centerdot \delta_{k,u} \\
        &= \delta X_k^i
        \end{aligned} \\
    & db^i = (\delta X^i)^T
    \end{aligned}
    $$

    其中：

    $$
    \begin{aligned}
    &\delta X^i = [\delta X_1^i, \cdots, \delta X_{h_i}^i] \\
    &X^{i-1} = [X_1^{i-1}, \cdots, X_{h_{i-1}}^i]
    \end{aligned}
    $$


2. ReLU

    对每个分量做截断：小于0令它为0，正部不变。

    $$
    Y_k = (X_k)_+ = max(0, X_k), \ k=1, \cdots, h_i
    $$

    ![image04]({{site.baseurl}}/image/20180815/relu.png)

    $$ Y $$对$$ X $$求导：

    $$
    \frac{\partial Y_k}{\partial X_v} = \delta_{kv} \centerdot \mathcal{I}\{X_v > 0\}
    $$

    反向传播时：

    $$
    \begin{aligned}
    \delta X_k^i &= \sum_v \delta_{k,v} \centerdot \mathcal{I}\{X_v^i > 0\} \centerdot \delta X_v^{i+1} \\
    &= \mathcal{I}\{X_k^i > 0\} \centerdot \delta X_k^{i+1}
    \end{aligned}
    $$

    上式的直观意义就是当前的i层的第k个神经元是激活的$$ X_k^i > 0 $$，那么就让上层的反馈信号$$ \delta X_k^{i+1} $$通过，否则反馈信号截断与此。
