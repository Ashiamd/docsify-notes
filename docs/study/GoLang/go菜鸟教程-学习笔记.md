# go菜鸟教程-学习笔记

> [Go 语言教程](https://www.runoob.com/go/go-tutorial.html)
>
> 大部分内容是直接摘至教程，主要是记录自己的学习。
>
> 有些地方会额外加上一些自己查找的博客；或者删减、加粗、加下划线等。
>
> 建议学习新东西，都养成自己记笔记的习惯吧。看不看是一回事，主要是自己记录的时候会再过一遍脑子。

# 1. 语言教程

## 1.1 特色

+ 简洁、快速、安全
+ 并行、有趣、开源
+ 内存管理、数组安全、编译迅速

## 1.2 用途

+ Web
+ 游戏服务器
+ ...

## 1.3 第一个Go程序

万物皆可hello world，编写`helloworld.go`文件

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World")
}
```

在当前目录执行`go run helloworld.go`

```shell
$ go run helloworld.go
Hello, World
```

**通过`go build` 命令生成二进制文件**

```shell
$ go build helloworld.go
$ ls
helloworld    helloworld.go
$ ./helloworld          
Hello, World
```

# 2. Go 语言环境安装

# 3. Go 语言结构

## 3.1 拆解Hello World实例

Go 语言的基础组成有以下几个部分：

- 包声明
- 引入包
- 函数
- 变量
- 语句 & 表达式
- 注释

回顾Hello World程序

```go
package main

import "fmt"

func main() {
   /* 这是我的第一个简单的程序 */
   fmt.Println("Hello, World!")
}
```

让我们来看下以上程序的各个部分：

1. 第一行代码 `package main`定义了包名。你必须在源文件中非注释的第一行指明这个文件属于哪个包，如：`package main`。

   **`package main`表示一个可独立执行的程序，每个 Go 应用程序都包含一个名为 main 的包。**

2. 下一行 `import "fmt"` 告诉 Go 编译器这个程序需要使用 fmt 包（的函数，或其他元素），**fmt 包实现了格式化 IO（输入/输出）的函数**。

3. 下一行 `func main()` 是程序开始执行的函数。

   **main 函数是每一个可执行程序所必须包含的，一般来说都是在启动后第一个执行的函数（如果有 init() 函数则会先执行该函数）。**

4. 下一行 /\*...\*/ 是注释，在程序执行时将被忽略。

   单行注释是最常见的注释形式，你可以在任何地方使用以 // 开头的单行注释。

   多行注释也叫块注释，均已以 /* 开头，并以 */ 结尾，且不可以嵌套使用，多行注释一般用于包的文档描述或注释成块的代码片段。

5. 下一行 `fmt.Println(...)` 可以将字符串输出到控制台，并在最后自动增加换行字符 \n。
   使用 fmt.Print("hello, world\n") 可以得到相同的结果。
   Print 和 Println 这两个函数也支持使用变量，如：fmt.Println(arr)。如果没有特别指定，它们会以默认的打印格式将变量 arr 输出到控制台。

6. 当标识符（包括常量、变量、类型、函数名、结构字段等等）以一个大写字母开头，如：Group1，那么使用这种形式的标识符的对象就可以被外部包的代码所使用（客户端程序需要先导入这个包），这被称为导出（像面向对象语言中的 public）；标识符如果以小写字母开头，则对包外是不可见的，但是他们在整个包的内部是可见并且可用的（像面向对象语言中的 protected ）。

   + **标识符大写开头：可被外部包引用（需import导入）。相当于面向对象语言的public**
   + **标识符小写开头：对包外不可见，包内可见。相当于面向对象语言的protected**

## 3.2 执行Go程序

+ ```shell
  # 运行go程序
  go run XXX.go
  ```

+ ```shell
  # 生成可执行的二进制文件
  go build XXXX.go
  ```

**注意**

**需要注意的是 { 不能单独放在一行**，所以以下代码在运行时会产生错误：

```go
package main
import "fmt"
func main() 
{ // 错误，{ 不能在单独的行上
  fmt.Println("Hello, World!")
}
```

# 4. Go 语言基础语法

## 4.1 Go 标记

Go 程序可以由多个标记组成，可以是**关键字，标识符，常量，字符串，符号**。如以下 GO 语句由 6 个标记组成：

```go
fmt.Println("Hello, World!")
```

6 个标记是(每行一个)：

```
1. fmt
2. .
3. Println
4. (
5. "Hello, World!"
6. )
```

## 4.2 行分隔符

在 Go 程序中，一行代表一个语句结束。每个语句不需要像 C 家族中的其它语言一样以分号` ; `结尾，因为这些工作都将由 Go 编译器自动完成。

<u>如果你打算将多个语句写在同一行，它们则必须使用` ; `人为区分，但在实际开发中我们并不鼓励这种做法</u>。

以下为两个语句：

```go
fmt.Println("Hello, World!")
fmt.Println("菜鸟教程：runoob.com")
```

## 4.3 注释

注释不会被编译，每一个包应该有相关注释。

单行注释是最常见的注释形式，你可以在任何地方使用以 // 开头的单行注释。多行注释也叫块注释，均已以 /* 开头，并以 */ 结尾。如：

```
// 单行注释
/*
 Author by 菜鸟教程
 我是多行注释
 */
```

## 4.4 标识符

标识符用来命名变量、类型等程序实体。一个标识符实际上就是一个或是多个字母(A~Z和a~z)数字(0~9)、下划线_组成的序列，但是第一个字符必须是字母或下划线而不能是数字。

+ [A-Za-z_]+(A-Za-z0-9\_)*

以下是有效的标识符：

```
mahesh   kumar   abc   move_name   a_123
myname50   _temp   j   a23b9   retVal
```

以下是无效的标识符：

- 1ab（以数字开头）
- case（Go 语言的关键字）
- a+b（运算符是不允许的）

## 4.5 字符串连接

Go 语言的字符串可以通过 **`+`** 实现：

```go
package main
import "fmt"
func main() {
    fmt.Println("Google" + "Runoob")
}
```

以上实例输出结果为：

```shell
GoogleRunoob
```

## 4.6 关键字

下面列举了 Go 代码中会使用到的 25 个关键字或保留字：

| break    | default     | func   | interface | select |
| -------- | ----------- | ------ | --------- | ------ |
| case     | defer       | go     | map       | struct |
| chan     | else        | goto   | package   | switch |
| const    | fallthrough | if     | range     | type   |
| continue | for         | import | return    | var    |

除了以上介绍的这些关键字，Go 语言还有 36 个预定义标识符：

| append | bool    | byte    | cap     | close  | complex | complex64 | complex128 | uint16  |
| ------ | ------- | ------- | ------- | ------ | ------- | --------- | ---------- | ------- |
| copy   | false   | float32 | float64 | imag   | int     | int8      | int16      | uint32  |
| int32  | int64   | iota    | len     | make   | new     | nil       | panic      | uint64  |
| print  | println | real    | recover | string | true    | uint      | uint8      | uintptr |

程序一般由关键字、常量、变量、运算符、类型和函数组成。

程序中可能会使用到这些分隔符：括号` ()`，中括号` [] `和大括号` {}`。

程序中可能会使用到这些标点符号：`.`、`,`、`;`、`:` 和` …`。

## 4.7 Go 语言的空格

Go 语言中变量的声明必须使用空格隔开，如：

```go
var age int;
```

语句中适当使用空格能让程序更易阅读。

无空格：

```go
fruit=apples+oranges;
```

在变量与运算符间加入空格，程序看起来更加美观，如：

```go
fruit = apples + oranges; 
```

## 4.8 格式化字符串

> 更多内容参见：[Go fmt.Sprintf 格式化字符串](https://www.runoob.com/go/go-fmt-sprintf.html)

Go 语言中使用 **fmt.Sprintf** 格式化字符串并赋值给新串：

```go
package main

import (
    "fmt"
)

func main() {
   // %d 表示整型数字，%s 表示字符串
    var stockcode=123
    var enddate="2020-12-31"
    var url="Code=%d&endDate=%s"
    var target_url=fmt.Sprintf(url,stockcode,enddate)
    fmt.Println(target_url)
}
```

输出结果为：

```shell
Code=123&endDate=2020-12-31
```

# 5. Go 语言数据类型

在 Go 编程语言中，**数据类型用于声明函数和变量**。

数据类型的出现是为了把数据分成所需内存大小不同的数据，编程的时候需要用大数据的时候才需要申请大内存，就可以充分利用内存。

Go 语言按类别有以下几种数据类型：

| 序号 | 类型和描述                                                   |
| :--- | :----------------------------------------------------------- |
| 1    | **布尔型** 布尔型的值只可以是常量 true 或者 false。一个简单的例子：var b bool = true。 |
| 2    | **数字类型** 整型 int 和浮点型 float32、float64，Go 语言支持整型和浮点型数字，并且支持复数，其中<u>位的运算采用补码</u>。 |
| 3    | **字符串类型:** 字符串就是一串固定长度的字符连接起来的字符序列。Go 的字符串是由单个字节连接起来的。<u>Go 语言的字符串的字节使用 UTF-8 编码标识 Unicode 文本</u>。 |
| 4    | **派生类型:** 包括：(a) 指针类型（Pointer）(b) 数组类型(c) 结构化类型(struct)(d) Channel 类型(e) 函数类型(f) 切片类型(g) 接口类型（interface）(h) Map 类型 |

## 5.1 数字类型

Go 也有基于架构的类型，例如：int、uint 和 uintptr。

| 序号 | 类型和描述                                                   |
| :--- | :----------------------------------------------------------- |
| 1    | **uint8** 无符号 8 位整型 (0 到 255)                         |
| 2    | **uint16** 无符号 16 位整型 (0 到 65535)                     |
| 3    | **uint32** 无符号 32 位整型 (0 到 4294967295)                |
| 4    | **uint64** 无符号 64 位整型 (0 到 18446744073709551615)      |
| 5    | **int8** 有符号 8 位整型 (-128 到 127)                       |
| 6    | **int16** 有符号 16 位整型 (-32768 到 32767)                 |
| 7    | **int32** 有符号 32 位整型 (-2147483648 到 2147483647)       |
| 8    | **int64** 有符号 64 位整型 (-9223372036854775808 到 9223372036854775807) |

## 5.2 浮点型

| 序号 | 类型和描述                        |
| :--- | :-------------------------------- |
| 1    | **float32** IEEE-754 32位浮点型数 |
| 2    | **float64** IEEE-754 64位浮点型数 |
| 3    | **complex64** 32 位实数和虚数     |
| 4    | **complex128** 64 位实数和虚数    |

## 5.3 其他数字类型

以下列出了其他更多的数字类型：

| 序号 | 类型和描述                               |
| :--- | :--------------------------------------- |
| 1    | **byte** 类似 uint8                      |
| 2    | **rune** 类似 int32                      |
| 3    | **uint** 32 或 64 位                     |
| 4    | **int** 与 uint 一样大小                 |
| 5    | **uintptr** 无符号整型，用于存放一个指针 |

# 6. Go 语言变量

声明变量的一般形式是使用 var 关键字：

```go
var identifier type
```

可以一次声明多个变量：

```go
var identifier1, identifier2 type
```

```go
package main
import "fmt"
func main() {
    var a string = "Runoob"
    fmt.Println(a)

    var b, c int = 1, 2
    fmt.Println(b, c)
}
```

以上实例输出结果为：

```shell
Runoob
1 2
```

## 6.1 变量声明

**第一种，指定变量类型，如果没有初始化，则变量默认为零值**。

```go
var v_name v_type
v_name = value
```

**零值就是变量没有做初始化时系统默认设置的值**。

- 数值类型（包括complex64/128）为 **0**
- 布尔类型为 **false**
- 字符串为 **""**（空字符串）
- 以下几种类型为 **nil**：

```go
var a *int
var a []int
var a map[string] int
var a chan int
var a func(string) int
var a error // error 是接口
```

下面演示程序

```go
package main
import "fmt"
func main() {

    // 声明一个变量并初始化
    var a = "RUNOOB"
    fmt.Println(a)

    // 没有初始化就为零值
    // 改成 var b 则编译时报错
    // syntax error: unexpected newline, expecting type
    var b int
    fmt.Println(b)

    // bool 零值为 false
    var c bool
    fmt.Println(c)
}
```

以上实例执行结果为：

```shell
RUNOOB
0
false
```

下面演示程序

> [Golang Printf、Sprintf 、Fprintf 格式化详细对比](https://blog.csdn.net/Chen_Jeep/article/details/103610041)
>
> + %v  按值的本来值输出
>
> + %q 源代码中那样带有双引号的输出   "\\"string\\""  对应 "\\"string\\""
>
>   `fmt.Printf("%q\n","\"string\"") // "\"string\""  对应 "string"`

```go
package main

import "fmt"

func main() {
    var i int
    var f float64
    var b bool
    var s string
    fmt.Printf("%v %v %v %q\n", i, f, b, s)
}
```

输出结果是：

```shell
0 0 false ""
```

**第二种，根据值自行判定变量类型。**

```go
var v_name = value
```

实例代码：

```go
package main
import "fmt"
func main() {
    var d = true
    fmt.Println(d)
}
```

输出结果如下：

```shell
true
```

**第三种，省略 var, 注意 `:=` 左侧如果没有声明新的变量，就产生编译错误，格式：**

```go
v_name := value
```

例如：

```go
var intVal int 
intVal :=1 // 这时候会产生编译错误，因为 intVal 已经声明，不需要重新声明
```

直接使用下面的语句即可：

```go
intVal := 1 // 此时不会产生编译错误，因为有声明新的变量，因为 := 是一个声明语句
```

**`intVal := 1`** 相等于：

```go
var intVal int 
intVal =1 
```

可以将 var f string = "Runoob" 简写为 f := "Runoob"：

```go
package main
import "fmt"
func main() {
    f := "Runoob" // var f string = "Runoob"

    fmt.Println(f)
}
```

输出如下：

```shell
Runoob
```

## 6.2 多变量声明

```go
//类型相同多个变量, 非全局变量
var vname1, vname2, vname3 type
vname1, vname2, vname3 = v1, v2, v3

var vname1, vname2, vname3 = v1, v2, v3 // 和 python 很像,不需要显示声明类型，自动推断

vname1, vname2, vname3 := v1, v2, v3 // 出现在 := 左侧的变量不应该是已经被声明过的，否则会导致编译错误


// 这种因式分解关键字的写法一般用于声明全局变量
var (
    vname1 v_type1
    vname2 v_type2
)
```

代码示例：

```go
package main

var x, y int
var (  // 这种因式分解关键字的写法一般用于声明全局变量
    a int
    b bool
)

var c, d int = 1, 2
var e, f = 123, "hello"

//这种不带声明格式的只能在函数体中出现
//g, h := 123, "hello"

func main(){
    g, h := 123, "hello"
    println(x, y, a, b, c, d, e, f, g, h)
}
```

以上实例执行结果为：

```shell
0 0 0 false 1 2 123 hello 123 hello
```

## 6.3 值类型和引用类型

所有像 int、float、bool 和 **string** 这些基本类型都属于**值类型**，使用这些类型的变量直接指向存在内存中的值：

