---
title:  “Spark ML - 带标签测试数据模型评价”
date:   2018-08-01 08:00:12 +0800
categories: machine learning
---

# **ROC/AUC**

如果测试数据是带标签的，ROC（Receiver Operating Characteristic）曲线和曲线面积AUC（Area Under the Curve）通常被用来评价一个"二分类器"的优劣。

涉及到一些概念：
- 预测值为正例，记为Positive
- 预测值为负例，记为Negative
- 预测值与真实值相同，记为True
- 预测值与真实值相反，记为False


于是有下表:

真实值\预测值 |  正例 | 反例
:----|:----|:----
正例 | TP | FN
反例 | FP | TN

样本中：
- 真实正例总数 TP + FN
- TPR (True Positive Rate) = TP/(TP + FN)
- 真实反例总数 FP + TN
- FPR (False Positive Rate) = FP/(FP + TN)

分类器通常会输出一个概率，表示某个样本多大概率属于正例（或者反例）。假设我们有20个测试样本，“Class”表示每个测试样本真实标签（p为正样本，n为负样本），“Score”表示测试样本属于正样本概率。

![image01]({{site.baseurl}}/image/20180801/score_ranking.png)

从高到底依次将“Score”值作为截断阈值（discrimination threshold），当测试样本输出概率大于或等于阈值时，判断为正样本；否则为负样本。
- 阈值取0.9，只有第1个点被预测为正样本；TP=1/FN=9/FP=0/TN=10；TPR=0.1/FPR=0
- 阈值取0.8，第1，2个点被预测为正样本；TP=2/FN=8/FP=0/TN=10；TPR=0.2/FPR=0
- 阈值取0.7，第1，2，3个点被预测为正样本；TP=2/FN=8/FP=1/TN=9；TPR=0.2/FPR=0.1
- ......
- 阈值取0.1，所有点均被预测为正样本；TP=10/FN=0/FP=10/TN=0；TPR=1.0/FPR=1.0

![image02]({{site.baseurl}}/image/20180801/roc_example.png)

以FPR为横轴，TPR为纵坐标是，即可画出ROC曲线。第一个坐标点一定是（0, 1/n），最后一个坐标点一定是（1，1）。因为是按照“Score”逆序，对于一个好模型，真实正样本集中在上面。

AUC定义为ROC曲线下面积，取值范围在0.5到1之间。AUC是一个概率，当你随机挑选一个正样本以及一个负样本，当前的分类算法根据计算得到的Score值将这个正样本排在负样本前面的概率就是AUC值。AUC值越大，当前的分类算法越有可能将正样本排在负样本前面，即能够更好的分类。

**ROC有个很好的特性：当测试集的正负样本分布不平衡时，ROC曲线能够保持不变**

# **PR曲线**

精确率（Precision） P = TP / (TP + FP)  
召回率（Recal）R = TP / (TP + FN)

![image03]({{site.baseurl}}/image/20180801/pr.jpg)

以Recall为横轴，Precision为纵轴，即可画出PR曲线：
- 阈值取0.9，只有第1个点被预测为正样本；TP=1/FN=9/FP=0/TN=10；R=0.1/P=1.0
- 阈值取0.8，第1，2个点被预测为正样本；TP=2/FN=8/FP=0/TN=10；R=0.2/P=1.0
- 阈值取0.7，第1，2，3个点被预测为正样本；TP=2/FN=8/FP=1/TN=9；R=0.2/P=0.66
- ......
- 阈值取0.1，所有点均被预测为正样本；TP=10/FN=0/FP=10/TN=0；R=1/P=0.5

![image04]({{site.baseurl}}/image/20180801/pr_example.png)


参考：  
[如何计算ROC/AUC](http://alexkong.net/2013/06/introduction-to-auc-and-roc/)  