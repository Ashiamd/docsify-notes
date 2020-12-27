# Scala菜鸟教程学习

> [scala菜鸟教程](https://www.runoob.com/scala/scala-intro.html)

# 1. Scala简介

## 1.1 Scala特性

### 1.1.1 面向对象特性

​	**Scala是一种纯面向对象的语言，每个值都是对象**。对象的数据类型以及行为由类和特质描述。

​	类抽象机制的扩展有两种途径：一种途径是子类继承，另一种途径是灵活的混入机制。这两种途径能避免多重继承的种种问题。

### 1.1.2 函数式编程

​	**Scala也是一种函数式语言，其函数也能当成值来使用**。Scala提供了轻量级的语法用以定义匿名函数，支持高阶函数，允许嵌套多层函数，并支持柯里化。Scala的case class及其内置的模式匹配相当于函数式编程语言中常用的代数类型。

​	更进一步，程序员可以利用Scala的模式匹配，编写类似正则表达式的代码处理XML数据。

### 1.1.3 静态类型

**Scala具备类型系统，通过编译时检查，保证代码的安全性和一致性**。类型系统具体支持以下特性：

- 泛型类
- 协变和逆变
- 标注
- 类型参数的上下限约束
- 把类别和抽象类型作为对象成员
- 复合类型
- 引用自己时显式指定类型
- 视图
- 多态方法

### 1.1.4 扩展性

​	Scala的设计秉承一项事实，即在实践中，某个领域特定的应用程序开发往往需要特定于该领域的语言扩展。Scala提供了许多独特的语言机制，可以以库的形式轻易无缝添加新的语言结构：

- **任何方法可用作前缀或后缀操作符**
- **可以根据预期类型自动构造闭包**

### 1.1.5 并发性

> [Actor模型](https://www.jianshu.com/p/d803e2a7de8e)

​	**Scala使用Actor作为其并发模型，Actor是类似线程的实体，通过邮箱发收消息**。

​	**Actor可以复用线程，因此可以在程序中可以使用数百万个Actor,而线程只能创建数千个**。

​	在2.10之后的版本中，使用Akka作为其默认Actor实现。

## 1.2 谁使用了Scala

- 2009年4月，Twitter宣布他们已经把大部分后端程序从Ruby迁移到Scala，其余部分也打算要迁移。
- 此外，Wattzon已经公开宣称，其整个平台都已经是基于Scala基础设施编写的。
- 瑞银集团把Scala用于一般产品中。
- Coursera把Scala作为服务器语言使用。

## 1.3 Scala Web 框架

以下列出了两个目前比较流行的 Scala 的 Web应用框架：

- [Lift 框架](http://liftweb.net/)
- [Play 框架](http://www.playframework.org/)

# 2. Scala 基础语法

​	Scala 与 Java 的最大区别是：Scala 语句末尾的分号 ; 是可选的。

​	我们可以认为 Scala 程序是对象的集合，通过调用彼此的方法来实现消息传递。接下来我们来理解下，类，对象，方法，实例变量的概念：

- **对象 -** 对象有属性和行为。例如：一只狗的属性有：颜色，名字，行为有：叫、跑、吃等。对象是一个类的实例。
- **类 -** 类是对象的抽象，而对象是类的具体实例。
- **方法 -** 方法描述的基本的行为，一个类可以包含多个方法。
- **字段 -** 每个对象都有它唯一的实例变量集合，即字段。对象的属性通过给字段赋值来创建。

## 2.1 Scala编程

### 2.1.1 交互式编程

交互式编程不需要创建脚本文件，可以通过以下命令调用：

```shell
$ scala
Welcome to Scala version 2.11.7 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_31).
Type in expressions to have them evaluated.
Type :help for more information.

scala> 1 + 1
res0: Int = 2

scala> println("Hello World!")
Hello World!

scala> 
```

### 2.1.2 脚本形式

我们也可以通过创建一个 HelloWorld.scala 的文件来执行代码，HelloWorld.scala 代码如下所示：

```scala
object HelloWorld {
  /* 这是我的第一个 Scala 程序
    * 以下程序将输出'Hello World!' 
    */
  def main(args: Array[String]) {
    println("Hello, world!") // 输出 Hello World
  }
}
```

接下来我们使用 scalac 命令编译它：(试了一下，确实是生成两个`.class`文件)

```shell
$ scalac HelloWorld.scala 
$ ls
HelloWorld$.class    HelloWorld.scala
HelloWorld.class 
```

编译后我们可以看到目录下生成了 HelloWorld.class 文件，**该文件可以在Java Virtual Machine (JVM)上运行**。编译后，我们可以使用以下命令来执行程序：(试了下`scala HelloWorld$`会报错)

```shell
$ scala HelloWorld
Hello, world!
```

## 2.2 基本语法

Scala 基本语法需要注意以下几点：

- **区分大小写** -  Scala是大小写敏感的，这意味着标识Hello 和 hello在Scala中会有不同的含义。

- **类名** - 对于所有的类名的第一个字母要大写。
  如果需要使用几个单词来构成一个类的名称，每个单词的第一个字母要大写。

  示例：*class MyFirstScalaClass*

- **方法名称** - 所有的方法名称的第一个字母用小写。
  如果若干单词被用于构成方法的名称，则每个单词的第一个字母应大写。

  示例：*def myMethodName()*

- **程序文件名** - 程序文件的名称应该与对象名称完全匹配(新版本不需要了，但建议保留这种习惯)。
  保存文件时，应该保存它使用的对象名称（记住Scala是区分大小写），并追加".scala"为文件扩展名。 （如果文件名和对象名称不匹配，程序将无法编译）。

  示例: 假设"HelloWorld"是对象的名称。那么该文件应保存为'HelloWorld.scala"

- **def main(args: Array[String])** - Scala程序从main()方法开始处理，这是每一个Scala程序的强制程序入口部分。

## 2.3 标识符

Scala 可以使用两种形式的标志符，字符数字和符号。

字符数字使用字母或是下划线开头，后面可以接字母或是数字，符号"$"在 Scala 中也看作为字母。

**然而以"$"开头的标识符为保留的 Scala 编译器产生的标志符使用，应用程序应该避免使用"$"开始的标识符，以免造成冲突**。

Scala 的命名规则采用和 Java 类似的 camel 命名规则，首字符小写，比如 toString。类名的首字符还是使用大写。此外也应该**避免使用以下划线结尾的标志符以避免冲突**。符号标志符包含一个或多个符号，如+，:，? 等，比如:

```
+ ++ ::: < ?> :->
```

**Scala 内部实现时会使用转义的标志符，比如:-> 使用 $colon$minus$greater 来表示这个符号。因此如果你需要在 Java 代码中访问:->方法，你需要使用 Scala 的内部名称 $colon$minus$greater。**

混合标志符由字符数字标志符后面跟着一个或多个符号组成，比如 unary_+ 为 Scala 对+方法的内部实现时的名称。字面量标志符为使用"定义的字符串，比如 `` `x` `` ,  `` `yield` ``。

**你可以在"之间使用任何有效的 Scala 标志符，Scala 将它们解释为一个 Scala 标志符**，一个典型的使用为 Thread 的 yield 方法， 在 Scala 中你不能使用 Thread.yield()是因为 yield 为 Scala 中的关键字， 你必须使用 Thread.`` `yield` ``()来使用这个方法。

## 2.4 Scala 关键字

下表列出了 scala 保留关键字，我们不能使用以下关键字作为变量：

| abstract  | case     | catch    | class   |
| --------- | -------- | -------- | ------- |
| def       | do       | else     | extends |
| false     | final    | finally  | for     |
| forSome   | if       | implicit | import  |
| lazy      | match    | new      | null    |
| object    | override | package  | private |
| protected | return   | sealed   | super   |
| this      | throw    | trait    | try     |
| true      | type     | val      | var     |
| while     | with     | yield    |         |
| -         | :        | =        | =>      |
| <-        | <:       | <%       | >:      |
| #         | @        |          |         |

## 2.5 Scala 注释

Scala 类似 Java 支持单行和多行注释。多行注释可以嵌套，但必须正确嵌套，一个注释开始符号对应一个结束符号。注释在 Scala 编译中会被忽略，实例如下：

```scala
object HelloWorld {
  /* 这是一个 Scala 程序
    * 这是一行注释
    * 这里演示了多行注释
    */
  def main(args: Array[String]) {
    // 输出 Hello World
    // 这是一个单行注释
    println("Hello, world!") 
  }
}
```

## 2.6 空行和空格

一行中只有空格或者带有注释，Scala 会认为其是空行，会忽略它。标记可以被空格或者注释来分割。

## 2.7 换行符

Scala是面向行的语言，语句可以用分号（;）结束或换行符。Scala 程序里,语句末尾的分号通常是可选的。如果你愿意可以输入一个,但若一行里仅 有一个语句也可不写。另一方面,如果一行里写多个语句那么分号是需要的。例如

```scala
val s = "菜鸟教程"; println(s)
```

## 2.8 Scala包

### 2.8.1 定义包

Scala 使用 package 关键字定义包，在Scala将代码定义到某个包中有两种方式：

第一种方法和 Java 一样，在文件的头定义包名，这种方法就后续所有代码都放在该包中。 比如：

```scala
package com.runoob
class HelloWorld
```

第二种方法有些类似 C#，如：

```scala
package com.runoob {
  class HelloWorld 
}
```

第二种方法，可以在一个文件中定义多个包。

### 2.8.2 引用

> [scala 重命名和隐藏方法](https://blog.csdn.net/liangzelei/article/details/80509306)

Scala 使用 import 关键字引用包。

```scala
import java.awt.Color  // 引入Color
 
import java.awt._  // 引入包内所有成员
 
def handler(evt: event.ActionEvent) { // java.awt.event.ActionEvent
  ...  // 因为引入了java.awt，所以可以省去前面的部分
}
```

import语句可以出现在任何地方，而不是只能在文件顶部。import的效果从开始延伸到语句块的结束。这可以大幅减少名称冲突的可能性。

**如果想要引入包中的几个成员，可以使用selector（选取器）**：

```scala
import java.awt.{Color, Font}
 
// 重命名成员
import java.util.{HashMap => JavaHashMap}
 
// 隐藏成员 
import java.util.{HashMap => _, _} // 引入了util包的所有成员，但是HashMap被隐藏了
// 现在，HashMap很明确的便指向了scala.collection.mutable.HashMap，因为java.util.HashMap被隐藏起来了
```

> **注意：***默认情况下，Scala 总会引入 java.lang._ 、 scala._ 和 Predef._，这里也能解释，为什么以scala开头的包，在使用时都是省去scala.的。*

# 3. Scala 数据类型

Scala 与 Java有着相同的数据类型，下表列出了 Scala 支持的数据类型：

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
| Unit     | **表示无值，和其他语言中void等同**。用作不返回任何结果的方法的结果类型。Unit只有一个实例值，写成()。 |
| Null     | null 或空引用                                                |
| Nothing  | **Nothing类型在Scala的类层级的最底端；它是任何其他类型的子类型** |
| Any      | **Any是所有其他类的超类**                                    |
| AnyRef   | **AnyRef类是Scala里所有引用类(reference class)的基类**       |

**上表中列出的数据类型都是对象，也就是说scala没有java中的原生类型**。

在scala是可以对数字等基础类型调用方法的。

## 3.1 Scala 基础字面量

Scala 非常简单且直观。接下来我们会详细介绍 Scala 字面量。

### 3.1.1 整型字面量

整型字面量用于 Int 类型，如果表示 Long，可以在数字后面添加 L 或者小写 l 作为后缀。：

```scala
0
035
21
0xFFFFFFFF 
0777L
```

### 3.1.2 浮点型字面量

如果浮点数后面有f或者F后缀时，表示这是一个Float类型，否则就是一个Double类型的。实例如下：

```scala
0.0 
1e30f 
3.14159f 
1.0e100
.1
```

### 3.1.3 布尔型字面量

布尔型字面量有 true 和 false。

### 3.1.4 符号字面量

> [case class 和 class的区别](https://blog.csdn.net/iamiman/article/details/53537427)
>
> [case class 和class的区别以及构造器参数辨析](https://www.cnblogs.com/PerkinsZhu/p/9307460.html)
>
> case class 主要就是为了方便模式匹配

符号字面量被写成： **'<标识符>** ，这里 **<标识符>** 可以是任何字母或数字的标识（注意：不能以数字开头）。这种字面量被映射成预定义类scala.Symbol的实例。

如： 符号字面量 **'x** 是表达式 **scala.Symbol("x")** 的简写，符号字面量定义如下：

```scala
package scala
final case class Symbol private (name: String) {
   override def toString: String = "'" + name
}
```

### 3.1.5 字符字面量

在 Scala 字符变量使用单引号 **'** 来定义，如下：

```scala
'a' 
'\u0041'
'\n'
'\t'
```

其中 **`\`** 表示转义字符，其后可以跟 **u0041** 数字或者 **\r\n** 等固定的转义字符。

### 3.1.6 字符串字面量

在 Scala 字符串字面量使用双引号**\"**来定义，如下：

```scala
"Hello,\nWorld!"
"菜鸟教程官网：www.runoob.com"
```

### 3.1.7 多行字符串的表示方法

多行字符串用三个双引号来表示分隔符，格式为：**""" ... """**。

实例如下：

```scala
val foo = """菜鸟教程
www.runoob.com
www.w3cschool.cc
www.runnoob.com
以上三个地址都能访问"""
```

### 3.1.8 Null 值

空值是 scala.Null 类型。

Scala.Null和scala.Nothing是用统一的方式处理Scala面向对象类型系统的某些"边界情况"的特殊类型。

**Null类是null引用对象的类型，它是每个引用类（继承自AnyRef的类）的子类。Null不兼容值类型。**

### 3.1.9 Scala 转义字符

下表列出了常见的转义字符：

| 转义字符 | Unicode | 描述                                |
| :------- | :------ | :---------------------------------- |
| \b       | \u0008  | 退格(BS) ，将当前位置移到前一列     |
| \t       | \u0009  | 水平制表(HT) （跳到下一个TAB位置）  |
| \n       | \u000a  | 换行(LF) ，将当前位置移到下一行开头 |
| \f       | \u000c  | 换页(FF)，将当前位置移到下页开头    |
| \r       | \u000d  | 回车(CR) ，将当前位置移到本行开头   |
| \"       | \u0022  | 代表一个双引号(")字符               |
| \'       | \u0027  | 代表一个单引号（'）字符             |
| \\       | \u005c  | 代表一个反斜线字符 '\'              |

0 到 255 间的 Unicode 字符可以用一个八进制转义序列来表示，即反斜线‟\‟后跟 最多三个八进制。

在字符或字符串中，反斜线和后面的字符序列不能构成一个合法的转义序列将会导致 编译错误。

以下实例演示了一些转义字符的使用：

```scala
object Test {
   def main(args: Array[String]) {
      println("Hello\tWorld\n\n" );
   }
}
```

执行以上代码输出结果如下所示：

```shell
$ scalac Test.scala
$ scala Test
Hello    World
```

# 4. Scala 变量

变量是一种使用方便的占位符，用于引用计算机内存地址，变量创建后会占用一定的内存空间。

基于变量的数据类型，操作系统会进行内存分配并且决定什么将被储存在保留内存中。因此，通过给变量分配不同的数据类型，你可以在这些变量中存储整数，小数或者字母。

## 4.1 变量声明

在学习如何声明变量与常量之前，我们先来了解一些变量与常量。

- 一、变量： 在程序运行过程中其值可能发生改变的量叫做变量。如：时间，年龄。
- 二、常量： 在程序运行过程中其值不会发生变化的量叫做常量。如：数值 3，字符'A'。

在 Scala 中，使用关键词 **"var"** 声明变量，使用关键词 **"val"** 声明常量。

声明变量实例如下：

```scala
var myVar : String = "Foo"
var myVar : String = "Too"
```

以上定义了变量 myVar，我们可以修改它。

声明常量实例如下：

```scala
val myVal : String = "Foo"
```

以上定义了常量 myVal，它是不能修改的。如果程序尝试修改常量 myVal 的值，程序将会在编译时报错。

## 4.2 变量类型声明

变量的类型在变量名之后等号之前声明。定义变量的类型的语法格式如下：

```scala
var VariableName : DataType [=  Initial Value]

或

val VariableName : DataType [=  Initial Value]
```

## 4.3 变量类型引用

在 Scala 中声明变量和常量不一定要指明数据类型，在没有指明数据类型的情况下，其数据类型是通过变量或常量的初始值推断出来的。

所以，如果**在没有指明数据类型的情况下声明变量或常量必须要给出其初始值，否则将会报错**。

```scala
var myVar = 10;
val myVal = "Hello, Scala!";
```

以上实例中，myVar 会被推断为 Int 类型，myVal 会被推断为 String 类型。

## 4.4 Scala 多个变量声明

Scala 支持多个变量的声明：

```scala
val xmax, ymax = 100  // xmax, ymax都声明为100
```

如果方法返回值是元组，我们可以使用 val 来声明一个元组：

```scala
scala> val pa = (40,"Foo")
pa: (Int, String) = (40,Foo)
```

# 5. Scala 访问修饰符

Scala 访问修饰符基本和Java的一样，分别有：private，protected，public。

**如果没有指定访问修饰符，默认情况下，Scala 对象的访问级别都是 public**。

**Scala 中的 private 限定符，比 Java 更严格，在嵌套类情况下，外层类甚至不能访问被嵌套类的私有成员**。

## 5.1 私有(Private)成员

用 private 关键字修饰，带有此标记的成员仅在包含了成员定义的类或对象内部可见，同样的规则还适用内部类。

```scala
class Outer{
  class Inner{
    private def f(){println("f")}
    class InnerMost{
      f() // 正确
    }
  }
  (new Inner).f() //错误
}
```

**(new Inner).f( )** 访问不合法是因为 **f** 在 Inner 中被声明为 private，而访问不在类 Inner 之内。

但在 InnerMost 里访问 **f** 就没有问题的，因为这个访问包含在 Inner 类之内。

> Java中允许这两种访问，因为它允许外部类访问内部类的私有成员。

## 5.2 保护(Protected)成员

在 scala 中，对保护（Protected）成员的访问比 java 更严格一些。因为**它只允许保护成员在定义了该成员的的类的子类中被访问**。而在java中，用protected关键字修饰的成员，除了定义了该成员的类的子类可以访问，同一个包里的其他类也可以进行访问。

```scala
package p{
  class Super{
    protected def f() {println("f")}
  }
  class Sub extends Super{
    f()
  }
  class Other{
    (new Super).f() //错误
  }
}
```

上例中，Sub 类对 f 的访问没有问题，因为 f 在 Super 中被声明为 protected，而 Sub 是 Super 的子类。相反，Other 对 f 的访问不被允许，因为 other 没有继承自 Super。

> 后者在 java 里同样被认可，因为 Other 与 Sub 在同一包里。

## 5.3 公共(Public)成员

Scala 中，如果没有指定任何的修饰符，则默认为 public。这样的成员在任何地方都可以被访问。

```scala
class Outer {
  class Inner {
    def f() { println("f") }
    class InnerMost {
      f() // 正确
    }
  }
  (new Inner).f() // 正确因为 f() 是 public
}
```

## 5.4 作用域保护

Scala中，访问修饰符可以通过使用限定词强调。格式为:

```scala
private[x] 

或 

protected[x]
```

这里的x指代某个所属的包、类或单例对象。

**如果写成private[x],读作"这个成员除了对[…]中的类或[…]中的包中的类及它们的伴生对像可见外，对其它所有类都是private**。

这种技巧在横跨了若干包的大型项目中非常有用，它允许你定义一些在你项目的若干子包中可见但对于项目外部的客户却始终不可见的东西。

```scala
package bobsrockets{
  package navigation{
    private[bobsrockets] class Navigator{
      protected[navigation] def useStarChart(){}
      class LegOfJourney{
        private[Navigator] val distance = 100
      }
      private[this] var speed = 200
    }
  }
  package launch{
    import navigation._
    object Vehicle{
      private[launch] val guide = new Navigator
    }
  }
}
```

上述例子中，类 Navigator 被标记为 private[bobsrockets] 就是说这个类对包含在 bobsrockets 包里的所有的类和对象可见。

比如说，从 Vehicle 对象里对 Navigator 的访问是被允许的，因为对象 Vehicle 包含在包 launch 中，而 launch 包在 bobsrockets 中，相反，所有在包 bobsrockets 之外的代码都不能访问类 Navigator。

# 6. Scala 运算符

一个运算符是一个符号，用于告诉编译器来执行指定的数学运算和逻辑运算。

Scala 含有丰富的内置运算符，包括以下几种类型：

- 算术运算符
- 关系运算符
- 逻辑运算符
- 位运算符
- 赋值运算符

接下来我们将为大家详细介绍以上各种运算符的应用。

## 6.1 算术运算符

下表列出了 Scala 支持的算术运算符。

假定变量 A 为 10，B 为 20：

| 运算符 | 描述 | 实例                 |
| :----- | :--- | :------------------- |
| +      | 加号 | A + B 运算结果为 30  |
| -      | 减号 | A - B 运算结果为 -10 |
| *      | 乘号 | A * B 运算结果为 200 |
| /      | 除号 | B / A 运算结果为 2   |
| %      | 取余 | B % A 运算结果为 0   |

```scala
object Test {
  def main(args: Array[String]) {
    var a = 10;
    var b = 20;
    var c = 25;
    var d = 25;
    println("a + b = " + (a + b) );
    println("a - b = " + (a - b) );
    println("a * b = " + (a * b) );
    println("b / a = " + (b / a) );
    println("b % a = " + (b % a) );
    println("c % a = " + (c % a) );

  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
a + b = 30
a - b = -10
a * b = 200
b / a = 2
b % a = 0
c % a = 5
```

## 6.2 关系运算符

下表列出了 Scala 支持的关系运算符。

假定变量 A 为 10，B 为 20：

| 运算符 | 描述     | 实例                      |
| :----- | :------- | :------------------------ |
| ==     | 等于     | (A == B) 运算结果为 false |
| !=     | 不等于   | (A != B) 运算结果为 true  |
| >      | 大于     | (A > B) 运算结果为 false  |
| <      | 小于     | (A < B) 运算结果为 true   |
| >=     | 大于等于 | (A >= B) 运算结果为 false |
| <=     | 小于等于 | (A <= B) 运算结果为 true  |

```scala
object Test {
  def main(args: Array[String]) {
    var a = 10;
    var b = 20;
    println("a == b = " + (a == b) );
    println("a != b = " + (a != b) );
    println("a > b = " + (a > b) );
    println("a < b = " + (a < b) );
    println("b >= a = " + (b >= a) );
    println("b <= a = " + (b <= a) );
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
a == b = false
a != b = true
a > b = false
a < b = true
b >= a = true
b <= a = false
```

## 6.3 逻辑运算符

下表列出了 Scala 支持的逻辑运算符。

假定变量 A 为 1，B 为 0：

| 运算符 | 描述   | 实例                       |
| :----- | :----- | :------------------------- |
| &&     | 逻辑与 | (A && B) 运算结果为 false  |
| \|\|   | 逻辑或 | (A \|\| B) 运算结果为 true |
| !      | 逻辑非 | !(A && B) 运算结果为 true  |

```scala
object Test {
  def main(args: Array[String]) {
    var a = true;
    var b = false;

    println("a && b = " + (a&&b) );

    println("a || b = " + (a||b) );

    println("!(a && b) = " + !(a && b) );
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
a && b = false
a || b = true
!(a && b) = true
```

## 6.4 位运算符

位运算符用来对二进制位进行操作，**~,&,|,^** 分别为取反，按位与与，按位与或，按位与异或运算，如下表实例：

| p    | q    | p & q | p \| q | p ^ q |
| :--- | :--- | :---- | :----- | :---- |
| 0    | 0    | 0     | 0      | 0     |
| 0    | 1    | 0     | 1      | 1     |
| 1    | 1    | 1     | 1      | 0     |
| 1    | 0    | 0     | 1      | 1     |

如果指定 A = 60; 及 B = 13; 两个变量对应的二进制为：

```shell
A = 0011 1100

B = 0000 1101

-------位运算----------

A&B = 0000 1100

A|B = 0011 1101

A^B = 0011 0001

~A  = 1100 0011
```

Scala 中的按位运算法则如下：

| 运算符 | 描述           | 实例                                                         |
| :----- | :------------- | :----------------------------------------------------------- |
| &      | 按位与运算符   | (a & b) 输出结果 12 ，二进制解释： 0000 1100                 |
| \|     | 按位或运算符   | (a \| b) 输出结果 61 ，二进制解释： 0011 1101                |
| ^      | 按位异或运算符 | (a ^ b) 输出结果 49 ，二进制解释： 0011 0001                 |
| ~      | 按位取反运算符 | (~a ) 输出结果 -61 ，二进制解释： 1100 0011， 在一个有符号二进制数的补码形式。 |
| <<     | 左移动运算符   | a << 2 输出结果 240 ，二进制解释： 1111 0000                 |
| >>     | 右移动运算符   | a >> 2 输出结果 15 ，二进制解释： 0000 1111                  |
| >>>    | 无符号右移     | A >>>2 输出结果 15, 二进制解释: 0000 1111                    |

```scala
object Test {
  def main(args: Array[String]) {
    var a = 60;           /* 60 = 0011 1100 */  
    var b = 13;           /* 13 = 0000 1101 */
    var c = 0;

    c = a & b;            /* 12 = 0000 1100 */
    println("a & b = " + c );

    c = a | b;            /* 61 = 0011 1101 */
    println("a | b = " + c );

    c = a ^ b;            /* 49 = 0011 0001 */
    println("a ^ b = " + c );

    c = ~a;               /* -61 = 1100 0011 */
    println("~a = " + c );

    c = a << 2;           /* 240 = 1111 0000 */
    println("a << 2 = " + c );

    c = a >> 2;           /* 15 = 1111 */
    println("a >> 2  = " + c );

    c = a >>> 2;          /* 15 = 0000 1111 */
    println("a >>> 2 = " + c );
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
a & b = 12
a | b = 61
a ^ b = 49
~a = -61
a << 2 = 240
a >> 2  = 15
a >>> 2 = 15
```

## 6.4 赋值运算符

以下列出了 Scala 语言支持的赋值运算符:

| 运算符 | 描述                                                         | 实例                                  |
| :----- | :----------------------------------------------------------- | :------------------------------------ |
| =      | 简单的赋值运算，指定右边操作数赋值给左边的操作数。           | C = A + B 将 A + B 的运算结果赋值给 C |
| +=     | 相加后再赋值，将左右两边的操作数相加后再赋值给左边的操作数。 | C += A 相当于 C = C + A               |
| -=     | 相减后再赋值，将左右两边的操作数相减后再赋值给左边的操作数。 | C -= A 相当于 C = C - A               |
| *=     | 相乘后再赋值，将左右两边的操作数相乘后再赋值给左边的操作数。 | C *= A 相当于 C = C * A               |
| /=     | 相除后再赋值，将左右两边的操作数相除后再赋值给左边的操作数。 | C /= A 相当于 C = C / A               |
| %=     | 求余后再赋值，将左右两边的操作数求余后再赋值给左边的操作数。 | C %= A is equivalent to C = C % A     |
| <<=    | 按位左移后再赋值                                             | C <<= 2 相当于 C = C << 2             |
| >>=    | 按位右移后再赋值                                             | C >>= 2 相当于 C = C >> 2             |
| &=     | 按位与运算后赋值                                             | C &= 2 相当于 C = C & 2               |
| ^=     | 按位异或运算符后再赋值                                       | C ^= 2 相当于 C = C ^ 2               |
| \|=    | 按位或运算后再赋值                                           | C \|= 2 相当于 C = C \| 2             |

```scala
object Test {
  def main(args: Array[String]) {
    var a = 10;      
    var b = 20;
    var c = 0;

    c = a + b;
    println("c = a + b  = " + c );

    c += a ;
    println("c += a  = " + c );

    c -= a ;
    println("c -= a = " + c );

    c *= a ;
    println("c *= a = " + c );

    a = 10;
    c = 15;
    c /= a ;
    println("c /= a  = " + c );

    a = 10;
    c = 15;
    c %= a ;
    println("c %= a  = " + c );

    c <<= 2 ;
    println("c <<= 2  = " + c );

    c >>= 2 ;
    println("c >>= 2  = " + c );

    c >>= a ;
    println("c >>= a  = " + c );

    c &= a ;
    println("c &= 2  = " + c );

    c ^= a ;
    println("c ^= a  = " + c );

    c |= a ;
    println("c |= a  = " + c );
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
c = a + b  = 30
c += a  = 40
c -= a = 30
c *= a = 300
c /= a  = 1
c %= a  = 5
c <<= 2  = 20
c >>= 2  = 5
c >>= a  = 0
c &= 2  = 0
c ^= a  = 10
c |= a  = 10
```

运算符优先级取决于所属的运算符组，它会影响算式的的计算。

实例： x = 7 + 3 * 2; 这里， x 计算结果为 13, 而不是 20，因为乘法（*） 高于加法（+）, 所以它先计算 3*2 再加上 7。

查看以下表格，优先级从上到下依次递减，最上面具有最高的优先级，逗号操作符具有最低的优先级。

| 类别 |               运算符               | 关联性 |
| :--: | :--------------------------------: | :----: |
|  1   |               () []                | 左到右 |
|  2   |                ! ~                 | 右到左 |
|  3   |               * / %                | 左到右 |
|  4   |                + -                 | 左到右 |
|  5   |             >> >>> <<              | 左到右 |
|  6   |             > >= < <=              | 左到右 |
|  7   |               == !=                | 左到右 |
|  8   |                 &                  | 左到右 |
|  9   |                 ^                  | 左到右 |
|  10  |                 \|                 | 左到右 |
|  11  |                 &&                 | 左到右 |
|  12  |                \|\|                | 左到右 |
|  13  | = += -= *= /= %= >>= <<= &= ^= \|= | 右到左 |
|  14  |                 ,                  | 左到右 |

# 7. Scala IF...ELSE 语句

Scala IF...ELSE 语句是通过一条或多条语句的执行结果（True或者False）来决定执行的代码块。

可以通过下图来简单了解条件语句的执行过程:

![img](https://www.runoob.com/wp-content/uploads/2015/12/if.png)

## 7.1 if 语句

if 语句有布尔表达式及之后的语句块组成。

**语法**

if 语句的语法格式如下：

```scala
if(布尔表达式)
{
   // 如果布尔表达式为 true 则执行该语句块
}
```

如果布尔表达式为 true 则执行大括号内的语句块，否则跳过大括号内的语句块，执行大括号之后的语句块。

**示例**

```scala
object Test {
  def main(args: Array[String]) {
    var x = 10;

    if( x < 20 ){
      println("x < 20");
    }
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
x < 20
```

## 7.2 if...else 语句

if 语句后可以紧跟 else 语句，else 内的语句块可以在布尔表达式为 false 的时候执行。

**语法**

if...else 的语法格式如下：

```scala
if(布尔表达式){
  // 如果布尔表达式为 true 则执行该语句块
}else{
  // 如果布尔表达式为 false 则执行该语句块
}
```

**示例**

```scala
object Test {
  def main(args: Array[String]) {
    var x = 30;

    if( x < 20 ){
      println("x 小于 20");
    }else{
      println("x 大于 20");
    }
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
x 大于 20
```

## 7.3 if...else if...else 语句

if 语句后可以紧跟 else if...else 语句，在多个条件判断语句的情况下很有用。

**语法**

if...else if...else 语法格式如下：

```scala
if(布尔表达式 1){
   // 如果布尔表达式 1 为 true 则执行该语句块
}else if(布尔表达式 2){
   // 如果布尔表达式 2 为 true 则执行该语句块
}else if(布尔表达式 3){
   // 如果布尔表达式 3 为 true 则执行该语句块
}else {
   // 如果以上条件都为 false 执行该语句块
}
```

**示例**

```scala
object Test {
  def main(args: Array[String]) {
    var x = 30;

    if( x == 10 ){
      println("X 的值为 10");
    }else if( x == 20 ){
      println("X 的值为 20");
    }else if( x == 30 ){
      println("X 的值为 30");
    }else{
      println("无法判断 X 的值");
    }
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
X 的值为 30
```

## 7.4 if...else 嵌套语句

if...else 嵌套语句可以实现在 if 语句内嵌入一个或多个 if 语句。

**语法**

if...else 嵌套语句语法格式如下：

```scala
if(布尔表达式 1){
  // 如果布尔表达式 1 为 true 则执行该语句块
  if(布尔表达式 2){
    // 如果布尔表达式 2 为 true 则执行该语句块
  }
}
```

else if...else 的嵌套语句 类似 if...else 嵌套语句。

**示例**

```scala
object Test {
  def main(args: Array[String]) {
    var x = 30;
    var y = 10;

    if( x == 30 ){
      if( y == 10 ){
        println("X = 30 , Y = 10");
      }
    }
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
X = 30 , Y = 10
```

# 8. Scala 循环

有的时候，我们可能需要多次执行同一块代码。一般情况下，语句是按顺序执行的：函数中的第一个语句先执行，接着是第二个语句，依此类推。

编程语言提供了更为复杂执行路径的多种控制结构。

循环语句允许我们多次执行一个语句或语句组，下面是大多数编程语言中循环语句的流程图：

![循环结构](https://www.runoob.com/wp-content/uploads/2015/12/loop.png)

## 8.1 循环类型

Scala 语言提供了以下几种循环类型。点击链接查看每个类型的细节。

| 循环类型                                                     | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [while 循环](https://www.runoob.com/scala/scala-while-loop.html) | 运行一系列语句，如果条件为true，会重复运行，直到条件变为false。 |
| [do...while 循环](https://www.runoob.com/scala/scala-do-while-loop.html) | 类似 while 语句区别在于判断循环条件之前，先执行一次循环的代码块。 |
| [for 循环](https://www.runoob.com/scala/scala-for-loop.html) | 用来重复执行一系列语句直到达成特定条件达成，一般通过在每次循环完成后增加计数器的值来实现。 |

### 8.1.1 Scala while 循环

只要给定的条件为 true，Scala 语言中的 **while** 循环语句会重复执行循环体内的代码块。

**语法**

Scala 语言中 **while** 循环的语法：

```scala
while(condition)
{
  statement(s);
}
```

在这里，**statement(s)** 可以是一个单独的语句，也可以是几个语句组成的代码块。

**condition** 可以是任意的表达式，当为任意非零值时都为 true。当条件为 true 时执行循环。 当条件为 false 时，退出循环，程序流将继续执行紧接着循环的下一条语句。

**流程图**

![](https://www.runoob.com/wp-content/uploads/2014/09/cpp_while_loop.png)

在这里，*while* 循环的关键点是循环可能一次都不会执行。当条件为 false 时，会跳过循环主体，直接执行紧接着 while 循环的下一条语句。

**示例**

```scala
object Test {
  def main(args: Array[String]) {
    // 局部变量
    var a = 10;

    // while 循环执行
    while( a < 20 ){
      println( "Value of a: " + a );
      a = a + 1;
    }
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
value of a: 10
value of a: 11
value of a: 12
value of a: 13
value of a: 14
value of a: 15
value of a: 16
value of a: 17
value of a: 18
value of a: 19
```

### 8.1.2 Scala do...while 循环

不像 while 循环在循环头部测试循环条件, Scala 语言中，do...while 循环是在循环的尾部检查它的条件。

do...while 循环与 while 循环类似，但是 do...while 循环会确保至少执行一次循环。

**语法**

Scala 语言中 **while** 循环的语法：

```scala
do {
   statement(s);
} while( condition );
```

**流程图**

![Scala 中的 do...while 循环](https://www.runoob.com/wp-content/uploads/2014/09/cpp_do_while_loop.jpg)

请注意，条件表达式出现在循环的尾部，所以循环中的 statement(s) 会在条件被测试之前至少执行一次。

如果条件为 true，控制流会跳转回上面的 do，然后重新执行循环中的 statement(s)。

这个过程会不断重复，直到给定条件变为 false 为止。

**示例**

```scala
object Test {
  def main(args: Array[String]) {
    // 局部变量
    var a = 10;

    // do 循环
    do{
      println( "Value of a: " + a );
      a = a + 1;
    }while( a < 20 )
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
value of a: 10
value of a: 11
value of a: 12
value of a: 13
value of a: 14
value of a: 15
value of a: 16
value of a: 17
value of a: 18
value of a: 19
```

### 8.1.3 Scala for循环

#### for循环

**语法**

Scala 语言中 **for** 循环的语法：

```scala
for( var x <- Range ){
   statement(s);
}
```

以上语法中，**Range** 可以是一个数字区间表示 **i to j** ，或者 **i until j**。左箭头 <- 用于为变量 x 赋值。

**示例**

以下是一个使用了 **i to j** 语法(包含 j)的实例:

```scala
object Test {
  def main(args: Array[String]) {
    var a = 0;
    // for 循环
    for( a <- 1 to 10){
      println( "Value of a: " + a );
    }
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
value of a: 1
value of a: 2
value of a: 3
value of a: 4
value of a: 5
value of a: 6
value of a: 7
value of a: 8
value of a: 9
value of a: 10
```

以下是一个使用了 **i until j** 语法(不包含 j)的实例:

```scala
object Test {
  def main(args: Array[String]) {
    var a = 0;
    // for 循环
    for( a <- 1 until 10){
      println( "Value of a: " + a );
    }
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
value of a: 1
value of a: 2
value of a: 3
value of a: 4
value of a: 5
value of a: 6
value of a: 7
value of a: 8
value of a: 9
```

在 **for 循环** 中你可以使用分号 (;) 来设置多个区间，它将迭代给定区间所有的可能值。以下实例演示了两个区间的循环实例：

```scala
object Test {
  def main(args: Array[String]) {
    var a = 0;
    var b = 0;
    // for 循环
    for( a <- 1 to 3; b <- 1 to 3){
      println( "Value of a: " + a );
      println( "Value of b: " + b );
    }
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
Value of a: 1
Value of b: 1
Value of a: 1
Value of b: 2
Value of a: 1
Value of b: 3
Value of a: 2
Value of b: 1
Value of a: 2
Value of b: 2
Value of a: 2
Value of b: 3
Value of a: 3
Value of b: 1
Value of a: 3
Value of b: 2
Value of a: 3
Value of b: 3
```

#### for循环集合

for 循环集合的语法如下：

```scala
for( x <- List ){
   statement(s);
}
```

以上语法中， **List** 变量是一个集合，for 循环会迭代所有集合的元素。

**示例**

以下实例将循环数字集合。我们使用 *List()* 来创建集合。再以后章节我们会详细介绍集合。

```scala
object Test {
  def main(args: Array[String]) {
    var a = 0;
    val numList = List(1,2,3,4,5,6);

    // for 循环
    for( a <- numList ){
      println( "Value of a: " + a );
    }
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
value of a: 1
value of a: 2
value of a: 3
value of a: 4
value of a: 5
value of a: 6
```

#### for循环过滤

Scala 可以使用一个或多个 **if** 语句来过滤一些元素。

以下是在 for 循环中使用过滤器的语法。

```scala
for( var x <- List
    if condition1; if condition2...
   ){
  statement(s);
```

你可以使用分号(;)来为表达式添加一个或多个的过滤条件。

**示例**

以下是 for 循环中过滤的实例：

```scala
object Test {
  def main(args: Array[String]) {
    var a = 0;
    val numList = List(1,2,3,4,5,6,7,8,9,10);

    // for 循环
    for( a <- numList
        if a != 3; if a < 8 ){
      println( "Value of a: " + a );
    }
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
value of a: 1
value of a: 2
value of a: 4
value of a: 5
value of a: 6
value of a: 7
```

#### for使用yield

你可以将 for 循环的返回值作为一个变量存储。语法格式如下：

```scala
var retVal = for{ var x <- List
                 if condition1; if condition2...
                }yield x
```

注意**大括号**中用于保存变量和条件，*retVal* 是变量， 循环中的 yield 会把当前的元素记下来，保存在集合中，循环结束后将返回该集合。

**示例**

以下实例演示了 for 循环中使用 yield：

```scala
object Test {
  def main(args: Array[String]) {
    var a = 0;
    val numList = List(1,2,3,4,5,6,7,8,9,10);

    // for 循环
    var retVal = for{ a <- numList
                     if a != 3; if a < 8
                    }yield a

    // 输出返回值
    for( a <- retVal){
      println( "Value of a: " + a );
    }
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
value of a: 1
value of a: 2
value of a: 4
value of a: 5
value of a: 6
value of a: 7
```

## 8.2 循环控制语句

循环控制语句改变你代码的执行顺序，通过它你可以实现代码的跳转。Scala 以下几种循环控制语句：

Scala 不支持 break 或 continue 语句，但从 2.8 版本后提供了一种中断循环的方式，点击以下链接查看详情。

| 控制语句                                                     | 描述     |
| :----------------------------------------------------------- | :------- |
| [break 语句](https://www.runoob.com/scala/scala-break-statement.html) | 中断循环 |

### 8.2.1 Scala break 语句

#### break

Scala 语言中默认是没有 break 语句，但是你在 Scala 2.8 版本后可以使用另外一种方式来实现 *break* 语句。当在循环中使用 **break** 语句，在执行到该语句时，就会中断循环并执行循环体之后的代码块。

**语法**

```scala
Scala 中 break 的语法有点不大一样，格式如下：

// 导入以下包
import scala.util.control._

// 创建 Breaks 对象
val loop = new Breaks;

// 在 breakable 中循环
loop.breakable{
  // 循环
  for(...){
    ....
    // 循环中断
    loop.break;
  }
}
```

**流程图**

![img](https://www.runoob.com/wp-content/uploads/2013/11/cpp_break_statement.jpg)

**示例**

```scala
import scala.util.control._

object Test {
  def main(args: Array[String]) {
    var a = 0;
    val numList = List(1,2,3,4,5,6,7,8,9,10);

    val loop = new Breaks;
    loop.breakable {
      for( a <- numList){
        println( "Value of a: " + a );
        if( a == 4 ){
          loop.break;
        }
      }
    }
    println( "After the loop" );
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
Value of a: 1
Value of a: 2
Value of a: 3
Value of a: 4
After the loop
```

#### 中断嵌套循环

以下实例演示了如何中断嵌套循环：

```scala
import scala.util.control._

object Test {
  def main(args: Array[String]) {
    var a = 0;
    var b = 0;
    val numList1 = List(1,2,3,4,5);
    val numList2 = List(11,12,13);

    val outer = new Breaks;
    val inner = new Breaks;

    outer.breakable {
      for( a <- numList1){
        println( "Value of a: " + a );
        inner.breakable {
          for( b <- numList2){
            println( "Value of b: " + b );
            if( b == 12 ){
              inner.break;
            }
          }
        } // 内嵌循环中断
      }
    } // 外部循环中断
  }
}
```

执行以上代码输出结果为：

```shell
$ scalac Test.scala
$ scala Test
Value of a: 1
Value of b: 11
Value of b: 12
Value of a: 2
Value of b: 11
Value of b: 12
Value of a: 3
Value of b: 11
Value of b: 12
Value of a: 4
Value of b: 11
Value of b: 12
Value of a: 5
Value of b: 11
Value of b: 12
```

## 8.3 无限循环

如果条件永远为 true，则循环将变成无限循环。我们可以使用 while 语句来实现无限循环：

```scala
object Test {
  def main(args: Array[String]) {
    var a = 10;
    // 无限循环
    while( true ){
      println( "a 的值为 : " + a );
    }
  }
}
```

以上代码执行后循环会永久执行下去，你可以使用 Ctrl + C 键来中断无限循环。

# 9. Scala方法与函数

Scala 有方法与函数，二者在语义上的区别很小。Scala 方法是类的一部分，而函数是一个对象可以赋值给一个变量。换句话来说在类中定义的函数即是方法。

Scala 中的方法跟 Java 的类似，方法是组成类的一部分。

Scala 中的函数则是一个完整的对象，Scala 中的函数其实就是继承了 Trait 的类的对象。

Scala 中使用 **val** 语句可以定义函数，**def** 语句定义方法。

```scala
class Test{
  def m(x: Int) = x + 3
  val f = (x: Int) => x + 3
}
```

> **注意：**有些翻译上函数(function)与方法(method)是没有区别的。

## 9.1 方法声明

Scala 方法声明格式如下：

```
def functionName ([参数列表]) : [return type]
```

如果你不写等于号和方法主体，那么方法会被隐式声明为**抽象(abstract)**，包含它的类型于是也是一个抽象类型。

## 9.2 方法定义

方法定义由一个 **def** 关键字开始，紧接着是可选的参数列表，一个冒号 **:** 和方法的返回类型，一个等于号 **=** ，最后是方法的主体。

Scala 方法定义格式如下：

```scala
def functionName ([参数列表]) : [return type] = {
   function body
   return [expr]
}
```

以上代码中 **return type** 可以是任意合法的 Scala 数据类型。参数列表中的参数可以使用逗号分隔。

以下方法的功能是将两个传入的参数相加并求和：

```scala
object add{
  def addInt( a:Int, b:Int ) : Int = {
    var sum:Int = 0
    sum = a + b

    return sum
  }
}
```

如果方法没有返回值，可以返回为 **Unit**，这个类似于 Java 的 **void**, 实例如下：

```scala
object Hello{
  def printMe( ) : Unit = {
    println("Hello, Scala!")
  }
}
```

## 9.3 方法调用

Scala 提供了多种不同的方法调用方式：

以下是调用方法的标准格式：

```scala
functionName( 参数列表 )
```

如果方法使用了实例的对象来调用，我们可以使用类似java的格式 (使用 **.** 号)：

```
[instance.]functionName( 参数列表 )
```

以上实例演示了定义与调用方法的实例:

```scala
object Test {
  def main(args: Array[String]) {
    println( "Returned Value : " + addInt(5,7) );
  }
  def addInt( a:Int, b:Int ) : Int = {
    var sum:Int = 0
    sum = a + b

    return sum
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala 
$ scala Test
Returned Value : 12
```

Scala 也是一种函数式语言，所以函数是 Scala 语言的核心。以下一些函数概念有助于我们更好的理解 Scala 编程：

|                      函数概念解析接案例                      |                                                              |
| :----------------------------------------------------------: | :----------------------------------------------------------- |
| [函数传名调用(Call-by-Name)](https://www.runoob.com/scala/functions-call-by-name.html) | [指定函数参数名](https://www.runoob.com/scala/functions-named-arguments.html) |
| [函数 - 可变参数](https://www.runoob.com/scala/functions-variable-arguments.html) | [递归函数](https://www.runoob.com/scala/recursion-functions.html) |
| [默认参数值](https://www.runoob.com/scala/functions-default-parameter-values.html) | [高阶函数](https://www.runoob.com/scala/higher-order-functions.html) |
| [内嵌函数](https://www.runoob.com/scala/nested-functions.html) | [匿名函数](https://www.runoob.com/scala/anonymous-functions.html) |
| [偏应用函数](https://www.runoob.com/scala/partially-applied-functions.html) | [函数柯里化(Function Currying)](https://www.runoob.com/scala/currying-functions.html) |

### 9.3.0 方法method和函数function的区别

> [Scala 方法与函数](https://www.runoob.com/scala/scala-functions.html)	<=	以下内容来自该文章最下面的高分笔记

#### 方法和函数的区别

1、函数可作为一个参数传入到方法中，而方法不行。

![img](https://www.runoob.com/wp-content/uploads/2018/05/scalafunction1.png)

```
object MethodAndFunctionDemo {
  //定义一个方法
  //方法 m1 参数要求是一个函数，函数的参数必须是两个Int类型
  //返回值类型也是Int类型
  def m1(f:(Int,Int) => Int) : Int = {
    f(2,6)
  }

  //定义一个函数f1,参数是两个Int类型，返回值是一个Int类型
  val f1 = (x:Int,y:Int) => x + y
  //再定义一个函数f2
  val f2 = (m:Int,n:Int) => m * n

  //main方法
  def main(args: Array[String]): Unit = {
    //调用m1方法，并传入f1函数
    val r1 = m1(f1)

    println(r1)

    //调用m1方法，并传入f2函数
    val r2 = m1(f2)
    println(r2)
  }
}
```

运行结果：

```
8
12
```

2、在Scala中无法直接操作方法，如果要操作方法，必须先将其转换成函数。有两种方法可以将方法转换成函数：

```
val f1 = m _
```

在方法名称m后面紧跟一个空格和下划线告诉编译器将方法m转换成函数，而不是要调用这个方法。 也可以显示地告诉编译器需要将方法转换成函数：

```
val f1: (Int) => Int = m
```

通常情况下编译器会自动将方法转换成函数，例如在一个应该传入函数参数的地方传入了一个方法，编译器会自动将传入的方法转换成函数。

![img](https://www.runoob.com/wp-content/uploads/2018/05/scalafunction2.png)

```
object TestMap {

  def ttt(f:Int => Int):Unit = {
    val r = f(10)
    println(r)
  }

  val f0 = (x : Int) => x * x

  //定义了一个方法
  def m0(x:Int) : Int = {
    //传递进来的参数乘以10
    x * 10
  }

  //将方法转换成函数，利用了神奇的下滑线
  val f1 = m0 _

  def main(args: Array[String]): Unit = {
    ttt(f0)

    //通过m0 _将方法转化成函数
    ttt(m0 _);

    //如果直接传递的是方法名称，scala相当于是把方法转成了函数
    ttt(m0)

    //通过x => m0(x)的方式将方法转化成函数,这个函数是一个匿名函数，等价：(x:Int) => m0(x)
    ttt(x => m0(x))
  }
}
```

输出结果为：

```
100
100
100
100
```

3、函数必须要有参数列表，而方法可以没有参数列表

![img](https://www.runoob.com/wp-content/uploads/2018/05/131403339175801.png)

4、在函数出现的地方我们可以提供一个方法

在需要函数的地方，如果传递一个方法，会自动进行ETA展开（把方法转换为函数）

![img](https://www.runoob.com/wp-content/uploads/2018/05/1527498797-6462-201504.png)

如果我们直接把一个方法赋值给变量会报错。如果我们指定变量的类型就是函数，那么就可以通过编译，如下：

![img](https://www.runoob.com/wp-content/uploads/2018/05/1527498797-2557-201504.png)

当然我们也可以强制把一个方法转换给函数，这就用到了 scala 中的部分应用函数：

![img](https://www.runoob.com/wp-content/uploads/2018/05/1527498797-2599-201504.png)

---

#### scala method 和 function 的区别

英文原文：

A **Function Type** is (roughly) a type of the form *(T1, ..., Tn) => U*, which is a shorth and for the trait `FunctionN` in the standard library. **Anonymous Functions** and **Method Values** have function types, and function types can be used as part of value, variable and function declarations and definitions. In fact, it can be part of a method type.

A **Method Type** is a ***non-value type***. That means there is ***no*** value - no object, no instance - with a method type. As mentioned above, a **Method Value** actually has a **Function Type**. A method type is a `def` declaration - everything about a `def` except its body.

例子：

```
scala> def m1(x:Int) = x+3
m1: (x: Int)Int　　　　

scala> val f1 = (x: Int) => x+3
f1: Int => Int = <function1>
```

看到没，方法定义和函数定义是不是在scala的解析器signature上就有显示了，def m1(x: Int) = x+3就是一个简单的method的定义。signature中m1: (x: Int)Int　表示method m1有一个参数Int型参数x，返回值是Int型。

val f1 = (x: Int) => x+3则是function的定义，解析器的signature中f1: Int => Int = \<function1\>表示function f1的method体接受一个Int型的参数，输出结果的类型是Int型。

从上面的例子，得出一个总结：

**方法是一个以def开头的带有参数列表（可以无参数列表）的一个逻辑操作块，这正如object或者class中的成员方法一样。**

**函数是一个赋值给一个变量（或者常量）的匿名方法（带或者不带参数列表），并且通过=>转换符号跟上逻辑代码块的一个表达式。=>转换符号后面的逻辑代码块的写法与method的body部分相同。**

##### 其他区别

method 可以作为一个表达式的一部分出现（调用函数并传参），但是 method（带参方法）不能作为最终的表达式（无参方法可以，但是这个就成了方法调用，因为 **scala 允许无参方法调用时省略（）括号**），而 function 可以作为最终的表达式出现。

```shell
scala> m1
<console>:12: error: missing arguments for method m1;
follow this method with `_' if you want to treat it as a partially applied function
       m1
       ^

scala> f1
res1: Int => Int = <function1>
```

method 可以没有参数列表，参数列表也可以为空。但是 function 必须有参数列表（也可以为空）。

**方法名意味着方法调用，函数名只是代表函数自身**：

```shell
scala> def m2 = 100;
m2: Int

scala> def m3() = 1000;
m3: ()Int

scala> var f2 = => 100;
<console>:1: error: illegal start of simple expression
var f2 = => 100;
^

scala> var f2 =()=> 100;
f2: () => Int = <function0>

scala> m2
res2: Int = 100

scala> m3
res3: Int = 1000

scala> m3()
res4: Int = 1000

scala> f2
res5: () => Int = <function0>

scala> f2()
res6: Int = 100
```

在函数出现的地方我们可以提供一个方法。

这是因为，**如果期望出现函数的地方我们提供了一个方法的话，该方法就会自动被转换成函数。该行为被称为 ETA expansion**。

注意：

期望出现函数的地方，我们可以使用方法。

**不期望出现函数的地方，方法并不会自动转换成函数**。

**在 scala 中操作符被解释称方法**：　

- 前缀操作符：op obj 被解释称 obj.op
- 中缀操作符：obj1 op obj2 被解释称 obj1.op(obj2)
- 后缀操作符：obj op 被解释称 obj.op

```shell
scala> val ml = List(1,2,3,4)
ml: List[Int] = List(1, 2, 3, 4)

scala> ml.map((x)=>2*x)
res0: List[Int] = List(2, 4, 6, 8)

scala> def m(x:Int) = 2*x
m: (x: Int)Int

scala> ml.map(m)
res1: List[Int] = List(2, 4, 6, 8)

scala> def m(x:Int) = 3*x
m: (x: Int)Int

scala> ml.map(m)
res2: List[Int] = List(3, 6, 9, 12)
```

**可以在方法名后面加一个下划线强制变成函数**。

**注意：** 方法名与下划线之间至少有一个空格哟!

```shell
scala> def m3(x: Int): Int = x * x * x
m3: (x: Int)Int

scala> val f3 = m3_
<console>:10: error: not found: value m3_
       val f3 = m3_
                ^

scala> val f3 = m3 _
f3: Int => Int = <function1>

scala> f3(3)
res0: Int = 27
```

### 9.3.1 Scala 函数传名调用(call-by-name)

Scala的解释器在解析函数参数(function arguments)时有两种方式：

- 传值调用（call-by-value）：先计算参数表达式的值，再应用到函数内部；
- **传名调用（call-by-name）：将未计算的参数表达式直接应用到函数内部**

在进入函数内部前，传值调用方式就已经将参数表达式的值计算完毕，而传名调用是在函数内部进行参数表达式的值计算的。

这就造成了一种现象，每次使用传名调用时，解释器都会计算一次表达式的值。

```scala
object Test {
   def main(args: Array[String]) {
        delayed(time());
   }

   def time() = {
      println("获取时间，单位为纳秒")
      System.nanoTime
   }
   def delayed( t: => Long ) = {
      println("在 delayed 方法内")
      println("参数： " + t)
      t
   }
}
```

以上实例中我们声明了 delayed 方法， 该方法在**变量名和变量类型使用 => 符号来设置传名调用**。

执行以上代码，输出结果如下：

```shell
$ scalac Test.scala 
$ scala Test
在 delayed 方法内
获取时间，单位为纳秒
参数： 241550840475831
获取时间，单位为纳秒
```

### 9.3.2 Scala 指定函数参数名

一般情况下函数调用参数，就按照函数定义时的参数顺序一个个传递。

但是**我们也可以通过指定函数参数名，并且不需要按照顺序向函数传递参数**，实例如下：

```scala
object Test {
   def main(args: Array[String]) {
        printInt(b=5, a=7);
   }
   def printInt( a:Int, b:Int ) = {
      println("Value of a : " + a );
      println("Value of b : " + b );
   }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
Value of a :  7
Value of b :  5
```

### 9.3.3 Scala 函数 - 可变参数

Scala 允许你指明函数的**最后一个参数**可以是重复的，即我们不需要指定函数参数的个数，可以向函数传入可变长度参数列表。

Scala 通过在参数的类型之后放一个**星号**来设置可变参数(可重复的参数)。例如：

```scala
object Test {
  def main(args: Array[String]) {
    printStrings("Runoob", "Scala", "Python");
  }
  def printStrings( args:String* ) = {
    var i : Int = 0;
    for( arg <- args ){
      println("Arg value[" + i + "] = " + arg );
      i = i + 1;
    }
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
Arg value[0] = Runoob
Arg value[1] = Scala
Arg value[2] = Python
```

### 9.3.4 Scala 函数 - 默认参数值

Scala 可以为函数参数指定默认参数值，使用了默认参数，你在调用函数的过程中可以不需要传递参数，这时函数就会调用它的默认参数值，如果传递了参数，则传递值会取代默认值。实例如下：

```scala
object Test {
  def main(args: Array[String]) {
    println( "返回值 : " + addInt() );
  }
  def addInt( a:Int=5, b:Int=7 ) : Int = {
    var sum:Int = 0
    sum = a + b

    return sum
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
返回值 : 12
```

### 9.3.5 Scala 函数嵌套

我们可以在 Scala 函数内定义函数，**定义在函数内的函数称之为局部函数**。

以下实例我们实现阶乘运算，并使用内嵌函数：

```scala
object Test {
  def main(args: Array[String]) {
    println( factorial(0) )
    println( factorial(1) )
    println( factorial(2) )
    println( factorial(3) )
  }

  def factorial(i: Int): Int = {
    def fact(i: Int, accumulator: Int): Int = {
      if (i <= 1)
      accumulator
      else
      fact(i - 1, i * accumulator)
    }
    fact(i, 1)
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
1
1
2
6
```

### 9.3.6 Scala 匿名函数

Scala 中定义匿名函数的语法很简单，箭头左边是参数列表，右边是函数体。

使用匿名函数后，我们的代码变得更简洁了。

下面的表达式就定义了一个接受一个Int类型输入参数的匿名函数:

```scala
var inc = (x:Int) => x+1
```

上述定义的匿名函数，其实是下面这种写法的简写：

```scala
def add2 = new Function1[Int,Int]{  
  def apply(x:Int):Int = x+1;  
} 
```

以上实例的 inc 现在可作为一个函数，使用方式如下：

```scala
var x = inc(7)-1
```

同样我们可以在匿名函数中定义多个参数：

```scala
var mul = (x: Int, y: Int) => x*y
```

mul 现在可作为一个函数，使用方式如下：

```scala
println(mul(3, 4))
```

我们也可以不给匿名函数设置参数，如下所示：

```scala
var userDir = () => { System.getProperty("user.dir") }
```

userDir 现在可作为一个函数，使用方式如下：

```scala
println( userDir() )
```

**示例**

```scala
object Demo {
  def main(args: Array[String]) {
    println( "multiplier(1) value = " +  multiplier(1) )
    println( "multiplier(2) value = " +  multiplier(2) )
  }
  var factor = 3
  val multiplier = (i:Int) => i * factor
}
```

将以上代码保持到 Demo.scala 文件中，执行以下命令：

```shell
$ scalac Demo.scala
$ scala Demo
```

输出结果为：

```scala
multiplier(1) value = 3
multiplier(2) value = 6
```

### 9.3.7 Scala 递归函数

递归函数在函数式编程的语言中起着重要的作用。

Scala 同样支持递归函数。

递归函数意味着函数可以调用它本身。

以上实例使用递归函数来计算阶乘：

```scala
object Test {
  def main(args: Array[String]) {
    for (i <- 1 to 10)
    println(i + " 的阶乘为: = " + factorial(i) )
  }

  def factorial(n: BigInt): BigInt = {  
    if (n <= 1)
    1  
    else    
    n * factorial(n - 1)
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
1 的阶乘为: = 1
2 的阶乘为: = 2
3 的阶乘为: = 6
4 的阶乘为: = 24
5 的阶乘为: = 120
6 的阶乘为: = 720
7 的阶乘为: = 5040
8 的阶乘为: = 40320
9 的阶乘为: = 362880
10 的阶乘为: = 3628800
```

### 9.3.8 Scala 偏应用函数

**Scala 偏应用函数是一种表达式**，你不需要提供函数需要的所有参数，只需要提供部分，或不提供所需参数。

如下实例，我们打印日志信息：

```scala
import java.util.Date

object Test {
  def main(args: Array[String]) {
    val date = new Date
    log(date, "message1" )
    Thread.sleep(1000)
    log(date, "message2" )
    Thread.sleep(1000)
    log(date, "message3" )
  }

  def log(date: Date, message: String)  = {
    println(date + "----" + message)
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
Mon Dec 02 12:52:41 CST 2018----message1
Mon Dec 02 12:52:41 CST 2018----message2
Mon Dec 02 12:52:41 CST 2018----message3
```

实例中，log() 方法接收两个参数：date 和 message。我们在程序执行时调用了三次，参数 date 值都相同，message 不同。

我们可以使用偏应用函数优化以上方法，绑定第一个 date 参数，第二个参数使用下划线(_)替换缺失的参数列表，并把这个新的函数值的索引的赋给变量。以上实例修改如下：

```scala
import java.util.Date

object Test {
  def main(args: Array[String]) {
    val date = new Date
    val logWithDateBound = log(date, _ : String)

    logWithDateBound("message1" )
    Thread.sleep(1000)
    logWithDateBound("message2" )
    Thread.sleep(1000)
    logWithDateBound("message3" )
  }

  def log(date: Date, message: String)  = {
    println(date + "----" + message)
  }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
Tue Dec 18 11:25:54 CST 2018----message1
Tue Dec 18 11:25:54 CST 2018----message2
Tue Dec 18 11:25:54 CST 2018----message3
```

### 9.3.9 高阶函数

> [Scala泛型的使用](https://www.cnblogs.com/itboys/p/10164173.html)
>
> 1. scala的类和方法、函数都可以是泛型。
>
> 2. 关于对类型边界的限定分为上边界和下边界（对类进行限制）
>
>    + 上边界：表达了泛型的类型必须是"某种类型"或某种类型的"子类"，语法为`<:`
>    + 下边界：表达了泛型的类型必须是"某种类型"或某种类型的"父类"，语法为`>:`
>
> 3. `<%` :view bounds可以进行某种神秘的转换，把你的类型在没有知觉的情况下转换成目标类型
>
>    其实你可以认为view bounds是上下边界的加强和补充，语法为：`<%`，要用到implicit进行隐式转换
>
> 4. `T:classTag`:相当于动态类型，你使用时传入什么类型就是什么类型
>
>    （spark的程序的编译和运行是区分了Driver和Executor的，只有在运行的时候才知道完整的类型信息）
>
>    语法为：`[T:ClassTag]`
>
> 5. 逆变和协变：`-T`和`+T`（下面有具体例子）+T可以传入其子类和本身（与继承关系一至）-T可以传入其父类和本身（与继承的关系相反）
>
> 6. `T:Ordering` :表示将T变成Ordering[T],可以直接用其方法进行比大小,可完成排序等工作
>
>    **scala的类和方法、函数都可以是泛型**

**高阶函数（Higher-Order Function）就是操作其他函数的函数**。

Scala 中允许使用高阶函数, 高阶函数可以使用其他函数作为参数，或者使用函数作为输出结果。

以下实例中，apply() 函数使用了另外一个函数 f 和 值 v 作为参数，而函数 f 又调用了参数 v：

```scala
object Test {
   def main(args: Array[String]) {

      println( apply( layout, 10) )

   }
   // 函数 f 和 值 v 作为参数，而函数 f 又调用了参数 v
   def apply(f: Int => String, v: Int) = f(v)

   def layout[A](x: A) = "[" + x.toString() + "]"
   
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
[10]
```

### 9.3.10 函数柯里化(Function Currying)

柯里化(Currying)指的是将原来接受两个参数的函数变成新的接受一个参数的函数的过程。

新的函数返回一个以原有第二个参数为参数的函数。

**示例**

首先我们定义一个函数:

```scala
def add(x:Int,y:Int)=x+y
```

那么我们应用的时候，应该是这样用：add(1,2)

现在我们把这个函数变一下形：

```scala
def add(x:Int)(y:Int) = x + y
```

那么我们应用的时候，应该是这样用：add(1)(2),最后结果都一样是3，这种方式（过程）就叫柯里化。

**实现过程**

add(1)(2) 实际上是依次调用两个普通函数（非柯里化函数），第一次调用使用一个参数 x，返回一个函数类型的值，第二次使用参数y调用这个函数类型的值。

实质上最先演变成这样一个方法：

```scala
def add(x:Int)=(y:Int)=>x+y
```

那么这个函数是什么意思呢？ 接收一个x为参数，返回一个匿名函数，该匿名函数的定义是：接收一个Int型参数y，函数体为x+y。现在我们来对这个方法进行调用。

```scala
val result = add(1) 
```

返回一个result，那result的值应该是一个匿名函数：(y:Int)=>1+y

所以为了得到结果，我们继续调用result。

```scala
val sum = result(2)
```

最后打印出来的结果就是3。

**完整示例**

下面是一个完整实例：

```scala
object Test {
   def main(args: Array[String]) {
      val str1:String = "Hello, "
      val str2:String = "Scala!"
      println( "str1 + str2 = " +  strcat(str1)(str2) )
   }

   def strcat(s1: String)(s2: String) = {
      s1 + s2
   }
}
```

执行以上代码，输出结果为：

```shell
$ scalac Test.scala
$ scala Test
str1 + str2 = Hello, Scala!
```

