---
title:  “Scikit ML - 异常检测”
date:   2018-07-20 08:00:12 +0800
categories: machine learning
---

异常检测分为novelty detection和outlier detection。
- Novelty detection: 适用于训练数据没有被异常点污染，我们关心的是新观测点是否是异常点
- Outlier detection: 训练数据包含异常点


# **Novelty Detection**

## 单分类SVM

One-Class SVM训练出一个高维超球面，将数据尽可能包围起来。

# **Outlier Detection**

## Elliptic Envelope

首先复习一下协方差的数学知识。两个随机变量X，Y的协方差定义为

$$
\begin{aligned}
cov(X,Y)&=E[(X-EX)(Y-EY)] \\
&=E(XY)-EXEY
\end{aligned}
$$

相关系数定义为

$$
\begin{aligned}
\rho_{XY}&=\frac{cov(X,Y)}{\sqrt{D(X)D(Y)}} \\
&=\frac{E[(X-EX)(Y-EY)]}{\sqrt{E[(X-EX)^2]E[(Y-EY)^2]}} \\
&=\frac{E(XY)-EXEY}{\sqrt{(E(X^2)-(EX)^2)(E(Y^2)-(EY)^2)}}
\end{aligned}
$$

1. 经验协方差（Empirical Covariance）

    在观测样本数量相对于数据特征数量足够大的情况下，样本的最大似然估计(Maximum Likelihood Estimator)是协方差矩阵的无偏估计。

    假设概率分布函数（$$ X \in \mathbb{R}^{d \times 1}, \mu \in \mathbb{R}^{d \times 1}, \Sigma \in \mathbb{R}^{d \times d} $$）：

    $$
    f(x)=\frac{1}{(2\pi)^{d/2}\left | \Sigma \right|^{1/2}}\exp\left ( -\frac{1}{2}(x-\mu)^T\Sigma^{-1}(x-\mu) \right )
    $$

    建立似然函数：

    $$
    \begin{aligned}
    \mathcal{L}(\mu,\Sigma)&=\prod \limits_{i=1}^m\frac{1}{(2\pi)^{d/2}\left | \Sigma \right|^{1/2}}\exp\left ( -\frac{1}{2}(x^i-\mu)^T\Sigma^{-1}(x^i-\mu) \right ) \\
    &=\frac{1}{(2\pi)^{nd/2}}\prod \limits_{i=1}^m\frac{1}{\left | \Sigma \right|^{1/2}}\exp\left ( -\frac{1}{2}(x^i-\mu)^T\Sigma^{-1}(x^i-\mu) \right )
    \end{aligned}
    $$
  
    用样本均值$$ \bar{x}=\frac{x^1+ \cdots +x^n}{n} $$估计$$ \mu $$，于是我们的目标是最大化下面的函数：

    $$
    \mathcal{L}(\bar{x},\Sigma) \propto \frac{1}{\left | \Sigma \right |^{n/2}} \exp\left ( -\frac{1}{2}\sum_{i=1}^n(x^i-\bar{x})^T\Sigma^{-1}(x^i-\bar{x}) \right )
    $$

    省略掉后面的证明（因为有部分没看懂），结论就是：

    $$
    \Sigma=\frac{1}{n}\sum_{i=1}^n(x^i-\bar{x})(x^i-\bar{x})^T
    $$

2. 最小协方差行列式（Minimum Covariance Determinant）
    
    实际数据集中通常都有异常值的出现，经验协方差估计对数据中的异常值非常敏感，因此需要使用鲁棒的协方差估计(Robust Covariance Estimation)来估计实际的协方差。

    最小协方差行列式估计的基本思想是选取一定百分比的非异常观测值，计算其经验协方差矩阵。在找到了最小协方差行列式估计后，根据马氏距离(Mahalanobis Distance)对观测值施加权重，最终得到一个加权后的协方差估计值。

