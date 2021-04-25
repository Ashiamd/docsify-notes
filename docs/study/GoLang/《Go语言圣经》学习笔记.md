# 《Go语言圣经》学习笔记

> [Go语言圣经（中文版）--在线阅读](http://books.studygolang.com/gopl-zh/)
>
> 建议先去过一遍go语言的菜鸟教程

# 0. 前言

Go语言类C，适合编写网络服务相关的基础设施。

## 0.1 Go语言起源

![img](http://books.studygolang.com/gopl-zh/images/ch0-01.png)

## 0.2 Go语言项目

**"优势"**

+ **垃圾回收**（Java）
+ 包系统（Java）
+ **函数一等公民**（Scala）
+ 只读的UTF8字符串
+ **向后兼容**
+ **轻量级线程goroutine**（用户线程）
+ ...

**"劣势"**

+ 无隐式数值转换（C、C++、java）
+ 无构造函数和析构函数（典型C++）
+ 无运算符重载（C++）
+ 无默认参数（javascript、C++、Scala）
+ **无继承**（C++、Java、Scala）
+ 无泛型（C++、Java、Scala）
+ 无异常（C、C++、Javascript、Java、Scala）

+ 无有宏（C）
+ 无函数修饰（C++、java、Scala）
+ 无线程局部存储（Java）
+ ...

## 0.3 本书的组织

书中所有的代码都可以从 [http://gopl.io](http://gopl.io/) 上的Git仓库下载。go get命令根据每个例子的导入路径智能地获取、构建并安装。只需要选择一个目录作为工作空间，然后将GOPATH环境变量设置为该路径。

必要时，Go语言工具会创建目录。例如：

```
$ export GOPATH=$HOME/gobook    # 选择工作目录
$ go get gopl.io/ch1/helloworld # 获取/编译/安装
$ $GOPATH/bin/helloworld        # 运行程序
Hello, 世界                     # 这是中文
```

运行这些例子需要安装Go1.5以上的版本。

```
$ go version
go version go1.5 linux/amd64
```

如果使用其他的操作系统，请参考 https://golang.org/doc/install 提供的说明安装。

## 0.4 更多的信息

> [go官网](https://golang.org/)
>
> [Go语言的官方博客](https://blog.golang.org/)
>
> [Go Playground](https://play.golang.org/)

# 1. 入门

## 1.1 Hello World

```go
// helloworld.go
// go run helloworld.go
package main
import "fmt"
func main(){
	fmt.Println("Hello, World")
}
```

