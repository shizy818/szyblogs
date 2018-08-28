---
title:  “牛顿迭代法”
date:   2018-08-06 08:00:12 +0800
categories: machine learning
---

# **牛顿法**

牛顿法可以近似求得函数$$ f(x) = 0 $$的解，如图：

![image01]({{site.baseurl}}/image/20180806/newton.jpg)

迭代公式：

$$
x_{n+1} := x_n - \frac{f(x_n)}{f^{'}(x_n)}
$$

# **牛顿法用于优化问题**

机器学习中很多都是建立模型，然后求解最大似然估计，于是变成$$ l^{'}(\theta)  =0 $$的问题。当维度是1时，迭代公式变成：

$$
\theta_j+1 := \theta_j - \frac{l^{'}(\theta_j)}{l^{''}(\theta_j)}
$$

如果$$ \theta $$是一个向量，迭代公式：

$$
\theta_j+1 := \theta_j - H^{-1} \nabla_\theta l(\theta_j)
$$

其中，H表示函数$$ f(x_1,x_2,\cdots,x_n) $$的海瑟矩阵（Hessian Matrix）:

$$
H(f) = \left [
        \begin{matrix}
            \frac {\partial^2 f}{\partial x_1^2} & \frac {\partial^2 f}{\partial x_1 \partial x_2} & \cdots & \frac {\partial^2 f}{\partial x_1 \partial x_n} \\
            \frac {\partial^2 f}{\partial x_2 \partial x_1} & \frac {\partial^2 f}{\partial x_2^2} & \cdots & \frac {\partial^2 f}{\partial x_2 \partial x_n} \\
            \vdots & \vdots & \ddots & \vdots \\
            \frac {\partial^2 f}{\partial x_n \partial x_1} & \frac {\partial^2 f}{\partial x_n \partial x_2} & \cdots & \frac {\partial^2 f}{\partial x_n^2} \\
        \end{matrix}
    \right ]
$$

相比于梯度下降法，牛顿法利用了二阶倒数信息，有更快的收敛速度。但是牛顿法需要计算庞大的海瑟矩阵取逆，销毁巨大，因此可能并不常用。