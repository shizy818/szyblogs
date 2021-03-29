---
title:  “Spark ML - 广义线性模型”
mathjax: true
layout: post
date:   2018-08-08 08:00:12 +0800
categories: machine learning
---

线性回归中，假设函数为：

$$
h_\theta(x) = \theta^Tx
$$

逻辑回归中，假设函数为：

$$
h_\theta(x) = \frac {1}{1 + e^{-\theta^Tx}}
$$

实际上，在分类和回归问题中，我们通过关于x的模型来预测y。这样的问题可以用广义线性模型解决（Generalized linear model)来解决。


# **指数分布簇**

指数分布簇（Exponential Family）的概率分布如下：

$$
p(y;\eta) = b(y) \centerdot exp(\eta^TT(y)-a(\eta))
$$


$$ \eta $$称为自然参数（natural parameter），下面看看一些特例：

## 伯努利分布

$$
\begin{aligned}
p(y;\phi) &= \phi^y(1 - \phi)^{1-y} \\
&=exp(y\log \phi + (1-y)\log (1-\phi)) \\
&=exp \left ( \left (\log \left (\frac{\phi}{1-\phi} \right) \right )y + \log(1-\phi) \right )
\end{aligned}
$$

对应参数：

$$
\begin{aligned}
&\eta = \log \left ( \frac{\phi}{1-\phi} \right ) \\
&T(y) = y \\
&a(\eta) = -\log(1-\phi) = log(1+e^\eta) \\
&b(y) = 1
\end{aligned}
$$

## 高斯分布

$$
\begin{aligned}
p(y;\mu) &= \frac{1}{\sqrt{2\pi}} exp \left ( -\frac{1}{2} (y - \mu)^2 \right ) \\
&=\frac{1}{\sqrt{2\pi}} exp \left ( -\frac{1}{2}y^2 \right ) \centerdot exp \left ( \mu y - \frac{1}{2} \mu^2 \right )
\end{aligned}
$$

对应参数：

$$
\begin{aligned}
&\eta = \mu \\
&T(y) = y \\
&a(\eta) = \frac{\mu^2}{2} = \frac{\eta^2}{2} \\
&b(y) = \frac{1}{\sqrt{2\pi}} exp \left ( -\frac{y^2}{2} \right )
\end{aligned}
$$

## 多项式分布

多项式分布$$ y \in {1,2,\cdots,k} $$, 其概率分布为$$ P(y = i) = \phi_i $$。因为$$ \sum_{i=1}^k \phi_i = 1 $$，所以可以只保留k-1个参数。

定义$$ T(y) \in \mathbb{R}^{k-1} $$是一个k-1维向量：

$$
T(1) = \left [
        \begin{matrix}
            1 \\
            0 \\
            \vdots \\
            0
        \end{matrix}
    \right ],
T(2) = \left [
        \begin{matrix}
            0 \\
            1 \\
            \vdots \\
            0
        \end{matrix}
    \right ],
\cdots,
T(k-1)= \left [
        \begin{matrix}
            0 \\
            0 \\
            \vdots \\
            1
        \end{matrix}
    \right ],
T(k)= \left [
        \begin{matrix}
            0 \\
            0 \\
            \vdots \\
            0
        \end{matrix}
    \right ]
$$

定义指示函数:

$$
    \mathcal{I}\{True\} = 1, \mathcal{I}\{False\} = 0,
$$

于是T(y)中某个元素可以表示为：

$$
T(y)_i = \mathcal{I}\{y=i\}
$$

多项式指数分布簇推导：

$$
\begin{aligned}
p(y;\phi) &= \phi_1^{\mathcal{I}\{y=1\}} \phi_2^{\mathcal{I}\{y=2\}} \cdots \phi_k^{\mathcal{I}\{y=k\}} \\
&= \phi_1^{T(y)_1} \phi_2^{T(y)_2} \cdots \phi_{k-1}^{T(y)_{k-1}} \phi_k^{1 - \sum_{j=1}^{k-1}T(y)_j} \\
&= e^{(\log \phi_1) T(y)_1 + (\log \phi_2) T(y)_2 + \cdots + (\log \phi_k) (1 - \sum_{j=1}^{k-1}T(y)_j) } \\
&=e^{ (\log \frac{\phi_1}{\phi_k}) T(y)_1 + (\log \frac{\phi_2}{\phi_k}) T(y)_2 + \cdots + (\log \frac{\phi_{k-1}}{\phi_k}) T(y)_{k-1} + \log \phi_k }
\end{aligned}
$$

可以得到：

$$
\begin{aligned}
&\eta = \left [
        \begin{matrix}
            \log (\phi_1/\phi_k)  \\
            \log (\phi_2/\phi_k) \\
            \vdots \\
            \log (\phi_{k-1}/\phi_k)
        \end{matrix}
    \right ] \\
&b(y) = 1 \\
&a(\eta) = -\log \phi_k
\end{aligned}
$$

反推可得$$ \phi_i $$：

$$
\phi_i = \frac{e^{\eta_i}}{1 + \sum_{j=1}^{k-1} e^{\eta_i}}
$$

