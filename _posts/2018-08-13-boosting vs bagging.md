---
title:  “Scikit ML - 提升算法”
date:   2018-08-13 08:00:12 +0800
categories: machine learning
---

集成学习（ensemble learning）就是将多个弱学习器结合起来，组成一个强的学习器。个体学习器可以选择决策树，神经网络等等；可以全是决策树，或全是神经网络，也可以来自不同算法。

目前主要有两种集成方式：
- Boosting：个体学习器间存在强依赖关系，串行生成
- Bagging：个体之间不存在强依赖关系，并行生成，如随机森林（Random Forrest）


# **自适应提升（Adaboost）**

Adaboost的基本思想是由初始训练数据，训练得出第一个基学习器；根据基学习器表现对样本进行调整，在之前学习器做错的样本上增加权重，训练下一个基学习器；重复T次，将T个学习器加权结合。

给定训练集$$ (x_1, y_1), (x_2, y_2), \cdots, (x_m, y_m) $$，其中 $$ y_i \in \{-1, 1\} $$。Adaboost的损失函数为指数函数
$$ e^{-yf(x)} $$，参数的推导过程参考文末的知乎链接。算法流程：

1. 初始化训练数据(m个样本）权值分布，每个样本都被赋予相同权值1/m

    $$
    D_1(i) = \frac{1}{m}, i = 1, 2, \cdots, m
    $$

2. 进入T次循环，即基学习器个数为T个

3. 根据当前权值分布$$ D_t $$的数据，学习得到分类器$$ h_t $$

4. 计算当前学习器误差

    $$
    \begin{aligned}
    \epsilon_t &= P_{i \sim D_t}(h_t(x_i) \neq y_i) \\
    &=\sum\nolimits_{i=1}^m D_t(i) \mathcal{I} (h_t(x_i) \neq y_i)
    \end{aligned}
    $$

5. 计算当前学习器权重

    Adaboost算法让正确率高的分类器占整体的权值更高，让正确率低的分类器权值更低。

    $$
    \alpha_t = \frac{1}{2} \log \frac{1-\epsilon_t}{\epsilon_t}
    $$

6. 得到下一个循环的训练数据权重分布

    如果某个样本点已经被准确地分类，那么在构造下一个训练集中，它被选中的概率就被降低；相反，如果某个样本点没有被准确地分类，那么它的权重就得到提高。

    $$
    D_{t+1}(i) = \frac{D_t(i)e^{-\alpha_ty_ih_t(x_i)}}{2[\epsilon_t(1-\epsilon_t)]^{1/2}}
    $$

7. 最终分类器

    $$
    H(x) = sign \left( \sum_{t=1}^T \alpha_th_t(x) \right)
    $$

伪代码：

![image01]({{site.baseurl}}/image/20180813/adaboost.jpg)

# **梯度提升决策树（Gradient Boosting Decision Tree）**

理论上，梯度提升算法(Gradient Boosting Machine)可以选择不同的学习算法作为基学习器，GBDT实际是GBM的一种，也就是GB和DT结合。决策树易于理解，可解释性强，预测速度快。通过梯度提升的方法集成多个决策树，能够比较好的解决过拟合问题。

在GBDT的迭代过程中，假设前一轮迭代得到学习器是$$ f_{t-1}(x) $$，损失函数$$ L(y, f_{t-1}(x)) $$，本轮的目标是找到一个弱学习器$$ h_t(x) $$，使得损失函数$$ L(y, f_{t-1}(x) + h_t(x)) $$最小。

网上一个GBDT的例子：

![image02]({{site.baseurl}}/image/20180813/gbdt_example.png)

## GBDT回归

回归树总体类似于分类树，区别在于，回归树的每个节点得到一个预测值，等于该节点下所有点的均值。属性分裂时需要找到最好的分割点，衡量标准不再是熵或者基尼系数，而是最小平方误差。

给定训练集$$ (x_1, y_1), (x_2, y_2), \cdots, (x_m, y_m) $$，算法流程：

1. 初始化弱学习器

    $$
    f_0(x) = \arg \min \sum_{i=1}^m L(y_i, c)
    $$

2. 进入T次循环，即基学习器个数为T个

3. 对样本$$ i = 1,2,\cdots,m $$，计算损失函数的负梯度：

    $$
    r_{ti} = - \left[ \frac{\partial L(y_i, f(x_i))}{\partial f(x_i)} \right ]_{f(x) = f_{t-1}(x)}
    $$

4. 利用$$ (x_i, r_{ti}), i = 1,2,\cdots,m $$，拟合一棵CART回归树。其对应的页节点区域$$ R_{tj}, j = 1,2,\cdots,J $$，其中j为叶节点个数。

5. 针对每一个叶子节点的样本，使损失函数最小，拟合叶节点最好输出值

    $$
    c_{tj} = \arg \min \sum_{x_i \in R_{tj}} L(y_i, f_{t-1}(x_i) + c)
    $$

6. 更新强学习器

    $$
    \begin{aligned}
    &h_t(x) = \sum_{j=1}^J c_{tj} \mathcal{I}(x \in R_{tj}) \\
    &f_t(x) = f_{t-1} + h_t(x)
    \end{aligned}
    $$

7. 最终得到强学习器

    $$
    f(x) = \sum_{t=1}^T \sum_{j=1}^J c_{tj} \mathcal{I}(x \in R_{tj})
    $$

伪代码如下（符号跟上面稍有不同）：

![image03]({{site.baseurl}}/image/20180813/gbdt.png)

对于回归算法，常用损失函数包括
- 均方差 $$ L(y, f(x)) = \frac{1}{2}(y - f(x))^2 $$

    负梯度误差为$$ y_i - f(x_i) $$
- 绝对损失 
$$ L(y, f(x)) = | y - f(x)| $$

    负梯度误差为$$ sign(y_i - f(x_i)) $$
- Huber损失，均方差和绝对损失的折中产物

    $$
    L(y, f(x)) = \left \{
            \begin{aligned}
            &\frac{1}{2} (y - f(x))^2 & \ |y - f(x)| \leq \delta \\
            &\delta \left ( |y - f(x)| - \frac{\delta}{2} \right ) & \ |y - f(x)| > \delta
            \end{aligned}
        \right.
    $$

    对应负梯度误差：

    $$
    r(y_i, f(x_i)) = \left \{
            \begin{aligned}
            &y_i - f(x_i) & \ |y - f(x)| \leq \delta \\
            &\delta \centerdot sign(y_i - f(x_i)) & \ |y - f(x)| > \delta
            \end{aligned}
        \right.
    $$

可见，当采用均方差作为损失函数时，残差作为负梯度的一种特例计算比较简单，意义也比较明确。

## GBDT分类

分类算法的样本输出不是连续值，而是离散的类别，因此无法直接从输出类别拟合输出误差。一种方法是用指数损失函数
$$
L(y, f(x)) = e^{-yf(x)}
$$
，GBDT退化为Adaboost算法；另一种是用类似于逻辑回归的对数损失函数
$$
L(y, f(x)) = \log(1 + e^{-yf(x)})
$$

以二元GBDT分类算法为例，负梯度误差为：

$$
r_{ti} = - \left[ \frac{\partial L(y_i, f(x_i))}{\partial f(x_i)} \right ]_{f(x) = f_{t-1}(x)} = \frac{y_i}{1 + e^{y_i f(x_i)}}
$$

各叶子节点的最佳残差拟合值为：

$$
c_{tj} = \arg \min \sum_{x_i \in R_{tj}} \log(1 + e^{-y_i(f_{t-1}(x_i) + c)})
$$

由于比较难优化，一般由近似值代替：

$$
c_{tj} = \frac{\sum_{x_i \in R_{tj}} r_{ti}} {\sum_{x_i \in R_{tj}}|r_{ti}|(1 - |r_{ti}|)}
$$

# **XGBoost**

XGBoost是华人陈天奇提出来的提升算法，在Kaggle比赛里被称为“神器”。回顾K个学习器模型和目标函数：

$$
\begin{aligned}
&\hat{y}_i = \sum_{k=1}^K f_k(x_i) \\
&\Omega(f) = \gamma T + \frac{1}{2}\lambda || \omega || ^2 \\
&\mathcal{L}(\phi) = \sum_{i} l(y_i, \hat{y}_i) + \sum_{k} \Omega(f_k)
\end{aligned}
$$

其中K是树的个数，$$ f_k(x_i) $$是第k棵树对输入$$ x_i $$的输出得分, $$ \Omega(f) $$是对树中叶子节点个数$$ T $$和每个树叶子节点输出分数$$ \omega_j $$的L2正则修正项。

泰勒展开：

$$
f(x + \Delta x) = \frac{f(x)}{0!} + \frac{f^{'}(x)}{1!} \Delta x + \frac{f^{''}(x)}{2!} \Delta x^2 + \cdots
$$

将目标函数做二次近似：

$$
\begin{aligned}
\mathcal{L}^{(t)} &= \sum_{i=1}^n l(y_i, \hat{y}_i^{(t-1)} + f_t(x_i)) + \Omega(f_t) \\
&\simeq \sum_{i=1}^n [ l(y_i, \hat{y}_i^{(t-1)}) + g_if_t(x_i) + \frac{1}{2} h_i f_t^2(x_i)] + \Omega(f_t) \\
& where \ g_i = \frac{\partial l(y_i, \hat{y}_i^{(t-1)})}{\partial \hat{y}_i^{(t-1)}}, h_i = \frac{\partial^2 l(y_i, \hat{y}_i^{(t-1)})}{\partial (\hat{y}_i^{(t-1)})^2}
\end{aligned}
$$

移除常量$$ l(y_i, \hat{y}_i^{(t-1)}) $$
，定义$$ I $$ 是每个叶子上的样本集合$$ I_j = {i|q(x_i)=j} $$

$$
\begin{aligned}
\tilde{\mathcal L}^{(t)} &= \sum_{i=1}^n [ g_if_t(x_i) + \frac{1}{2} h_i f_t^2(x_i)] + \gamma T + \frac{1}{2}\lambda \sum_{j=1}^T || \omega_j || ^2 \\
&= \sum_{j=1}^T [(\sum_{i \in I_j} g_i)\omega_j + \frac{1}{2}(\sum_{i \in I_j} h_i + \lambda)\omega_j^2] + \gamma T
\end{aligned}
$$

对$$ \omega_j $$求导可得：

$$
\begin{aligned}
& G_j = \sum_{i \in I_j} g_i, H_j = \sum_{i \in I_j} h_i \\
& \omega_j = - \frac{G_i}{ H_j + \lambda} \\
& \tilde{\mathcal L}^{(t)} = - \frac{1}{2} \sum_{j=1}^T \frac{G_i^2}{H_i + \lambda} + \gamma T
\end{aligned}
$$

最后利用如下增益公式，可以选取合适的分裂特征：

$$
Gain = \frac{G_L^2}{H_L + \lambda} + \frac{G_R^2}{H_R + \lambda} - \frac{(G_L+G_R)^2}{H_L + H_R + \lambda} - \gamma
$$

作为特例，当均方差作为损失函数时应用到XGBoost时，$$ g_i = 2( \hat{y}_i^{(t-1)} - y_i), h_i = 2 $$。论文上的例子如下：

![image04]({{site.baseurl}}/image/20180813/xgboost_1.jpg)

![image05]({{site.baseurl}}/image/20180813/xgboost_2.jpg)

参考：  
[简书：Adaboost算法](https://www.jianshu.com/p/389d28f853c0)  
[Adaboost入门详解](https://louisscorpio.github.io/2017/11/28/AdaBoost%E5%85%A5%E9%97%A8%E8%AF%A6%E8%A7%A3/)  
[知乎：GBDT梯度提升树算法](https://zhuanlan.zhihu.com/p/29802325)  
[简书：GBDT梯度提升决策树](https://www.jianshu.com/p/005a4e6ac775)  
[博客园：XGBoost原理](https://www.cnblogs.com/xlingbai/p/8274250.html)
[简书：xgboost原理没你想象那么难](https://www.jianshu.com/p/7467e616f227)  
[知乎：xgboost原理](https://zhuanlan.zhihu.com/p/31706381)  
[CSDN: xgboost入门与实战](https://blog.csdn.net/sb19931201/article/details/52557382)  
[KDD 2016: XGBoost: A scalable Tree Boosting System](http://www.kdd.org/kdd2016/papers/files/rfp0697-chenAemb.pdf)