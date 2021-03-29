---
title:  “Spark ML - Factor Analysis”
mathjax: true
layout: post
date:   2018-09-03 08:00:12 +0800
categories: machine learning
---

因子分析（Factor Analysis）是另一种降维方法，目标是从大量观察变量中分析出潜在的因子变量。例如一个学生的可观察变量包括：数学，英语，语文，物理等等各科目的成绩，跟成绩相关的隐藏因子变量可能包括：记忆力，逻辑推理能力，毅力等等。

当训练集数量远远少于向量维度时（$$ m \ll n$$），高斯混合模型中协方差矩阵$$\Sigma$$是一个奇异矩阵，因此不可逆。此时因子分析是一个可行的建模方案。

1. 假设x由一个d维多元高斯分布z生成

    $$
    \begin{aligned}
    &z \sim \mathcal N(0, I), z \in \mathbb{R}^d (d < n) \\
    &x|z \sim \mathcal N(\mu + \Lambda z, \Psi)
    \end{aligned}
    $$

    其中$$\mu \in \mathbb{R}^n, \Lambda \in \mathbb{R}^{n \times d}, \Psi \in \mathbb{R}^{n \times n}$$，且$$\Psi$$是一个对角阵。等价的写法为：
    
    $$
    \begin{aligned}
    &x = \mu + \Lambda z + \epsilon \\
    &\epsilon \sim \mathcal N(0, \Psi)
    \end{aligned}
    $$

2. x的边缘分布

    $$(x,z)$$的联合分布为：

    $$
    \left [
        \begin{matrix}
        z \\
        x
        \end{matrix}
    \right ]
    \sim \mathcal N(\mu_{zx}, \Sigma)
    $$    

    因为$$E[z]=0, E[\epsilon]=0$$，可得$$E[x]=E[\mu + \Lambda z + \epsilon]=\mu$$，所以：

    $$
    \mu_{zx} = \left [
        \begin{matrix}
        0 \\
        \mu
        \end{matrix}
    \right ]
    $$

    继续计算协方差矩阵$$\Sigma$$：

    $$
    \Sigma = \left [
        \begin{matrix}
        \Sigma_{zz} & \Sigma_{zx} \\
        \Sigma_{xz} & \Sigma_{xx}
        \end{matrix}
    \right ]
    $$

    其中：
    
    $$
    \begin{aligned}
    \Sigma_{zz} &= Cov(z) = I \\
    \Sigma_{zx} &= E[(z-E[z])(x-E[x])^T] \\
    &= E[z(\mu + \Lambda z + \epsilon - \mu)^T] \\
    &= E[zz^T] \Lambda^T + E[z \epsilon^T] \\
    &= \Lambda^T \\
    \Sigma_{xz} &= E[(x-E[x])(z-E[z])^T] \\
    &= E[(\mu + \Lambda z + \epsilon - \mu)z^T] \\
    &= \Lambda E[zz^T] + E[\epsilon z^T] \\
    &= \Lambda \\
    \Sigma_{xx} &= E[(x-E[x])(z-E[x])^T] \\
    &= E[(\mu + \Lambda z + \epsilon - \mu)(\mu + \Lambda z + \epsilon - \mu)^T] \\
    &= E[\Lambda zz^T \Lambda^T + \epsilon z^T \Lambda^T + \Lambda z \epsilon^T + \epsilon \epsilon^T] \\
    &= \Lambda E[zz^T] \Lambda^T + E[\epsilon \epsilon^T] \\
    &= \Lambda \Lambda^T + \Psi
    \end{aligned}
    $$

    于是：

    $$
    \left [
        \begin{matrix}
        z \\
        x
        \end{matrix}
    \right ]
    \sim \mathcal N 
    \left (
        \left [
            \begin{matrix}
            0 \\
            \mu
            \end{matrix}
        \right ], 
        \left [
            \begin{matrix}
            I & \Lambda^T \\
            \Lambda & \Lambda \Lambda^T + \Psi
            \end{matrix}
        \right ]
    \right )
    $$
    
    x的边缘分布为:

    $$
    x \sim \mathcal N(\mu, \Lambda \Lambda^T + \Psi)
    $$

3. EM算法计算因子分析模型

    优化目标是最大似然对数函数：

    $$
    \begin{aligned}
    \mathcal{l}(\mu,\Lambda, \Psi) &= log \prod_{i=1}^m P(x^{(i)}) \\
    &= log \prod_{i=1}^m \frac{1}{(2\pi)^{n/2} |\Lambda \Lambda^T + \Psi|} \exp \left ( -\frac{1}{2}(x^{(i)}-\mu)^T(\Lambda \Lambda^T + \Psi)^{-1}(x^{(i)}-\mu) \right ) 
    \end{aligned}
    $$

    一般的EM算法表示为, E-step for each i:

    $$
    \begin{aligned}
    w_j^{(i)} &:= Q_i(z^{(i)} = j) \\ 
    &= p(z^{(i)} = j |x^{(i)}; \theta)
    \end{aligned}
    $$

    M-step:

    $$
    \theta := \arg max_\theta \sum_i \sum_{z^{(i)}} Q_i(z^{(i)}) log \frac{p(x^{(i)}, z^{(i)}; \theta)}{Q_i(z^{(i)})}
    $$

    在因子分析的特例中，E-step：

    $$
    \begin{aligned}
    \mu_{z^{(i)}|x^{(i)}} &= \mu_z + \Sigma_{zx} \Sigma_{xx}^{-1} (x - \mu_x) \\
    &= \Lambda^T (\Lambda \Lambda^T + \Psi)^{-1} (x^{(i)} - \mu) \\
    \Sigma_{z^{(i)}|x^{(i)}} &=  \Sigma_{zz} - \Sigma_{zx} \Sigma_{xx}^{-1} \Sigma_{xz} \\
    &= I - \Lambda^T (\Lambda \Lambda^T + \Psi)^{-1} \Lambda \\
    Q_i(z^{(i)}) &= p(z^{(i)}|x^{(i)}; \mu, \Lambda, \Psi) \\
    &= \frac{1}{ (2\pi)^{k/2} |\Sigma_{z^{(i)}|x^{(i)}}| }  \exp \left ( 
        -\frac{1}{2}(z^{(i)}-\mu_{z^{(i)}|x^{(i)}})^T \Sigma_{z^{(i)}|x^{(i)}}^{-1} (z^{(i)}-\mu_{z^{(i)}|x^{(i)}}) \right )
    \end{aligned} \\
    $$

    M-step：

    $$
    \arg \max_{\mu, \Lambda, \Psi} \sum_{i=1}^m \int_{z^{(i)}} Q_i(z^{(i)}) log 
    \frac{p(x^{(i)}, z^{(i)}; \mu, \Lambda, \Psi)}{Q_i(z^{(i)})} dz^{(i)}
    $$

    后面的求导和结果不写了，基本已经到了我数学能力的极限了。

参考：  
[csdn - 因子分析与EM算法](https://blog.csdn.net/danerer/article/details/80295989)  
[aliyun - Factor Analysis](https://yq.aliyun.com/articles/85946)