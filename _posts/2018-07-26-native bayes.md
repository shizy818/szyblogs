---
title:  “Spark ML - 朴素贝叶斯”
date:   2018-07-26 08:00:12 +0800
categories: machine learning
---

朴素贝叶斯是高斯判别分析之后，介绍的第二种生成学习算法。

# **垃圾邮件分类问题**

Andrew Ng大神在cs 229课程中以垃圾邮件分类为例，y=1表示该邮件是垃圾邮件。首先需要定义输入向量，一个简单的方案是建立单词字典（假设字典里面有n个单词），然后如果邮件中包含字典第i个位置的单词，则相应位置为1。比如“buy a coffee”，表示如下：

$$
    x = \left [
        \begin{matrix}
            1 \\
            0 \\
            \vdots \\
            1 \\
            \vdots \\
            1 \\
            \vdots \\
            0
        \end{matrix}
    \right ]
    \begin{matrix}
        a \\
        abstract \\
        \vdots \\
        buy \\
        \vdots \\
        coffee \\
        \vdots \\
        zyme
    \end{matrix}
$$


对
$$ p(x|y) $$建模：

$$
\begin{aligned}
p(x_1,x_2,\cdots,x_n|y) &= p(x_1|y)p(x_2|y,x_1)\cdots p(x_n|y,x_1,x_2,\cdots,x_{n-1}) \\
&= p(x_1|y)p(x_2|y)\cdots p(x_n|y) \\
&=\prod_{i=1}^n p(x_i|y)
\end{aligned}
$$

上面的推导中引入了一个假设，即文本中每一个单词的出现概率是独立的。当然这个假设是不正确的，或许可以尝试用一阶马尔科夫链建模，即文本中某个单词出现的概率，依赖于上一个单词是什么。当然大神说独立假设的效果挺好，那接着用最大似然估计建模：

$$
\begin{aligned}
L(\theta_y, \theta_{j|y=0}, \theta_{j|y=1}) &= \prod_{i=1}^m p(x^{(i)}, y^{(i)}; \theta_y, \theta_{j|y=0}, \theta_{j|y=1}) \\
&= \prod_{i=1}^m \prod_{j=1}^n p(x_j^{(i)}|y^{(i)};\theta_{j|y=0}, \theta_{j|y=1}) \centerdot p(y^{(i)}; \theta_y)
\end{aligned}
$$

其中：

$$
\begin{aligned}
&\theta_y = p(y=1) \\
&\theta_{j|y=0} = p(x_j=1|y=0) \\
&\theta_{j|y=1} = p(x_j=1|y=1)
\end{aligned}
$$

常规取对数求导可得：

$$
\begin{aligned}
&\theta_{j|y=0} = \frac{\sum_{i=1}^m \mathcal{I}\{x_j^{(i)}=1,y^{(i)}=0\}}{\sum_{i=1}^m \mathcal{I}\{y^{(i)}=0\}} \\
&\theta_{j|y=1} = \frac{\sum_{i=1}^m \mathcal{I}\{x_j^{(i)}=1,y^{(i)}=1\}}{\sum_{i=1}^m \mathcal{I}\{y^{(i)}=1\}} \\
&\theta_y = \frac{\sum_{i=1}^m \mathcal{I}\{y^{(i)}=1\}}{m}
\end{aligned}
$$

$$ \theta_{j|y=1} $$
结论中分子部分的解释是包含字典里第j个单词的垃圾邮件个数；分母部分表示所有垃圾邮件的个数。

预测的时候：

$$
\arg \max_y p(y|x) = \arg \max_y p(x|y)p(y)
$$

训练好参数
$$ \theta_y, \theta_{j|y=0}, \theta_{j|y=1} $$，分类结果就是使得$$ p(y|x) $$概率最大的y值。

可能会出现极端情况，比如某个词没有出现，造成$$ p(x_i) = 0 $$。因此需要引入了拉普拉斯平滑，参数计算公式变成：

$$
\begin{aligned}
&\theta_{j|y=0} = \frac{\sum_{i=1}^m \mathcal{I}\{x_j^{(i)}=1,y^{(i)}=0\} + 1}{\sum_{i=1}^m \mathcal{I}\{y^{(i)}=0\} + 2} \\
&\theta_{j|y=1} = \frac{\sum_{i=1}^m \mathcal{I}\{x_j^{(i)}=1,y^{(i)}=1\} + 1}{\sum_{i=1}^m \mathcal{I}\{y^{(i)}=1\} + 2} \\
&\theta_y = \frac{\sum_{i=1}^m \mathcal{I}\{y^{(i)}=1\} + 1}{m + 2}
\end{aligned}
$$

# **Spark ML 2.3.1**

Spark ML中NaiveBayes可以是Multinomial或者Bernoulli类型的，核心是调用了org.apache.spark.ml.classification.NaiveBayes类的trainWithLabelCheck方法。

