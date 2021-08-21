# gRPC-官方文档学习

> [Documentation | gRPC](https://www.grpc.io/docs/)	=>	主要就是根据官方文档简单学习一下gRPC，中间会对一些概念另外查文章学习。
>
> <small>之前写过一个简单的移动端多人聊天室，服务端主要用到的就是netty+protobuf（客户端dart+protobuf）。但是当时主要就用到protobuf的message（就是用来UDP即时通讯时序列化数据），没用到gRPC定义service服务接口。</small>

# 1. gRPC介绍

## 1.1 gRPC概述

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

## 1.2 Working with Protocol Buffers

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

## 1.3 Protocol buffer versions

​	简言之就是现在有proto3和proto2，官方推荐用比较新的proto3。

> While [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview) have been available to open source users for some time, most examples from this site use protocol buffers version 3 (proto3), which has a slightly simplified syntax, some useful new features, and supports more languages. Proto3 is currently available in Java, C++, Dart, Python, Objective-C, C#, a lite-runtime (Android Java), Ruby, and JavaScript from the [protocol buffers GitHub repo](https://github.com/google/protobuf/releases), as well as a Go language generator from the [golang/protobuf official package](https://pkg.go.dev/google.golang.org/protobuf), with more languages in development. You can find out more in the [proto3 language guide](https://developers.google.com/protocol-buffers/docs/proto3) and the [reference documentation](https://developers.google.com/protocol-buffers/docs/reference/overview) available for each language. The reference documentation also includes a [formal specification](https://developers.google.com/protocol-buffers/docs/reference/proto3-spec) for the `.proto` file format.
>
> In general, while you can use proto2 (the current default protocol buffers version), we recommend that you use proto3 with gRPC as it lets you use the full range of gRPC-supported languages, as well as avoiding compatibility issues with proto2 clients talking to proto3 servers and vice versa.

# 2. 核心概念、体系架构和生命周期

> Core concepts, architecture and lifecycle
>
> An introduction to key gRPC concepts, with an overview of gRPC architecture and RPC life cycle.
>
> Not familiar with gRPC? First read [Introduction to gRPC](https://www.grpc.io/docs/what-is-grpc/introduction/). For language-specific details, see the quick start, tutorial, and reference documentation for your language of choice.

## 2.1 Service definition

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

## 2.2 Using the API

​	通过gRPC提供的protoc插件编译`.proto`文件，生成client端和server端的代码。通常client端调用API，server端实现API。

+ server端：实现`.proto`中声明service API，运行gRPC服务处理client请求。gRPC基础架构负责解码接收到的requests请求（参数），执行service实现方法，编码service的reponse（传回client）。

+ client端：client有一个作为stub存根的本地object对象（对于某些语言，首选术语是*client*），该stub存根object对象也实现同一个service API方法。client可以恰当类型的message包装参数，仅调用本地stub的方法，接下去由gRPC负责将这（些）requests发送到server，并且之后也是gRPC将server的protocol buffer responses带回client。

  *ps：简言之，client调用gRPC按照proto生成的本地对象，而gRPC负责实际的client和server之间的message数据交互。（从调用者的角度来看，网络交互细节被省略了，只需要与本地对象交互）*

## 2.3 Synchronous vs. asynchronous

​	Synchronous RPC 调用在服务器收到响应之前一直阻塞，这与RPC所期望的过程调用的抽象最接近。另一方面，网络本身就是异步的，在大多数情况下，无需阻塞线程就可启动（和使用）RPC服务，这很有用（很便利）。

​	大多数gRPC编程API都支持同步和异步两种方式。（详情参考所选语言的文档）

## 2.4 RPC life cycle

​	该章节可以了解到client通过gRPC调用server方法的一些过程，但详情还是要参照选定语言的参考文档。

*ps：下面直接看全英版本，就不继续使用我拙劣的译文了，全英比较不容易有理解上的歧义。*

### Unary RPC

​	First consider the simplest type of RPC where the client sends a single request and gets back a single response.

1. Once the client calls a stub method, the server is notified that the RPC has been invoked with the client’s [**metadata**](https://www.grpc.io/docs/what-is-grpc/core-concepts/#metadata) for this call, the **method name**, and the specified [**deadline**](https://www.grpc.io/docs/what-is-grpc/core-concepts/#deadlines) if applicable.

   *client调用stub方法，server随后收到RPC调用的通知，可读取到本次client调用时携带的metadata（方法名、响应等待截止时间等）*

2. The server can then either send back its own initial metadata (which must be sent before any response) straight away, or wait for the client’s request message. Which happens first, is application-specific.

   *server可以直接先发送自己的matadata（必须在响应任意一个response之前）到client，也可选择先等待客户端的request请求。哪个先执行，需要程序员事先指定。*

3. Once the server has the client’s request message, it does whatever work is necessary to create and populate a response. The response is then returned (if successful) to the client together with status details (status code and optional status message) and optional trailing metadata.

   *一旦serevr接收到来自client的request message，就会执行创建和填充response的相关流程。如果处理成功，reponse会携带一些status details状态信息（状态码、可选的状态message）和可选trailing metadata返回给client。*

4. If the response status is OK, then the client gets the response, which completes the call on the client side.

   *如果response的status正常，那么client获取response，客户端调用流程终止。*

### Server streaming RPC

​	A server-streaming RPC is similar to a unary RPC, except that the server returns a stream of messages in response to a client’s request. After sending all its messages, the server’s status details (status code and optional status message) and optional trailing metadata are sent to the client. This completes processing on the server side. The client completes once it has all the server’s messages.

​	*server-stream RPC和前面的unary RPC类似，只不过server返回值response变为stream（里面是一系列message）。在server（借助stream）发送完所有message后，再发送server的status details状态信息（状态码、可选的状态message）和可选trailing metadata到client。server服务端流程至此结束，而client在接收所有来自server的messages之后结束流程。*

### Client streaming RPC

​	A client-streaming RPC is similar to a unary RPC, except that the client sends a stream of messages to the server instead of a single message. The server responds with a single message (along with its status details and optional trailing metadata), typically but not necessarily after it has received all the client’s messages.

​	*client-stream RPC同理和RPC类似，不过就是client发送到server的request变为stream（里面是一系列message）。server响应一个message（携带一些status details状态信息和可选trailing metadata），通常在接收所有来自client的messages后响应（注意：并不是硬性要求在接收完所有数据后才response）*

### Bidirectional streaming RPC

​	In a bidirectional streaming RPC, the call is initiated by the client invoking the method and the server receiving the client metadata, method name, and deadline. The server can choose to send back its initial metadata or wait for the client to start streaming messages.

​	*在 bidirectional streaming RPC中，client调用方法、server接收client的metadata、method、deadline时启动RPC调用流程。server可以选择发送其metadata或者等待client通过stream传输messages过来。*

​	Client- and server-side stream processing is application specific. Since the two streams are independent, the client and server can read and write messages in any order. For example, a server can wait until it has received all of a client’s messages before writing its messages, or the server and client can play “ping-pong” – the server gets a request, then sends back a response, then the client sends another request based on the response, and so on.

​	*client和server端的stream处理流程是应用程序实现中指定的。由于两个stream彼此独立，client和server可随机读写messages。比如，server可以等到接收完所有来自client的messages再进行写messages操作，或者server和client之间进行"ping-pong"操作（即server每接收一个request就回送一个response，之后client又根据新收到的response发送新的request，如此往复。）或者进行其他形式的request/response交互操作。*

### * Deadlines/Timeouts

​	gRPC allows clients to specify how long they are willing to wait for an RPC to complete before the RPC is terminated with a `DEADLINE_EXCEEDED` error. On the server side, the server can query to see if a particular RPC has timed out, or how much time is left to complete the RPC.

​	*gRPC允许client指定RPC调用完成的最长执行时间。server端可以查询特定的RPC是否处理超时，或者还剩多少时间可以用于处理该RPC。*

​	Specifying a deadline or timeout is language specific: some language APIs work in terms of timeouts (durations of time), and some language APIs work in terms of a deadline (a fixed point in time) and may or may not have a default deadline.

​	*指定deadline或者timeout和具体使用的编程语言有关：有些编程语言的API根据timeouts（duration of time）运作RPC，有些则根据deadline（a fixed point in time），还有一些有或者没有默认的deadline。*

### * RPC termination

​	**In gRPC, both the client and server make independent and local determinations of the success of the call, and their conclusions may not match. This means that, for example, you could have an RPC that finishes successfully on the server side (“I have sent all my responses!") but fails on the client side (“The responses arrived after my deadline!"). It’s also possible for a server to decide to complete before a client has sent all its requests.**

​	***在gRPC中，client和server对"RPC调用成功/结束"只在本地进行独立的判定，它们判定的结论可能不匹配。这意味着，你在server端可能完成了RPC调用，但是在client端出现失败（如client判定来自server端response超出预期的deadline）。还有可能server在client发送完所有requets之前就单方宣告server端已经完成RPC处理。***

### * Cancelling an RPC

​	Either the client or the server can cancel an RPC at any time. A cancellation terminates the RPC immediately so that no further work is done.

​	*在client和server端都可以随时取消RPC调用。一次cancellation取消操作能立即终止RPC，使后续流程不再进行。*

> Warning
>
> Changes made before a cancellation are not rolled back.
>
> 注意：
>
> **在cancellation取消操作之前的所有改变都不会被回滚！！！**
>
> **在cancellation取消操作之前的所有改变都不会被回滚！！！**
>
> **在cancellation取消操作之前的所有改变都不会被回滚！！！**

### * Metadata

​	Metadata is information about a particular RPC call (such as [authentication details](https://www.grpc.io/docs/guides/auth/)) in the form of a list of key-value pairs, where the keys are strings and the values are typically strings, but can be binary data. Metadata is opaque to gRPC itself - it lets the client provide information associated with the call to the server and vice versa.

​	Access to metadata is language dependent.

​	*Metadata是关于特定某个RPC调用的信息（比如身份认证信息），其以key-value对的形式存在，key为字符串，value通常是字符串，但也可以是二进制数据。Metadata对gRPC本身来说是隐藏无需感知的，client和server用其来关联自定义信息到RPC调用动作（可以类比HTTP请求中的header）。*

​	*每个编程语言关于metadata的操作不尽相同。*

### * Channels

​	A gRPC channel provides a connection to a gRPC server on a specified host and port. It is used when creating a client stub. Clients can specify channel arguments to modify gRPC’s default behavior, such as switching message compression on or off. A channel has state, including `connected` and `idle`.

​	How gRPC deals with closing a channel is language dependent. Some languages also permit querying channel state.

​	*一个gRPC channel对应一个在特定host、port开放的gRPC server。在创建client stub时就会用到channel。Clients 可以指定不同的channel参数去改变gRPC的默认行为，比如改变message的压缩情况（压缩or不压缩）。channel具有state状态，包括`connected`、`idle`。*

​	*gRPC关闭channel的流程依赖编程语言实现。一些编程语言还支持查询channel的state状态。*

# 3. FAQ

## 3.1 什么是gRPC

​	gRPC is a modern, open source remote procedure call (RPC) framework that can run anywhere. It enables client and server applications to communicate transparently, and makes it easier to build connected systems.

​	gRPC是开源的RPC远程过程调用框架。它使得client和server应用能够透明通信（即gRPC维护client和server之间的通信，我们只需按照一定语法进行编写即可，无需关心底层通信实现），使用gRPC能够使系统间联系更加简单。

## 3.2 适合gRPC的场景

+ 低延迟、高扩展性、分布式系统

  Low latency, highly scalable, distributed systems.

+ 开发与云服务器通信的移动客户端

  Developing mobile clients which are communicating to a cloud server.

+ 设计准确、高效、语言无关的（通信）协议

  Designing a new protocol that needs to be accurate, efficient and language independent.

+ 分层设计，以支持扩展性，例如：（用户身份）认证、负载均衡、日志记录和监控等

  Layered design to enable extension eg. authentication, load balancing, logging and monitoring etc.

## 3.3 gRPC release支持多久

​	**The gRPC project does not do LTS releases**. Given the rolling release model above, we support the current, latest release and the release prior to that. Support here means bug fixes and security fixes.

## 3.4 我能在浏览器上使用gRPC吗

​	The [gRPC-Web](https://github.com/grpc/grpc-web) project is Generally Available.

## 3.5 可以在gRPC上使用其他数据格式(JSON、protobuf、Thrift、XML)吗

​	Yes. gRPC is designed to be extensible to support multiple content types. The initial release contains support for Protobuf and with external support for other content types such as FlatBuffers and Thrift, at varying levels of maturity.

## 3.6 Can I use gRPC in a service mesh

​	Yes. gRPC applications can be deployed in a service mesh like any other application. gRPC also supports [xDS APIs](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol) which enables deploying gRPC applications in a service mesh without sidecar proxies. The proxyless service mesh features supported in gRPC are listed [here](https://github.com/grpc/grpc/blob/master/doc/grpc_xds_features.md).

## 3.7 gRPC如何助长移动端开发

​	gRPC and Protobuf provide an easy way to precisely define a service and auto generate reliable client libraries for iOS, Android and the servers providing the back end. The clients can take advantage of advanced streaming and connection features which help save bandwidth, do more over fewer TCP connections and save CPU usage and battery life.

​	简言之gRPC可生成IOS、Android的client客户端代码，这些代码中采用先进的流处理和连接特性，能够有效节省带宽、节省CPU使用率，（变向）延长电池寿命。

## 3.8 Why is gRPC better than any binary blob over HTTP/2?

​	This is largely what gRPC is on the wire. However gRPC is also a set of libraries that will provide higher-level features consistently across platforms that common HTTP libraries typically do not. Examples of such features include:

+ interaction with flow-control at the application layer
+ cascading call-cancellation
+ load balancing & failover

## 3.9 Why is gRPC better/worse than REST?

​	gRPC largely follows HTTP semantics over HTTP/2 but we explicitly allow for full-duplex streaming. We diverge from typical REST conventions as we use static paths for performance reasons during call dispatch as parsing call parameters from paths, query parameters and payload body adds latency and complexity. We have also formalized a set of errors that we believe are more directly applicable to API use cases than the HTTP status codes.

​	gRPC很大程度上遵循基于HTTP/2的HTTP语义，但明确支持全双工流。我们偏离了典型的REST协议，我们出于性能考量，决定在分发调用请求时采用静态路由，因为从路由中解析调度参数、查询参数和数据载体payload body会带来额外的延迟和（处理/设计的）复杂性。我们同时还定义了一组errors，确信其比HTTP状态码更适用于API。

# 4. Languages

## Java

### Quick start

> [Quick start | Java | gRPC](https://www.grpc.io/docs/languages/java/quickstart/)	=>	跟着做即可

*ps：虽然和这个没关系，但是我顺便安装了一下AdoptOpenJDK-11，用的jenv管理jdk8和jdk11*

### 基础教程

​	通过该基础教程，将学到：

+ Define a service in a `.proto` file.
+ Generate server and client code using the protocol buffer compiler.
+ Use the Java gRPC API to write a simple client and server for your service.

​	It assumes that you have read the [Introduction to gRPC](https://www.grpc.io/docs/what-is-grpc/introduction/) and are familiar with [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview). Note that the example in this tutorial uses the [proto3](https://github.com/google/protobuf/releases) version of the protocol buffers language: you can find out more in the [proto3 language guide](https://developers.google.com/protocol-buffers/docs/proto3) and [Java generated code guide](https://developers.google.com/protocol-buffers/docs/reference/java-generated).

#### 为什么使用gRPC

​	Our example is a simple route mapping application that lets clients get information about features on their route, create a summary of their route, and exchange route information such as traffic updates with the server and other clients.

​	With gRPC we can define our service once in a `.proto` file and generate clients and servers in any of gRPC’s supported languages, which in turn can be run in environments ranging from servers inside a large data center to your own tablet — all the complexity of communication between different languages and environments is handled for you by gRPC. We also get all the advantages of working with protocol buffers, including efficient serialization, a simple IDL, and easy interface updating.

简言之：

1. gRPC方便客户端获取路由信息，以及和服务端交换路由信息
2. gRPC使得开发时对语言的依赖性降低，只要符合接口定义，可以使用任何gRPC支持的语言进行开发
3. gRPC默认使用protobuf，享有高效的序列化、简单的IDL、（更方便定义）易于更新的接口

#### 示例代码

The example code for our tutorial is in [grpc/grpc-java/examples/src/main/java/io/grpc/examples/routeguide](https://github.com/grpc/grpc-java/tree/master/examples/src/main/java/io/grpc/examples/routeguide). To download the example, clone the latest release in `grpc-java` repository by running the following command:

```shell
$ git clone -b v1.39.0 https://github.com/grpc/grpc-java.git
```

Then change your current directory to `grpc-java/examples`:

```shell
$ cd grpc-java/examples
```

#### 定义service

> Our first step (as you’ll know from the [Introduction to gRPC](https://www.grpc.io/docs/what-is-grpc/introduction/)) is to define the gRPC *service* and the method *request* and *response* types using [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview). You can see the complete .proto file in [grpc-java/examples/src/main/proto/route_guide.proto](https://github.com/grpc/grpc-java/blob/master/examples/src/main/proto/route_guide.proto).

+ `java_package`用于指定`.proto`生成的java类所在的包路径

  ```java
  option java_package = "io.grpc.examples.routeguide";
  ```

  > This specifies the package we want to use for our generated Java classes. If no explicit `java_package` option is given in the .proto file, then by default the proto package (specified using the “package” keyword) will be used. However, proto packages generally do not make good Java packages since proto packages are not expected to start with reverse domain names. If we generate code in another language from this .proto, the `java_package` option has no effect.

+ `service`声明一个命名服务

  ```java
  service RouteGuide {
     ...
  }
  ```

Then we define `rpc` methods inside our service definition, specifying their request and response types. gRPC lets you define four kinds of service methods, all of which are used in the `RouteGuide` service:

- A *simple RPC* where the client sends a request to the server using the stub and waits for a response to come back, just like a normal function call.

  ```java
  // Obtains the feature at a given position.
  rpc GetFeature(Point) returns (Feature) {}
  ```

- A *server-side streaming RPC* where the client sends a request to the server and gets a stream to read a sequence of messages back. The client reads from the returned stream until there are no more messages. As you can see in our example, you specify a server-side streaming method by placing the `stream` keyword before the *response* type.

  ```java
  // Obtains the Features available within the given Rectangle.  Results are
  // streamed rather than returned at once (e.g. in a response message with a
  // repeated field), as the rectangle may cover a large area and contain a
  // huge number of features.
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
  ```

- A *client-side streaming RPC* where the client writes a sequence of messages and sends them to the server, again using a provided stream. Once the client has finished writing the messages, it waits for the server to read them all and return its response. You specify a client-side streaming method by placing the `stream` keyword before the *request* type.

  ```java
  // Accepts a stream of Points on a route being traversed, returning a
  // RouteSummary when traversal is completed.
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
  ```

- A *bidirectional streaming RPC* where <u>both sides send a sequence of messages using a read-write stream.</u> **The two streams operate independently**, so clients and servers can read and write in whatever order they like: for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes. The order of messages in each stream is preserved. You specify this type of method by placing the `stream` keyword before both the request and the response.

  ```java
  // Accepts a stream of RouteNotes sent while a route is being traversed,
  // while receiving other RouteNotes (e.g. from other users).
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
  ```

除了声明service，当然`.proto`也包含了我们service中自定义的request和response的types：

```java
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

#### 生成client和server代码

通过`protoc`插件生成`.proto`文件对应的java代码。

>  You need to use the [proto3](https://github.com/google/protobuf/releases) compiler (which supports both proto2 and proto3 syntax) in order to generate gRPC services.

​	When using Gradle or Maven, the protoc build plugin can generate the necessary code as part of the build. You can refer to the [grpc-java README](https://github.com/grpc/grpc-java/blob/master/README.md) for how to generate code from your own `.proto` files.

​	The following classes are generated from our service definition:

- `Feature.java`, `Point.java`, `Rectangle.java`, and others which contain all the protocol buffer code to populate, serialize, and retrieve our request and response message types.
- `RouteGuideGrpc.java` which contains (along with some other useful code):
  - a base class for `RouteGuide` servers to implement, `RouteGuideGrpc.RouteGuideImplBase`, with all the methods defined in the `RouteGuide` service.
  - *stub* classes that clients can use to talk to a `RouteGuide` server.

#### 创建 server

​	First let’s look at how we create a `RouteGuide` server. If you’re only interested in creating gRPC clients, you can skip this section and go straight to [Creating the client](https://www.grpc.io/docs/languages/java/basics/#client) (though you might find it interesting anyway!).

​	There are two parts to making our `RouteGuide` service do its job:

- Overriding the service base class generated from our service definition: doing the actual “work” of our service.
- Running a gRPC server to listen for requests from clients and return the service responses.

> You can find our example `RouteGuide` server in [grpc-java/examples/src/main/java/io/grpc/examples/routeguide/RouteGuideServer.java](https://github.com/grpc/grpc-java/blob/master/examples/src/main/java/io/grpc/examples/routeguide/RouteGuideServer.java). Let’s take a closer look at how it works.

##### Implementing RouteGuide

As you can see, our server has a `RouteGuideService` class that extends the generated `RouteGuideGrpc.RouteGuideImplBase` abstract class:

```java
private static class RouteGuideService extends RouteGuideGrpc.RouteGuideImplBase {
...
}
```

##### Simple RPC

`RouteGuideService` implements all our service methods. Let’s look at the simplest method first, `GetFeature()`, which just gets a `Point` from the client and returns the corresponding feature information from its database in a `Feature`.

```java
@Override
public void getFeature(Point request, StreamObserver<Feature> responseObserver) {
  responseObserver.onNext(checkFeature(request));
  responseObserver.onCompleted();
}

...

private Feature checkFeature(Point location) {
  for (Feature feature : features) {
    if (feature.getLocation().getLatitude() == location.getLatitude()
        && feature.getLocation().getLongitude() == location.getLongitude()) {
      return feature;
    }
  }

  // No feature was found, return an unnamed feature.
  return Feature.newBuilder().setName("").setLocation(location).build();
}
```

The `getFeature()` method takes two parameters:

- `Point`: the request
- `StreamObserver<Feature>`: a response observer, which is a special interface for the server to call with its response.

To return our response to the client and complete the call:

1. We construct and populate a `Feature` response object to return to the client, as specified in our service definition. In this example, we do this in a separate private `checkFeature()` method.
2. We use the response observer’s `onNext()` method to return the `Feature`.
3. We use the response observer’s `onCompleted()` method to specify that we’ve finished dealing with the RPC.

##### Server-side streaming RPC

Next let’s look at one of our streaming RPCs. `ListFeatures` is a server-side streaming RPC, so we need to send back multiple `Feature`s to our client.

```java
private final Collection<Feature> features;

...

@Override
public void listFeatures(Rectangle request, StreamObserver<Feature> responseObserver) {
  int left = min(request.getLo().getLongitude(), request.getHi().getLongitude());
  int right = max(request.getLo().getLongitude(), request.getHi().getLongitude());
  int top = max(request.getLo().getLatitude(), request.getHi().getLatitude());
  int bottom = min(request.getLo().getLatitude(), request.getHi().getLatitude());

  for (Feature feature : features) {
    if (!RouteGuideUtil.exists(feature)) {
      continue;
    }

    int lat = feature.getLocation().getLatitude();
    int lon = feature.getLocation().getLongitude();
    if (lon >= left && lon <= right && lat >= bottom && lat <= top) {
      responseObserver.onNext(feature);
    }
  }
  responseObserver.onCompleted();
}
```

Like the simple RPC, this method gets a request object (the `Rectangle` in which our client wants to find `Feature`s) and a `StreamObserver` response observer.

This time, we get as many `Feature` objects as we need to return to the client (in this case, we select them from the service’s feature collection based on whether they’re inside our request `Rectangle`), and write them each in turn to the response observer using its `onNext()` method. Finally, as in our simple RPC, we use the response observer’s `onCompleted()` method to tell gRPC that we’ve finished writing responses.

##### Client-side streaming RPC

Now let’s look at something a little more complicated: the client-side streaming method `RecordRoute()`, where we get a stream of `Point`s from the client and return a single `RouteSummary` with information about their trip.

```java
@Override
public StreamObserver<Point> recordRoute(final StreamObserver<RouteSummary> responseObserver) {
  return new StreamObserver<Point>() {
    int pointCount;
    int featureCount;
    int distance;
    Point previous;
    long startTime = System.nanoTime();

    @Override
    public void onNext(Point point) {
      pointCount++;
      if (RouteGuideUtil.exists(checkFeature(point))) {
        featureCount++;
      }
      // For each point after the first, add the incremental distance from the previous point
      // to the total distance value.
      if (previous != null) {
        distance += calcDistance(previous, point);
      }
      previous = point;
    }

    @Override
    public void onError(Throwable t) {
      logger.log(Level.WARNING, "Encountered error in recordRoute", t);
    }

    @Override
    public void onCompleted() {
      long seconds = NANOSECONDS.toSeconds(System.nanoTime() - startTime);
      responseObserver.onNext(RouteSummary.newBuilder().setPointCount(pointCount)
          .setFeatureCount(featureCount).setDistance(distance)
          .setElapsedTime((int) seconds).build());
      responseObserver.onCompleted();
    }
  };
}
```

As you can see, like the previous method types our method gets a `StreamObserver` response observer parameter, but this time it returns a `StreamObserver` for the client to write its `Point`s.

In the method body we instantiate an anonymous `StreamObserver` to return, in which we:

- Override the `onNext()` method to get features and other information each time the client writes a `Point` to the message stream.
- Override the `onCompleted()` method (called when the *client* has finished writing messages) to populate and build our `RouteSummary`. We then call our method’s own response observer’s `onNext()` with our `RouteSummary`, and then call its `onCompleted()` method to finish the call from the server side.

##### Bidirectional streaming RPC

Finally, let’s look at our bidirectional streaming RPC `RouteChat()`.

```java
@Override
public StreamObserver<RouteNote> routeChat(final StreamObserver<RouteNote> responseObserver) {
  return new StreamObserver<RouteNote>() {
    @Override
    public void onNext(RouteNote note) {
      List<RouteNote> notes = getOrCreateNotes(note.getLocation());

      // Respond with all previous notes at this location.
      for (RouteNote prevNote : notes.toArray(new RouteNote[0])) {
        responseObserver.onNext(prevNote);
      }

      // Now add the new note to the list
      notes.add(note);
    }

    @Override
    public void onError(Throwable t) {
      logger.log(Level.WARNING, "Encountered error in routeChat", t);
    }

    @Override
    public void onCompleted() {
      responseObserver.onCompleted();
    }
  };
}
```

As with our client-side streaming example, we both get and return a `StreamObserver` response observer, **except this time we return values via our method’s response observer while the client is still writing messages to *their* message stream**. 

The syntax for reading and writing here is exactly the same as for our client-streaming and server-streaming methods. 

Although **each side will always get the other’s messages in the order they were written**, <u>both the client and server can read and write in any order — the streams operate completely independently.</u>

#### 运行 server

​	Once we’ve implemented all our methods, we also need to start up a gRPC server so that clients can actually use our service. The following snippet shows how we do this for our `RouteGuide` service:

```java
public RouteGuideServer(int port, URL featureFile) throws IOException {
  this(ServerBuilder.forPort(port), port, RouteGuideUtil.parseFeatures(featureFile));
}

/** Create a RouteGuide server using serverBuilder as a base and features as data. */
public RouteGuideServer(ServerBuilder<?> serverBuilder, int port, Collection<Feature> features) {
  this.port = port;
  server = serverBuilder.addService(new RouteGuideService(features))
      .build();
}
...
public void start() throws IOException {
  server.start();
  logger.info("Server started, listening on " + port);
 ...
}
```

As you can see, we build and start our server using a `ServerBuilder`.

To do this, we:

1. Specify the address and port we want to use to listen for client requests using the builder’s `forPort()` method.
2. Create an instance of our service implementation class `RouteGuideService` and pass it to the builder’s `addService()` method.
3. Call `build()` and `start()` on the builder to create and start an RPC server for our service.

#### 创建 client

> In this section, we’ll look at creating a client for our `RouteGuide` service. You can see our complete example client code in [grpc-java/examples/src/main/java/io/grpc/examples/routeguide/RouteGuideClient.java](https://github.com/grpc/grpc-java/blob/master/examples/src/main/java/io/grpc/examples/routeguide/RouteGuideClient.java).

##### Instantiating a stub

​	To call service methods, we first need to create a *stub*, or rather, two stubs:

- a *blocking/synchronous* stub: this means that the RPC call waits for the server to respond, and will either return a response or raise an exception.
- a *non-blocking/asynchronous* stub that makes non-blocking calls to the server, where the response is returned asynchronously. You can make certain types of streaming call only using the asynchronous stub.

​	First we need to create a gRPC *channel* for our stub, specifying the server address and port we want to connect to:

```java
public RouteGuideClient(String host, int port) {
  this(ManagedChannelBuilder.forAddress(host, port).usePlaintext());
}

/** Construct client for accessing RouteGuide server using the existing channel. */
public RouteGuideClient(ManagedChannelBuilder<?> channelBuilder) {
  channel = channelBuilder.build();
  blockingStub = RouteGuideGrpc.newBlockingStub(channel);
  asyncStub = RouteGuideGrpc.newStub(channel);
}
```

​	We use a `ManagedChannelBuilder` to create the channel.

​	Now we can use the channel to create our stubs using the `newStub` and `newBlockingStub` methods provided in the `RouteGuideGrpc` class we generated from our .proto.

```java
blockingStub = RouteGuideGrpc.newBlockingStub(channel);
asyncStub = RouteGuideGrpc.newStub(channel);
```

##### Calling service methods

*Now let’s look at how we call our service methods.*

##### Simple RPC

Calling the simple RPC `GetFeature` on the blocking stub is as straightforward as calling a local method.

```java
Point request = Point.newBuilder().setLatitude(lat).setLongitude(lon).build();
Feature feature;
try {
  feature = blockingStub.getFeature(request);
} catch (StatusRuntimeException e) {
  logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
  return;
}
```

We create and populate a request protocol buffer object (in our case `Point`), pass it to the `getFeature()` method on our blocking stub, and get back a `Feature`.

If an error occurs, it is encoded as a `Status`, which we can obtain from the `StatusRuntimeException`.

##### Server-side streaming RPC

Next, let’s look at a server-side streaming call to `ListFeatures`, which returns a stream of geographical `Feature`s:

```java
Rectangle request =
    Rectangle.newBuilder()
        .setLo(Point.newBuilder().setLatitude(lowLat).setLongitude(lowLon).build())
        .setHi(Point.newBuilder().setLatitude(hiLat).setLongitude(hiLon).build()).build();
Iterator<Feature> features;
try {
  features = blockingStub.listFeatures(request);
} catch (StatusRuntimeException e) {
  logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
  return;
}
```

As you can see, it’s very similar to the simple RPC we just looked at, except instead of returning a single `Feature`, the method returns an `Iterator` that the client can use to read all the returned `Feature`s.

##### Client-side streaming RPC

Now for something a little more complicated: the client-side streaming method `RecordRoute`, where we send a stream of `Point`s to the server and get back a single `RouteSummary`. For this method we need to use the **asynchronous stub**. If you’ve already read [Creating the server](https://www.grpc.io/docs/languages/java/basics/#server) some of this may look very familiar - <u>asynchronous streaming RPCs are implemented in a similar way on both sides</u>.

```java
public void recordRoute(List<Feature> features, int numPoints) throws InterruptedException {
  info("*** RecordRoute");
  final CountDownLatch finishLatch = new CountDownLatch(1);
  StreamObserver<RouteSummary> responseObserver = new StreamObserver<RouteSummary>() {
    @Override
    public void onNext(RouteSummary summary) {
      info("Finished trip with {0} points. Passed {1} features. "
          + "Travelled {2} meters. It took {3} seconds.", summary.getPointCount(),
          summary.getFeatureCount(), summary.getDistance(), summary.getElapsedTime());
    }

    @Override
    public void onError(Throwable t) {
      Status status = Status.fromThrowable(t);
      logger.log(Level.WARNING, "RecordRoute Failed: {0}", status);
      finishLatch.countDown();
    }

    @Override
    public void onCompleted() {
      info("Finished RecordRoute");
      finishLatch.countDown();
    }
  };

  StreamObserver<Point> requestObserver = asyncStub.recordRoute(responseObserver);
  try {
    // Send numPoints points randomly selected from the features list.
    Random rand = new Random();
    for (int i = 0; i < numPoints; ++i) {
      int index = rand.nextInt(features.size());
      Point point = features.get(index).getLocation();
      info("Visiting point {0}, {1}", RouteGuideUtil.getLatitude(point),
          RouteGuideUtil.getLongitude(point));
      requestObserver.onNext(point);
      // Sleep for a bit before sending the next one.
      Thread.sleep(rand.nextInt(1000) + 500);
      if (finishLatch.getCount() == 0) {
        // RPC completed or errored before we finished sending.
        // Sending further requests won't error, but they will just be thrown away.
        return;
      }
    }
  } catch (RuntimeException e) {
    // Cancel RPC
    requestObserver.onError(e);
    throw e;
  }
  // Mark the end of requests
  requestObserver.onCompleted();

  // Receiving happens asynchronously
  finishLatch.await(1, TimeUnit.MINUTES);
}
```

As you can see, to call this method we need to create a `StreamObserver`, which implements a special interface for the server to call with its `RouteSummary` response. In our `StreamObserver` we:

- Override the `onNext()` method to print out the returned information when the server writes a `RouteSummary` to the message stream.
- Override the `onCompleted()` method (called when the *server* has completed the call on its side) to reduce a `CountDownLatch` that we can check to see if the server has finished writing.

We then pass the `StreamObserver` to the asynchronous stub’s `recordRoute()` method and get back our own `StreamObserver` request observer to write our `Point`s to send to the server. Once we’ve finished writing points, we use the request observer’s `onCompleted()` method to tell gRPC that we’ve finished writing on the client side. Once we’re done, we check our `CountDownLatch` to check that the server has completed on its side.

##### Bidirectional streaming RPC

Finally, let’s look at our bidirectional streaming RPC `RouteChat()`.

```java
public void routeChat() throws Exception {
  info("*** RoutChat");
  final CountDownLatch finishLatch = new CountDownLatch(1);
  StreamObserver<RouteNote> requestObserver =
      asyncStub.routeChat(new StreamObserver<RouteNote>() {
        @Override
        public void onNext(RouteNote note) {
          info("Got message \"{0}\" at {1}, {2}", note.getMessage(), note.getLocation()
              .getLatitude(), note.getLocation().getLongitude());
        }

        @Override
        public void onError(Throwable t) {
          Status status = Status.fromThrowable(t);
          logger.log(Level.WARNING, "RouteChat Failed: {0}", status);
          finishLatch.countDown();
        }

        @Override
        public void onCompleted() {
          info("Finished RouteChat");
          finishLatch.countDown();
        }
      });

  try {
    RouteNote[] requests =
        {newNote("First message", 0, 0), newNote("Second message", 0, 1),
            newNote("Third message", 1, 0), newNote("Fourth message", 1, 1)};

    for (RouteNote request : requests) {
      info("Sending message \"{0}\" at {1}, {2}", request.getMessage(), request.getLocation()
          .getLatitude(), request.getLocation().getLongitude());
      requestObserver.onNext(request);
    }
  } catch (RuntimeException e) {
    // Cancel RPC
    requestObserver.onError(e);
    throw e;
  }
  // Mark the end of requests
  requestObserver.onCompleted();

  // Receiving happens asynchronously
  finishLatch.await(1, TimeUnit.MINUTES);
}
```

As with our client-side streaming example, we both get and return a `StreamObserver` response observer, except this time we send values via our method’s response observer while the server is still writing messages to *their* message stream. The syntax for reading and writing here is exactly the same as for our client-streaming method. Although each side will always get the other’s messages in the order they were written, both the client and server can read and write in any order — the streams operate completely independently.

#### Try it out

>  Follow the instructions in the [example directory README](https://github.com/grpc/grpc-java/blob/master/examples/README.md) to build and run the client and server.

### ALTS

> An overview of gRPC authentication in Java using Application Layer Transport Security (ALTS).

#### 概述

​	**Application Layer Transport Security (ALTS)** is a mutual authentication and transport encryption system developed by Google. It is used for securing RPC communications within Google’s infrastructure. ALTS is similar to mutual TLS but has been designed and optimized to meet the needs of Google’s production environments. For more information, take a look at the [ALTS whitepaper](https://cloud.google.com/security/encryption-in-transit/application-layer-transport-security).

​	简言之，ALTS就是类似TLS的产物，用于通信双方的身份认证和数据加密，使得google的基建RPC通信能安全进行。ALTS类似TLS，google在其基础上进行优化实现，可用于生产环境。

​	ALTS在gRPC方面具有如下特性：

+ Create gRPC servers & clients with ALTS as the transport security protocol.
+ ALTS connections are end-to-end protected with privacy and integrity（隐私和完整性）.
+ Applications can access peer information such as the peer service account.
+ Client authorization and server authorization support.（支持客户端授权、服务端授权）
+ Minimal code changes to enable ALTS.（启用ALTS的代码侵入性小）

​	gRPC users can configure their applications to use ALTS as a transport security protocol with few lines of code.

> Note that ALTS is fully functional if the application runs on [Google Cloud Platform](https://cloud.google.com/). ALTS could be run on any platforms with a pluggable [ALTS handshaker service](https://github.com/grpc/grpc/blob/7e367da22a137e2e7caeae8342c239a91434ba50/src/proto/grpc/gcp/handshaker.proto#L224-L234).

#### gRPC Client使用ALTS

​	gRPC clients can use ALTS credentials to connect to servers, as illustrated in the following code excerpt:

```java
import io.grpc.alts.AltsChannelBuilder;
import io.grpc.ManagedChannel;

ManagedChannel managedChannel = AltsChannelBuilder.forTarget(serverAddress).build();
```

#### gRPC Server使用ALTS

​	gRPC servers can use ALTS credentials to allow clients to connect to them, as illustrated next:

```java
import io.grpc.alts.AltsServerBuilder;
import io.grpc.Server;

Server server = AltsServerBuilder.forPort(<port>)
    .addService(new MyServiceImpl()).build().start();
```

#### Server Authorization

gRPC has built-in server authorization support using ALTS. A gRPC client using ALTS can set the expected server service accounts prior to establishing a connection. Then, at the end of the handshake, server authorization guarantees that the server identity matches one of the service accounts specified by the client. Otherwise, the connection fails.

```java
import io.grpc.alts.AltsChannelBuilder;
import io.grpc.ManagedChannel;

ManagedChannel channel =
    AltsChannelBuilder.forTarget(serverAddress)
        .addTargetServiceAccount("expected_server_service_account1")
        .addTargetServiceAccount("expected_server_service_account2")
        .build();
```

#### Client Authorization

On a successful connection, the peer information (e.g., client’s service account) is stored in the [AltsContext](https://github.com/grpc/grpc/blob/master/src/proto/grpc/gcp/altscontext.proto). gRPC provides a utility library for client authorization check. Assuming that the server knows the expected client identity (e.g., `foo@iam.gserviceaccount.com`), it can run the following example codes to authorize the incoming RPC.

```java
import io.grpc.alts.AuthorizationUtil;
import io.grpc.ServerCall;
import io.grpc.Status;

ServerCall<?, ?> call;
Status status = AuthorizationUtil.clientAuthorizationCheck(
    call, Lists.newArrayList("foo@iam.gserviceaccount.com"));
```

### API

> [grpc-all 1.39.0 API](https://grpc.github.io/grpc-java/javadoc/)

### Generated code

#### Packages

​	大意就是，`java_package`指定生成的java类的包路径；生成的java类的类名为`.proto`文件中`service`服务名加上后缀`Grpc`。

For each service defined in a .proto file, the Java code generation produces a Java class. The class name is the service’s name suffixed by `Grpc`. The package for the generated code is specified in the .proto file using the `java_package` option.

For example, if `ServiceName` is defined in a .proto file containing the following:

```protobuf
package grpcexample;

option java_package = "io.grpc.examples";
```

Then the generated class will be `io.grpc.examples.ServiceNameGrpc`.

If `java_package` is not specified, the generated class will use the `package` as specified in the .proto file. This should be avoided, as proto packages usually do not begin with a reversed domain name.

#### Service Stub

​	生成的Java类里面含有一个抽象内部类，其名带有后缀`ImplBase`。其为`.proto`的`service`定义的所有rpc方法提供了默认实现，如果不重写对应的方法，默认实现就是返回error，告知client此方法未实现。

The generated Java code contains an inner abstract class suffixed with `ImplBase`, such as `ServiceNameImplBase`. This class defines one Java method for each method in the service definition. It is up to the service implementer to extend this class and implement the functionality of these methods. Without being overridden, the methods return an error to the client saying the method is unimplemented.

​	ServiceNameImplBase 中stub方法声明形式与其处理的RPC类型有关（4种 gRPC service方法: unary、server- streaming、client-streaming、bidirectional-streaming）

The signatures of the stub methods in `ServiceNameImplBase` vary depending on the type of RPCs it handles. There are four types of gRPC service methods: unary, server-streaming, client-streaming, and bidirectional-streaming.

##### Unary

The service stub signature for a unary RPC method `unaryExample`：

```java
public void unaryExample(
    RequestType request,
    StreamObserver<ResponseType> responseObserver)
```

##### Server-streaming

The service stub signature for a server-streaming RPC method `serverStreamingExample`:

```java
public void serverStreamingExample(
    RequestType request,
    StreamObserver<ResponseType> responseObserver)
```

<u>Notice that the signatures for unary and server-streaming RPCs are the same.</u> A single `RequestType` is received from the client, and the service implementation sends its response(s) by invoking `responseObserver.onNext(ResponseType response)`.

##### Client-streaming

The service stub signature for a client-streaming RPC method `clientStreamingExample`:

```java
public StreamObserver<RequestType> clientStreamingExample(
    StreamObserver<ResponseType> responseObserver)
```

##### Bidirectional-streaming

The service stub signature for a bidirectional-streaming RPC method `bidirectionalStreamingExample`:

```java
public StreamObserver<RequestType> bidirectionalStreamingExample(
    StreamObserver<ResponseType> responseObserver)
```

<u>The signatures for client and bidirectional-streaming RPCs are the same</u>. Since the client can send multiple messages to the service, the service implementation is responsible for returning a `StreamObserver<RequestType>` instance. This `StreamObserver` is invoked whenever additional messages are received from the client.

#### Client Stubs

client端生成的stub代码包装了`Channel`，stub通过channel向service发送数据。

The generated class also contains stubs for use by gRPC clients to call methods defined by the service. Each stub wraps a `Channel`, supplied by the user of the generated code. The stub uses this channel to send RPCs to the service.

gRPC支持生成3种类型的stub方法：

+ **asynchronous**
+ **blocking**
+ **future**

<u>gRPC Java generates code for three types of stubs: **asynchronous**, **blocking**, and **future**</u>. Each type of stub has a corresponding class in the generated code, such as `ServiceNameStub`, `ServiceNameBlockingStub`, and `ServiceNameFutureStub`.

##### Asynchronous Stub

RPCs made via an asynchronous stub operate entirely through callbacks on `StreamObserver`.

The asynchronous stub contains one Java method for each method from the service definition.

A new asynchronous stub is instantiated via the `ServiceNameGrpc.newStub(Channel channel)` static method. 

###### Unary

The asynchronous stub signature for a unary RPC method `unaryExample`:

```java

public void unaryExample(
    RequestType request,
    StreamObserver<ResponseType> responseObserver)
```

###### Server-streaming

The asynchronous stub signature for a server-streaming RPC method `serverStreamingExample`:

```java
public void serverStreamingExample(
    RequestType request,
    StreamObserver<ResponseType> responseObserver)
```

###### Client-streaming[ ](https://www.grpc.io/docs/languages/java/generated-code/#client-streaming-1)

The asynchronous stub signature for a client-streaming RPC method `clientStreamingExample`:

```java
public StreamObserver<RequestType> clientStreamingExample(
    StreamObserver<ResponseType> responseObserver)
```

###### Bidirectional-streaming

The asynchronous stub signature for a bidirectional-streaming RPC method `bidirectionalStreamingExample`:

```java
public StreamObserver<RequestType> bidirectionalStreamingExample(
    StreamObserver<ResponseType> responseObserver)
```

##### Blocking Stub

RPCs made through a blocking stub, as the name implies, block until the response from the service is available.

The blocking stub contains one Java method for each unary and server-streaming method in the service definition. **Blocking stubs do not support client-streaming or bidirectional-streaming RPCs**.

A new blocking stub is instantiated via the `ServiceNameGrpc.newBlockingStub(Channel channel)` static method.

###### Unary

The blocking stub signature for a unary RPC method `unaryExample`:

```java
public ResponseType unaryExample(RequestType request)
```

###### Server-streaming

The blocking stub signature for a server-streaming RPC method `serverStreamingExample`:

```java
public Iterator<ResponseType> serverStreamingExample(RequestType request)
```

##### Future Stub

RPCs made via a future stub wrap the return value of the asynchronous stub in a `GrpcFuture<ResponseType>`, which implements the `com.google.common.util.concurrent.ListenableFuture` interface.

The future stub contains one Java method for each unary method in the service definition. **Future stubs do not support streaming calls**.

A new future stub is instantiated via the `ServiceNameGrpc.newFutureStub(Channel channel)` static method.

###### Unary

The future stub signature for a unary RPC method `unaryExample`:

```java
public ListenableFuture<ResponseType> unaryExample(RequestType request)
```

#### 代码生成

Typically the build system handles creation of the gRPC generated code.

For protobuf-based codegen, you can put your `.proto` files in the `src/main/proto` and `src/test/proto` directories along with an appropriate plugin.

A typical [protobuf-maven-plugin](https://www.xolstice.org/protobuf-maven-plugin/) configuration for generating gRPC and Protocol Buffers code would look like the following:

```xml
<build>
  <extensions>
    <extension>
      <groupId>kr.motd.maven</groupId>
      <artifactId>os-maven-plugin</artifactId>
      <version>1.4.1.Final</version>
    </extension>
  </extensions>
  <plugins>
    <plugin>
      <groupId>org.xolstice.maven.plugins</groupId>
      <artifactId>protobuf-maven-plugin</artifactId>
      <version>0.5.0</version>
      <configuration>
        <protocArtifact>com.google.protobuf:protoc:3.3.0:exe:${os.detected.classifier}</protocArtifact>
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.4.0:exe:${os.detected.classifier}</pluginArtifact>
      </configuration>
      <executions>
        <execution>
          <goals>
            <goal>compile</goal>
            <goal>compile-custom</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

Eclipse and NetBeans users should also look at `os-maven-plugin`’s [IDE documentation](https://github.com/trustin/os-maven-plugin#issues-with-eclipse-m2e-or-other-ides).

A typical [protobuf-gradle-plugin](https://github.com/google/protobuf-gradle-plugin) configuration would look like the following:

```gradle
apply plugin: 'java'
apply plugin: 'com.google.protobuf'

buildscript {
  repositories {
    mavenCentral()
  }
  dependencies {
    // ASSUMES GRADLE 2.12 OR HIGHER. Use plugin version 0.7.5 with earlier
    // gradle versions
    classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.0'
  }
}

protobuf {
  protoc {
    artifact = "com.google.protobuf:protoc:3.2.0"
  }
  plugins {
    grpc {
      artifact = 'io.grpc:protoc-gen-grpc-java:1.4.0'
    }
  }
  generateProtoTasks {
    all()*.plugins {
      grpc {}
    }
  }
}
```

Bazel developers can use the [`java_grpc_library`](https://github.com/grpc/grpc-java/blob/master/java_grpc_library.bzl) rule, typically as follows:

```java
load("@grpc_java//:java_grpc_library.bzl", "java_grpc_library")

proto_library(
    name = "helloworld_proto",
    srcs = ["src/main/proto/helloworld.proto"],
)

java_proto_library(
    name = "helloworld_java_proto",
    deps = [":helloworld_proto"],
)

java_grpc_library(
    name = "helloworld_java_grpc",
    srcs = [":helloworld_proto"],
    deps = [":helloworld_java_proto"],
)
```

Android developers, see [Generating client code](https://www.grpc.io/docs/languages/android/basics/#generating-client-code) for reference.

If you wish to invoke the protobuf plugin for gRPC Java directly, the command-line syntax is as follows:

```sh
$ protoc --plugin=protoc-gen-grpc-java \
  --grpc-java_out="$OUTPUT_FILE" --proto_path="$DIR_OF_PROTO_FILE" "$PROTO_FILE"
```

# 5. Platforms

gRPC is supported across different software and hardware platforms.

Each gRPC [language](https://www.grpc.io/docs/languages/) / platform has links to the following pages and more: quick start, tutorials, API reference.

New sections coming soon:

- Flutter
  - Docs coming soon
- Mobile:
  - iOS – docs coming soon

Select a development or target platform to get started:

- [Android](https://www.grpc.io/docs/platforms/android/)
- [Web](https://www.grpc.io/docs/platforms/web/)

# 6. Guides

Task-oriented walkthroughs of common use cases

The documentation covers the following techniques:

[Authentication](https://www.grpc.io/docs/guides/auth/)

An overview of gRPC authentication, including built-in auth mechanisms, and how to plug in your own authentication systems.

[Benchmarking](https://www.grpc.io/docs/guides/benchmarking/)

gRPC is designed to support high-performance open-source RPCs in many languages. This page describes performance benchmarking tools, scenarios considered by tests, and the testing infrastructure.

[Error handling](https://www.grpc.io/docs/guides/error/)

How gRPC deals with errors, and gRPC error codes.

[Performance Best Practices](https://www.grpc.io/docs/guides/performance/)

A user guide of both general and language-specific best practices to improve performance.

----

## 6.1 Authentication

> An overview of gRPC authentication, including built-in auth mechanisms, and how to plug in your own authentication systems.

### 概述

​	gRPC is designed to work with a variety of authentication mechanisms, making it easy to safely use gRPC to talk to other systems. You can use our supported mechanisms - SSL/TLS with or without Google token-based authentication - or you can plug in your own authentication system by extending our provided code.

​	gRPC also provides a simple authentication API that lets you provide all the necessary authentication information as `Credentials` when creating a channel or making a call.

​	gRPC本身设计时就考虑到多种认证机制，使用gRPC能够简单地实现系统间安全通信。可以使用携带or不携带用于认证的Google token的SSL/TLS来实现安全通信，也可以考虑拓展gRPC提供的代码来实现自己的认证系统。

​	gRPC提供了简单的认证API，使得在创建和使用channel的时候能够携带必要的认证信息（诸如`Credentials`）

### 支持的认证机制

​	gRPC内置了以下几种认证机制：

+ **SSL/TLS**

  gRPC has SSL/TLS integration and promotes the use of SSL/TLS to authenticate the server, and to encrypt all the data exchanged between the client and the server. Optional mechanisms are available for clients to provide certificates for mutual authentication.

+ **ALTS**

  gRPC supports [ALTS](https://cloud.google.com/security/encryption-in-transit/application-layer-transport-security) as a transport security mechanism, if the application is running on [Google Cloud Platform (GCP)](https://cloud.google.com/). For details, see one of the following language-specific pages: [ALTS in C++](https://www.grpc.io/docs/languages/cpp/alts/), [ALTS in Go](https://www.grpc.io/docs/languages/go/alts/), [ALTS in Java](https://www.grpc.io/docs/languages/java/alts/), [ALTS in Python](https://www.grpc.io/docs/languages/python/alts/).

+ **Token-based authentication with Google**

  gRPC provides a generic mechanism (described below) to attach metadata based credentials to requests and responses. Additional support for acquiring access tokens (typically OAuth2 tokens) while accessing Google APIs through gRPC is provided for certain auth flows: you can see how this works in our code examples below. In general this mechanism must be used *as well as* SSL/TLS on the channel - <u>Google will not allow connections without SSL/TLS, and most gRPC language implementations will not let you send credentials on an unencrypted channel</u>.

> Warning
>
> Google credentials should only be used to connect to Google services. <u>Sending a Google issued OAuth2 token to a non-Google service could result in this token being stolen and used to impersonate the client to Google services.</u>

### 认证API

> gRPC provides a simple authentication API based around the unified concept of Credentials objects, which can be used when creating an entire gRPC channel or an individual call.

#### Credential types

Credentials can be of two types:

- **Channel credentials**, which are attached to a `Channel`, such as SSL credentials.
- **Call credentials**, which are attached to a call (or `ClientContext` in C++).

​	You can also combine these in a `CompositeChannelCredentials`, allowing you to specify, for example, SSL details for the channel along with call credentials for each call made on the channel. A `CompositeChannelCredentials` associates a `ChannelCredentials` and a `CallCredentials` to create a new `ChannelCredentials`. The result will send the authentication data associated with the composed `CallCredentials` with every call made on the channel.

​	For example, you could create a `ChannelCredentials` from an `SslCredentials` and an `AccessTokenCredentials`. The result when applied to a `Channel` would send the appropriate access token for each call on this channel.

​	Individual `CallCredentials` can also be composed using `CompositeCallCredentials`. The resulting `CallCredentials` when used in a call will trigger the sending of the authentication data associated with the two `CallCredentials`.

#### Using client-side SSL/TLS

​	Now let’s look at how `Credentials` work with one of our supported auth mechanisms. This is the simplest authentication scenario, where a client just wants to authenticate the server and encrypt all data. The example is in C++, but the API is similar for all languages: you can see how to enable SSL/TLS in more languages in our Examples section below.

```protobuf
// Create a default SSL ChannelCredentials object.
auto channel_creds = grpc::SslCredentials(grpc::SslCredentialsOptions());
// Create a channel using the credentials created in the previous step.
auto channel = grpc::CreateChannel(server_name, channel_creds);
// Create a stub on the channel.
std::unique_ptr<Greeter::Stub> stub(Greeter::NewStub(channel));
// Make actual RPC calls on the stub.
grpc::Status s = stub->sayHello(&context, *request, response);
```

​	For advanced use cases such as modifying the root CA or using client certs, the corresponding options can be set in the `SslCredentialsOptions` parameter passed to the factory method.

> Note
>
> Non-POSIX-compliant systems (such as Windows) need to specify the root certificates in `SslCredentialsOptions`, since the defaults are only configured for POSIX filesystems.

#### Using Google token-based authentication

​	gRPC applications can use a simple API to create a credential that works for authentication with Google in various deployment scenarios. Again, our example is in C++ but you can find examples in other languages in our Examples section.

```java
auto creds = grpc::GoogleDefaultCredentials();
// Create a channel, stub and make RPC calls (same as in the previous example)
auto channel = grpc::CreateChannel(server_name, creds);
std::unique_ptr<Greeter::Stub> stub(Greeter::NewStub(channel));
grpc::Status s = stub->sayHello(&context, *request, response);
```

​	This channel credentials object works for applications using Service Accounts as well as for applications running in [Google Compute Engine (GCE)](https://cloud.google.com/compute/). In the former case, the service account’s private keys are loaded from the file named in the environment variable `GOOGLE_APPLICATION_CREDENTIALS`. The keys are used to generate bearer tokens that are attached to each outgoing RPC on the corresponding channel.

​	<u>For applications running in GCE, a default service account and corresponding OAuth2 scopes can be configured during VM setup.</u> At run-time, this credential handles communication with the authentication systems to obtain OAuth2 access tokens and attaches them to each outgoing RPC on the corresponding channel.

#### Extending gRPC to support other authentication mechanisms

​	The Credentials plugin API allows developers to plug in their own type of credentials. This consists of:

- The `MetadataCredentialsPlugin` abstract class, which contains the pure virtual `GetMetadata` method that needs to be implemented by a sub-class created by the developer.
- The `MetadataCredentialsFromPlugin` function, which creates a `CallCredentials` from the `MetadataCredentialsPlugin`.

​	Here is example of a simple credentials plugin which sets an authentication ticket in a custom header.

```c++
class MyCustomAuthenticator : public grpc::MetadataCredentialsPlugin {
  public:
  MyCustomAuthenticator(const grpc::string& ticket) : ticket_(ticket) {}

  grpc::Status GetMetadata(
    grpc::string_ref service_url, grpc::string_ref method_name,
    const grpc::AuthContext& channel_auth_context,
    std::multimap<grpc::string, grpc::string>* metadata) override {
    metadata->insert(std::make_pair("x-custom-auth-ticket", ticket_));
    return grpc::Status::OK;
  }

  private:
  grpc::string ticket_;
};

auto call_creds = grpc::MetadataCredentialsFromPlugin(
  std::unique_ptr<grpc::MetadataCredentialsPlugin>(
    new MyCustomAuthenticator("super-secret-ticket")));
```

​	A deeper integration can be achieved by plugging in a gRPC credentials implementation at the core level. <u>gRPC internals also allow switching out SSL/TLS with other encryption mechanisms.</u>

### Examples

> These authentication mechanisms will be available in all gRPC’s supported languages. The following sections demonstrate how authentication and authorization features described above appear in each language: more languages are coming soon.

#### Java

**Base case - no encryption or authentication**

```java
ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 50051)
    .usePlaintext(true)
    .build();
GreeterGrpc.GreeterStub stub = GreeterGrpc.newStub(channel);
```

**With server authentication SSL/TLS**

<u>In Java we recommend that you use OpenSSL when using gRPC over TLS</u>. You can find details about installing and using OpenSSL and other required libraries for both Android and non-Android Java in the gRPC Java [Security](https://github.com/grpc/grpc-java/blob/master/SECURITY.md#transport-security-tls) documentation.

To enable TLS on a server, a certificate chain and private key need to be specified in PEM format. Such private key should not be using a password. The order of certificates in the chain matters: more specifically, the certificate at the top has to be the host CA, while the one at the very bottom has to be the root CA. <u>The standard TLS port is 443, but we use 8443 below to avoid needing extra permissions from the OS</u>.

```java
Server server = ServerBuilder.forPort(8443)
    // Enable TLS
    .useTransportSecurity(certChainFile, privateKeyFile)
    .addService(TestServiceGrpc.bindService(serviceImplementation))
    .build();
server.start();
```

If the issuing certificate authority is not known to the client then a properly configured `SslContext` or `SSLSocketFactory` should be provided to the `NettyChannelBuilder` or `OkHttpChannelBuilder`, respectively.

On the client side, server authentication with SSL/TLS looks like this:

```java
// With server authentication SSL/TLS
ManagedChannel channel = ManagedChannelBuilder.forAddress("myservice.example.com", 443)
    .build();
GreeterGrpc.GreeterStub stub = GreeterGrpc.newStub(channel);

// With server authentication SSL/TLS; custom CA root certificates; not on Android
ManagedChannel channel = NettyChannelBuilder.forAddress("myservice.example.com", 443)
    .sslContext(GrpcSslContexts.forClient().trustManager(new File("roots.pem")).build())
    .build();
GreeterGrpc.GreeterStub stub = GreeterGrpc.newStub(channel);
```

**Authenticate with Google**

The following code snippet shows how you can call the [Google Cloud PubSub API](https://cloud.google.com/pubsub/overview) using gRPC with a service account. The credentials are loaded from a key stored in a well-known location or by detecting that the application is running in an environment that can provide one automatically, e.g. Google Compute Engine. While this example is specific to Google and its services, similar patterns can be followed for other service providers.

```java
GoogleCredentials creds = GoogleCredentials.getApplicationDefault();
ManagedChannel channel = ManagedChannelBuilder.forTarget("greeter.googleapis.com")
    .build();
GreeterGrpc.GreeterStub stub = GreeterGrpc.newStub(channel)
    .withCallCredentials(MoreCallCredentials.from(creds));
```

## 6.2 Benchmarking

> gRPC is designed to support high-performance open-source RPCs in many languages. This page describes performance benchmarking tools, scenarios considered by tests, and the testing infrastructure.

### 概述

gRPC is designed for both high-performance and high-productivity design of distributed applications. Continuous performance benchmarking is a critical part of the gRPC development workflow. Multi-language performance tests run every few hours against the master branch, and these numbers are reported to a dashboard for visualization.

- [Multi-language performance dashboard @master (latest dev version)](https://performance-dot-grpc-testing.appspot.com/explore?dashboard=5180705743044608)

### 性能测试设计

Each language implements a performance testing worker that implements a gRPC [WorkerService](https://github.com/grpc/grpc/blob/master/src/proto/grpc/testing/worker_service.proto). This service directs the worker to act as either a client or a server for the actual benchmark test, represented as [BenchmarkService](https://github.com/grpc/grpc/blob/master/src/proto/grpc/testing/benchmark_service.proto). That service has two methods:

- UnaryCall - a unary RPC of a simple request that specifies the number of bytes to return in the response
- StreamingCall - a streaming RPC that allows repeated ping-pongs of request and response messages akin to the UnaryCall

![gRPC performance testing worker diagram](https://www.grpc.io/img/testing_framework.png)

These workers are controlled by a [driver](https://github.com/grpc/grpc/blob/master/test/cpp/qps/qps_json_driver.cc) that takes as input a scenario description (in JSON format) and an environment variable specifying the host:port of each worker process.

### Languages under test

The following languages have continuous performance testing as both clients and servers at master:

- C++
- Java
- Go
- C#
- node.js
- Python
- Ruby

In addition to running as both the client-side and server-side of performance tests, all languages are tested as clients against a C++ server, and as servers against a C++ client. This test aims to provide the current upper bound of performance for a given language’s client or server implementation without testing the other side.

Although PHP or mobile environments do not support a gRPC server (which is needed for our performance tests), their client-side performance can be benchmarked using a proxy WorkerService written in another language. This code is implemented for PHP but is not yet in continuous testing mode.

### Scenarios under test

There are several important scenarios under test and displayed in the dashboards above, including the following:

- Contentionless latency - the median and tail response latencies seen with only 1 client sending a single message at a time using StreamingCall
- QPS - the messages/second rate when there are 2 clients and a total of 64 channels, each of which has 100 outstanding messages at a time sent using StreamingCall
- Scalability (for selected languages) - the number of messages/second per server core

Most performance testing is using secure communication and protobufs. Some C++ tests additionally use insecure communication and the generic (non-protobuf) API to display peak performance. Additional scenarios may be added in the future.

### Testing infrastructure

All performance benchmarks are run in our dedicated GKE cluster, where each benchmark worker (a client or a server) gets scheduled to different GKE node (and each GKE node is a separate GCE VM) in one of our worker pools. The source code for the benchmarking framework we use is publicly available in the [test-infra github repository](https://github.com/grpc/test-infra).

Most test instances are 8-core systems, and these are used for both latency and QPS measurement. For C++ and Java, we additionally support QPS testing on 32-core systems. All QPS tests use 2 identical client machines for each server, to make sure that QPS measurement is not client-limited.

## 6.3 Error handling

> How gRPC deals with errors, and gRPC error codes.

### Standard error model

As you’ll have seen in our concepts document and examples, when a gRPC call completes successfully the server returns an `OK` status to the client (depending on the language the `OK` status may or may not be directly used in your code). But what happens if the call isn’t successful?

If an error occurs, gRPC returns one of its error status codes instead, with an optional string error message that provides further details about what happened. Error information is available to gRPC clients in all supported languages.

### Richer error model

The error model described above is the official gRPC error model, is supported by all gRPC client/server libraries, and is independent of the gRPC data format (whether protocol buffers or something else). You may have noticed that it’s quite limited and doesn’t include the ability to communicate error details.

If you’re using protocol buffers as your data format, however, you may wish to consider using the richer error model developed and used by Google as described [here](https://cloud.google.com/apis/design/errors#error_model). This model enables servers to return and clients to consume additional error details expressed as one or more protobuf messages. It further specifies a [standard set of error message types](https://github.com/googleapis/googleapis/blob/master/google/rpc/error_details.proto) to cover the most common needs (such as invalid parameters, quota violations, and stack traces). **The protobuf binary encoding of this extra error information is provided as trailing metadata in the response.**

This richer error model is already supported in the C++, Go, Java, Python, and Ruby libraries, and at least the grpc-web and Node.js libraries have open issues requesting it. Other language libraries may add support in the future if there’s demand, so check their github repos if interested. Note however that the grpc-core library written in C will not likely ever support it since it is purposely data format agnostic.

You could use a similar approach (put error details in trailing response metadata) if you’re not using protocol buffers, but you’d likely need to find or develop library support for accessing this data in order to make practical use of it in your APIs.

There are important considerations to be aware of when deciding whether to use such an extended error model, however, including:

- Library implementations of the extended error model may not be consistent across languages in terms of requirements for and expectations of the error details payload
- Existing proxies, loggers, and other standard HTTP request processors don’t have visibility into the error details and thus wouldn’t be able to leverage them for monitoring or other purposes
- Additional error detail in the trailers interferes with head-of-line blocking, and will decrease HTTP/2 header compression efficiency due to more frequent cache misses
- Larger error detail payloads may run into protocol limits (like max headers size), effectively losing the original error

### Error status codes

Errors are raised by gRPC under various circumstances, from network failures to unauthenticated connections, each of which is associated with a particular status code. The following error status codes are supported in all gRPC languages.

#### General errors

| Case                                                         | Status code                     |
| ------------------------------------------------------------ | ------------------------------- |
| Client application cancelled the request                     | `GRPC_STATUS_CANCELLED`         |
| Deadline expired before server returned status               | `GRPC_STATUS_DEADLINE_EXCEEDED` |
| Method not found on server                                   | `GRPC_STATUS_UNIMPLEMENTED`     |
| Server shutting down                                         | `GRPC_STATUS_UNAVAILABLE`       |
| Server threw an exception (or did something other than returning a status code to terminate the RPC) | `GRPC_STATUS_UNKNOWN`           |

#### Network failures

| Case                                                         | Status code                     |
| ------------------------------------------------------------ | ------------------------------- |
| No data transmitted before deadline expires. Also applies to cases where some data is transmitted and no other failures are detected before the deadline expires | `GRPC_STATUS_DEADLINE_EXCEEDED` |
| Some data transmitted (for example, the request metadata has been written to the TCP connection) before the connection breaks | `GRPC_STATUS_UNAVAILABLE`       |

#### Protocol errors

| Case                                                         | Status code                      |
| ------------------------------------------------------------ | -------------------------------- |
| Could not decompress but compression algorithm supported     | `GRPC_STATUS_INTERNAL`           |
| Compression mechanism used by client not supported by the server | `GRPC_STATUS_UNIMPLEMENTED`      |
| Flow-control resource limits reached                         | `GRPC_STATUS_RESOURCE_EXHAUSTED` |
| Flow-control protocol violation                              | `GRPC_STATUS_INTERNAL`           |
| Error parsing returned status                                | `GRPC_STATUS_UNKNOWN`            |
| Unauthenticated: credentials failed to get metadata          | `GRPC_STATUS_UNAUTHENTICATED`    |
| Invalid host set in authority metadata                       | `GRPC_STATUS_UNAUTHENTICATED`    |
| Error parsing response protocol buffer                       | `GRPC_STATUS_INTERNAL`           |
| Error parsing request protocol buffer                        | `GRPC_STATUS_INTERNAL`           |

### Sample code

For sample code illustrating how to handle various gRPC errors, see the [grpc-errors](https://github.com/avinassh/grpc-errors) repo.

## 6.4 Performance Best Practices

> A user guide of both general and language-specific best practices to improve performance.

### General

- Always **re-use stubs and channels** when possible.

- **Use keepalive pings** to keep HTTP/2 connections alive during periods of inactivity to allow initial RPCs to be made quickly without a delay (i.e. C++ channel arg GRPC_ARG_KEEPALIVE_TIME_MS).

- **Use streaming RPCs** when handling a long-lived logical flow of data from the client-to-server, server-to-client, or in both directions. Streams can avoid continuous RPC initiation, which includes connection load balancing at the client-side, starting a new HTTP/2 request at the transport layer, and invoking a user-defined method handler on the server side.

  Streams, however, cannot be load balanced once they have started and can be hard to debug for stream failures. They also might increase performance at a small scale but can reduce scalability due to load balancing and complexity, so they should only be used when they provide substantial performance or simplicity benefit to application logic. Use streams to optimize the application, not gRPC.

  ***Side note:*** *This does not apply to Python (see Python section for details).*

- *(Special topic)* <u>Each gRPC channel uses 0 or more HTTP/2 connections and each connection usually has a limit on the number of concurrent streams</u>. When the number of active RPCs on the connection reaches this limit, additional RPCs are queued in the client and must wait for active RPCs to finish before they are sent. <u>Applications with high load or long-lived streaming RPCs might see performance issues because of this queueing</u>. There are two possible solutions:

  1. **Create a separate channel for each area of high load** in the application.
  2. **Use a pool of gRPC channels** to distribute RPCs over multiple connections (channels must have different channel args to prevent re-use so define a use-specific channel arg such as channel number).

  ***Side note:*** *The gRPC team has plans to add a feature to fix these performance issues (see [grpc/grpc#21386](https://github.com/grpc/grpc/issues/21386) for more info), so any solution involving creating multiple channels is a temporary workaround that should eventually not be needed.*

### Java

- **Use non-blocking stubs** to parallelize RPCs.
- **Provide a custom executor that limits the number of threads, based on your workload** (cached (default), fixed, forkjoin, etc).

