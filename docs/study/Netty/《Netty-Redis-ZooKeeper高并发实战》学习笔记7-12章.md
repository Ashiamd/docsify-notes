# 《Netty、Redis、ZooKeeper高并发实战》学习笔记

> 最近一段时间学习IM编程，但是之前急于需要成品，所以学得不是很懂，最后也没能弄好，所以打算系统地学习一下，然后再继续IM编程。

## 第7章 Decoder与Encoder重要组件

​	大家知道，**Netty从底层Java通道读到ByteBuf二进制数据**，传入Netty通道的流水线，随后开始入站处理。

​	在入站处理过程中，需要将ByteBuf二进制类型，解码成Java POJO对象。这个解码过程，可以通过Netty的Decoder解码器去完成。

​	在出站处理过程中，业务处理后的结果（出站数据），需要从某个Java POJO对象，编码为最终的ByteBuf二进制数据，然后通过底层Java通道发送到对端。在编码过程中，需要用到Netty的Encoder编码器去完成数据的编码工作。

​	本章专门为大家解读，什么是Netty的编码器和解码器。

### 7.1 Decoder原理与实践

​	什么叫Netty的解码器呢？

​	首先，<u>它是一个InBound入站处理器，解码器负责处理”入站数据“</u>。

​	其次，它能将上一站Inbound入站处理器传过来的输入(Input)数据，进行数据的解码或者格式转化，然后输出（Output）到下一站Inbound入站处理器。

​	一个标准的解码器将输入类型为ByteBuf缓冲区的数据进行解码，输出一个一个的Java POJO对象。Netty内置了这个解码器，叫作ByteToMessageDecoder，位于Netty的io.netty.handler.coder包中。

​	强调一下，**所有的Netty中的解码器，都是Inbound入站处理器类型，都直接或者间接地实现了ChannelInboundHandler接口**。

#### 7.1.1 ByteToMessageDecoder解码器

​	ByteToMessageDecoder是一个非常重要的解码器类，它是一个<u>抽象类</u>，实现了解码的基础逻辑和流程。ByteToMessageDecoder继承自ChannelInboundHandlerAdapter适配器，是一个入站处理器，**实现了从ByteBuf到Java POJO对象的解码功能**。

​	ByteToMessageDecoder解码的流程，具体可以描述为：首先，它将上一站传过来的输入到ByteBuf中的数据进行解码，解码出一个List\<Object\>对象列表；然后，迭代List\<Object\>列表，逐个将Java POJO对象传入下一站Inbound入站处理器。

​	查看Netty源代码，我们会惊奇地发现：**ByteToMessageDecoder类，并不能完成ByteBuf字节码到具体Java类型的解码，还得依赖于它的具体实现**。

​	ByteToMessageDecoder的解码方法名为decode。通过源代码我们可以发现，decode方法只是提供给了一个抽象方法，也就是说，<u>decode方法的具体解码过程，ByteToMessageDecoder 没有具体的实现</u>。换句话说，如何将ByteBuf数据编程Object数据，需要子类去完成，父类不管。

​	总之，作为解码器的父类，ByteToMessageDecoder仅仅提供了一个流程性质的框架：它仅仅将子类的decode方法解码之后的Object结果，放入自己内部的结果列表List\<Object\>中，最终，父类会负责将List\<Object\>中的元素，一个一个地传递给下一个站。<u>将子类的Object结果放入父类的List\<Object\>列表，也是交由子类的decode方法完成的</u>。

​	如果要实现一个自己的解码器，首先继承ByteToMessageDecoder抽象类。然后，实现其基类的decode抽象方法。将解码的逻辑，写入此方法。总体来说，如果要实现一个自己的ByteBuf解码器，流程大致如下：

（1）首先继承ByteToMessageDecoder抽象类。

（2）然后实现其基类的decode抽象方法，将ByteBuf到POJO解码的逻辑写入此方法。将ByteBuf二进制数据，解码成一个一个的Java POJO对象。

（3）**在子类 的decode方法中，需要将解码后的Java POJO对象，放入decode的List\<Object\>实参中。**这个实参是ByteToMessageDecoder父类传入的，也就是父类的结果收集列表。

​	<u>在流水线的过程中，ByteToMessageDecoder调用子类decode方法解码完成后，会将List\<Object\>中的结果，一个一个地分开传递到下一站的Inbound入站处理器</u>。

#### 7.1.2 自定义Byte2IntegerDecoder整数解码器的实践案例

​	下面是一个小小的实践案例：整数解码器。

​	其功能是，将ByteBuf缓冲区中的字节，解码成Integer整数类型。按照前面的流程，大致的步骤为：

​	（1）定义一个新的整数解码器——Byte2IntegerDecoder类，让这个类继承NettyByteToMessageDecoder字节码解码抽象类。

