---
title:  “Spark ML - 梯度下降”
date:   2018-07-25 08:00:12 +0800
categories: machine learning
---

# **梯度**

在微积分中，对多元函数求偏导，以各参数的偏导数为元素的向量称之为梯度。比如函数$$ f(x,y) $$，梯度向量即是$$ (\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y})^T $$。几何意义上说，梯度是函数变化增加最快的地方；反之，沿着梯度向量相反的方向，最容易找到函数的最小值。


# **损失函数**

损失函数（Loss Function）是用来估量模型的预测值和真实值的不一致程度。损失函数越小，模型的鲁棒性就越好。经验风险函数通常包括经验风险项（损失函数均值）和正则项（惩罚项）。

$$
\theta^* = \arg \min_{\theta} \sum \limits_{i=1}^N L(y_i, f(x_i;\theta)) + \lambda \Phi(\theta)
$$

常用的损失函数：
- 平方损失函数（最小二乘）
    
    $$
    L(y, f(x)) = (y - f(x))^2 
    $$

- 对数损失函数/Softmax（逻辑回归）

    逻辑回归概率
    $$ P(Y=y|x) $$ :

    $$
    \begin{aligned}
    &h_\theta(x) = \frac{1}{1 + e^{-f(x)}} \\
    &P(y=1|x; \theta) = h_\theta(x)  \\
    &P(y=0|x; \theta) = 1 - h_\theta(x) \\
    &P(y|x; \theta) = (h_\theta(x))^y(1- h_\theta(x))^{1-y}
    \end{aligned}
    $$

    logistic损失函数表达式：

    $$
    \begin{aligned}
    L(y, P(y|x)) &= -\log P(y|x) \\
    &= \left \{
            \begin{aligned}
            &\log (1 + e^{-f(x)}) &, \ y = 1 \\
            &\log (1 + e^{f(x)}) &, \ y = 0
            \end{aligned}
        \right.
    \end{aligned}
    $$

    换个写法，可以表达为：

    $$
    L(y, P(y|x)) = y \log h_\theta(x) + (1-y) \log (1 - h_\theta(x))
    $$

    逻辑回归的目标式：

    $$
    J(\theta) = -\frac{1}{m} \sum_{i=1}^m [y^i \log h_\theta(x^i) + (1-y^i) \log (1 - h_\theta(x^i))]
    $$

- 指数损失函数（Adaboost）

    $$
    L(y, f(x)) = e^{-yf(x)}
    $$

- Hinge损失函数（SVM）

    $$
    L(y) = \max (0, 1 - t \centerdot y), \ t = \pm 1
    $$

    在线性SVM中，$$ y = wx + b $$。如果被正确分类，y和t正负号相同且
    $$ | y | \ge 1$$
    ，损失为0；否则损失为$$ 1-t \centerdot y $$。

- 其他损失函数（0-1损失函数，绝对值损失函数）

# **梯度下降法**

以线性回归为例，损失函数是残差平方和，目标是函数值最小。

$$
\begin{aligned}
&h_\theta(x) = \sum_{k=0}^n \theta_kx_k = \theta^Tx \\
&J(\theta) = \frac{1}{2} \sum_{i=1}^m(h_\theta(x^i)-y^i)^2
\end{aligned}
$$

- 代数法

    原始公式如下，$$ \alpha $$为步长： 

    $$
    \theta_j := \theta_j - \alpha \centerdot \frac{\partial}{\partial \theta_j} J(\theta)
    $$

    $$ J(\theta) $$的偏导推导过程：
    
    $$
    \begin{aligned}
    \frac{\partial}{\partial \theta_j} J(\theta) &= \frac{\partial}{\partial \theta_j} \frac{1}{2} \sum_{i=1}^m(h_\theta(x^i)-y^i)^2 \\
    &=2 \centerdot \frac{1}{2} \sum_{i=1}^m(h_\theta(x^i)-y^i) \centerdot \frac{\partial}{\partial \theta_j} (h_\theta(x^i)-y^i）\\
    &=\sum_{i=1}^m(h_\theta(x^i)-y^i) \centerdot \frac{\partial}{\partial \theta_j} (\sum_{k=0}^n \theta_kx_k^i - y^i）\\
    &=\sum_{i=1}^m(h_\theta(x^i)-y^i)x_j^i
    \end{aligned}
    $$

    最终得到的结果是：

    $$
    \theta_j := \theta_j - \alpha \sum_{i=1}^m(h_\theta(x^i)-y^i)x_j^i
    $$

    上述公式更新参数时使用了所以的样本点的梯度数据进行更新，所以又称为批量梯度下降法（Batch Gradient Descent）。批量梯度下降在每次循环中都使用所有样本点，在大样本数据下，训练速度较慢。有两个变种分别是随机梯度下降法（Stochastic Gradient Descent）和小批量梯度下降法（Mini-batch Gradient Descent）。随机梯度下降法在一次循环中仅选择一个样本点i进行更新：

    $$
    \theta_j := \theta_j - \alpha (h_\theta(x^i)-y^i)x_j^i
    $$

    小批量梯度下降法是一个折中方案，对应m个样本，选取其中的n个样本点进行更新

    $$
    \theta_j := \theta_j - \alpha \sum_{i=t}^{t+n-1}(h_\theta(x^i)-y^i)x_j^i
    $$

- 矩阵解法

    函数是从m*n的矩阵到实数域的映射，其梯度为：

    $$
    \nabla_Af(A) = \left [
        \begin{matrix}
            \frac{\partial f}{\partial A_{11}} & \cdots & \frac{\partial f}{\partial A_{1n}} \\
            \vdots & \ddots & \vdots \\
            \frac{\partial f}{\partial A_{m1}} & \cdots & \frac{\partial f}{\partial A_{mn}}
        \end{matrix}
    \right ]
    $$

    假设有m个样本数据，$$ x^i $$是n维列向量，其倒置$$ (x^i)^T $$作为矩阵$$ X $$的第i行:

    $$
    X_{m \times n} = \left [
        \begin{matrix}
            x_{11} & \cdots & x_{1n} \\
            \vdots & \ddots & \vdots \\
            x_{m1} & \cdots & x_{mn}
        \end{matrix}
    \right ]
    $$

    观测值列向量：

    $$
    \vec{y} = \left [
        \begin{matrix}
            y_{1} \\
            \vdots \\
            y_{m}
        \end{matrix}
    \right ]
    $$

    $$ J(\theta) $$的矩阵表达式

    $$
    J(\theta) = \frac{1}{2}(X\theta - \vec{y})^T(X\theta - \vec{y})
    $$

    矩阵求导，应用迹的特性：

    $$
    \begin{aligned}
    \nabla_\theta J(\theta) &= \nabla_\theta \frac{1}{2} (X\theta - \vec{y})^T(X\theta - \vec{y}) \\
    &= \frac{1}{2} \nabla_\theta (\theta^TX^T - \vec{y}^T)(X\theta - \vec{y}) \\
    &= \frac{1}{2} \nabla_\theta (\theta^TX^TX\theta - \theta^TX^T\vec{y} - \vec{y}^TX\theta + \vec{y}^T\vec{y}) \\
    &= \frac{1}{2} \nabla_\theta tr(\theta^TX^TX\theta - \theta^TX^T\vec{y} - \vec{y}^TX\theta + \vec{y}^T\vec{y}) \\
    &= \frac{1}{2} \nabla_\theta (tr\theta^TX^TX\theta  - 2 \centerdot tr \vec{y}^TX\theta) \\
    &= X^TX\theta  - X^T\vec{y}
    \end{aligned}
    $$

    令$$ J(\theta) = 0 $$，可得:

    $$
    \theta = (X^TX)^{-1}X^T\vec{y}
    $$

# **正则化**

由于可能出现的过拟合情况，需要加入正则项来进行修正；通过加入惩罚约束，使得某些系数为0。仍以线性回归为例：

- L2-norm/Ridge回归

    $$
    J(\theta) = \frac{1}{2} \sum_{i=1}^m(h_\theta(x^i)-y^i)^2 + \frac{\lambda}{2} \sum_{j=1}^n \theta_j^2
    $$

    修正后的梯度下降公式为：

    $$
    \theta_j := \theta_j(1 - \alpha \lambda) - \alpha \sum_{i=1}^m(h_\theta(x^i)-y^i)x_j^i
    $$

- L1-norm/Lasso回归

    $$
    J(\theta) = \frac{1}{2} \sum_{i=1}^m(h_\theta(x^i)-y^i)^2 + \frac{\lambda}{2} \sum_{j=1}^n | \theta_j |
    $$

    Lasso回归的损失函数不是连续可导，在mllib L1Updater里用了一个soft threasholding的方法处理参数更新。

- Elastic Net

    $$
    J(\theta) = \frac{1}{2} \sum_{i=1}^m(h_\theta(x^i)-y^i)^2 + \frac{\lambda}{2} \left (\rho \sum_{j=1}^n | \theta_j | + (1 - \rho) \sum_{j=1}^n \theta_j^2 \right)
    $$


# **Spark ML 2.3.1**

## 补充Mlib BLAS线性代数运算库

函数名称 | 说明
:---|:---
dot | 向量内积
axpy | 常量乘以向量加另一个向量
rot | 旋转
copy | 把向量x复制到向量y
swap | 交换x和y
nrm2 | L2-范数
asum | 绝对值求和
scal | 常数乘以向量
amax | 最大绝对值元素索引
syr |  $$ A := \alpha x x^T + A $$，其中x是n维列向量
gemn | 矩阵相乘
gemv | 矩阵与向量相乘

在Gradient.scala中定义了三种损失函数，均实现了抽象类Gradient中的compute方法：
- LogisticGradient
- LeastSquaresGradient
- HingeGradient

核心代码在GradientDescent的runMiniBatchSGD方法中，gradient.compute根据损失函数计算梯度，updater.compute更新每次迭代后的模型参数。在每次迭代中，随机抽取部分数据进行小批量梯度下降算法，treeAggregate将m个样本累加计算的过程通过Spark并行计算。

```scala
@DeveloperApi
object GradientDescent extends Logging {
  /**
   * Run stochastic gradient descent (SGD) in parallel using mini batches.
   * In each iteration, we sample a subset (fraction miniBatchFraction) of the total data
   * in order to compute a gradient estimate.
   * Sampling, and averaging the subgradients over this subset is performed using one standard
   * spark map-reduce in each iteration.
   *
   * @param data Input data for SGD. RDD of the set of data examples, each of
   *             the form (label, [feature values]).
   * @param gradient Gradient object (used to compute the gradient of the loss function of
   *                 one single data example)
   * @param updater Updater function to actually perform a gradient step in a given direction.
   * @param stepSize initial step size for the first step
   * @param numIterations number of iterations that SGD should be run.
   * @param regParam regularization parameter
   * @param miniBatchFraction fraction of the input data set that should be used for
   *                          one iteration of SGD. Default value 1.0.
   * @param convergenceTol Minibatch iteration will end before numIterations if the relative
   *                       difference between the current weight and the previous weight is less
   *                       than this value. In measuring convergence, L2 norm is calculated.
   *                       Default value 0.001. Must be between 0.0 and 1.0 inclusively.
   * @return A tuple containing two elements. The first element is a column matrix containing
   *         weights for every feature, and the second element is an array containing the
   *         stochastic loss computed for every iteration.
   */
  def runMiniBatchSGD(
      data: RDD[(Double, Vector)],
      gradient: Gradient,
      updater: Updater,
      stepSize: Double,
      numIterations: Int,
      regParam: Double,
      miniBatchFraction: Double,
      initialWeights: Vector,
      convergenceTol: Double): (Vector, Array[Double]) = {

    // convergenceTol should be set with non minibatch settings
    if (miniBatchFraction < 1.0 && convergenceTol > 0.0) {
      logWarning("Testing against a convergenceTol when using miniBatchFraction " +
        "< 1.0 can be unstable because of the stochasticity in sampling.")
    }

    if (numIterations * miniBatchFraction < 1.0) {
      logWarning("Not all examples will be used if numIterations * miniBatchFraction < 1.0: " +
        s"numIterations=$numIterations and miniBatchFraction=$miniBatchFraction")
    }

    val stochasticLossHistory = new ArrayBuffer[Double](numIterations)
    // Record previous weight and current one to calculate solution vector difference

    var previousWeights: Option[Vector] = None
    var currentWeights: Option[Vector] = None

    val numExamples = data.count()

    // if no data, return initial weights to avoid NaNs
    if (numExamples == 0) {
      logWarning("GradientDescent.runMiniBatchSGD returning initial weights, no data found")
      return (initialWeights, stochasticLossHistory.toArray)
    }

    if (numExamples * miniBatchFraction < 1) {
      logWarning("The miniBatchFraction is too small")
    }

    // Initialize weights as a column vector
    var weights = Vectors.dense(initialWeights.toArray)
    val n = weights.size

    /**
     * For the first iteration, the regVal will be initialized as sum of weight squares
     * if it's L2 updater; for L1 updater, the same logic is followed.
     */
    var regVal = updater.compute(
      weights, Vectors.zeros(weights.size), 0, 1, regParam)._2

    var converged = false // indicates whether converged based on convergenceTol
    var i = 1
    while (!converged && i <= numIterations) {
      val bcWeights = data.context.broadcast(weights)
      // Sample a subset (fraction miniBatchFraction) of the total data
      // compute and sum up the subgradients on this subset (this is one map-reduce)
      val (gradientSum, lossSum, miniBatchSize) = data.sample(false, miniBatchFraction, 42 + i)
        .treeAggregate((BDV.zeros[Double](n), 0.0, 0L))(
          seqOp = (c, v) => {
            // c: (grad, loss, count), v: (label, features)
            val l = gradient.compute(v._2, v._1, bcWeights.value, Vectors.fromBreeze(c._1))
            (c._1, c._2 + l, c._3 + 1)
          },
          combOp = (c1, c2) => {
            // c: (grad, loss, count)
            (c1._1 += c2._1, c1._2 + c2._2, c1._3 + c2._3)
          })
      bcWeights.destroy(blocking = false)

      if (miniBatchSize > 0) {
        /**
         * lossSum is computed using the weights from the previous iteration
         * and regVal is the regularization value computed in the previous iteration as well.
         */
        stochasticLossHistory += lossSum / miniBatchSize + regVal
        val update = updater.compute(
          weights, Vectors.fromBreeze(gradientSum / miniBatchSize.toDouble),
          stepSize, i, regParam)
        weights = update._1
        regVal = update._2

        previousWeights = currentWeights
        currentWeights = Some(weights)
        if (previousWeights != None && currentWeights != None) {
          converged = isConverged(previousWeights.get,
            currentWeights.get, convergenceTol)
        }
      } else {
        logWarning(s"Iteration ($i/$numIterations). The size of sampled batch is zero")
      }
      i += 1
    }

    logInfo("GradientDescent.runMiniBatchSGD finished. Last 10 stochastic losses %s".format(
      stochasticLossHistory.takeRight(10).mkString(", ")))

    (weights, stochasticLossHistory.toArray)

  }
```

参考：  
[机器学习之梯度下降-Andrew Ng机器学习笔记](https://blog.csdn.net/xiaocainiaodeboke/article/details/50371986)  
[梯度下降小结](https://www.cnblogs.com/pinard/p/5970503.html)  
[机器学习中损失函数](https://blog.csdn.net/u010976453/article/details/78488279)  
[机器学习-损失函数](http://www.csuldw.com/2016/03/26/2016-03-26-loss-function/)