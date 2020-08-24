#  Intro to Machine Learning 

> 之前google的machine-learning课程到了中间有点懵了，就干脆先换一个看看，这个如果比较基础就先着手这个了。
>
>  https://www.kaggle.com/learn/intro-to-machine-learning 

## How Models Work

### Introduction

决策树是机器学习底层部件之一。数据科学中，我们通过划分数据集为**training set**训练集和**test set**测试集，用训练集训练模型，再用测试集测试模型，直到筛选出准确率在我们预期之内的模型。然后使用模型就可以进行诸如有监督学习的分类、回归分析，无监督学习的数据降维、聚类等操作。我自己扯得有点偏了。这里介绍决策树最底部的叶子节点**leaf**即我们预测的结果。

## Basic Data Exploration

### Using Pandas to Get Familiar With Your Data

使用pandas的DataFrame导入数据。这个比较基础，网上大概查查就懂了

### Interpreting Data Description

进行数据分析前，先自己了解下自己数据集的特点、特征等。

+ dataframe.describe()输出基本统计量

  >count 数量
  >
  >mean 均值
  >
  >std 标准差
  >
  >min 最小值
  >
  >25% 下四分位
  >
  >50% 中位数
  >
  >75% 上四分位
  >
  >max 最大值

### Exercise：Explore Your Data

就很基础地使用pandas.read_csv()读取文件，然后调用dataframe.describe()输出基本统计量，再从要求中找出(计算)指定值。

## Your First Machine Learning Model

### Select Data for Modeling

获取dataFrame的一列**Series**，其中作为预测目标**prediction target**的一列，俗称**y**

### Choose "Features"

输入模型中且后来用于预测数据的列，叫做“features”。有时候，我们需要使用除了target的所有列作为features。但是，有时更少的features会更好。

通常，features所包含的所有列数据被称为**X**

我们可以使用dataframe.describe()方法来查看数据集的基础统计量；使用dataframe.head()方法获取数据集前几行的数据。

使用这些简单的指令检查数据也是数据科学家工作的重要一环，没准就能从数据集中找到惊喜(比如某些数据和某些数据线性相关之类的吧)。

### Building Your Model

使用**scikit-learn**包创建模型models，代码中，其被写做**sklearn**。

创建模型model的步骤如下：

+ **Define**：选择使用的模型类别。
+ **Fit**：从提供的数据中捕获特征(训练模型)，这是创建模型的核心步骤。
+ **Predict**：用模型(对新数据)进行预测
+ **Evaluate**：评估模型的预测准确度。

使用sklearn包内的类创建模型时，指定random_state可以设置一个随机数，让模型随机从训练集中取值（当我们手动指定值时，模型每次运行会得到相同的值。我感觉意思就是模型默认每次训练使用一个随机数算法之类的选取features中某些数据进行训练，如果我们指定这个random_state,那么每次训练都用相同的features值训练。）。不过，random_state值的设定，并不会影响模型质量。

```python
from sklearn.tree import DecisionTreeRegressor

# Define model. Specify a number for random_state to ensure same results each run
melbourne_model = DecisionTreeRegressor(random_state=1)

# Fit model
melbourne_model.fit(X, y)
```

获取训练后的模型fitted model后，我们就可以用它来进行数据的预测了。

```python
print("Making predictions for the following 5 houses:")
print(X.head())
print("The predictions are")
print(melbourne_model.predict(X.head()))
```

### Exercise:Your First Machine Learning Model

1. 读取数据，选定y(需要预测的数据列)
2. 创建X(features，特征集)，在创建model之前，最好再检查一下X，确保其可靠性
3. 选定需要的模型并创建，之后对模型进行训练(拟合)**Fit**，可根据需要选择给model指定一个random_state，使模型具有再生性。
4. 使用模型对指定的数据进行预测