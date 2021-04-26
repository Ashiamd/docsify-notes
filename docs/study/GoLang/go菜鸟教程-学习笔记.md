# go菜鸟教程-学习笔记

> [Go 语言教程](https://www.runoob.com/go/go-tutorial.html)

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