许多其他分部也属于指数分布族，例如泊松分布（Poisson）、伽马分布（Gamma）、指数分布（Exponential）、β分布、Dirichlet分布、Wishart分布。

# **构建广义线性模型**

对m个样本数据建模的通用步骤是首先建立最大似然函数对数，然后通过牛顿法或者梯度下降法求得参数$$ \theta $$，最后通过假设函数$$ h_\theta $$进行预测。

$$
\begin{aligned}
&L(\theta) = \prod_{i=1}^m p(y^{(i)}|x^{(i)};\theta) \\
&l(\theta) = \log L(\theta) = \sum_{i=1}^m \log p(y^{(i)}|x^{(i)};\theta)
\end{aligned}
$$


广义线性模型基于三个假设：
- $$ y|x;\theta \sim ExponentialFamily(\eta) $$
，即给定输入$$ x $$ 和参数$$ \theta $$, $$ y $$满足指数簇分布
- 给定$$ x $$，预测$$ T(y) $$期望值函数：
$$ h(x) = E[T(y)|x] $$。大多数情况下$$ T(y) = y $$。
- 参数 $$ \eta $$ 和特征输入$$ x $$线性相关，即$$ \eta = \theta^Tx $$

## 高斯分布构建最小二乘模型

y是一个连续值，假设
$$ y|x \sim \mathcal N(\mu, \sigma) $$，则
$$ E[y|x; \theta] = \mu $$。目标$$ y $$期望推导如下：

$$
\begin{aligned}
h_\theta(x) &= E[y|x;\theta] \\
&= \mu \\
&= \eta \\
&= \theta^Tx
\end{aligned}
$$

最大似然函数对数：

$$
\begin{aligned}
l(\theta) &= \log \prod_{i=1}^m \frac{1}{\sqrt{2\pi}\sigma} e^{ -\frac{(y^{(i)} - \mu)^2}{2\sigma^2} } \\
&=-m \log \frac{1}{\sqrt{2\pi}\sigma} \sum_{i=1}^m \frac{(y^{(i)} - \theta^Tx^{(i)})^2}{2\sigma^2}
\end{aligned}
$$

## 伯努利分布构建逻辑回归模型

y是{0，1}离散值，假设
$$ y|x \sim \mathcal Bernoulli(\phi) $$，则
$$ E[y|x; \theta] = \phi $$。假设函数$$ y $$期望推导如下：

$$
\begin{aligned}
h_\theta(x) &= E[y|x;\theta] \\
&= \phi \\
&= \frac{1}{1 + e^{-\eta}} \\
&= \frac{1}{1 + e^{-\theta^Tx}}
\end{aligned}
$$

最大似然函数对数：

$$
\begin{aligned}
l(\theta) &= \log \prod_{i=1}^m \phi^{y^{(i)}} (1 - \phi)^{1 - y^{(i)}} \\
&=\sum_{i=1}^m ( y^{(i)} \log \phi + (1 - y^{(i)})\log (1 - \phi) )
\end{aligned}
$$

## 多项式分布构建softmax模型

y是1到k的离散值，假设
$$ y|x \sim \mathcal{PN} $$，则：

$$
\begin{aligned}
h_\theta(x) &= E[T(y)|x;\theta] \\
&= E\left [
        \left.
            \begin{matrix}
                \mathcal{I}\{y=1\}  \\
                \mathcal{I}\{y=2\} \\
                \vdots \\
                \mathcal{I}\{y=k-1\}
            \end{matrix}
        \right | 
        x; \theta
    \right ] \\
&= \left [
        \begin{matrix}
            \phi_1  \\
            \phi_2 \\
            \vdots \\
            \phi_{k-1}
        \end{matrix}
    \right ] \\
&= \left [
        \begin{matrix}
            e^{\theta_1^Tx}/(1 + \sum\limits_{j=1}^{k-1} e^{\theta_j^Tx})  \\
            e^{\theta_2^Tx}/(1 + \sum\limits_{j=1}^{k-1} e^{\theta_j^Tx}) \\
            \vdots \\
            e^{\theta_{k-1}^Tx}/(1 + \sum\limits_{j=1}^{k-1} e^{\theta_j^Tx})
        \end{matrix}
    \right ]
\end{aligned}
$$

最大似然函数对数：

$$
\begin{aligned}
l(\theta) &= \log \prod_{i=1}^m \prod_{j=1}^k \phi_j^{ \mathcal{I}\{y^{(i)}=j\} } \\
&=\sum_{i=1}^m \sum_{j=1}^k \mathcal{I}\{y^{(i)}=j\} \log \phi_j
\end{aligned}
$$

# **Spark ML 2.3.1**

在Spark中定义了GeneralizedLinearModel和GeneralizedLinearAlgorithm，在前文[支持向量机](/szyblogs/machine/learning/2018/07/28/ml-svm/)中也有提及，核心是run方法。

参考：  
[CSDN：广义线性模型](https://blog.csdn.net/v1_vivian/article/details/52055760)  
[知乎：牛顿法，GLM以及softmax回归算法](https://zhuanlan.zhihu.com/p/37296647)