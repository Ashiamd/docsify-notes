# gRPC-官方文档学习

> [Documentation | gRPC](https://www.grpc.io/docs/)	=>	主要就是根据官方文档简单学习一下gRPC，中间会对一些概念另外查文章学习。
>
> <small>之前写过一个简单的移动端多人聊天室，服务端主要用到的就是netty+protobuf（客户端dart+protobuf）。但是当时主要就用到message（就是用来UDP即时通讯时序列化数据），没用到里面的service定义服务接口。</small>

# 1. 文档

## 1.1 gRPC介绍

### 1.1.1 gRPC概述

​	gRPC can use protocol buffers as both its Interface Definition Language (**IDL**) and as its underlying message interchange format.

> If you’re new to gRPC and/or protocol buffers, read this! If you just want to dive in and see gRPC in action first, [select a language](https://www.grpc.io/docs/languages/) and try its **Quick start**.

---

​	使用gRPC，编写代码时能够像调用本地对象一样去调用不同机器上的远程方法，使得部署分布式应用和服务更加简单。和很多RPC系统一样，gRPC基于定义服务的思想，制定具有参数和返回值的远程调用方法。

+ 在服务端，其实现gRPC定义的接口并运行gRPC服务以处理客户端调用请求。
+ 在客户端，留有对应的存根stub，提供和gRPC服务端相同的方法。

![Concept Diagram](https://www.grpc.io/img/landing-2.svg)

​	gRPC客户端和服务端可以处在不同的环境。例如gRPC服务端使用java提供服务，而gRPC客户端采用go语言请求调用gRPC服务。

> [Documentation | gRPC](https://www.grpc.io/docs/)	=>	下方提供具体支持的操作系统、语言版本
>
> These are the officially supported gRPC language, platform and OS versions:
>
> | Language    | OS                     | Compilers / SDK                             |
> | ----------- | ---------------------- | ------------------------------------------- |
> | C/C++       | Linux, Mac             | GCC 4.9+, Clang 3.4+                        |
> | C/C++       | Windows 7+             | Visual Studio 2015+                         |
> | C#          | Linux, Mac             | .NET Core, Mono 4+                          |
> | C#          | Windows 7+             | .NET Core, NET 4.5+                         |
> | Dart        | Windows, Linux, Mac    | Dart 2.12+                                  |
> | Go          | Windows, Linux, Mac    | Go 1.13+                                    |
> | Java        | Windows, Linux, Mac    | JDK 8 recommended (Jelly Bean+ for Android) |
> | Kotlin      | Windows, Linux, Mac    | Kotlin 1.3+                                 |
> | Node.js     | Windows, Linux, Mac    | Node v8+                                    |
> | Objective-C | macOS 10.10+, iOS 9.0+ | Xcode 7.2+                                  |
> | PHP         | Linux, Mac             | PHP 7.0+                                    |
> | Python      | Windows, Linux, Mac    | Python 3.5+                                 |
> | Ruby        | Windows, Linux, Mac    | Ruby 2.3+                                   |

### 1.1.2 Working with Protocol Buffers

> - [Language Guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3)

​	By default, gRPC uses [Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview), Google’s mature open source mechanism for **serializing structured data** (although it can be used with other data formats such as JSON). Here’s a quick intro to how it works. If you’re already familiar with protocol buffers, feel free to skip ahead to the next section.

​	第一步：编写`.proto`二进制文件，自定义message需要的字段。

​	<small>The first step when working with protocol buffers is to define the structure for the data you want to serialize in a *proto file*: this is an ordinary text file with a `.proto` extension. Protocol buffer data is structured as *messages*, where each message is a small logical record of information containing a series of name-value pairs called *fields*. Here’s a simple example:</small>

```protobuf
message Person {
  string name = 1;
  int32 id = 2;
  bool has_ponycopter = 3;
}
```

​	第二步：通过`protoc`指令根据`.proto`文件生成对应message的数据访问类（生成的类，自带基础的setter等方法）。

​	<small>Then, once you’ve specified your data structures, you use the protocol buffer compiler `protoc` to generate data access classes in your preferred language(s) from your proto definition. These provide simple accessors for each field, like `name()` and `set_name()`, as well as methods to serialize/parse the whole structure to/from raw bytes. So, for instance, if your chosen language is C++, running the compiler on the example above will generate a class called `Person`. You can then use this class in your application to populate, serialize, and retrieve `Person` protocol buffer messages.</small>

​	gRPC服务同样也是在`.proto`文件中定义，下面定义的gRPC服务中含有一个RPC调用的方法，其参数和返回值都是message。

​	<small>You define gRPC services in ordinary proto files, with RPC method parameters and return types specified as protocol buffer messages:</small>

```protobuf
// The greeter service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

​		通过gRPC插件`protoc`生成的代码包含客户端和服务端，代码内容包括数据访问、赋值、序列化等。

​	<small>gRPC uses `protoc` with a special gRPC plugin to generate code from your proto file: you get generated gRPC client and server code, as well as the regular protocol buffer code for populating, serializing, and retrieving your message types. You’ll see an example of this below.</small>

> ​	To learn more about protocol buffers, including how to install `protoc` with the gRPC plugin in your chosen language, see the [protocol buffers documentation](https://developers.google.com/protocol-buffers/docs/overview).

### 1.1.3 Protocol buffer versions

​	简言之就是现在有proto3和proto2，官方推荐用比较新的proto3。

> While [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview) have been available to open source users for some time, most examples from this site use protocol buffers version 3 (proto3), which has a slightly simplified syntax, some useful new features, and supports more languages. Proto3 is currently available in Java, C++, Dart, Python, Objective-C, C#, a lite-runtime (Android Java), Ruby, and JavaScript from the [protocol buffers GitHub repo](https://github.com/google/protobuf/releases), as well as a Go language generator from the [golang/protobuf official package](https://pkg.go.dev/google.golang.org/protobuf), with more languages in development. You can find out more in the [proto3 language guide](https://developers.google.com/protocol-buffers/docs/proto3) and the [reference documentation](https://developers.google.com/protocol-buffers/docs/reference/overview) available for each language. The reference documentation also includes a [formal specification](https://developers.google.com/protocol-buffers/docs/reference/proto3-spec) for the `.proto` file format.
>
> In general, while you can use proto2 (the current default protocol buffers version), we recommend that you use proto3 with gRPC as it lets you use the full range of gRPC-supported languages, as well as avoiding compatibility issues with proto2 clients talking to proto3 servers and vice versa.

## 1.2 核心概念、体系架构和生命周期

> Core concepts, architecture and lifecycle
>
> An introduction to key gRPC concepts, with an overview of gRPC architecture and RPC life cycle.
>
> Not familiar with gRPC? First read [Introduction to gRPC](https://www.grpc.io/docs/what-is-grpc/introduction/). For language-specific details, see the quick start, tutorial, and reference documentation for your language of choice.

### 1.2.1 Service definition

​	下面这段前面重复过了，就是"1.1.1 gRPC"概述的内容

​	Like many RPC systems, gRPC is based around the idea of defining a service, specifying the methods that can be called remotely with their parameters and return types. By default, gRPC uses [protocol buffers](https://developers.google.com/protocol-buffers) as the Interface Definition Language (IDL) for describing both the service interface and the structure of the payload messages. It is possible to use other alternatives if desired.

```proto
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

gRPC支持四种service方法：

- Unary RPCs：像普通函数调用一样，client向server发起一个request请求，然后得到一个respone应答。

  ```proto
  rpc SayHello(HelloRequest) returns (HelloResponse);
  ```

- Server streaming RPCs：client向server发起请求并获得一个stream流以读取连续多个messages。client读取stream流数据直到stream流不再返回messages。 gRPC保证在单次RPC调用中stream返回的message有序。

  ```proto
  rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse);
  ```

- Client streaming RPCs：client每次通过一个新的stream写入一系列messages，然后发送到server。client写完数据后，等待server读取数据和返回response。同样，gRPC保证单次RPC调用中stream的message有序。 

  ```proto
  rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse);
  ```

- Bidirectional streaming RPCs：client和server彼此都通过读写流read-write stream向对方发送一系列message数据。这两个（入参、返回值）stream在操作上彼此隔离、独立，client和server可以随机读写它们（stream）：比如，server可以等到收到来自client的所有message后再写response，也可交替读一个message然后回复一个message（即response），或者其他读写的组合。每个stream中的messages的顺序都是preserved（个人理解就是message在stream中按照写入次序排列）。

  ```proto
  rpc BidiHello(stream HelloRequest) returns (stream HelloResponse);
  ```

>  You’ll learn more about the different types of RPC in the [RPC life cycle](https://www.grpc.io/docs/what-is-grpc/core-concepts/#rpc-life-cycle) section below.

### 1.2.2 Using the API

​	通过gRPC提供的protoc插件编译`.proto`文件，生成client端和server端的代码。通常client端调用API，server端实现API。

+ server端：实现`.proto`中声明service API，运行gRPC服务处理client请求。gRPC基础架构负责解码接收到的requests请求（参数），执行service实现方法，编码service的reponse（传回client）。

+ client端：client有一个作为stub存根的本地object对象（对于某些语言，首选术语是*client*），该stub存根object对象也实现同一个service API方法。client可以恰当类型的message包装参数，仅调用本地stub的方法，接下去由gRPC负责将这（些）requests发送到server，并且之后也是gRPC将server的protocol buffer responses带回client。

  *ps：简言之，client调用gRPC按照proto生成的本地对象，而gRPC负责实际的client和server之间的message数据交互。（从调用者的角度来看，网络交互细节被省略了，只需要与本地对象交互）*

### 1.2.3 Synchronous vs. asynchronous

​	Synchronous RPC 调用在服务器收到响应之前一直阻塞，这与RPC所期望的过程调用的抽象最接近。另一方面，网络本身就是异步的，在大多数情况下，无需阻塞线程就可启动（和使用）RPC服务，这很有用（很便利）。

​	大多数gRPC编程API都支持同步和异步两种方式。（详情参考所选语言的文档）

### 1.2.4 RPC life cycle

​	该章节可以了解到client通过gRPC调用server方法的一些过程，但详情还是要参照选定语言的参考文档。

*ps：下面直接看全英版本，就不继续使用我拙劣的译文了，全英比较不容易有理解上的歧义。*

#### Unary RPC

​	First consider the simplest type of RPC where the client sends a single request and gets back a single response.

1. Once the client calls a stub method, the server is notified that the RPC has been invoked with the client’s [**metadata**](https://www.grpc.io/docs/what-is-grpc/core-concepts/#metadata) for this call, the **method name**, and the specified [**deadline**](https://www.grpc.io/docs/what-is-grpc/core-concepts/#deadlines) if applicable.

   *client调用stub方法，server随后收到RPC调用的通知，可读取到本次client调用时携带的metadata（方法名、响应等待截止时间等）*

2. The server can then either send back its own initial metadata (which must be sent before any response) straight away, or wait for the client’s request message. Which happens first, is application-specific.

   *server可以直接先发送自己的matadata（必须在响应任意一个response之前）到client，也可选择先等待客户端的request请求。哪个先执行，需要程序员事先指定。*

3. Once the server has the client’s request message, it does whatever work is necessary to create and populate a response. The response is then returned (if successful) to the client together with status details (status code and optional status message) and optional trailing metadata.

   *一旦serevr接收到来自client的request message，就会执行创建和填充response的相关流程。如果处理成功，reponse会携带一些状态信息（状态码、可选的状态message）和可选trailing metadata返回给client。*

4. If the response status is OK, then the client gets the response, which completes the call on the client side.

   

#### Server streaming RPC

#### Client streaming RPC

#### Bidirectional streaming RPC

#### Deadlines/Timeouts

#### RPC termination

#### Cancelling an RPC

> Warning
>
> Changes made before a cancellation are not rolled back.

#### Metadata

#### Channels