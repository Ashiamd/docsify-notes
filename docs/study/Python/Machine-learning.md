# 机器学习google教程学习

>建议学习时，打开两个页面，一个切换English，一个切换中文-简体，实在不懂再切到中文页面查看相对应的描述
>
>google的机器学习>速成课：
>
>https://developers.google.cn/machine-learning/crash-course/ml-intro

## ML Concepts

### Introduction to ML

1. 为什么使用机器学习：

   比如遇到像是拼写检查检查、软件多语言化、复杂图像处理等，可以使用机器学习。

   拼写检查，多输入样例，给其判断就好了；

   软件多语言化，做出一种语言，然后其他的语言靠样例输入，有充足样例即可；

   复杂图像处理也是靠样例多次输入，让其自行生成模型处理事件就好了

   

   学习机器学习，还有一个好处，就是改变我们过去处理问题的思维方式，过去处理方法就是完全从已知的逻辑、方式出发；而机器学习更多的是靠统计学等方式处理无法完全确定或事先不知道的事件。

### Framing

1. 了解几个基本的概念：
   + Labels标签：需要预测的目标(y)
   
   + Features特征：影响y的线性回归方程自变量(x)
   
   + Examples样本：特定的数据实例，分为有标签样本 labeled examples 和无标签样本 unlabeled examples 。
   
     ```python
     labeled examples: {features, label}: (x,y)
     ```
   
     使用样本训练模型。在垃圾邮件检测系统中，有标签的样例就是那些被用户明确标记了是"垃圾邮件"or"非垃圾邮件"的分类邮件集合。
   
     无标签的样本只包含features特征，而没有label标签。
   
     ```python
     unlabeled examples: {features, ?}: (x, ?)
     ```
   
     一旦我们用有标签的样本label examples训练了模型，我们就可以用这个模型去预测无标签样本unlabeled examples的标签label。在前面提到的垃圾邮件检测系统中，无标签样本unlabeled examples就是人们还没有标记过的邮件(标记是"垃圾邮件"，还是"非垃圾邮件")。
   
   + Models模型：模型定义了特征features和标签labels之间的关系。比如，一个垃圾邮件检测模型可能把某些特征值数据和“垃圾邮件”紧密联系到一起。下面学习一下模型model生命周期中的2个阶段：
     + **Training** 训练，意味着创造或者**learning**学习模型model。也就是你向模型model展示有标签的样本labeled examples使其能够逐渐学习到给定特征值features和标签labels的关系(联系)。
     + **Inference**推断，把训练过的模型trained model应用到没有标签的样本unlabeled examples。即使用训练过的模型trained model去做有意义的预测(y')。打比方，推断中，你可以预测新的没标签样本的平均房屋价格。
   
   + Regression vs. classification
   
     一个回归模型**regression** model预测的是连续的值。比如，回归模型做出的预测可以回答如下问题：
   
     + XX地区的房屋价格
     + 用户可能点击广告的概率
   
   + 一个分类模型**classification** model预测离散值(离散的值)。比如，分类模型做出的预测可以回答如下问题：
   
     + 给定的邮件是否“垃圾邮件”
     + 图片描述的是狗、猫还是仓鼠

### Descending into ML

1. 学习其他模型前，先学习最简单的线性回归模型。
2. Linear Regression

3. Training and Loss

   **Training**训练一个模型意味着学习有标签样本的所有权重和偏差。有监督的学习，机器学习算法会通过检验大量样本构建一个模型，且试图找到一个损失最小的模型。这一过程称为**empirical risk minization**经验风险最小化

   损失是对一个差劲的预测的惩罚。即，**loss**是一个表示模型对简单样本的预测的糟糕程度的数值。如果模型预测没问题，那么损失loss为0；反之，损失loss会很大。 训练模型的目标是从所有样本中找到一组平均损失“较小”的权重和偏差。

   + **Suqared loss**(平方损失、方差)：a popular loss function

     ```python
       = the square of the difference between the label and the prediction
       = (observation - prediction(x))2
       = (y - y')2
     ```

   +  **Mean square error** (**MSE**)  : 指的是每个样本的平均平方损失。要计算 MSE，请求出各个样本的所有平方损失之和，然后除以样本数量。

### Reducing Loss

1. Video Lecture

   了解梯度下降法；了解随机梯度下降法；小批量梯度下降法；

   可以在每步上计算整个数据集的梯度，但事实证明没有必要这么做。

   计算小型数据样本的梯度很好。

   随机梯度下降法：一次抽取一个样本

   小批量梯度下降法：每批包括10-1000个样本，损失和梯度在整批范围内达到平衡

2. An Iterative Approach

   机器学习系统根据输入的标签评估所有特征， 为损失函数生成一个新值，而该值又产生新的参数值。这种学习过程会持续迭代，直到该算法发现损失可能最低的模型参数。通常，您可以不断迭代，直到总体损失不再变化或至少变化极其缓慢为止。这时候，我们可以说该模型已**收敛** **converged** 。 

3. Gradient Descent

   Key Terms：

   + gradient descent 梯度下降法
   + step 步

4. Learning Rate

   Key Terms：

   + hyperparameter 超参数
   + learning rate 学习速率
   + step size 步长

5. Optimizing Learning Rate

6. Stochastic Gradient Descent

   Key Terms:

   + batch 批量
   + batch size 批量大小
   + mini-batch 小批量
   + stochastic gradient descent(SGD) 随机梯度下降法

### First Steps with TensorFlow

1. Toolkit

   Key Terms:

   + Estimators
   + graph 图
   + tensor 张量

2. Programming Exercises
   + Common hyperparameters in Machine Learning Crash Course exercises
     + steps
     + batch size

### Generalization

1. Video Lecture

2. Peril of Overfitting

   过拟合 **overfits** 在训练的过程产生的损失较低，但是在预测新数据的表现差。

   + **training set** -- 用于训练模型的子集
   + **test set** -- 用于测试模型的子集

    一般来说，在测试集上表现是否良好是衡量能否在新数据上表现良好的有用指标，前提是 :

   + The test set is large enough.
   + You don't cheat by using the same test set over and over.

3. The ML fine print

   以下三项基本假设阐明了泛化：

   + 我们从分布中随机抽取独立且同分布**independently and identically(i.i.d)**的样本。换言之，样本间互不影响
   + 分布是平稳**stationary**的，即在数据集中的分布不会变化
   + 我们从同一分布**same distribution**中的数据划分中抽取样本

   然而，事件中，我们偶尔会违背这些假设，例如：

   + 想象有一个选择要展示的广告的模型。如果该模型在某种程度上根据用户以前看过的广告选择广告，则会违背 i.i.d. 假设。
   + 想象有一个包含一年零售信息的数据集。用户的购买行为会出现季节性变化，这会违反平稳性。

    如果违背了上述三项基本假设中的任何一项，那么我们就必须密切注意指标 

### Training and Test Sets

1. Video Lecture

   将数据集划分成训练集和测试集，训练集用于训练模型，而测试集就是用来测试模型的。

   典型陷阱：请勿对测试数据进行训练

2. Splitting Lecture

   + **training set** -- a subset to train a model
   + **test set** -- a subset to test the trained model

   保证划分后的测试集满足以下条件：

   + 规模足够大，能够产生有统计学意义的结果
   + 能够代表整个数据集。换言之，具有和训练集一样的特征。

   **切记不要对测试数据进行训练**。如果评估取得意外的好结果，有可能是不小心对测试集进行了训练。高准确率可能表明测试数据泄露到了训练集。

### Validation Set

1. Check Your Intuition

2. Video Lecture

   传统地使用训练集训练模型后直接就使用测试集评估模型(之后根据测试集的效果调整模型，在测试集上选择最佳的模型)，可能导致对测试集过拟合。

   为了避免上述问题，中间添加一个验证集(替代原本测试集的位置），然后选出在验证集上最佳效果的模型再使用测试集测试效果，要是效果不好，那可能前面是对验证集进行了过拟合。

3. Another Partition

   大意就是在训练集和测试集之间插入一个验证集，避免测试集过拟合，导致模型在新数据的预测中泛用性(效果)差。

   此时数据集划分为3部分：training set、validation set、tes tset；

   训练集训练模型后，验证集验证模型的准确度，最后才挑选通过验证最好的模型进行测试，要是测试效果差，说明可能对验证集过拟合。这种双重检验能够保证获取更准确的模型。

4. Programming Exercise

   这里开始蒙蔽了，先歇一歇。换个更基础一点的教程

