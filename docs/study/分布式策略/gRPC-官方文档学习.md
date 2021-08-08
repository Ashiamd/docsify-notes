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

# 5. Platforms

# 6. Guides