异常检测一个通常的想法是假设正常的观测值符合某种统计分布，比如高斯分布。于是我们
估计均值和协方差矩阵，然后通过马氏距离，我们可以预测某个新观测值是否是异常值。

$$
d_{\mu,\Sigma}(x^i)^2=(x^i-\hat{\mu})^T\hat{\Sigma}^{-1}(x^i-\hat{\mu})
$$

## 孤立森林（Isolation Forest）

孤立森林的想法很有意思，其依据是，在数据空间里面，分布稀疏的区域表示数据发生在此区域的概率很低，因此如果有观测点落在了这些区域里即是异常的。

随机用超平面切割数据空间，循环直到每个子空间只有一个数据点，可以得到一颗孤立树(Isolation Tree)。反复重头开始切割，得到N个孤立树，然后可以得到训练数据在孤立树的平均高度。高度越大，说明数据在密度高的区域；反之说明在密度低的区域，很大可能是异常值。

## Local Outlier Factor

传统的异常检测算法都是基于统计的（比如Elliptic Envelope），假设数据服从特定的概率分布；或是借用一些聚类算法（比如DBSCAN）。但是服从特定概率分布的假设往往是不成立的。基于密度的LOF算法，不需要对数据分布做过多要求，而且可以量化每个数据的异常程度。下面几个概念：

- K邻近距离(k-distance)：在距离数据点p最近的几个点中，第k个最近的点跟点p之间的距离成为点p的K邻近距离，记为$$ k-distance(p) $$

    ![image01]({{site.baseurl}}/image/20180720/k_distance.png)

- 可达距离(reachability distance): 给定参数k时，数据点p到数据点o的可达距离为数据点o的K邻近距离和数据点p与点o之间直线距离的最大值：$$ reach-distance_k(p,o)=\max\{k-distance(o), d(p,o)\} $$。可见，如果密度低的区域点稀疏，可达距离大。

    ![image02]({{site.baseurl}}/image/20180720/reach_distance.png)

- 局部可达密度（local reachability density): 表示点p的第K邻域内所有点到p的平均可达距离的倒数。如果一个数据点跟其他点比较疏远的话，则局部可达密度就小。

    $$
    lrd_k(p)=\frac{1}{\frac{\sum_{o \in N_k(p)}reach-dist_k(p,o)}{|N_k(p)|}}
    $$

- 局部异常因子（local outlier factor）: 衡量一个数据点的异常程度，不是看绝对局部密度，而是跟周围邻近数据点的相对密度。这样允许数据分布不均匀，密度不同的情况。

    $$
    \begin{aligned}
    LOF_k(p)&=\frac{\sum_{o \in N_k(p)}\frac{lrd(o)}{lrd(p)}}{|N_k(p)|}
    &=\frac{\sum_{o \in N_k(p)}lrd(o)}{|N_k(p)| \cdot lrd(p)}
    \end{aligned}
    $$

    如果数据点 p 的 LOF 得分在1附近，表明数据点p的局部密度跟它的邻居们差不多；如果数据点 p 的 LOF 得分小于1，表明数据点p处在一个相对密集的区域，不像是一个异常点；如果数据点 p 的 LOF 得分远大于1，表明数据点p跟其他点比较疏远，很有可能是一个异常点

    ![image03]({{site.baseurl}}/image/20180720/lof.png)

参考：  
[协方差估计Wiki](https://en.wikipedia.org/wiki/Estimation_of_covariance_matrices)  
[马氏距离](http://www.cnblogs.com/kevinGaoblog/archive/2012/06/19/2555448.html)  
[孤立森林入门篇](https://www.jianshu.com/p/5af3c66e0410)  
[CSDN: 异常点检测算法LOF](https://blog.csdn.net/wangyibo0201/article/details/51705966)  
[知乎：Isolation Forest](https://zhuanlan.zhihu.com/p/27777266)  
[知乎：Local Outlier Factor](https://zhuanlan.zhihu.com/p/28178476)