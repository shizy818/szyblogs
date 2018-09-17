---
title:  “Spark ML - 决策树”
date:   2018-08-11 08:00:12 +0800
categories: machine learning
---

决策树模型具有解释性强的特点，因此容易被接受。在使用模型时，根据输入参数依次在各个树节点判断，直到叶子节点即为预测结果。


构建决策树模型的关键步骤是选择`分裂属性`，常用的有三种方法：

- 信息增益（Information Gain）
    
    根据香农的信息论，熵表示不确定度，均匀分布时，不确定度最大，也就是熵最大。假设样本集合S，假设样本有k个类别，
    $$ |C_i| $$表示类别i的样本个数，
    $$ |S| $$表示样本总个数，集合S熵定义为：

    $$
    H(S) = - \sum_{i=1}^k \frac{|C_i|}{|S|} \log_2 \frac{|C_i|}{|S|}
    $$

    如果依据某特征对数据集分类，信息熵减少，其差值即为信息增益。$$ S_v $$是 $$ S $$中属性A的值为v的子集。

    $$
    Gain(S, A) = H(S) - \sum_{v \in Values(A)} \frac{|S_v|}{|S|} H(S_v)
    $$

    每层分类时，选择使得信息增益最大的属性作为分裂属性，典型算法是ID3。缺点是样本分布不均时，可能导致过拟合。比如样本某属性值各不相同，以此属性划分信息增益最大，则只需要一颗高度为1的树就可以将样本完全分类。

- 信息增益比率（Gain Ratio）

    针对上述问题，引入信息增益比率定义：

    $$
    \begin{aligned}
    &H_A(S) = \sum_{v \in Values(A)} \frac{|S_v|}{|S|} \log_2 \frac{|S_v|}{|S|} \\
    &G_R(S,A) = \frac{G(S,A)}{H_A(S)}
    \end{aligned}
    $$

    典型算法C4.5，分裂时偏向于取值较少的特征。

- 基尼系数(Gini index)

    另一种数据不纯度的定义：

    $$
    \begin{aligned}
    &Gini(p) = \sum_{i=i}^k p_k(1-p_k) =  1- \sum_{i=i}^k p_k^2 \\
    &Gini(S) = 1- \sum_{i=i}^k \left ( \frac{|C_i|}{|S|} \right )^2
    \end{aligned}
    $$

    典型算法CART。但是CART是一棵二叉树，拥有多值的属性需要以某个取值v作为划分点，对样本S划分为两个子集。

    $$
    Gini(S,v) = \frac{|S_1|}{|S|}Gini(S_1) + \frac{|S_2|}{|S|}Gini(S_2)
    $$

    然后从所有可能划分的$$ Gini(S,v) $$找出基尼指数最小的划分点。

# **Spark ML 2.3.1**