​	（2）实现父类的decode方法，将ByteBuf缓冲区数据，解码成一个一个的Integer对象。

​	（3）在decode方法中，将解码后得到的Integer整数，加入到父类传递过来的List\<Object\>实参中。

​	Byte2IntegerDecoder整数解码器：

```java
public class Byte2IntegerDecoder extends ByteToMessageDecoder {
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in,
                       List<Object> out) {
        while (in.readableBytes() >= 4) {
            int i = in.readInt();
            Logger.info("解码出一个整数: " + i);
            out.add(i);
        }
    }
}
```

​	上面实践案例程序的decode方法中的逻辑大致如下:

​	首先，Byte2IntegerDecoder解码器继承自ByteToMessageDecode；

​	其次，在其实现的decode方法中，通过ByteBuf的readInt()实例方法，从in缓冲区读取到整数，将二进制数据解码成一个一个的整数。

​	再次，将解码后的整数增加到基类所传入的List\<Object\>列表参数中。

​	最后，它不断地循环，不断地解码，并且不断地添加到List\<Object\>结果列表中。

​	前面反复讲到，decode方法处理完成后，基类会继续后面的传递处理：将List\<Object\>结果列表中所得到的整数，一个一个地传递给下一个Inbound入站处理器。

​	至此，一个简单的解码器就已经完成了。

​	如何使用这个自定义的Byte2IntegerDecoder解码器呢？

​	首先，需要将其加入到通道的流水线中。其次，由于解码器的功能仅仅是完成ByteBuf的解码，不做其他的业务处理，所以还需要编写一个业务处理器，用于在读取解码后的Java POJO对象后，完成具体的业务处理。

​	这里编写一个简单的业务处理器IntegerProcessHandler ，用于处理Byte2IntegerDecoder解码之后的Java Integer整数。其功能是：读取上一站的入站数据，把它转换成整数，并且输出到Console控制台上。实践案例的代码如下：

```java
public class IntegerProcessHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Integer integer = (Integer) msg;
        Logger.info("打印出一个整数: " + integer);
    }
}
```

​	至此，已经编写了Byte2IntegerDecoder和IntegerProcessHandler这两个自己的入站处理器：一个负责解码，另一个负责业务处理。

​	最终，如何测试这两个入站处理器？使用EmbeddedChannel嵌入式通道(6.4.2)，编写一个测试实例，完整的代码如下：

```java
public class Byte2IntegerDecoderTester {
    /**
     * 整数解码器的使用实例
     */
    @Test
    public void testByteToIntegerDecoder() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new Byte2IntegerDecoder());
                ch.pipeline().addLast(new IntegerProcessHandler());
            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);

        for (int j = 0; j < 100; j++) {
            ByteBuf buf = Unpooled.buffer();
            buf.writeInt(j);
            channel.writeInbound(buf);
        }
        
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

​	在测试用例中，新建了一个EmbeddedChannel嵌入式通道实例，将两个自己的入站处理器Byte2IntegerDecoder和IntegerProcessHandler加入通信的流水线上。

​	请注意先后次序：Byte2IntegerDecoder解码器在前、IntegerProcessHandler整数处理器在后。因为入站处理次序为——从前到后。

​	为了测试入站处理器，需要确保通道能接收到ByteBuf入站数据。这里调用writeInbound方法，模拟入站数据的写入，向嵌入式通道EmbeddedChannel

写入100次ByteBuf入站缓冲。每一次写入仅仅包含一个整数。

​	EmbeddedChannel的writeInbound方法模拟入站数据，会被流水线上的两个入站处理器所接受和处理，接着，这些入站的二进制字节被解码成一个一个的整数，然后逐个地输出到控制台上。

​	部分输出结果如下：

```java
// ... 省略部分输出
解码出一个整数：0
打印出一个整数：0
    解码出一个整数：1
打印出一个整数：1
    解码出一个整数：2
打印出一个整数：2
        解码出一个整数：3
打印出一个整数：3
```

​	强烈建议大家参考以上代码，自行实践一下以上解码器，并仔细分析一下运行的结果。

​	还可以仿照这个例子，实现除了整数解码器之外，Java基本数据类型的解码器：Short、Char、Long、Float、Double等。

​	最后说明下：**ByteToMessageDecoder传递给下一站的是解码之后的Java POJO对象，不是ByteBuf缓冲区**。问题来了，ByteBuf缓冲区由谁负责进行引用计数和释放管理的呢？

​	其实，**<u>基类ByteToMessageDecoder负责解码器的ByteBuf缓冲区的释放工作，它会调用ReferenceCountUtil.release(in)方法，将之前的ByteBuf缓冲区的引用数减1.这个工作是自动完成的。</u>**

​	也会有同学问：如果这个ByteBuf被释放了，在后面还需要用到，怎么办？可以在decode方法中调用一次ReferenceCountUtil.retain(in)来增加一次引用计数