# 《Go语言圣经》学习笔记

> [Go语言圣经（中文版）--在线阅读](http://books.studygolang.com/gopl-zh/)
>
> 建议先去过一遍[go语言的菜鸟教程](https://www.runoob.com/go/go-tutorial.html)

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

Go是一门**编译型语言**，Go语言的工具链将源代码及其依赖转换成计算机的机器指令（译注：静态编译）。

Go语言提供的工具都通过一个单独的命令`go`调用，`go`命令有一系列子命令。最简单的一个子命令就是run。这个命令编译一个或多个以.go结尾的源文件，链接库文件，并运行最终生成的可执行文件。（本书使用$表示命令行提示符。）

```shell
$ go run helloworld.go
```

如果不只是一次性实验，你肯定希望能够**编译**这个程序，保存编译结果以备将来之用。

```
$ go build helloworld.go
```

这个命令生成一个名为helloworld的可执行的二进制文件。

+ Go语言的代码通过**包**（package）组织，包类似于其它语言里的库（libraries）或者模块（modules）。一个包由位于单个目录下的一个或多个.go源代码文件组成，目录定义包的作用。

+ `main`包比较特殊。它定义了一个独立可执行的程序，而不是一个库。在`main`里的`main` *函数* 也很特殊，它是整个程序执行时的入口（译注：C系语言差不多都这样）。`main`函数所做的事情就是程序做的。当然了，`main`函数一般调用其它包里的函数完成很多工作（如：`fmt.Println`）。

+ Go语言在代码格式上采取了很强硬的态度。`gofmt`工具把代码格式化为标准格式（译注：这个格式化工具没有任何可以调整代码格式的参数，Go语言就是这么任性），并且`go`工具中的`fmt`子命令会对指定包，否则默认为当前目录中所有.go源文件应用`gofmt`命令。

+ 很多文本编辑器都可以配置为保存文件时自动执行`gofmt`，这样你的源代码总会被恰当地格式化。还有个相关的工具，`goimports`，可以根据代码需要，自动地添加或删除`import`声明。这个工具并没有包含在标准的分发包中，可以用下面的命令安装：

  ```
  $ go get golang.org/x/tools/cmd/goimports
  ```

  对于大多数用户来说，下载、编译包、运行测试用例、察看Go语言的文档等等常用功能都可以用go的工具完成。

## 1.2 命令行参数

+ `os`包以跨平台的方式，提供了一些与操作系统交互的函数和变量。程序的命令行参数可从os包的Args变量获取；os包外部使用os.Args访问该变量。

+ os.Args的第一个元素：os.Args[0]，是命令本身的名字；其它的元素则是程序启动时传给它的参数。

  <small>s[m:n]形式的切片表达式，产生从第m个元素到第n-1个元素的切片，下个例子用到的元素包含在os.Args[1:len(os.Args)]切片中。如果省略切片表达式的m或n，会默认传入0或len(s)，因此前面的切片可以简写成os.Args[1:]。</small>

示例代码1

```go
// Echo1 prints its command-line arguments.
package main

import (
    "fmt"
    "os"
)

func main() {
    var s, sep string
    for i := 1; i < len(os.Args); i++ {
        s += sep + os.Args[i]
        sep = " "
    }
    fmt.Println(s)
}
```

示例代码2

```go
// Echo2 prints its command-line arguments.
package main

import (
    "fmt"
    "os"
)

func main() {
    s, sep := "", ""
    for _, arg := range os.Args[1:] {
        s += sep + arg
        sep = " "
    }
    fmt.Println(s)
}
```

示例代码3

```go
func main() {
    fmt.Println(strings.Join(os.Args[1:], " "))
}
```

## 1.3 查找重复的行

示例代码：

```go
// Dup1 prints the text of each line that appears more than
// once in the standard input, preceded by its count.
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    counts := make(map[string]int)
    input := bufio.NewScanner(os.Stdin)
    for input.Scan() {
        counts[input.Text()]++
    }
    // NOTE: ignoring potential errors from input.Err()
    for line, n := range counts {
        if n > 1 {
            fmt.Printf("%d\t%s\n", n, line)
        }
    }
}
```

`bufio`包，它使处理输入和输出方便又高效。`Scanner`类型是该包最有用的特性之一，它读取输入并将其拆成行或单词；通常是处理行形式的输入最简单的方法。

程序使用短变量声明创建`bufio.Scanner`类型的变量`input`。

程序使用短变量声明创建`bufio.Scanner`类型的变量`input`。

```
input := bufio.NewScanner(os.Stdin)
```

该变量从程序的标准输入中读取内容。每次调用`input.Scan()`，即读入下一行，并移除行末的换行符；读取的内容可以调用`input.Text()`得到。`Scan`函数在读到一行时返回`true`，不再有输入时返回`false`。

`Printf`有一大堆这种转换，Go程序员称之为*动词（verb）*。下面的表格虽然远不是完整的规范，但展示了可用的很多特性：

```
%d          十进制整数
%x, %o, %b  十六进制，八进制，二进制整数。
%f, %g, %e  浮点数： 3.141593 3.141592653589793 3.141593e+00
%t          布尔：true或false
%c          字符（rune） (Unicode码点)
%s          字符串
%q          带双引号的字符串"abc"或带单引号的字符'c'
%v          变量的自然形式（natural format）
%T          变量的类型
%%          字面上的百分号标志（无操作数）
```

`dup1`的格式字符串中还含有制表符`\t`和换行符`\n`。字符串字面上可能含有这些代表不可见字符的**转义字符（escape sequences）**。默认情况下，`Printf`不会换行。按照惯例，以字母`f`结尾的格式化函数，如`log.Printf`和`fmt.Errorf`，都采用`fmt.Printf`的格式化准则。而以`ln`结尾的格式化函数，则遵循`Println`的方式，以跟`%v`差不多的方式格式化参数，并在最后添加一个换行符。（译注：后缀`f`指`format`，`ln`指`line`。）

很多程序要么从标准输入中读取数据，如上面的例子所示，要么从一系列具名文件中读取数据。`dup`程序的下个版本读取标准输入或是使用`os.Open`打开各个具名文件，并操作它们。

```go
// Dup2 prints the count and text of lines that appear more than once
// in the input.  It reads from stdin or from a list of named files.
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    counts := make(map[string]int)
    files := os.Args[1:]
    if len(files) == 0 {
        countLines(os.Stdin, counts)
    } else {
        for _, arg := range files {
            f, err := os.Open(arg)
            if err != nil {
                fmt.Fprintf(os.Stderr, "dup2: %v\n", err)
                continue
            }
            countLines(f, counts)
            f.Close()
        }
    }
    for line, n := range counts {
        if n > 1 {
            fmt.Printf("%d\t%s\n", n, line)
        }
    }
}

func countLines(f *os.File, counts map[string]int) {
    input := bufio.NewScanner(f)
    for input.Scan() {
        counts[input.Text()]++
    }
    // NOTE: ignoring potential errors from input.Err()
}
```

`os.Open`函数返回两个值。第一个值是被打开的文件(`*os.File`），其后被`Scanner`读取。

`os.Open`返回的第二个值是内置`error`类型的值。如果`err`等于内置值`nil`（译注：相当于其它语言里的NULL），那么文件被成功打开。读取文件，直到文件结束，然后调用`Close`关闭该文件，并释放占用的所有资源。相反的话，如果`err`的值不是`nil`，说明打开文件时出错了。这种情况下，错误值描述了所遇到的问题。我们的错误处理非常简单，只是使用`Fprintf`与表示任意类型默认格式值的动词`%v`，向标准错误流打印一条信息，然后`dup`继续处理下一个文件；`continue`语句直接跳到`for`循环的下个迭代开始执行。

注意`countLines`函数在其声明前被调用。**函数和包级别的变量（package-level entities）可以任意顺序声明，并不影响其被调用**。（译注：最好还是遵循一定的规范）

`dup`的前两个版本以"流”模式读取输入，并根据需要拆分成多个行。理论上，这些程序可以处理任意数量的输入数据。还有另一个方法，就是一口气把全部输入数据读到内存中，一次分割为多行，然后处理它们。下面这个版本，`dup3`，就是这么操作的。这个例子引入了`ReadFile`函数（来自于`io/ioutil`包），其读取指定文件的全部内容，`strings.Split`函数把字符串分割成子串的切片。（`Split`的作用与前文提到的`strings.Join`相反。）

我们略微简化了`dup3`。首先，由于`ReadFile`函数需要文件名作为参数，因此只读指定文件，不读标准输入。其次，由于行计数代码只在一处用到，故将其移回`main`函数。

```go
package main

import (
    "fmt"
    "io/ioutil"
    "os"
    "strings"
)

func main() {
    counts := make(map[string]int)
    for _, filename := range os.Args[1:] {
        data, err := ioutil.ReadFile(filename)
        if err != nil {
            fmt.Fprintf(os.Stderr, "dup3: %v\n", err)
            continue
        }
        for _, line := range strings.Split(string(data), "\n") {
            counts[line]++
        }
    }
    for line, n := range counts {
        if n > 1 {
            fmt.Printf("%d\t%s\n", n, line)
        }
    }
}
```

`ReadFile`函数返回一个字节切片（byte slice），必须把它转换为`string`，才能用`strings.Split`分割。我们会在3.5.4节详细讲解字符串和字节切片。

实现上，`bufio.Scanner`、`ioutil.ReadFile`和`ioutil.WriteFile`都使用`*os.File`的`Read`和`Write`方法，但是，大多数程序员很少需要直接调用那些低级（lower-level）函数。高级（higher-level）函数，像`bufio`和`io/ioutil`包中所提供的那些，用起来要容易点。

## 1.4 GIF动画

下面的程序会演示Go语言标准库里的image这个package的用法，我们会用这个包来生成一系列的bit-mapped图，然后将这些图片编码为一个GIF动画。我们生成的图形名字叫利萨如图形（Lissajous figures），这种效果是在1960年代的老电影里出现的一种视觉特效。它们是协振子在两个纬度上振动所产生的曲线，比如两个sin正弦波分别在x轴和y轴输入会产生的曲线。图1.1是这样的一个例子：
![img](http://books.studygolang.com/gopl-zh/images/ch1-01.png)

译注：要看这个程序的结果，需要将标准输出重定向到一个GIF图像文件（使用 `./lissajous > output.gif` 命令）。下面是GIF图像动画效果：

![img](http://books.studygolang.com/gopl-zh/images/ch1-01.gif)

代码如下：

```go
// Lissajous generates GIF animations of random Lissajous figures.
package main

import (
    "image"
    "image/color"
    "image/gif"
    "io"
    "math"
    "math/rand"
    "os"
    "time"
)

var palette = []color.Color{color.White, color.Black}

const (
    whiteIndex = 0 // first color in palette
    blackIndex = 1 // next color in palette
)

func main() {
    // The sequence of images is deterministic unless we seed
    // the pseudo-random number generator using the current time.
    // Thanks to Randall McPherson for pointing out the omission.
    rand.Seed(time.Now().UTC().UnixNano())
    lissajous(os.Stdout)
}

func lissajous(out io.Writer) {
    const (
        cycles  = 5     // number of complete x oscillator revolutions
        res     = 0.001 // angular resolution
        size    = 100   // image canvas covers [-size..+size]
        nframes = 64    // number of animation frames
        delay   = 8     // delay between frames in 10ms units
    )

    freq := rand.Float64() * 3.0 // relative frequency of y oscillator
    anim := gif.GIF{LoopCount: nframes}
    phase := 0.0 // phase difference
    for i := 0; i < nframes; i++ {
        rect := image.Rect(0, 0, 2*size+1, 2*size+1)
        img := image.NewPaletted(rect, palette)
        for t := 0.0; t < cycles*2*math.Pi; t += res {
            x := math.Sin(t)
            y := math.Sin(t*freq + phase)
            img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5),
                blackIndex)
        }
        phase += 0.1
        anim.Delay = append(anim.Delay, delay)
        anim.Image = append(anim.Image, img)
    }
    gif.EncodeAll(out, &anim) // NOTE: ignoring encoding errors
}
```

这个程序里的常量声明给出了一系列的常量值，常量是指在程序编译后运行时始终都不会变化的值，比如圈数、帧数、延迟值。**常量声明和变量声明一般都会出现在包级别**，所以这些常量在整个包中都是可以共享的，或者你也可以把常量声明定义在函数体内部，那么这种常量就只能在函数体内用。目前常量声明的值必须是一个数字值、字符串或者一个固定的boolean值。

lissajous函数内部有两层嵌套的for循环。外层循环会循环64次，每一次都会生成一个单独的动画帧。它生成了一个包含两种颜色的201*201大小的图片，白色和黑色。所有像素点都会被默认设置为其零值（也就是调色板palette里的第0个值），这里我们设置的是白色。每次外层循环都会生成一张新图片，并将一些像素设置为黑色。其结果会append到之前结果之后。这里我们用到了append(参考4.2.1)内置函数，将结果append到anim中的帧列表末尾，并设置一个默认的80ms的延迟值。循环结束后所有的延迟值被编码进了GIF图片中，并将结果写入到输出流。out这个变量是io.Writer类型，这个类型支持把输出结果写到很多目标，很快我们就可以看到例子。

内层循环设置两个偏振值。x轴偏振使用sin函数。y轴偏振也是正弦波，但其相对x轴的偏振是一个0-3的随机值，初始偏振值是一个零值，随着动画的每一帧逐渐增加。循环会一直跑到x轴完成五次完整的循环。每一步它都会调用SetColorIndex来为(x,y)点来染黑色。

main函数调用lissajous函数，用它来向标准输出流打印信息，所以下面这个命令会像图1.1中产生一个GIF动画。

```shell
$ go build gopl.io/ch1/lissajous
$ ./lissajous >out.gif
```

## 1.5 获取URL

为了最简单地展示基于HTTP获取信息的方式，下面给出一个示例程序fetch，这个程序将获取对应的url，并将其源文本打印出来；这个例子的灵感来源于curl工具（译注：unix下的一个用来发http请求的工具，具体可以man curl）。当然，curl提供的功能更为复杂丰富，这里只编写最简单的样例。这个样例之后还会多次被用到。

```go
// Fetch prints the content found at a URL.
package main

import (
    "fmt"
    "io/ioutil"
    "net/http"
    "os"
)

func main() {
    for _, url := range os.Args[1:] {
        resp, err := http.Get(url)
        if err != nil {
            fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
            os.Exit(1)
        }
        b, err := ioutil.ReadAll(resp.Body)
        resp.Body.Close()
        if err != nil {
            fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
            os.Exit(1)
        }
        fmt.Printf("%s", b)
    }
}
```

这个程序从两个package中导入了函数，net/http和io/ioutil包，http.Get函数是创建HTTP请求的函数，如果获取过程没有出错，那么会在resp这个结构体中得到访问的请求结果。resp的Body字段包括一个可读的服务器响应流。ioutil.ReadAll函数从response中读取到全部内容；将其结果保存在变量b中。resp.Body.Close关闭resp的Body流，防止资源泄露，Printf函数会将结果b写出到标准输出流中。

```shell
$ go build gopl.io/ch1/fetch
$ ./fetch http://gopl.io
<html>
<head>
<title>The Go Programming Language</title>title>
...
```

HTTP请求如果失败了的话，会得到下面这样的结果：

```shell
$ ./fetch http://bad.gopl.io
fetch: Get http://bad.gopl.io: dial tcp: lookup bad.gopl.io: no such host
```

译注：在大天朝的网络环境下很容易重现这种错误，下面是Windows下运行得到的错误信息：

```shell
$ go run main.go http://gopl.io
fetch: Get http://gopl.io: dial tcp: lookup gopl.io: getaddrinfow: No such host is known.
```

无论哪种失败原因，我们的程序都用了os.Exit函数来终止进程，并且返回一个status错误码，其值为1。

## 1.6 并发获取多个URL

示例代码如下：

```go
// Fetchall fetches URLs in parallel and reports their times and sizes.
package main

import (
    "fmt"
    "io"
    "io/ioutil"
    "net/http"
    "os"
    "time"
)

func main() {
    start := time.Now()
    ch := make(chan string)
    for _, url := range os.Args[1:] {
        go fetch(url, ch) // start a goroutine
    }
    for range os.Args[1:] {
        fmt.Println(<-ch) // receive from channel ch
    }
    fmt.Printf("%.2fs elapsed\n", time.Since(start).Seconds())
}

func fetch(url string, ch chan<- string) {
    start := time.Now()
    resp, err := http.Get(url)
    if err != nil {
        ch <- fmt.Sprint(err) // send to channel ch
        return
    }
    nbytes, err := io.Copy(ioutil.Discard, resp.Body)
    resp.Body.Close() // don't leak resources
    if err != nil {
        ch <- fmt.Sprintf("while reading %s: %v", url, err)
        return
    }
    secs := time.Since(start).Seconds()
    ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)
}
```

下面使用fetchall来请求几个地址：

```shell
$ go build gopl.io/ch1/fetchall
$ ./fetchall https://golang.org http://gopl.io https://godoc.org
0.14s     6852  https://godoc.org
0.16s     7261  https://golang.org
0.48s     2475  http://gopl.io
0.48s elapsed
```

当一个goroutine尝试在一个channel上做send或者receive操作时，这个goroutine会**阻塞**在调用处，直到另一个goroutine从这个channel里接收或者写入值，这样两个goroutine才会继续执行channel操作之后的逻辑。在这个例子中，每一个fetch函数在执行时都会往channel里发送一个值（ch <- expression），主函数负责接收这些值（<-ch）。这个程序中我们用main函数来接收所有fetch函数传回的字符串，可以避免在goroutine异步执行还没有完成时main函数提前退出。

## 1.7 Web服务

Go语言的内置库使得写一个类似fetch的web服务器变得异常地简单。在本节中，我们会展示一个微型服务器，这个服务器的功能是返回当前用户正在访问的URL。比如用户访问的是 http://localhost:8000/hello ，那么响应是URL.Path = "hello"。

```go
// Server1 is a minimal "echo" server.
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", handler) // each request calls handler
    log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

// handler echoes the Path component of the request URL r.
func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```

在这个服务的基础上叠加特性是很容易的。一种比较实用的修改是为访问的url添加某种状态。比如，下面这个版本输出了同样的内容，但是会对请求的次数进行计算；对URL的请求结果会包含各种URL被访问的总次数，直接对/count这个URL的访问要除外。

```go
// Server2 is a minimal "echo" and counter server.
package main

import (
    "fmt"
    "log"
    "net/http"
    "sync"
)

var mu sync.Mutex
var count int

func main() {
    http.HandleFunc("/", handler)
    http.HandleFunc("/count", counter)
    log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

// handler echoes the Path component of the requested URL.
func handler(w http.ResponseWriter, r *http.Request) {
    mu.Lock()
    count++
    mu.Unlock()
    fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}

// counter echoes the number of calls so far.
func counter(w http.ResponseWriter, r *http.Request) {
    mu.Lock()
    fmt.Fprintf(w, "Count %d\n", count)
    mu.Unlock()
}
```

这个服务器有两个请求处理函数，根据请求的url不同会调用不同的函数：对/count这个url的请求会调用到counter这个函数，其它的url都会调用默认的处理函数。如果你的请求pattern是以/结尾，那么所有以该url为前缀的url都会被这条规则匹配。在这些代码的背后，服务器每一次接收请求处理时都会另起一个goroutine，这样服务器就可以同一时间处理多个请求。然而在并发情况下，假如真的有两个请求同一时刻去更新count，那么这个值可能并不会被正确地增加；这个程序可能会引发一个严重的bug：竞态条件（参见9.1）。为了避免这个问题，我们必须保证每次修改变量的最多只能有一个goroutine，这也就是**代码里的mu.Lock()和mu.Unlock()调用**将修改count的所有行为包在中间的目的。第九章中我们会进一步讲解共享变量。

下面是一个更为丰富的例子，handler函数会把请求的http头和请求的form数据都打印出来，这样可以使检查和调试这个服务更为方便：

```go
// handler echoes the HTTP request.
func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "%s %s %s\n", r.Method, r.URL, r.Proto)
    for k, v := range r.Header {
        fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
    }
    fmt.Fprintf(w, "Host = %q\n", r.Host)
    fmt.Fprintf(w, "RemoteAddr = %q\n", r.RemoteAddr)
    if err := r.ParseForm(); err != nil {
        log.Print(err)
    }
    for k, v := range r.Form {
        fmt.Fprintf(w, "Form[%q] = %q\n", k, v)
    }
}
```

## 1.8 本章要点

+ 控制流：

  Go语言里的switch还可以不带操作对象（译注：**switch不带操作对象时默认用true值代替，然后将每个case的表达式和true值进行比较**）；可以直接罗列多种条件，像其它语言里面的多个if else一样，下面是一个例子：

  ```go
  func Signum(x int) int {
      switch {
      case x > 0:
          return +1
      default:
          return 0
      case x < 0:
          return -1
      }
  }
  ```

+ 命名类型

  类型声明使得我们可以很方便地给一个特殊类型一个名字。因为struct类型声明通常非常地长，所以我们总要给这种struct取一个名字。本章中就有这样一个例子，二维点类型：

  ```go
  type Point struct {
      X, Y int
  }
  var p Point
  ```

  类型声明和命名类型会在第二章中介绍。

+ 指针

  指针是可见的内存地址，&操作符可以返回一个变量的内存地址，并且*操作符可以获取指针指向的变量内容，**但是在Go语言里没有指针运算，也就是不能像c语言里可以对指针进行加或减操作**。

+ 方法和接口

  方法是和命名类型关联的一类函数。Go语言里比较特殊的是方法可以被关联到任意一种命名类型。在第六章我们会详细地讲方法。接口是一种抽象类型，这种类型可以让我们以同样的方式来处理不同的固有类型，不用关心它们的具体实现，而只需要关注它们提供的方法。第七章中会详细说明这些内容。

+ **包（packages）**

  在你开始写一个新程序之前，最好先去检查一下是不是已经有了现成的库可以帮助你更高效地完成这件事情。你可以在 https://golang.org/pkg 和 [https://godoc.org](https://godoc.org/) 中找到标准库和社区写的package。godoc这个工具可以让你直接在本地命令行阅读标准库的文档。比如下面这个例子。

  ```go
  $ go doc http.ListenAndServe
  package http // import "net/http"
  func ListenAndServe(addr string, handler Handler) error
      ListenAndServe listens on the TCP network address addr and then
      calls Serve with handler to handle requests on incoming connections.
  ...
  ```

# 2. 程序结构

## 2.1 命名

Go语言中的函数名、变量名、常量名、类型名、语句标号和包名等所有的命名，都遵循一个简单的命名规则：一个名字必须以一个字母（Unicode字母）或下划线开头，后面可以跟任意数量的字母、数字或下划线。大写字母和小写字母是不同的：heapSort和Heapsort是两个不同的名字。

Go语言中类似if和switch的关键字有25个；关键字不能用于自定义名字，只能在特定语法结构中使用。

```
break      default       func     interface   select
case       defer         go       map         struct
chan       else          goto     package     switch
const      fallthrough   if       range       type
continue   for           import   return      var
```

此外，还有大约30多个预定义的名字，比如int和true等，主要对应内建的常量、类型和函数。

```
内建常量: true false iota nil

内建类型: int int8 int16 int32 int64
          uint uint8 uint16 uint32 uint64 uintptr
          float32 float64 complex128 complex64
          bool byte rune string error

内建函数: make len cap new append copy close delete
          complex real imag
          panic recover
```

+ 函数内部=>局部变量
+ 函数外部：
  + 开头小写：仅当前包可见
  + 开头大写：可导出的，当前包可见
+ 名字长度尽量短。如果作用域大、生命周期长，则可考虑用较长的名称
+ **驼峰命名**

## 2.2 声明

+ var 变量
+ const 常量
+ type 类型
+ func 函数实体对象

## 2.3 变量

var声明语句可以创建一个特定类型的变量，然后给变量附加一个名字，并且设置变量的初始值。变量声明的一般语法如下：

```go
var 变量名字 类型 = 表达式
```

其中“*类型*”或“*= 表达式*”两个部分可以省略其中的一个。

+ 如果省略的是类型信息，那么将根据初始化表达式来推导变量的类型信息。
+ 如果初始化表达式被省略，那么将用**零值**初始化该变量。 
  + 数值类型变量对应的零值是0
  + 布尔类型变量对应的零值是false
  + 字符串类型对应的零值是空字符串""
  + 接口或引用类型（包括slice、指针、map、chan和函数）变量对应的零值是nil。
  + 数组或结构体等聚合类型对应的零值是每个元素或字段都是对应该类型的零值。

**零值初始化机制可以确保每个声明的变量总是有一个良好定义的值，因此在Go语言中不存在未初始化的变量**。

### 2.3.1 简短变量声明

在函数内部，有一种称为简短变量声明语句的形式可用于声明和初始化局部变量。它以“名字 := 表达式”形式声明变量，**变量的类型根据表达式来自动推导**。

因为简洁和灵活的特点，简短变量声明被广泛用于大部分的局部变量的声明和初始化。

<u>var形式的声明语句往往是用于需要显式指定变量类型的地方，或者因为变量稍后会被重新赋值而初始值无关紧要的地方</u>。

```go
i := 100                  // an int
var boiling float64 = 100 // a float64
var names []string
var err error
var p Point
```

和var形式声明语句一样，简短变量声明语句也可以用来声明和初始化一组变量：

```Go
i, j := 0, 1
```

在下面的代码中，第一个语句声明了in和err两个变量。

在第二个语句只声明了out一个变量，然后<u>对已经声明的err进行了赋值操作</u>。

```go
in, err := os.Open(infile)
// ...
out, err := os.Create(outfile)
```

**简短变量声明语句中必须至少要声明一个新的变量**，下面的代码将不能编译通过：

```Go
f, err := os.Open(infile)
// ...
f, err := os.Create(outfile) // compile error: no new variables
```

**简短变量声明语句只有对已经在同级词法域声明过的变量才和赋值操作语句等价，如果变量是在外部词法域声明的，那么简短变量声明语句将会在当前词法域重新声明一个新的变量**。

### 2.3.2 指针

+ 一个指针的值是另一个变量的地址。

  如果用“var x int”声明语句声明一个x变量，那么&x表达式（取x变量的内存地址）将产生一个指向该整数变量的指针，指针对应的数据类型是`*int`，指针被称之为“指向int类型的指针”。

+ **任何类型的指针的零值都是nil**。

+ **在Go语言中，返回函数中局部变量的地址也是安全的**。

  例如下面的代码，调用f函数时创建局部变量v，在局部变量地址被返回之后依然有效，因为指针p依然引用这个变量。

  ```Go
  var p = f()
  
  func f() *int {
      v := 1
      return &v
  }
  ```

  **每次调用f函数都将返回不同的结果**：

  ```Go
  fmt.Println(f() == f()) // "false"
  ```

+ 可以在函数中通过该指针来更新变量的值。

  ```go
  func incr(p *int) int {
      *p++ // 非常重要：只是增加p指向的变量的值，并不改变p指针！！！
      return *p
  }
  
  v := 1
  incr(&v)              // side effect: v is now 2
  fmt.Println(incr(&v)) // "3" (and v is 3)
  ```

  *每次我们对一个变量取地址，或者复制指针，我们都是为原变量创建了新的别名。例如，`*p`就是变量v的别名。指针特别有价值的地方在于我们可以不用名字而访问一个变量，但是这是一把双刃剑：要找到一个变量的所有访问者并不容易，我们必须知道变量全部的别名（译注：这是Go语言的垃圾回收器所做的工作）。**不仅仅是指针会创建别名，很多其他引用类型也会创建别名，例如slice、map和chan，甚至结构体、数组和接口都会创建所引用变量的别名。***

+ 指针是实现标准库中flag包的关键技术，它使用命令行参数来设置对应变量的值，而这些对应命令行标志参数的变量可能会零散分布在整个程序中。为了说明这一点，在早些的echo版本中，就包含了两个可选的命令行参数：`-n`用于忽略行尾的换行符，`-s sep`用于指定分隔字符（默认是空格）。下面这是第四个版本，对应包路径为gopl.io/ch2/echo4。

  *gopl.io/ch2/echo4*

  ```Go
  // Echo4 prints its command-line arguments.
  package main
  
  import (
      "flag"
      "fmt"
      "strings"
  )
  
  var n = flag.Bool("n", false, "omit trailing newline")
  var sep = flag.String("s", " ", "separator")
  
  func main() {
      flag.Parse()
      fmt.Print(strings.Join(flag.Args(), *sep))
      if !*n {
          fmt.Println()
      }
  }
  ```

  调用flag.Bool函数会创建一个新的对应布尔型标志参数的变量。它有三个属性：第一个是命令行标志参数的名字“n”，然后是该标志参数的默认值（这里是false），最后是该标志参数对应的描述信息。如果用户在命令行输入了一个无效的标志参数，或者输入`-h`或`-help`参数，那么将打印所有标志参数的名字、默认值和描述信息。类似的，调用flag.String函数将创建一个对应字符串类型的标志参数变量，同样包含命令行标志参数对应的参数名、默认值、和描述信息。程序中的`sep`和`n`变量分别是指向对应命令行标志参数变量的指针，因此必须用`*sep`和`*n`形式的指针语法间接引用它们。

  当程序运行时，必须在使用标志参数对应的变量之前先调用flag.Parse函数，用于更新每个标志参数对应变量的值（之前是默认值）。对于非标志参数的普通命令行参数可以通过调用flag.Args()函数来访问，返回值对应一个字符串类型的slice。如果在flag.Parse函数解析命令行参数时遇到错误，默认将打印相关的提示信息，然后调用os.Exit(2)终止程序。

  让我们运行一些echo测试用例：

  ```shell
  $ go build gopl.io/ch2/echo4
  $ ./echo4 a bc def
  a bc def
  $ ./echo4 -s / a bc def
  a/bc/def
  $ ./echo4 -n a bc def
  a bc def$
  $ ./echo4 -help
  Usage of ./echo4:
    -n    omit trailing newline
    -s string
          separator (default " ")
  ```

### 2.3.3 new函数

另一个创建变量的方法是调用内建的new函数。

**表达式new(T)将创建一个T类型的匿名变量**，初始化为T类型的零值，然后返回变量地址，**返回的指针类型为`*T`**。

```go
p := new(int)   // p, *int 类型, 指向匿名的 int 变量
fmt.Println(*p) // "0"
*p = 2          // 设置 int 匿名变量的值为 2
fmt.Println(*p) // "2"
```

用new创建变量和普通变量声明语句方式创建变量没有什么区别，除了不需要声明一个临时变量的名字外，我们还可以在表达式中使用new(T)。

**换言之，new函数类似是一种语法糖，而不是一个新的基础概念**。

下面的两个newInt函数有着相同的行为：

```Go
func newInt() *int {
    return new(int)
}

func newInt() *int {
    var dummy int
    return &dummy
}
```

**每次调用new函数都是返回一个新的变量的地址**，因此下面两个地址是不同的：

```Go
p := new(int)
q := new(int)
fmt.Println(p == q) // "false"
```

*当然也可能有特殊情况：如果两个类型都是空的，也就是说类型的大小是0，例如`struct{}`和`[0]int`，有可能有相同的地址（依赖具体的语言实现）（译注：**请谨慎使用大小为0的类型，因为如果类型的大小为0的话，可能导致Go语言的自动垃圾回收器有不同的行为**，具体请查看`runtime.SetFinalizer`函数相关文档）*

**由于new只是一个预定义的函数，它并不是一个关键字，因此我们可以将new名字重新定义为别的类型。**

例如下面的例子：

```Go
func delta(old, new int) int { return new - old }
```

由于new被定义为int类型的变量名，因此在delta函数内部是无法使用内置的new函数的。

### 2.3.4 变量的生命周期

+ 对于在包一级声明的变量来说，它们的生命周期和整个程序的运行周期是一致的。
+ 局部变量的生命周期则是动态的：每次从创建一个新变量的声明语句开始，直到该变量不再被引用为止，然后变量的存储空间可能被回收。
  + 函数的参数变量和返回值变量都是局部变量。它们在函数每次被调用的时候创建。

```Go
for t := 0.0; t < cycles*2*math.Pi; t += res {
    x := math.Sin(t)
    y := math.Sin(t*freq + phase)
    img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5),
        blackIndex)
}
```

译注：函数的右小括弧也可以另起一行缩进，同时**为了防止编译器在行尾自动插入分号而导致的编译错误，可以在末尾的参数变量后面显式插入逗号**。像下面这样：

```Go
for t := 0.0; t < cycles*2*math.Pi; t += res {
    x := math.Sin(t)
    y := math.Sin(t*freq + phase)
    img.SetColorIndex(
        size+int(x*size+0.5), size+int(y*size+0.5),
        blackIndex, // 最后插入的逗号不会导致编译错误，这是Go编译器的一个特性
    )               // 小括弧另起一行缩进，和大括弧的风格保存一致
}
```

在<u>每次循环的开始会创建临时变量t，然后在每次循环迭代中创建临时变量x和y</u>。

+ **因为一个变量的有效周期只取决于是否可达，因此一个循环迭代内部的局部变量的生命周期可能超出其局部作用域。同时，局部变量可能在函数返回之后依然存在**。

+ **编译器会自动选择在栈上还是在堆上分配局部变量的存储空间**，但可能令人惊讶的是，这个选择并不是由用var还是new声明变量的方式决定的。

  ```go
  var global *int
  
  func f() {
      var x int
      x = 1
      global = &x
  }
  
  func g() {
      y := new(int)
      *y = 1
  }
  ```

  **f函数里的x变量必须在堆上分配**，因为它在函数退出后依然可以通过包一级的global变量找到，虽然它是在函数内部定义的；

  **用Go语言的术语说，这个x局部变量从函数f中逃逸了**。

  相反，当g函数返回时，变量`*y`将是不可达的，也就是说可以马上被回收的。因此，`*y`并没有从函数g中逃逸，编译器可以选择在栈上分配`*y`的存储空间（译注：也可以选择在堆上分配，然后由Go语言的GC回收这个变量的内存空间），虽然这里用的是new方式。其实在任何时候，你并不需为了编写正确的代码而要考虑变量的逃逸行为，要记住的是，逃逸的变量需要额外分配内存，同时对性能的优化可能会产生细微的影响。

+ Go语言的自动垃圾收集器对编写正确的代码是一个巨大的帮助，但也并不是说你完全不用考虑内存了。你虽然不需要显式地分配和释放内存，但是要编写高效的程序你依然需要了解变量的生命周期。例如，**如果将指向短生命周期对象的指针保存到具有长生命周期的对象中，特别是保存到全局变量时，会阻止对短生命周期对象的垃圾回收（从而可能影响程序的性能）**。

## 2.4 赋值

使用赋值语句可以更新一个变量的值，最简单的赋值语句是将要被赋值的变量放在=的左边，新值的表达式放在=的右边。

```Go
x = 1                       // 命名变量的赋值
*p = true                   // 通过指针间接赋值
person.name = "bob"         // 结构体字段赋值
count[x] = count[x] * scale // 数组、slice或map的元素赋值
```

数值变量也可以支持`++`递增和`--`递减语句

（译注：**自增和自减是语句，而不是表达式，因此`x = i++`之类的表达式是错误的**）：

```Go
v := 1
v++    // 等价方式 v = v + 1；v 变成 2
v--    // 等价方式 v = v - 1；v 变成 1
```

### 2.4.1 元组赋值

**元组赋值是另一种形式的赋值语句，它允许同时更新多个变量的值**。在赋值之前，赋值语句右边的所有表达式将会先进行求值，然后再统一更新左边对应变量的值。

这对于处理有些同时出现在元组赋值语句左右两边的变量很有帮助，例如我们可以这样交换两个变量的值：

```go
x, y = y, x

a[i], a[j] = a[j], a[i]
```

+ 但如果表达式太复杂的话，应该尽量避免过度使用元组赋值；因为每个变量单独赋值语句的写法可读性会更好。

+ **有些表达式会产生多个值，比如调用一个有多个返回值的函数**。

  当这样一个函数调用出现在元组赋值右边的表达式中时（译注：右边不能再有其它表达式），<u>左边变量的数目必须和右边一致</u>。

  ```go
  f, err = os.Open("foo.txt") // function call returns two values
  ```

  通常，这类函数会用额外的返回值来表达某种错误类型，例如os.Open是用额外的返回值返回一个error类型的错误，还有一些是用来返回布尔值，通常被称为ok。在稍后我们将看到的三个操作都是类似的用法。如果map查找（§4.3）、类型断言（§7.10）或通道接收（§8.4.2）出现在赋值语句的右边，它们都可能会产生两个结果，有一个额外的布尔结果表示操作是否成功：

  ```Go
  v, ok = m[key]             // map lookup
  v, ok = x.(T)              // type assertion
  v, ok = <-ch               // channel receive
  ```

  译注：map查找（§4.3）、类型断言（§7.10）或通道接收（§8.4.2）出现在赋值语句的右边时，并不一定是产生两个结果，也可能只产生一个结果。对于只产生一个结果的情形，map查找失败时会返回零值，类型断言失败时会发生运行时panic异常，通道接收失败时会返回零值（阻塞不算是失败）。例如下面的例子：

  ```Go
  v = m[key]                // map查找，失败时返回零值
  v = x.(T)                 // type断言，失败时panic异常
  v = <-ch                  // 管道接收，失败时返回零值（阻塞不算是失败）
  
  _, ok = m[key]            // map返回2个值
  _, ok = mm[""], false     // map返回1个值
  _ = mm[""]                // map返回1个值
  ```

+ 和变量声明一样，我们可以用下划线空白标识符`_`来丢弃不需要的值。

  ```go
  _, err = io.Copy(dst, src) // 丢弃字节数
  _, ok = x.(T)              // 只检测类型，忽略具体值
  ```

### 2.4.2 可赋值性

赋值语句是显式的赋值形式，但是程序中还有很多地方会发生隐式的赋值行为：函数调用会隐式地将调用参数的值赋值给函数的参数变量，一个返回语句会隐式地将返回操作的值赋值给结果变量，一个复合类型的字面量（§4.2）也会产生赋值行为。例如下面的语句：

```Go
medals := []string{"gold", "silver", "bronze"}
```

隐式地对slice的每个元素进行赋值操作，类似这样写的行为：

```Go
medals[0] = "gold"
medals[1] = "silver"
medals[2] = "bronze"
```

map和chan的元素，虽然不是普通的变量，但是也有类似的隐式赋值行为。

不管是隐式还是显式地赋值，**在赋值语句左边的变量和右边最终的求到的值必须有相同的数据类型**。更直白地说，只有右边的值对于左边的变量是可赋值的，赋值语句才是允许的。

可赋值性的规则对于不同类型有着不同要求，对每个新类型特殊的地方我们会专门解释。对于目前我们已经讨论过的类型，它的规则是简单的：**类型必须完全匹配，nil可以赋值给任何指针或引用类型的变量**。常量（§3.6）则有更灵活的赋值规则，因为这样可以避免不必要的显式的类型转换。

对于两个值是否可以用`==`或`!=`进行相等比较的能力也和可赋值能力有关系：**对于任何类型的值的相等比较，第二个值必须是对第一个值类型对应的变量是可赋值的，反之亦然**。和前面一样，我们会对每个新类型比较特殊的地方做专门的解释。

## 2.5 类型

**一个类型声明语句创建了一个新的类型名称，和现有类型具有相同的底层结构**。

新命名的类型提供了一个方法，用来分隔不同概念的类型，这样即**使它们底层类型相同也是不兼容的**。

```go
type 类型名字 底层类型
```

+ 类型声明语句一般出现在包一级，因此如果新创建的类型名字的首字符大写，则在包外部也可以使用。

+ 译注：对于中文汉字，Unicode标志都作为小写字母处理，因此中文的命名默认不能导出；不过国内的用户针对该问题提出了不同的看法，根据RobPike的回复，在Go2中有可能会将中日韩等字符当作大写字母处理。下面是RobPik在 [Issue763](https://github.com/golang/go/issues/5763) 的回复：

  > A solution that's been kicking around for a while:
  >
  > For Go 2 (can't do it before then): Change the definition to “lower case letters and *are package-local; all else is exported”. Then with non-cased languages, such as Japanese, we can write 日本语 for an exported name and* 日本语 for a local name. This rule has no effect, relative to the Go 1 rule, with cased languages. They behave exactly the same.

为了说明类型声明，我们将不同温度单位分别定义为不同的类型：

```go
// Package tempconv performs Celsius and Fahrenheit temperature computations.
package tempconv

import "fmt"

type Celsius float64    // 摄氏温度
type Fahrenheit float64 // 华氏温度

const (
    AbsoluteZeroC Celsius = -273.15 // 绝对零度
    FreezingC     Celsius = 0       // 结冰点温度
    BoilingC      Celsius = 100     // 沸水温度
)

func CToF(c Celsius) Fahrenheit { return Fahrenheit(c*9/5 + 32) }

func FToC(f Fahrenheit) Celsius { return Celsius((f - 32) * 5 / 9) }
```

我们在这个包声明了两种类型：Celsius和Fahrenheit分别对应不同的温度单位。**它们虽然有着相同的底层类型float64，但是它们是不同的数据类型，因此它们不可以被相互比较或混在一个表达式运算**。

+ 对于每一个类型T，都有一个对应的类型转换操作T(x)，用于将x转为T类型（译注：如果T是指针类型，可能会需要用小括弧包装T，比如`(*int)(0)`）。**只有当两个类型的底层基础类型相同时，才允许这种转型操作，或者是两者都是指向相同底层结构的指针类型，这些转换只改变类型而不会影响值本身**。如果x是可以赋值给T类型的值，那么x必然也可以被转为T类型，但是一般没有这个必要。

+ 数值类型之间的转型也是允许的。**在任何情况下，运行时不会发生转换失败的错误（译注: 错误只会发生在编译阶段）**。

底层数据类型决定了内部结构和表达方式，也决定是否可以像底层类型一样对内置运算符的支持。这意味着，Celsius和Fahrenheit类型的算术运算行为和底层的float64类型是一样的，正如我们所期望的那样。

```Go
fmt.Printf("%g\n", BoilingC-FreezingC) // "100" °C
boilingF := CToF(BoilingC)
fmt.Printf("%g\n", boilingF-CToF(FreezingC)) // "180" °F
fmt.Printf("%g\n", boilingF-FreezingC)       // compile error: type mismatch
```

比较运算符`==`和`<`也可以用来比较一个命名类型的变量和另一个有相同类型的变量，或有着相同底层类型的未命名类型的值之间做比较。但是**如果两个值有着不同的类型，则不能直接进行比较**：

```go
var c Celsius
var f Fahrenheit
fmt.Println(c == 0)          // "true"
fmt.Println(f >= 0)          // "true"
fmt.Println(c == f)          // compile error: type mismatch
fmt.Println(c == Celsius(f)) // "true"!
```

注意最后那个语句。尽管看起来像函数调用，但是Celsius(f)是类型转换操作，它并不会改变值，仅仅是改变值的类型而已。**测试为真的原因是因为c和g都是零值**。

+ 命名类型还可以为该类型的值定义新的行为。这些行为表示为一组关联到该类型的函数集合，我们称为**类型的方法集**。我们将在第六章中讨论方法的细节，这里只说些简单用法。

  下面的声明语句，Celsius类型的参数c出现在了函数名的前面，表示声明的是Celsius类型的一个名叫String的方法，该方法返回该类型对象c带着°C温度单位的字符串：

  ```Go
  func (c Celsius) String() string { return fmt.Sprintf("%g°C", c) }
  ```

+ **许多类型都会定义一个String方法，因为当使用fmt包的打印方法时，将会优先使用该类型对应的String方法返回的结果打印**，我们将在7.1节讲述。

  ```Go
  c := FToC(212.0)
  fmt.Println(c.String()) // "100°C"
  fmt.Printf("%v\n", c)   // "100°C"; no need to call String explicitly
  fmt.Printf("%s\n", c)   // "100°C"
  fmt.Println(c)          // "100°C"
  fmt.Printf("%g\n", c)   // "100"; does not call String
  fmt.Println(float64(c)) // "100"; does not call String
  ```

## 2.6 包和文件

Go语言中的包和其他语言的库或模块的概念类似，目的都是为了支持模块化、封装、单独编译和代码重用。一个包的源代码保存在一个或多个以.go为文件后缀名的源文件中，通常一个包所在目录路径的后缀是包的导入路径；例如包gopl.io/ch1/helloworld对应的目录路径是$GOPATH/src/gopl.io/ch1/helloworld。

**每个包都对应一个独立的名字空间**。例如，在image包中的Decode函数和在unicode/utf16包中的 Decode函数是不同的。要在外部引用该函数，必须显式使用image.Decode或utf16.Decode形式访问。

包还可以让我们通过控制哪些名字是外部可见的来隐藏内部实现信息。在Go语言中，一个简单的规则是：**如果一个名字是大写字母开头的，那么该名字是导出的**（译注：因为汉字不区分大小写，因此汉字开头的名字是没有导出的）。

*在每个源文件的包声明前紧跟着的注释是包注释（§10.7.4）。通常，包注释的第一句应该先是包的功能概要说明。一个包通常只有一个源文件有包注释（译注：如果有多个包注释，目前的文档工具会根据源文件名的先后顺序将它们链接为一个包注释）。如果包注释很大，通常会放到一个独立的doc.go文件中。*

### 2.6.1 导入包

**在Go语言程序中，每个包都有一个全局唯一的导入路径**。

除了包的导入路径，每个包还有一个包名，包名一般是短小的名字（并不要求包名是唯一的），包名在包的声明处指定。**按照惯例，一个包的名字和包的导入路径的最后一个字段相同**，例如gopl.io/ch2/tempconv包的名字一般是tempconv。

要使用gopl.io/ch2/tempconv包，需要先导入：

```Go
// Cf converts its numeric argument to Celsius and Fahrenheit.
package main

import (
    "fmt"
    "os"
    "strconv"

    "gopl.io/ch2/tempconv"
)

func main() {
    for _, arg := range os.Args[1:] {
        t, err := strconv.ParseFloat(arg, 64)
        if err != nil {
            fmt.Fprintf(os.Stderr, "cf: %v\n", err)
            os.Exit(1)
        }
        f := tempconv.Fahrenheit(t)
        c := tempconv.Celsius(t)
        fmt.Printf("%s = %s, %s = %s\n",
            f, tempconv.FToC(f), c, tempconv.CToF(c))
    }
}
```

*导入语句将导入的包绑定到一个短小的名字，然后通过该短小的名字就可以引用包中导出的全部内容。上面的导入声明将允许我们以tempconv.CToF的形式来访问gopl.io/ch2/tempconv包中的内容。在默认情况下，导入的包绑定到tempconv名字（译注：指包声明语句指定的名字），但是我们也可以绑定到另一个名称，以避免名字冲突（§10.4）。*

+ **如果导入了一个包，但是又没有使用该包将被当作一个编译错误处理**。

  这种强制规则可以有效减少不必要的依赖，虽然在调试期间可能会让人讨厌，因为删除一个类似log.Print("got here!")的打印语句可能导致需要同时删除log包导入声明，否则，编译器将会发出一个错误。在这种情况下，我们需要将不必要的导入删除或注释掉。

+ 不过有更好的解决方案，**我们可以使用golang.org/x/tools/cmd/goimports导入工具，它可以根据需要自动添加或删除导入的包**；许多编辑器都可以集成goimports工具，然后在保存文件的时候自动运行。**类似的还有gofmt工具，可以用来格式化Go源文件**。

### 2.6.2 包的初始化

**包的初始化首先是解决包级变量的依赖顺序，然后按照包级变量声明出现的顺序依次初始化**：

```Go
var a = b + c // a 第三个初始化, 为 3
var b = f()   // b 第二个初始化, 为 2, 通过调用 f (依赖c)
var c = 1     // c 第一个初始化, 为 1

func f() int { return c + 1 }
```

+ **如果包中含有多个.go源文件，它们将按照发给编译器的顺序进行初始化，Go语言的构建工具首先会将.go文件根据文件名排序，然后依次调用编译器编译。**

+ 对于在包级别声明的变量，如果有初始化表达式则用表达式初始化，还有一些没有初始化表达式的，例如某些表格数据初始化并不是一个简单的赋值过程。在这种情况下，我们可以用一个特殊的init初始化函数来简化初始化工作。**每个文件都可以包含多个init初始化函数**

  ```Go
  func init() { /* ... */ }
  ```

  **这样的init初始化函数除了不能被调用或引用外，其他行为和普通函数类似。在每个文件中的init初始化函数，在程序开始执行时按照它们声明的顺序被自动调用**。

  **每个包在解决依赖的前提下，以导入声明的顺序初始化，每个包只会被初始化一次。**因此，如果一个p包导入了q包，那么在p包初始化的时候可以认为q包必然已经初始化过了。**初始化工作是自下而上进行的，main包最后被初始化**。以这种方式，可以确保在main函数执行之前，所有依赖的包都已经完成初始化工作了。

## 2.7 作用域

一个声明语句将程序中的实体和一个名字关联，比如一个函数或一个变量。声明语句的作用域是指源代码中可以有效使用这个名字的范围。

不要将作用域和生命周期混为一谈。

+ 声明语句的作用域对应的是一个源代码的文本区域；它是一个**编译时**的属性。
+ 一个变量的生命周期是指程序运行时变量存在的有效时间段，在此时间区域内它可以被程序的其他部分引用；是一个**运行时**的概念。

句法块是由花括弧所包含的一系列语句，就像函数体或循环体花括弧包裹的内容一样。句法块内部声明的名字是无法被外部块访问的。这个块决定了内部声明的名字的作用域范围。我们可以把块（block）的概念推广到包括其他声明的群组，这些声明在代码中并未显式地使用花括号包裹起来，我们称之为词法块。**对全局的源代码来说，存在一个整体的词法块，称为全局词法块**；对于每个包；每个for、if和switch语句，也都有对应词法块；每个switch或select的分支也有独立的词法块；当然也包括显式书写的词法块（花括弧包含的语句）。

声明语句对应的词法域决定了作用域范围的大小。对于内置的类型、函数和常量，比如int、len和true等是在全局作用域的，因此可以在整个程序中直接使用。**任何在函数外部（也就是包级语法域）声明的名字可以在同一个包的任何源文件中访问的**。对于导入的包，例如tempconv导入的fmt包，则是对应源文件级的作用域，因此只能在当前的文件中访问导入的fmt包，**当前包的其它源文件无法访问在当前源文件导入的包**。还有许多声明语句，比如tempconv.CToF函数中的变量c，则是局部作用域的，它只能在函数内部（甚至只能是局部的某些部分）访问。

**控制流标号，就是break、continue或goto语句后面跟着的那种标号，则是函数级的作用域**。

**一个程序可能包含多个同名的声明，只要它们在不同的词法域就没有关系**。例如，你可以声明一个局部变量，和包级的变量同名。或者是像2.3.3节的例子那样，你可以将一个函数参数的名字声明为new，虽然内置的new是全局作用域的。但是物极必反，如果滥用不同词法域可重名的特性的话，可能导致程序很难阅读。

下面的例子同样有三个不同的x变量，每个声明在不同的词法域，一个在函数体词法域，一个在for隐式的初始化词法域，一个在for循环体词法域；只有两个块是显式创建的：

```Go
func main() {
    x := "hello"
    for _, x := range x {
        x := x + 'A' - 'a'
        fmt.Printf("%c", x) // "HELLO" (one letter per iteration)
    }
}
```

和for循环类似，if和switch语句也会在条件部分创建隐式词法域，还有它们对应的执行体词法域。下面的if-else测试链演示了x和y的有效作用域范围：

```Go
if x := f(); x == 0 {
    fmt.Println(x)
} else if y := g(x); x == y {
    fmt.Println(x, y)
} else {
    fmt.Println(x, y)
}
fmt.Println(x, y) // compile error: x and y are not visible here
```

**第二个if语句嵌套在第一个内部，因此第一个if语句条件初始化词法域声明的变量在第二个if中也可以访问**。switch语句的每个分支也有类似的词法域规则：**条件部分为一个隐式词法域，然后是每个分支的词法域。**

在包级别，声明的顺序并不会影响作用域范围，因此一个先声明的可以引用它自身或者是引用后面的一个声明，这可以让我们定义一些相互嵌套或递归的类型或函数。但是如果一个变量或常量递归引用了自身，则会产生编译错误。

在这个程序中：

```Go
if f, err := os.Open(fname); err != nil { // compile error: unused: f
    return err
}
f.ReadByte() // compile error: undefined f
f.Close()    // compile error: undefined f
```

变量f的作用域只在if语句内，因此后面的语句将无法引入它，这将导致编译错误。你可能会收到一个局部变量f没有声明的错误提示，具体错误信息依赖编译器的实现。

通常需要在if之前声明变量，这样可以确保后面的语句依然可以访问变量：

```Go
f, err := os.Open(fname)
if err != nil {
    return err
}
f.ReadByte()
f.Close()
```

你可能会考虑通过将ReadByte和Close移动到if的else块来解决这个问题：

```Go
if f, err := os.Open(fname); err != nil {
    return err
} else {
    // f and err are visible here too
    f.ReadByte()
    f.Close()
}
```

但这不是Go语言推荐的做法，**Go语言的习惯是在if中处理错误然后直接返回，这样可以确保正常执行的语句不需要代码缩进。**

要特别注意短变量声明语句的作用域范围，考虑下面的程序，它的目的是获取当前的工作目录然后保存到一个包级的变量中。这本来可以通过直接调用os.Getwd完成，但是将这个从主逻辑中分离出来可能会更好，特别是在需要处理错误的时候。函数log.Fatalf用于打印日志信息，然后调用os.Exit(1)终止程序。

```Go
var cwd string

func init() {
    cwd, err := os.Getwd() // compile error: unused: cwd
    if err != nil {
        log.Fatalf("os.Getwd failed: %v", err)
    }
}
```

**虽然cwd在外部已经声明过，但是`:=`语句还是将cwd和err重新声明为新的局部变量。因为内部声明的cwd将屏蔽外部的声明，因此上面的代码并不会正确更新包级声明的cwd变量**。

由于当前的编译器会检测到局部声明的cwd并没有使用，然后报告这可能是一个错误，但是这种检测并不可靠。因为一些小的代码变更，例如增加一个局部cwd的打印语句，就可能导致这种检测失效。

```Go
var cwd string

func init() {
    cwd, err := os.Getwd() // NOTE: wrong!
    if err != nil {
        log.Fatalf("os.Getwd failed: %v", err)
    }
    log.Printf("Working directory = %s", cwd)
}
```

全局的cwd变量依然是没有被正确初始化的，而且看似正常的日志输出更是让这个BUG更加隐晦。

有许多方式可以避免出现类似潜在的问题。最直接的方法是通过单独声明err变量，来避免使用`:=`的简短声明方式：

```Go
var cwd string

func init() {
    var err error
    cwd, err = os.Getwd()
    if err != nil {
        log.Fatalf("os.Getwd failed: %v", err)
    }
}
```

我们已经看到包、文件、声明和语句如何来表达一个程序结构。在下面的两个章节，我们将探讨数据的结构。

# 3. 基础数据类型