![4.4.2_fig4.1](https://www.runoob.com/wp-content/uploads/2015/06/4.4.2_fig4.1.jpgrawtrue)

当使用等号 `=` 将一个变量的值赋值给另一个变量时，如：`j = i`，实际上是在内存中将 i 的值进行了拷贝：

![4.4.2_fig4.2](https://www.runoob.com/wp-content/uploads/2015/06/4.4.2_fig4.2.jpgrawtrue)

你可以**通过 &i 来获取变量 i 的内存地址**，例如：0xf840000040（每次的地址都可能不一样）。**值类型的变量的值存储在栈中**。

内存地址会根据机器的不同而有所不同，甚至相同的程序在不同的机器上执行后也会有不同的内存地址。因为每台机器可能有不同的存储器布局，并且位置分配也可能不同。

更复杂的数据通常会需要使用多个字，这些数据一般使用引用类型保存。

**一个引用类型的变量 r1 存储的是 r1 的值所在的内存地址（数字），或内存地址中第一个字所在的位置。**

![4.4.2_fig4.3](https://www.runoob.com/wp-content/uploads/2015/06/4.4.2_fig4.3.jpgrawtrue)

这个内存地址称之为指针，这个指针实际上也被存在另外的某一个值中。

同一个引用类型的指针指向的多个字可以是在连续的内存地址中（内存布局是连续的），这也是计算效率最高的一种存储形式；也可以将这些字分散存放在内存中，每个字都指示了下一个字所在的内存地址。

当使用赋值语句 r2 = r1 时，只有引用（地址）被复制。

如果 r1 的值被改变了，那么这个值的所有引用都会指向被修改后的内容，在这个例子中，r2 也会受到影响。

## 6.4 简短形式，使用 := 赋值操作符

### 要点个人小结

+ `:=`只能用在第一次声明，其相当于"声明+初始化"的语法糖，称为"初始化声明"

+ **go要求局部变量必须被使用，否则编译错误；全局变量则允许只声明不使用**

+ `a, b, c := 5, 7, "abc"`这种操作，称为"并行赋值"。

+ **`a,b = b,a`可以完成a和b的值交换(前提类型相同)**

+ **`_`是只写变量**。

  这样做是因为 Go 语言中你必须使用所有被声明的变量，但有时你并不需要使用从一个函数得到的所有返回值

+ **Go允许函数有多个返回值**。

  并行赋值也被用于当一个函数返回多个返回值时，比如这里的 val 和错误 err 是通过调用 Func1 函数同时得到：val, err = Func1(var1)

---

我们知道可以在变量的初始化时省略变量的类型而由系统自动推断，声明语句写上 var 关键字其实是显得有些多余了，因此我们可以将它们简写为 a := 50 或 b := false。

a 和 b 的类型（int 和 bool）将由编译器自动推断。

这是使用变量的**首选形式**，但是它只能被用在函数体内，而不可以用于全局变量的声明与赋值。

使用操作符 := 可以高效地创建一个新的变量，称之为**初始化声明**。

**注意事项**

如果<u>在相同的代码块中，我们不可以再次对于相同名称的变量使用**初始化声明**</u>，例如：a := 20 就是不被允许的，编译器会提示错误 no new variables on left side of :=，但是 a = 20 是可以的，因为这是给相同的变量赋予一个新的值。

如果你在定义变量 a 之前使用它，则会得到编译错误 undefined: a。

**如果你声明了一个局部变量却没有在相同的代码块中使用它，同样会得到编译错误**，例如下面这个例子当中的变量 a：

```go
package main

import "fmt"

func main() {
   var a string = "abc"
   fmt.Println("hello, world")
}
```

尝试编译这段代码将得到错误 **a declared and not used**。

此外，单纯地给 a 赋值也是不够的，这个值必须被使用，所以使用

```go
fmt.Println("hello, world", a)
```

会移除错误。

但是**全局变量是允许声明但不使用**的。 同一类型的多个变量可以声明在同一行，如：

```go
var a, b, c int
```

多变量可以在同一行进行赋值，如：

```go
var a, b int
var c string
a, b, c = 5, 7, "abc"
```

上面这行假设了变量 a，b 和 c 都已经被声明，否则的话应该这样使用：

```go
a, b, c := 5, 7, "abc"
```

右边的这些值以相同的顺序赋值给左边的变量，所以 a 的值是 5， b 的值是 7，c 的值是 "abc"。

**这被称为 并行 或 同时 赋值**。

如果你想要交换两个变量的值，则可以简单地使用 **a, b = b, a**，两个变量的类型必须是相同。

**空白标识符 _ 也被用于抛弃值，如值 5 在：_, b = 5, 7 中被抛弃**。

**_ 实际上是一个只写变量，你不能得到它的值**。<u>这样做是因为 Go 语言中你必须使用所有被声明的变量，但有时你并不需要使用从一个函数得到的所有返回值</u>。

<u>并行赋值也被用于当一个函数返回多个返回值时，比如这里的 val 和错误 err 是通过调用 Func1 函数同时得到：val, err = Func1(var1)</u>。

# 7. Go 语言常量

常量是一个简单值的标识符，在程序运行时，不会被修改的量。

常量中的数据类型<u>只可以是布尔型、数字型（整数型、浮点型和**复数**）和字符串型</u>。

常量的定义格式：

```go
const identifier [type] = value
```

你可以省略类型说明符 [type]，因为编译器可以根据变量的值来推断其类型。

- 显式类型定义： `const b string = "abc"`
- 隐式类型定义： `const b = "abc"`

多个**相同类型**的声明可以简写为：

```go
const c_name1, c_name2 = value1, value2
```

以下实例演示了常量的应用：

```go
package main

import "fmt"

func main() {
   const LENGTH int = 10
   const WIDTH int = 5  
   var area int
   const a, b, c = 1, false, "str" //多重赋值

   area = LENGTH * WIDTH
   fmt.Printf("面积为 : %d", area)
   println()
   println(a, b, c)  
}
```

以上实例运行结果为：

```shell
面积为 : 50
1 false str
```

常量还可以用作**枚举**：

```go
const (
    Unknown = 0
    Female = 1
    Male = 2
)
```

数字 0、1 和 2 分别代表未知性别、女性和男性。

常量可以用`len()`，` cap()`， `unsafe.Sizeof()`函数计算表达式的值。

**常量表达式中，函数必须是内置函数，否则编译不过**：

```go
package main

import "unsafe"
const (
    a = "abc"
    b = len(a)
    c = unsafe.Sizeof(a)
)

func main(){
    println(a, b, c)
}
```

以上实例运行结果为：

```shell
abc 3 16
```

> 下方评论区留言：
>
> 输出结果为：16
>
> **字符串类型在 go 里是个结构, 包含指向底层数组的指针和长度**,这两部分每部分都是 8 个字节，所以字符串类型大小为 16 个字节。
>
> ----
>
> **在定义常量组时，如果不提供初始值，则表示将使用上行的表达式。**

## 7.1 iota

**iota，特殊常量，可以认为是一个可以被编译器修改的常量**。

**iota 在 const关键字出现时将被重置为 0(const 内部的第一行之前)，const 中每新增一行常量声明将使 iota 计数一次(iota 可理解为 const 语句块中的行索引)**。

iota 可以被用作枚举值：

```go
const (
    a = iota
    b = iota
    c = iota
)
```

**第一个 iota 等于 0，每当 iota 在新的一行被使用时，它的值都会自动加 1**；所以 a=0, b=1, c=2 可以简写为如下形式：

```go
const (
    a = iota
    b
    c
)
```

### iota 用法

```go
package main

import "fmt"

func main() {
    const (
            a = iota   //0
            b          //1
            c          //2
            d = "ha"   //独立值，iota += 1
            e          //"ha"   iota += 1
            f = 100    //iota +=1
            g          //100  iota +=1
            h = iota   //7,恢复计数
            i          //8
    )
    fmt.Println(a,b,c,d,e,f,g,h,i)
}
```

以上实例运行结果为：

```shell
0 1 2 ha ha 100 100 7 8
```

再看个有趣的的 iota 实例：

```go
package main

import "fmt"
const (
    i=1<<iota
    j=3<<iota
    k
    l
)

func main() {
    fmt.Println("i=",i)
    fmt.Println("j=",j)
    fmt.Println("k=",k)
    fmt.Println("l=",l)
}
```

以上实例运行结果为：

```shell
i= 1
j= 6
k= 12
l= 24
```

iota 表示从 0 开始自动加 1，所以 **i=1<<0**, **j=3<<1**（**<<** 表示左移的意思），即：i=1, j=6，这没问题，关键在 k 和 l，从输出结果看 **k=3<<2**，**l=3<<3**。

简单表述: 

+ **i=1**：左移 0 位,不变仍为 1;

- **j=3**：左移 1 位,变为二进制 110, 即 6;
- **k=3**：左移 2 位,变为二进制 1100, 即 12;
- **l=3**：左移 3 位,变为二进制 11000,即 24。

注：**<<n==\*(2^n)**。

> 下方评论留言：
>
> **iota 只是在同一个 const 常量组内递增，每当有新的 const 关键字时，iota 计数会重新开始。**
>
> ```go
> package main
> 
> const (
>     i = iota
>     j = iota
>     x = iota
> )
> const xx = iota
> const yy = iota
> func main(){
>     println(i, j, x, xx, yy)
> }
> 
> // 输出是 0 1 2 0 0
> ```

# 8. Go 语言运算符

运算符用于在程序运行时执行数学或逻辑运算。

Go 语言内置的运算符有：

- 算术运算符
- 关系运算符
- 逻辑运算符
- 位运算符
- 赋值运算符
- 其他运算符

接下来让我们来详细看看各个运算符的介绍。

## 8.1 算数运算符

下表列出了所有Go语言的算术运算符。假定 A 值为 10，B 值为 20。

| 运算符 | 描述 | 实例               |
| :----- | :--- | :----------------- |
| +      | 相加 | A + B 输出结果 30  |
| -      | 相减 | A - B 输出结果 -10 |
| *      | 相乘 | A * B 输出结果 200 |
| /      | 相除 | B / A 输出结果 2   |
| %      | 求余 | B % A 输出结果 0   |
| ++     | 自增 | A++ 输出结果 11    |
| --     | 自减 | A-- 输出结果 9     |

以下实例演示了各个算术运算符的用法：

```go
package main

import "fmt"

func main() {

   var a int = 21
   var b int = 10
   var c int

   c = a + b
   fmt.Printf("第一行 - c 的值为 %d\n", c )
   c = a - b
   fmt.Printf("第二行 - c 的值为 %d\n", c )
   c = a * b
   fmt.Printf("第三行 - c 的值为 %d\n", c )
   c = a / b
   fmt.Printf("第四行 - c 的值为 %d\n", c )
   c = a % b
   fmt.Printf("第五行 - c 的值为 %d\n", c )
   a++
   fmt.Printf("第六行 - a 的值为 %d\n", a )
   a=21   // 为了方便测试，a 这里重新赋值为 21
   a--
   fmt.Printf("第七行 - a 的值为 %d\n", a )
}
```

以上实例运行结果：

```shell
第一行 - c 的值为 31
第二行 - c 的值为 11
第三行 - c 的值为 210
第四行 - c 的值为 2
第五行 - c 的值为 1
第六行 - a 的值为 22
第七行 - a 的值为 20
```

> 下方评论留言
>
> **Go 的自增，自减只能作为表达式使用，而不能用于赋值语句**。
>
> ```
> a++ // 这是允许的，类似 a = a + 1,结果与 a++ 相同
> a-- //与 a++ 相似
> a = a++ // 这是不允许的，会出现编译错误 syntax error: unexpected ++ at end of statement
> ```

## 8.2 关系运算符

下表列出了所有Go语言的关系运算符。假定 A 值为 10，B 值为 20。

| 运算符 | 描述                                                         | 实例              |
| :----- | :----------------------------------------------------------- | :---------------- |
| ==     | 检查两个值是否相等，如果相等返回 True 否则返回 False。       | (A == B) 为 False |
| !=     | 检查两个值是否不相等，如果不相等返回 True 否则返回 False。   | (A != B) 为 True  |
| >      | 检查左边值是否大于右边值，如果是返回 True 否则返回 False。   | (A > B) 为 False  |
| <      | 检查左边值是否小于右边值，如果是返回 True 否则返回 False。   | (A < B) 为 True   |
| >=     | 检查左边值是否大于等于右边值，如果是返回 True 否则返回 False。 | (A >= B) 为 False |
| <=     | 检查左边值是否小于等于右边值，如果是返回 True 否则返回 False。 | (A <= B) 为 True  |

以下实例演示了关系运算符的用法：

```go
package main

import "fmt"

func main() {
   var a int = 21
   var b int = 10

   if( a == b ) {
      fmt.Printf("第一行 - a 等于 b\n" )
   } else {
      fmt.Printf("第一行 - a 不等于 b\n" )
   }
   if ( a < b ) {
      fmt.Printf("第二行 - a 小于 b\n" )
   } else {
      fmt.Printf("第二行 - a 不小于 b\n" )
   }
   
   if ( a > b ) {
      fmt.Printf("第三行 - a 大于 b\n" )
   } else {
      fmt.Printf("第三行 - a 不大于 b\n" )
   }
   /* Lets change value of a and b */
   a = 5
   b = 20
   if ( a <= b ) {
      fmt.Printf("第四行 - a 小于等于 b\n" )
   }
   if ( b >= a ) {
      fmt.Printf("第五行 - b 大于等于 a\n" )
   }
}
```

以上实例运行结果：

```shell
第一行 - a 不等于 b
第二行 - a 不小于 b
第三行 - a 大于 b
第四行 - a 小于等于 b
第五行 - b 大于等于 a
```

## 8.3 逻辑运算符

下表列出了所有Go语言的逻辑运算符。假定 A 值为 True，B 值为 False。

| 运算符 | 描述                                                         | 实例               |
| :----- | :----------------------------------------------------------- | :----------------- |
| &&     | 逻辑 AND 运算符。 如果两边的操作数都是 True，则条件 True，否则为 False。 | (A && B) 为 False  |
| \|\|   | 逻辑 OR 运算符。 如果两边的操作数有一个 True，则条件 True，否则为 False。 | (A \|\| B) 为 True |
| !      | 逻辑 NOT 运算符。 如果条件为 True，则逻辑 NOT 条件 False，否则为 True。 | !(A && B) 为 True  |

以下实例演示了逻辑运算符的用法：

```go
package main

import "fmt"

func main() {
   var a bool = true
   var b bool = false
   if ( a && b ) {
      fmt.Printf("第一行 - 条件为 true\n" )
   }
   if ( a || b ) {
      fmt.Printf("第二行 - 条件为 true\n" )
   }
   /* 修改 a 和 b 的值 */
   a = false
   b = true
   if ( a && b ) {
      fmt.Printf("第三行 - 条件为 true\n" )
   } else {
      fmt.Printf("第三行 - 条件为 false\n" )
   }
   if ( !(a && b) ) {
      fmt.Printf("第四行 - 条件为 true\n" )
   }
}
```

以上实例运行结果：

```shell
第二行 - 条件为 true
第三行 - 条件为 false
第四行 - 条件为 true
```

## 8.4 位运算符

位运算符对整数在内存中的二进制位进行操作。

下表列出了位运算符 &, |, 和 ^ 的计算：

| p    | q    | p & q | p \| q | p ^ q |
| :--- | :--- | :---- | :----- | :---- |
| 0    | 0    | 0     | 0      | 0     |
| 0    | 1    | 0     | 1      | 1     |
| 1    | 1    | 1     | 1      | 0     |
| 1    | 0    | 0     | 1      | 1     |

假定 A = 60; B = 13; 其二进制数转换为：

```none
A = 0011 1100

B = 0000 1101

-----------------

A&B = 0000 1100

A|B = 0011 1101

A^B = 0011 0001
```

Go 语言支持的位运算符如下表所示。假定 A 为60，B 为13：

| 运算符 | 描述                                                         | 实例                                   |
| :----- | :----------------------------------------------------------- | :------------------------------------- |
| &      | 按位与运算符"&"是双目运算符。 其功能是参与运算的两数各对应的二进位相与。 | (A & B) 结果为 12, 二进制为 0000 1100  |
| \|     | 按位或运算符"\|"是双目运算符。 其功能是参与运算的两数各对应的二进位相或 | (A \| B) 结果为 61, 二进制为 0011 1101 |
| ^      | 按位异或运算符"^"是双目运算符。 其功能是参与运算的两数各对应的二进位相异或，当两对应的二进位相异时，结果为1。 | (A ^ B) 结果为 49, 二进制为 0011 0001  |
| <<     | 左移运算符"<<"是双目运算符。左移n位就是乘以2的n次方。 其功能把"<<"左边的运算数的各二进位全部左移若干位，由"<<"右边的数指定移动的位数，高位丢弃，低位补0。 | A << 2 结果为 240 ，二进制为 1111 0000 |
| >>     | 右移运算符">>"是双目运算符。右移n位就是除以2的n次方。 其功能是把">>"左边的运算数的各二进位全部右移若干位，">>"右边的数指定移动的位数。 | A >> 2 结果为 15 ，二进制为 0000 1111  |

以下实例演示了位运算符的用法：

```go
package main

import "fmt"

func main() {

   var a uint = 60      /* 60 = 0011 1100 */  
   var b uint = 13      /* 13 = 0000 1101 */
   var c uint = 0          

   c = a & b       /* 12 = 0000 1100 */
   fmt.Printf("第一行 - c 的值为 %d\n", c )

   c = a | b       /* 61 = 0011 1101 */
   fmt.Printf("第二行 - c 的值为 %d\n", c )

   c = a ^ b       /* 49 = 0011 0001 */
   fmt.Printf("第三行 - c 的值为 %d\n", c )

   c = a << 2     /* 240 = 1111 0000 */
   fmt.Printf("第四行 - c 的值为 %d\n", c )

   c = a >> 2     /* 15 = 0000 1111 */
   fmt.Printf("第五行 - c 的值为 %d\n", c )
}
```

以上实例运行结果：

```shell
第一行 - c 的值为 12
第二行 - c 的值为 61
第三行 - c 的值为 49
第四行 - c 的值为 240
第五行 - c 的值为 15
```

## 8.5 赋值运算符

下表列出了所有Go语言的赋值运算符。

| 运算符 | 描述                                           | 实例                                  |
| :----- | :--------------------------------------------- | :------------------------------------ |
| =      | 简单的赋值运算符，将一个表达式的值赋给一个左值 | C = A + B 将 A + B 表达式结果赋值给 C |
| +=     | 相加后再赋值                                   | C += A 等于 C = C + A                 |
| -=     | 相减后再赋值                                   | C -= A 等于 C = C - A                 |
| *=     | 相乘后再赋值                                   | C *= A 等于 C = C * A                 |
| /=     | 相除后再赋值                                   | C /= A 等于 C = C / A                 |
| %=     | 求余后再赋值                                   | C %= A 等于 C = C % A                 |
| <<=    | 左移后赋值                                     | C <<= 2 等于 C = C << 2               |
| >>=    | 右移后赋值                                     | C >>= 2 等于 C = C >> 2               |
| &=     | 按位与后赋值                                   | C &= 2 等于 C = C & 2                 |
| ^=     | 按位异或后赋值                                 | C ^= 2 等于 C = C ^ 2                 |
| \|=    | 按位或后赋值                                   | C \|= 2 等于 C = C \| 2               |

以下实例演示了赋值运算符的用法：

```go
package main

import "fmt"

func main() {
   var a int = 21
   var c int

   c =  a
   fmt.Printf("第 1 行 - =  运算符实例，c 值为 = %d\n", c )

   c +=  a
   fmt.Printf("第 2 行 - += 运算符实例，c 值为 = %d\n", c )

   c -=  a
   fmt.Printf("第 3 行 - -= 运算符实例，c 值为 = %d\n", c )

   c *=  a
   fmt.Printf("第 4 行 - *= 运算符实例，c 值为 = %d\n", c )

   c /=  a
   fmt.Printf("第 5 行 - /= 运算符实例，c 值为 = %d\n", c )

   c  = 200;

   c <<=  2
   fmt.Printf("第 6行  - <<= 运算符实例，c 值为 = %d\n", c )

   c >>=  2
   fmt.Printf("第 7 行 - >>= 运算符实例，c 值为 = %d\n", c )

   c &=  2
   fmt.Printf("第 8 行 - &= 运算符实例，c 值为 = %d\n", c )

   c ^=  2
   fmt.Printf("第 9 行 - ^= 运算符实例，c 值为 = %d\n", c )

   c |=  2
   fmt.Printf("第 10 行 - |= 运算符实例，c 值为 = %d\n", c )

}
```

以上实例运行结果：

```shell
第 1 行 - =  运算符实例，c 值为 = 21
第 2 行 - += 运算符实例，c 值为 = 42
第 3 行 - -= 运算符实例，c 值为 = 21
第 4 行 - *= 运算符实例，c 值为 = 441
第 5 行 - /= 运算符实例，c 值为 = 21
第 6行  - <<= 运算符实例，c 值为 = 800
第 7 行 - >>= 运算符实例，c 值为 = 200
第 8 行 - &= 运算符实例，c 值为 = 0
第 9 行 - ^= 运算符实例，c 值为 = 2
第 10 行 - |= 运算符实例，c 值为 = 2
```

## 8.6 其他运算符

下表列出了Go语言的其他运算符。

| 运算符 | 描述             | 实例                       |
| :----- | :--------------- | :------------------------- |
| &      | 返回变量存储地址 | &a; 将给出变量的实际地址。 |
| *      | 指针变量。       | *a; 是一个指针变量         |

以下实例演示了其他运算符的用法：

```go
package main

import "fmt"

func main() {
   var a int = 4
   var b int32
   var c float32
   var ptr *int

   /* 运算符实例 */
   fmt.Printf("第 1 行 - a 变量类型为 = %T\n", a );
   fmt.Printf("第 2 行 - b 变量类型为 = %T\n", b );
   fmt.Printf("第 3 行 - c 变量类型为 = %T\n", c );

   /*  & 和 * 运算符实例 */
   ptr = &a     /* 'ptr' 包含了 'a' 变量的地址 */
   fmt.Printf("a 的值为  %d\n", a);
   fmt.Printf("*ptr 为 %d\n", *ptr);
}
```

以上实例运行结果：

```shell
第 1 行 - a 变量类型为 = int
第 2 行 - b 变量类型为 = int32
第 3 行 - c 变量类型为 = float32
a 的值为  4
*ptr 为 4
```

## 8.7 运算符优先级

有些运算符拥有较高的优先级，**二元运算符的运算方向均是从左至右**。下表列出了所有运算符以及它们的优先级，由上至下代表优先级由高到低：

| 优先级 | 运算符           |
| :----- | :--------------- |
| 5      | * / % << >> & &^ |
| 4      | + - \| ^         |
| 3      | == != < <= > >=  |
| 2      | &&               |
| 1      | \|\|             |

当然，你可以通过使用括号来临时提升某个表达式的整体运算优先级。

```go
package main

import "fmt"

func main() {
   var a int = 20
   var b int = 10
   var c int = 15
   var d int = 5
   var e int;

   e = (a + b) * c / d;      // ( 30 * 15 ) / 5
   fmt.Printf("(a + b) * c / d 的值为 : %d\n",  e );

   e = ((a + b) * c) / d;    // (30 * 15 ) / 5
   fmt.Printf("((a + b) * c) / d 的值为  : %d\n" ,  e );

   e = (a + b) * (c / d);   // (30) * (15/5)
   fmt.Printf("(a + b) * (c / d) 的值为  : %d\n",  e );

   e = a + (b * c) / d;     //  20 + (150/5)
   fmt.Printf("a + (b * c) / d 的值为  : %d\n" ,  e );  
}
```

以上实例运行结果：

```shell
(a + b) * c / d 的值为 : 90
((a + b) * c) / d 的值为  : 90
(a + b) * (c / d) 的值为  : 90
a + (b * c) / d 的值为  : 50
```

# 9. Go 语言条件语句

条件语句需要开发者通过指定一个或多个条件，并通过测试条件是否为 true 来决定是否执行指定语句，并在条件为 false 的情况在执行另外的语句。

下图展示了程序语言中条件语句的结构：

![img](https://www.runoob.com/wp-content/uploads/2015/06/ZBuVRsKmCoH6fzoz.png)

Go 语言提供了以下几种条件判断语句：

| 语句                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [if 语句](https://www.runoob.com/go/go-if-statement.html)    | **if 语句** 由一个布尔表达式后紧跟一个或多个语句组成。       |
| [if...else 语句](https://www.runoob.com/go/go-if-else-statement.html) | **if 语句** 后可以使用可选的 **else 语句**, else 语句中的表达式在布尔表达式为 false 时执行。 |
| [if 嵌套语句](https://www.runoob.com/go/go-nested-if-statements.html) | 你可以在 **if** 或 **else if** 语句中嵌入一个或多个 **if** 或 **else if** 语句。 |
| [switch 语句](https://www.runoob.com/go/go-switch-statement.html) | **switch** 语句用于基于不同条件执行不同动作。                |
| [select 语句](https://www.runoob.com/go/go-select-statement.html) | **select** 语句类似于 **switch** 语句，但是select会随机执行一个可运行的case。如果没有case可运行，它将阻塞，直到有case可运行。 |

> **注意：Go 没有三目运算符，所以不支持 ?: 形式的条件判断。**

## 9.1 if 语句

if 语句由布尔表达式后紧跟一个或多个语句组成。

**语法**

Go 编程语言中 if 语句的语法如下：

```
if 布尔表达式 {
   /* 在布尔表达式为 true 时执行 */
}
```

If 在布尔表达式为 true 时，其后紧跟的语句块执行，如果为 false 则不执行。

流程图如下：

![Go 语言 if 语句](https://www.runoob.com/wp-content/uploads/2015/06/G2XHghEwpDrpgWcg__original.png)

使用 if 判断一个数变量的大小：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 10
 
   /* 使用 if 语句判断布尔表达式 */
   if a < 20 {
       /* 如果条件为 true 则执行以下语句 */
       fmt.Printf("a 小于 20\n" )
   }
   fmt.Printf("a 的值为 : %d\n", a)
}
```

以上代码执行结果为：

```shell
a 小于 20
a 的值为 : 10
```

> 下方评论留言
>
> **if 语句使用 tips**：
>
> **（1）** 不需使用括号将条件包含起来
>
> **（2）** 大括号{}必须存在，即使只有一行语句
>
> **（3）** 左括号必须在if或else的同一行
>
> **（4）** 在if之后，条件语句之前，可以添加变量初始化语句，使用；进行分隔
>
> **（5）** 在有返回值的函数中，最终的return不能在条件语句中
>
> ---
>
> Go 的 if 还有一个强大的地方就是条件判断语句里面**允许声明一个变量**(试了一下，声明两个就报错)，这个变量的作用域只能在该条件逻辑块内，其他地方就不起作用了，如下所示:
>
> ```go
> package main
> 
> import "fmt"
> func main() {
>   if num := 9; num < 0 {
>     fmt.Println(num, "is negative")
>   } else if num < 10 {
>     fmt.Println(num, "has 1 digit")
>   } else {
>     fmt.Println(num, "has multiple digits")
>   }
>   // fmt.Println(num) // undefined: num
> }
> ```
>
> 运行结果：
>
> ```shell
> 9 has 1 digit
> ```

## 9.2 if...else 语句

if 语句 后可以使用可选的 else 语句, else 语句中的表达式在布尔表达式为 false 时执行。

**语法**

Go 编程语言中 if...else 语句的语法如下：

```
if 布尔表达式 {
   /* 在布尔表达式为 true 时执行 */
} else {
  /* 在布尔表达式为 false 时执行 */
}
```

If 在布尔表达式为 true 时，其后紧跟的语句块执行，如果为 false 则执行 else 语句块。

流程图如下：

![img](https://www.runoob.com/wp-content/uploads/2015/06/bTaq6OdDxlnPXWDv.png)

使用 if else 判断一个数的大小：

```go
package main

import "fmt"

func main() {
   /* 局部变量定义 */
   var a int = 100;
 
   /* 判断布尔表达式 */
   if a < 20 {
       /* 如果条件为 true 则执行以下语句 */
       fmt.Printf("a 小于 20\n" );
   } else {
       /* 如果条件为 false 则执行以下语句 */
       fmt.Printf("a 不小于 20\n" );
   }
   fmt.Printf("a 的值为 : %d\n", a);

}
```

以上代码执行结果为：

```shell
a 不小于 20
a 的值为 : 100
```

## 9.3 if 嵌套语句

你可以在 if 或 else if 语句中嵌入一个或多个 if 或 else if 语句。

**语法**

Go 编程语言中 if...else 语句的语法如下：

```go
if 布尔表达式 1 {
   /* 在布尔表达式 1 为 true 时执行 */
   if 布尔表达式 2 {
      /* 在布尔表达式 2 为 true 时执行 */
   }
}
```

你可以以同样的方式在 if 语句中嵌套 **else if...else** 语句

**实例**

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 100
   var b int = 200
 
   /* 判断条件 */
   if a == 100 {
       /* if 条件语句为 true 执行 */
       if b == 200 {
          /* if 条件语句为 true 执行 */
          fmt.Printf("a 的值为 100 ， b 的值为 200\n" );
       }
   }
   fmt.Printf("a 值为 : %d\n", a );
   fmt.Printf("b 值为 : %d\n", b );
}
```

以上代码执行结果为：

```shell
a 的值为 100 ， b 的值为 200
a 值为 : 100
b 值为 : 200
```

## 9.4 switch 语句

switch 语句用于基于不同条件执行不同动作，每一个 case 分支都是唯一的，从上至下逐一测试，直到匹配为止。

switch 语句执行的过程从上至下，直到找到匹配项，匹配项后面也不需要再加 break。

**switch 默认情况下 case 最后自带 break 语句**，匹配成功后就不会执行其他 case，如果我们需要执行后面的 case，可以使用 **fallthrough** 。

**语句**

Go 编程语言中 switch 语句的语法如下：

```
switch var1 {
    case val1:
        ...
    case val2:
        ...
    default:
        ...
}
```

变量 var1 可以是**任何类型**，而 val1 和 val2 则可以是**同类型**的任意值。类型不被局限于常量或整数，但必须是相同的类型；或者最终结果为相同类型的表达式。

您**可以同时测试多个可能符合条件的值，使用逗号分割它们，例如：case val1, val2, val3**。

流程图：

![img](https://www.runoob.com/wp-content/uploads/2015/06/JhOEChInS6HWjqCV.png)

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var grade string = "B"
   var marks int = 90

   switch marks {
      case 90: grade = "A"
      case 80: grade = "B"
      case 50,60,70 : grade = "C"
      default: grade = "D"  
   }

   switch {
      case grade == "A" :
         fmt.Printf("优秀!\n" )    
      case grade == "B", grade == "C" :
         fmt.Printf("良好\n" )      
      case grade == "D" :
         fmt.Printf("及格\n" )      
      case grade == "F":
         fmt.Printf("不及格\n" )
      default:
         fmt.Printf("差\n" );
   }
   fmt.Printf("你的等级是 %s\n", grade );      
}
```

以上代码执行结果为：

```shell
优秀!
你的等级是 A
```

### Type switch

**switch 语句还可以被用于 type-switch 来判断某个 interface 变量中实际存储的变量类型**。

Type Switch 语法格式如下：

```go
switch x.(type){
    case type:
       statement(s);      
    case type:
       statement(s); 
    /* 你可以定义任意个数的case */
    default: /* 可选 */
       statement(s);
}
```

代码样例

```go
package main

import "fmt"

func main() {
   var x interface{}
     
   switch i := x.(type) {
      case nil:  
         fmt.Printf(" x 的类型 :%T",i)                
      case int:  
         fmt.Printf("x 是 int 型")                      
      case float64:
         fmt.Printf("x 是 float64 型")          
      case func(int) float64:
         fmt.Printf("x 是 func(int) 型")                      
      case bool, string:
         fmt.Printf("x 是 bool 或 string 型" )      
      default:
         fmt.Printf("未知型")    
   }  
}
```

以上代码执行结果为：

```
x 的类型 :<nil>
```

### fallthrough

**使用 fallthrough 会强制执行后面的 case 语句，fallthrough 不会判断下一条 case 的表达式结果是否为 true。**

```go
package main

import "fmt"

func main() {

    switch {
    case false:
            fmt.Println("1、case 条件语句为 false")
            fallthrough
    case true:
            fmt.Println("2、case 条件语句为 true")
            fallthrough
    case false:
            fmt.Println("3、case 条件语句为 false")
            fallthrough
    case true:
            fmt.Println("4、case 条件语句为 true")
    case false:
            fmt.Println("5、case 条件语句为 false")
            fallthrough
    default:
            fmt.Println("6、默认 case")
    }
}
```

以上代码执行结果为：

```
2、case 条件语句为 true
3、case 条件语句为 false
4、case 条件语句为 true
```

从以上代码输出的结果可以看出：switch 从第一个判断表达式为 true 的 case 开始执行，如果 case 带有 fallthrough，程序会继续执行下一条 case，且它不会去判断下一个 case 的表达式是否为 true。

> 下方评论区留言
>
> **如果想要执行多个 case，需要使用 fallthrough 关键字，也可用 break 终止。**
>
> ```go
> switch{
>     case 1:
>     ...
>     if(...){
>         break
>     }
> 
>     fallthrough // 此时switch(1)会执行case1和case2，但是如果满足if条件，则只执行case1
> 
>     case 2:
>     ...
>     case 3:
> }
> ```

## 9.5 select 语句

**select 是 Go 中的一个控制结构，类似于用于通信的 switch 语句**。**每个 case 必须是一个通信操作，要么是发送要么是接收。**

select 随机执行一个可运行的 case。如果没有 case 可运行，它将阻塞，直到有 case 可运行。一个默认的子句应该总是可运行的。

**语法**

Go 编程语言中 select 语句的语法如下：

```go
select {
    case communication clause  :
       statement(s);      
    case communication clause  :
       statement(s);
    /* 你可以定义任意数量的 case */
    default : /* 可选 */
       statement(s);
}
```

以下描述了 select 语句的语法：

- 每个 case 都必须是一个通信

- **所有 channel 表达式都会被求值**

- **所有被发送的表达式都会被求值**

- 如果任意某个通信可以进行，它就执行，其他被忽略。

- **如果有多个 case 都可以运行，Select 会随机公平地选出一个执行。其他不会执行。**

  否则：

  1. 如果有 default 子句，则执行该语句。
  2. 如果没有 default 子句，select 将阻塞，直到某个通信可以运行；Go 不会重新对 channel 或值进行求值。

**实例**

select 语句应用演示：

```go
package main

import "fmt"

func main() {
   var c1, c2, c3 chan int
   var i1, i2 int
   select {
      case i1 = <-c1:
         fmt.Printf("received ", i1, " from c1\n")
      case c2 <- i2:
         fmt.Printf("sent ", i2, " to c2\n")
      case i3, ok := (<-c3):  // same as: i3, ok := <-c3
         if ok {
            fmt.Printf("received ", i3, " from c3\n")
         } else {
            fmt.Printf("c3 is closed\n")
         }
      default:
         fmt.Printf("no communication\n")
   }    
}
```

以上代码执行结果为：

```shell
no communication
```

> 下方评论留言（涉及后面章节的内容较多）
>
> select 会循环检测条件，如果有满足则执行并退出，否则一直循环检测。
>
> 示例代码：https://play.golang.org/p/abKvSe-Nn30
>
> ```go
> package main
> 
> import (
>     "fmt"
>     "time"
> )
> 
> func Chann(ch chan int, stopCh chan bool) {
>     var i int
>     i = 10
>     for j := 0; j < 10; j++ {
>         ch <- i
>         time.Sleep(time.Second)
>     }
>     stopCh <- true
> }
> 
> func main() {
> 
>     ch := make(chan int)
>     c := 0
>     stopCh := make(chan bool)
> 
>     go Chann(ch, stopCh)
> 
>     for {
>         select {
>         case c = <-ch:
>             fmt.Println("Receive", c)
>             fmt.Println("channel")
>         case s := <-ch:
>             fmt.Println("Receive", s)
>         case _ = <-stopCh:
>             goto end
>         }
>     }
> end:
> }
> ```

# 10. Go 语言循环语句

在不少实际问题中有许多具有规律性的重复操作，因此在程序中就需要重复执行某些语句。

以下为大多编程语言循环程序的流程图：

![img](https://www.runoob.com/wp-content/uploads/2015/06/go-loops.svg)

Go 语言提供了以下几种类型循环处理语句：

| 循环类型                                                   | 描述                                 |
| :--------------------------------------------------------- | :----------------------------------- |
| [for 循环](https://www.runoob.com/go/go-for-loop.html)     | 重复执行语句块                       |
| [循环嵌套](https://www.runoob.com/go/go-nested-loops.html) | 在 for 循环中嵌套一个或多个 for 循环 |

## for 循环

for 循环是一个循环控制结构，可以执行指定次数的循环。

**语法**

Go 语言的 For 循环有 3 种形式，只有其中的一种使用分号。

和 C 语言的 for 一样：

```go
for init; condition; post { }
```

和 C 的 while 一样：

```go
for condition { }
```

和 C 的 for(;;) 一样：

```go
for { }
```

- init： 一般为赋值表达式，给控制变量赋初值；
- condition： 关系表达式或逻辑表达式，循环控制条件；
- post： 一般为赋值表达式，给控制变量增量或减量。

for语句执行过程如下：

- 1、先对表达式 1 赋初值；
- 2、判别赋值表达式 init 是否满足给定条件，若其值为真，满足循环条件，则执行循环体内语句，然后执行 post，进入第二次循环，再判别 condition；否则判断 condition 的值为假，不满足条件，就终止for循环，执行循环体外语句。

**for 循环的 range 格式可以对 slice、map、数组、字符串等进行迭代循环**。格式如下：

```go
for key, value := range oldMap {
    newMap[key] = value
}
```

for 语句语法流程如下图所示：

![img](https://www.runoob.com/wp-content/uploads/2015/06/PVFUw4TZATYSimWQ.png)

计算 1 到 10 的数字之和：

```go
package main

import "fmt"

func main() {
        sum := 0
        for i := 0; i <= 10; i++ {
                sum += i
        }
        fmt.Println(sum)
}
```

输出结果为：

```
55
```

init 和 post 参数是可选的，我们可以直接省略它，类似 While 语句。

以下实例在 sum 小于 10 的时候计算 sum 自相加后的值：

```go
package main

import "fmt"

func main() {
        sum := 1
        for ; sum <= 10; {
                sum += sum
        }
        fmt.Println(sum)

        // 这样写也可以，更像 While 语句形式
        for sum <= 10{
                sum += sum
        }
        fmt.Println(sum)
}
```

输出结果为：

```
16
16
```

无限循环:

```go
package main

import "fmt"

func main() {
        sum := 0
        for {
            sum++ // 无限循环下去
        }
        fmt.Println(sum) // 无法输出
}
```

要停止无限循环，可以在命令窗口按下**ctrl-c** 。

### For-each range 循环

这种格式的循环可以对字符串、数组、切片等进行迭代输出元素。

```go
package main
import "fmt"

func main() {
  strings := []string{"google", "runoob"}
  for i, s := range strings {
    fmt.Println(i, s)
  }
  numbers := [6]int{1, 2, 3, 5}
  for i,x:= range numbers {
    fmt.Printf("第 %d 位 x 的值 = %d\n", i,x)
  }  
}
```

以上实例运行输出结果为:

```shell
0 google
1 runoob
第 0 位 x 的值 = 1
第 1 位 x 的值 = 2
第 2 位 x 的值 = 3
第 3 位 x 的值 = 5
第 4 位 x 的值 = 0
第 5 位 x 的值 = 0
```

## Go 语言循环嵌套

Go 语言允许用户在循环内使用循环。接下来我们将为大家介绍嵌套循环的使用。

**语法**

以下为 Go 语言嵌套循环的格式：

```go
for [condition |  ( init; condition; increment ) | Range]
{
   for [condition |  ( init; condition; increment ) | Range]
   {
      statement(s);
   }
   statement(s);
}
```

以下实例使用循环嵌套来输出 2 到 100 间的素数：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var i, j int

   for i=2; i < 100; i++ {
      for j=2; j <= (i/j); j++ {
         if(i%j==0) {
            break; // 如果发现因子，则不是素数
         }
      }
      if(j > (i/j)) {
         fmt.Printf("%d  是素数\n", i);
      }
   }  
}
```

以上实例运行输出结果为:

```shell
2  是素数
3  是素数
5  是素数
7  是素数
11  是素数
13  是素数
...
83  是素数
89  是素数
97  是素数
```

## 10.1 循环控制语句

循环控制语句可以控制循环体内语句的执行过程。

GO 语言支持以下几种循环控制语句：

| 控制语句                                                     | 描述                                             |
| :----------------------------------------------------------- | :----------------------------------------------- |
| [break 语句](https://www.runoob.com/go/go-break-statement.html) | 经常用于中断当前 for 循环或跳出 switch 语句      |
| [continue 语句](https://www.runoob.com/go/go-continue-statement.html) | 跳过当前循环的剩余语句，然后继续进行下一轮循环。 |
| [goto 语句](https://www.runoob.com/go/go-goto-statement.html) | 将控制转移到被标记的语句。                       |

### break 语句

Go 语言中 break 语句用于以下两方面：

- 用于循环语句中跳出循环，并开始执行循环之后的语句。
- break 在 switch（开关语句）中在执行一条 case 后跳出语句的作用。（swtich里的case默认最后会有一个break。）
- **在多重循环中，可以用标号 label 标出想 break 的循环**。

**语法**

break 语法格式如下：

```
break;
```

break 语句流程图如下：

![img](https://www.runoob.com/wp-content/uploads/2015/06/6AGB5nw0d43KcDJg.png)

**实例**

在变量 a 大于 15 的时候跳出循环：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 10

   /* for 循环 */
   for a < 20 {
      fmt.Printf("a 的值为 : %d\n", a);
      a++;
      if a > 15 {
         /* 使用 break 语句跳出循环 */
         break;
      }
   }
}
```

以上实例执行结果为：

```
a 的值为 : 10
a 的值为 : 11
a 的值为 : 12
a 的值为 : 13
a 的值为 : 14
a 的值为 : 15
```

以下实例有多重循环，演示了使用标记和不使用标记的区别：

```go
package main

import "fmt"

func main() {

    // 不使用标记
    fmt.Println("---- break ----")
    for i := 1; i <= 3; i++ {
        fmt.Printf("i: %d\n", i)
                for i2 := 11; i2 <= 13; i2++ {
                        fmt.Printf("i2: %d\n", i2)
                        break
                }
        }

    // 使用标记
    fmt.Println("---- break label ----")
    re:
        for i := 1; i <= 3; i++ {
            fmt.Printf("i: %d\n", i)
            for i2 := 11; i2 <= 13; i2++ {
                fmt.Printf("i2: %d\n", i2)
                break re
            }
        }
}
```

以上实例执行结果为：

```
---- break ----
i: 1
i2: 11
i: 2
i2: 11
i: 3
i2: 11
---- break label ----
i: 1
i2: 11    
```

### continue 语句

Go 语言的 continue 语句 有点像 break 语句。但是 continue 不是跳出循环，而是跳过当前循环执行下一次循环语句。

for 循环中，执行 continue 语句会触发 for 增量语句的执行。

在多重循环中，可以用标号 label 标出想 continue 的循环。

**语法**

continue 语法格式如下：

```
continue;
```

continue 语句流程图如下：

![img](https://www.runoob.com/wp-content/uploads/2015/06/6djaDcRMhzS6XCT0.png)

**实例**

在变量 a 等于 15 的时候跳过本次循环执行下一次循环：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 10

   /* for 循环 */
   for a < 20 {
      if a == 15 {
         /* 跳过此次循环 */
         a = a + 1;
         continue;
      }
      fmt.Printf("a 的值为 : %d\n", a);
      a++;    
   }  
}
```

以上实例执行结果为：

```
a 的值为 : 10
a 的值为 : 11
a 的值为 : 12
a 的值为 : 13
a 的值为 : 14
a 的值为 : 16
a 的值为 : 17
a 的值为 : 18
a 的值为 : 19
```

以下实例有多重循环，演示了使用标记和不使用标记的区别：

```go
package main

import "fmt"

func main() {

    // 不使用标记
    fmt.Println("---- continue ---- ")
    for i := 1; i <= 3; i++ {
        fmt.Printf("i: %d\n", i)
            for i2 := 11; i2 <= 13; i2++ {
                fmt.Printf("i2: %d\n", i2)
                continue
            }
    }

    // 使用标记
    fmt.Println("---- continue label ----")
    re:
        for i := 1; i <= 3; i++ {
            fmt.Printf("i: %d\n", i)
                for i2 := 11; i2 <= 13; i2++ {
                    fmt.Printf("i2: %d\n", i2)
                    continue re
                }
        }
}
```

以上实例执行结果为：

```
---- continue ---- 
i: 1
i2: 11
i2: 12
i2: 13
i: 2
i2: 11
i2: 12
i2: 13
i: 3
i2: 11
i2: 12
i2: 13
---- continue label ----
i: 1
i2: 11
i: 2
i2: 11
i: 3
i2: 11
```

### goto 语句

**Go 语言的 goto 语句可以无条件地转移到过程中指定的行**。

goto 语句通常与条件语句配合使用。可用来实现条件转移， 构成循环，跳出循环体等功能。

**但是，在结构化程序设计中一般不主张使用 goto 语句， 以免造成程序流程的混乱，使理解和调试程序都产生困难。**

**语法**

goto 语法格式如下：

```
goto label;
..
.
label: statement;
```

goto 语句流程图如下：

![img](https://www.runoob.com/wp-content/uploads/2015/06/xsTjcmiTVayxBjYe.png)

在变量 a 等于 15 的时候跳过本次循环并回到循环的开始语句 LOOP 处：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 10

   /* 循环 */
   LOOP: for a < 20 {
      if a == 15 {
         /* 跳过迭代 */
         a = a + 1
         goto LOOP
      }
      fmt.Printf("a的值为 : %d\n", a)
      a++    
   }  
}
```

以上实例执行结果为：

```
a的值为 : 10
a的值为 : 11
a的值为 : 12
a的值为 : 13
a的值为 : 14
a的值为 : 16
a的值为 : 17
a的值为 : 18
a的值为 : 19
```

## 10.2 无限循环

如果循环中条件语句永远不为 false 则会进行无限循环，我们可以通过 for 循环语句中只设置一个条件表达式来执行无限循环：

```go
package main

import "fmt"

func main() {
    for true  {
        fmt.Printf("这是无限循环。\n");
    }
}
```

# 11. Go 语言函数

函数是基本的代码块，用于执行一个任务。

Go 语言最少有个 main() 函数。

你可以通过函数来划分不同功能，逻辑上每个函数执行的是指定的任务。

函数声明告诉了编译器函数的名称，返回类型，和参数。

Go 语言标准库提供了多种可动用的内置的函数。例如，len() 函数可以接受不同类型参数并返回该类型的长度。如果我们传入的是字符串则返回字符串的长度，如果传入的是数组，则返回数组中包含的元素个数。

## 11.1 函数定义

Go 语言函数定义格式如下：

```go
func function_name( [parameter list] ) [return_types] {
   函数体
}
```

函数定义解析：

- func：函数由 func 开始声明
- function_name：函数名称，函数名和参数列表一起构成了函数签名。
- parameter list：参数列表，参数就像一个占位符，当函数被调用时，你可以将值传递给参数，这个值被称为实际参数。参数列表指定的是参数类型、顺序、及参数个数。参数是可选的，也就是说函数也可以不包含参数。
- return_types：返回类型，函数返回一列值。return_types 是该列值的数据类型。有些功能不需要返回值，这种情况下 return_types 不是必须的。
- 函数体：函数定义的代码集合。

**实例**

以下实例为 max() 函数的代码，该函数传入两个整型参数 num1 和 num2，并返回这两个参数的最大值：

```go
/* 函数返回两个数的最大值 */
func max(num1, num2 int) int {
   /* 声明局部变量 */
   var result int

   if (num1 > num2) {
      result = num1
   } else {
      result = num2
   }
   return result
}
```

## 11.2 函数调用

当创建函数时，你定义了函数需要做什么，通过调用该函数来执行指定任务。

调用函数，向函数传递参数，并返回值，例如：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 100
   var b int = 200
   var ret int

   /* 调用函数并返回最大值 */
   ret = max(a, b)

   fmt.Printf( "最大值是 : %d\n", ret )
}

/* 函数返回两个数的最大值 */
func max(num1, num2 int) int {
   /* 定义局部变量 */
   var result int

   if (num1 > num2) {
      result = num1
   } else {
      result = num2
   }
   return result
}
```

以上实例在 main() 函数中调用 max（）函数，执行结果为：

```
最大值是 : 200
```

## 11.3 函数返回多个值

```go
package main

import "fmt"

func swap(x, y string) (string, string) {
   return y, x
}

func main() {
   a, b := swap("Google", "Runoob")
   fmt.Println(a, b)
}
```

以上实例执行结果为：

```
Runoob Google
```

## 11.4 函数参数

函数如果使用参数，该变量可称为函数的形参。

形参就像定义在函数体内的局部变量。

调用函数，可以通过两种方式来传递参数：

| 传递类型                                                     | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [值传递](https://www.runoob.com/go/go-function-call-by-value.html) | 值传递是指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。 |
| [引用传递](https://www.runoob.com/go/go-function-call-by-reference.html) | 引用传递是指在调用函数时将实际参数的地址传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。 |

默认情况下，Go 语言使用的是值传递，即在调用过程中不会影响到实际参数。

### 函数值传递值

传递是指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。

默认情况下，Go 语言使用的是值传递，即在调用过程中不会影响到实际参数。

以下定义了 swap() 函数：

```go
/* 定义相互交换值的函数 */
func swap(x, y int) int {
   var temp int

   temp = x /* 保存 x 的值 */
   x = y    /* 将 y 值赋给 x */
   y = temp /* 将 temp 值赋给 y*/

   return temp;
}
```

接下来，让我们使用值传递来调用 swap() 函数：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 100
   var b int = 200

   fmt.Printf("交换前 a 的值为 : %d\n", a )
   fmt.Printf("交换前 b 的值为 : %d\n", b )

   /* 通过调用函数来交换值 */
   swap(a, b)

   fmt.Printf("交换后 a 的值 : %d\n", a )
   fmt.Printf("交换后 b 的值 : %d\n", b )
}

/* 定义相互交换值的函数 */
func swap(x, y int) int {
   var temp int

   temp = x /* 保存 x 的值 */
   x = y    /* 将 y 值赋给 x */
   y = temp /* 将 temp 值赋给 y*/

   return temp;
}
```

以上代码执行结果为：

```
交换前 a 的值为 : 100
交换前 b 的值为 : 200
交换后 a 的值 : 100
交换后 b 的值 : 200
```

> 下方评论留言：
>
> 交换值可以这么写：（前面介绍过的**初始化赋值**）
>
> ```
> a := 100
> b := 200
> a, b = b, a
> // a == 200
> // b == 100
> ```

### 函数引用传递值

引用传递是指在调用函数时将实际参数的**地址**传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。

引用传递指针参数传递到函数内，以下是交换函数 swap() 使用了引用传递：

```go
/* 定义交换值函数*/
func swap(x *int, y *int) {
   var temp int
   temp = *x    /* 保持 x 地址上的值 */
   *x = *y      /* 将 y 值赋给 x */
   *y = temp    /* 将 temp 值赋给 y */
}
```

以下我们通过使用引用传递来调用 swap() 函数：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 100
   var b int= 200

   fmt.Printf("交换前，a 的值 : %d\n", a )
   fmt.Printf("交换前，b 的值 : %d\n", b )

   /* 调用 swap() 函数
   * &a 指向 a 指针，a 变量的地址
   * &b 指向 b 指针，b 变量的地址
   */
   swap(&a, &b)

   fmt.Printf("交换后，a 的值 : %d\n", a )
   fmt.Printf("交换后，b 的值 : %d\n", b )
}

func swap(x *int, y *int) {
   var temp int
   temp = *x    /* 保存 x 地址上的值 */
   *x = *y      /* 将 y 值赋给 x */
   *y = temp    /* 将 temp 值赋给 y */
}
```

以上代码执行结果为：

```
交换前，a 的值 : 100
交换前，b 的值 : 200
交换后，a 的值 : 200
交换后，b 的值 : 100
```

## 11.5 函数用法

| 函数用法                                                     | 描述                                     |
| :----------------------------------------------------------- | :--------------------------------------- |
| [函数作为另外一个函数的实参](https://www.runoob.com/go/go-function-as-values.html) | 函数定义后可作为另外一个函数的实参数传入 |
| [闭包](https://www.runoob.com/go/go-function-closures.html)  | **闭包是匿名函数，可在动态编程中使用**   |
| [方法](https://www.runoob.com/go/go-method.html)             | 方法就是一个包含了接受者的函数           |

### 函数作为另一个函数的实参

Go 语言可以很灵活的创建函数，并作为另外一个函数的实参。以下实例中我们在定义的函数中初始化一个变量，该函数仅仅是为了使用内置函数 **math.sqrt()**，实例为：

```go
package main

import (
   "fmt"
   "math"
)

func main(){
   /* 声明函数变量 */
   getSquareRoot := func(x float64) float64 {
      return math.Sqrt(x)
   }

   /* 使用函数 */
   fmt.Println(getSquareRoot(9))

}
```

以上代码执行结果为：

```
3
```

> 下方评论区留言
>
> 函数作为参数传递，实现回调。
>
> ```go
> package main
> import "fmt"
> 
> // 声明一个函数类型
> type cb func(int) int
> 
> func main() {
>     testCallBack(1, callBack)
>     testCallBack(2, func(x int) int {
>         fmt.Printf("我是回调，x：%d\n", x)
>         return x
>     })
> }
> 
> func testCallBack(x int, f cb) {
>     f(x)
> }
> 
> func callBack(x int) int {
>     fmt.Printf("我是回调，x：%d\n", x)
>     return x
> }
> ```

### 函数闭包

> [golang 闭包是分配在堆上还是栈上？](https://www.cnblogs.com/peteremperor/p/14595769.html)
>
> **闭包环境中引用的变量是不能够在栈上分配的，而是在堆上分配**。因为如果引用的变量在栈上分配，那么该变量会跟随函数f返回之后回收，那么闭包函数就不可能访问未分配的一个变量，即未声明的变量，之所以能够再堆上分配，而不是在栈上分配，是Go的一个语言特性----**escape analyze（能够自动分析出变量的作用范围，是否将变量分配堆上）**。

Go 语言支持匿名函数，可作为闭包。匿名函数是一个"内联"语句或表达式。匿名函数的优越性在于可以直接使用函数内的变量，不必申明。

以下实例中，我们创建了函数 getSequence() ，返回另外一个函数。该函数的目的是在闭包中递增 i 变量，代码如下：

```go
package main

import "fmt"

func getSequence() func() int {
   i:=0
   return func() int {
      i+=1
     return i  
   }
}

func main(){
   /* nextNumber 为一个函数，函数 i 为 0 */
   nextNumber := getSequence()  

   /* 调用 nextNumber 函数，i 变量自增 1 并返回 */
   fmt.Println(nextNumber())
   fmt.Println(nextNumber())
   fmt.Println(nextNumber())
   
   /* 创建新的函数 nextNumber1，并查看结果 */
   nextNumber1 := getSequence()  
   fmt.Println(nextNumber1())
   fmt.Println(nextNumber1())
}
```

以上代码执行结果为：

```
1
2
3
1
2
```

> 下方评论区的留言
>
> + 带参数的闭包函数调用
>
>   ```go
>   package main
>   
>   import "fmt"
>   func main() {
>       add_func := add(1,2)
>       fmt.Println(add_func())
>       fmt.Println(add_func())
>       fmt.Println(add_func())
>   }
>   
>   // 闭包使用方法
>   func add(x1, x2 int) func()(int,int)  {
>       i := 0
>       return func() (int,int){
>           i++
>           return i,x1+x2
>       }
>   }
>   ```
>
> + 闭包带参数补充
>
>   ```go
>   package main
>   import "fmt"
>   func main() {
>       add_func := add(1,2)
>       fmt.Println(add_func(1,1))
>       fmt.Println(add_func(0,0))
>       fmt.Println(add_func(2,2))
>   } 
>   // 闭包使用方法
>   func add(x1, x2 int) func(x3 int,x4 int)(int,int,int)  {
>       i := 0
>       return func(x3 int,x4 int) (int,int,int){ 
>          i++
>          return i,x1+x2,x3+x4
>       }
>   }
>   ```
>
> + 闭包带参考继承补充
>
>   ```go
>   package main
>     
>   import "fmt"
>     
>   // 闭包使用方法，函数声明中的返回值(闭包函数)不用写具体的形参名称
>   func add(x1, x2 int) func(int, int) (int, int, int) {
>     i := 0
>     return func(x3, x4 int) (int, int, int) {
>       i += 1
>       return i, x1 + x2, x3 + x4
>     }
>   }
>     
>   func main() {
>     add_func := add(1, 2)
>     fmt.Println(add_func(4, 5))
>     fmt.Println(add_func(1, 3))
>     fmt.Println(add_func(2, 2)) 
>   }
>   ```

### 方法

Go 语言中同时有函数和方法。

**一个方法就是一个包含了接受者的函数，接受者可以是命名类型或者结构体类型的一个值或者是一个指针**。

所有给定类型的方法属于该类型的方法集。语法格式如下：

```go
func (variable_name variable_data_type) function_name() [return_type]{
   /* 函数体*/
}
```

下面定义一个结构体类型和该类型的一个方法：

```go
package main

import (
   "fmt"  
)

/* 定义结构体 */
type Circle struct {
  radius float64
}

func main() {
  var c1 Circle
  c1.radius = 10.00
  fmt.Println("圆的面积 = ", c1.getArea())
}

//该 method 属于 Circle 类型对象中的方法
func (c Circle) getArea() float64 {
  //c.radius 即为 Circle 类型对象中的属性
  return 3.14 * c.radius * c.radius
}
```

以上代码执行结果为：

```
圆的面积 =  314
```

> 下方评论区留言：
>
> + Go 没有面向对象，而我们知道常见的 Java。
>
>   C++ 等语言中，实现类的方法做法都是编译器隐式的给函数加一个 this 指针，而在 Go 里，这个 this 指针需要明确的申明出来，其实和其它 OO 语言并没有很大的区别。
>
>   在 C++ 中是这样的:
>
>   ```c++
>   class Circle {
>     public:
>       float getArea() {
>          return 3.14 * radius * radius;
>       }
>     private:
>       float radius;
>   }
>
>   // 其中 getArea 经过编译器处理大致变为
>   float getArea(Circle *const c) {
>     ...
>   }
>   ```
>
>   在 Go 中则是如下:
>
>   ```go
>   func (c Circle) getArea() float64 {
>     //c.radius 即为 Circle 类型对象中的属性
>     return 3.14 * c.radius * c.radius
>   }
>   ```
>
> + 关于值和指针，如果想在方法中改变结构体类型的属性，需要对方法传递指针，体会如下对结构体类型改变的方法 changRadius() 和普通的函数 change() 中的指针操作:
>
>   ```go
>   package main
>  
>   import (
>      "fmt"  
>   )
>  
>   /* 定义结构体 */
>   type Circle struct {
>     radius float64
>   }
>   ```
>
>
>   func main()  { 
>      var c Circle
>      fmt.Println(c.radius)
>      c.radius = 10.00
>      fmt.Println(c.getArea())
>      c.changeRadius(20)
>      fmt.Println(c.radius)
>      change(&c, 30)
>      fmt.Println(c.radius)
>   }
>   func (c Circle) getArea() float64  {
>      return c.radius * c.radius
>   }
>   // 注意如果想要更改成功c的值，这里需要传指针
>   func (c *Circle) changeRadius(radius float64)  {
>      c.radius = radius
>   }
>
>   // 以下操作将不生效
>   //func (c Circle) changeRadius(radius float64)  {
>   //   c.radius = radius
>   //}
>   // 引用类型要想改变值需要传指针
>   func change(c *Circle, radius float64)  {
>      c.radius = radius
>   }
>   ```
> 
>   输出为：
> 
>   ```
>   0
>   100
>   20
>   30
>
>   ```
> 
>   ```

# 12. Go 语言变量作用域

作用域为已声明标识符所表示的常量、类型、变量、函数或包在源代码中的作用范围。

Go 语言中变量可以在三个地方声明：

- **函数内定义的变量称为局部变量**
- **函数外定义的变量称为全局变量**
- **函数定义中的变量称为形式参数**

接下来让我们具体了解局部变量、全局变量和形式参数。

## 12.1 局部变量

在函数体内声明的变量称之为局部变量，它们的作用域只在函数体内，参数和返回值变量也是局部变量。

以下实例中 main() 函数使用了局部变量 a, b, c：

```go
package main

import "fmt"

func main() {
   /* 声明局部变量 */
   var a, b, c int

   /* 初始化参数 */
   a = 10
   b = 20
   c = a + b

   fmt.Printf ("结果： a = %d, b = %d and c = %d\n", a, b, c)
}
```

以上实例执行输出结果为：

```
结果： a = 10, b = 20 and c = 30
```

> 下方评论区留言：
>
> + 可通过花括号来控制变量的作用域，花括号中的变量是单独的作用域，同名变量会覆盖外层。
>
>   **1.**
>
>   ```go
>   a := 5
>   {
>       a := 3
>       fmt.Println("in a = ", a)
>   }
>   fmt.Println("out a = ", a)
>   ```
>
>   输出:
>
>   ```
>   in a = 3
>   out a = 5
>   ```
>
>   **2.**
>
>   ```go
>   a := 5
>   {
>       fmt.Println("in a = ", a)
>   }
>   fmt.Println("out a = ", a)
>   ```
>
>   输出:
>
>   ```
>   in a = 5
>   out a = 5
>   ```
>
>   **3.**
>
>   ```go
>   a := 5
>   {
>       a := 3    
>       fmt.Println("a = ", a)
>   }
>   ```
>
>   输出:
>
>   ```
>   a declared and not used
>   ```

## 12.2 全局变量

在函数体外声明的变量称之为全局变量，**全局变量可以在整个包甚至外部包（被导出后）使用**。

全局变量可以在任何函数中使用，以下实例演示了如何使用全局变量：

```go
package main

import "fmt"

/* 声明全局变量 */
var g int

func main() {

   /* 声明局部变量 */
   var a, b int

   /* 初始化参数 */
   a = 10
   b = 20
   g = a + b

   fmt.Printf("结果： a = %d, b = %d and g = %d\n", a, b, g)
}
```

以上实例执行输出结果为：

```
结果： a = 10, b = 20 and g = 30
```

Go 语言程序中全局变量与局部变量名称可以相同，但是函数内的局部变量会被优先考虑。实例如下：

```go
package main

import "fmt"

/* 声明全局变量 */
var g int = 20

func main() {
   /* 声明局部变量 */
   var g int = 10

   fmt.Printf ("结果： g = %d\n",  g)
}
```

以上实例执行输出结果为：

```
结果： g = 10
```

> 下方评论区留言：
>
> + 全局变量可以在整个包甚至外部包（被导出后）使用。
>
>   下述代码在 test.go 中定义了全局变量 Total_sum，然后在 hello.go 中引用。
>
>   **test.go:**
>
>   ```go
>   package main
>   import "fmt"
>   var Total_sum int = 0
>   func Sum_test(a int, b int) int {
>       fmt.Printf("%d + %d = %d\n", a, b, a+b)
>       Total_sum += (a + b)
>       fmt.Printf("Total_sum: %d\n", Total_sum)
>       return a+b
>   }
>   ```
>   
>    **hello.go：**
>   
>   ```go
>   package main
>     
>   import (
>       "fmt"
>   )
>     
>   func main() {
>       var sum int
>       sum = Sum_test(2, 3)
>       fmt.Printf("sum: %d; Total_sum: %d\n", sum, Total_sum)
>   }
>   ```


## 12.3 形式参数

形式参数会作为函数的局部变量来使用。实例如下：

```go
package main

import "fmt"

/* 声明全局变量 */
var a int = 20;

func main() {
   /* main 函数中声明局部变量 */
   var a int = 10
   var b int = 20
   var c int = 0

   fmt.Printf("main()函数中 a = %d\n",  a);
   c = sum( a, b);
   fmt.Printf("main()函数中 c = %d\n",  c);
}

/* 函数定义-两数相加 */
func sum(a, b int) int {
   fmt.Printf("sum() 函数中 a = %d\n",  a);
   fmt.Printf("sum() 函数中 b = %d\n",  b);

   return a + b;
}
```

以上实例执行输出结果为：

```
main()函数中 a = 10
sum() 函数中 a = 10
sum() 函数中 b = 20
main()函数中 c = 30
```

## 12.4 初始化局部和全局变量

不同类型的局部和全局变量默认值为：

| 数据类型    | 初始化默认值 |
| :---------- | :----------- |
| int         | 0            |
| float32     | 0            |
| **pointer** | **nil**      |

# 13. Go 语言数组

Go 语言提供了数组类型的数据结构。

数组是具有相同唯一类型的一组已编号且**长度固定**的数据项序列，这种类型可以是任意的原始类型例如整型、字符串或者自定义类型。

相对于去声明 **number0, number1, ..., number99** 的变量，使用数组形式 **numbers[0], numbers[1] ..., numbers[99]** 更加方便且易于扩展。

数组元素可以通过索引（位置）来读取（或者修改），索引从 0 开始，第一个元素索引为 0，第二个索引为 1，以此类推。

![img](https://www.runoob.com/wp-content/uploads/2015/06/goarray.png)

## 13.1 声明数组

Go 语言数组声明需要指定元素类型及元素个数，语法格式如下：

```go
var variable_name [SIZE] variable_type
```

以上为一维数组的定义方式。例如以下定义了数组 balance 长度为 10 类型为 float32：

```go
var balance [10] float32
```

## 13.2 初始化数组

以下演示了数组初始化：

```go
var balance = [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
```

我们也可以通过字面量在声明数组的同时快速初始化数组：

```go
balance := [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
```

**如果数组长度不确定，可以使用 ... 代替数组的长度**，编译器会根据元素个数自行推断数组的长度：

```go
var balance = [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
或
balance := [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
```

如果设置了数组的长度，我们还可以通过指定下标来初始化元素：

```go
//  将索引为 1 和 3 的元素初始化
balance := [5]float32{1:2.0,3:7.0}
```

初始化数组中 **{}** 中的元素个数不能大于 **[]** 中的数字。

**如果忽略 [] 中的数字不设置数组大小，Go 语言会根据元素的个数来设置数组的大小**：

```go
 balance[4] = 50.0
```

以上实例读取了第五个元素。数组元素可以通过索引（位置）来读取（或者修改），索引从 0 开始，第一个元素索引为 0，第二个索引为 1，以此类推。

![img](https://www.runoob.com/wp-content/uploads/2015/06/array_presentation.jpg)

## 13.3 访问数组元素

数组元素可以通过索引（位置）来读取。格式为数组名后加中括号，中括号中为索引的值。例如：

```
var salary float32 = balance[9]
```

以上实例读取了数组 balance 第 10 个元素的值。

以下演示了数组完整操作（声明、赋值、访问）的实例：

```go
package main

import "fmt"

func main() {
   var n [10]int /* n 是一个长度为 10 的数组 */
   var i,j int

   /* 为数组 n 初始化元素 */        
   for i = 0; i < 10; i++ {
      n[i] = i + 100 /* 设置元素为 i + 100 */
   }

   /* 输出每个数组元素的值 */
   for j = 0; j < 10; j++ {
      fmt.Printf("Element[%d] = %d\n", j, n[j] )
   }
}
```

以上实例执行结果如下：

```
Element[0] = 100
Element[1] = 101
Element[2] = 102
Element[3] = 103
Element[4] = 104
Element[5] = 105
Element[6] = 106
Element[7] = 107
Element[8] = 108
Element[9] = 109
```

```go
package main

import "fmt"

func main() {
   var i,j,k int
   // 声明数组的同时快速初始化数组
   balance := [5]float32{1000.0, 2.0, 3.4, 7.0, 50.0}

   /* 输出数组元素 */         ...
   for i = 0; i < 5; i++ {
      fmt.Printf("balance[%d] = %f\n", i, balance[i] )
   }
   
   balance2 := [...]float32{1000.0, 2.0, 3.4, 7.0, 50.0}
   /* 输出每个数组元素的值 */
   for j = 0; j < 5; j++ {
      fmt.Printf("balance2[%d] = %f\n", j, balance2[j] )
   }

   //  将索引为 1 和 3 的元素初始化
   balance3 := [5]float32{1:2.0,3:7.0}  
   for k = 0; k < 5; k++ {
      fmt.Printf("balance3[%d] = %f\n", k, balance3[k] )
   }
}
```

执行结果如下：

```shell
以上实例执行结果如下：

balance[0] = 1000.000000
balance[1] = 2.000000
balance[2] = 3.400000
balance[3] = 7.000000
balance[4] = 50.000000
balance2[0] = 1000.000000
balance2[1] = 2.000000
balance2[2] = 3.400000
balance2[3] = 7.000000
balance2[4] = 50.000000
balance3[0] = 0.000000
balance3[1] = 2.000000
balance3[2] = 0.000000
balance3[3] = 7.000000
balance3[4] = 0.000000
```

> 下方评论区留言：
>
> + **使用数组来打印杨辉三角**
>
>   根据上一行的内容，来获取下一行的内容并打印出来。
>
>   ```go
>   package main
>   import "fmt"
>   func GetYangHuiTriangleNextLine(inArr []int) []int {
>       var out []int
>       var i int
>       arrLen := len(inArr)
>       out = append(out, 1)
>       if 0 == arrLen {
>           return out
>       }
>       for i = 0; i < arrLen-1; i++ {
>           out = append(out, inArr[i]+inArr[i+1])
>       }
>       out = append(out, 1)
>       return out
>   }
>   func main() {
>       nums := []int{}
>       var i int
>       for i = 0; i < 10; i++ {
>           nums = GetYangHuiTriangleNextLine(nums)
>           fmt.Println(nums)
>       }
>   }
>   ```
>
>   输出结果为：
>
>   ```
>   [1]
>   [1 1]
>   [1 2 1]
>   [1 3 3 1]
>   [1 4 6 4 1]
>   [1 5 10 10 5 1]
>   [1 6 15 20 15 6 1]
>   [1 7 21 35 35 21 7 1]
>   [1 8 28 56 70 56 28 8 1]
>   [1 9 36 84 126 126 84 36 9 1]
>   ```
>
> + **数组初始化**
>
>   初始化数组的初始化有多种形式。
>
>   ```
>   [5] int {1,2,3,4,5}
>   ```
>
>   长度为5的数组，其元素值依次为：1，2，3，4，5。
>
>   ```
>   [5] int {1,2}
>   ```
>
>   长度为 5 的数组，其元素值依次为：1，2，0，0，0 。
>
>   在初始化时没有指定初值的元素将会赋值为其元素类型 int 的默认值0，string 的默认值是 ""。
>
>   ```
>   [...] int {1,2,3,4,5}
>   ```
>
>   长度为 5 的数组，其长度是根据初始化时指定的元素个数决定的。
>
>   ```
>   [5] int { 2:1,3:2,4:3}
>   ```
>
>   长度为 5 的数组，key:value，其元素值依次为：0，0，1，2，3。在初始化时指定了 2，3，4 索引中对应的值：1，2，3
>
>   ```
>   [...] int {2:1,4:3}
>   ```
>
>   长度为5的数组，起元素值依次为：0，0，1，0，3。由于指定了最大索引 4 对应的值 3，根据初始化的元素个 数确定其长度为5赋值与使用。
>
>   **切片的初始化**
>
>   切片可以通过数组来初始化，也可以通过内置函数 make() 初始化。
>
>   初始化时 len=cap，在追加元素时如果容量 cap 不足时将按 len 的 2 倍扩容。
>
>   **s :=[] int {1,2,3 }** 直接初始化切片，[] 表示是切片类型，{1,2,3} 初始化值依次是 1,2,3。其 cap=len=3。
>
>   **s := arr[:]** 初始化切片 s，是数组 arr 的引用。
>
>   **s := arr[startIndex:endIndex]** 将 arr 中从下标 startIndex 到 endIndex-1 下的元素创建为一个新的切片。
>
>   **s := arr[startIndex:]** 缺省 endIndex 时将表示一直到 arr 的最后一个元素。
>
>   **s := arr[:endIndex]** 缺省 startIndex 时将表示从 arr 的第一个元素开始。
>
>   **s1 := s[startIndex:endIndex]** 通过切片 s 初始化切片 s1
>
>   **s :=make([]int,len,cap)** 通过内置函数 make() 初始化切片 s,[]int 标识为其元素类型为 int 的切片。

## 13.3 更多内容

数组对 Go 语言来说是非常重要的，以下我们将介绍数组更多的内容：

| 内容                                                         | 描述                                            |
| :----------------------------------------------------------- | :---------------------------------------------- |
| [多维数组](https://www.runoob.com/go/go-multi-dimensional-arrays.html) | Go 语言支持多维数组，最简单的多维数组是二维数组 |
| [向函数传递数组](https://www.runoob.com/go/go-passing-arrays-to-functions.html) | 你可以向函数传递数组参数                        |

### 多维数组

Go 语言支持多维数组，以下为常用的多维数组声明方式：

```go
var variable_name [SIZE1][SIZE2]...[SIZEN] variable_type
```

以下实例声明了三维的整型数组：

```go
var threedim [5][10][4]int
```

#### 二维数组

二维数组是最简单的多维数组，二维数组本质上是由一维数组组成的。二维数组定义方式如下：

```
var arrayName [ x ][ y ] variable_type
```

variable_type 为 Go 语言的数据类型，arrayName 为数组名，二维数组可认为是一个表格，x 为行，y 为列，下图演示了一个二维数组 a 为三行四列：

![img](https://www.runoob.com/wp-content/uploads/2015/06/go-array-2021-01-19.png)

二维数组中的元素可通过 **a\[ i ]\[ j ]** 来访问。

```go
package main

import "fmt"

func main() {
    // Step 1: 创建数组
    values := [][]int{}

    // Step 2: 使用 appped() 函数向空的二维数组添加两行一维数组
    row1 := []int{1, 2, 3}
    row2 := []int{4, 5, 6}
    values = append(values, row1)
    values = append(values, row2)

    // Step 3: 显示两行数据
    fmt.Println("Row 1")
    fmt.Println(values[0])
    fmt.Println("Row 2")
    fmt.Println(values[1])

    // Step 4: 访问第一个元素
    fmt.Println("第一个元素为：")
    fmt.Println(values[0][0])
}
```

以上实例运行输出结果为：

```shell
Row 1
[1 2 3]
Row 2
[4 5 6]
第一个元素为：
1
```

#### 初始化二维数组

多维数组可通过大括号来初始值。以下实例为一个 3 行 4 列的二维数组：

```go
a := [3][4]int{  
 {0, 1, 2, 3} ,   /*  第一行索引为 0 */
 {4, 5, 6, 7} ,   /*  第二行索引为 1 */
 {8, 9, 10, 11},   /* 第三行索引为 2 */
}
```

> **注意：**以上代码中倒数第二行的 **}** 必须要有逗号，因为最后一行的 **}** 不能单独一行，也可以写成这样：
>
> ```go
> a := [3][4]int{  
>  {0, 1, 2, 3} ,   /*  第一行索引为 0 */
>  {4, 5, 6, 7} ,   /*  第二行索引为 1 */
>  {8, 9, 10, 11}}   /* 第三行索引为 2 */
> ```

以下实例初始化一个 2 行 2 列 的二维数组：

```go
package main

import "fmt"

func main() {
    // 创建二维数组
    sites := [2][2]string{}

    // 向二维数组添加元素
    sites[0][0] = "Google"
    sites[0][1] = "Runoob"
    sites[1][0] = "Taobao"
    sites[1][1] = "Weibo"

    // 显示结果
    fmt.Println(sites)
}
```

以上实例运行输出结果为：

```shell
[[Google Runoob] [Taobao Weibo]]
```

#### 访问二维数组

二维数组通过指定坐标来访问。如数组中的行索引与列索引，例如：

```go
val := a[2][3]
或
var value int = a[2][3]
```

以上实例访问了二维数组 val 第三行的第四个元素。

二维数组可以使用循环嵌套来输出元素：

```go
package main

import "fmt"

func main() {
   /* 数组 - 5 行 2 列*/
   var a = [5][2]int{ {0,0}, {1,2}, {2,4}, {3,6},{4,8}}
   var i, j int

   /* 输出数组元素 */
   for  i = 0; i < 5; i++ {
      for j = 0; j < 2; j++ {
         fmt.Printf("a[%d][%d] = %d\n", i,j, a[i][j] )
      }
   }
}
```

以上实例运行输出结果为：

```shell
a[0][0] = 0
a[0][1] = 0
a[1][0] = 1
a[1][1] = 2
a[2][0] = 2
a[2][1] = 4
a[3][0] = 3
a[3][1] = 6
a[4][0] = 4
a[4][1] = 8
```

以下实例创建各个维度元素数量不一致的多维数组：

```go
package main

import "fmt"

func main() {
    // 创建空的二维数组
    animals := [][]string{}

    // 创建三一维数组，各数组长度不同
    row1 := []string{"fish", "shark", "eel"}
    row2 := []string{"bird"}
    row3 := []string{"lizard", "salamander"}

    // 使用 append() 函数将一维数组添加到二维数组中
    animals = append(animals, row1)
    animals = append(animals, row2)
    animals = append(animals, row3)

    // 循环输出
    for i := range animals {
        fmt.Printf("Row: %v\n", i)
        fmt.Println(animals[i])
    }
}
```

以上实例运行输出结果为：

```shell
Row: 0
[fish shark eel]
Row: 1
[bird]
Row: 2
[lizard salamander]
```

### 向函数传递数组

如果你想向函数传递数组参数，你需要在函数定义时，声明形参为数组，我们可以通过以下两种方式来声明：

**方法一**

**形参设定数组大小**：

```go
void myFunction(param [10]int)
{
.
.
.
}
```

**方法二**

形参未设定数组大小：

```go
void myFunction(param []int)
{
.
.
.
}
```

让我们看下以下实例，实例中函数接收整型数组参数，另一个参数指定了数组元素的个数，并返回平均值：

```go
func getAverage(arr []int, size int) float32
{
   var i int
   var avg, sum float32  

   for i = 0; i < size; ++i {
      sum += arr[i]
   }

   avg = sum / size

   return avg;
}
```

接下来我们来调用这个函数：

```go
package main

import "fmt"

func main() {
   /* 数组长度为 5 */
   var  balance = [5]int {1000, 2, 3, 17, 50}
   var avg float32

   /* 数组作为参数传递给函数 */
   avg = getAverage( balance, 5 ) ;

   /* 输出返回的平均值 */
   fmt.Printf( "平均值为: %f ", avg );
}
func getAverage(arr [5]int, size int) float32 {
   var i,sum int
   var avg float32  

   for i = 0; i < size;i++ {
      sum += arr[i]
   }

   avg = float32(sum) / float32(size)

   return avg;
}
```

以上实例执行输出结果为：

```
平均值为: 214.399994
```

浮点数计算输出有一定的偏差，你也可以转整型来设置精度。

```go
package main
import (
    "fmt"
)
func main() {
    a := 1.69
    b := 1.7
    c := a * b      // 结果应该是2.873
    fmt.Println(c)  // 输出的是2.8729999999999998
}
```

设置固定精度：

```go
package main
import (
    "fmt"
)
func main() {
    a := 1690           // 表示1.69
    b := 1700           // 表示1.70
    c := a * b          // 结果应该是2873000表示 2.873
    fmt.Println(c)      // 内部编码
    fmt.Println(float64(c) / 1000000) // 显示
}
```

> 下方评论区留言：
>
> + 我觉得这个解说有点模糊，因为没有区分数组和切片:
>
>   声明数组：
>
>   ```go
>   nums := [3]int{1,2,3,}
>   ```
>
>   声明切片：
>
>   ```go
>   nums := []int{1,2,3}
>   ```
>
>   **没有所谓没有声明长度的数组存在**。
>
> + **与 c 语言不同，go 的数组作为函数参数传递的是副本，函数内修改数组并不改变原来的数组。**
>
>   ```go
>   package main
>   import "fmt"
>   // 这里 如果写成 nums[] int 会报错
>   func change(nums[3] int){
>      nums[0]=100
>   }
>   func main() {
>       var nums=[3]int{1,2,3}   
>       change(nums)   //nums并未被改变   
>       fmt.Println(nums[0])   
>       return
>   }
>   ```
>
> + 这里没有区分**数组**与**切片**
>
>   - Go 语言的数组是值，其**长度是其类型的一部分**，作为函数参数时，是**值传递**，函数中的修改对调用者不可见
>
>   - Go 语言中对数组的处理，一般采用**切片**的方式，**切片包含对底层数组内容的引用**，作为函数参数时，类似于**指针传递**，函数中的修改对调用者可见
>
>   ```go
>   // 数组
>   b := [...]int{2, 3, 5, 7, 11, 13}
>   
>   func boo(tt [6]int) {
>       tt[0], tt[len(tt)-1] = tt[len(tt)-1], tt[0]
>   }
>   
>   boo(b)
>   fmt.Println(b) // [2 3 5 7 11 13]
>   // 切片
>   p := []int{2, 3, 5, 7, 11, 13}
>   
>   func poo(tt []int) {
>       tt[0], tt[len(tt)-1] = tt[len(tt)-1], tt[0]
>   }
>   poo(p)
>   fmt.Println(p)  // [13 3 5 7 11 2]
>   ```
>
> + 函数里面的数组改动对定义类型为切片的数组外部数据也会有影响：
>
>   ```go
>   func main(){
>       var slice1 = []int{1,2,3,4,5,6,7,8}
>       var array = [8]int{1,2,3,4,5,6,7,8}
>       change_arr1(slice1)
>       change_arr2(array)
>       fmt.Println("slice1 = ", slice1) //[10,2,3,4,5,6,7,8]
>       fmt.Println("array = ", array) //[1,2,3,4,5,6,7,8]
>   }
>     
>   func change_arr1(arr []int) {
>       arr[0] = 10
>       fmt.Println("arr = ", arr) //[10,2,3,4,5,6,7,8]
>   }
>   func change_arr2(arr [8]int) {
>       arr[0] = 10
>       fmt.Println("arr = ", arr) //[10,2,3,4,5,6,7,8]
>   }
>   ```

# 14. Go 语言指针

Go 语言中指针是很容易学习的，Go 语言中使用指针可以更简单的执行一些任务。

接下来让我们来一步步学习 Go 语言指针。

我们都知道，变量是一种使用方便的占位符，用于引用计算机内存地址。

**Go 语言的取地址符是 &**，放到一个变量前使用就会返回相应变量的内存地址。

以下实例演示了变量在内存中地址：

```go
package main

import "fmt"

func main() {
   var a int = 10  

   fmt.Printf("变量的地址: %x\n", &a  )
}
```

执行以上代码输出结果为：

```
变量的地址: 20818a220
```

现在我们已经了解了什么是内存地址和如何去访问它。接下来我们将具体介绍指针。

## 14.1 什么是指针

一个指针变量指向了一个值的内存地址。

类似于变量和常量，在使用指针前你需要声明指针。指针声明格式如下：

```go
var var_name *var-type
```

var-type 为指针类型，var_name 为指针变量名，* 号用于指定变量是作为一个指针。以下是有效的指针声明：

```go
var ip *int        /* 指向整型*/
var fp *float32    /* 指向浮点型 */
```

本例中这是一个指向 int 和 float32 的指针。

## 14.2 如何使用指针

指针使用流程：

- 定义指针变量。
- 为指针变量赋值。
- 访问指针变量中指向地址的值。

**在指针类型前面加上 * 号（前缀）来获取指针所指向的内容**。

```go
package main

import "fmt"

func main() {
   var a int= 20   /* 声明实际变量 */
   var ip *int        /* 声明指针变量 */

   ip = &a  /* 指针变量的存储地址 */

   fmt.Printf("a 变量的地址是: %x\n", &a  )

   /* 指针变量的存储地址 */
   fmt.Printf("ip 变量储存的指针地址: %x\n", ip )

   /* 使用指针访问值 */
   fmt.Printf("*ip 变量的值: %d\n", *ip )
}
```

以上实例执行输出结果为：

```shell
a 变量的地址是: 20818a220
ip 变量储存的指针地址: 20818a220
*ip 变量的值: 20
```

## 14.3 Go 空指针

**当一个指针被定义后没有分配到任何变量时，它的值为 nil**。

nil 指针也称为空指针。

nil在概念上和其它语言的null、None、nil、NULL一样，都指代零值或空值。

一个指针变量通常缩写为 ptr。

查看以下实例：

```go
package main

import "fmt"

func main() {
   var  ptr *int

   fmt.Printf("ptr 的值为 : %x\n", ptr  )
}
```

以上实例输出结果为：

```shell
ptr 的值为 : 0
```

空指针判断：

```shell
if(ptr != nil)     /* ptr 不是空指针 */
if(ptr == nil)    /* ptr 是空指针 */
```

## 14.4 更多内容

接下来我们将为大家介绍Go语言中更多的指针应用：

| 内容                                                         | 描述                                         |
| :----------------------------------------------------------- | :------------------------------------------- |
| [Go 指针数组](https://www.runoob.com/go/go-array-of-pointers.html) | 你可以定义一个指针数组来存储地址             |
| [Go 指向指针的指针](https://www.runoob.com/go/go-pointer-to-pointer.html) | Go 支持指向指针的指针                        |
| [Go 向函数传递指针参数](https://www.runoob.com/go/go-passing-pointers-to-functions.html) | 通过引用或地址传参，在函数调用时可以改变其值 |

### Go 指针数组

在我们了解指针数组前，先看个实例，定义了长度为 3 的整型数组：

```go
package main

import "fmt"

const MAX int = 3

func main() {

   a := []int{10,100,200}
   var i int

   for i = 0; i < MAX; i++ {
      fmt.Printf("a[%d] = %d\n", i, a[i] )
   }
}
```

以上代码执行输出结果为：

```shell
a[0] = 10
a[1] = 100
a[2] = 200
```

有一种情况，我们可能需要保存数组，这样我们就需要使用到指针。

以下声明了整型指针数组：

```shell
var ptr [MAX]*int;
```

ptr 为整型指针数组。因此每个元素都指向了一个值。以下实例的三个整数将存储在指针数组中：

```go
package main

import "fmt"

const MAX int = 3

func main() {
   a := []int{10,100,200}
   var i int
   var ptr [MAX]*int;

   for  i = 0; i < MAX; i++ {
      ptr[i] = &a[i] /* 整数地址赋值给指针数组 */
   }

   for  i = 0; i < MAX; i++ {
      fmt.Printf("a[%d] = %d\n", i,*ptr[i] )
   }
}
```

以上代码执行输出结果为：

```shell
a[0] = 10
a[1] = 100
a[2] = 200
```

> 下方评论区留言：
>
> + 创建指针数组的时候，不适合用 **range** 循环。
>
>   > [Go range实现原理及性能优化剖析](https://blog.csdn.net/weixin_33910137/article/details/91699614)
>
>   错误代码:
>
>   ```
>   const max = 3
>   
>   func main() {
>       number := [max]int{5, 6, 7}
>       var ptrs [max]*int //指针数组
>       //将number数组的值的地址赋给ptrs
>       for i, x := range &number {
>           ptrs[i] = &x
>       }
>       for i, x := range ptrs {
>           fmt.Printf("指针数组：索引:%d 值:%d 值的内存地址:%d\n", i, *x, x)
>       }
>   }
>   ```
>
>   输出结果为：
>
>   ```
>   指针数组：索引:0 值:7 值的内存地址:824634204304
>   指针数组：索引:1 值:7 值的内存地址:824634204304
>   指针数组：索引:2 值:7 值的内存地址:824634204304
>   ```
>
>   从结果中我们发现内存地址都一样，而且值也一样。怎么回事？
>
>   这个问题是range循环的实现逻辑引起的。跟for循环不一样的地方在于range循环中的x变量是临时变量。range循环只是将值拷贝到x变量中。因此内存地址都是一样的。
>
>   正确代码如下：
>
>   ```
>   const max = 3
>   
>   func main() {
>       number := [max]int{5, 6, 7}
>       var ptrs [max]*int //指针数组
>       //将number数组的值的地址赋给ptrs
>       for i := 0; i < max; i++ {
>           ptrs[i] = &number[i]
>       }
>       for i, x := range ptrs {
>           fmt.Printf("指针数组：索引:%d 值:%d 值的内存地址:%d\n", i,*x, x)
>       }
>   }
>   ```
>
>   输出结果为：
>
>   ```
>   指针数组：索引:0 值:5 值的内存地址:824634212672
>   指针数组：索引:1 值:6 值的内存地址:824634212680
>   指针数组：索引:2 值:7 值的内存地址:824634212688
>   ```
>
>   从结果中可以看出内存地址都不一样。
>
> + 关于楼上的创建指针数组的时候，说的不适合用 range 循环。
>
>   其实不是不能用 range 循环，只是楼主不会用而已。
>
>   这是楼主的原始代码：
>
>   ```go
>   const max = 3
>   
>   func main() {
>       number := [max]int{5, 6, 7}
>       var ptrs [max]*int //指针数组
>       //将number数组的值的地址赋给ptrs
>       for i, x := range &number {
>           ptrs[i] = &x
>       }
>       for i, x := range ptrs {
>           fmt.Printf("指针数组：索引:%d 值:%d 值的内存地址:%d\n", i, *x, x)
>       }
>   }
>   ```
>
>   这样是使用 range 的正确用法，两个地方改动：**range &number** 改成 **range number**，**ptrs[i] = &x** 改成 **ptrs[i]=&number[i]**：
>
>   ```go
>   const max = 3
>   
>   func main() {
>       number := [max]int{5, 6, 7}
>       var ptrs [max]*int //指针数组
>       //将number数组的值的地址赋给ptrs
>       for i := range number {
>           ptrs[i] = &number[i]
>       }
>       for i, x := range ptrs {
>           fmt.Printf("指针数组：索引:%d 值:%d 值的内存地址:%d\n", i, *x, x)
>       }
>   }
>   ```
>
> + **flying81621** 的方法与**青云刀歌**的方法并无区别，flying81621 的 for range 方法也只是用于遍历数组，并没用使用 range 方法的临时变量。真正赋值的地方只有一次，for 的 body 内 ，青云刀歌真正原因在于，**x临时变量仅被声明一次，此后都是将迭代 number 出的值赋值给 x ， x 变量的内存地址始终未变，这样再将 x 的地址发送给 ptrs 数组，自然也是相同的。**
>
>   原代码：
>
>   ```go
>   for i, x := range &number {
>       ptrs[i] = &x
>   }
>   ```
>
>   输出结果：
>
>   ```
>   指针数组：索引:0 值:7 值的内存地址:824633794712
>   指针数组：索引:1 值:7 值的内存地址:824633794712
>   指针数组：索引:2 值:7 值的内存地址:824633794712
>   ```
>
>   "正确"使用应增加一个临时变量（其实这种做法不对，和原意不一致）
>
>   ```go
>   for i, x := range &number {
>       temp := x
>       ptrs[i] = &temp
>   }
>   ```
>
>   输出结果
>
>   ```
>   指针数组：索引:0 值:5 值的内存地址:824633794712
>   指针数组：索引:1 值:6 值的内存地址:824633794736
>   指针数组：索引:2 值:7 值的内存地址:824633794744
>   ```
>
> + 楼上的代码只是把临时变量 temp 的地址赋给了指针数组。
>
>   ```go
>   var arr [3]int
>   var parr [3]*int
>   for k, v := range arr {
>       temp := v
>       parr[k] = &temp;
>   }
>     
>   // 输出地址比对
>   for i := 0; i < 3; i+=1 {
>       fmt.Println(&arr[i], parr[i]);
>   }
>   ```
>
>   可以发现两组地址是不同的：
>
>   ```
>   0xc00011c140 0xc00011e068
>   0xc00011c148 0xc00011e080
>   0xc00011c150 0xc00011e088
>   ```
>
>   ***\*清云刀歌\**** 的是没有问题的，还可以直接用一个数组指针。
>
>   ```go
>   var arr [3]int
>   var parr [3]*int // 指针数组
>   var p *[3]int = &arr // 数组指针
>   for k, _ := range arr {
>       parr[k] = &arr[k];
>   }
>     
>   // 输出地址比对
>   for i := 0; i < 3; i+=1 {
>       fmt.Println(&arr[i], parr[i], &(*p)[i]);
>   }
>   ```
>
>   输出结果：
>
>   ```shell
>   0xc00000a420 0xc00000a420 0xc00000a420
>   0xc00000a428 0xc00000a428 0xc00000a428
>   0xc00000a430 0xc00000a430 0xc00000a430
>   ```

### Go 指向指针的指针

如果一个指针变量存放的又是另一个指针变量的地址，则称这个指针变量为指向指针的指针变量。

当定义一个指向指针的指针变量时，第一个指针存放第二个指针的地址，第二个指针存放变量的地址：

![img](https://www.runoob.com/wp-content/uploads/2015/06/pointer_to_pointer.jpg)

指向指针的指针变量声明格式如下：

```go
var ptr **int;
```

以上指向指针的指针变量为整型。

访问指向指针的指针变量值需要使用两个 * 号，如下所示：

```go
package main

import "fmt"

func main() {

   var a int
   var ptr *int
   var pptr **int

   a = 3000

   /* 指针 ptr 地址 */
   ptr = &a

   /* 指向指针 ptr 地址 */
   pptr = &ptr

   /* 获取 pptr 的值 */
   fmt.Printf("变量 a = %d\n", a )
   fmt.Printf("指针变量 *ptr = %d\n", *ptr )
   fmt.Printf("指向指针的指针变量 **pptr = %d\n", **pptr)
}
```

以上实例执行输出结果为：

```shell
变量 a = 3000
指针变量 *ptr = 3000
指向指针的指针变量 **pptr = 3000
```

### Go 指针作为函数参数

Go 语言允许向函数传递指针，只需要在函数定义的参数上设置为指针类型即可。

以下实例演示了如何向函数传递指针，并在函数调用后修改函数内的值：

```go
package main

import "fmt"

func main() {
   /* 定义局部变量 */
   var a int = 100
   var b int= 200

   fmt.Printf("交换前 a 的值 : %d\n", a )
   fmt.Printf("交换前 b 的值 : %d\n", b )

   /* 调用函数用于交换值
   * &a 指向 a 变量的地址
   * &b 指向 b 变量的地址
   */
   swap(&a, &b);

   fmt.Printf("交换后 a 的值 : %d\n", a )
   fmt.Printf("交换后 b 的值 : %d\n", b )
}

func swap(x *int, y *int) {
   var temp int
   temp = *x    /* 保存 x 地址的值 */
   *x = *y      /* 将 y 赋值给 x */
   *y = temp    /* 将 temp 赋值给 y */
}
```

以上实例允许输出结果为：

```shell
交换前 a 的值 : 100
交换前 b 的值 : 200
交换后 a 的值 : 200
交换后 b 的值 : 100
```

> 下方评论区留言：
>
> 以下是一个更简洁的变量交换实例：
>
> ```go
> package main
> 
> import "fmt"
> 
> func main() {
>     /* 定义局部变量 */
>    var a int = 100
>    var b int= 200
>    swap(&a, &b);
> 
>    fmt.Printf("交换后 a 的值 : %d\n", a )
>    fmt.Printf("交换后 b 的值 : %d\n", b )
> }
> 
> /* 交换函数这样写更加简洁，也是 go 语言的特性，可以用下，c++ 和 c# 是不能这么干的 */
>  
> func swap(x *int, y *int){
>     *x, *y = *y, *x
> }
> ```

# 15. Go 语言结构体

Go 语言中数组可以存储同一类型的数据，但在结构体中我们可以为不同项定义不同的数据类型。

结构体是由一系列具有相同类型或不同类型的数据构成的数据集合。

结构体表示一项记录，比如保存图书馆的书籍记录，每本书有以下属性：

- Title ：标题
- Author ： 作者
- Subject：学科
- ID：书籍ID

## 15.1 定义结构体

结构体定义需要使用 type 和 struct 语句。struct 语句定义一个新的数据类型，结构体中有一个或多个成员。type 语句设定了结构体的名称。结构体的格式如下：

```go
type struct_variable_type struct {
   member definition
   member definition
   ...
   member definition
}
```

一旦定义了结构体类型，它就能用于变量的声明，语法格式如下：

```go
variable_name := structure_variable_type {value1, value2...valuen}
或
variable_name := structure_variable_type { key1: value1, key2: value2..., keyn: valuen}
```

实例如下：

```go
package main

import "fmt"

type Books struct {
   title string
   author string
   subject string
   book_id int
}


func main() {

    // 创建一个新的结构体
    fmt.Println(Books{"Go 语言", "www.runoob.com", "Go 语言教程", 6495407})

    // 也可以使用 key => value 格式
    fmt.Println(Books{title: "Go 语言", author: "www.runoob.com", subject: "Go 语言教程", book_id: 6495407})

    // 忽略的字段为 0 或 空
   fmt.Println(Books{title: "Go 语言", author: "www.runoob.com"})
}
```

输出结果为：

```shell
{Go 语言 www.runoob.com Go 语言教程 6495407}
{Go 语言 www.runoob.com Go 语言教程 6495407}
{Go 语言 www.runoob.com  0}
```

## 15.2 访问结构体成员

如果要访问结构体成员，需要使用点号 **.** 操作符，格式为：

```
结构体.成员名
```

结构体类型变量使用 struct 关键字定义，实例如下：

```go
package main

import "fmt"

type Books struct {
   title string
   author string
   subject string
   book_id int
}

func main() {
   var Book1 Books        /* 声明 Book1 为 Books 类型 */
   var Book2 Books        /* 声明 Book2 为 Books 类型 */

   /* book 1 描述 */
   Book1.title = "Go 语言"
   Book1.author = "www.runoob.com"
   Book1.subject = "Go 语言教程"
   Book1.book_id = 6495407

   /* book 2 描述 */
   Book2.title = "Python 教程"
   Book2.author = "www.runoob.com"
   Book2.subject = "Python 语言教程"
   Book2.book_id = 6495700

   /* 打印 Book1 信息 */
   fmt.Printf( "Book 1 title : %s\n", Book1.title)
   fmt.Printf( "Book 1 author : %s\n", Book1.author)
   fmt.Printf( "Book 1 subject : %s\n", Book1.subject)
   fmt.Printf( "Book 1 book_id : %d\n", Book1.book_id)

   /* 打印 Book2 信息 */
   fmt.Printf( "Book 2 title : %s\n", Book2.title)
   fmt.Printf( "Book 2 author : %s\n", Book2.author)
   fmt.Printf( "Book 2 subject : %s\n", Book2.subject)
   fmt.Printf( "Book 2 book_id : %d\n", Book2.book_id)
}
```

以上实例执行运行结果为：

```
Book 1 title : Go 语言
Book 1 author : www.runoob.com
Book 1 subject : Go 语言教程
Book 1 book_id : 6495407
Book 2 title : Python 教程
Book 2 author : www.runoob.com
Book 2 subject : Python 语言教程
Book 2 book_id : 6495700
```

## 15.3 结构体作为函数参数

你可以像其他数据类型一样将结构体类型作为参数传递给函数。并以以上实例的方式访问结构体变量：

```go
package main

import "fmt"

type Books struct {
   title string
   author string
   subject string
   book_id int
}

func main() {
   var Book1 Books        /* 声明 Book1 为 Books 类型 */
   var Book2 Books        /* 声明 Book2 为 Books 类型 */

   /* book 1 描述 */
   Book1.title = "Go 语言"
   Book1.author = "www.runoob.com"
   Book1.subject = "Go 语言教程"
   Book1.book_id = 6495407

   /* book 2 描述 */
   Book2.title = "Python 教程"
   Book2.author = "www.runoob.com"
   Book2.subject = "Python 语言教程"
   Book2.book_id = 6495700

   /* 打印 Book1 信息 */
   printBook(Book1)

   /* 打印 Book2 信息 */
   printBook(Book2)
}

func printBook( book Books ) {
   fmt.Printf( "Book title : %s\n", book.title)
   fmt.Printf( "Book author : %s\n", book.author)
   fmt.Printf( "Book subject : %s\n", book.subject)
   fmt.Printf( "Book book_id : %d\n", book.book_id)
}
```

以上实例执行运行结果为：

```
Book title : Go 语言
Book author : www.runoob.com
Book subject : Go 语言教程
Book book_id : 6495407
Book title : Python 教程
Book author : www.runoob.com
Book subject : Python 语言教程
Book book_id : 6495700
```

## 15.4 结构体指针

你可以定义指向结构体的指针类似于其他指针变量，格式如下：

```
var struct_pointer *Books
```

以上定义的指针变量可以存储结构体变量的地址。查看结构体变量地址，可以将 & 符号放置于结构体变量前：

```
struct_pointer = &Book1
```

使用结构体指针访问结构体成员，使用 "." 操作符：

```
struct_pointer.title
```

接下来让我们使用结构体指针重写以上实例，代码如下：

```go
package main

import "fmt"

type Books struct {
   title string
   author string
   subject string
   book_id int
}

func main() {
   var Book1 Books        /* 声明 Book1 为 Books 类型 */
   var Book2 Books        /* 声明 Book2 为 Books 类型 */

   /* book 1 描述 */
   Book1.title = "Go 语言"
   Book1.author = "www.runoob.com"
   Book1.subject = "Go 语言教程"
   Book1.book_id = 6495407

   /* book 2 描述 */
   Book2.title = "Python 教程"
   Book2.author = "www.runoob.com"
   Book2.subject = "Python 语言教程"
   Book2.book_id = 6495700

   /* 打印 Book1 信息 */
   printBook(&Book1)

   /* 打印 Book2 信息 */
   printBook(&Book2)
}
func printBook( book *Books ) {
   fmt.Printf( "Book title : %s\n", book.title)
   fmt.Printf( "Book author : %s\n", book.author)
   fmt.Printf( "Book subject : %s\n", book.subject)
   fmt.Printf( "Book book_id : %d\n", book.book_id)
}
```

以上实例执行运行结果为：

```
Book title : Go 语言
Book author : www.runoob.com
Book subject : Go 语言教程
Book book_id : 6495407
Book title : Python 教程
Book author : www.runoob.com
Book subject : Python 语言教程
Book book_id : 6495700
```

> 下方评论留言：
>
> + **结构体是作为参数的值传递**：
>
>   ```go
>   package main
>
>   import "fmt"
>
>   type Books struct {
>       title string
>       author string
>       subject string
>       book_id int
>   }
>
>   func changeBook(book Books) {
>       book.title = "book1_change"
>   }
>
>   func main() {
>       var book1 Books
>       book1.title = "book1"
>       book1.author = "zuozhe"
>       book1.book_id = 1
>       changeBook(book1)
>       fmt.Println(book1)
>   }
>   ```
>
>   结果为：
>
>   ```shell
>   {book1 zuozhe  1}
>   ```
>
>   **如果想在函数里面改变结果体数据内容，需要传入指针**：
>
>   ```go
>   package main
>
>   import "fmt"
>
>   type Books struct {
>       title string
>       author string
>       subject string
>       book_id int
>   }
>
>   func changeBook(book *Books) {
>       book.title = "book1_change"
>   }
>
>   func main() {
>       var book1 Books
>       book1.title = "book1"
>       book1.author = "zuozhe"
>       book1.book_id = 1
>       changeBook(&book1)
>       fmt.Println(book1)
>   }
>   ```
>
>   结果为：
>
>   ```shell
>   {book1_change zuozhe  1}
>   ```
>
> + **结构体中属性的首字母大小写问题**
>
>   -  首字母大写相当于 public。
>   -  首字母小写相当于 protected。
>
>   **注意:** 这个 public 和 private 是相对于包（go 文件首行的 package 后面跟的包名）来说的。
>
>   **敲黑板，划重点**
>
>   当要将结构体对象转换为 JSON 时，对象中的属性首字母必须是大写，才能正常转换为 JSON。
>
>   示例一：
>
>   ```go
>   type Person struct {
>   　　　Name string　　　　　　//Name字段首字母大写
>   　　　age int               //age字段首字母小写
>   }
>     
>   func main() {
>   　　person:=Person{"小明",18}
>   　　if result,err:=json.Marshal(&person);err==nil{  //json.Marshal 将对象转换为json字符串
>   　　　　fmt.Println(string(result))
>   　　}
>   }
>   ```
>
>   控制台输出：
>
>   ```
>   {"Name":"小明"}    //只有Name，没有age
>   ```
>
>   示例二：
>
>   ```go
>   type Person  struct{
>      　　Name  string      //都是大写
>      　　Age    int               
>   }
>   ```
>
>   控制台输出：
>
>   ```
>   {"Name":"小明","Age":18}   //两个字段都有
>   ```
>
>   那这样 JSON 字符串以后就只能是大写了么？ 当然不是，可以使用 tag 标记要返回的字段名。
>
>   示例三：
>
>   ```go
>   type Person  struct{
>      　　Name  string   `json:"name"`　  //标记json名字为name　　　
>      　　Age    int     `json:"age"`
>      　　Time int64    `json:"-"`        // 标记忽略该字段
>     
>   }
>     
>   func main(){
>   　　person:=Person{"小明",18, time.Now().Unix()}
>   　　if result,err:=json.Marshal(&person);err==nil{
>   　　　fmt.Println(string(result))
>   　　}
>   }
>   ```
>
>   控制台输出：
>
>   ```
>   {"name":"小明","age":18}
>   ```

# 16. Go 语言切片(Slice)

**Go 语言切片是对数组的抽象**。

Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，Go 中提供了一种灵活，功能强悍的内置类型切片("动态数组")，与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。

## 16.1 定义切片

你可以声明一个未指定大小的数组来定义切片：

```go
var identifier []type
```

**切片不需要说明长度**。

或使用 **make()** 函数来创建切片:

```go
var slice1 []type = make([]type, len)

也可以简写为

slice1 := make([]type, len)
```

也可以指定容量，其中 **capacity** 为可选参数。

```go
make([]T, length, capacity)
```

这里 len 是数组的长度并且也是切片的初始长度。

### 切片初始化

```go
s :=[] int {1,2,3 } 
```

直接初始化切片，**[]** 表示是切片类型，**{1,2,3}** 初始化值依次是 **1,2,3**，其 **cap=len=3**。

```go
s := arr[:] 
```

初始化切片 **s**，是数组 arr 的引用。

```go
s := arr[startIndex:endIndex] 
```

将 arr 中从下标 startIndex 到 endIndex-1 下的元素创建为一个新的切片。

```go
s := arr[startIndex:] 
```

默认 endIndex 时将表示一直到arr的最后一个元素。

```go
s := arr[:endIndex] 
```

默认 startIndex 时将表示从 arr 的第一个元素开始。

```go
s1 := s[startIndex:endIndex] 
```

通过切片 s 初始化切片 s1。

```go
s :=make([]int,len,cap) 
```

通过内置函数 **make()** 初始化切片**s**，**[]int** 标识为其元素类型为 int 的切片。

## 16.2 len() 和 cap() 函数

切片是可索引的，并且可以由 len() 方法获取长度。

切片提供了计算容量的方法 cap() 可以测量切片最长可以达到多少。

以下为具体实例：

```go
package main

import "fmt"

func main() {
   var numbers = make([]int,3,5)

   printSlice(numbers)
}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
```

以上实例运行输出结果为:

```shell
len=3 cap=5 slice=[0 0 0]
```

## 16.3 空(nil)切片

**一个切片在未初始化之前默认为 nil，长度为 0**，实例如下：

```go
package main

import "fmt"

func main() {
   var numbers []int

   printSlice(numbers)

   if(numbers == nil){
      fmt.Printf("切片是空的")
   }
}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
```

以上实例运行输出结果为:

```shell
len=0 cap=0 slice=[]
切片是空的
```

## 16.4 切片截取

可以通过设置下限及上限来设置截取切片 *`[lower-bound:upper-bound]`*，实例如下：

```go
package main

import "fmt"

func main() {
   /* 创建切片 */
   numbers := []int{0,1,2,3,4,5,6,7,8}  
   printSlice(numbers)

   /* 打印原始切片 */
   fmt.Println("numbers ==", numbers)

   /* 打印子切片从索引1(包含) 到索引4(不包含)*/
   fmt.Println("numbers[1:4] ==", numbers[1:4])

   /* 默认下限为 0*/
   fmt.Println("numbers[:3] ==", numbers[:3])

   /* 默认上限为 len(s)*/
   fmt.Println("numbers[4:] ==", numbers[4:])

   numbers1 := make([]int,0,5)
   printSlice(numbers1)

   /* 打印子切片从索引  0(包含) 到索引 2(不包含) */
   number2 := numbers[:2]
   printSlice(number2)

   /* 打印子切片从索引 2(包含) 到索引 5(不包含) */
   number3 := numbers[2:5]
   printSlice(number3)

}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
```

执行以上代码输出结果为：

```shell
len=9 cap=9 slice=[0 1 2 3 4 5 6 7 8]
numbers == [0 1 2 3 4 5 6 7 8]
numbers[1:4] == [1 2 3]
numbers[:3] == [0 1 2]
numbers[4:] == [4 5 6 7 8]
len=0 cap=5 slice=[]
len=2 cap=9 slice=[0 1]
len=3 cap=7 slice=[2 3 4]
```

## 16.4 append() 和 copy() 函数

如果想增加切片的容量，我们必须创建一个新的更大的切片并把原分片的内容都拷贝过来。

下面的代码描述了从拷贝切片的 copy 方法和向切片追加新元素的 append 方法。

```go
package main

import "fmt"

func main() {
   var numbers []int
   printSlice(numbers)

   /* 允许追加空切片 */
   numbers = append(numbers, 0)
   printSlice(numbers)

   /* 向切片添加一个元素 */
   numbers = append(numbers, 1)
   printSlice(numbers)

   /* 同时添加多个元素 */
   numbers = append(numbers, 2,3,4)
   printSlice(numbers)

   /* 创建切片 numbers1 是之前切片的两倍容量*/
   numbers1 := make([]int, len(numbers), (cap(numbers))*2)

   /* 拷贝 numbers 的内容到 numbers1 */
   copy(numbers1,numbers)
   printSlice(numbers1)  
}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}
```

以上代码执行输出结果为：

```shell
len=0 cap=0 slice=[]
len=1 cap=1 slice=[0]
len=2 cap=2 slice=[0 1]
len=5 cap=6 slice=[0 1 2 3 4]
len=5 cap=12 slice=[0 1 2 3 4]
```

> 下方评论留言：
>
> + ```
>   numbers := []int{0,1,2,3,4,5,6,7,8}
>   number3 := numbers[2:5]
>   printSlice(number3)
>   ```
>
>   结果为:
>
>   ```
>   len=3 cap=7 slice=[2 3 4]
>   ```
>
>   capacity 为 7 是因为 number3 的 ptr 指向第三个元素， 后面还剩 2,3,4,5,6,7,8, 所以 cap=7。
>
>   如：
>
>   ```
>   number4 := numbers[5:]
>   printSlice(number4)
>   ```
>
>   结果为：
>
>   ```
>   len=4 cap=4 slice=[5 6 7 8]
>   ```
>
>   capacity 为 4 是因为 number4 的 ptr 指向第六个元素， 后面还剩 5,6,7,8 所以 cap=4。
>
> + 当append(list, [params])，先判断 list 的 cap 长度是否大于等于 **len(list) + len([params])**，如果大于那么 cap 不变，否则 cap = **2 \*** **max{cap(list), cap[params]}**，所以当 append(numbers, 2, 3, 4) cap 从 2 变成 6。
>
>   前面已有相关详解，很可惜关键部分把“2 * ”给漏了，在此补充。
>
> + **可以通过查看$GOROOT/src/runtime/slice.go源码，其中扩容相关代码如下：**
>
>   ```
>   newcap := old.cap
>   doublecap := newcap + newcap
>   if cap > doublecap {
>       newcap = cap
>   } else {
>       if old.len < 1024 {
>           newcap = doublecap
>       } else {
>           // Check 0 < newcap to detect overflow
>           // and prevent an infinite loop.
>           for 0 < newcap && newcap < cap {
>               newcap += newcap / 4
>           }
>           // Set newcap to the requested cap when
>           // the newcap calculation overflowed.
>           if newcap <= 0 {
>               newcap = cap
>           }
>       }
>   }
>   ```
>
>   **从上面的代码可以看出以下内容：**
>
>   - **首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）。**
>   - **否则判断，如果旧切片的长度小于1024，则最终容量(newcap)就是旧容量(old.cap)的两倍，即（newcap=doublecap），**
>   - **否则判断，如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的1/4，即（newcap=old.cap,for {newcap += newcap/4}）直到最终容量（newcap）大于等于新申请的容量(cap)，即（newcap >= cap）**
>   - **如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量（cap）。**
>
>   **需要注意的是，切片扩容还会根据切片中元素的类型不同而做不同的处理，比如`int`和`string`类型的处理方式就不一样。**

# 17. Go 语言范围(Range)

Go 语言中 range 关键字用于 for 循环中迭代数组(array)、切片(slice)、**通道(channel)**或集合(map)的元素。在数组和切片中它返回元素的索引和索引对应的值，在集合中返回 key-value 对。

```go
package main
import "fmt"
func main() {
    //这是我们使用range去求一个slice的和。使用数组跟这个很类似
    nums := []int{2, 3, 4}
    sum := 0
    for _, num := range nums {
        sum += num
    }
    fmt.Println("sum:", sum)
    //在数组上使用range将传入index和值两个变量。上面那个例子我们不需要使用该元素的序号，所以我们使用空白符"_"省略了。有时侯我们确实需要知道它的索引。
    for i, num := range nums {
        if num == 3 {
            fmt.Println("index:", i)
        }
    }
    //range也可以用在map的键值对上。
    kvs := map[string]string{"a": "apple", "b": "banana"}
    for k, v := range kvs {
        fmt.Printf("%s -> %s\n", k, v)
    }
    //range也可以用来枚举Unicode字符串。第一个参数是字符的索引，第二个是字符（Unicode的值）本身。
    for i, c := range "go" {
        fmt.Println(i, c)
    }
}
```

以上实例运行输出结果为：

```shell
sum: 9
index: 1
a -> apple
b -> banana
0 103
1 111
```

# 18. Go 语言Map(集合)

Map 是一种无序的键值对的集合。Map 最重要的一点是通过 key 来快速检索数据，key 类似于索引，指向数据的值。

Map 是一种集合，所以我们可以像迭代数组和切片那样迭代它。不过，Map 是无序的，我们无法决定它的返回顺序，这是因为 **Map 是使用 hash 表来实现的**。

## 18.1 定义 Map

可以使用内建函数 make 也可以使用 map 关键字来定义 Map:

```go s
/* 声明变量，默认 map 是 nil */
var map_variable map[key_data_type]value_data_type

/* 使用 make 函数 */
map_variable := make(map[key_data_type]value_data_type)
```

如果不初始化 map，那么就会创建一个 nil map。nil map 不能用来存放键值对

下面实例演示了创建和使用map:

```go
package main

import "fmt"

func main() {
    var countryCapitalMap map[string]string /*创建集合 */
    countryCapitalMap = make(map[string]string)

    /* map插入key - value对,各个国家对应的首都 */
    countryCapitalMap [ "France" ] = "巴黎"
    countryCapitalMap [ "Italy" ] = "罗马"
    countryCapitalMap [ "Japan" ] = "东京"
    countryCapitalMap [ "India " ] = "新德里"

    /*使用键输出地图值 */
    for country := range countryCapitalMap {
        fmt.Println(country, "首都是", countryCapitalMap [country])
    }

    /*查看元素在集合中是否存在 */
    capital, ok := countryCapitalMap [ "American" ] /*如果确定是真实的,则存在,否则不存在 */
    /*fmt.Println(capital) */
    /*fmt.Println(ok) */
    if (ok) {
        fmt.Println("American 的首都是", capital)
    } else {
        fmt.Println("American 的首都不存在")
    }
}
```

以上实例运行结果为：

```shell
France 首都是 巴黎
Italy 首都是 罗马
Japan 首都是 东京
India  首都是 新德里
American 的首都不存在
```

## 18.2 delete() 函数

delete() 函数用于删除集合的元素, 参数为 map 和其对应的 key。实例如下：

```go
package main

import "fmt"

func main() {
        /* 创建map */
        countryCapitalMap := map[string]string{"France": "Paris", "Italy": "Rome", "Japan": "Tokyo", "India": "New delhi"}

        fmt.Println("原始地图")

        /* 打印地图 */
        for country := range countryCapitalMap {
                fmt.Println(country, "首都是", countryCapitalMap [ country ])
        }

        /*删除元素*/ delete(countryCapitalMap, "France")
        fmt.Println("法国条目被删除")

        fmt.Println("删除元素后地图")

        /*打印地图*/
        for country := range countryCapitalMap {
                fmt.Println(country, "首都是", countryCapitalMap [ country ])
        }
}
```

以上实例运行结果为：

```shell
原始地图
India 首都是 New delhi
France 首都是 Paris
Italy 首都是 Rome
Japan 首都是 Tokyo
法国条目被删除
删除元素后地图
Italy 首都是 Rome
Japan 首都是 Tokyo
India 首都是 New delhi
```

> 下方评论留言：
>
> + 基于 go 实现简单 HashMap，暂未做 key 值的校验。
>
>   ```go
>   package main
>     
>   import (
>       "fmt"
>   )
>     
>   type HashMap struct {
>       key string
>       value string
>       hashCode int
>       next *HashMap
>   }
>     
>   var table [16](*HashMap)
>     
>   func initTable() {
>       for i := range table{
>           table[i] = &HashMap{"","",i,nil}
>       }
>   }
>     
>   func getInstance() [16](*HashMap){
>       if(table[0] == nil){
>           initTable()
>       }
>       return table
>   }
>     
>   func genHashCode(k string) int{
>       if len(k) == 0{
>           return 0
>       }
>       var hashCode int = 0
>       var lastIndex int = len(k) - 1
>       for i := range k {
>           if i == lastIndex {
>               hashCode += int(k[i])
>               break
>           }
>           hashCode += (hashCode + int(k[i]))*31
>       }
>       return hashCode
>   }
>     
>   func indexTable(hashCode int) int{
>       return hashCode%16
>   }
>     
>   func indexNode(hashCode int) int {
>       return hashCode>>4
>   }
>     
>   func put(k string, v string) string {
>       var hashCode = genHashCode(k)
>       var thisNode = HashMap{k,v,hashCode,nil}
>     
>       var tableIndex = indexTable(hashCode)
>       var nodeIndex = indexNode(hashCode)
>     
>       var headPtr [16](*HashMap) = getInstance()
>       var headNode = headPtr[tableIndex]
>     
>       if (*headNode).key == "" {
>           *headNode = thisNode
>           return ""
>       }
>     
>       var lastNode *HashMap = headNode
>       var nextNode *HashMap = (*headNode).next
>     
>       for nextNode != nil && (indexNode((*nextNode).hashCode) < nodeIndex){
>           lastNode = nextNode
>           nextNode = (*nextNode).next
>       }
>       if (*lastNode).hashCode == thisNode.hashCode {
>           var oldValue string = lastNode.value
>           lastNode.value = thisNode.value
>           return oldValue
>       }
>       if lastNode.hashCode < thisNode.hashCode {
>           lastNode.next = &thisNode
>       }
>       if nextNode != nil {
>           thisNode.next = nextNode
>       }
>       return ""
>   }
>     
>   func get(k string) string {
>       var hashCode = genHashCode(k)
>       var tableIndex = indexTable(hashCode)
>     
>       var headPtr [16](*HashMap) = getInstance()
>       var node *HashMap = headPtr[tableIndex]
>     
>       if (*node).key == k{
>           return (*node).value
>       }
>     
>       for (*node).next != nil {
>           if k == (*node).key {
>               return (*node).value
>           }
>           node = (*node).next
>       }
>       return ""
>   }
>     
>   //examples 
>   func main() {
>       getInstance()
>       put("a","a_put")
>       put("b","b_put")
>       fmt.Println(get("a"))
>       fmt.Println(get("b"))
>       put("p","p_put")
>       fmt.Println(get("p"))
>   }
>   ```

# 19. Go 语言递归函数

递归，就是在运行的过程中调用自己。

语法格式如下：

```go
func recursion() {
   recursion() /* 函数调用自身 */
}

func main() {
   recursion()
}
```

Go 语言支持递归。但我们在使用递归时，开发者需要设置退出条件，否则递归将陷入无限循环中。

递归函数对于解决数学上的问题是非常有用的，就像计算阶乘，生成斐波那契数列等。

## 19.1 阶乘

以下实例通过 Go 语言的递归函数实例阶乘：

```go
package main

import "fmt"

func Factorial(n uint64)(result uint64) {
    if (n > 0) {
        result = n * Factorial(n-1)
        return result
    }
    return 1
}

func main() {  
    var i int = 15
    fmt.Printf("%d 的阶乘是 %d\n", i, Factorial(uint64(i)))
}
```

以上实例执行输出结果为：

```shell
15 的阶乘是 1307674368000
```

## 19.2 斐波那契数列

```go
package main

import "fmt"

func fibonacci(n int) int {
  if n < 2 {
   return n
  }
  return fibonacci(n-2) + fibonacci(n-1)
}

func main() {
    var i int
    for i = 0; i < 10; i++ {
       fmt.Printf("%d\t", fibonacci(i))
    }
}
```

以上实例执行输出结果为：

```
0    1    1    2    3    5    8    13    21    34
```

# 20. Go 语言类型转换

类型转换用于将一种数据类型的变量转换为另外一种类型的变量。Go 语言类型转换基本格式如下：

```go
type_name(expression)
```

type_name 为类型，expression 为表达式。

以下实例中将整型转化为浮点型，并计算结果，将结果赋值给浮点型变量：

```go
package main

import "fmt"

func main() {
   var sum int = 17
   var count int = 5
   var mean float32
   
   mean = float32(sum)/float32(count)
   fmt.Printf("mean 的值为: %f\n",mean)
}
```

以上实例执行输出结果为：

```shell
mean 的值为: 3.400000
```

> 下方评论留言：
>
> go 不支持隐式转换类型，比如 :
>
> ```go
> package main
> import "fmt"
> 
> func main() {  
>     var a int64 = 3
>     var b int32
>     b = a
>     fmt.Printf("b 为 : %d", b)
> }
> ```
>
> 此时会报错
>
> ```shell
> cannot use a (type int64) as type int32 in assignment
> cannot use b (type int32) as type string in argument to fmt.Printf
> ```
>
> 但是如果改成 **b = int32(a)** 就不会报错了:
>
> ```go
> package main
> import "fmt"
> 
> func main() {  
>     var a int64 = 3
>     var b int32
>     b = int32(a)
>     fmt.Printf("b 为 : %d", b)
> }
> ```

# 21. Go 语言接口

Go 语言提供了另外一种数据类型即接口，它把所有的具有共性的方法定义在一起，**任何其他类型只要实现了这些方法就是实现了这个接口。**

```go
/* 定义接口 */
type interface_name interface {
   method_name1 [return_type]
   method_name2 [return_type]
   method_name3 [return_type]
   ...
   method_namen [return_type]
}

/* 定义结构体 */
type struct_name struct {
   /* variables */
}

/* 实现接口方法 */
func (struct_name_variable struct_name) method_name1() [return_type] {
   /* 方法实现 */
}
...
func (struct_name_variable struct_name) method_namen() [return_type] {
   /* 方法实现*/
}
```

实例

```go
package main

import (
    "fmt"
)

type Phone interface {
    call()
}

type NokiaPhone struct {
}

func (nokiaPhone NokiaPhone) call() {
    fmt.Println("I am Nokia, I can call you!")
}

type IPhone struct {
}

func (iPhone IPhone) call() {
    fmt.Println("I am iPhone, I can call you!")
}

func main() {
    var phone Phone

    phone = new(NokiaPhone)
    phone.call()

    phone = new(IPhone)
    phone.call()

}
```

在上面的例子中，我们定义了一个接口Phone，接口里面有一个方法call()。然后我们在main函数里面定义了一个Phone类型变量，并分别为之赋值为NokiaPhone和IPhone。然后调用call()方法，输出结果如下：

```shell
I am Nokia, I can call you!
I am iPhone, I can call you!
```

> 下方评论留言：
>
> + 如果想要通过接口方法修改属性，需要在传入指针的结构体才行，具体代码入下的 [1][2] 处：
>
>   ```go
>   type fruit interface{
>       getName() string   
>       setName(name string)
>   }
>   type apple struct{   
>       name string
>   }
>   //[1]
>   func (a *apple) getName() string{   
>       return a.name
>   }
>   //[2]
>   func (a *apple) setName(name string) {
>      a.name = name
>   }
>   func testInterface(){   
>       a:=apple{"红富士"}   
>       fmt.Print(a.getName())   
>       a.setName("树顶红")   
>       fmt.Print(a.getName())
>   }
>   ```

# 22. Go 错误处理

**Go 语言通过内置的错误接口提供了非常简单的错误处理机制**。

**error类型是一个接口类型**，这是它的定义：

```go
type error interface {
    Error() string
}
```

我们可以在编码中通过实现 error 接口类型来生成错误信息。

函数通常在最后的返回值中返回错误信息。使用errors.New 可返回一个错误信息：

```go
func Sqrt(f float64) (float64, error) {
    if f < 0 {
        return 0, errors.New("math: square root of negative number")
    }
    // 实现
}
```

在下面的例子中，我们在调用Sqrt的时候传递的一个负数，然后就得到了non-nil的error对象，将此对象与nil比较，结果为true，所以fmt.Println(fmt包在处理error时会调用Error方法)被调用，以输出错误，请看下面调用的示例代码：

```go
result, err:= Sqrt(-1)

if err != nil {
   fmt.Println(err)
}
```

实例

```go
package main

import (
    "fmt"
)

// 定义一个 DivideError 结构
type DivideError struct {
    dividee int
    divider int
}

// 实现 `error` 接口
func (de *DivideError) Error() string {
    strFormat := `
    Cannot proceed, the divider is zero.
    dividee: %d
    divider: 0
`
    return fmt.Sprintf(strFormat, de.dividee)
}

// 定义 `int` 类型除法运算的函数
func Divide(varDividee int, varDivider int) (result int, errorMsg string) {
    if varDivider == 0 {
            dData := DivideError{
                    dividee: varDividee,
                    divider: varDivider,
            }
            errorMsg = dData.Error()
            return
    } else {
            return varDividee / varDivider, ""
    }

}

func main() {

    // 正常情况
    if result, errorMsg := Divide(100, 10); errorMsg == "" {
            fmt.Println("100/10 = ", result)
    }
    // 当除数为零的时候会返回错误信息
    if _, errorMsg := Divide(100, 0); errorMsg != "" {
            fmt.Println("errorMsg is: ", errorMsg)
    }

}
```

执行以上程序，输出结果为：

```shell
100/10 =  10
errorMsg is:  
    Cannot proceed, the divider is zero.
    dividee: 100
    divider: 0
```

> 下方评论留言：
>
> - <u>这里应该介绍一下 panic 与 recover,一个用于**主动抛出错误**，一个用于**捕获panic抛出的错误**。</u>
>
>   **概念**
>
>   panic 与 recover 是 Go 的两个内置函数，这两个内置函数用于处理 Go 运行时的错误，**panic 用于主动抛出错误，recover 用来捕获 panic 抛出的错误**。
>
>   ![img](https://www.runoob.com/wp-content/uploads/2019/01/99459110.jpg)
>
>   - 引发`panic`有两种情况，一是程序主动调用，二是程序产生运行时错误，由运行时检测并退出。
>   - 发生`panic`后，程序会从调用`panic`的函数位置或发生`panic`的地方立即返回，逐层向上执行函数的`defer`语句，然后逐层打印函数调用堆栈，直到被`recover`捕获或运行到最外层函数。
>   - `panic`不但可以在函数正常流程中抛出，在`defer`逻辑里也可以再次调用`panic`或抛出`panic`。`defer`里面的`panic`能够被后续执行的`defer`捕获。
>   - `recover`用来捕获`panic`，阻止`panic`继续向上传递。`recover()`和`defer`一起使用，但是`defer`只有在后面的函数体内直接被掉用才能捕获`panic`来终止异常，否则返回`nil`，异常继续向外传递。
>
>   例子1
>
>   ```go
>   //以下捕获失败
>   defer recover()
>   defer fmt.Prinntln(recover)
>   defer func(){
>       func(){
>           recover() //无效，嵌套两层
>       }()
>   }()
>   
>   //以下捕获有效
>   defer func(){
>       recover()
>   }()
>   
>   func except(){
>       recover()
>   }
>   func test(){
>       defer except()
>       panic("runtime error")
>   }
>   ```
>
>   例子2
>
>   多个panic只会捕捉最后一个：
>
>   ```go
>   package main
>   import "fmt"
>   func main(){
>       defer func(){
>           if err := recover() ; err != nil {
>               fmt.Println(err)
>           }
>       }()
>       defer func(){
>           panic("three")
>       }()
>       defer func(){
>           panic("two")
>       }()
>       panic("one")
>   }
>   // 输出 three
>   ```
>
>   **使用场景**
>
>   一般情况下有两种情况用到：
>
>   -  程序遇到无法执行下去的错误时，抛出错误，主动结束运行。
>   -  在调试程序时，通过 panic 来打印堆栈，方便定位错误。
>
> - fmt.Println 打印结构体的时候，会把其中的 error 的返回的信息打印出来。
>
>   ```go
>   type User struct {
>      username string
>      password string
>   }
>     
>   func (p *User) init(username string ,password string) (*User,string)  {
>      if ""==username || ""==password {
>         return p,p.Error()
>      }
>      p.username = username
>      p.password = password
>      return p,""}
>     
>   func (p *User) Error() string {
>         return "Usernam or password shouldn't be empty!"}
>   }
>     
>   func main() {
>      var user User
>      user1, _ :=user.init("","");
>      fmt.Println(user1)
>   }
>   ```
>
>   结果：
>
>   ```shell
>   Usernam or password shouldn't be empty!
>   ```

# 23. Go 并发

Go 语言支持并发，我们只需要通过 go 关键字来开启 goroutine 即可。

**goroutine 是轻量级线程**，goroutine 的调度是由 Golang 运行时进行管理的。

goroutine 语法格式：

```go
go 函数名( 参数列表 )
```

例如：

```go
go f(x, y, z)
```

开启一个新的 goroutine:

```go
f(x, y, z)
```

Go 允许使用 go 语句开启一个新的运行期线程， 即 goroutine，以一个不同的、新创建的 goroutine 来执行一个函数。 **同一个程序中的所有 goroutine 共享同一个地址空间**。

```go
package main

import (
        "fmt"
        "time"
)

func say(s string) {
        for i := 0; i < 5; i++ {
                time.Sleep(100 * time.Millisecond)
                fmt.Println(s)
        }
}

func main() {
        go say("world")
        say("hello")
}
```

执行以上代码，你会看到输出的 hello 和 world 是没有固定先后顺序。因为它们是两个 goroutine 在执行：

```shell
world
hello
hello
world
world
hello
hello
world
world
hello
```

## 23.1 通道（channel）

**通道（channel）是用来传递数据的一个数据结构。**

通道可用于**两个 goroutine 之间**通过传递一个指定类型的值来同步运行和通讯。

**操作符 `<-` 用于指定通道的方向，发送或接收**。

**如果未指定方向，则为双向通道**。

```
ch <- v    // 把 v 发送到通道 ch
v := <-ch  // 从 ch 接收数据
           // 并把值赋给 v
```

声明一个通道很简单，我们使用**chan关键字**即可，通道在使用前必须先创建：

```
ch := make(chan int)
```

**注意**：**默认情况下，通道是不带缓冲区的**。<u>发送端发送数据，同时必须有接收端相应的接收数据</u>。

以下实例通过两个 goroutine 来计算数字之和，在 goroutine 完成计算后，它会计算两个结果的和：

```go
package main

import "fmt"

func sum(s []int, c chan int) {
        sum := 0
        for _, v := range s {
                sum += v
        }
        c <- sum // 把 sum 发送到通道 c
}

func main() {
        s := []int{7, 2, 8, -9, 4, 0}

        c := make(chan int)
        go sum(s[:len(s)/2], c)
        go sum(s[len(s)/2:], c)
        x, y := <-c, <-c // 从通道 c 中接收

        fmt.Println(x, y, x+y)
}
```

输出结果为：

```shell
-5 17 12
```

### 通道缓冲区

通道可以设置缓冲区，通过 make 的第二个参数指定缓冲区大小：

```go
ch := make(chan int, 100)
```

**带缓冲区的通道允许发送端的数据发送和接收端的数据获取处于异步状态，就是说发送端发送的数据可以放在缓冲区里面，可以等待接收端去获取数据，而不是立刻需要接收端去获取数据。**

不过由于缓冲区的大小是有限的，所以还是必须有接收端来接收数据的，否则缓冲区一满，数据发送端就无法再发送数据了。

**注意**：

+ **如果通道不带缓冲，发送方会阻塞直到接收方从通道中接收了值**。
+ **如果通道带缓冲，发送方则会阻塞直到发送的值被拷贝到缓冲区内**；
+ **如果缓冲区已满，则意味着需要等待直到某个接收方获取到一个值**。
+ **接收方在有值可以接收之前会一直阻塞**。

```go
package main

import "fmt"

func main() {
    // 这里我们定义了一个可以存储整数类型的带缓冲通道
        // 缓冲区大小为2
        ch := make(chan int, 2)

        // 因为 ch 是带缓冲的通道，我们可以同时发送两个数据
        // 而不用立刻需要去同步读取数据
        ch <- 1
        ch <- 2

        // 获取这两个数据
        fmt.Println(<-ch)
        fmt.Println(<-ch)
}
```

执行输出结果为：

```shell
1
2
```

### Go 遍历通道与关闭通道

Go 通过 range 关键字来实现遍历读取到的数据，类似于与数组或切片。格式如下：

```go
v, ok := <-ch
```

**如果通道接收不到数据后 ok 就为 false**，这时通道就可以使用 **close()** 函数来关闭。

```go
package main

import (
        "fmt"
)

func fibonacci(n int, c chan int) {
        x, y := 0, 1
        for i := 0; i < n; i++ {
                c <- x
                x, y = y, x+y
        }
        close(c)
}

func main() {
        c := make(chan int, 10)
        go fibonacci(cap(c), c)
        // range 函数遍历每个从通道接收到的数据，因为 c 在发送完 10 个
        // 数据之后就关闭了通道，所以这里我们 range 函数在接收到 10 个数据
        // 之后就结束了。如果上面的 c 通道不关闭，那么 range 函数就不
        // 会结束，从而在接收第 11 个数据的时候就阻塞了。
        for i := range c {
                fmt.Println(i)
        }
}
```

执行输出结果为：

```shell
0
1
1
2
3
5
8
13
21
34
```

> 下方评论区留言：
>
> + goroutine 是 golang 中在语言级别实现的轻量级线程，仅仅利用 go 就能立刻起一个新线程。多线程会引入线程之间的同步问题，在 golang 中可以使用 channel 作为同步的工具。
>
>   通过 channel 可以实现两个 goroutine 之间的通信。
>
>   创建一个 channel， make(chan TYPE {, NUM}) TYPE 指的是 channel 中传输的数据类型，第二个参数是可选的，指的是 channel 的容量大小。
>
>   向 channel 传入数据， CHAN <- DATA ， CHAN 指的是目的 channel 即收集数据的一方， DATA 则是要传的数据。
>
>   从 channel 读取数据， DATA := <-CHAN ，和向 channel 传入数据相反，在数据输送箭头的右侧的是 channel，形象地展现了数据从隧道流出到变量里。
>
> + 我们单独写一个 say2 函数来跑 goroutine，并且 Sleep 时间设置长一点，150 毫秒，看看会发生什么：
>
>   ```go
>   package main
>   import (
>       "fmt"
>       "time"
>   )
>   func say(s string) {
>       for i := 0; i < 5; i++ {
>           time.Sleep(100 * time.Millisecond)
>           fmt.Println(s, (i+1)*100)
>       }
>   }
>   func say2(s string) {
>       for i := 0; i < 5; i++ {
>           time.Sleep(150 * time.Millisecond)
>           fmt.Println(s, (i+1)*150)
>       }
>   }
>   func main() {
>       go say2("world")
>       say("hello")
>   }
>   ```
>
>   输出结果：
>
>   ```shell
>   hello 100
>   world 150
>   hello 200
>   hello 300
>   world 300
>   hello 400
>   world 450
>   hello 500
>
>   [Done] exited with code=0 in 2.066 seconds
>   ```
>
>   问题来了，say2 只执行了 3 次，而不是设想的 5 次，为什么呢？
>
>   原来，**在 goroutine 还没来得及跑完 5 次的时候，主函数已经退出了**。
>
>   我们要想办法阻止主函数的结束，要等待 goroutine 执行完成之后，再退出主函数：
>
>   ```go
>   package main
>
>   import (
>       "fmt"
>       "time"
>   )
>
>   func say(s string) {
>       for i := 0; i < 5; i++ {
>           time.Sleep(100 * time.Millisecond)
>           fmt.Println(s, (i+1)*100)
>       }
>   }
>   func say2(s string, ch chan int) {
>       for i := 0; i < 5; i++ {
>           time.Sleep(150 * time.Millisecond)
>           fmt.Println(s, (i+1)*150)
>       }
>       ch <- 0
>       close(ch)
>   }
>
>   func main() {
>       ch := make(chan int)
>       go say2("world", ch)
>       say("hello")
>       fmt.Println(<-ch)
>   }
>   ```
>
>   输出如下：
>
>   ```shell
>   hello 100
>   world 150
>   hello 200
>   world 300
>   hello 300
>   hello 400
>   world 450
>   hello 500
>   world 600
>   world 750
>   0
>   ```
>
>   我们引入一个信道，默认的，信道的存消息和取消息都是阻塞的，在 goroutine 中执行完成后给信道一个值 0，则主函数会一直等待信道中的值，一旦信道有值，主函数才会结束。
>
> + 关闭通道并不会丢失里面的数据，只是让读取通道数据的时候不会读完之后一直阻塞等待新数据写入
>
> + **Channel 是可以控制读写权限的**。具体如下:
>
>   ```go
>   go func(c chan int) { //读写均可的channel c } (a)
>   go func(c <- chan int) { //只读的Channel } (a)
>   go func(c chan <- int) {  //只写的Channel } (a)
>   ```
>
> + ```shell
>   package main
>     
>   import "fmt"
>     
>   func main() {
>       ch := make(chan int, 2)
>     
>       ch <- 1
>       a := <-ch
>       ch <- 2
>       ch <- 3
>     
>       fmt.Println(<-ch)
>       fmt.Println(<-ch)
>       fmt.Println(a)
>   }
>   ```
>
>   **通道遵循先进先出原则**。
>
>   不带缓冲区的通道在向通道发送值时，必须及时接收，且必须一次接收完成。
>
>   而带缓冲区的通道则会以缓冲区满而阻塞，直到先塞发送到通道的值被从通道中接收才可以继续往通道传值。就像往水管里推小钢珠一样，如果钢珠塞满没有从另一头放出，那么这一头就没法再往里塞，是一个道理。例如上面的例子，最多只能让同时在通道中停放2个值，想多传值，就需要把前面的值提前从通道中接收出去。
>
>   因此，上面的输出结果为：
>
>   ```
>   2
>   3
>   1
>   ```

# 24. Go 语言开发工具

1. GoLand
2. LiteIDE
3. Eclipse

