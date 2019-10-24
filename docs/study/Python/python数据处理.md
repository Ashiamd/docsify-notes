# python数据处理

> 笔记所记录的内容大部分出至《数据科学技术与应用》,还有些是我学习中遇到的问题，穿插其中补充。

## 第1章 数据科学基础

### 1.3 python语言基础

#### 1.3.1 常用数据类型

1. 字符串(string)

   **任何类型都可以使用内置函数str()转换为字符串**

2. 元组tuple和列表list(两者是有序元素序列)

   + tuple()的元素不可修改，list[]可以修改

   + tuple的元素使用“变量名[索引]”来表示，索引范围由[0,n-10] or [-n,-1]

   + tuple和list的相互转换

     ```python
     #tuple转list
     list(tuple对象)
     #list转tuple
     tuple(list对象)
     ```

     

3. 字典dictionary(无序键值对)

   + dictionary{xx:yy}

#### 1.3.2 流程控制

1. 注释

   + 单行注释# 被注释的代码
   + 多行注释 """  被多行注释的代码  """

2. 分支结构(if-else)

   ```python
   if XXX:
   
   	代码1
   
   else if YYY:
   
   	代码2
   
   else:
   
   	代码3
   ```

3. 循环结构(while,for)

   + ```python
     for xx in yy :
     
     	代码快
     ```

   + ```python
     while xxxx :
     
     	代码块
     ```

#### 1.3.3 函数和方法库

1. Python提供大量内置函数，如input,range等，大部分第三方库library,包package没加载到解释器中，需导入

   + 直接导入整个方法库或者包，调用时加上包名

     ```python
     >>> import math
     >>> math.sqrt(5)
     ```

   + 导入方法库中某个函数，调用时直接使用函数名

     ```python
     >>> from math import sqrt
     >>> sqrt(5)
     ```

   + 导入方法库中某个类或者函数并重命名，调用时使用临时替代名

     ```python
     >>> from math import sqrt as sq
     >>> sq(5)
     ```

2. Python自定义函数

   使用def关键字定义函数，函数定义时，变量类型无需说明，同时<u>可以在参数列表的最后定义多个带有默认值的参数</u>。函数调用时，具有默认值的形参，可以不传实参。

   ```python
   >>> def sum(a,b):
       print(a + b)
   >>> sum(1,2)
   3
   ```

3. 常用的方法：

   + type(对象)

     用于得到和判断指定对象的数据类型

## 第2章 多维数据结构与运算

### 2.1 多维数组对象

> NumPy库提供多维数组ndarray，其**所有元素类型必须相同**，且**大小固定**，在创建时定义，**使用过程中不可改变**。一般采用如下方式导入NumPy库：
>
> \>\>\> import numpy as np
>
> 为避免函数命名冲突，后续使用Numpy时使用np代替。

#### 2.1.1 一维数组对象