```scala
class NaiveBayes private (
    private var lambda: Double,
    private var modelType: String) extends Serializable with Logging {

    /**
    * Run the algorithm with the configured parameters on an input RDD of LabeledPoint entries.
    *
    * @param data RDD of [[org.apache.spark.mllib.regression.LabeledPoint]].
    */
    @Since("0.9.0")
    def run(data: RDD[LabeledPoint]): NaiveBayesModel = {
        val spark = SparkSession
        .builder()
        .sparkContext(data.context)
        .getOrCreate()

        import spark.implicits._

        val nb = new NewNaiveBayes()
        .setModelType(modelType)
        .setSmoothing(lambda)

        val dataset = data.map { case LabeledPoint(label, features) => (label, features.asML) }
        .toDF("label", "features")

        // mllib NaiveBayes allows input labels like {-1, +1}, so set `positiveLabel` as false.
        val newModel = nb.trainWithLabelCheck(dataset, positiveLabel = false)

        val pi = newModel.pi.toArray
        val theta = Array.fill[Double](newModel.numClasses, newModel.numFeatures)(0.0)
        newModel.theta.foreachActive {
        case (i, j, v) =>
            theta(i)(j) = v
        }

        assert(newModel.oldLabels != null,
        "The underlying ML NaiveBayes training does not produce labels.")
        new NaiveBayesModel(newModel.oldLabels, pi, theta, modelType)
    }
}
```

默认情况下，weightCol的值是1.0。伯努利分布的样本记录为(0.0, [1.0,1.0,0.0,0.0])，表示分类标签为0，特征是1一个四维向量。aggregateByKey分别计算每种标签的出现频率。
```scala
@Since("1.5.0")
class NaiveBayes @Since("1.5.0") (
    @Since("1.5.0") override val uid: String)
  extends ProbabilisticClassifier[Vector, NaiveBayes, NaiveBayesModel]
  with NaiveBayesParams with DefaultParamsWritable {
  
    /**
    * ml assumes input labels in range [0, numClasses). But this implementation
    * is also called by mllib NaiveBayes which allows other kinds of input labels
    * such as {-1, +1}. `positiveLabel` is used to determine whether the label
    * should be checked and it should be removed when we remove mllib NaiveBayes.
    */
    private[spark] def trainWithLabelCheck(
        dataset: Dataset[_],
        positiveLabel: Boolean): NaiveBayesModel = {
        val instr = Instrumentation.create(this, dataset)
        ......

        // 分布式计算label标签的频率

        // '$'是方法getOrDefault()的别名
        val aggregated = dataset.select(col($(labelCol)), w, col($(featuresCol))).rdd
        .map { row => (row.getDouble(0), (row.getDouble(1), row.getAs[Vector](2)))
        }.aggregateByKey[(Double, DenseVector)]((0.0, Vectors.zeros(numFeatures).toDense))(
        seqOp = {
            case ((weightSum: Double, featureSum: DenseVector), (weight, features)) =>
            requireValues(features)
            BLAS.axpy(weight, features, featureSum)
            (weightSum + weight, featureSum)
        },
        combOp = {
            case ((weightSum1, featureSum1), (weightSum2, featureSum2)) =>
            BLAS.axpy(1.0, featureSum2, featureSum1)
            (weightSum1 + weightSum2, featureSum1)
        }).collect().sortBy(_._1)

        val numLabels = aggregated.length
        instr.logNumClasses(numLabels)
        val numDocuments = aggregated.map(_._2._1).sum

        val labelArray = new Array[Double](numLabels)
        val piArray = new Array[Double](numLabels)
        val thetaArray = new Array[Double](numLabels * numFeatures)

        val lambda = $(smoothing)
        val piLogDenom = math.log(numDocuments + numLabels * lambda)
        var i = 0
        aggregated.foreach { case (label, (n, sumTermFreqs)) =>
        labelArray(i) = label
        piArray(i) = math.log(n + lambda) - piLogDenom
        val thetaLogDenom = $(modelType) match {
            case Multinomial => math.log(sumTermFreqs.values.sum + numFeatures * lambda)
            case Bernoulli => math.log(n + 2.0 * lambda)
            case _ =>
            // This should never happen.
            throw new UnknownError(s"Invalid modelType: ${$(modelType)}.")
        }
        var j = 0
        while (j < numFeatures) {
            thetaArray(i * numFeatures + j) = math.log(sumTermFreqs(j) + lambda) - thetaLogDenom
            j += 1
        }
        i += 1
        }

        val pi = Vectors.dense(piArray)
        val theta = new DenseMatrix(numLabels, numFeatures, thetaArray, true)
        val model = new NaiveBayesModel(uid, pi, theta).setOldLabels(labelArray)
        instr.logSuccess(model)
        model
    }
}
```