# Scala杂记

> [2021版 Scala大数据 完整版-通俗易懂（半天掌握）](https://www.bilibili.com/video/BV1JX4y1u7HK?p=11)	<=	主要根据这个视频，先大致了解一下scala语法，后面打算再快速过一遍[Scala语言规范.pdf](https://static.runoob.com/download/Scala语言规范.pdf)

# 1. 六大特性

## 1. 混编

## 2. 类型自动推断

## 3. 适合高并发和分布式

## 4. 特征特质trait

## 5. 模式匹配

## 6. 高阶函数

# 2. 基本操作

## 1. 数据类型

| 数据类型 | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| Byte     | 8位有符号补码整数。数值区间为 -128 到 127                    |
| Short    | 16位有符号补码整数。数值区间为 -32768 到 32767               |
| Int      | 32位有符号补码整数。数值区间为 -2147483648 到 2147483647     |
| Long     | 64位有符号补码整数。数值区间为 -9223372036854775808 到 9223372036854775807 |
| Float    | 32 位, IEEE 754 标准的单精度浮点数                           |
| Double   | 64 位 IEEE 754 标准的双精度浮点数                            |
| Char     | 16位无符号Unicode字符, 区间值为 U+0000 到 U+FFFF             |
| String   | 字符序列                                                     |
| Boolean  | true或false                                                  |
| Unit     | 表示无值，和其他语言中void等同。用作不返回任何结果的方法的结果类型。Unit只有一个实例值，写成()。 |
| Null     | null 或空引用                                                |
| Nothing  | Nothing类型在Scala的类层级的最底端；它是任何其他类型的子类型。 |
| Any      | Any是所有其他类的超类                                        |
| AnyRef   | AnyRef类是Scala里所有引用类(reference class)的基类           |
| Nil      | 空List(长度为0)                                              |

## 2. 操作

### 1. if控制语句

### 2. 循环语句

+ for循环
+ while循环

# 3. 函数

## 1. 函数的定义

+ def方法名(参数列表):返回类型={方法体}
+ {}只有一行逻辑代码时可以省略
+ return可以省略，自动将最后一行的内容作为返回值，如果return没有省略，那么必须显式声明返回类型
+ =可以省略，表示丢弃返回值
+ 函数中的参数为val常量，值只能使用不可修改

## 2. 递归函数

+ 必须声明返回类型

## 3. 有默认值参数的函数

## 4. 可变参数函数

+ 类型后加星号，比如`int*`

## 5. 嵌套函数

+ 记得调用嵌套的函数

## 6. 偏应用函数

+ 绑定不变的参数

## 7. 柯里化函数

> [Scala函数的柯里化](https://zhuanlan.zhihu.com/p/98096436)

实际就是利用返回值为函数的匿名函数来实现的

```scala
def add(x:Int)(y:Int)=x+y
def add(x:Int) = (y:Int) => x + y
```

## 8. 匿名函数

## 9. 高阶函数

+ 函数的参数是函数
+ 函数的返回类型是函数
+ 函数的参数、返回类型都是函数

## 10. 偏函数

+ 首先从观感上来看，没有match的模式匹配。所以说match的参数隐藏在了偏函数的参数中
+ isdefinedat
+ apply
+ 偏函数是一个只关注自己感兴趣的东西的函数

# 4. 集合操作

## 1. Array

### 创建

```scala
Array(1,2,3)
new Array[Int](3)
```

## 2. List

### map和flatmap

> [scala中的flatMap详解?](https://www.zhihu.com/question/34548588)

+ 一对一
+ 一对多

## 3. Set

### 交并差

+ intersect
+ diff
+ union
+ subsetof

## 4. Map

## 5. Tuple

### 遍历

productIterator

### 二元组翻转

swap

# 5. trait

+ trait = 接口+抽象类

+ 多继承 extends with
+ 在继承多个trait的过程中，有相同的属性时，要override

# 6. 默认匹配

+ ```scala
  x match{
    case 1 => ...
    case x:String => ...
    case _ => ...
  }
  ```

+ 相比java的switch case不光可以匹配值，还可以匹配类型

# 7. 样例类

+ case class 类名

+ 可以不用new
+ 具体属性值完全相同的对象equals返回true

# 8. Actor

+ 写法类似于线程
+ actor之间的相互通信
  + 异步非阻塞
  + 消息发出后不可更改
  + actor内部有一个类似消息队列，会定期检查队列中的消息进行处理
+ 使用
  + ！：`actor ! msg`，表示给actor发msg
  + 先actor.start => 调用act方法 => `receive(case)`

# 9. 隐式转换

## implicit

## 隐式值和隐式参数

## 隐式转换函数

## 隐式类