> Numpy库的array函数可以基于python的tuple和list创建ndarray对象。(亲测使用dictionary生成的对象0维，而tuple和list创建对象没有区别

1. 查看数组的属性

   ```pywhon
   np.array(xxx).ndim	#数组维度
   np.array(xxx).size	#数组元素的个数
   np.array(xxx).ndim	#数组的数据类型
   ```

2. 单个数组元素访问

   也是通过索引序号访问，一维数组的索引序号范围[0,n-1]和[-n,-1]

3. 数组切片(slicing)

   抽取数组的一部分元素生成新数组称为切片操作。切片根据给出的索引，抽取对应的元素

   ```python
   #切片方式1：传入索引列表[x,y,z,m]
   arr2 = np.array(["零","一","二","三","四","五","六","七","八"])
   print (arr2[[0,3,5]]) #得到 "['零' '三' '五']"
   
   """
   切片方式2：索引通过 start:end:step 形式给出，它生成一个等差数列，元素从start开始，end-1结束，step为步长，start默认为从头开始，end默认为最后一个元素结束，step默认步长为1
   """
   print(arr2[:-1:3]) #得到"['零' '三' '六']"
   ```

   + 使用索引列表进行切片操作时，外层的方括号表示数组索引操作，内层的方括号表示多个索引组成的列表

4. 根据条件筛选数组元素

   > Python 中 （&，|）和（and，or）之间的区别
   >
   >  https://blog.csdn.net/weixin_40041218/article/details/80868521 

   ```python
   arr2 = np.array(["零","一","二","三","四","五","六","七","八"])
   print(arr2[(arr2 == '零') | (arr2 == '三')])
   #得到结果 ['零' '三']
   
   print(arr2[(arr2 == '零') or (arr2 == '三')])
   #用or报错，这里就不贴报错信息了
   ```

   条件筛选分为2步骤，首先利用(arr2 == '零') | (arr2 == '三')条件表达式<u>创建一个布尔型的数组</u>，然后使用此对象对arr2内的元素按位置选择，值是True的选中，Flase的不选。分步骤实现代码如下：

   ```python
   arr2 = np.array(["零","一","二","三","四","五","六","七","八"])
   selector = (arr2 == '零') | (arr2 == '三')
   print(arr2[selector])
   #得到结果 ['零' '三']
   print(selector)
   # [ True False False  True False False False False False]
   ```

#### 2.1.2 二维数组对象

1. 查看数组属性

   ```python
   arr = np.array([[1,2],[3,4]])
   print(arr.ndim) # 数组维度 2
   print(arr.size) # 数组元素总数 = 行数*列数 4
   print(arr.shape) # 数组的行数和列数 (2,2)
   print(arr.dtype) # 数组元素的类型 int32
   ```

2. 二维数组切片

   二维数组切片的基本格式：

   ```python
   arr[ row , column ]
   """
   其中row为行序号，column为列序号，中间用“，”隔开。行、列切片的表示方式与一维数组相同。如果行或列用“：”代替，表示选中对应的所有行或列
   """
   ```

   1）访问指定行、列的元素，给定行和列两个索引值

   ```python
   arr = np.array([[1,2],[3,4,5,6],[7,8,9]])
   print(arr)
   # [list([1, 2]) list([3, 4, 5, 6]) list([7, 8, 9])]
   print(arr[2][2])
   # 9
   # 上面这是一维的，不过元素是list对象,访问arr[0,1]报错,不贴error信息
   arr = np.array([[1,2,3,4],[5,6,7,8],[9,10,11,12]])
   print(arr)
   """
   [[ 1  2  3  4]
    [ 5  6  7  8]
    [ 9 10 11 12]]
   """
   print(arr[2,3])
   # 12
   print(arr[[0,1],[1,2],[2,3]])
   # IndexError: too many indices for array
   a = arr[[0,2],[1,3]]
   print(type(a))
   # <class 'numpy.ndarray'>
   print(arr[[0,2],[1,3]])
   # [ 2 12]
   ```

   + 上例在方括号中给出行切片[0,2]和列切片[1,3]，表示抽取行序号为0、列序号为1，以及行序号为2、列序号为3的元素，**得到一维的ndarray数组**

   2）访问部分行元素，给出行列表即可，**列索引的":"可以省略**

   ```python
   print(arr[[0,1]])
   """
   [[1 2 3 4]
    [5 6 7 8]]
   """
   print(arr[[0,1],:])
   """
   [[1 2 3 4]
    [5 6 7 8]]
   """
   ```

   3）访问部分列元素，**前面的行索引“:"不能省略**，否则无法识别是对列切片

   ```python
   print(arr[,[0,1]])
   # SyntaxError: invalid syntax
   print(arr[:,[0,1]])
   """
   [[ 1  2]
    [ 5  6]
    [ 9 10]]
   """
   print(arr[0:2:1,[0,1]])
   # [[1 2]]
   print(arr[0:2:1,[0,1]])
   """
   [[1 2]
    [5 6]]
   """
   print(arr[0:3:1,[0,1]])
   """
   [[ 1  2]
    [ 5  6]
    [ 9 10]]
   """
   ```

   + 后面几条尝试了前面提到的start: end: step形式，可以看出end是不包含的，也就是区间是[start,end)​

   4）**访问部分行和列数据**

   ```python
   arr = np.array([[1,2,3,4],[5,6,7,8],[9,10,11,12]])
   print(arr[ [0,2] , 1:3 ]) # 0、2行中1~2列的所有元素
   """
   [[ 2  3]
    [10 11]]
   """
   ```

   如果需要抽取指定**某些行中指定列**的所有元素，则需要进行**两层切片**

   ```python
   print(arr[0,1][:,[0,3]])
   # IndexError: invalid index to scalar variable.
   print(arr[[0,1]][:,[0,3]])
   """
   [[1 4]
    [5 8]]
   """
   #首先arr[[0,1]]抽取指定0、1行组成的二维ndarray对象，再在此对象上进行切片操作，取所有行中0、3列的元素
   ```

3. 条件筛选

   可以使用布尔型数组筛选访问其他数组的元素。用于筛选的布尔数组，<u>需要具有与访问数组相同的函数或列数</u>。

   ```python
   flag = np.array(['one','two','three'])
   arr = np.array([[1,2,3,4],[5,6,7,8],[9,10,11,12]])
   selector = (flag == 'one') | (flag == 'three')
   print(selector)
   print(arr[selector])
   """
   [ True False  True]
   [[ 1  2  3  4]
    [ 9 10 11 12]]
   """
   flag = np.array(['one','two','three','four'])
   arr = np.array([[1,2,3,4],[5,6,7,8],[9,10,11,12]])
   selector = (flag == 'one') | (flag == 'three')
   print(selector)
   print(arr[:,selector])
   """
   [ True False  True False]
   [[ 1  3]
    [ 5  7]
    [ 9 11]]
   """
   ```

#### 2.1.3 创建多维数组的常用方法

1. arange()函数

   生成指定头、尾、步长的<u>等差数组</u>，**参数可以是浮点数**

   ```python
   np.arange(4)
   # array([0, 1, 2, 3])
   np.arange(0,4)
   # array([0, 1, 2, 3])
   np.arange(0,5,2)
   # array([0, 2, 4])
   np.arange(0.3,2.2,0.3)
   # array([0.3, 0.6, 0.9, 1.2, 1.5, 1.8, 2.1])
   ```

2. reshape()函数

   将一维数组转换为指定的多维数组

   ```python
   # arr.reshape(m,n) 获取arr转换为m行n列的新数组
   arr1 = np.array([1,2,3,4,5,6,7,8,9])
   arr2 = arr1.reshape(3,3)
   print(arr2)
   """
   [[1 2 3]
    [4 5 6]
    [7 8 9]]
   """
   ```

3. zeros()函数和ones()函数

   zeros()函数和ones()函数生成指定大小全0和全1的数组。

   ```python
   print(np.zeros(5))
   print(np.ones(4))
   """
   [0. 0. 0. 0. 0.]
   [1. 1. 1. 1.]
   """
   print(np.zeros((3,4)))
   print(np.ones((4,3)))
   """
   [[0. 0. 0. 0.]
    [0. 0. 0. 0.]
    [0. 0. 0. 0.]]
   [[1. 1. 1.]
    [1. 1. 1.]
    [1. 1. 1.]
    [1. 1. 1.]]
   """
   ```

### 2.2 多维数组运算

#### 2.2.1 基本算数运算

1. 二维数组与标量运算

   ```python
   arr = np.zeros((2,4))
   arr += 5
   print(arr)
   """
   [[5. 5. 5. 5.]
    [5. 5. 5. 5.]]
   """
   arr *= 2
   print(arr)
   """
   [[10. 10. 10. 10.]
    [10. 10. 10. 10.]]
   """
   ```

2. 二维数组与一维数组运算

   ```python
   arr = np.zeros((2,4))
   arr += np.array([1,2,3,4])
   print(arr)
   """
   [[1. 2. 3. 4.]
    [1. 2. 3. 4.]]
   """
   ```

3. 选定元素运算

   ```python
   flag = np.array([3,4,2,5])
   arr = np.zeros((4,6))
   arr[flag > 3] += np.array([1,2,3,4,5,6])
   print(flag>3)
   print(arr)
   """
   [False  True False  True]
   [[0. 0. 0. 0. 0. 0.]
    [1. 2. 3. 4. 5. 6.]
    [0. 0. 0. 0. 0. 0.]
    [1. 2. 3. 4. 5. 6.]]
   """
   ```

#### 2.2.2 函数和矩阵运算

1. 通用函数(ufunc)

   + 常用的一元函数(eg: np.abs(arr)等 )

   <escape>

   <table class="tg">
     <tr>
       <th class="tg-c3ow">函数</th>
       <th class="tg-0pky">描述</th>
     </tr>
     <tr>
       <td class="tg-c3ow">abs、fabs</td>
       <td class="tg-0pky">计算整数、浮点数或负数的绝对值</td>
     </tr>
     <tr>
       <td class="tg-0pky">sqrt</td>
       <td class="tg-0pky">计算各元素的平方根</td>
     </tr>
     <tr>
       <td class="tg-0pky">square</td>
       <td class="tg-0pky">计算各元素的平方</td>
     </tr>
     <tr>
       <td class="tg-0pky">exp</td>
       <td class="tg-0pky">计算各元素的指数</td>
     </tr>
     <tr>
       <td class="tg-0pky">log、log10</td>
       <td class="tg-0pky">自然对数、底数为10的log</td>
     </tr>
     <tr>
       <td class="tg-0pky">sign</td>
       <td class="tg-0pky">计算各元素的正负号</td>
     </tr>
     <tr>
       <td class="tg-0pky">ceil</td>
       <td class="tg-0pky">计算各元素的ceiling值，即大于或等于该值的最小整数</td>
     </tr>
     <tr>
       <td class="tg-0pky">floor</td>
       <td class="tg-0pky">计算各元素的floor值，即小于或等于该值的最大整数</td>
     </tr>
     <tr>
       <td class="tg-0pky">cos、cosh、sin、sinh、tan、tanh</td>
       <td class="tg-0pky">普通和双曲型三角函数</td>
     </tr>
   </table>

   </escape>

   + 常用的二元函数

   <escape>

   <table class="tg">
     <tr>
       <th class="tg-c3ow">函数</th>
       <th class="tg-0pky">描述</th>
     </tr>
     <tr>
       <td class="tg-c3ow">add</td>
       <td class="tg-0pky">将数据中对应的元素相加</td>
     </tr>
     <tr>
       <td class="tg-0pky">subtract</td>
       <td class="tg-0pky">从第一个数组中践去第二个数组中的元素</td>
     </tr>
     <tr>
       <td class="tg-0pky">multiply</td>
       <td class="tg-0pky">数组元素相乘</td>
     </tr>
     <tr>
       <td class="tg-0pky">divide</td>
       <td class="tg-0pky">数组对应元素相除</td>
     </tr>
     <tr>
       <td class="tg-0pky">power</td>
       <td class="tg-0pky">对第一个数组中的元素A，根据第二个数组中的相应元素B，计算AB</td>
     </tr>
     <tr>
       <td class="tg-0lax">mod</td>
       <td class="tg-0lax">元素级的求模运算</td>
     </tr>
     <tr>
       <td class="tg-0lax">copysign</td>
       <td class="tg-0lax">将第二个数组中的值的符号复制给第一个数组中的值</td>
     </tr>
     <tr>
       <td class="tg-0lax">equal,not_equal</td>
       <td class="tg-0lax">执行元素级的比较运算，产生布尔型数组</td>
     </tr>
   </table>

   </escape>

2. 聚焦函数
   + 常用的聚集函数

   <escape>
   
   <table class="tg">
     <tr>
       <th class="tg-c3ow">函数</th>
       <th class="tg-0pky">描述</th>
     </tr>
     <tr>
       <td class="tg-c3ow">sum</td>
       <td class="tg-0pky">求和</td>
     </tr>
     <tr>
       <td class="tg-0pky">mean</td>
       <td class="tg-0pky">算数平均值</td>
     </tr>
     <tr>
       <td class="tg-0pky">min、max</td>
       <td class="tg-0pky">最小值和最大值</td>
     </tr>
     <tr>
       <td class="tg-0pky">argmin、argmax</td>
       <td class="tg-0pky">最小值和最大值的索引</td>
     </tr>
     <tr>
       <td class="tg-0pky">cumsum</td>
       <td class="tg-0pky">从0开始向前累加各元素</td>
     </tr>
     <tr>
       <td class="tg-0pky">cumprod</td>
       <td class="tg-0pky">从1开始向前累乘各元素</td>
     </tr>
   </table>
   
   </escape>
   
   + 聚焦函数的使用样例(下面数据看看就好，主要了解参数)
   
   ```python
   #导入numpy
   import numpy as np
   
   #创建一维数组
   names = np.array(['王微', '肖良英', '方绮雯', '刘旭阳','钱易铭'])
   subjects = np.array(['Math', 'English', 'Python', 'Chinese','Art', 'Database', 'Physics'])
   #创建二维数组
   scores = np.array([[70,85,77,90,82,84,89],[60,64,80,75,80,92,90],[90,93,88,87,86,90,91],[80,82,91,88,83,86,80],[88,72,78,90,91,73,80]])
   
   # 1)统计不同科目的成绩总分
   # 首先利用布尔型数组选择“王微”的所有成绩，然后使用求平均值函数mean()
   scores.sum(axis = 0) # 按列求和
   # array([388, 396, 414, 430, 422, 425, 430])
   
   # 2)求“王微”所有课程成绩的平均分
   scores[names == '王微'].mean()
   # 82.42857142857143
   
   # 3)查询英语考试成绩最高的同学的姓名
   names[ scores[:,subjects == 'English'].argmax() ]
   # argmax()函数能返回特定元素的下标。首先通过列筛选得到由所有学生英语成绩组成的一维数组，接着通过argmax()函数返回一维数组中最高分的索引值，最后利用该索引值在names数组中查找到该学生的姓名
   ```
   
   > 对于二维数组对象，可以指定聚集函数是在行上操作还是在列上操作。**当参数axis为0时，函数操作的对象是同一列不同行的数组元素；当参数axis为1时，函数操作的对象是同一行不同列的数组元素**。

#### 2.2.3 随机数组生成函数

+ 常用函数

  <escape>
  
  <table class="tg">
    <tr>
      <th class="tg-c3ow">函数</th>
      <th class="tg-0pky">描述</th>
    </tr>
  <tr>
      <td class="tg-c3ow">random</td>
      <td class="tg-0pky">随机产生[0,1)之间的浮点值</td>
    </tr>
    <tr>
      <td class="tg-0pky">randint</td>
      <td class="tg-0pky">随机生成给定范围的一组整数</td>
    </tr>
    <tr>
      <td class="tg-0pky">uniform</td>
      <td class="tg-0pky">随机生成给定范围内服从均匀分布的一组浮点数</td>
    </tr>
    <tr>
      <td class="tg-0pky">choice</td>
      <td class="tg-0pky">在给定的序列内随机选择元素</td>
    </tr>
    <tr>
      <td class="tg-0pky">normal</td>
      <td class="tg-0pky">随机生成一组服从给定均值和方差的正态分布随机数</td>
    </tr>
  </table>
  
  </escape>
  
  ```python
  # 生成5*6的二维随机整数，随机数的取值是0或1
  np.random.randint(0,2,size = (5,6))
  """
  array([[0, 0, 0, 0, 1, 0],
         [1, 0, 1, 0, 0, 0],
         [1, 1, 1, 1, 1, 0],
         [1, 0, 1, 1, 0, 0],
         [0, 0, 1, 1, 0, 0]])
  """
  
  # 生成均值为0、方差为1服从正太分布的4*5二维数组
  np.random.normal(0,1,size = (4,5))
  """
  array([[ 0.27348588, -1.06847557, -0.84807463, -0.82102134,  0.31033654],
         [-0.76208868,  0.9522973 ,  0.53462609, -0.07880294,  1.08901491],
         [-1.45998785, -1.72776158, -1.09198982, -0.91469086, -1.2952753 ],
         [ 0.56222164, -1.13944672, -0.64216053, -0.03491689,  0.44231984]])
  """
  ```

## 第3章 数据汇总与统计

### 3.2 pandas数据结构

#### 3.2.1 Series对象

​	Series是类似于数组的一维数据结构，由<u>两个</u>相关联的数组组成。名为“values”的<u>值数组</u>存放数据(任意类型的数据)，每一个元素都有一个与之关联的标签，存储在名为“index”的<u>索引数组</u>中。

```python
# 使用pandas的Series()函数来创建Series对象变量，格式如下：
Series([data,index,...])
"""
其中，data可以是列表或者Numpy的一维ndarray对象;index是列表，如果省略，则创建Series对象时自动生成0~n-1的序号标签，n为data元素个数
"""

import pandas as pd
values = [175,168,186,190,166,172,178,182,181]
indexs = [1,2,3,4,5,6,7,8,9]
series = pd.Series(values,index=indexs)
print(series)

"""
1    175
2    168
3    186
4    190
5    166
6    172
7    178
8    182
9    181
dtype: int64
"""

"""
Series对象和字典类型类似，可以将index和values数组序列中序号相同的一对元素视为字典的键-值对。用字典创建Series对象，将字典的key作为索引
"""
print(pd.Series({1:123,2:1234,3:12345,4:123456}))
"""
1       123
2      1234
3     12345
4    123456
dtype: int64
"""
```

#### 3.2.2 Series数据访问

​	Series数据访问方式类似于ndarray数组，可以通过值得位置序号获取，同时由于每个值都关联到索引标签，也可以通过索引来访问。

​	下表列举了Series数据常用的几种选取方法

<escape>

<table>
    <tr>
        <th>选取类型</th>
        <th>选取方法</th>
        <th>说明</th>
    </tr>
    <tr>
        <td rowspan="2">基于索引名选取</td>
        <td >obj[index]</td>
        <td>选取某个值</td>
    </tr>
    <tr>
        <td >obj[indexList]</td>
        <td>选取多个值</td>
    </tr>
    <tr>
        <td rowspan="3">基于位置选取</td>
        <td >obj[loc]</td>
        <td>选取某个值</td>
    </tr>
    <tr>
        <td >obj[locList]</td>
        <td>选取多个值</td>
    </tr>
    <tr>
        <td >obj[a:b,c]</td>
        <td>选取位置a~(b-1)以及c的值</td>
    </tr>
    <tr>
    	<td>条件筛选</td>
        <td>obj[condition]</td>
        <td>选取满足条件表达式的值</td>
    </tr>
</table>

</escape>

+ 查询

  ```python
  series = pd.Series({1:123,2:1234,3:12345,4:123456})
  print(series[2])
  print(series[[2,3]])
  print(series[0:2])
  """
  1234
  2     1234
  3    12345
  dtype: int64
  1     123
  2    1234
  dtype: int64
  """
  ```

+ 修改

  ```python
  series = pd.Series({1:123,2:1234,3:12345,4:123456})
  series[3] = 33333
  series[0:2] = 987654321
  print(series)
  """
  1    987654321
  2    987654321
  3        33333
  4       123456
  dtype: int64
  """
  ```

+ 添加 series.append(series),原Series不变

  ```python
  series = pd.Series({1:123,2:1234,3:12345,4:123456})
  new1 = series.append(pd.Series({5:67}))
  print(new1)
  """
  1       123
  2      1234
  3     12345
  4    123456
  5        67
  dtype: int64
  """
  ```

+ 删除 series.drop(索引值、索引值集合)

  Series的drop()不能删除原始对象的数据

  ```python
  series = pd.Series({1:123,2:1234,3:12345,4:123456})
  new2 = series.drop([1,2])
  print(series)
  print(new2)
  """
  1       123
  2      1234
  3     12345
  4    123456
  dtype: int64
  3     12345
  4    123456
  dtype: int64
  """
  ```

+ <u>Series对象创建后，可以修改值，也可以修改索引，用新的列表替换即可</u>。

  **如果Series对象的index本身为数字类型，基于位置序号的访问需要使用iloc方式实现。**

  ```python
  series = pd.Series({1:123,2:1234,3:12345,4:123456})
  series.index = [-1,0,1,2]
  print(series)
  """
  -1       123
   0      1234
   1     12345
   2    123456
  dtype: int64
  """
  print(series[[-1,0]])
  """
  -1     123
   0    1234
  dtype: int64
  """
  print(series.iloc[[0,1]])
  """
  -1     123
   0    1234
  dtype: int64
  """
  ```

#### 3.2.3 DataFrame对象

​	DataFrame类似于表格的二维数据结构，包括值(values)、行索引(index)、列索引(columns)3部分。值由ndarray的二维数组对象构成，行、列索引则保存为ndarray一维数组。**DataFrame对象的任意一行数据或一列数据都可视为Series对象**。通常DataFrame对象中每列表示一个总体的所有样本，每行为某个体在各个总体中的值。

```python
# 创建DataFrame方法如下：
DataFrame(data,index=[...],columns=[...])
"""
data可以是列表或者NumPy的二维ndarray对象；
index是行索引列表，columns是列索引列表，
如果省略，创建时会使用位置序号作为索引标签
"""

data = [[19,170,68],[20,165,65],[18,175,65]]
students = pd.DataFrame(data, index=[1,2,3], columns=['age','height','weight'])  
print(students)
"""
   age  height  weight
1   19     170      68
2   20     165      65
3   18     175      65
"""
```

#### 3.2.4 DataFrame数据访问

​	DataFrame数据访问方式类似ndarray数组，可以通过值得位置序号获取，同时由于行、列都关联到索引标签，也可以通过索引来访问。

+ DataFrame数据选取方法

<escape>

<table>
  <tr>
    <th>选取类型</th>
    <th>选取方法</th>
    <th>说明</th>
  </tr>
  <tr>
    <td rowspan="4">基于索引名选取</td>
    <td>obj[col]</td>
    <td>选取某列</td>
  </tr>
  <tr>
    <td>obj[colList]</td>
    <td>选取某几列</td>
  </tr>
  <tr>
    <td>obj.loc[index,col]</td>
    <td>选取某行某列</td>
  </tr>
  <tr>
    <td>obj[indexList,colList]</td>
    <td>选取多行多列</td>
  </tr>
  <tr>
    <td rowspan="3">基于位置序号选取</td>
    <td>obj.iloc[iloc,cloc]</td>
    <td>选取某行某列</td>
  </tr>
  <tr>
    <td>obj.iloc[ilocList,clocList]</td>
    <td>选取多行多列</td>
  </tr>
  <tr>
    <td>obj.iloc[a:b,c:d]</td>
    <td>选取a~(b-1)行,c~(d-1)列</td>
  </tr>
  <tr>
    <td rowspan="2">条件筛选</td>
    <td>obj.loc[condition,colList]</td>
    <td>使用索引构造条件表达式<br>选取满足条件的行</td>
  </tr>
  <tr>
    <td>obj.iloc[condition,colcList]</td>
    <td>使用位置序号构造条件表达式<br>选取满足条件的行</td>
  </tr>
</table>

</escape>

+ **如果行或列部分用“:”代替，则表示选中整行或整列**。

+ 查询

  ```python
  data = [[19,170,68],[20,165,65],[18,175,65]]
  students = pd.DataFrame(data, index=[1,2,3], columns=['age','height','weight'])  
  print(students)
  """
     age  height  weight
  1   19     170      68
  2   20     165      65
  3   18     175      65
  """
  
  # 下面只贴部分结果，需要可以自行运行
  students.loc[1] # 获取行索引为1的数据
  """
  age        19
  height    170
  weight     68
  Name: 1, dtype: int64
  """
  students.loc[1] # 获取行位置序号为1的数据
  """
  age        20
  height    165
  weight     65
  Name: 2, dtype: int64
  """
  students.loc[[1,3],['height','weight']] 
  # 查询行索引1、3且列索引为'height'、'weight'的数据
  """
  	height	weight
  1	170	68
  3	175	65
  """
  students.iloc[[0,2],[0,1]]
  # 查询行位置序号为0、2且列位置序号为0、1的数据
  
  students.loc[:,'height']
  students.loc[:,['height','weight']]
  students['height','weight'] # KeyError: ('height', 'weight')
  students[['height','weight']]
  
  students.iloc[1:,0:2]
  """
     age  height
  2   20     165
  3   18     175
  """
  
  students.iloc[1:]
  """
  age	height	weight
  2	20	165	65
  3	18	175	65
  """
  
  students[1:3] # 行位置序号1~行位置序号2，列的":"可以省略
  """
  	age	height	weight
  2	20	165	65
  3	18	175	65
  """
  
  mask = students['height'] >= 170
  print(mask)
  print(students.loc[mask,'weight'])
  """
  1     True
  2    False
  3     True
  Name: height, dtype: bool
  1    68
  3    65
  Name: weight, dtype: int64
  """
  ```

+ 添加

  ​	DataFrame对象可以添加新的列，但不能直接增加新的行。增加新的行需要通过2个DataFrame对象的合并实现。当新增的列索引标签不存在时，增加新列；**若存在增修改列值**。

  ```python
  students['freetime'] = [4,5,8]
  print(students)
  """
     age  height  weight  freetime
  1   19     170      68         4
  2   20     165      65         5
  3   18     175      65         8
  """
  ```

+ 修改

  ```python
  data = [[19,170,68],[20,165,65],[18,175,65]]
  students = pd.DataFrame(data, index=[1,2,3], columns=['age','height','weight']) 
  students.loc[students['height']==175,'weight'] = 100
  print(students)
  """
     age  height  weight
  1   19     170      68
  2   20     165      65
  3   18     175     100
  """
  
  students.loc[1,:] = [22,180,68]
  """
     age  height  weight
  1   22     180      68
  2   20     165      65
  3   18     175      65
  """
  ```

+ 删除

  ​	DataFrame对象的drop()函数通过axis指明按照行(0)或列(1)删除，且**不修改原始对象的数据**

  ```python
  students.drop(1,axis=0)	# axis=0表示行
  """
  	age	height	weight
  2	20	165	65
  3	18	175	65
  """
  students.drop(['weight','age'],axis=1)
  """
  	height
  1	180
  2	165
  3	175
  """
  ```

  + 如果需要直接删除原始对象的行或列，使用参数inplace=True

  ```python
  students.drop(['weight','age'],axis=1,inplace=True)
  print(students)
  """
     height
  1     180
  2     165
  3     175
  """
  ```

### 3.3 数据文件读写

#### 3.3.1 读写CSV和TXT文件

1. 读写CSV格式文件

   >CSV(Comma Separated Value)是一种特殊的文本文件，通常使用逗号作为字段之间的分隔符，用换行符作为记录之间的分隔符
   >
   >pandas.read_csv()默认第1行为列索引，第1列不是行索引，而是额外生成0~n-1为行索引

   ```python
   pd.read_csv(file,sep=',',header='infer',index_col=None,names,skiprows,...)
   """
   参数说明：
   file:字符串，文件路径和文件名
   sep：字符串，每行各数据之间的分隔符，默认为“，”
   header：header=None，文件中第一行不是列索引
   index_col：数字，用作行索引的列序号
   names:列表，定义列索引，默认文件中第一行为列索引
   skiprows：整数或列表，需要忽略的行数或需要跳过的行号列表
   """
   ```

   下面是几个读取的例子(本项目没有上传对应文件,只自己测试了下)

   ```python
   students = pd.read_csv('data/student1.csv')
   print(students.iloc[:,:])
   """
      序号    性别  年龄   身高  体重        省份  成绩
   0   1  male  20  170  70  LiaoNing  71
   1   2  male  22  180  71   GuangXi  77
   2   3  male  22  180  62    FuJian  57
   3   4  male  20  177  72  LiaoNing  79
   4   5  male  20  172  74  ShanDong  91
   """
   
   students = pd.read_csv('data/student1.csv',header=None)
   print(students.iloc[:,:])
   """
       0     1   2    3   4         5   6
   0  序号    性别  年龄   身高  体重        省份  成绩
   1   1  male  20  170  70  LiaoNing  71
   2   2  male  22  180  71   GuangXi  77
   3   3  male  22  180  62    FuJian  57
   4   4  male  20  177  72  LiaoNing  79
   5   5  male  20  172  74  ShanDong  91
   """
   
   students = pd.read_csv('data/student1.csv',index_col=0)
   print(students)
   print(students.loc[1])
   print(students.iloc[:,0])
   """
         性别  年龄   身高  体重        省份  成绩
   序号                                 
   1   male  20  170  70  LiaoNing  71
   2   male  22  180  71   GuangXi  77
   3   male  22  180  62    FuJian  57
   4   male  20  177  72  LiaoNing  79
   5   male  20  172  74  ShanDong  91
   性别        male
   年龄          20
   身高         170
   体重          70
   省份    LiaoNing
   成绩          71
   Name: 1, dtype: object
   序号
   1    male
   2    male
   3    male
   4    male
   5    male
   Name: 性别, dtype: object
   """
   
   students = pd.read_csv('data/student1.csv',names=['index0','index1','index2','index3','index4','index5','index6'])
   print(students)
   """
     index0 index1 index2 index3 index4    index5 index6
   0     序号     性别     年龄     身高     体重        省份     成绩
   1      1   male     20    170     70  LiaoNing     71
   2      2   male     22    180     71   GuangXi     77
   3      3   male     22    180     62    FuJian     57
   4      4   male     20    177     72  LiaoNing     79
   5      5   male     20    172     74  ShanDong     91
   """
   
   students = pd.read_csv('data/student1.csv',skiprows=[0,2,4])
   print(students)
   """
      1  male  20  170  70  LiaoNing  71
   0  3  male  22  180  62    FuJian  57
   1  5  male  20  172  74  ShanDong  91
   """
   ```

2. 读取TXT文件

   ​	**如果文件不是以逗号作为分隔符的文本（TXT）文件，则读取时需要设置分隔符参数sep。分隔符可以是指定字符串，也可以是正则表达式。**

   + 下表列出最常用的通配符

   <escape>

   <table class="tg">
     <tr>
       <th class="tg-c3ow">通配符</th>
       <th class="tg-0pky">描述</th>
     </tr>
     <tr>
       <td class="tg-c3ow">\s</td>
       <td class="tg-0pky">空格等空白字符</td>
     </tr>
     <tr>
       <td class="tg-0pky">\S</td>
       <td class="tg-0pky">非空白字符</td>
     </tr>
     <tr>
       <td class="tg-0pky">\t</td>
       <td class="tg-0pky">制表符</td>
     </tr>
     <tr>
       <td class="tg-0pky">\n</td>
       <td class="tg-0pky">换行符</td>
     </tr>
     <tr>
       <td class="tg-0pky">\d</td>
       <td class="tg-0pky">数字</td>
     </tr>
     <tr>
       <td class="tg-0lax">\D</td>
       <td class="tg-0lax">非数字字符</td>
     </tr>
   </table>

   </escape>

   以下是读取txt文件的例子，和读取csv文件没有本质区别

   ```python
   students = pd.read_csv('data/student2.txt')
   print(students)
   """
      1\tmale\t20\t170\t70\tLiaoNing\t71
   0   2\tmale\t22\t180\t71\tGuangXi\t77
   1    3\tmale\t22\t180\t62\tFuJian\t57
   2  4\tmale\t20\t177\t72\tLiaoNing\t79
   3  5\tmale\t20\t172\t74\tShanDong\t91
   """
   
   colNames=['性别','年龄','身高','体重','省份','成绩']
   student = pd.read_csv('data/student2.txt',sep='\t',index_col=0,header=None,names=colNames)
   print(student[:2])
   """
        性别  年龄   身高  体重        省份  成绩
   1  male  20  170  70  LiaoNing  71
   2  male  22  180  71   GuangXi  77
   """
   ```

3. 保存CSV格式文件

   ```python
   pd.to_csv(file, sep, mode, index, header, ...)
   """
   参数说明：
   file：文件路径和文件名
   sep：分隔符，默认为逗号
   mode：导出模式，'w'为导出到新文件，'a'为追加到现有文件
   index：是否导出行索引，默认为True
   header：是否导出列索引，默认为True
   """
   
   data = [[19,68,170],[20,65,165],[18,65,175]]
   student = pd.DataFrame(data,index=[1,2,3],columns=['age','weight','height'])
   student.to_csv('out.csv', mode='w', header=True, index=False)
   dataFrame = pd.read_csv('out.csv')
   print(dataFrame)
   """
      age  weight  height
   0   19      68     170
   1   20      65     165
   2   18      65     175
   """
   
   data = [[19,68,170],[20,65,165],[18,65,175]]
   student = pd.DataFrame(data,index=[1,2,3],columns=['age','weight','height'])
   student.to_csv('out.csv', mode='w', header=True, index=True)
   dataFrame = pd.read_csv('out.csv',index_col=0)
   print(dataFrame)
   """
      age  weight  height
   1   19      68     170
   2   20      65     165
   3   18      65     175
   """
   
   data = [[19,68,170],[20,65,165],[18,65,175]]
   student = pd.DataFrame(data,index=[1,2,3],columns=['age','weight','height'])
   student.to_csv('out.csv', mode='w', header=False, index=False)
   dataFrame = pd.read_csv('out.csv',header=None)
   print(dataFrame)
   
   """
       0   1    2
   0  19  68  170
   1  20  65  165
   2  18  65  175
   """
   ```

#### 3.3.2 读取Excel文件

​	从Excel文件读取数据的函数类似CSV文件，只需再给出数据所在的表单名即可，其余的参数含义一致。

```python
pd.read_excel(file, sheetname, ...)

"""
从student3.xlsx文件中名为Group1的表单中读取数据，保存为DataFrame对象
skiprows=3，表示忽略前3行(0,1,2行)；
如果只忽略指定行，则需给出行号，如忽略2、3行，skiprows=[1,2]
"""
student = pd.read_excel('data/student3.xlsx','Group1',index_col=0,skiprows=3)
print(student[:2])
"""
      性别  年龄   身高  体重        省份  成绩
序号                                 
1   male  20  170  70  LiaoNing  71
2   male  22  180  71   GuangXi  77
"""
```

### 3.4 数据清洗

#### 3.4.1 缺失数据处理

​	使用计算机对大量数据进行缺失处理，主要有**数据滤除**和**数据填充**两类方法，DataFrame提供了处理函数实现对应功能。

```python
import pandas as pd
import numpy as np
stu = pd.read_excel('data\studentsInfo.xlsx','Group1',index_col=0) 
print( stu.loc[:3] )
"""
      性别    年龄   身高    体重        省份    成绩    月生活费  课程兴趣  案例教学
序号                                                           
1   male  20.0  170  70.0  LiaoNing   NaN   800.0     5     4
2   male  22.0  180  71.0   GuangXi  77.0  1300.0     3     4
3   male   NaN  180  62.0    FuJian  57.0  1000.0     2     4
"""
```

>输出中缺失的数据被表示为NaN。NaN是在NumPy中定义的，若某个数据填充为缺失值，可以用np.NaN(或np.nan)来赋值。

+ 对缺失数据是填充还是滤除取决于实际应用。<u>如果样本容量很大，则缺失行可以忽略，否则应考虑采用合适的值来填充，以避免样本的浪费。</u>

1. 数据滤除

   ​	DataFrame的dropna()函数删除空值所在的行或列，产生新数据对象，**不修改原始对象**，格式如下：

   ```python
   pandas.DataFrame.dropna(axis, how, thresh, ...)
   
   """
   参数说明：
   axis:0表示按行滤除，1表示按列滤除，默认为axis=0
   how:'all'表示滤除全部值都为NaN的行或列
   thresh:只留下有效数据数大于或等于thresh值得行或列
   """
   
   stu.dropna() #默认删除包含缺失值的行
   stu.dropna(thresh=8) #保留有效数据(非NaN)个数≥8的行
   ```

2. 数据填充

   ​	不能滤除的NaN需要填充后才能保证样本数据完整性。填充有两种基本思路，用默认值填充或用已有数据的均值/中位数填充

   ​	DataFrame的fillna()函数可以实现NaN数据的批量填充功能，也可以对指定的列进行填充，格式如下：

   ```python
   pandas.DataFrame.fillna(value, method, implace, ...)
   
   """
   参数说明：
   value:填充值，可以是标量、字典、Series或DataFrame
   method:'ffill'为用同列前一行数据填充缺失值,'bfill'为用后一行数据填充
   inplace:是否修改原始数值的值,默认为False,产生一个新的数据对象
   """
   
   stu.fillna({'年龄':20,'体重':stu['体重'].mean()})
   stu.fillna(method='bfill')
   ```

#### 3.4.2 去除重复数据

​	用DataFrame的drop_duplicates()函数去除数据值与前面行重复的行，形式如下：

```python
DataFrame.drop_duplicates()
```

### 3.5 数据规整化

#### 3.5.1 数据合并

1. 行数据追加

   ```python
   pd.concat(objs, axis, ...)
   """
   原数据的列索引与新增数据的列索引完全相同，此时数据追加可以通过pandas的轴向连接函数concat()实现，将新增数据保存为另一个DataFrame对象
   """
   """
   参数说明：
   objs:Series、DataFrame的序列或字典
   """
   ```

2. 列数据连接

   ```python
   pd.merge(x,y,how,left_on,right_on, ...)
   """
   参数说明：
   x:左数据对象
   y:右数据对象
   how:数据对象连接的方式，inner、outer、left、right
   left_on:左数据对象用于连接的键
   right_on:右数据对象用于连接的键
   
   参数how定义了四种合并方式
   1)inner:内连接，连接两个数据对象中键值交集的行，其余忽略
   2)outer:外连接，连接两个数据对象中键值并集的行。
   3)left:左连接，取出x的全部行，连接y中匹配的键值行
   4)right:右链接，取出y的全部行，连接x中匹配的键值行
   
   使用第2)、3)或4)种方法合并，当某列数据不存在时自动填充NaN
   """
   ```

#### 3.5.2 数据排序

 1. 值排序

    ```python
    #DataFrame值排序的函数格式如下:
    pd.DataFrame.sort_values(by, ascending, inplace...)
    """
    参数说明:
    by:列索引,定义用于排序的列
    ascending:排序方式,True为升序,False为降序
    inplace:是否修改原始数据对象,True为修改,默认为False
    
    Series值排序省略参数by即可
    """
    
    res = stu.sort_values(by=['身高','体重'],ascending=True)
    #先按"身高"排序,若某些行的"身高"相同,这些行再按"体重"排序
    ```

2. 排名

   ​	排名在排序基础上,进一步给出每行的名次,排名时可以定义等值数据的处理方式,如并列名次最小值或最大值,也可以取均值.

   ```python
   pd.DataFrame.rank(axis,method,ascending, ...)
   
   """
   参数说明:
   axis:0为按行数据排名,1为按列数据排名
   method:并列取值,min、max、mean
   ascending:排序方式,True为升序,False为降序
   """
   
   stu = pd.read_excel('data/studentsInfo.xlsx','Group3',index_col=0)
   print(stu[:5])
   stu['成绩排名'] = stu['成绩'].rank(method='min',ascending=False)
   print(stu[:5])
   """
           性别  年龄   身高  体重        省份  成绩  月生活费  课程兴趣  案例教学
   序号                                                     
   21  female  21  165  45  ShangHai  93  1200     5     5
   22  female  19  167  42     HuBei  89   800     5     5
   23    male  21  169  80     GanSu  93   900     5     5
   24  female  21  160  49     HeBei  59  1100     3     5
   25  female  21  162  54     GanSu  68  1300     4     5
           性别  年龄   身高  体重        省份  成绩  月生活费  课程兴趣  案例教学  成绩排名
   序号                                                           
   21  female  21  165  45  ShangHai  93  1200     5     5   2.0
   22  female  19  167  42     HuBei  89   800     5     5   4.0
   23    male  21  169  80     GanSu  93   900     5     5   2.0
   24  female  21  160  49     HeBei  59  1100     3     5  10.0
   25  female  21  162  54     GanSu  68  1300     4     5   7.0
   """
   ```

### 3.6 统计分析

​	原始数据经过清洗、合并等处理过程后完成数据准备,后续分析通常需要数学计算实现。<u>Series和DataFrame继承了NumPy的数学函数，并提供了更完善的统计、汇总报告分析方法</u>。

#### 3.6.1 通用函数与运算

DataFrame可以实现与DataFrame、Series或标量之间的算数运算

+ DataFrame算术运算

  <escape>

  <table class="tg">
    <tr>
      <th class="tg-c3ow">运算符</th>
      <th class="tg-0pky">描述</th>
    </tr>
    <tr>
      <td class="tg-c3ow">df.T</td>
      <td class="tg-0pky">DataFrame转置</td>
    </tr>
    <tr>
      <td class="tg-0pky">df1 + df2</td>
      <td class="tg-0pky">按照行列索引相加，得到并集，NaN填充</td>
    </tr>
    <tr>
      <td class="tg-0pky">df1.add(df2,fill_value=0)</td>
      <td class="tg-0pky">按照行列索引相加，NaN用指定值填充</td>
    </tr>
    <tr>
      <td class="tg-0pky">df1.add/sub/mul/div</td>
      <td class="tg-0pky">四则运算</td>
    </tr>
    <tr>
      <td class="tg-0pky">df - sr</td>
      <td class="tg-0pky">DataFrame的所有行同时减去Series</td>
    </tr>
    <tr>
      <td class="tg-0lax">df * n</td>
      <td class="tg-0lax">所有元素乘以n</td>
    </tr>
  </table>

  </escape>

+ DataFrame元素级的函数运算可以通过Numpy的一元通用函数(ufunc)实现，格式如下

  ```python
  np.ufunc(df)
  
  # BMI(kg/m^2) = 体重 / 身高^2
  stu['BMI'] = stu['体重'] / (np.square(stu['身高']/100))
  print(stu[:3])
  """
          性别  年龄   身高  体重        省份  成绩  月生活费  课程兴趣  案例教学  成绩排名        BMI
  序号                                                                      
  21  female  21  165  45  ShangHai  93  1200     5     5   2.0  16.528926
  22  female  19  167  42     HuBei  89   800     5     5   4.0  15.059701
  23    male  21  169  80     GanSu  93   900     5     5   2.0  28.010224
  """
  ```

#### 3.6.2 统计函数

+ pandas的常用统计函数，包括Series和DataFrame

  <escape>

  <table class="tg">
    <tr>
      <th class="tg-c3ow">函数</th>
      <th class="tg-0pky">描述</th>
    </tr>
    <tr>
      <td class="tg-c3ow">sr.value_counts()</td>
      <td class="tg-0pky">统计频数</td>
    </tr>
    <tr>
      <td class="tg-0pky">sr.describe()</td>
      <td class="tg-0pky">返回基本统计量和分位数</td>
    </tr>
    <tr>
      <td class="tg-0pky">sr1.corr(sr2)</td>
      <td class="tg-0pky">sr1与sr2的相关系统</td>
    </tr>
    <tr>
      <td class="tg-0pky">df.count()</td>
      <td class="tg-0pky">统计每列数据的个数</td>
    </tr>
    <tr>
      <td class="tg-0pky">df.max()、df.min()</td>
      <td class="tg-0pky">最大值和最小值</td>
    </tr>
    <tr>
      <td class="tg-0lax">dif.idxmax()、dif.idxmin()</td>
      <td class="tg-0lax">最大值、最小值对应的索引</td>
    </tr>
    <tr>
      <td class="tg-0lax">df.sum()</td>
      <td class="tg-0lax">按行或列求和</td>
    </tr>
    <tr>
      <td class="tg-0lax">df.mean()、df.median()</td>
      <td class="tg-0lax">计算均值、中位数</td>
    </tr>
    <tr>
      <td class="tg-0lax">df.quantile()</td>
      <td class="tg-0lax">计算给定的四分位数</td>
    </tr>
    <tr>
      <td class="tg-0lax">df.var()、df.std()</td>
      <td class="tg-0lax">计算方差、标准差</td>
    </tr>
    <tr>
      <td class="tg-0lax">df.mode()</td>
      <td class="tg-0lax">计算众数</td>
    </tr>
    <tr>
      <td class="tg-0lax">df.cumsum()</td>
      <td class="tg-0lax">从0开始向前累加各元素</td>
    </tr>
    <tr>
      <td class="tg-0lax">df.cow()</td>
      <td class="tg-0lax">计算协方差矩阵</td>
    </tr>
    <tr>
      <td class="tg-0lax">pd.crosstab(df[col1],df[col2])</td>
      <td class="tg-0lax">pandas函数，交叉表，计算分组的频数</td>
    </tr>
  </table>

  </escape>

```python
stu = pd.read_excel('data/studentsInfo.xlsx','Group3',index_col=0)
print(stu[:3])
print('-----------------------------------')
print(stu['成绩'].mean()) #计算成绩的平均值
print('-----------------------------------')
print(stu['月生活费'].quantile([.25, .75])) # 计算月生活费的上、下四分位数
print('-----------------------------------')
# 函数describe()可以一次性计算多项统计值，也称为描述统计
print(stu[['身高','体重','成绩']].describe()) # 对身高、体重和成绩3；列数据进行描述统计
print('-----------------------------------')

"""
        性别  年龄   身高  体重        省份  成绩  月生活费  课程兴趣  案例教学
序号                                                     
21  female  21  165  45  ShangHai  93  1200     5     5
22  female  19  167  42     HuBei  89   800     5     5
23    male  21  169  80     GanSu  93   900     5     5
-----------------------------------
78.0
-----------------------------------
0.25     800.0
0.75    1175.0
Name: 月生活费, dtype: float64
-----------------------------------
               身高       体重         成绩
count   10.000000  10.0000  10.000000
mean   165.500000  55.1000  78.000000
std      6.381397  12.8448  14.476034
min    160.000000  42.0000  59.000000
25%    161.250000  49.0000  65.750000
50%    163.500000  51.5000  76.500000
75%    167.750000  53.5000  92.000000
max    181.000000  80.0000  98.000000
-----------------------------------
"""
```

+ <u>分组是根据某些索引将数据对象划分为多个组，然后对每个分组进行排序或统计计算</u>，具体方法如下：

  ```python
  grouped = pd.DataFrame.groupby(col)
  grouped.aggregate({'col1':fun1, 'col2':fun2, ...})
  
  """
  参数说明：
  col：统计列索引名
  fun：NumPy的聚合函数名，如sum、mean、std等
  """
  
  stu = pd.read_excel('data/studentsInfo.xlsx','Group3',index_col=0)
  grouped = stu.groupby(['性别', '年龄'])
  grouped.aggregate({'身高':np.mean,'月生活费':np.max})
  """
  		身高	月生活费
  性别	年龄		
  female	19	167.00	800
  20	164.50	1250
  21	162.25	1300
  22	160.00	800
  male	21	175.00	900
  """
  ```

+ pandas提供类似Excel交叉表的统计函数crosstab(),格式如下。

  函数按照给定的第1列分组，对第2列计数。

  ```python
  pd.crosstab(obj1, obj2, ...)
  """
  参数说明：
  obj1:用于分组的列
  obj2:用于计数的列
  """
  
  pd.crosstab(stu['性别'],stu['月生活费']) #pandas函数
  """
  月生活费	700	800	900	950	1100	1200	1250	1300
  性别								
  female	1	2	0	1	1	1	1	1
  male	0	1	1	0	0	0	0	0
  """
  ```

#### 3.6.3 相关性分析

相关系数r的数学知识，这里不做解释

```python
stu['身高'].corr(stu['体重']) # 两列数据之间的相关性
# 0.6757399098527682

stu[['身高','体重','成绩']].corr() # 多列数据之间的相关性
"""
	身高	体重	成绩
身高	1.000000	0.675740	0.080587
体重	0.675740	1.000000	-0.072305
成绩	0.080587	-0.072305	1.000000
"""
```

## 第4章 数据可视化

### 4.1 Python绘图基础

> Python的Matplotlib是专门用于开发二维(包括三维)图表的工具包，可以实现图像元素精细化控制，绘制专业的分析图表，是目前应用最广泛的数据可视化工具。pandas封装了Matplotlib的主要绘图功能，利用Series和DataFrame对象的数据组织特点简便、快捷地创建标准化图表

#### 4.1.1 认识基本图形

​	按照数据值特性，常用可视图形大致可以分为以下3类：

​	1）展示离散数据：散点图、柱状图、饼图等

​	2）展示连续数据：直方图、箱型图、折线图、半对数图等

​	3）展示数据的区域或空间分布：统计地图、曲面图等

#### 4.1.2 pandas快速绘图

​	pandas基于Series和DataFrame绘图非常简单，只要3个步骤：

​	1）导入Matplotlib、pandas：导入Matplotlib用于图形显示

​	2）准备数据：使用Series或DataFrame封装数据

​	3）绘图：调用Series.plot()或DataFrame.plot()函数完成绘图

```python
# 将绘图显示在控制台console
%matplotlib inline
import matplotlib.pyplot as plt
from pandas import DataFrame
gdp = [41.3,48.9,54.0,59.5,64.4,68.9,74.4]
data = DataFrame({'GDP: Trillion':gdp}, index=['2010','2011','2012','2013','2014','2015','2016'])
print(data)
data.plot()
plt.show() # 显示图形

"""
绘图显示需要自行编写类似的代码查看结果
      GDP: Trillion
2010           41.3
2011           48.9
2012           54.0
2013           59.5
2014           64.4
2015           68.9
2016           74.4
"""
```

​	pandas默认的plot()函数完成了图形的主要信息绘制，但<u>添加各类图元信息，如标题、图例、刻度标签及注释等，或者选择图形的展示类别、控制颜色、位置等，则需要在plot()函数中对相关参数进行设置</u>。

+ 下面列举DataFrame.plot()函数的常用参数，Series.plot()的多数参数与之类似

  <escape>

  <table class="tg">
    <tr>
      <th class="tg-c3ow">参数名</th>
      <th class="tg-0pky">说明</th>
    </tr>
    <tr>
      <td class="tg-0lax">x</td>
      <td class="tg-0lax">x轴数据，默认值为None</td>
    </tr>
    <tr>
      <td class="tg-0lax">y</td>
      <td class="tg-0lax">y轴数据，默认值为None</td>
    </tr>
    <tr>
      <td class="tg-0lax">kind</td>
      <td class="tg-0lax">绘图类型。'line'：折线图，默认值；'bar'：垂直柱状图；'barh':水平柱状图；‘hist’：直方图；<br>‘box’：箱型图；'kde'：Kernel核密度估计图；'density'与kde相同；‘pie’：饼图；‘scatter’：散点图</td>
    </tr>
    <tr>
      <td class="tg-c3ow">title</td>
      <td class="tg-0pky">图形标题，字符串</td>
    </tr>
    <tr>
      <td class="tg-0pky">color</td>
      <td class="tg-0pky">画笔颜色。用颜色缩写，如'r'、'b'，或者RBG值，如#CECECE。<br>主要颜色缩写：‘b’:blue; 'c':cyan; 'g':green; 'k':black; 'm':magenta; 'r':red; 'w':white; 'y':yellow</td>
    </tr>
    <tr>
      <td class="tg-0pky">grid</td>
      <td class="tg-0pky">图形是否有网络，默认值为None</td>
    </tr>
    <tr>
      <td class="tg-0pky">fontsize</td>
      <td class="tg-0pky">坐标轴(包括x轴和y轴)刻度的字体大小。整数，默认值为None</td>
    </tr>
    <tr>
      <td class="tg-0pky">alpha</td>
      <td class="tg-0pky">图表的透明度，值为0~1，值越大颜色越深</td>
    </tr>
    <tr>
      <td class="tg-0pky">use_index</td>
      <td class="tg-0pky">默认为True，用索引作为x轴刻度</td>
    </tr>
    <tr>
      <td class="tg-0pky">linewidth</td>
      <td class="tg-0pky">绘图线宽</td>
    </tr>
    <tr>
      <td class="tg-0pky">linestyle</td>
      <td class="tg-0pky">绘图线型。'-'：实线；‘- -’：破折线；‘-.’：点画线；‘:’虚线</td>
    </tr>
    <tr>
      <td class="tg-0lax">marker</td>
      <td class="tg-0lax">标记风格。‘.’：点；‘,’：像素(极小点)；’o‘:实心圆；’v‘:倒三角；’^‘：上三角；’&gt;‘:右三角；’&lt;‘:左三角；<br>’1‘：下花三角；’2‘：上花三角；’3‘：左花三角；’4‘：右花三角；’s‘：实心方形；’p‘：实星五角；<br>'*'：星形；'h/H'：竖/横六边形；’|‘：垂直线；’+‘：十字；’x‘：x；'D'：菱形；’d‘：瘦菱形</td>
    </tr>
    <tr>
      <td class="tg-0lax">xlim、ylim</td>
      <td class="tg-0lax">x轴、y轴的范围，二元组表示最小值和最大值</td>
    </tr>
    <tr>
      <td class="tg-0lax">ax</td>
      <td class="tg-0lax">axes对象</td>
    </tr>
  </table>

  </escape>

  ```python
  """
  为上面一个绘图的data.plot()函数增加相关参数，自行运行查看效果
  """
  # 将绘图显示在控制台console
  %matplotlib inline
  import matplotlib.pyplot as plt
  from pandas import DataFrame
  gdp = [41.3,48.9,54.0,59.5,64.4,68.9,74.4]
  data = DataFrame({'GDP: Trillion':gdp}, index=['2010','2011','2012','2013','2014','2015','2016'])
  data.plot(title='2010~2016 GDP',linewidth=2, marker='o', linestyle='dashed',color='r', grid=True,alpha=0.9,use_index=True,yticks=[35,40,45,50,55,60,65,70,75])
  plt.show() # 显示图形
  ```

#### 4.1.3 Matplotlib精细绘图

​	pandas绘图简单直接，可以完成基本的标准图形绘制，但如果需要更细致地控制图表样式，如添加标注、在一幅图中包括多幅子图等，必须使用Matplotlib提供的基础函数。

1. 绘图

   使用Matplotlib绘图，需要4个步骤：

   1）导入Matplotlib。导入绘图工具包Matplotlib的pyplot模块。

   2）**创建figure对象。Matplotlib的图像都位于figure对象内**。

   3）绘图。利用pyplot的绘图命令或pandas绘图命令。其中plot()是主要的绘图函数，可实现基本绘图。

   4）设置图元。<u>使用pyplot的图元设置函数，实现图形精细控制</u>。

   ```python
   %matplotlib inline
   import matplotlib.pyplot as plt # 导入绘图库
   plt.figure() # 创建绘图对象
   GDPdata = [41.3,48.9,54.0,59.5,64.4,68.9,74.4] # 准备绘图的序列数据
   plt.plot(GDPdata,color="red",linewidth=2,linestyle='dashed',marker='o',label='GDP') # 绘图
   # 精细设置图元
   plt.title('2010~2016 GDP: Trillion')
   plt.xlim(0,6) # x轴绘图范围（两头闭区间）
   plt.ylim(35,75) # y轴绘图范围
   plt.xticks(range(0,7),('2010','2011','2012','2013','2014','2015','2016')) # 将x轴刻度映射为字符串
   plt.legend(loc='upper right') # 在右上角显示图例说明
   plt.grid() # 显示网格线
   plt.show() # 显示并关闭绘图
   """
   使用matplotlib.pyplot库，图形绘制完成后再通过plt.show()函数显示图形并关闭此次绘图
   """
   ```

2.  多子图

   ​	**figure对象可以绘制多个子图**，以便从不同角度观察数据。<u>首先在figure对象创建子图对象axes，然后在子图上绘制图形</u>，绘图使用pyplot或axes对象提供的各种绘图命令，也可以使用pandas绘图。

   >【Python】 【绘图】plt.figure()的使用
   >
   > https://blog.csdn.net/m0_37362454/article/details/81511427
   >
   >python使用matplotlib:subplot绘制多个子图 
   >
   > https://www.cnblogs.com/xiaoboge/p/9683056.html 

   ```python
   # 创建子图的函数如下：
   figure.add_subplot(numRows, numCols, plotNum)
   """
   参数说明：
   numRows：绘图区被分成numRows行
   numCols：绘图区被分成numCols列
   plotNum：创建的axes对象所在的区域
   """
   
   """
   用多个子图绘制2010-2016年的GDP状况
   """
   %matplotlib inline
   from pandas import Series
   data = Series([41.3,48.9,54.0,59.5,64.4,68.9,74.4], 
                 index=['2010','2011','2012','2013','2014','2015','2016'])
   fig=plt.figure(figsize=(6,6)) #figsize定义图形大小
   ax1=fig.add_subplot(2,1,1)   #创建子图1 
   ax1.plot(data)               #用AxesSubplot绘制折线图
   ax2=fig.add_subplot(2,2,3)   #创建子图2 
   data.plot(kind='bar',use_index=True,fontsize='small',ax=ax2)#用pandas绘柱状图
   ax3=fig.add_subplot(2,2,4)   #创建子图3 
   data.plot(kind='box',fontsize='small',xticks=[],ax=ax3) #用pandas绘柱状图
   ```

3. 设置图元属性和说明

   ​	Matplotlib提供了<u>对图中各种图元信息增加和设置</u>的功能，常用图元设置函数如下，具体参数参见官方文档资料。

   <escape>

   <table class="tg">
     <tr>
       <th class="tg-c3ow">函数</th>
       <th class="tg-0pky">说明</th>
     </tr>
     <tr>
       <td class="tg-0lax">plt.title</td>
       <td class="tg-0lax">设置图标题</td>
     </tr>
     <tr>
       <td class="tg-0lax">plt.xlabel、plt.ylabel</td>
       <td class="tg-0lax">设置x轴、y轴标题</td>
     </tr>
     <tr>
       <td class="tg-0lax">plt.xlim、plt.ylim</td>
       <td class="tg-0lax">设置x轴、y轴刻度范围</td>
     </tr>
     <tr>
       <td class="tg-c3ow">plt.xticks、plt.yticks</td>
       <td class="tg-0pky">设置x轴、y轴刻度值</td>
     </tr>
     <tr>
       <td class="tg-0pky">plt.legend</td>
       <td class="tg-0pky">添加图例说明</td>
     </tr>
     <tr>
       <td class="tg-0pky">plt.grid</td>
       <td class="tg-0pky">显示网格线</td>
     </tr>
     <tr>
       <td class="tg-0pky">plt.text</td>
       <td class="tg-0pky">添加注释文字</td>
     </tr>
     <tr>
       <td class="tg-0pky">plt.annotate</td>
       <td class="tg-0pky">添加注释</td>
     </tr>
   </table>

   </escape>

   ```python
   %matplotlib inline
   import matplotlib.pyplot as plt #导入matplotlib.pyplot
   import pandas as pd
   from pandas import Series
   data=Series([41.3,48.9,54.0,59.5,64.4,68.9,74.4], index=['2010','2011','2012','2013','2014','2015','2016'])
   data.plot(title='2010-2016 GDP',LineWidth=2, marker='o', linestyle='dashed',color='r',grid=True,alpha=0.9)
   plt.annotate('turning point',xy=(1,48.5),xytext=(1.8,42), arrowprops=dict(arrowstyle='->'))
   plt.text(1.8,70,'GDP keeps booming!',fontsize='larger')
   plt.xlabel('Year',fontsize=12)
   plt.ylabel('GDP Increment Speed(%)',fontsize=12)
   
   """
   #将绘制图形保存到文件
   plt.savefig("2010-2016GDP.png",dpi=200,bbox_inches='tight')
   plt.show()  #注意保存文件需在显示之前
   """
   ```

4. 保存图表到文件

   可以将创建的图表保存到文件中，函数格式如下

   ```python
   figure.savefig(filename, dpi, bbox_inches)
   plt.savefig(filename, dpi, bbox_inches)
   """
   参数说明：
   filename:文件路径及文件名，文件类型可以是jpg、png、pdf、svg、ps等
   dpi：图片分辨率，每英寸点数，默认值为100
   bbox_inches：图表需保存的部分，设置为"tight"可以剪除当前图表周围的空白部分
   """
   plt.savefig('2010-2016GDP.jpg',dpi=400,bbox_inches='tight')
   # savefig()函数必须在show()函数前使用方能保存当前图像
   ```

   + **savefig()函数必须在show()函数前使用方能保存当前图像**

### 4.2 可视化数据探索

#### 4.2.1 绘制常用图形

​	数据探索中常用的图形有曲线图、散点图、柱状图等，每种图形的特点及适应性各不相同。本节绘制实现以pandas绘图函数为主，辅以Matplotlib的一些函数

1. 函数绘图

   ​	函数y=f(x)描述了变量y随自变量x的变化过程。通过函数视图可以直观地观察两个变量之间的关系，也可以为线性或逻辑回归等模型提供结果展示。绘制函数plt.plot()根据给定的x坐标值数组，以及对应的y坐标值数组绘图。x的采样值越多，绘制的曲线越精确。

   ```python
   %matplotlib inline
   import numpy as np
   x = np.linspace(0,6.28,50) # start, end, num-points
   y = np.sin(x) # #计算y=sin(x)数组
   plt.plot(x,y,color='r') # 用红色绘图y=sin(x)
   plt.plot(x,np.exp(-x),c='b') # 用蓝色绘图y=exp(-x)
   ```

2. 散点图(Scatter Diagram)

   ​	散点图描述两个一维数据序列之间的关系，可以表示两个指标的相关关系。它将两组数据分别作为点的横坐标和纵坐标。通过散点图可以分析两个数据序列之间是否具有线性关系，辅助线性或逻辑回归算法建立合理的预测模型

   + 散点图的绘制函数：

   ```python
   DataFrame.plot(kind='scatter',x,y,title,grid,xlim,ylim,label,...)
   DataFrame.plot.scatter(x,y,title,grid,xlim,ylim,label,...)
   """
   参数说明
   x：DataFrame中x轴对应的数据列名
   y：DataFrame中y轴对应的数据列名
   label:图例标签
   """
   
   """
   Matplotlib的scatter()函数也可以绘制散点图，这时各种图元的设置需要采用独立的语句实现
   """
   plt.scatter(x,y,...)
   """
   参数说明：
   x:x轴对应的数据列表或一维数组
   y:y轴对应的数据列表或一维数组
   """
   ```

   + 散点图例子，这里不上传文件，了解下参数使用就好了

   ```python
   %matplotlib inline
   stdata = pd.read_csv('data\students.csv')      #读文件
   stdata.plot(kind='scatter',x='Height',y='Weight',title='Students Body Shape', marker='*',grid=True, xlim=[150,200], ylim=[40,80], label='(Height,Weight)')    #绘图
   plt.show()
   """
   使用Height列作为散点图的x轴，Weight列作为散点图y轴；
   限制x显示范围[150,200],y显示范围[40,80]
   label设置‘(Height,Weight)’作为图例标签的文字
   """
   
   #将数据按性别分组，分别绘制散点图
   #将数据按男生和女生分组
   data1= stdata[stdata['Gender'] == 'male']  #筛选出男生
   data2= stdata[stdata['Gender'] == 'female']  #筛选出女生
   #分组绘制男生、女生的散点图
   plt.figure()
   plt.scatter(data1['Height'],data1['Weight'],c='r',marker='s',label='Male')   
   plt.scatter(data2['Height'],data2['Weight'],c='b',marker='^',label='Female') 
   plt.xlim(150,200)                 #x轴范围
   plt.ylim(40,80)              #y轴范围
   plt.title('Student Body')    #标题
   plt.xlabel('Weight')             #x轴标题
   plt.ylabel('Height')             #y轴标题
   plt.grid()                         #网格线
   plt.legend(loc='upper right')  #图例显示位置
   plt.show()
   ```

   + 绘制散点图矩阵

   ```python
   """
   在数据探索时，可能需要同时观察多组数据之间的关系，可以绘制散点图矩阵。
   pandas提供了scatter_matrix()函数实现此功能
   """
   pd.plotting.scatter_matrix(data,diagonal, ...)
   """
   参数说明：
   data：包含多列数据的DataFrame对象
   diagonal：对角线上的图形类型。通常放置该列数据的密度图或直方图
   """
   
   data = stdata[['Height','Weight','Age','Score']] # 准备数据
   pd.plotting.scatter_matrix(data,diagonal='kde',color='k') # 绘图
   ```

3. 柱状图(Bar Chart)

   ​	柱状图用多个柱体描述单个总体处于不同状态的数量，并按状态序列的顺序排序，柱体高度或长度与该状态下的数量成正比。

   ​	柱状图易于展示数据的大小和比较数据之间的差别，还能用来表示均值和方差估计。按照排列方式的不同，可分为垂直柱状图和水平柱状图。按照表达总体的个数可分为单式柱状图和复式柱状图。**把多个总体同一状态的直条叠加在一起称为堆叠柱状图**。

   + pandas使用plot()函数绘制柱状图，格式如下：

     ```python
     Series.plot(kind,xerr,yerr,stacked,...)
     DataFrame.plot(kind,xerr,yerr,stacked,...)
     """
     参数说明：
     kind:‘bar’为垂直柱状图;'barh'为水平柱状图
     xerr,yerr:x轴、y轴的轴向误差线
     stacked:是否为堆叠图，默认为False
     rot:刻度标签旋转度数，值为0-360
     
     Series和DataFrame的索引会自动作为x轴或y轴的刻度
     """
     ```

   + 柱状图的例子,不上传文件，只需搞懂参数即可

     ```python
     import matplotlib.pyplot as plt #导入matplotlib.pyplot
     import pandas as pd
     import numpy as np
     
     #3. 柱状图 
     #例4-7：绘制出生人口性别比较图
     
     data = pd.read_csv('data\population.csv', index_col ='Year') 
     data1 = data[['Boys','Girls']]
     mean = np.mean(data1,axis=0)      #计算均值
     std = np.std(data1,axis=0)        #计算标准差     
     #创建图
     fig = plt.figure(figsize = (6,2)) #设置图片大小
     plt.subplots_adjust(wspace = 0.6) #设置两个图之间的纵向间隔
     #绘制均值的垂直和水平柱状图，标准差使用误差线来表示
     ax1 = fig.add_subplot(1, 2, 1)
     mean.plot(kind='bar',yerr=std,color='cadetblue',title = 'Average of Births', rot=45, ax=ax1)
     ax2 = fig.add_subplot(1, 2, 2)
     mean.plot(kind='barh',xerr=std,color='cadetblue',title = 'Average of Births', ax=ax2)
     plt.show()
     
     #绘制复式柱状图和堆叠柱状图
     data1.plot(kind='bar',title = 'Births of Boys & Girls')
     data1.plot(kind='bar', stacked=True,title = 'Births of Boys & Girls')
     plt.show()
     """
     print(data)
           Total   Boys  Girls   Ratio
     Year                             
     2010    1592   862    730  117.94
     2011    1604   867    737  117.78
     2012    1635   884    751  117.70
     2013    1640   886    754  117.60
     2014    1683   903    780  115.88
     2015    1655   880    775  113.51
     2016    1786   947    839  112.88
     """
     ```

4.  折线图

   ​	折线图用线条描述事物的发展变化及趋势。横、纵坐标轴上都使用算数刻度的则先图称为**普通折线图，反映事物变化趋势**。一个坐标轴使用算数刻度、另一个坐标轴使用对数刻度的折线图称为**半对数折线图，反应事物变化速度**。

   ​	当比较的两种或多种事物的数据值域相差较大时，用半对数折线图可确切反映出指标“相对增长量”的变化关系。

   >例如，GDP和人均可支配收入有一定的相关性。但两者不在一个数量级，GDP在几十万亿间变化，人均可支配收入在几万元间变化，两者的“绝对增长量”相差较远；“相对增长量”却各自保持相对稳定的范围，用半对数折线图可以直观看出变化速度。

   + **绘制半对数折线图需要在plot()函数中设置参数logx或logy为True**

   ```python
   data = pd.read_csv('data/GDP.csv',index_col = 'Year') # 读取数据
   # 绘制GDP和Income的折线图
   data.plot(title='GDP & Income',linewidth=2,marker='o',linestyle='dashed',grid=True,use_index=True)
   
   # 绘制GDP和Income的半对数折线图
   data.plot(logy=True,LineWidth=2,marker='o',linestyle='dashed',color='G')
   """
   print(data)
                  GDP  Income
   Year                      
   2006  2.190000e+13  0.6416
   2007  2.700000e+13  0.7572
   2008  3.200000e+13  0.8707
   2009  3.490000e+13  0.9514
   2010  4.130000e+13  1.0919
   2011  4.890000e+13  1.3134
   2012  5.400000e+13  1.4699
   2013  5.950000e+13  1.6190
   2014  6.440000e+13  1.7778
   2015  6.890000e+13  1.9397
   2016  7.440000e+13  2.3821
   """
   
   """
   有兴趣的存储上面数据后运行会发现，普通折线图可以看出GDP增长趋势，但Income值太小，在相同刻度下无法反应其变化；使用半对数图，则可以看出人均可支配收入随GDP增长，其增长速度超过了GDP增长速度。
   """
   ```

5. 直方图(Histogram)

   ​	直方图用于描述总体的频数分布情况。它将横坐标按区间个数等分，每个区间上长方形的高度表示该区间样本的频率，面积表示频数。直方图的外观和柱状图相似，但表达含义不同。柱状图的一个柱体高度表示横坐标某点对应的数据值，柱体间有间隔；直方图的一个柱体表示一个区间对应的样本个数，柱体间无分割。

   + pandas使用plot()函数绘制直方图，格式如下:

   ```python
   Series.plot(kind='hist',bins,normed,...)
   """
   参数说明：
   bins：横坐标区间个数
   normed：是否标准化直方图，默认值为False
   """
   ```

   + 直方图例子：在直方图中，分箱的数量与数据集大小和分布本身有关，通过改变分箱bins的数量，可以改变分布的离散化程度。

   ```python
   stdata = pd.read_csv('data/students.csv') # 读文件
   stdata['Height'].plot(kind='hist',bins=6,title='Students Height Distribution')
   ```

6. 密度图(Kernel Density Estimate)

   ​	密度图基于样本数据，采用平滑的峰值函数(称为"核")来拟合概率密度函数，对真实的概率分布曲线进行模拟。有很多种核函数，默认采用高斯核。

   ​	密度图经常和直方图画在一起，这时直方图需要标准化，以便与估计的概率密度进行对比。

   + pandas使用plot()函数绘制概率密度函数曲线，格式如下：

   ```python
   Series.plot(kind='kde', style, ...)
   """
   参数说明：
   style：风格字符串，包括颜色和线型，如'k--','r-'
   """
   ```

   + 密度图使用样例，看看参数使用就好了.

   ```python
   stdata['Height'].plot(kind='hist',bins=6,normed=True,title='Students Height Distribution') # 绘制直方图
   stdata['Height'].plot(kind='kde',title='Students Height Distribution',xlim=[155,185],style= 'k--') # 绘制密度图
   ```

7. 饼图(Pie Chart)

   ​	饼图又称扇形图，描述总体的样本值构成比。它以一个圆的面积表示总体，以各扇形面积表示一类样本占总体的百分数。饼图可以清楚地反应出部分与部分、部分与整体之间的数量关系。

   + pandas使用plot()函数绘制饼图，格式如下：

   ```python
   Series.plot(kind='pie',explode,shadow,startangle,autopct, ...)
   """
   参数说明：
   explode:列表，表示各扇形块离开中心的距离
   shadow:扇形块是否有阴影，默认值为False
   startangle:起始绘制角度，默认从x轴正方向逆时针开始
   autopct:百分比格式，可用format字符串或format function，'%1.1f%%'指小数点前后各1位(不足空格补齐)
   """
   ```

   + 饼图示例
   
   ```python
   # 准备数据，计算各类广告投入费用总和
   data = pd.read_csv('data/advertising.csv')
   piedata = data[['TV','Weibo','WeChat']]
   datasum = piedata.sum()
   # 绘制饼图
   datasum.plot(kind='pie',figsize=(6,6),title='Advertising Expenditure',
                fontsize=14,explode=[0,0.2,0],shadow=True,startangle=60,autopct='%1.1f%%')
   ```

8. 箱型图(Box Plot)

   ​	<u>箱型图又称盒式图，适于表达数据的**分位数**分布</u>，帮助找到异常值。它将样本居中的50%值域用一个长方形表示，较小和较大的四分之一值域更用一根线表示，异常值用'o'表示。

   + pandas可以使用plot()函数绘制箱型图，格式如下：

   ```python
   Series.plot(kind='box', ...)
   ```

   + Series.plot绘制箱型图示例

   ```python
   import matplotlib.pyplot as plt
   import pandas as pd
   data = pd.read_csv('data\Advertising.csv')
   advdata = data[['TV','Weibo','WeChat']]
   advdata.plot(kind='box', figsize=(6,6), title='Advertising Expenditure')
   plt.show()
   ```

   ​	观察箱型图，可以快速确定一个样本是否有利于进行分组判别。再分配直方图和密度图就可以更完整地观察数据的分布。

   ​	pandas也提供了专门绘制箱型图的函数boxplot(),方便将观察样本按照其他特征进行分组对比，格式如下：

   ```python
   DataFrame.boxplot(by, ...)
   # by: 用于分组的别名
   ```

   + 使用示例：

   ```python
   stdata = pd.read_csv('data\students.csv')
   stdata1 = stdata[['Gender','Score']]
   stdata1.boxplot(by='Gender',figsize=(6,6))
   plt.show()
   ```

#### 4.2.2 绘制数据地图

​	将总体样本的数量与地域上的分布情况用各种几何图形、实物形象或不同线纹、颜色等在地图上表示出来的图形，称为数据地图。它可以直观地描述某种现象的地域分布。

​	Basemap是Matplotlib的扩展工具包，可以处理地理数据，但Anaconda3中没有包含，需要下载和安装pyproj和basemap工具包后方可导入。

​	这里不展开作笔记了，因为用得也不多，感兴趣的可以再查查。



