---
title:  “Structuring Machine Learning Projects”
date:   2019-08-10 08:00:12 +0800
categories: deeplearning.ai
---

结构化机器学习项目(Structuring Machine Learning Projects)是deeplearning.ai的第三门课程，主要涉及机器学习的策略原则。本章概念和总结性的内容比较多，只列出了小部分章节。详细内容可以参考课程视频。


# 机器学习策略

1. 正交化(Orthogonalization)

2. 单一数字评估指标

    * 查准率(Precision)
    * 查全率(Recall)
    * 调和平均 ($$F_1 = \frac {2}{\frac {1}{P} + \frac {1}{R}}$$)

3. 优化指标

    在不同的场景，可能需要将其他一些指标加入到优化目标里，比如运行时间或者假阳性(False Postive)数量。

4. 训练/开发/测试集合划分

    开发集和测试集必须是来自同一分布。

5. 开发集和测试集的大小

    在机器学习的早期，70/30或者60/20/20分的经验法则是相当合理的。但是在现代机器学习中，当操作规模大得多的数据集，比如说你有1百万个训练样本，98%作为训练集，1%开发集，1%测试可能更为合理。
