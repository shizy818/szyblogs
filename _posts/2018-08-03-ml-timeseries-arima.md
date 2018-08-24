---
title:  “StatsModels - 时间序列ARIMA”
date:   2018-08-03 08:00:12 +0800
categories: machine learning
---

时间序列是将某种统计指标的数值，按时间先后顺序排序所形成的数列。时间序列的预测就是通过分析时间序列，根据时间序列所反映出来的发展过程、方向和趋势，进行类推或延伸，预测下一段时间或以后若干年内可能达到的水平。时间序列的异常检测就是通过历史的数据分析，查看当前的数据是否发生了明显偏离了正常的情况。


随机过程的特征有均值，方差，协方差等等。如果随机过程的特征随时间变化，则此过程是非平稳的；反之，如果随机过程不随时间变化，则此过程是平稳的。传统时序模型建立在平稳的时间序列基础上，判断时序是否稳定的方法是通过ADF(Augmented Dickey-Fuller)检验。如果时间序列不稳定，可以通过一些操作（如取对数，差分）使得其稳定，然后进行ARIMA模型预测。

1. p阶自回归(Auto Regression)模型AR(p)，该模型认为通过时间序列过去的点的线性组合加上白噪声，即可预测当前时间点。

    $$
    x_t = \delta + \phi_1x_{t-1} + \phi_2x_{t-2} + \cdots + \phi_px_{t-p} + \epsilon_t
    $$

    AR(p)的特征方程是：

    $$
    \Phi(L) = 1 - \phi_1L - \phi_2L^2 - \cdots - \phi_pL^p=0
    $$

    AR(p)的充要条件是特征根都在单位圆之外。

2. q阶移动平均（Moving Average）模型MA(q)，该模型是历史白噪声的线性组合。

    $$
    x_t = \mu + \theta_1\epsilon_{t-1} + \theta_2\epsilon_{t-2} + \cdots + \theta_q\epsilon_{t-q} + \epsilon_t
    $$

3. 自回归求和移动平均（Auto Regression Moving Average）模型ARMA(p,q)，AR和MA模型组合。

    $$
    x_t = \delta + \phi_1x_{t-1} + \phi_2x_{t-2} + \cdots + \phi_px_{x-p} + \theta_1\epsilon_{t-1} + \theta_2\epsilon_{t-2} + \cdots + \theta_q\epsilon_{t-q} + \epsilon_t
    $$

4. 自回归差分移动平均（Auto Regression Integrated Moving Average）模型ARIMA(p,d,q), 即ARMA加上差分算子。

    令$$ y_t $$为：

    $$
    \begin{aligned}
    &\bigtriangleup x_t = x_t - x_{t-1} = (1-L)x_t \\
    &\bigtriangleup^2x_t = \bigtriangleup x_t - \bigtriangleup x_{t-1} = (1-L)x_t - (1-L)x_{t-1} = (1-L)^2x_t \\
    &y_t = \bigtriangleup^dx_t = (1-L)^dx_t
    \end{aligned}
    $$

    则公式ARIMA(p,d,q)写成：

    $$
    y_t = \delta + \phi_1y_{t-1} + \phi_2y_{t-2} + \cdots + \phi_py_{t-p} + \theta_1\epsilon_{t-1} + \theta_2\epsilon_{t-2} + \cdots + \theta_q\epsilon_{t-q} + \epsilon_t
    $$

## ARIMA模型运用流程

- 数据可视化，识别平稳性。  
- 对非平稳的时间序列数据，做差分，得到平稳序列。
- 建立合适的模型
    * 平稳化处理后，若偏自相关函数是截尾的，而自相关函数是拖尾的，则建立AR模型；
    * 若偏自相关函数是拖尾的，而自相关函数是截尾的，则建立MA模型；
    * 若偏自相关函数和自相关函数均是拖尾的，则序列适合ARMA模型。
- 模型的阶数在确定之后，对ARMA模型进行参数估计，比较常用是最小二乘法进行参数估计。
- 假设检验，判断（诊断）残差序列是否为白噪声序列。
- 利用已通过检验的模型进行预测。

## 补充数学  

随机过程$$ {x_t} $$中的每一个元素$$ x_t, t=1,2,\dots $$都是随机变量。平稳随机过程的期望为常数：

$$ E(x_t) = \mu, t=1,2,\dots $$

平稳随机过程方差也是一个常量:

$$ Var(x_t) = \sigma^2 $$

相隔k期的随机变量$$ x_t $$和$$ x_{t-k} $$的协方差即滞后k期的自协方差:

$$
\begin{aligned}
&\gamma_k = Cov(x_t, x_{t-k}) = E[(x_t- \mu)(x_{t-k}- \mu)] \\
&\gamma_0 = Var(x_t) = \sigma^2
\end{aligned}
$$

自相关系数：

$$
\begin{aligned}
\rho_k &= \frac{Cov(x_t, x_{t-k})}{\sqrt{Var(x_t)}\sqrt{Var(x_{t-k})}} \\
&= \frac{Cov(x_t, x_{t-k})}{\sigma^2} \\
&= \frac{\gamma_k}{\gamma_0}
\end{aligned}
$$

以滞后期k为变量的自相关系数列$$ \rho_k (k = 0, 1, 2, \cdots) $$称为自相关函数（ACF），**描述了时间序列观测值与其过去观察值之间的线性相关性**。自相关函数是对称的$$ \rho_k = \rho_{-k}, \rho_0 = 1 $$，因为：

$$ Cov(x_t, x_{t-k}) = Cov(x_{t-k}, x_t) = Cov(x_{t'}, x_{t'+k}) $$ 

**偏自相关函数（PACF）描述在给定中间观察值的条件下，时间序列观测值与过去观测值之间的线性相关性。**

如果用$$ \phi_{kj} $$表示k阶自回归过程中的第j个回归系数：

$$
x_t = \phi_{k1}x_{t-1} + \phi_{k2}x_{t-2} + \cdots + \phi_{kk}x_{t-k} + \epsilon_t
$$

$$ \phi_{kk} $$ 是最后一个回归系数，若把$$ \phi_{kk} $$看做滞后期k的函数，则称$$ \phi_{kk} $$为偏自相关系数（PACF），**恰好表示了$$ x_t $$与$$ x_{t-k} $$在排除了中间变量$$ x_{t-1}, x_{t-2}, \cdots, x_{t-k+1} $$影响后的自相关系数**。

参考：  
[简书：时间序列模型](https://www.jianshu.com/p/4130bac8ebec)  
[CSDN: ARIMA模型](https://blog.csdn.net/aspirinvagrant/article/details/46323271)  
[时间序列之ARIMA模型](https://www.cnblogs.com/bradleon/p/6827109.html)  
[东北财经大学计量经济学](http://classroom.dufe.edu.cn/spsk/c102/wlkj/CourseContents/Chapter10)