# 《bytebuddy核心教程》-学习笔记

> 视频教程: [bytebuddy核心教程_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1G24y1a7bd)
> ByteBuddy github项目: [raphw/byte-buddy: Runtime code generation for the Java virtual machine. (github.com)](https://github.com/raphw/byte-buddy)
>
> ByteBuddy官方教程: [Byte Buddy - runtime code generation for the Java virtual machine](http://bytebuddy.net/#/tutorial-cn)
>
> ---
>
> 下面的图片全部来自网络博客/文章

# 一、简介

> [raphw/byte-buddy: Runtime code generation for the Java virtual machine. (github.com)](https://github.com/raphw/byte-buddy)

​	ByteBuddy是基于[ASM (ow2.io)](https://asm.ow2.io/)实现的字节码操作类库。比起ASM，ByteBuddy的API更加简单易用。开发者无需了解[class file format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)知识，也可通过ByteBuddy完成字节码编辑。

+ ByteBuddy使用java5实现，并且支持生成JDK6及以上版本的字节码(由于jdk6和jdk7使用未加密的HTTP类库, 作者建议至少使用jdk8版本)

+ 和其他字节码操作类库一样，ByteBuddy支持生成类和修改现存类
+ 与与静态编译器类似，需要在快速生成代码和生成快速的代码之间作出平衡，ByteBuddy主要关注以最少的运行时间生成代码

> [Byte Buddy - runtime code generation for the Java virtual machine](http://bytebuddy.net/#/tutorial-cn)

| JIT优化后的平均ns纳秒耗时(标准差) | 基线          | Byte Buddy                             | cglib              | Javassist          | Java proxy         |
| --------------------------------- | ------------- | -------------------------------------- | ------------------ | ------------------ | ------------------ |
| 普通类创建                        | 0.003 (0.001) | 142.772 (1.390)                        | 515.174 (26.753)   | 193.733 (4.430)    | 70.712 (0.645)     |
| 接口实现                          | 0.004 (0.001) | 1'126.364 (10.328)                     | 960.527 (11.788)   | 1'070.766 (59.865) | 1'060.766 (12.231) |
| stub方法调用                      | 0.002 (0.001) | 0.002 (0.001)                          | 0.003 (0.001)      | 0.011 (0.001)      | 0.008 (0.001)      |
| 类扩展                            | 0.004 (0.001) | 885.983 *5'408.329* (7.901) *(52.437)* | 1'632.730 (52.737) | 683.478 (6.735)    | –                  |
| super method invocation           | 0.004 (0.001) | 0.004 *0.004* (0.001) *(0.001)*        | 0.021 (0.001)      | 0.025 (0.001)      | –                  |

​	上表通过一些测试，对比各种场景下，不同字节码生成的耗时。对比其他同类字节码生成类库，Byte Buddy在生成字节码方面整体耗时还是可观的，并且生成后的字节码运行时耗时和基线十分相近。

+ [Java 代理](http://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Proxy.html)

  Java 类库自带的一个代理工具包，它允许创建实现了一组给定接口的类。这个内置的代理很方便，但是受到的限制非常多。 例如，上面提到的安全框架不能以这种方式实现，因为我们想要扩展类而不是接口。

+ [cglib](https://github.com/cglib/cglib)

  该*代码生成库*是在 Java 开始的最初几年实现的，不幸的是，它没有跟上 Java 平台的发展。尽管如此，cglib仍然是一个相当强大的库， 但它是否积极发展变得很模糊。出于这个原因，许多用户已不再使用它。

  (cglib目前已不再维护，并且github中也推荐开发者转向使用Byte Buddy)

+ [ Javassist](https://github.com/jboss-javassist/javassist)

  该库带有一个编译器，该编译器采用包含 Java 源码的字符串，这些字符串在应用程序运行时被翻译成 Java 字节码。 这是非常雄心勃勃的，原则上是一个好主意，因为 Java 源代码显然是描述 Java 类的非常的好方法。但是， Javassist 编译器在功能上无法与 javac 编译器相比，并且在动态组合字符串以实现更复杂的逻辑时容易出错。此外， Javassist 带有一个代理库，它类似于 Java 的代理程序，但允许扩展类并且不限于接口。然而， Javassist 代理工具的范围在其API和功能方面同样受限限制。

  (2023-11-26看javassist在github上一次更新在一年前，而ByteBuddy在3天前还有更新)

