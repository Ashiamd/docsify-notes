# 《Netty、Redis、ZooKeeper高并发实战》学习笔记

> 对亏了系统的学习和各种社区的帖子帮助，总算完成了手机App和服务器的IM通讯了

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

​	<u>在流水线的过程中，ByteToMessageDecoder调用子类decode方法解码完成后，会将List\<Object\>中的结果，**一个一个地分开传递到下一站**的Inbound入站处理器</u>。

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

​	也会有同学问：如果这个ByteBuf被释放了，在后面还需要用到，怎么办？可以在decode方法中调用一次ReferenceCountUtil.retain(in)来增加一次引用计数。

#### 7.1.3 ReplayingDecoder解码器

​	使用上面的Byte2IntegerDecoder整数解码器会面临一个问题：需要对ByteBuf的长度进行检查，如果由足够的字节，才能进行整数的读取。这种长度的判断，是否可以由Netty帮忙来完成呢？答案是：使用Netty的ReplayingDecoder类可以省去长度的判断。

​	ReplayingDecoder类是ByteToMessageDecoder的子类。其作用是：

+ **在读取ByteBuf缓冲区的数据之前，需要检查缓冲区是否有足够的字节。**
+ **若ByteBuf中有足够的字节，则会正常读取；反之，如果没有足够的字节，则会停止解码。**

​	使用ReplayingDecoder基类，编写整数解码器，则可以不用进行长度检测。

​	改写上一个的整数解码器，继承ReplayingDecoder类，创建一个新的整数解码器，类名为Byte2IntegerReplayDecoder，代码如下：

```java
public class Byte2IntegerReplayDecoder extends ReplayingDecoder {
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        int i = in.readInt();
        Logger.info("解码出一个整数: " + i);
        out.add(i);
    }
}
```

​	通过这个实例程序，我们可以看到：继承ReplayingDecoder类实现一个解码器，就不用编写长度判断的代码。ReplayingDecoder进行长度判断的原理，其实很简单：它的内部定义了一个新的二进制缓冲区类，对ByteBuf缓冲区进行了装饰，这个类名为ReplayingDecoderBuffer。**该装饰器的特点是：在缓冲区真正读数据之前，首先进行长度的判断：如果长度合格，则读取数据；否则抛出ReplayError。ReplayingDecoder捕获到ReplayError后，会留着数据，<u>等待下一次IO事件到来时再读取</u>。**

​	简单来讲，ReplayingDecoder基类的关键技术就是偷梁换柱，在将外部传入的ByteBuf缓冲区传给子类之前，换成了自己装饰过的ReplayingDecoderBuffer缓冲区。也就是说，<u>在示例程序中，Byte2IntegerReplayDecoder中的decode方法所得到的实参in的值，它的直接类型并不是原始的ByteBuf类型，而是ReplayingDecoderBuffer类型</u>。

​	*ps：看了一下源码，ReplayingDecoderByteBuf extends ByteBuf，内部含成员变量private ByteBuf buffer;*

​	ReplayingDecoderBuffer类型，首先是一个内部的类，其次它继承了ByteBuf类型，包装了ByteBuf类型的大部分读取方法。ReplayingDecoderBuffer类型的读取方法与ByteBuf类型的读取方法相比，做了什么样的功能增强呢？主要是进行二进制数据长度的判断，如果长度不足，则抛出异常。这个异常会反过来被ReplayingDecoder基类所捕获，将解码工作停掉。

​	ReplayingDecoder的作用，远远不止进行长度判断，它更重要的作用是用于<u>分包传输</u>的应用场景。

#### 7.1.4 整数的分包解码器的实践案例

​	前面讲到，**底层通信协议是分包传输的，一份数据可能分几次达到对端**。发送端出去的包在传输过程中会进行多次的拆分和组装。接收端所收到的包和发送端所发送的包不是一模一样的。例如，在发送端发送4个字符串，Netty NIO接收端可能只是接收到了个ByteBuf数据缓冲。

​	对端出站数据：ABCD、EFGH、IJKL、MN

​	ByteBuf解析前(可能出现该情况)：ABC、DEFGHI、JKLMN

​	ByteBuf解析后：ABCD、EFGH、IJKL、MN

​	*ps：典型的粘包、拆包/半包问题。*

​	<u>在Java OIO流式传输中，不会出现这样的问题，因为它的策略是：不读到完整的信息，就一直阻塞程序，不往后执行。</u>但是，在Java的NIO中，由于NIO的非阻塞性，就会出现以上的问题。

​	我们知道，Netty接收到的数据都可以通过解码器进行解码。Netty可以使用ReplayingDecoder解决上述问题。前面讲到，<u>在进行数据解析时，如果发现当前ByteBuf中所有可读的数据不够，ReplayingDecoder会结束解析，直到可读数据是足够的</u>。这一切都是在ReplayingDecoder内部进行，它是通过和缓冲区装饰器类ReplayingDecoderBuffer相互配合完成的，根本就不需要用户程序来操心。

​	一上来就实现对字符串的解码和纠正，相对比较复杂些。在此之前，作为铺垫，我们先看一个简单点的例子：解码整数序列，并且将它们两两一组进行相加。

​	要完成以上的例子，需要用到ReplayingDecoder一个很重要的属性——**state成员属性。该成员属性的作用就是保存当前解码器在解码过程中的当前阶段**。在Netty源代码中，该属性的定义具体如下：

```java
public abstract class ReplayingDecoder<S> extends ByteToMessageDecoder {
	// ... 省略不相干的代码
    
    // 重要的成员属性，表示阶段，类型为泛型，默认为Object
    private S state;
    
    // 默认的构造器
    protected ReplayingDecoder() {
        this((Object)null);
    }
    
    // 重载的构造器
    protected ReplayingDecoder(S initialState) {
        // 初始化内部的ByteBuf缓冲装饰器类
        this.replayable = new ReplayingDecoderByteBuf();
        
        // 读断点指针，默认为-1
        this.checkpoint = -1;
        
        // 状态state的默认值为null
        this.state = initialState;
    }
    
    // ...省略不相干的方法
}
```

​	上一小节，在定义的整数解码实例中，使用的是默认的无参构造器，也就是说，**state的值为null，泛型实参类型，为默认的Object**。总之，就是没有用到state属性。

​	这一小节，就需要用到state成员属性了。为什么呢？整个解码工作通过一次解码不能完成。要完成两个整数相加就需要解码两次，每一次解码只能解码出一个整数。只有两个整数都得到之后，然后求和，整个解码的工作才算完成。

​	下面，先基于ReplayingDecoder基础解码器，编写一个整数相加的解码器：解码两个整数，并把这两个数据相加之和作为解码的结果。代码如下：

```java
public class IntegerAddDecoder
        extends ReplayingDecoder<IntegerAddDecoder.Status> {

    enum Status {
        PARSE_1, PARSE_2
    }

    private int first;
    private int second;

    public IntegerAddDecoder() {
        //构造函数中，需要初始化父类的state 属性，表示当前阶段
        super(Status.PARSE_1);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in,
                          List<Object> out) throws Exception {


        switch (state()) {
            case PARSE_1:
                //从装饰器ByteBuf 中读取数据
                first = in.readInt();
                //第一步解析成功，
                // 进入第二步，并且设置“读指针断点”为当前的读取位置
                checkpoint(Status.PARSE_2);
                break;
            case PARSE_2:
                second = in.readInt();
                Integer sum = first + second;
                out.add(sum);
                checkpoint(Status.PARSE_1);
                break;
            default:
                break;
        }
    }
}
```

*ps:checkponit的部分代码如下：*

```java
public abstract class ReplayingDecoder<S> extends ByteToMessageDecoder {
    // ... 
    protected void checkpoint() {
        this.checkpoint = this.internalBuffer().readerIndex();
    }

    protected void checkpoint(S state) {
        this.checkpoint();
        this.state(state);
    }
}
```

​	IntegerAddDecoder类继承了ReplayingDecoder\<IntegerAddDecoder.Status\>，后面的泛型实参为自定义的状态类型，是一个<u>enum枚举类型</u>，这里的读取有两个阶段：

（1）第一个阶段读取前面的整数。

（2）第二个阶段读取后面的整数，然后相加。

​	状态的值保存在父类的成员变量state中。前文说明过：该成员变量保存当前解码的阶段，需要在构造函数中进行初始化。在上面的子类构造函数中，调用了super(Status.PARSE_1)对state进行了初始化。

​	在IntegerAddDecoder类中，每一次decode方法中的解码，有两个阶段：

（1）第一个阶段，解码出前一个整数。

（2）第二个阶段，解码出后一个整数，然后求和。

​	每一个阶段一完成，就通过checkpoint(Status)方法，把当前的状态设置为新的Status值，这个值保存在ReplayingDecoder的state属性中。

​	严格来说，checkpoint(Status)方法有两个作用：

（1）设置state属性的值，更新一下当前的状态。

（2）设置“读断点指针”。

​	**“读断点指针”是ReplayingDecoder类的另一个重要的成员，它保存着装饰器内部ReplayingDecoderBuffer成员的起始读指针，类似于Java NIO中Buffer的mark标记。**<u>当读数据时，一旦可读数据不够，ReplayingDecoderBuffer在抛出ReplayErorr异常之前，ReplayingDecoder会把读指针的值还原到之前的checkpoint（IntegerAddDecoder.Status）方法设置的”读断点指针“（checkpoint）</u>。于是乎，在ReplayingDecoder下一次读取时，还会从之前设置的断点位置开始。

​	checkpoint(IntegerAddDecoder.Status)方法，仅仅从参数上看比较奇怪，参数为要设置的阶段，但是它的功能却又包含了”读断点指针“的设置。

​	总结一下，在这个IntegerAddDecoder的使用实例中，解码器保持了以下状态信息：

（1）当前通道的读取阶段，是Status.PARSE_1或者Status.PARSE_2。

（2）<u>每一次读取，还要保持当前”读断点指针“，便于在可读数据不足时进行恢复</u>。

​	因此，IntegerAddDecoder时**有状态的，不能在不同的通道之间进行共享**。更加进一步说，**ReplayingDecoder类型和其所有的子类都需要保存状态信息，都有状态的，都不适合在不同的通道之间共享**。

#### 7.1.5 字符串的分包解码器的实践案例

​	到目前为止，通过前面的整数分包传输，对ReplayingDecoder的分阶段解码，大家应该有了一个完整的了解。现在来看一下字符串的分包传输。在原理上，字符串分包解码和整数分包解码是一样的。有所不同的是：整数的长度是固定的，目前在Java中是4个字节；而字符串的长度不是固定的，是可变长度的，这就是一个小小的难题。

​	获取字符串的长度信息和程序所使用的具体传输协议是强相关的。一般来说，在Netty中进行字符串的传输，可以采用普通的Header-Content内容传输协议：

（1）在协议的Head部分放置字符串的字节长度。Head部分可以用一个整性int来描述即可。

（2）在协议的Content部分，放置的是字符串的字节数组。

​	后面会专门介绍，<u>在实际的传输过程中，一个Header-Content内容包，在发送端会被编码成为一个ByteBuf内容发送包，当到达接收端后，可能被分成很多ByteBuf接收包</u>。对于这些参差不齐的接收包，该如何解码成为最初的ByteBuf内容发送包，来获得Header-Content内容分包呢？

​	不用急，采用ReplayingDecoder解码器即可。下面就是基于ReplayingDecoder实现自定义的字符串分包解码器的示例程序，它的代码如下：

```java
public class StringReplayDecoder
        extends ReplayingDecoder<StringReplayDecoder.Status> {

    enum Status {
        PARSE_1, PARSE_2
    }

    private int length;
    private byte[] inBytes;

    public StringReplayDecoder() {
        //构造函数中，需要初始化父类的state 属性，表示当前阶段
        super(Status.PARSE_1);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in,
                          List<Object> out) throws Exception {
        switch (state()) {
            case PARSE_1:
                //第一步，从装饰器ByteBuf 中读取长度
                length = in.readInt();
                inBytes = new byte[length];
                // 进入第二步，读取内容
                // 并且设置“读指针断点”为当前的读取位置
                checkpoint(Status.PARSE_2);
                break;
            case PARSE_2:
                //第二步，从装饰器ByteBuf 中读取内容数组
                in.readBytes(inBytes, 0, length);
                out.add(new String(inBytes, "UTF-8"));
                // 第二步解析成功，
                // 进入第一步，读取下一个字符串的长度
                // 并且设置“读指针断点”为当前的读取位置
                checkpoint(Status.PARSE_1);
                break;
            default:
                break;
        }
    }
}
```

​	在StringReplayDecoder类中，每一次decode方法的解码分为两个步骤：
​	第1步骤，解码出一个字符串的长度。

​	第2步骤，按照第一个阶段的字符串长度解码出字符串的内容。

​	在decode方法中，每个阶段一完成，就通过checkpoint(Status)方法把当前的状态设置为新的Status值。

​	为了处理StringReplayDecoder解码后的字符串，这里编写一个简单的业务处理器，其功能是：读取上一站的入站数据，把它转换成字符串，并且输出到Console控制台上。新业务处理器名称为StringProcessHandler。

```java
public class StringProcessHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        String s = (String) msg;
        System.out.println("打印: " + s);
    }
}
```

​	至此，已经编写了StringReplyDecoder和StringProcessHandler两个自己的入站处理器：一个负责字符串编码，另外一个负责字符串输出。

​	测试用例代码如下：

```java
public class StringReplayDecoderTester {
    static String content = "疯狂创客圈：高性能学习社群!";

    /**
     * 字符串解码器的使用实例
     */
    @Test
    public void testStringReplayDecoder() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new StringReplayDecoder());
                ch.pipeline().addLast(new StringProcessHandler());
            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);
        byte[] bytes = content.getBytes(Charset.forName("utf-8"));
        for (int j = 0; j < 100; j++) {
            //1-3之间的随机数
            int random = RandomUtil.randInMod(3);
            ByteBuf buf = Unpooled.buffer();
            buf.writeInt(bytes.length * random);
            for (int k = 0; k < random; k++) {
                buf.writeBytes(bytes);
            }
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

部分输出结果：

```java
打印: 疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!
打印: 疯狂创客圈：高性能学习社群!
打印: 疯狂创客圈：高性能学习社群!
打印: 疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!
打印: 疯狂创客圈：高性能学习社群!
打印: 疯狂创客圈：高性能学习社群!
打印: 疯狂创客圈：高性能学习社群!
// ...
```

​	在测试样例中，新建了一个EmbeddedChannel嵌入式通道实例，将两个自己的入站处理器加入通道流水线中。为了测试入站处理器，调用writeInbound方法，向EmbeddedChannel嵌入式通道写入100次ByteBuf入站缓冲；每一个ByteBuf缓冲仅仅包含一个字符串。EmbeddedChannel嵌入式通道接收到入站数据后，流水线上的两个处理器就能不断地处理这些入站数据：将接到地二进制字节解码成一个一个的字符串，然后逐个输出到控制台上。

​	小结：通过ReplayingDecoder解码器，可以正确地解码分包后地ByteBuf数据包。但是，在**实际的开发中，不太建议继承这个类**，原因是：

​	（1）不是所有的ByteBuf操作都将被ReplayingDecoderBuffer装饰类所支持，**可能有些ByteBuf操作在ReplayingDecoder子类的decode实现方法中被使用时就会抛出ReplayError异常**。

​	（2）在数据解析逻辑复杂的应用场景，ReplayingDecoder在**解析速度上相对较差**。

​	原因是什么呢？<u>在ByteBuf中长度不够时，ReplayingDecoder会捕获一个ReplayError异常，这时会把ByteBuf中的读指针还原到之前的读断点指针(checkpoint)，然后结束这次解析操作，等待下一次IO读事件</u>。在网络条件比较糟糕时，一个数据包的解析逻辑会被反复执行多次，如果解析过程是一个消耗CPU的操作，那么对CPU是个大负担。

​	所以，**ReplayingDecoder更多的是应用于数据解析逻辑简单的场景**。在数据解析复杂的应用场景，建议使用在前文介绍的解码器ByteToMessageDecoder或者其子类（后文介绍），它们更加合适。

​	继承ByteToMessageDecoder基类，实现Header-Content协议传输分包的字符串内容解码器，代码如下：

```java
public class StringHeaderDecoder extends ByteToMessageDecoder {    //头是一个整数
    @Override
    protected void decode(ChannelHandlerContext channelHandlerContext,
                          ByteBuf buf,
                          List<Object> out) throws Exception {

        //可读字节小于int的大小4字节，消息头还没读满，return
        if (buf.readableBytes() < 4) {
            return;
        }

        //消息头已经完整
        //在真正开始从buffer读取数据之前，调用markReaderIndex()设置回滚点
        // 回滚点为消息头header的readIndex位置
        buf.markReaderIndex();
        int length = buf.readInt();
        //从buffer中读出头的大小，这会使得readIndex读指针前移
        //剩余长度不够body消息体，reset 读指针
        if (buf.readableBytes() < length) {
            //读指针回滚到header的readIndex位置处，没进行状态的保存
            buf.resetReaderIndex();
            return;
        }
        // 读取数据，编码成字符串
        byte[] inBytes = new byte[length];
        buf.readBytes(inBytes, 0, length);
        out.add(new String(inBytes, "UTF-8"));
    }
}
```

​	在上面的示例中，读取数据之前，需要调用buf.markReaderIndex()记录当前的位置指针，当可读内容不够，也就是buf.readableBytes()  < length时，需要调用buf.resetReaderIndex()方法将读指针回滚到旧的读起始位置。

​	表面上，ByteToMessageDecoder基类时无状态的，它不像ReplayingDecoder，需要使用状体位来保存当前的读取阶段。但是，实际上，ByteToMessageDecoder也是有状态的。

​	**在ByteToMessageDecoder内部，有一个二进制字节的积累器（cumulation），它用来保存没有解析完的二进制内容**。所以，**<u>ByteToMessageDecoder及其子类都是有状态的业务处理器，也就是说，它不能共享，在每次初始化通道的流水线时，都要重新创建一个ByteToMessageDecoder或者它的子类的实例</u>**。

#### 7.1.6 MessageToMessageDecoder解码器

​	前面的解码器都是讲ByteBuf缓冲区中的二进制数据解码成Java的普通POJO对象。有一个问题：是否存在一些解码器，将一种POJO对象解码成另外一种POJO对象呢？答案是：存在的。

​	与前面不同的是，这种应用场景下的Decoder解码器，需要继承一个新的Netty解码器基类：MessageToMessageDecoder\<I\>。在继承它的时候，需要明确的泛型实参\<I\>。这个<u>实参的作用就是指定入站消息Java POJO类型</u>。

​	这里有个问题：为什么MessageToMessageDecoder\<I\>需要指定入站数据的类型。ByteToMessageDecoder的入站消息类型就是ByteBuf类型，MessageToMessageDecoder\<I\>的入站消息类型不明确，所以需要指定。

​	MessageToMessageDecoder也是一个入站处理器，也有一个decode抽象方法。decode的具体解码的逻辑需要子类去实现。

​	下面通过实现一个整数Integer到字符串String的解码器，演示一下MessageToMessageDecoder的使用。此解码器的具体功能是将整数转换成字符串，然后输出到下一站。

```java
public class Integer2StringDecoder extends
        MessageToMessageDecoder<Integer> {
    @Override
    public void decode(ChannelHandlerContext ctx, Integer msg,
                       List<Object> out) throws Exception {
        out.add(String.valueOf(msg));
    }
}
```

​	这里定义的Integer2StringDecoder新类，继承自MessageToMessageDecoder基类。基类泛型实参Integer，明确了入站的数据类型为Integer。在decode方法中，将整数转成字符串，再加入到一个List列表参数中即可。这个List实参是由父类在调用时传递过来的。在子类decode方法处理完成后，父类会将这个List实例的所有元素进行迭代，逐个发送给下一站Inbound入站处理器(7.1.2有介绍过了解码的流程了)。

### 7.2 开箱即用的Netty内置的Decoder

​	Netty提供了不少开箱即用的Decoder解码器。在一般情况下，能满足很多编码应用场景的需求，这为大家省去了开发Decoder时间。下面将几个比较基础的解码器梳理下，大致如下：

（1）**固定长度数据包解码器——FixedLengthFrameDecoder**

​	适用场景：每个接收到的数据包的长度，都是固定的，例如100字节。

​	在这种场景下，只需要把这个解码器加到流水线中，它会把入站ByteBuf数据包拆分成一个个长度为100字节的数据包，然后发送往下一个channelHandler入站处理器。补充说明一下：**这里所指的一个数据包，在Netty中就是一个ByteBuf实例。注：数据帧（Frame），本书也通称为数据包。**

（2）**行分割数据包解码器——LineBasedFrameDecoder**

​	适用场景：每个ByteBuf数据包，使用换行符（或者回车换行符）作为数据包的边界分割符。

​	如果每个接收到的数据包，都已换行符/回车作为分隔，在这种场景下，只需要把这个解码器加到流水线中,Netty会使用换行分隔符，把ByteBuf数据包分割成一个一个完整的应用层ByteBuf数据包，再发送给下一站。

（3）**自定义分隔符数据包解码器——DelimiterBasedFrameDecoder**

​	DelimiterBasedFrameDecoder是LineBasedFrameDecoder按照行分割的通用版本。不同之处在于，这个解码器更加灵活，可以自定义分隔符，而不是局限于换行符。如果使用这个解码器，那么接收到的数据包，末尾必须带上对应的分隔符。

（4）**自定义长度数据包解码器——LengthFieldBasedFrameDecoder**

​	这是一种基于灵活长度的解码器。在ByteBuf数据包中，加了一个长度字段，保存了原始数据的长度。解码时，会按照这个长度进行原始数据包的提取。

​	这种解码器在所有开箱即用解码器是最为复杂的一种，后面会重点介绍

#### 7.2.1 LineBasedFrameDecoder解码器

​	前面字符串分包解码器中，内容是按照Header-Content协议进行传输的。如果不使用Header-Content协议，而是在发送端通过换行符("\n"或者"\r\n")来分割每一次发送的字符串，接收端是否可以正确地解析？答案是肯定的。

​	在Netty中，提供了一个开箱即用的、使用换行符分割字符串的解码器，即LineBasedFrameDecoder，它也是最为基础的Netty内置解码器。其工作原理很简答，依次遍历ByteBuf数据包中的可读字节，判断在二进制字节流中，是否存在换行符“\n”或者"\r\n"的字节码。如果有，就以此位置为结束位置，把从可读索引到结束位置之间的字节作为解码成功后的ByteBuf书包。当然，这个ByteBuf数据包，也就是解码后的那行字符串的二进制字节码。

​	LineBasedFrameDecoder支持配置一个最大长度值，表示一行最大能包含的字节数。如果连续读取到最大长度后，仍然没有发现换行符，就会抛出异常。

```java
public class NettyOpenBoxDecoder {
    // ... 
    static String spliter = "\r\n";
    static String content = "疯狂创客圈：高性能学习社群!";
    
    public void testLineBasedFrameDecoder() {
        try {
            ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
                protected void initChannel(EmbeddedChannel ch) {
                    ch.pipeline().addLast(new LineBasedFrameDecoder(1024));
                    ch.pipeline().addLast(new StringDecoder());
                    ch.pipeline().addLast(new StringProcessHandler());
                }
            };
            EmbeddedChannel channel = new EmbeddedChannel(i);

            for (int j = 0; j < 100; j++) {

                //1-3之间的随机数
                int random = RandomUtil.randInMod(3);
                ByteBuf buf = Unpooled.buffer();
                for (int k = 0; k < random; k++) {
                    buf.writeBytes(content.getBytes("UTF-8"));
                }
                buf.writeBytes(spliter.getBytes("UTF-8"));
                channel.writeInbound(buf);
            }


            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
}
```

​	在这个示例程序中，向通道写入100个入站数据包，每一个入站包都会以"\r\n"回车换行符作为结束。通道的LineBasedFrameDecoder解码器会将“\r\n”作为分隔符，分割出一个一个的入站ByteBuf，然后发哦那个给SttringDecoder，其作用把ByteBuf二进制数据转换为字符串。最后字符串被发送到StringProcessHandler业务处理器，由他负责将字符串展示出来。

#### 7.2.2 DelimiterBasedFrameDecoder解码器

​	DelimiterBasedFrameDecoder解码器不仅可以使用换行符，还可以将其他的特殊字符作为数据包的分隔符，例如制表符"\t"。其源部分代码如下：

```java
public class DelimiterBasedFrameDecoder extends ByteToMessageDecoder {
    private final ByteBuf[] delimiters;	//分隔符
    private final int maxFrameLength;	//解码的数据包的最大长度
    private final boolean stripDelimiter;	//解码后数据包是否去掉分隔符，一般选择是
 	// ... 其他成员变量
    //... 其他方法
}
```

​	DelimiterBasedFrameDecoder解码器的使用方法与LineBasedFrameDecoder是一样的，只是在构造参数上有一点点不同。

```java
public class NettyOpenBoxDecoder {
	// ...
    static String spliter2 = "\t";
    static String content = "疯狂创客圈：高性能学习社群!";
    
    public void testDelimiterBasedFrameDecoder() {
        try {
            final ByteBuf delimiter = Unpooled.copiedBuffer(spliter2.getBytes("UTF-8"));
            ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
                protected void initChannel(EmbeddedChannel ch) {
                    ch.pipeline().addLast(
                            new DelimiterBasedFrameDecoder(1024, true, delimiter));
                    ch.pipeline().addLast(new StringDecoder());
                    ch.pipeline().addLast(new StringProcessHandler());
                }
            };
            EmbeddedChannel channel = new EmbeddedChannel(i);
            for (int j = 0; j < 100; j++) {

                //1-3之间的随机数
                int random = RandomUtil.randInMod(3);
                ByteBuf buf = Unpooled.buffer();
                for (int k = 0; k < random; k++) {
                    buf.writeBytes(content.getBytes("UTF-8"));
                }
                buf.writeBytes(spliter2.getBytes("UTF-8"));
                channel.writeInbound(buf);
            }


            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
}
```

#### 7.2.3 LengthFieldBasedFrameDecoder解码器

​	在Netty的开箱即用解码器中，最为复杂的是自定义长度数据包解码器——LengthFieldBasedFrameDecoder。它的难点在于参数比较多，也比较难以理解。**同时它又比较常用**。

​	LengthFieldBasedFrameDecoder解码器，它的名字可以翻译为“长度字段数据包解码器”。传输内容中的LengthField长度字段的值，是指存放在数据包中要传输内容的字节数。<u>普通的基于Header-Content协议的内容传输，尽量用内置的LengthFieldBasedFrameDecoder来解码</u>。

```java
public class NettyOpenBoxDecoder {
    public static final int VERSION = 100; 
    static String content = "疯狂创客圈：高性能学习社群!";
}

public void testLengthFieldBasedFrameDecoder1() {
    try {

        final LengthFieldBasedFrameDecoder spliter =
            new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4);
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(spliter);
                ch.pipeline().addLast(new StringDecoder(Charset.forName("UTF-8")));
                ch.pipeline().addLast(new StringProcessHandler());
            }
        };
        EmbeddedChannel channel = new EmbeddedChannel(i);

        for (int j = 1; j <= 100; j++) {
            ByteBuf buf = Unpooled.buffer();
            String s = j + "次发送->" + content;
            byte[] bytes = s.getBytes("UTF-8");
            buf.writeInt(bytes.length);
            System.out.println("bytes length = " + bytes.length);
            buf.writeBytes(bytes);
            channel.writeInbound(buf);
        }

        Thread.sleep(Integer.MAX_VALUE);
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (UnsupportedEncodingException e) {
        e.printStackTrace();
    }
}
```

输出结果：

```java
打印: 1次发送->疯狂创客圈：高性能学习社群!
打印: 2次发送->疯狂创客圈：高性能学习社群!
打印: 3次发送->疯狂创客圈：高性能学习社群!
打印: 4次发送->疯狂创客圈：高性能学习社群!
打印: 5次发送->疯狂创客圈：高性能学习社群!
// ...
```

上面示例用到了一个LengthFieldBasedFrameDecoder构造器，部分代码如下

```java
public class LengthFieldBasedFrameDecoder extends ByteToMessageDecoder {
    private final int maxFrameLength;		//发送的数据包最大长度
    private final int lengthFieldOffset;	//长度字段偏移量
    private final int lengthFieldLength;	//长度字段自己占用的字节数
    private final int lengthAdjustment;		//长度字段的偏移量矫正
    private final int initialBytesToStrip;	//丢弃的起始字节数
    // ... 其他成员变量
    // ... 其他方法
}
```

​	在前面的示例程序中，设计5个参数和值，分别解读如下：

（1）maxFrameLength：发送的数据包最大长度。示例程序中改值为1024，表示一个数据包最多可发送1024个字节。

（2）lengthFieldOffset：长度字段偏移量。指的是长度字段位于整个数据包内部的字节数组中的下标值。

（3）lengthFiledLength：长度字段所占的字节数。指的是长度字段是一个int整数，则为4，如果长度字段是一个short整数，则为2。

（4）**lengthAdjustment：长度的矫正值。这个参数最难懂。在传输协议比较复杂的情况下，例如包含了长度字段、协议版本号、魔数等等。那么，解码时，就需要进行长度矫正。长度矫正值的计算公式为：内容字段偏移量 - 长度字段偏移量 - 长度字段的字节数。**下一节会有详细的举例说明。

（5）initialBytesToStrip：丢弃的起始字节数。在有效数据字段Content前面，还有一些其他的字段的字节，作为最终的解析结果，可以丢弃。例如，上面的示例程序中，前面又4个节点的长度字段，它起辅助的作用，最终的结果中不需要这个长度，所以丢弃的字节数为4。

​		上面示例程序中，第1个参数表示数据包最大长度1024；第2个参数表示长度字段的偏移量为0，长度字段放在了最前面，处于数据包起始位置；第3个参数表示长度字段的长度为4个字节；第4个参数0，内容字段偏移量4-长度字段偏移量0-长度字段的字节数4=0；第5个参数表示最终内容Content的字节数组中抛弃最前面4个字节的数据。

​	整个数据包56字节，内容Content字段52字节，长度字段4字节。

#### 7.2.4 多字段Header-Content协议数据帧解析的实践案例

​	Header-Content协议是最为简单的内容传输协议。而在实际使用过程中，则没有那么简单。除了长度和内容，在数据包中还可能包含了其他字段，例如，包含了协议版本号。

​	那么在先前LengthFieldBasedFrameDecoder解码器样例代码，要是需要长度字段4字节、版本字段2字节、内容content字段52字节，则构造函数参数依次为1024，0，4，2，6。

```java
public class NettyOpenBoxDecoder {
    public static final int VERSION = 100;
    static String content = "疯狂创客圈：高性能学习社群！";
    
    public void testLengthFieldBasedFrameDecoder2() {
        try {

            final LengthFieldBasedFrameDecoder spliter =
                    new LengthFieldBasedFrameDecoder(1024, 0, 4, 2, 6);
            ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
                protected void initChannel(EmbeddedChannel ch) {
                    ch.pipeline().addLast(spliter);
                    ch.pipeline().addLast(new StringDecoder(Charset.forName("UTF-8")));
                    ch.pipeline().addLast(new StringProcessHandler());
                }
            };
            EmbeddedChannel channel = new EmbeddedChannel(i);

            for (int j = 1; j <= 100; j++) {
                ByteBuf buf = Unpooled.buffer();
                String s = j + "次发送->" + content;
                byte[] bytes = s.getBytes("UTF-8");
                buf.writeInt(bytes.length);
                buf.writeChar(VERSION);
                buf.writeBytes(bytes);
                channel.writeInbound(buf);
            }

            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
}
```

​	运行结果（与上次没差）

```java
打印: 1次发送->疯狂创客圈：高性能学习社群!
打印: 2次发送->疯狂创客圈：高性能学习社群!
打印: 3次发送->疯狂创客圈：高性能学习社群!
打印: 4次发送->疯狂创客圈：高性能学习社群!
打印: 5次发送->疯狂创客圈：高性能学习社群!
// ...
```

​	如果将协议设计得更复杂一点：在长度字段前面添加两字节的版本字段，在长度字段后面加上4个字节的魔数，用来对数据包做一些安全的认证。协议的数据包(62字节：版本2、长度4、魔数4、content52)，前面构造参数传参改为maxFrameLength:1024，lengthFieldOffset:2，lengthFiledLength:4，lengthAdjustment:4，initialBytesToStrip:10。

```java
public class NextOpenBoxDecoder {
    // ... 
    public void testLengthFieldBasedFrameDecoder3() {
        try {

            final LengthFieldBasedFrameDecoder spliter =
                    new LengthFieldBasedFrameDecoder(1024, 2, 4, 4, 10);
            ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
                protected void initChannel(EmbeddedChannel ch) {
                    ch.pipeline().addLast(spliter);
                    ch.pipeline().addLast(new StringDecoder(Charset.forName("UTF-8")));
                    ch.pipeline().addLast(new StringProcessHandler());
                }
            };
            EmbeddedChannel channel = new EmbeddedChannel(i);

            for (int j = 1; j <= 100; j++) {
                ByteBuf buf = Unpooled.buffer();
                String s = j + "次发送->" + content;
                byte[] bytes = s.getBytes("UTF-8");
                buf.writeChar(VERSION);
                buf.writeInt(bytes.length);
                buf.writeInt(MAGICCODE);
                buf.writeBytes(bytes);
                channel.writeInbound(buf);
            }

            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }
}
```

### 7.3 Encoder原理与实践

​	<u>在Netty业务处理完成后，业务处理的结果往往是某个Java POJO对象，需要编码成最终的ByteBuf二进制类型。通过流水线写入到底层的Java通道</u>。

​	在Netty中，什么是编码器呢？首先，编码器是一个Outbound出战处理器，负责处理“出站”数据；其次，编码器将上一站Outbound出战处理器传过来的输入（Input）数据进行编码或者格式转化，然后传到下一站ChannelOutboundHandler出站处理器。

​	编码器与解码器相呼应，Netty中的编码器负责将“出站”数据的某种Java POJO对象编码成二进制ByteBuf，或者编码成另一种Java POJO对象。

​	编码器是ChannelOutboundHandler出站处理器的实现类。一个编码器将出站对象编码之后，编码后数据将被传递到下一个ChannelOutboundHandler出站处理器，进行后面出站处理。

​	由于**最后只有ByteBuf才能写入到通道中去**，因此可以肯定通道流水线上装配的第一个编码器（**如果不算默认添加的HeadHandler用于释放ByteBuf的话**）一定是把数据编码成了ByteBuf类型。注意，出站的处理顺序是从后向前的。

ps：看了一部分DefaultChannelPipeline（ChannelPipeline的默认实现类）的源码

```java
final class DefaultChannelPipeline implements ChannelPipeline{
    // ... 
    public DefaultChannelPipeline(AbstractChannel channel) {
        if (channel == null) {
            throw new NullPointerException("channel");
        } else {
            this.channel = channel;
            this.tail = new DefaultChannelPipeline.TailContext(this);
            this.head = new DefaultChannelPipeline.HeadContext(this);
            this.head.next = this.tail;
            this.tail.prev = this.head;
        }
    }
    // ...
}
```

#### 7.3.1 MessageToByteEncoder编码器

​	MessageToByteEncoder是一个非常重要的编码器基类，它位于Netty的io.netty.handler.codec包中。MessageToByteEncoder的功能是将一个Java POJO对象编码成一个ByteBuf数据包。它是一个抽象类，仅仅实现了编码的基础流程，在编码过程中，通过调用encode抽象方法来实现。encode抽象方法没有逻辑代码实现，需要子类实现。

​	如果要实现一个自己的编码器，则需要继承自MessageToByteEncoder基类，实现它的encode抽象方法。下面实现一个整数编码器，将Java整数编码成二进制ByteBuf数据包。

```java
public class Integer2ByteEncoder extends MessageToByteEncoder<Integer> {
    @Override
    public void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out)
            throws Exception {
        out.writeInt(msg);
        Logger.info("encoder Integer = " + msg);
    }
}
```

​	泛型实参表示编码之前的类型是Java整数类型。

​	上面的encode方法实现很简单：将入站数据对象msg写入到Out实参即可（基类传入的ByteBuf类型的对象）。编码完成后，基类MessageToByteEncoder会将输出的ByteBuf数据包发送到下一站。

​	测试代码如下：

```java
public class Integer2ByteEncoderTester {
    /**
     * 测试整数编码器
     */
    @Test
    public void testIntegerToByteDecoder() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new Integer2ByteEncoder());
            }
        };

        EmbeddedChannel channel = new EmbeddedChannel(i);

        for (int j = 0; j < 100; j++) {
            channel.write(j);
        }
        channel.flush();

        //取得通道的出站数据帧
        ByteBuf buf = (ByteBuf) channel.readOutbound();
        while (null != buf) {
            System.out.println("o = " + buf.readInt());
            buf = (ByteBuf) channel.readOutbound();
        }
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

}
```

​	输出结果：

```java
encoder Integer = 0
encoder Integer = 1
encoder Integer = 2
encoder Integer = 3
// ...
encoder Integer = 99
o = 0
o = 1
o = 2
// ...
o = 99
```

​	上面实例中，首先将Integer2ByteEncoder加入了嵌入式通道，然后调用write方法向通道写入100个数字。写完之后，调用channel.readOutbound()方法从通道中读取模拟的出站数据包，然后不断循环，打印数据帧包中的数字。

#### 7.3.2 MessageToMessageEncoder编码器

​	上一届的示例把POJO对象编码成ByteBuf二进制对象。这里介绍把POJO转换成另一POJO对象的编码器——MessageToMessageEncoder。在子类的encode方法实现中，完成原POJO类型到目标POJO类型的编码逻辑。在encode实现方法中，编码完成后，将解码后的目标对象加入到encode方法中的List实参列表中即可。

```java
public class String2IntegerEncoder extends MessageToMessageEncoder<String> {
    
    @Override
    protected void encode(ChannelHandlerContext c, String s, List<Object> list) throws Exception {
        char[] array = s.toCharArray();
        for (char a : array) {
            //48 是0的编码，57 是9 的编码
            if (a >= 48 && a <= 57) {
                list.add(new Integer(a));
            }
        }
    }
}
```

​	这里定义的String2IntegerEncoder新类继承自MessageToMessageEncoder基类，并且明确了入站的数据类型为String。在encode方法中，将字符串的数字提取出来之后，放入到list列表，其他的字符直接略过。在子类的encode方法处理完后，基类会对这个List实例的所有元素进行迭代，将List列表的元素**逐个发送**给下一站。

​	测试代码如下：

```java
public class String2IntegerEncoderTester {

    /**
     * 测试字符串到整数编码器
     */
    @Test
    public void testStringToIntergerDecoder() {
        ChannelInitializer i = new ChannelInitializer<EmbeddedChannel>() {
            protected void initChannel(EmbeddedChannel ch) {
                ch.pipeline().addLast(new Integer2ByteEncoder());
                ch.pipeline().addLast(new String2IntegerEncoder());
            }
        };

        EmbeddedChannel channel = new EmbeddedChannel(i);

        for (int j = 0; j < 100; j++) {
            String s = "i am " + j;
            channel.write(s);
        }
        channel.flush();
        ByteBuf buf = (ByteBuf) channel.readOutbound();
        while (null != buf) {
            System.out.println("o = " + buf.readInt());
            buf = (ByteBuf) channel.readOutbound();
        }
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

}
```

输出结果如下：

```java
encoder Integer = 48
encoder Integer = 49
encoder Integer = 50
// ...
encoder Integer = 57
encoder Integer = 56
encoder Integer = 57
encoder Integer = 57
o = 48
o = 49
o = 50
// ...
o = 57
o = 56
o = 57
o = 57
```

​	这里先用String2IntegerEncoder将字符串编码成整数，再用Integer2ByteEncoder将整数编码成ByteBuf数据包。

### 7.4 解码器和编码器的结合

​	前面讲到的编码器和解码器都是分开实现的。例如，通过继承ByteToMessageDecoder基类或者它的子类，完成ByteBuf数据包到POJO的解码工作；通过继承基类MessageToByteEncoder或其子类，完成POJO到ByteBuf数据包的编码工作。由于相反逻辑的编码器和解码器实现，导致加入流水线时需要分两次添加。

​	而具有互相配套逻辑的编码器和解码器如果要放到同一个类，需要用到Netty的新类型——Codec类型。

#### 7.4.1 ByteToMessageCodec编解码器

​	完成POJO到ByteBuf数据包的配套的编码器和解码器的基类，ByteToMessageCodec，是一个抽象类。从功能上，继承它等于继承了ByteToMessageDecoder解码器和MessageToByteEncoder编码器两个基类。

​	其包含的encode和decode抽象方法

+ 编码方法——encode(ChannelHandlerContext, I, ByteBuf)
+ 解码方法——decode(ChannelHandlerContext, ByteBuf, List\<Object\>)

​	下面是一个整数到字节、字节到整数的编解码器

```java
public class Byte2IntegerCodec extends ByteToMessageCodec<Integer> {
    @Override
    public void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out)
            throws Exception {
        out.writeInt(msg);
        System.out.println("write Integer = " + msg);
    }

    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in,
                       List<Object> out) throws Exception {
        if (in.readableBytes() >= 4) {
            int i = in.readInt();
            System.out.println("Decoder i= " + i);
            out.add(i);
        }

    }
}
```

​	将encode和decode放在同一个自定义类中，逻辑更紧密。使用时，加到流水线中只要加入一次。

​	这仅是使用形式上简化，技术上没有增加太多难度，不展开介绍。

#### 7.4.2 CombinedChannelDuplexHandler组合器

​	<u>前面编码器和解码器相结合是通过继承完成的。继承的方式有继承的不足，在于：编码器和解码器的逻辑强制性地放在同一个类中，在只需要编码或者解码单边操作地流水线上，逻辑上不合适</u>。

​	编码器和解码器如果要结合起来，除了继承地方法以外，可以通过组合地方式实现。与继承相比，组合会带来更大的灵活性：编码器和解码器可以捆绑使用，也可以单独使用。

​	Netty提供了一个新的组合器——CombinedChannelDuplexHandler基类。下面演示将前面的IntegerFromByteDecoder整数解码器和它对应的IntegerToByteEncoder整数编码器组合起来。

```java
public class IntegerDuplexHandler extends CombinedChannelDuplexHandler<
        Byte2IntegerDecoder,
        Integer2ByteEncoder>
{
    public IntegerDuplexHandler() {
        super(new Byte2IntegerDecoder(), new Integer2ByteEncoder());
    }
}
```

​	继承CombinedChannelDuplexHandler，不需要像ByteToMessageCodec那样，把编码逻辑和解码逻辑都挤在同一个类中了，还是复用原来的编码器和解码器。

​	总之，使用CombinedChannelDuplexHandler可以保证有了相反逻辑关系的encoder和decoder解码器，既可以结合使用，又可以分开使用，十分方便。

### 7.5 本章小结

​	本章介绍了Netty的编码器和解码器。

​	在Netty中，解码器有ByteToMessageDecoder和MessageToMessageDecoder两大基类。如果要从ByteBuf到POJO解码，则可以继承ByteToMessageDecoder基类；如果要从一种POJO到另一种POJO，则可继承

MessageToMessageDecoder基类。

​	Netty提供了不少开箱即用的Decoder解码器，能满足很多解码的场景需求，几个比较基础的解码器如下：

+ 固定长度数据包解码器——FixedLengthFrameDecoder
+ 行分割数据包解码器——LineBasedFrameDecoder
+ 自定义分隔符数据包解码器——DelimiterBasedFrameDecoder
+ 自定义长度数据包解码器——LengthFieldBasedFrameDecoder

​	在Netty中的编码器有MessageToByteEncoder和MessageToMessageEncoder两大重要的基类。POJO到ByteBuf编码继承前者，POJO到POJO继承后者。

## 第8章 JSON和ProtoBuf序列化

​	我们在开发一些远程过程调用(RPC)的程序时，通常会涉及对象的序列化/反序列化的问题，例如一个“Person”对象从客户端通过TCP的方式发送到服务器端；因为TCP协议（UDP等这种低层协议）只能发送字节流，所以需要应用层将Java POJO对象序列化成字节流，数据接收端再反序列化成Java POJO对象即可。“序列化”一定会涉及编码和格式化（Encoding & Format），目前我们可选择的编码方式有：

+ 使用JSON。将Java POJO对象转换成JSON结构化字符串。基于HTTP协议，在Web应用、移动开发方面等，这是常用的编码方式，因为JSON的可读性较强。但是它的性能稍差。

+ 基于XML。和JSON一样，数据在序列化成字节流之前都转换成字符串。可读性强，性能差，异构系统、OpenAPI类型的应用中常用。
+ 使用Java内置的编码和序列化机制，可移植性强，性能稍差，无法跨平台（语言）
+ 其他开源的序列化/反序列化框架，例如Apache Avro，Apache Thrift，这两个框架和Protobuf相比，性能非常接近，而且设计原理如出一辙；其中Avro 在大数据存储（RPC数据交换，本地存储）时比较常用；Thrift的亮点在于内置了RPC机制，所以在开发一些RPC交互式应用时，客户端和服务器端的开发与部署都非常简单。

​	如何选择序列化/反序列化框架呢？

​	评价一个序列化框架的优缺点，大概从两个方面着手：

**（1）结果数据大小，原则上说，序列化后的数据尺寸越小，传输效率越高。**

**（2）结构复杂度，这会影响序列化/反序列化的效率，结构越复杂，越耗时。**

​	理论上说，<u>对于性能要求不是太高的服务器程序，可以选择**JSON系列**的序列化框架；对于性能要求比较高的服务器程序，则应该选择传输效率更高的**二进制序列化**框架</u>，目前的建议是Protobuf。

> [【转】三种通用应用层协议protobuf、thrift、avro对比](https://blog.csdn.net/baodi_z/article/details/82222652)
>
> 看了下官方github，现在protobuf也支持多种语言了。
>
> [protocolbuffers/protobuf](https://github.com/protocolbuffers/protobuf)

​	Protobuf是一个高性能、易扩展的序列化框架，与它的性能测试有关的数据可以参看官方文档。Protobuf本身非常简单，易于开发，而且集合Netty框架，可以非常便捷地实现一个通信应用程序。反过来，Netty也提供了相应的编解码器，为ProtoBuf解决了有关Socket通信中“半包、粘包”等问题。

​	**无论是使用JSON和Protobuf，还是其他反序列化协议，我们必须保证在数据包的反序列化之前，接收端的ByteBuf二进制包一定是一个完整的应用层二进制包，不能是一个半包或者粘包**。

### 8.1 详解粘包和拆包

​	什么是粘包和半包？先从数据包的发送和接收开始讲起。大家知道，Netty发送和读取数据的“场所”是ByteBuf缓冲区。

​	每一次发送就是向通道写入一个ByteBuf。发送数据时先填好ByteBuf，然后通过通道发送出去。对于接收端，每一次读取是通过Handler业务处理器的入站方法，从通道读到一个ByteBuf。读取数据的方法如下：

```java
public void channelRead(ChannelHandlerContext ctx, Object msg){
    ByteBuf byteBuf = (ByteBuf) msg;
    // ... 省略入站处理
}
```

​	最为理想的情况是：发送端是发送一个ByteBuf缓冲区，接收端就能接收到一个ByteBuf，并且发送端和接收端的ByteBuf内容能一模一样。

​	现实总是那么残酷。或者说，理想很丰满，现实很骨感。在实际的通信过程中，并没有大家预料的那么完美。

​	下面给大家看一个实例，看看实际通信过程中所遇到的诡异情况。

#### 8.1.1 半包问题的实践案例

​	改造一下前面的NettyEchoClient实例，通过循环的方式，向NettyEchoServer回显服务器写入大量的ByteBuf，然后看看实际的服务器响应结果。注意：服务器类不需要改造，直接使用之前的回显服务器即可。

​	改造好的客户端类——叫作NettyDumpSendClient。在客户端建立连接成功之后，使用一个for循环，不断通过通道向服务器端写ByteBuf。一直写到1000次，写入的ByteBuf的内容相同，都是字符串的内容：“疯狂创客圈：高性能学习社群！”。代码如下：

```java
public class NettyDumpSendClient {

    private int serverPort;
    private String serverIp;
    Bootstrap b = new Bootstrap();

    public NettyDumpSendClient(String ip, int port) {
        this.serverPort = port;
        this.serverIp = ip;
    }

    public void runClient() {
        //创建reactor 线程组
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

        try {
            //1 设置reactor 线程组
            b.group(workerLoopGroup);
            //2 设置nio类型的channel
            b.channel(NioSocketChannel.class);
            //3 设置监听端口
            b.remoteAddress(serverIp, serverPort);
            //4 设置通道的参数
            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

            //5 装配子通道流水线
            b.handler(new ChannelInitializer<SocketChannel>() {
                //有连接到达时会创建一个channel
                protected void initChannel(SocketChannel ch) throws Exception {
                    // pipeline管理子通道channel中的Handler
                    // 向子channel流水线添加一个handler处理器
                    ch.pipeline().addLast(NettyEchoClientHandler.INSTANCE);
                }
            });
            ChannelFuture f = b.connect();
            f.addListener((ChannelFuture futureListener) ->
            {
                if (futureListener.isSuccess()) {
                    Logger.info("EchoClient客户端连接成功!");

                } else {
                    Logger.info("EchoClient客户端连接失败!");
                }

            });

            // 阻塞,直到连接完成
            f.sync();
            Channel channel = f.channel();

            //6发送大量的文字
            byte[] bytes = "疯狂创客圈：高性能学习社群!".getBytes(Charset.forName("utf-8"));
            for (int i = 0; i < 1000; i++) {
                //发送ByteBuf
                ByteBuf buffer = channel.alloc().buffer();
                buffer.writeBytes(bytes);
                channel.writeAndFlush(buffer);
            }


            // 7 等待通道关闭的异步任务结束
            // 服务监听通道会一直等待通道关闭的异步任务结束
            ChannelFuture closeFuture =channel.closeFuture();
            closeFuture.sync();

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 优雅关闭EventLoopGroup，
            // 释放掉所有资源包括创建的线程
            workerLoopGroup.shutdownGracefully();
        }

    }

    public static void main(String[] args) throws InterruptedException {
        int port = NettyDemoConfig.SOCKET_SERVER_PORT;
        String ip = NettyDemoConfig.SOCKET_SERVER_IP;
        new NettyDumpSendClient(ip, port).runClient();
    }
}
```

​	运行程序查看结果之前，首先要启动的是前面介绍过的NettyEchoServer回显服务器。然后启动的是新编写的客户端NettyDumpSendClient程序。客户端程序连接成功后，会向服务器发送1000个ByteBuf内容缓冲区，服务器NettyEchoServer收到后，会输出到控制台，然后回写给客户端。

服务器的输出如下（部分）：

```java
// ...
msg type: 直接内存
server received: 疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性
写回前，msg.refCnt:1
写回后，msg.refCnt:0
msg type: 直接内存
server received: 能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!
写回前，msg.refCnt:1
写回后，msg.refCnt:0
// ...
```

客户端的输出如下（部分）：

```java
//...
client received: 疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!
client received: 疯狂创客圈：高性能学习社群!
client received: 疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈�
client received: ��高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈�
client received: ��高性能学习社群!疯狂创客圈：高性能学习社群!
client received: 疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!
client received: 疯狂创客圈：高性能学习社群!疯狂创客圈：高性能学习社群!
client received: 疯狂创客圈：高性能学习社群!
//...
```

​	仔细观察服务端的控制台输出，可以看出存在三种类型的输出：

（1）读到一个完整的客户端输入ByteBuf。

（2）读到多个客户端的ByteBuf输入，但是“粘”在了一起。

（3）读到部分ByteBuf的内容，并且有乱码。

​	再仔细观看客户端的输出。可以看到，客户端和服务器端同样存在以上三种类型的输出。

​	对应第1种情况，这里把接收端接收到的这种完整的ByteBuf称为“全包”。

​	对应第2种情况，多个发送端的输入ByteBuf“粘”在了一起，就把这种读取的ByteBuf称为“粘包”。

​	对应第3种情况，一个输入的ByteBuf被“拆”开读取，读取到一个破碎的包，就把这种读取的ByteBuf称为“半包”。

​	<u>为了简单起见，也可以将“粘包”的情况看成特殊的“半包”。“粘包”和“半包”可以统称为传输的“半包问题”</u>。

#### 8.1.2 什么是半包问题

​	半包问题包含了“粘包”和“半包”两种情况：

**（1）粘包，指接收端（Receiver）收到一个Bytebuf，包含了多个发送端（Sender）的ByteBuf，多个ByteBuf“粘”在了一起。**

**（2）半包，就是接收端将一个发送端ByteBuf“拆”开了，收到多个破碎的包。换句话说，一个接收端收到的ByteBuf是发送端的一个ByteBuf的一部分。**

​	粘包和半包指的都是一次是不正常的ByteBuf缓存区接收。

#### 8.1.3 半包现象的原理

​	寻根粘包和半包的来源得从操作系统底层说起。

​	大家都知道，**底层网络是以二进制字节报文的形式来传输数据的**。读数据的过程大致为：当IO可读时，Netty会从底层网络将二进制数据读到ByteBuf缓冲区中，再交给Netty程序转成Java POJO对象。写数据的过程大致为：这中间编码器起作用，是将一个Java类型的数据转换成底层能够传输的二进制ByteBuf缓冲数据。解码器的作用与之相反，是将底层传递过来的二进制ByteBuf缓冲数据转换成Java能够处理的Java POJO对象。

​	在发送端Netty的**应用层进程缓冲区**，程序以ByteBuf为单位来发送数据，但是到了**底层操作系统内核缓冲区**，底层会按照协议的规范对数据包进行二进制拼装，拼装成传输层TCP层的协议报文，再进行发送。在接送端接收到传输层的二进制包后，首先保存在**内核缓冲区**，<u>Netty读取ByteBuf时才复制到**进程缓冲区**</u>。

​	在接收端，当Netty程序将数据从**内核缓冲区**复制到**Netty进程缓冲区**的ByteBuf时，问题就来了：

​	**（1）首先，每次读取底层缓冲的数据容量是有限制的，当TCP底层缓冲区的数据包比较大时，会将一个底层包分成多次ByteBuf进行复制，进而造成进程缓冲区读到的是半包。**

​	**（2）当TCP底层缓冲的数据包比较小时，一次复制的却不止一个内核缓冲区包，进而造成进程缓冲区读到的时粘包。**

​	如何解决呢？

​	**基本思路是，在接收端，Netty程序需要根据自定义协议，将读取到的进程缓冲区ByteBuf，在应用程进行二次拼装，重新组装我们应用层的数据包。接收端的这个过程通常也被称为分包，或者叫作拆包。**

​	在Netty中，分包的方法，从第7章可知，主要有两种方法：

<u>（1）可以自定义解码器分包器：基于ByteToMessageDecoder或者ReplayingDecoder，自定义自己的进程缓冲区分包器。</u>

<u>（2）使用Netty内置的解码器。如，使用Netty内置的LengthFieldBasedFrameDecoder自定义分隔符数据包解码器，对进程缓冲区ByteBuf进行正确的分包。</u>

​	在本章后面，这两种方法都会用到。

#### 8.2 JSON协议通信

#### 8.2.1 JSON序列化的通用类

​	Java处理JSON数据有三个比较流行的开源类库：阿里的FastJson、谷歌的Gson和开源社区的Jackson。

​	Jackson是一个简单的、基于Java的JSON开源库。使用Jackson开源库，可以轻松地将Java POJO对象转换成JSON、XML格式字符串；同样也可以方便地将JSON、XML字符串转换成Java POJO对象。Jackson开源库的优点是：所依赖的jar包较少、简单易用、性能也还不错，另外Jackson社区相对比较活跃。<u>Jackson开源库的缺点是：对于复杂POJO类型、复杂的集合Map、List的转换结果，不是标准的JSON格式，或者会出现一些问题</u>。

​	Google的Gson开源库是一个功能齐全的JSON解析库，起源于Google公司内部需求而由Google自行研发而来，在2008年5月公开发布第一版之后已被许多公司或用户使用。<u>Gson可以完成复杂类型的POJO和JSON字符串的相互转换，转换能力非常强</u>。

​	阿里巴巴的FastJson是一个高性能的JSON库。<u>传闻说FastJson在复杂类型的POJO转换JSON时，可能会出现一些引用类型而导致JSON转换出错，需要进行引用的定制</u>。顾名思义，从性能上说，FastJson库采用独创的算法，将JSON转换POJO的速度提升到极致，超过其他JSON开源库。

​	在实际开发中，目前主流的策略是：Google的Gson库和阿里的FastJson库两者结合使用。<u>在POJO序列化成JSON字符串的应用场景，使用Google的Gson库；在JSON字符串反序列化成POJO的应用场景，使用阿里的FastJson库</u>。

​	下面将JSON的序列化和反序列化功能放在一个通用类JsonUtil中，方便后面统一使用。代码如下：

```java
public class JsonUtil {

    //  谷歌GsonBuilder构造器
    static GsonBuilder gb = new GsonBuilder();
    static {
        // 不需要html escape
        gb.disableHtmlEscaping();
    }

    //  序列化：使用谷歌Gson将POJO转成字符串
    public static String pojoToJson(Object obj){
        // String json = new Gson().toJson(obj);
        String json = gb.create().toJson(obj);
        return json;
    }

    //  反序列化：使用阿里Fastjson将字符串转换成POJO对象
    public static <T> T jsonToPojo(String json, Class<T> tClass){
        T t = JSONObject.parseObject(json, tClass);
        return t;
    }
}
```

> json相关：
>
> [GSON](https://www.jianshu.com/p/75a50aa0cad1)
>
> [王学岗Gson解析和泛型和集合数据](https://www.jianshu.com/p/fb53745f8752)
>
> [fastJson与jackson性能对比](https://blog.csdn.net/u013433821/article/details/82905222)
>
> 泛型相关：
>
> [java 泛型详解-绝对是对泛型方法讲解最详细的，没有之一](https://blog.csdn.net/s10461/article/details/53941091)
>
> [java泛型(尖括号里看不懂的东西)](https://blog.csdn.net/weixin_39408343/article/details/95761171)

#### 8.2.2 JSON序列化与反序列化的实践案例

​	下面通过一个小实例，演示一下POJO对象的JSON协议的序列化与反序列化。首先定义一个POJO类，名为JsonMsg类，包含id和content两个属性。然后使用lombok开源库的@Data注释，为属性加上getter和setter方法。

```java
@Data
public class JsonMsg {
    //id Field(域/字段)
    private int id;
    //content Field(域/字段)
    private String content;

    //反序列化：在通用方法中，使用阿里FastJson转成Java POJO对象
    public static JsonMsg parseFromJson(String json) {
        return JsonUtil.jsonToPojo(json, JsonMsg.class);
    }

    //序列化：在通用方法中，使用谷歌Gson转成字符串
    public String convertToJson() {
        return JsonUtil.pojoToJson(this);
    }

}
```

​	在POJO类JsonMsg中，首先加上了一个JSON序列化方法convertToJson()：它调用通用类定义的JsonUtil.pojoToJson(Object)方法，将对象自身序列化为JSON字符串。另外，JsonMsg还加上了一个JSON反序列化方法parseFromJson(String)：它是一个静态方法，调用通用类定义的JsonUtil.jsonToPojo(String, Class)，将JSON字符串反序列化为JsonMsg类型的对象实例。

​	使用POJO类的JsonMsg实现从POJO对象到JSON的序列化、反序列化的实践案例演示，代码如下：

```java
public class JsonMsgDemo {

    //构建Json对象
    public JsonMsg buildMsg() {
        JsonMsg user = new JsonMsg();
        user.setId(1000);
        user.setContent("疯狂创客圈:高性能学习社群");
        return user;
    }

    //序列化 serialization & 反序列化 Deserialization
    @Test
    public void serAndDesr() throws IOException {
        JsonMsg message = buildMsg();
        //将POJO对象，序列化成字符串
        String json = message.convertToJson();
        //可以用于网络传输,保存到内存或外存
        System.out.println("json:=" + json);

        //JSON 字符串,反序列化成对象POJO
        JsonMsg inMsg = JsonMsg.parseFromJson(json);
        System.out.println("id:=" + inMsg.getId());
        System.out.println("content:=" + inMsg.getContent());
    }
}
```

​	运行结果如下：

```java
json:={"id":1000,"content":"疯狂创客圈:高性能学习社群"}
id:=1000
content:=疯狂创客圈:高性能学习社群
```

#### 8.2.3 JSON传输的编码器和解码器之原理

​	<u>本质上来说，JSON格式仅仅是字符串的一种组织形式</u>。所以，传输JSON的所用到的协议与传输普通文本所使用的协议没有什么不同。下面使用常用的Head-Content协议来介绍一下JSON的传输。

​	Head-Content数据包的解码过程大致如下：

​	先使用LengthFieldBasedFrameDecoder（Netty内置的自定义长度数据包解码器）解码Head-Content二进制数据包，解码出Content字段的二进制内容。然后，使用StringDecoder字符串解码器（Netty内置的解码器）将二进制内容解码成JSON字符串。最后，使用JsonMsgDecoder解码器（一个自定义解码器）将JSON字符串解码成POJO对象。

​	Head-Content数据包的编码过程大致如下：

​	先使用StringEncoder编码器（Netty内置）将JSON字符串编码成二进制字节数组。然后，使用LengthFieldPrepender编码器（Netty内置）将二进制字节数组编码成Head-Content二进制数据包。

​	**LengthFieldPrepender编码器的作用：在数据包的前面加上内容的二进制字节数组的长度**。<u>这个编码器和LengthFieldBasedFrameDecoder解码器是天生的一对，常常配套使用</u>。这组“天仙配”属于Netty所提供的一组非常重要的编码器和解码器，<u>常常用于Head-Content数据包的传输</u>。

​	LengthFieldPrepender编码器有两个常用的构造器：

```java
public class LengthFieldPrepender extends MessageToByteEncoder<ByteBuf> {
    // ...成员变量
    
    public LengthFieldPrepender(int lengthFieldLength) {
        this(lengthFieldLength, false);
    }

    public LengthFieldPrepender(int lengthFieldLength, boolean lengthIncludesLengthFieldLength) {
        this(lengthFieldLength, 0, lengthIncludesLengthFieldLength);
    }
    // ...其他构造函数
}
```

​	上面的构造器中，第一个参数lengthFieldLength表示Head长度字段所占用的字节数。另一个参数lengthIncludesLengthFieldLength表示Head字段是否包含长度字段自身的字节数。<u>如果该参数的值为true，表示长度字段的值（总长度）包含了自己的字节数。如果值为false，表示长度字段的值仅仅包含Content内容的二进制数据的长度。一般来说，lengthIncludesLengthFieldLength的默认值为false。</u>

#### 8.2.4 JSON传输之服务器端的实践案例

​	为了清晰地演示JSON传输，下面设计一个简单的客户端/服务器传输程序：服务器接收客户端的数据包，并解码成JSON，再转换POJO；客户端将POJO转换成JSON字符串，编码后发送给到服务器端。

​	为了简化流程，此服务器端的代码仅仅包含Inbound入站处理的流程，不包含OutBound出站处理的流程。也就是说，服务器端的程序仅仅读取客户端数据包并完成解码。服务器端的程序没有写出任何的输出数据包到对端（即客户端）。服务器端实践案例的程序代码如下：

```java
public class JsonServer {

    private final int serverPort;
    ServerBootstrap b = new ServerBootstrap();

    public JsonServer(int port) {
        this.serverPort = port;
    }

    public void runServer() {
        //创建reactor 线程组
        EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

        try {
            //1 设置reactor 线程组
            b.group(bossLoopGroup, workerLoopGroup);
            //2 设置nio类型的channel
            b.channel(NioServerSocketChannel.class);
            //3 设置监听端口
            b.localAddress(serverPort);
            //4 设置通道的参数
            b.option(ChannelOption.SO_KEEPALIVE, true);
            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
            b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

            //5 装配子通道流水线
            b.childHandler(new ChannelInitializer<SocketChannel>() {
                //有连接到达时会创建一个channel
                protected void initChannel(SocketChannel ch) throws Exception {
                    // pipeline管理子通道channel中的Handler
                    // 向子channel流水线添加3个handler处理器
                    ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4, 0, 4));
                    ch.pipeline().addLast(new StringDecoder(CharsetUtil.UTF_8));
                    ch.pipeline().addLast(new JsonMsgDecoder());
                }
            });
            // 6 开始绑定server
            // 通过调用sync同步方法阻塞直到绑定成功
            ChannelFuture channelFuture = b.bind().sync();
            System.out.println(" 服务器启动成功，监听端口: " +
                    channelFuture.channel().localAddress());

            // 7 等待通道关闭的异步任务结束
            // 服务监听通道会一直等待通道关闭的异步任务结束
            ChannelFuture closeFuture = channelFuture.channel().closeFuture();
            closeFuture.sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 8 优雅关闭EventLoopGroup，
            // 释放掉所有资源包括创建的线程
            workerLoopGroup.shutdownGracefully();
            bossLoopGroup.shutdownGracefully();
        }

    }

    //服务器端业务处理器
    static class JsonMsgDecoder extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            String json = (String) msg;
            JsonMsg jsonMsg = JsonMsg.parseFromJson(json);
            System.out.println("收到一个 Json 数据包 =》" + jsonMsg);

        }
    }


    public static void main(String[] args) throws InterruptedException {
        int port = 8099;
        new JsonServer(port).runServer();
    }
}
```

#### 8.2.5 JSON传输之客户端的实践案例

​	为了简化流程，客户端的代码仅仅包含Outbound出站处理的流程，不包含Inbound入站处理的流程。也就是说，客户端的程序仅仅进行数据的编码，然后把数据包写到服务器端。客户端的程序并没有去处理从对端（即服务器端）过来的输入数据包。客户端的流程大致如下：

​	（1）通过谷歌的Gson框架，将POJO序列化成JSON字符串。

​	（2）然后，使用StringEncoder编码器（Netty内置）将JSON字符串编码成二进制字节数组。

​	（3）最后，使用LengthFieldPrepender编码器（Netty内置）将二进制字节数组编码成Head-Content格式的二进制数据包。

​	客户端实践案例的程序代码如下：

```java
public class JsonSendClient {
    static String content = "疯狂创客圈：高性能学习社群!";

    private int serverPort;
    private String serverIp;
    Bootstrap b = new Bootstrap();

    public JsonSendClient(String ip, int port) {
        this.serverPort = port;
        this.serverIp = ip;
    }

    public void runClient() {
        //创建reactor 线程组
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

        try {
            //1 设置reactor 线程组
            b.group(workerLoopGroup);
            //2 设置nio类型的channel
            b.channel(NioSocketChannel.class);
            //3 设置监听端口
            b.remoteAddress(serverIp, serverPort);
            //4 设置通道的参数
            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

            //5 装配通道流水线
            b.handler(new ChannelInitializer<SocketChannel>() {
                //初始化客户端channel
                protected void initChannel(SocketChannel ch) throws Exception {
                    // 客户端channel流水线添加2个handler处理器
                    ch.pipeline().addLast(new LengthFieldPrepender(4));
                    ch.pipeline().addLast(new StringEncoder(CharsetUtil.UTF_8));
                }
            });
            ChannelFuture f = b.connect();
            f.addListener(new GenericFutureListener<ChannelFuture>() {
                @Override
                public void operationComplete(ChannelFuture futureListener) throws Exception {
                    if (futureListener.isSuccess()) {
                        System.out.println("EchoClient客户端连接成功!");
                    } else {
                        System.out.println("EchoClient客户端连接失败!");
                    }
                }
            });

            // 阻塞,直到连接完成
            f.sync();
            Channel channel = f.channel();

            //发送 Json 字符串对象
            for (int i = 0; i < 1000; i++) {
                JsonMsg user = build(i, i + "->" + content);
                channel.writeAndFlush(user.convertToJson());
                System.out.println("发送报文：" + user.convertToJson());
            }
            channel.flush();


            // 7 等待通道关闭的异步任务结束
            // 服务监听通道会一直等待通道关闭的异步任务结束
            ChannelFuture closeFuture = channel.closeFuture();
            closeFuture.sync();

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 优雅关闭EventLoopGroup，
            // 释放掉所有资源包括创建的线程
            workerLoopGroup.shutdownGracefully();
        }

    }

    //构建Json对象
    public JsonMsg build(int id, String content) {
        JsonMsg user = new JsonMsg();
        user.setId(id);
        user.setContent(content);
        return user;
    }


    public static void main(String[] args) throws InterruptedException {
        int port = 8099;
        String ip = "127.0.0.1";
        new JsonSendClient(ip, port).runClient();
    }
}
```

​	依次运行服务器端和客户端程序后，输出结果部分如下：

```java
// 服务器端
服务器启动成功，监听端口: /0:0:0:0:0:0:0:0:8099
收到一个 Json 数据包 =》JsonMsg(id=0, content=0->疯狂创客圈：高性能学习社群!)
收到一个 Json 数据包 =》JsonMsg(id=1, content=1->疯狂创客圈：高性能学习社群!)
收到一个 Json 数据包 =》JsonMsg(id=2, content=2->疯狂创客圈：高性能学习社群!)
// ... 

// 客户端
EchoClient客户端连接成功!
发送报文：{"id":0,"content":"0->疯狂创客圈：高性能学习社群!"}
发送报文：{"id":1,"content":"1->疯狂创客圈：高性能学习社群!"}
发送报文：{"id":2,"content":"2->疯狂创客圈：高性能学习社群!"}
// ...
```

### 8.3 Protobuf协议通讯

​	Protobuf是Google提出的一种数据交换的格式，是一套类似JSON或者XML的数据传输格式和规范，用于不同应用或进程之间的通信。Protobuf的编码过程为：使用预先定义的Message数据结构将实际的传输数据进行打包，然后编码成二进制的码流进行传输或者存储。Protobuf的解码过程刚刚好与编码过程相反：将二进制码流解码成Protobuf自己定义的Message结构的POJO实例。

​	**Protobuf既独立于语言，又独立于平台**。Google官方提供了多种语言的实现：Java、C#、C++、GO、JavaScript和Python。**Protobuf数据包是一种二进制的格式，相对于文本格式的数据交换（JSON、XML）来说，速度要快很多**。由于Protobuf优异的性能，使得它更加适用于**分布式应用场景下的数据通信或者异构环境下的数据交换**。

​	与JSON、XML相比，Protobuf算是后起之秀，是Google开源的一种数据格式。只是<u>Protobuf更加适合于高性能、快速响应的数据传输应用场景</u>。另外，JSON、XML是文本格式，数据具有可读性；而Protobuf是二进制数据格式，数据本身不具有可读性，只有反序列化之后才能得到真正可读的数据。正因为Protobuf是二进制数据格式，数据序列化之后，体积比JSON和XML更小，更加适合网络传输。

​	总体来说，<u>在一个需要大量数据传输的应用场景中，因为数据量很大，则选择Protobuf可以明显地减少传输的数据量和提升网络IO的速度</u>。对于打造一款高性能的通信服务器来说，Protobuf传输协议是最高性能的传输协议之一。微信的消息传输就采用了Protobuf协议。

> [深入 ProtoBuf - 简介](https://www.jianshu.com/p/a24c88c0526a)
>
> [深入 ProtoBuf - 编码](https://www.jianshu.com/p/73c9ed3a4877)

#### 8.3.1 一个简单的proto文件的实践案例

​	Protobuf使用proto文件来预先定义的消息格式。数据包是按照proto文件所定义的消息格式完成二进制码流的编码和解码。proto文件，简单地说，就是一个消息的协议文件，这个协议文件的后缀文件名为".proto"。

​	作为演示，下面介绍一个非常简单的proto文件：仅仅定义一个消息结构体，并且该消息结构体也非常简单，仅包含两个字段。实例如下：

```protobuf
// [开始头部声明]
syntax = "proto3";
package com.crazymakercircle.im.common.bean.msg;
// [结束头部声明]

// [开始 java选项配置]
option java_package = "com.crazymakercircle.netty.protocol";
option java_outer_classname = "MsgProtos";
// [结束 java选项配置]

// [开始消息定义]
message Msg{
	uint32 id = 1;	// 消息ID
	string content = 2;	// 消息内容
}
// [结束消息定义]
```

​	在”.proto“文件的头部声明中，需要声明”.proto“所使用的Protobuf协议版本，这里使用的是”proto3“。也可以使用旧一点的版本"proto2"，两个版本的消息格式有一些细微的不同。默认的协议版本为”proto2“。

​	Protobuf支持很多语言，所以它为不同的语言提供了一些可选的声明选项，选项的前面有option关键字。<u>”java_package“选项的作用为：在生成”proto“文件中消息的POJO类和Builder（构造者）的Java代码时，将Java代码放入指定的package中。”java_outer_classname“选项的作用为：在生成”proto“文件所对应Java代码时，所生产的Java外部类的名称</u>。

​	在”proto“文件中，使用message这个关键字来定义消息的结构体。在生成”proto“对应的Java代码时，每个具体的消息结构体都对应一个最终的Java POJO类。<u>消息结构体的字段对应到POJO类的属性</u>。也就是说，每定义一个”message“结构体相当于声明一个Java中的类。并且message中可以内嵌message，就像Java的内部类一样。

​	每一个消息结构体可以有多个字段。定义一个字段的格式，简单来说就是”类型 名称 = 编号“。例如”string content = 2；“表示该字段是string类型，名为content，序号为2。**字段序号表示为：在Protobuf数据包的序列化、反序列化时，该字段的具体排序**。

​	在每一个".proto"文件中，可以声明多个”message“。**大部分情况下，会把有依赖关系或者包含关系的message消息结构体写入一个.proto文件。将那些没有关联关系的message消息结构体，分别写入不同的文件，这样便于管理。**

#### 8.3.2 控制台命令生成POJO和Builder

​	完成”.proto“文件定义后，下一步就是生成消息的POJO类和Builder（构造者）类。有两种方式生成Java类：一种是通过控制台命令的方式；另一种是使用Maven插件的方式。

​	先看第一种方式：通过控制台命令生成消息的POJO类和Builder构造者。

​	首先从”https://github.com/protocolbuffers/protobuf/releases“下载Protobuf的安装包，可以选择不同的版本，这里下载的是3.6.1的java版本。在Windows下解压后执行安装。备注：这里以Windows平台为例子，对于在Linux或者Mac平台下，自行尝试。

​	生成构造者代码，需要用到安装文件中的protoc.exe可执行文件。安装完成后，设置一下path环境变量。将poto的安装目录加入到path环境变量中。

​	下面开始使用protoc.exe文件生成的Java的Builder（构造者）。生成的命令如下：

```none
protoc.exe --java_out=./src/main/java/ ./Msg.proto
```

​	上面的命令表示“proto”文件的名称为：“./Msg.proto”；所生产的POJO类和构造者类的输出文件夹为./src/main/java/。

#### 8.3.3 Maven插件生成POJO和Builder

​	使用命令行生成Java类的操作比较繁琐。另一种更加方便的方式是：使用protobuf-maven-plugin插件，它可以非常方便地生成消息的POJO类和Buidler（构造者）类的Java代码。在Maven的pom文件中增加此plugin插件的配置项，具体如下：

```xml
    <plugin>
        <groupId>org.xolstice.maven.plugins</groupId>
        <artifactId>protobuf-maven-plugin</artifactId>
        <version>0.5.0</version>
        <extensions>true</extensions>
        <configuration>
            <!--proto文件路径-->
            <protoSourceRoot>${project.basedir}/protobuf</protoSourceRoot>
            <!--目标路径-->
            <outputDirectory>${project.build.sourceDirectory}</outputDirectory>
            <!--设置是否在生成java文件之前清空outputDirectory的文件-->
            <clearOutputDirectory>false</clearOutputDirectory>
            <!--临时目录-->
            <temporaryProtoFileDirectory>${project.build.directory}/protoc-temp</temporaryProtoFileDirectory>
            <!--protoc 可执行文件路径-->
            <protocExecutable>${project.basedir}/protobuf/protoc3.6.1.exe</protocExecutable>
        </configuration>
        <executions>
            <execution>
                <goals>
                    <goal>compile</goal>
                    <goal>test-compile</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
```

​	protobuf-maven-plugin插件的配置项，具体介绍如下：

+ protoSourceRoot：“proto”消息结构体文件的路径
+ outputDirectory：生成的POJO类和Builder类的目标路径
+ protocExecutable：Java代码生成器工具的protoc3.6.1.exe可执行文件的路径。

​	配置好之后，执行插件的compile命令，Java代码就利索生成了。或者在Maven的项目编译时，POJO类和Builder类也会自动生成。

#### 8.3.4 消息POJO和Builder的使用之实践案例

​	在Maven的pom.xml文件上加上protobuf的Java运行包的依赖，代码如下：

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>${protobuf.version}</version>
</dependency>
```

​	这里的protobuf.version版本号的具体值为3.6.1。也就是说，Java运行时的protobuf依赖包的版本和“.proto”消息结构体文件中的syntax配置版本，以及编译“.proto”文件所使用的编译器“protoc3.6.1.exe”的版本，这三个版本需要配套一致。

1. 使用Builder构造者，构造POJO消息对象

```java
public class ProtobufDemo {
    public static MsgProtos.Msg buildMsg() {
        MsgProtos.Msg.Builder personBuilder = MsgProtos.Msg.newBuilder();
        personBuilder.setId(1000);
        personBuilder.setContent("疯狂创客圈:高性能学习社群");
        MsgProtos.Msg message = personBuilder.build();
        return message;
    }
    // ...
}
```

​	**Protobuf为每个message消息结构体生成的Java类中，包含了一个POJO类、一个Builder类**。构造POJO消息，首先需要使用POJO类的newBuilder<u>静态方法</u>获得一个Builder构造者。<u>每一个POJO字段的值，需要通过Builder构造者的setter方法去设置</u>。注意，<u>消息POJO对象并没有setter方法</u>。字段值设置完成之后，使用构造者的build()方法构造出POJO消息对象。

2. 序列化 serialization & 反序列化 Deserialization的方式一

​	获得消息POJO实例之后，可以通过多种方法将POJO对象序列化为二进制字节，或者反序列化。下面是方式一：

```java
public class ProtobufDemo {
    public void serAndDesr1() throws IOException {
        MsgProtos.Msg message = buildMsg();
        //将Protobuf对象，序列化成二进制字节数组
        byte[] data = message.toByteArray();
        //可以用于网络传输,保存到内存或外存
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        outputStream.write(data);
        data = outputStream.toByteArray();
        //二进制字节数组,反序列化成Protobuf 对象
        MsgProtos.Msg inMsg = MsgProtos.Msg.parseFrom(data);
        System.out.println("id:=" + inMsg.getId());
        System.out.println("content:=" + inMsg.getContent());
    }
}
```

​	输出结果：

```java
id:=1000
content:=疯狂创客圈:高性能学习社群
```

​	这种方式通过调用POJO对象的toByteArray()方法<u>将POJO对象序列化成字节数组</u>。通过调用parseFrom(byte[] data)方法，Protobuf也可以从<u>字节数组中重新反序列化得到POJO新的实例</u>。

​	**这种方式类似于普通Java对象的序列化，适用于很多将Protobuf的POJO序列化到内存或者外层的应用场景。**

3. 序列化 serialization & 反序列化 Deserialization 的方式二

```java
public class ProtobufDemo {
    public void serAndDesr2() throws IOException {
        MsgProtos.Msg message = buildMsg();
        //序列化到二进制流
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        message.writeTo(outputStream);
        ByteArrayInputStream inputStream = new ByteArrayInputStream(outputStream.toByteArray());
        //从二进流,反序列化成Protobuf 对象
        MsgProtos.Msg inMsg = MsgProtos.Msg.parseFrom(inputStream);
        System.out.println("id:=" + inMsg.getId());
        System.out.println("content:=" + inMsg.getContent());
    }
}
```

​	这种方式通过调用POJO对象的writeTo(OutputStream)方法将<u>POJO对象的二进制字节写出到输出流</u>。通过调用parseFrom(InputStream)方法，Protobuf<u>从输入流中读取二进制码流重新反序列化</u>，得到POJO新的实例。

​	在阻塞式的二进制码流传输到应用场景中，这种序列化和反序列化的方式是没有问题的。例如，可以将二进制码流写入阻塞式的Java OIO套接字或者输出到文件。但是，**这种方式在异步操作的NIO应用场景中，存在着粘包/半包的问题**。

4. 序列化 serialization & 反序列化 Deserialization的方式三

```java
public class ProtobufDemo {
    //带字节长度：[字节长度][字节数据],解决粘包/半包问题
    public void serAndDesr3() throws IOException {
        MsgProtos.Msg message = buildMsg();
        //序列化到二进制流
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        message.writeDelimitedTo(outputStream);
        ByteArrayInputStream inputStream = new ByteArrayInputStream(outputStream.toByteArray());
        //从二进流,反序列化成Protobuf 对象
        MsgProtos.Msg inMsg = MsgProtos.Msg.parseDelimitedFrom(inputStream);
        System.out.println("id:=" + inMsg.getId());
        System.out.println("content:=" + inMsg.getContent());
    }
}
```

​	这种方式通过调用POJO对象的writeDelimitedTo(OutputStream)方法在序列化的字节码之前添加了字节数组的长度。<u>这一点类似于前面介绍的Head-Content协议，只不过Protobuf做了优化，长度的类型不是固定长度的int类型，而是可变长度varint32类型</u>。

​	反序列化时，调用parseDelimitedFrom(InputStream)方法。Protobuf从输入流中先读取varint32类型的长度值，然后根据长度值读取此消息的二进制字节，再反序列化得到POJO新的实例。

​	**这种方式可以用于异步操作的NIO应用场景中，解决了粘包/半包的问题**。

### 8.4 Protobuf编解码的实践案例

​	<u>Netty默认支持Protobuf的编码和解码，内置了一套基础的Protobuf编码和解码器。</u>

#### 8.4.1 Protobuf编码器和解码器的原理

​	Netty内置的Protobuf专用的基础编码器/解码器为：ProtobufEncoder编码器和ProtobufDecoder解码器。

1. ProtobufEncoder编码器

   ​	翻开Netty源代码，我们发现ProtobufEncoder的实现逻辑非常简单，<u>直接使用了message.toByteArray()方法将Protobuf的POJO消息对象编码成二进制字节，数据放入Netty的Bytebuf数据包中，然后交给了下一站的编码器。</u>

   ```java
   @Sharable
   public class ProtobufEncoder extends MessageToMessageEncoder<MessageLiteOrBuilder> {
       public ProtobufEncoder() {
       }
   
       protected void encode(ChannelHandlerContext ctx, MessageLiteOrBuilder msg, List<Object> out) throws Exception {
           if (msg instanceof MessageLite) {
               out.add(Unpooled.wrappedBuffer(((MessageLite)msg).toByteArray()));
           } else {
               if (msg instanceof Builder) {
                   out.add(Unpooled.wrappedBuffer(((Builder)msg).build().toByteArray()));
               }
   
           }
       }
   }
   ```

2. ProtobufDecoder解码器

   ​	ProtobufDecoder解码器和ProtobufEncoder编码器相互对应。ProtobufDecoder需要指定一个POJO消息的prototype原型POJO实例，根据原型实例找到对应的Parer解析器，将二进制的字节解析为Protobuf POJO消息对象。

   ​	**在Java NIO通信中，仅仅使用以上这组编码器和解码器会存在粘包/半包的问题**。Netty也提供了配套的Head-Content类型的Protobuf编码器和解码器，在二进制码流之前加上二进制字节数组的长度。

3. ProtobufVarint32LengthFieldPrepender长度编码器

   ​	这个编码器的作用是，可以在ProtobufEncoder生成的字节数组之前，前置一个varint32数字，表示序列化的二进制字节数。

4. ProtobufVarint32FrameDecoder长度解码器

   ​	ProtobufVarint32FrameDecoder和ProtobufVarint32LengthFieldPrepender相互对应。其作用是，根据数据包中的varint32的长度值，解码一个足额的字节数组。然后将字节数组交给下一站的解码器ProbufDecoder。

​	**varint32是一种紧凑的表示数字的方法，它不是一种具体的数据类型**。<u>varint32它用一个或多个字节来表示一个数字，值越小的数字使用越少的字节数，值越大使用的字节数越多</u>。varint32根据值的大小自动进行长度的收缩，这能减少用于保存长度的字节数。也就是说，**varint32与int类型的最大区别是：varint32用一个或多个字节来表示一个数字。varint32不是固定长度，所以为了更好地减少通信过程中的传输量，消息头中的长度尽量采用varint格式**。

​	至此，Netty的内置的Protobuf的编码器和解码器已经初步介绍完了。可以通过这两组编码器/解码器完成Length+ProtobufData(Head-Content)协议的数据传输。但是，<u>在更加复杂的传输应用场景，Netty的内置编码器和解码器是不够用的</u>。例如，在Head部分加上魔数字段进行安全验证；或者还需要对Protobuf Data的内容进行加密和解密等。也就是说，**在复杂的传输应用场景下，需要定制属于自己的Protobuf编码器和解码器**。

*ps：吐槽，果然到最后，还是提倡自己写编码器和解码器，哈哈。*

#### 8.4.2 Protobuf传输之服务器端的实践案例

​	为了清晰地演示Protobuf传输，下面设计了一个简单的客户端/服务器端传输程序：服务器端接收客户端的数据包，并解码成Protobuf的POJO；客户端将Protobuf的POJO编码成二进制数据包，再发送到服务器端。

​	在服务器端，Protobuf协议的解码过程如下：

​	先使用Netty内置的ProtobufVarint32FrameDecoder，根据varint32格式的可变长度值，从入站数据包中解码出二进制Protobuf POJO对象。最后，自定义一个ProtobufBussinessDecoder解码器来处理Protobuf POJO对象。

​	服务器端的实践案例程序代码如下：

```java
public class ProtoBufServer {

    private final int serverPort;
    ServerBootstrap b = new ServerBootstrap();

    public ProtoBufServer(int port) {
        this.serverPort = port;
    }

    public void runServer() {
        //创建reactor 线程组
        EventLoopGroup bossLoopGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

        try {
            //1 设置reactor 线程组
            b.group(bossLoopGroup, workerLoopGroup);
            //2 设置nio类型的channel
            b.channel(NioServerSocketChannel.class);
            //3 设置监听端口
            b.localAddress(serverPort);
            //4 设置通道的参数
            b.option(ChannelOption.SO_KEEPALIVE, true);
            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
            b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

            //5 装配子通道流水线
            b.childHandler(new ChannelInitializer<SocketChannel>() {
                //有连接到达时会创建一个channel
                protected void initChannel(SocketChannel ch) throws Exception {
                    // pipeline管理子通道channel中的Handler
                    // 向子channel流水线添加3个handler处理器
                    ch.pipeline().addLast(new ProtobufVarint32FrameDecoder());
                    ch.pipeline().addLast(new ProtobufDecoder(MsgProtos.Msg.getDefaultInstance()));
                    ch.pipeline().addLast(new ProtobufBussinessDecoder());
                }
            });
            // 6 开始绑定server
            // 通过调用sync同步方法阻塞直到绑定成功
            ChannelFuture channelFuture = b.bind().sync();
            System.out.println(" 服务器启动成功，监听端口: " +
                    channelFuture.channel().localAddress());

            // 7 等待通道关闭的异步任务结束
            // 服务监听通道会一直等待通道关闭的异步任务结束
            ChannelFuture closeFuture = channelFuture.channel().closeFuture();
            closeFuture.sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 8 优雅关闭EventLoopGroup，
            // 释放掉所有资源包括创建的线程
            workerLoopGroup.shutdownGracefully();
            bossLoopGroup.shutdownGracefully();
        }

    }

    //服务器端业务处理器
    static class ProtobufBussinessDecoder extends ChannelInboundHandlerAdapter {
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            MsgProtos.Msg protoMsg = (MsgProtos.Msg) msg;
            //经过pipeline的各个decoder，到此Person类型已经可以断定
            System.out.println("收到一个 MsgProtos.Msg 数据包 =》");
            System.out.println("protoMsg.getId():=" + protoMsg.getId());
            System.out.println("protoMsg.getContent():=" + protoMsg.getContent());
        }
    }

    public static void main(String[] args) throws InterruptedException {
        int port = 8099;
        new ProtoBufServer(port).runServer();
    }
}
```

#### 8.4.3 Protobuf传输之客户端的实践案例

​	在客户端开始出站之前，需要提前构造好Protobuf的POJO对象。然后可以使用通道的write/writeAndFlush方法，启动出站处理的流水线执行工作。

​	客户端的出站处理流程中，Protobuf协议的编码过程大致如下：

​	先使用Netty内置的ProtobufEncoder，将Protobuf POJO对象编码成二进制的字节数组；然后，使用Netty内置的ProtobufVarint32LengthFieldPrepender编码器，加上varint32格式的可变长度。Netty会将完成了编码后的Length+Content格式的二进制字节码发送到服务器端。

​	客户端的实践案例程序代码如下：

```java
public class ProtoBufSendClient {
    static String content = "疯狂创客圈：高性能学习社群!";

    private int serverPort;
    private String serverIp;
    Bootstrap b = new Bootstrap();

    public ProtoBufSendClient(String ip, int port) {
        this.serverPort = port;
        this.serverIp = ip;
    }

    public void runClient() {
        //创建reactor 线程组
        EventLoopGroup workerLoopGroup = new NioEventLoopGroup();

        try {
            //1 设置reactor 线程组
            b.group(workerLoopGroup);
            //2 设置nio类型的channel
            b.channel(NioSocketChannel.class);
            //3 设置监听端口
            b.remoteAddress(serverIp, serverPort);
            //4 设置通道的参数
            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

            //5 装配通道流水线
            b.handler(new ChannelInitializer<SocketChannel>() {
                //初始化客户端channel
                protected void initChannel(SocketChannel ch) throws Exception {
                    // 客户端channel流水线添加2个handler处理器
                    ch.pipeline().addLast(new ProtobufVarint32LengthFieldPrepender());
                    ch.pipeline().addLast(new ProtobufEncoder());
                }
            });
            ChannelFuture f = b.connect();
            f.addListener(new GenericFutureListener<ChannelFuture>() {
                @Override
                public void operationComplete(ChannelFuture futureListener) throws Exception {
                    if (futureListener.isSuccess()) {
                        System.out.println("EchoClient客户端连接成功!");

                    } else {
                        System.out.println("EchoClient客户端连接失败!");
                    }

                }
            });

            // 阻塞,直到连接完成
            f.sync();
            Channel channel = f.channel();

            //发送 Protobuf 对象
            for (int i = 0; i < 1000; i++) {
                MsgProtos.Msg user = build(i, i + "->" + content);
                channel.writeAndFlush(user);
                System.out.println("发送报文数：" + i);
            }
            channel.flush();


            // 7 等待通道关闭的异步任务结束
            // 服务监听通道会一直等待通道关闭的异步任务结束
            ChannelFuture closeFuture = channel.closeFuture();
            closeFuture.sync();

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 优雅关闭EventLoopGroup，
            // 释放掉所有资源包括创建的线程
            workerLoopGroup.shutdownGracefully();
        }

    }

    //构建ProtoBuf对象
    public MsgProtos.Msg build(int id, String content) {
        MsgProtos.Msg.Builder builder = MsgProtos.Msg.newBuilder();
        builder.setId(id);
        builder.setContent(content);
        return builder.build();
    }

    public static void main(String[] args) throws InterruptedException {
        int port = 8099;
        String ip = "127.0.0.1";
        new ProtoBufSendClient(ip, port).runClient();
    }
}
```

​	运行结果如下：

```java
// 服务器端：
 服务器启动成功，监听端口: /0:0:0:0:0:0:0:0:8099
收到一个 MsgProtos.Msg 数据包 =》
protoMsg.getId():=0
protoMsg.getContent():=0->疯狂创客圈：高性能学习社群!
收到一个 MsgProtos.Msg 数据包 =》
protoMsg.getId():=1
protoMsg.getContent():=1->疯狂创客圈：高性能学习社群!
收到一个 MsgProtos.Msg 数据包 =》
// ...

// 客户端：
EchoClient客户端连接成功!
发送报文数：0
发送报文数：1
发送报文数：2
// ...
```

### 8.5 详解Protobuf协议语法

> [Protobuf 的 proto3 与 proto2 的区别](https://solicomo.com/network-dev/protobuf-proto3-vs-proto2.html)

​	在Protobuf中，通信协议的格式是通过”.proto“文件定义的。**一个”.proto“文件有两大组成部分：头部声明、消息结构体的定义**。

​	**头部声明部分，包含了协议的版本、包名、特定语言的选项设置等；消息结构体部分，可以定义一个或者多个消息结构体。**

​	在Java中，当用Protobuf编译器（如“protoc3.6.exe”）来编译“.proto”文件时，编译器将生成Java语言的POJO消息类和Builder构造者类。<u>这些代码可以操作在.proto文件中定义的消息类型，包括获取、设置字段值，将消息序列化到一个输出流中（序列化），以及从一个输入流中解析信息（反序列化）</u>。

#### 8.5.1 proto的头部声明

​	前面介绍了一个简单的“.proto”文件，其头部声明如下：

```java
// [开始声明]
syntax = "proto3";
 //定义protobuf的包名称空间
package protocol;
// [结束声明]

// [开始 java 选项配置]
option java_package = "protocol";
option java_outer_classname = "MsgProtos";
// [结束 java 选项配置]
```

1. syntax版本号

   ​	对于一个".proto"文件而言，文件**首个非空、非注释的行必须注明Protobuf的语法版本**，即syntax=“proto3”，否则<u>默认版本是proto2</u>。

2. package包

   ​	和Java语言类似，通过package指定包名，用来避免信息（message）名字冲突。**如果两个信息的名称相同，但是package包名不同，则它们可以共同存在。**

   ​	如果第一个“.proto”文件定义了一个Msg结构体，package包名如下：

   ```java
   package test;
   message Msg{...}
   ```

   ​	假设另一个“.proto”文件，定义了一个相同名字的消息，package包名如下：

   ```java
   package haha;
   message Msg{
       // ...
       test.Msg testMsg = 1;
       // ...
   }
   ```

   ​	我们可以看到，第二个".proto"文件中，可以用”包名+消息名称“（全限定名）来引用第一个Msg结构体。这一点和Java中package的使用方法是一样的。

   ​	另外，package指定包名后，会对生成的消息POJO代码产生影响。在Java语言中，会以package指定的包名作为生成的POJO类的包名。

3. 选项

   ​	**选项是否生效与”.proto“文件使用的一些特定的语言场景有关**。在Java语言中，以”java_“打头的”option“选项会生效。

   ​	选项option java_package表示Protobuf编译器在生成Java POJO消息类时，生成类所在的Java包名。<u>如果没有该选项，则会以头部声明中的package作为Java包名</u>。

   ​	选项option java_multiple_files表示在生成Java类时的打包方式。有两种方式：

   ​	方式1：一个消息对应一个独立的Java类。

   ​	方式2：所有的消息都作为内部类，打包到一个外部类中。

   ​	<u>此选项的值，默认为false，也就是方式2</u>，表示使用外部类打包的方式。如果设置option java_multiple_files=true，则使用方式1生成Java类，则一个消息对应一个POJO Java类。

   ​	<u>选项option java_outer_classname表示Protobuf编译器在生成Java POJO消息类时，如果”.proto“定义的全部POJO消息类都作为内部类打包在同一个外部类中，则以此作为外部类的类名。</u>

#### 8.5.2 消息结构体与消息字段

​	定义Protobuf消息结构体的关键字为message。一个消息结构体由一个或者多个消息字段组合而成。

```java
// [开始 消息定义]
message Msg {
  uint32 id = 1;  // 消息ID
  string content = 2; // 消息内容
}
// [结束 消息定义]
```

​	**Protobuf消息字段的格式为：**

​	**限定修饰符①|数据类型②|字段名称③|=|分配标识号④**

1. 限定修饰符

   + repeated限定修饰符：表示该字段可以包含0~N个元素值，相当于Java中的List（列表数据类型）。
   + singular限定修饰符：表示该字段可以包含0~1个元素值。singular限定修饰符是默认的字段修饰符。
   + reserved限定修饰符：用来保留字段名称（Field Name）和分配标识号（Assigning Tags），用于将来的扩展。

   ```java
   message MsgFoo{
       // ...
       reserved 12, 15, 9 to 11; // 预留将来使用的分配标识号（Assigning Tags）
   	reserved "foo", "bar";	  // 预留将来使用的字段名（field name）
   }
   ```

2. 数据类型

   ​	详见下一个小节

3. 字段名称

   ​	字段名称的命名与Java语言的成员变量命名方式几乎是相同的。

   ​	<u>Protobuf建议字段的命名以下划线分割，例如使用first_name形式，而不是驼峰式firstName</u>；

4. 分配标识号

   ​	**在消息定义中，每个字段都有唯一的一个数字标识符，可以理解为字段编码值，叫作分配标识号（Assigning Tags）。通过该值，通信双方才能互相识别对方的字段。**当然，相同的编码值，它的限定修饰符和数据类型必须相同。**分配标识号是用来在消息的二进制格式中识别各个字段的，一旦开始使用就不能够再改变**。

   ​	分配标识号的取值范围为1~2<sup>32</sup>(4294967296)。**<u>其中编号[1, 15]之内的分配标识号，时间和空间效率是最高的</u>**。为什么呢？[1 , 15]之内的标识符，在编码的时候只会占用一个字节。[16 , 2047]之内的标识号要占用2个字节。所以那些频繁出现的消息字段，应该使用[1 , 15]之内的标识号。切记：**要为将来有可能添加的、频繁出现的字段预留一些标识号。另外，[1900 , 2000]之内的标识号，为Google Protobuf系统的内部保留值，建议不要在自己的项目中使用**。

   ​	一个消息结构体中的标识号无须是连续的。另外，在同一个消息结构体中，不同的字段不能够使用相同的标识号。

#### 8.5.3 字段的数据类型

​	Protobuf定义了一套基本数据类型。几乎都可以映射到C++\Java等语言的基本数据类型。

| proto Type | Notes                                                        | Java Type  |
| ---------- | ------------------------------------------------------------ | ---------- |
| double     |                                                              | double     |
| Float      |                                                              | float      |
| int32      | 使用变长编码，对于负值的效率很低，如果字段有可能是负值，请使用sint64代替 | int        |
| uint32     | 使用变长编码                                                 | int        |
| uint64     | 使用变长编码                                                 | long       |
| sint32     | 使用变长编码，这些编码在负值时比int32高效得多                | int        |
| sint64     | 使用变长编码，有符号的整型值。编码时比通常的int64高效        | long       |
| fixed32    | 总是4个字节，如果数值总是比2<sup>28</sup>大的话，这个类型会比unit32高效 | int        |
| fixed64    | 总是8个字节，如果数值总是比2<sup>56</sup>大的话，这个类型会比unit64高效 | long       |
| sfixed32   | 总是4个字节                                                  | int        |
| sfixed64   | 总是8个字节                                                  | long       |
| Bool       |                                                              | boolean    |
| String     | 一个字符串必须是UTF-8编码或者7-bit ASCII编码的文本           | String     |
| Bytes      | 可能包含任意顺序的字节数据                                   | ByteString |

​	**变长编码的类型表示打包的字节并不是固定，而是根据数据的带下或者长度来定。**例如int32，如果数值比较小，在0~127时，使用一个字节打包。

​	<u>关于fixed32和int32的区别。**fixed32的打包效率比int32的效率高，但是使用的空间一般比int32多。因此一个属于空间效率高**。根据项目的实际情况，一般选择fixed32，如果遇到对传输数据量要求比较苛刻的环境，则可以选择int32。</u>

#### 8.5.4 其他的语法规范

1. import声明

   ​	在需要多个消息结构体时，".proto"文件可以像Java语言的类文件一样分离多个，在需要的时候通过import导入需要的文件。导入的操作和Java的import操作大致相同。

2. 嵌套消息

   ​	“.proto”文件支持嵌套消息，消息中可以包含另一个消息作为其字段，也可以在消息中定义一个新的消息。

   ```protobuf
   message Outer{				// Level 0
       message MiddleA {		// Level 1
           message Inner{		// Level 2
               int64 ival = 1;
               bool booly = 2;
           }
       }
       message MiddleB {		// Level 1
           message Inner{		// Level 2
               int32 ival = 1;
               bool booly = 2;
           }
       }
   }
   ```

   ​	如果你想在它的父消息类型的外部重复使用这个内部的消息类型，可以使用Parent.Type的形式来使用它，例如：

   ```protobuf
   message SomeOtherMessage{
   	Outer.MiddlerA.Inner ref = 1;
   }
   ```

3. enum枚举

   ​	**枚举的定义和Java相同，但是有一些限制。枚举值必须大于等于0的整数**。使用分号(；)分隔枚举变量，而不是Java语言中的逗号“，”。

   ```java
   enum VoipProtocil{
       H323 = 1;
       SIP = 2;
       MGCP = 3;
       H248 = 4;
   }
   ```

### 8.6 本章小结

​	**JSON格式是直观的文本序列化方式，在实际的开发中，尤其是基于RESTful进行远程交互的应用开发中使用的非常多。一般来说，在实际开发中使用较多的JSON开发包是阿里的FastJson、谷歌的Gson。**

​	**Protobuf格式是非直观的二进制序列化方式，效率比较高，主要用于高性能的通信开发。**

## 第9章 基于Netty的单体IM系统的开发实践

​	本章是Netty应用的综合实践篇：将综合使用前面学到的编码器、解码器、业务处理器等知识，完成一个聊天系统的设计和实现。

### 9.1 自定义Protobuf编解码器

​	前面已经详细介绍过：在Netty中，内置了一组Protobuf编/解码器——ProtobufDecoder解码器和ProtobufEncoder编码器，它们负责Protobuf POJO和二进制字节之间的编码和解码。除此之外，Netty还自带了一组配套的Protobuf半包处理器：可变长度ProtobufVarint32FrameDecoder解码器、ProtobufVarint32LengthFieldPrepender编码器。为二进制ByteBuf加上varint32格式的可变长度，解决了Protobuf传输过程中的粘包/半包问题。

​	<u>使用Netty内置的Protobuf系列编解码器，可以解决简单的Protobuf协议的传输问题，不过，面对复杂Head-Content协议的解析，例如，在数据包Head部分，加上魔数、版本号字段，内置Protobuf系列编解码器，就显得无能为力了</u>。

​	**在通信数据包中，魔数可以理解为通信的口令**。例如，在电影《智取威虎山》中，土匪内部使用暗号接头，魔数和土匪的接头暗号在原理上是一样的。<u>无论是服务器端还是客户端，通信之前首先是对口令，如果口令不对，就不是符合自定义协议规范的数据包，也不是安全的数据包。通过魔数的对比，服务器端能够在第一时间识别出不符合规范的数据包，为了安全考虑，可以直接关闭连接</u>。

​	**在通信数据包中，版本号是什么呢？如果在程序中有协议升级的需求，又需要同时兼顾新旧版本的协议，就会用到这个版本号。例如：APP协议升级后，旧版本还需要使用**。

​	<u>解析复杂的Head-Content协议就需要自定义Protobuf编/解码器，需要开发者自己去解决半包问题</u>，包括以下两个方面：

​	（1）继承Netty提供的MessageToByteEncoder编码器，完后Head-Content协议的复杂数据包的编码，将Protobuf POJO编码成Head-Content协议的二进制ByteBuf数据包。

​	（2）继承Netty提供的ByteToMessageDecoder解码器，完成Head-Content协议的复杂数据包的解码，将二进制ByteBuf数据包最终解码出Protobuf POJO实例。

#### 9.1.1 自定义Protobuf编码器

​	自定义Protobuf编码器，通过继承Netty中基础的MessageToByteEncoder编码器类，实现其抽象的编码方法encode，在该方法中把以下内容写入到目标Bytebuf：

+ 写入Protobuf的POJO的字节码长。
+ 写入其他的字段，如魔数、版本号。
+ 写入Protobuf的POJO的字节码内容。

​	按照上面的步骤，自定义一个ProtobufEncoder，代码如下：

```java
/**
 * 编码器
 */
@Slf4j
public class ProtobufEncoder extends MessageToByteEncoder<ProtoMsg.Message>
{

    @Override
    protected void encode(ChannelHandlerContext ctx,
                          ProtoMsg.Message msg, ByteBuf out)
            throws Exception
    {

        byte[] bytes = msg.toByteArray();// 将对象转换为byte

        // 加密消息体
        /*ThreeDES des = channel.channel().attr(AppAttrKeys.ENCRYPT).get();
        byte[] encryptByte = des.encrypt(bytes);*/
        int length = bytes.length;// 读取消息的长度

        ByteBuf buf = ctx.alloc().buffer(2 + length);

        // 先将消息长度写入，也就是消息头
        buf.writeShort(length);
        // 消息体中包含我们要发送的数据
        buf.writeBytes(bytes);
        out.writeBytes(buf);

        log.debug("send "
                + "[remote ip:" + ctx.channel().remoteAddress()
                + "][total length:" + length
                + "][bare length:" + msg.getSerializedSize() + "]");

        if(buf.refCnt()>0)
        {
            log.debug("释放临时缓冲");
            buf.release();
        }

    }

}
```

​	*注意，这里写入的消息长度，调用了writeShort(length)方法，仅仅两个字节。这就存在一个问题，数据包的最大的净内容长度为32767个字节（有符号的短整型）。如果一个数据包需要传输更多的内容，可以调用writeInt(length)方法。*

#### 9.1.2 自定义Protobuf解码器

​	自定义Protobuf解码器，通过继承Netty中基础的ByteToMessageDecoder解码器类，将ByteBuf字节码解码成Protobuf的POJO对象，大致的过程如下：

​	（1）首先读取长度，如果长度位数不够，则终止读取。

​	（2）然后读取魔数、版本号等其他的字段。

​	（3）最后按照净长度读取内容。如果内容的字节数不够，则恢复到之前的起始位置（也就是长度的位置），然后终止读取。

​	自定义Protobuf解码器，如下所示：

```java
/**
 * 解码器
 */

@Slf4j
public class ProtobufDecoder extends ByteToMessageDecoder
{

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in,
                          List<Object> out) throws Exception
    {
        // 标记一下当前的readIndex的位置
        in.markReaderIndex();
        // 判断包头长度
        if (in.readableBytes() < 2)
        {// 不够包头
            return;
        }

        // 读取传送过来的消息的长度。
        int length = in.readUnsignedShort();

        // 长度如果小于0
        if (length < 0)
        {// 非法数据，关闭连接
            ctx.close();
        }

        if (length > in.readableBytes())
        {// 读到的消息体长度如果小于传送过来的消息长度
            // 重置读取位置
            in.resetReaderIndex();
            return;
        }


        byte[] array ;
        int offset=0;
        if (in.hasArray())
        {
            //堆缓冲
//            offset = in.arrayOffset() + in.readerIndex();
            ByteBuf slice=in.slice();
            array=slice.array();
        }
        else
        {
            //直接缓冲
            array = new byte[length];
            in.readBytes( array, 0, length);
        }

        // 字节转成Protobuf对象的POJO对象
        ProtoMsg.Message outmsg =
                ProtoMsg.Message.parseFrom(array);


        if (outmsg != null)
        {
            // 获取业务消息
            out.add(outmsg);
        }

    }
}
```

​	<u>在定义的解码过程中，如果需要进行版本号或者魔数的校验，也是非常简单的：只需读相应的字节数，进行值得比较即可</u>。

#### 9.1.3 IM系统中Protobuf消息格式的设计

> [Protobuf 的 proto3 与 proto2 的区别](https://www.cnblogs.com/jhj117/p/6202688.html)

​	一般来说，网络通信涉及消息的定义，不管是直接使用二进制格式，还是XML、JSON等字符串格式，大体都可以分为<u>3大消息类型</u>：

​	**（1）请求消息**

​	**（2）应答消息**

​	**（3）命令消息**

​	**每个消息基本上会包含一个序列号和一个类型定义。序列号用来唯一区分一个消息，类型用来决定消息的处理方式。**

​	Protobuf消息格式，大致有以下几个可供参考的原则：

​	**原则一：消息类型使用enum定义**

​	在“.proto”协议文件中，可以定义一个HeadType枚举类型，包含系统用到的所有消息类型，具体的例子如下：

```protobuf
enum HeadType {
    LOGIN_REQUEST = 0;			//登录请求
    LOGIN_RESPONSE = 1;			//登录响应
    LOGOUT_REQUEST = 2;			//登出请求
    LOGOUT_RESPONSE = 3;		//登出响应
    KEEPALIVE_REQUEST = 4;		//心跳请求
    KEEPALIVE_RESPONSE = 5;		//心跳响应
    MESSAGE_REQUEST = 6;		//聊天信息请求
    MESSAGE_RESPONSE = 7;		//聊天消息响应
    MESSAGE_NOTIFICATION = 8;	//服务器通知
}
```

​	**原则二：使用一个Protobufmessage结构定义一类信息**

​	例如，对应于登录请求类型——LOGIN_REQUEST类型的消息，Protobuf message消息结构如下：

```protobuf
/*登录请求信息*/
message LoginRequest {
     string uid = 1;			// 用户唯一IO
     string deviceId = 2;		// 设备ID
     string token = 3;			// 用户token
     uint32 platform = 4;		// 客户端平台 windows、mac、android、ios、web
     string app_version = 5;	// APP版本号
}
```

​	**原则三：建议给应答消息加上成功标记和应答序号**

​	对于应答消息并非总是成功的，因此建议在应答消息中加上两个字段：成功标记和应答序号。

​	成功标记是一个用于描述应答是否成功的标记，建议是哟个bool类型，true表示发送成功，false表示发送失败。

​	建议设置info字段的类型为字符串，用于放置失败时的提示信息。

​	**应答序号的作用是什么呢？如果一个请求有多个响应，则发送端可以设计为：每一个响应消息可以包含一个应答的序号，最后一个响应消息包含一个结束标记。接收端在处理的时候，根据应答序号和结束标记，可以合并所有的响应消息。**

​	对于聊天响应类型——MESSAGE_RESPONSE类型的消息，Protobufmessage消息结构如下：

```protobuf
/* 聊天响应 */
message MessageResponse {
    bool result = 1;            // true表示发送成功，false表示发送失败
    uint32 code = 2;            // 错误码
    string info = 3;            // 错误描述
    uint32 expose = 4;          // 错误描述是否提示给用户：1 提示；0不提示
    bool last_block = 5;        // 是否为最后的应答
    fixed32 block_index = 6;    // 应答的序号
}
```

​	**原则四：编解码从顶层消息开始**

​	建议定义一个外层的消息，把所有的消息类型全部封装在一起。在通信的时候，可以从外层消息开始编码或者解码。

​	对应于聊天器中的外层消息，Protobuf message消息结构大致如下：

```protobuf
message Message {
    HeadType type = 1;                      // 消息类型
    uint64 sequence = 2;                    // 序列号
    string session_id = 3;                  // 会话ID
    LoginRequest loginRequest = 4;          // 登录请求
    LoginResponse loginResponse = 5;        // 登录响应
    MessageRequest messageRequest = 6;      // 聊天请求
    MessageResponse messageResponse = 7;    // 聊天响应
    MessageNotification notification = 8;   // 通知消息
}
```

​	什么是序列号（sequence）呢？<u>序列号主要用于请求数据包Request和响应数据包Response的配套，Response的值必须和Request相同，使得发送端可以进行请求-响应的匹配处理</u>。

​	下面是一份完整的聊天器的”.proto“协议文件的定义：

```protobuf
// [开始声明]
syntax = "proto3";
//定义protobuf的包名称空间
package protocol;
// [结束声明]

// [开始 java 选项配置]
option java_package = "chatcommon.im.common.bean.msg";
option java_outer_classname = "ProtoMsg";
// [结束 java 选项配置]

// [开始 消息定义]

enum HeadType {
    LOGIN_REQUEST = 0; //登录请求
    LOGIN_RESPONSE = 1; //登录响应
    LOGOUT_REQUEST = 2; //登出请求
    LOGOUT_RESPONSE = 3; //登出响应
    KEEPALIVE_REQUEST = 4; //心跳请求
    KEEPALIVE_RESPONSE = 5; //心跳响应
    MESSAGE_REQUEST = 6; //聊天信息请求
    MESSAGE_RESPONSE = 7; //聊天消息响应
    MESSAGE_NOTIFICATION = 8; //服务器通知
}

/*登录请求信息*/
message LoginRequest {
    string uid = 1; // 用户唯一IO
    string deviceId = 2; // 设备ID
    string token = 3; // 用户token
    uint32 platform = 4; // 客户端平台 windows、mac、android、ios、web
    string app_version = 5; // APP版本号
}

message LoginResponse {
    bool result = 1;
    uint32 code = 2;
    string info = 3;
    uint32 expose = 4;
}


message MessageRequest {
    uint64 msg_id = 1;
    string from = 2;
    string to = 3;
    uint64 time = 4;
    uint32 msg_type = 5;
    string content = 6;
    string url = 8;
    string property = 9;
    string from_nick = 10;
    string json = 11;
}

/* 聊天响应 */
message MessageResponse {
    bool result = 1; // true表示发送成功，false表示发送失败
    uint32 code = 2; // 错误码
    string info = 3; // 错误描述
    uint32 expose = 4; // 错误描述是否提示给用户：1 提示；0不提示
    bool last_block = 5; // 是否为最后的应答
    fixed32 block_index = 6; // 应答的序号
}


message MessageNotification {
    uint32 msg_type = 1;
    bytes sender = 2;
    string json = 3;
    string timestamp = 4;
}


message Message {
    HeadType type = 1;                      // 消息类型
    uint64 sequence = 2;                    // 序列号
    string session_id = 3;                  // 会话IO
    LoginRequest loginRequest = 4;          // 登录请求
    LoginResponse loginResponse = 5;        // 登录响应
    MessageRequest messageRequest = 6;      // 聊天请求
    MessageResponse messageResponse = 7;    // 聊天响应
    MessageNotification notification = 8;   // 通知消息
}

// [结束 消息定义]
```

​	可以使用Maven插件生成对应的Protobuf Builder和POJO类，以供后续使用。

### 9.2 概述IM的登录流程

​	单体IM系统中，首先需要登录。登录的流程，从端到端（End to End）的角度来说，包括以下环节：

1. 客户端发送登录数据包。
2. 服务器端进行用户信息验证。
3. 服务器端创建Session会话。
4. 服务器端返回登录结束结果的信息给客户端，包括成功标志、Session ID等。

​	在端到端（End to End）的流程中总计需要完成4次编/解码：

+ 客户端编码：编码登录请求的Protobuf数据包编码。
+ 服务器端解码：解码登录请求的Protobuf数据包解码。
+ 服务器端编码：编码登录响应的Protobuf数据包编码。
+ 客户端解码：解码登录响应的Protobuf数据包解码。

#### 9.2.1 图解登录/响应流程的9个环节

​	从细分的角度来看，整个登录/响应的流程大概包含9个环节。

​	控制台命令 -> 发送器 -> 客户端通道 -> 服务端子通道 -> 登录响应handler -> 登录请求业务处理器 -> 服务端子通道 -> 客户端通道 -> 登录响应handler

​	从客户端到服务器端再到客户端，9个环节的介绍如下：

（1）首先，客户端手机用户ID和密码，这一步需要使用到LoginConsoleCommand控制台命令类。

（2）然后，客户端发送Protobuf数据包到客户端通道，这一步需要通过LoginSender发送器组装Protobuf数据包。

（3）客户端通道将Protobuf数据包发送到对端，这一步需要通过Netty底层来完成。

（4）服务器子通道接收到Protobuf数据包，这一步需要通过Netty底层来完成。

（5）服务端UserLoginHandler入站处理器收到登录消息，交给业务处理器LoginMsgProcesser处理异步的业务逻辑。

（6）服务端LoginMsgProcesser处理完异步的业务逻辑，就将处理结果写入用户绑定的子通道。

（7）服务器子通道将登录响应的Protobuf数据帧写入到对端，这一步需要通过Netty底层来完成。

（8）客户端通道收到Protobuf登录响应数据包，这一步需要通过Netty底层来完成。

（9）客户端LoginResponceHandler业务处理器处理登录响应，例如设置登录的状态，保存会话的Session ID等等。

#### 9.2.2 客户端涉及的主要模块

​	在具体的开发实践中，客户端所涉及的主要模块大致如下：

+ ClientCommand模块：控制台命令收集器。
+ ProtobufBuilder模块：Protobuf数据包构造者。
+ Sender模块：数据包发送器。
+ Handler模块：服务器响应处理器。

​	在具体的实现中，上面的这些模块都有一个或者多个专门的POJO Java类来完成对应的工作：

+ LoginConsoleCommand类：属于ClientCommand模块，它负责收集用户在控制台输入的用户ID和密码。
+ CommandController类：属于ClientCommand模块，它负责收集用户在控制台输入的命令类型，根据相应的类型调用相应的命令处理器和收集相应的信息。例如，如果用户输入的命令类型为登录，则调用LoginConsoleCommand命令处理器，将收集到的用户ID和密码封装成User类，然后启动登录处理。
+ LoginMsgBuilder类：属于ProtobufBuilder模块，它负责将User类组装成Protobuf登录请求数据包。
+ LoginSender类：属于Sender模块，它负责将组装好Protobuf登录数据包发送到服务器端。
+ LoginResponseHandler：属于Handler模块，它负责处理服务器端的登录响应。

#### 9.2.3 服务器端涉及的主要模块

​	在具体的开发实践中，服务器端所涉及的主要模块如下：

+ Handler模块：客户端请求的处理。
+ Processer模块：以异步方式完成请求的业务逻辑处理。
+ Session模块：管理用户与通道的绑定关系。

​	在具体的服务器登录流程中，上面的这些模块都有一个或者多个专门的Java类来完成对应的工作。在服务器端的登录流程中，上面模块中对应的类为：

+ UserLoginRequestHandler类：属于Handler模块，负责处理收到的Protobuf登录请求包，然后使用LoginProcesser类，以异步方式进行用户校验。
+ LoginProcesser类：属于Processer模块，完成服务器端的用户校验，再将校验的结果组装成一个登录响应Protobuf数据包写回到客户端。
+ ServerSession类：属于Session模块，如果校验成功，设置相应的会话状态（isLogin=true;）；然后，将会话加入到服务器端的SessionMap映射中，可以接受其他用户发送的聊天信息。

​	这里有一个疑问：为什么在服务器端登录处理需要分成两个模块，一个模块是Handler业务处理器，另一个模块是Processer以异步方式完成请求的业务逻辑处理？而不是像客户端一样，在Netty的Handler入站处理器模块中，统一完成业务的处理逻辑呢？

​	具体答案，稍后揭晓。

### 9.3 客户端的登录处理的实践案例

 	在输入登录信息之前，用户所选择的菜单是登录的选项。最开始的时候，客户端通过ClientCommandMenu菜单展示类展示出一个命令菜单，以供用户选择。效果如下：

```java
// ... 省略其他的输出
客户端开始连接
服务器连接成功
请输入某个操作指令：
[menu] 0->show 所有命令 | 1->登录 | 2->聊天 | 10->退出 |
// ... 省略其他的输出
```

​	从上面的输出可以看出，ClientCommandMenu菜单展示类展示了4个选项：

（1）登录

（2）聊天

（3）退出

（4）查看全部命令

​	每一个菜单项都对应到一个信息的收集类：

（1）聊天命令的信息收集类：ChatConsoleCommand

（2）登录命令的信息收集类：LoginConsoleCommand

（3）登出命令的信息收集类：LogoutConsoleCommand

（4）命令的类型收集类：ClientCommandMenu

​	以上4个客户端主要的命令收集类都组合在CommandClient类中。CommandClient类代表了整个客户端。

​	当用户输入的命令为“1”（表示登录），CommandClient类会通过LoginConsoleCommand类完成用户ID和密码的收集。

#### 9.3.1 LoginConsoleCommand和User POJO

​	登录命令和信息收集类LoginConsoleCommand负责从Scanner控制台实例收集客户端登录的用户ID和密码，代码如下：

```java
// ...
@Data
@Service("LoginConsoleCommand")
public class LoginConsoleCommand implements BaseCommand {
    public static final String KEY = "1";

    private String userName;
    private String password;

    @Override
    public void exec(Scanner scanner) {
        System.out.println("请输入用户信息（id:password）：");
        String[] info = null;
        while (true) {
            String input = scanner.next();
            info = input.split(":");
            if (info.length != 2) {
                System.out.println("请按照格式输入（id:password）：");
            } else {
                break;
            }
        }
        userName = info[0];
        password = info[1];
    }

    @Override
    public String getKey(){
        return KEY;
    }

    @Override
    public String getTip(){
        return "登录";
    }
}
```

​	成功获取到用户密码和ID后，客户端CommandClient将这些内容组装成User POJO用户对象，然后通过客户端的发送器LoginSender开始向服务器端发送登录请求。

```java
@Service("CommandClient")
public class CommandClient {
    //...
    //命令收集线程
    public void startCommandThread() throws InterruptedException {
        Thread.currentThread().setName("主线程");
        while (true) {
            //建立连接
            while (connectFlag == false) {
                // 开始连接
                startConnectServer();
                waitCommandThread();
            }

            //处理命令
            while (null != session && session.isConnected()) {
                Scanner scanner = new Scanner(System.in);
                clientCommandMenu.exec(scanner);
                String key = clientCommandMenu.getCommandInput();
                //取到命令收集类POJO
                BaseCommand command = commandMap.get(key);
                switch (key) {
                    //登录的命令
                    case LoginConsoleCommand.KEY:
                        command.exec(scanner);
                        startLogin((LoginConsoleCommand) command);
                        break;

                    // ... 其他命令
                }
            }
        }
    }

    // 开始发送登录请求
    private void startLogin(LoginConsoleCommand command) {
        //登录
        if (!isConnectFlag()) {
            log.info("连接异常，请重新建立连接");
            return;
        }
        User user = new User();
        user.serUid(command.getUserName());
        user.serToken(command.getPassword());
        user.setDevId("1111");
        loginSender.setUser(user);
        loginSender.setSession(session);
        loginSender.sendLoginMsg();
    }
    // ... 
}
```

#### 9.3.2 LoginSender发送器

​	LoginSender消息发送器的sendLoginMsg方法用于在第一步生成Protobuf登录数据包，在第二步则是调用BaseSender基类的sendMsg来发送数据包。

```java
@Slf4j
@Service("LoginSender")
public class LoginSender extends BaseSender {
    
    public void sendLoginMsg() {
        if (!isConnected()) {
            log.info("还没有建立连接!");
            return;
        }
        log.info("构造登录消息");
        ProtoMsg.Message message =
                LoginMsgBuilder.buildLoginMsg(getUser(), getSession());
        log.info("发送登录消息");
        super.sendMsg(message);
    }
}
```

​	首先使用LoginMsgBuilder消息构造者来构造一个登录请求的Protobuf消息。这一步比较简单，大家直接看源码即可。

​	然后调用基类的sendMsg方法来发送登录消息。

```java
@Data
@Slf4j
public abstract class BaseSender {

    private User user;
    private ClientSession session;

    public boolean isConnected() {
        if (null == session) {
            log.info("session is null");
            return false;
        }
        return session.isConnected();
    }
    
    public boolean isLogin() {
        if (null == session) {
            log.info("session is null");
            return false;
        }
        return session.isLogin();
    }

    public void sendMsg(ProtoMsg.Message message) { 
        if (null == getSession() || !isConnected()) {
            log.info("连接还没成功");
            return;
        }
        Channel channel = getSession().getChannel();
        ChannelFuture f = channel.writeAndFlush(message);
        f.addListener(new GenericFutureListener<Future<? super Void>>() {
            @Override
            public void operationComplete(Future<? super Void> future)
                    throws Exception {
                // 回调
                if (future.isSuccess()) {
                    sendSucced(message);
                } else {
                    sendfailed(message);
                }
            }
        });
    }
    
    protected void sendSucced(ProtoMsg.Message message) {
        log.info("发送成功");

    }

    protected void sendfailed(ProtoMsg.Message message) {
        log.info("发送失败");
    }
}
```

​	一般来说，在Netty中会调用write(pkg)/writeAndFlush(pkg)这一组方法来发送数据包。前面多次反复讲到，<u>发送方法channel.writeAndFlush(pkg)是立即返回的，返回的类型是一个ChannelFuture异步任务实例。</u>问题是：当channel.writeAndFlush(pkg)函数返回时，如何判断是否已经成功将数据包写入到了底层的TCP连接呢？

​	答案是没有。在writeAndFlush(pkg)方法返回时，真正的TCP写入的操作其实还没有执行。为什么呢？

​	在Netty中，无论是入站操作还是出站操作，都有两大特点：

1. ​	**同一条通道的所有出入/站处理都是串行的，而不是并行的**。换句话说，<u>同一条通道上的所有出/入站处理都会在它所绑定的EventLoop线程上执行</u>。既然只有一个线程负责，那就只有串行的可能。

   ​	假如某个出/入站的处理在最开始执行的时候，会对当前的执行线程进行判断：如果当前线程不是普通的EventLoop线程，则当前出/入站的处理暂时不执行；**Netty会将当前出/入站的处理，通过建立一个新的异步可执行任务，加入到通道的EventLoop线程的任务队列中**。

   ​	EventLoop线程的任务队列是一个<u>MPSC队列（即多生产者单消费者队列）</u>。什么是MPSC队列呢？只有EventLoop线程自己是唯一的消费者，它将遍历任务队列，逐个执行任务；其他线程只能作为生产者，它们的出/入站操作都会作为异步任务加入到任务队列。**通过MPSC队列，确保了EventLoop线程能做到：同一个通道上所有的IO操作是串行的，不是并行的。这样，不同的Handler业务处理器之间不需要进行线程的同步，这点也能大大提升IO的性能**。

2. ​	**Netty的一个出/入站操作不是一次的单一Handler业务处理器操作，而是流水线上的一系列的出/入站处理流程。只有整个流程都处理完，出/入站操作才真正处理完成**。

​	基于以上两点，大家可以简单地推断，在调用完channel.writeAndFlush(pkg)后，真正的出栈操作肯定是没有执行完成的，可能还需要在EventLoop的任务队列中排队等待。

​	如何才能判断writeAndFlush()执行完毕了呢？writeAndFlush()方法会返回一个ChannelFuture异步任务实例，通过为ChannelFuture异步任务增加GenericFutureListener监听器的方式来判断writeAndFlush()是否已经执行完毕。<u>当GenericFutureListener监听器的operationComplete方法被回调时，表示writeAndFlush()方法已经执行完毕了。而具体的回调业务逻辑，可以放在operationComplete监听器的方法中</u>。

​	在上面的代码中，设计了两个sendSucced/sendfailed业务回调方法，放置在operationComplete监听器方法中，在发送完成后进行回调，并且将sendSucced和sendfailed方法封装在发送器的BaseSender基类中。如果需要改变默认的回调处理逻辑，可以在发送子类重写基类的sendSucced和sendfailed方法即可。

​	再看另外一个话题：在上面的代码中，为获取客户端的通道，使用了ClientSession客户端会话。什么是会话呢？会话的作用是什么呢？什么时候创建会话呢？预知后事，请看下回。

#### 9.3.3 ClientSession客户端会话

​	ClientSession是一个很重要的胶水类，有两个成员：一个是user，代表用户，另一个是channel，代表了连接的通道。在实际开发中，这两个成员的作用是：

1. 通过user，可以获得当前的用户信息。
2. 通过channel，可以向服务器端发送信息。

​	ClientSession会话“左拥右抱”，左手“拥有”用户信息，右手“抱有”服务器端的连接。通过user成员可以获取当前的用户信息。借助channel通道，ClientSession可以写入Protobuf数据包，或者关闭Netty连接。

​	其次，客户端会话ClientSession保存着当前的状态：

​	（1）是否成功连接isConnected

​	（2）是否成功登录isLogin

​	第三，ClientSession绑定在通道上，因而可以在入站处理器中通过通道反向取得绑定的ClientSession。

```java
@Slf4j
@Data
public class ClientSession {

    public static final AttributeKey<ClientSession> SESSION_KEY =
            AttributeKey.valueOf("SESSION_KEY");

    /**
     * 用户实现客户端会话管理的核心
     */
    private Channel channel;
    private User user;

    /**
     * 保存登录后的服务端sessionid
     */
    private String sessionId;

    private boolean isConnected = false;
    private boolean isLogin = false;

    /**
     * session中存储的session 变量属性值
     */
    private Map<String, Object> map = new HashMap<String, Object>();

    //绑定通道
    public ClientSession(Channel channel) {
        this.channel = channel;
        this.sessionId = String.valueOf(-1);
        channel.attr(ClientSession.SESSION_KEY).set(this);
    }

    //登录成功之后,设置sessionId
    public static void loginSuccess(
            ChannelHandlerContext ctx, ProtoMsg.Message pkg) {
        Channel channel = ctx.channel();
        ClientSession session = channel.attr(ClientSession.SESSION_KEY).get();
        session.setSessionId(pkg.getSessionId());
        session.setLogin(true);
        log.info("登录成功");
    }

    //获取channel
    public static ClientSession getSession(ChannelHandlerContext ctx) {
        Channel channel = ctx.channel();
        ClientSession session = channel.attr(ClientSession.SESSION_KEY).get();
        return session;
    }

    public String getRemoteAddress() {
        return channel.remoteAddress().toString();
    }

    //写protobuf 数据帧
    public ChannelFuture witeAndFlush(Object pkg) {
        ChannelFuture f = channel.writeAndFlush(pkg);
        return f;
    }

    public void writeAndClose(Object pkg) {
        ChannelFuture future = channel.writeAndFlush(pkg);
        future.addListener(ChannelFutureListener.CLOSE);
    }

    //关闭通道
    public void close() {
        isConnected = false;

        ChannelFuture future = channel.close();
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (future.isSuccess()) {
                    log.error("连接顺利断开");
                }
            }
        });
    }
}
```

​	什么时候创建客户端会话呢？在Netty客户端连接建立之后，并增加一个监听异步任务完成的监听器，代码如下：

```java
@Slf4j
@Data
@Service("CommandController")
public class CommandController {
    // ... 
    GenericFutureListener<ChannelFuture> connectedListener = (ChannelFuture f) ->
    {
        final EventLoop eventLoop
                = f.channel().eventLoop();
        if (!f.isSuccess()) {
            log.info("连接失败!在10s之后准备尝试重连!");
            eventLoop.schedule(
                    () -> nettyClient.doConnect(),
                    10,
                    TimeUnit.SECONDS);

            connectFlag = false;
        } else {
            connectFlag = true;

            log.info("疯狂创客圈 IM 服务器 连接成功!");
            channel = f.channel();

            // 创建会话
            session = new ClientSession(channel);
            session.setConnected(true);
            channel.closeFuture().addListener(closeListener);

            //唤醒用户线程
            notifyCommandThread();
        }

    };
    // ...
}
```

#### 9.3.4 LoginResponseHandler登录相应处理器

​	LoginResponseHandler登录响应处理器对消息类型进行判断：

​	（1）如果消息类型是请求响应消息并且登录成功，则取出绑定的会话（Session），再设置登录成功的状态。在完成登录称后之后，进行其他的客户端业务处理。

​	（2）如果消息类型不是请求响应数据，则调用父类默认的super.channelRead()入站处理方法，将数据包交给流水线的下一站Handler业务处理器去处理。

```java
@Slf4j
@ChannelHandler.Sharable
@Service("LoginResponceHandler")
public class LoginResponceHandler extends ChannelInboundHandlerAdapter {
    /**
     * 业务逻辑处理
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        //判断消息实例
        if (null == msg || !(msg instanceof ProtoMsg.Message)) {
            super.channelRead(ctx, msg);
            return;
        }

        //判断类型
        ProtoMsg.Message pkg = (ProtoMsg.Message) msg;
        ProtoMsg.HeadType headType = ((ProtoMsg.Message) msg).getType();
        if (!headType.equals(ProtoMsg.HeadType.LOGIN_RESPONSE)) {
            super.channelRead(ctx, msg);
            return;
        }


        //判断返回是否成功
        ProtoMsg.LoginResponse info = pkg.getLoginResponse();

        ProtoInstant.ResultCodeEnum result =
                ProtoInstant.ResultCodeEnum.values()[info.getCode()];

        if (!result.equals(ProtoInstant.ResultCodeEnum.SUCCESS)) {
            //登录失败
            log.info(result.getDesc());
        } else {
            //登录成功
            ClientSession.loginSuccess(ctx, pkg);
            ChannelPipeline p = ctx.pipeline();
            //移除登录响应处理器
            p.remove(this);

            //在编码器后面，动态插入心跳处理器
            p.addAfter("encoder", "heartbeat", new HeartBeatClientHandler());
        }

    }

}
```

​	在的登录成功之后，需要将LoginResponseHandler登录响应处理器实例从流水线上移除，因为不需要再处理登陆响应了。同时，需要在客户端和服务器端之间开启定时的心跳处理。心跳是一个比较复杂的议题，后面会有单独的小节专门详细介绍客户端和服务器之间的心跳。

#### 9.3.5 客户端流水线的装配

​	对于客户端的业务处理器流水线（Pipeline），首先是需要装配一个ProtobufDecoder解码器和一个ProtobufEncoder编码器。编码器和解码器一般都是装配在最前面。然后需要装配业务处理器——LoginResponseHandler登录响应处理器的实例。

​	<u>一般来说，在流水线最末端还需要装配一个ExceptionHander异常处理器，它也是一个入站处理器，用来实现Netty异常的处理以及在连接异常中断后进行重连</u>。

```java
@Slf4j
@Data
@Service("NettyClient")
public class NettyClient {
    // 服务器ip地址
    @Value("${server.ip}")
    private String host;
    // 服务器端口
    @Value("${server.port}")
    private int port;


    @Autowired
    private ChatMsgHandler chatMsgHandler;

    @Autowired
    private LoginResponceHandler loginResponceHandler;


    @Autowired
    private ExceptionHandler exceptionHandler;


    private Channel channel;
    private ChatSender sender;
    private LoginSender l;

    /**
     * 唯一标记
     */
    private boolean initFalg = true;
    private User user;
    private GenericFutureListener<ChannelFuture> connectedListener;

    private Bootstrap b;
    private EventLoopGroup g;

    public NettyClient() {

        /**
         * 客户端的是Bootstrap，服务端的则是 ServerBootstrap。
         * 都是AbstractBootstrap的子类。
         **/

        /**
         * 通过nio方式来接收连接和处理连接
         */

        g = new NioEventLoopGroup();
    }

    /**
     * 重连
     */
    public void doConnect() {
        try {
            b = new Bootstrap();

            b.group(g);
            b.channel(NioSocketChannel.class);
            b.option(ChannelOption.SO_KEEPALIVE, true);
            b.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
            b.remoteAddress(host, port);

            // 设置通道初始化
            b.handler(
                    new ChannelInitializer<SocketChannel>() {
                        public void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast("decoder", new ProtobufDecoder());
                            ch.pipeline().addLast("encoder", new ProtobufEncoder());
                            ch.pipeline().addLast(loginResponceHandler);
                            ch.pipeline().addLast(chatMsgHandler);
                            ch.pipeline().addLast(exceptionHandler);
                        }
                    }
            );
            log.info("客户端开始连接 [疯狂创客圈IM]");

            ChannelFuture f = b.connect();
            f.addListener(connectedListener);


            // 阻塞
            // f.channel().closeFuture().sync();

        } catch (Exception e) {
            log.info("客户端连接失败!" + e.getMessage());
        }
    }

    public void close() {
        g.shutdownGracefully();
    }
}
```

​	关于业务处理器的执行次序，再强调一下：登录响应处理器，必须装配在ProtobufDecoder解码器之后。具体原因是：Netty客户端读到二进制Bytebuf数据包之后，首先需要通过protobufDecoder完成解码操作。解码后组装好Protobuf消息POJO，再进入loginResponseHandler登录响应处理器。

### 9.4 服务器端的登录响应的实践案例

​	服务器端的登录响应流程是：

​	（1）ProtobufDecoder解码器把请求ByteBuf数据包解码成Protobuf数据包。

​	（2）UserLoginRequestHandler登录处理器用于处理Protobuf数据包，进行一些必要的判断和预处理后，启动LoginProcesser登录业务处理器，开始以异步方式进行登录验证处理。

​	（3）LoginProcesser通过数据库或者远程接口完成用户验证，根据验证处理的结果，生成登录成功/或者失败的登录响应报文，并发送到客户端。

#### 9.4.1 服务器流水线的装配

​	与客户端一样，对于服务器端流水线的，首先需要装配一个ProtobufDecoder解码器和一个ProtobufEncoder编码器。然后需要装配loginRequestHandler登录业务处理器的实例。<u>在流水线的最末端，还需要装配一个serverExceptionHandler异常处理器实例。</u>

```java
@Data
@Slf4j
@Service("ChatServer")
public class ChatServer {
    // 服务器端口
    @Value("${server.port}")
    private int port;
    // 通过nio方式来接收连接和处理连接
    private EventLoopGroup bg =
            new NioEventLoopGroup();
    private EventLoopGroup wg =
            new NioEventLoopGroup();

    // 启动引导器
    private ServerBootstrap b =
            new ServerBootstrap();
    @Autowired
    private LoginRequestHandler loginRequestHandler;

    @Autowired
    private ServerExceptionHandler serverExceptionHandler;

    @Autowired
    private ChatRedirectHandler chatRedirectHandler;

    public void run() {
        try {   //1 设置reactor 线程
            b.group(bg, wg);
            //2 设置nio类型的channel
            b.channel(NioServerSocketChannel.class);
            //3 设置监听端口
            b.localAddress(new InetSocketAddress(port));
            //4 设置通道选项
            b.option(ChannelOption.SO_KEEPALIVE, true);
            b.option(ChannelOption.ALLOCATOR,
                    PooledByteBufAllocator.DEFAULT);

            //5 装配流水线
            b.childHandler(new ChannelInitializer<SocketChannel>() {
                //有连接到达时会创建一个channel
                protected void initChannel(SocketChannel ch) throws Exception {
                    // 管理pipeline中的Handler
                    ch.pipeline().addLast(new ProtobufDecoder());
                    ch.pipeline().addLast(new ProtobufEncoder());
                    ch.pipeline().addLast(new HeartBeatServerHandler());
                    // 在流水线中添加handler来处理登录,登录后删除
                    ch.pipeline().addLast(loginRequestHandler);
                    ch.pipeline().addLast(chatRedirectHandler);
                    ch.pipeline().addLast(serverExceptionHandler);
                }
            });
            // 6 开始绑定server
            // 通过调用sync同步方法阻塞直到绑定成功

            ChannelFuture channelFuture = b.bind().sync();
            log.info(
                    "疯狂创客圈 CrazyIM 服务启动, 端口 " +
                            channelFuture.channel().localAddress());
            // 7 监听通道关闭事件
            // 应用程序会一直等待，直到channel关闭
            ChannelFuture closeFuture =
                    channelFuture.channel().closeFuture();
            closeFuture.sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 8 优雅关闭EventLoopGroup，
            // 释放掉所有资源包括创建的线程
            wg.shutdownGracefully();
            bg.shutdownGracefully();
        }
    }
}
```

​	在服务器端的登录处理流程中，ProtobufDecoder解码器把登录请求的二进制ByteBuf数据包解码成Protobuf数据包，然后发送给下一站loginRequestHandler登录请求处理器，由loginRequestHandler登录请求处理器完成实际的登录处理。

#### 9.4.2 LoginRequestHandler登录请求处理器

​	这是个入站处理器，它继承自ChannelInboundHandlerAdapter入站适配器，重写了channelRead方法，主要的工作如下：
​	（1）对消息进行必要的判断：判断是否为登录请求Protobuf数据包。如果不是，通过super.channelRead(ctx,msg)将消息交给流水线的下一站。

​	（2）创建一个ServerSession，即为客户建立一个服务器端的会话。

​	（3）使用自定义的CallbackTaskScheduler异步任务调度器，提交一个异步任务，启动LoginProcesser执行登录用户验证逻辑。

```java
@Slf4j
@Service("LoginRequestHandler")
@ChannelHandler.Sharable
public class LoginRequestHandler extends ChannelInboundHandlerAdapter {

    @Autowired
    LoginProcesser loginProcesser;

    /**
     * 收到消息
     */
    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        if (null == msg
                || !(msg instanceof ProtoMsg.Message)) {
            super.channelRead(ctx, msg);
            return;
        }

        ProtoMsg.Message pkg = (ProtoMsg.Message) msg;

        //取得请求类型
        ProtoMsg.HeadType headType = pkg.getType();

        if (!headType.equals(loginProcesser.type())) {
            super.channelRead(ctx, msg);
            return;
        }


        ServerSession session = new ServerSession(ctx.channel());

        //异步任务，处理登录的逻辑
        CallbackTaskScheduler.add(new CallbackTask<Boolean>() {
            @Override
            public Boolean execute() throws Exception {
                boolean r = loginProcesser.action(session, pkg);
                return r;
            }

            //异步任务返回
            @Override
            public void onBack(Boolean r) {
                if (r) {
                    ctx.pipeline().remove(LoginRequestHandler.this);
                    log.info("登录成功:" + session.getUser());

                } else {
                    ServerSession.closeSession(ctx);
                    log.info("登录失败:" + session.getUser());
                }
            }
            //异步任务异常

            @Override
            public void onException(Throwable t) {
                ServerSession.closeSession(ctx);
                log.info("登录失败:" + session.getUser());
            }
        });

    }
}
```

#### 9.4.3 LoginProcesser用户验证逻辑

​	LoginProcesser用户验证逻辑包括：密码验证、将验证的结果写入到通道。如果登入验证成功，还需要绑定服务器端的会话，并且加入到在线用户列表中。

```java
@Slf4j
@Service("LoginProcesser")
public class LoginProcesser extends AbstractServerProcesser {
    @Autowired
    LoginResponceBuilder loginResponceBuilder;

    @Override
    public ProtoMsg.HeadType type() {
        return ProtoMsg.HeadType.LOGIN_REQUEST;
    }

    @Override
    public boolean action(ServerSession session,
                          ProtoMsg.Message proto) {
        // 取出token验证
        ProtoMsg.LoginRequest info = proto.getLoginRequest();
        long seqNo = proto.getSequence();

        User user = User.fromMsg(info);

        //检查用户
        boolean isValidUser = checkUser(user);
        if (!isValidUser) {
            ProtoInstant.ResultCodeEnum resultcode =
                    ProtoInstant.ResultCodeEnum.NO_TOKEN;
            //构造登录失败的报文
            ProtoMsg.Message response =
                    loginResponceBuilder.loginResponce(resultcode, seqNo, "-1");
            //发送登录失败的报文
            session.writeAndFlush(response);
            return false;
        }

        session.setUser(user);

        session.bind();

        //登录成功
        ProtoInstant.ResultCodeEnum resultcode =
                ProtoInstant.ResultCodeEnum.SUCCESS;
        //构造登录成功的报文
        ProtoMsg.Message response =
                loginResponceBuilder.loginResponce(
                        resultcode, seqNo, session.getSessionId());
        //发送登录成功的报文
        session.writeAndFlush(response);
        return true;
    }

    private boolean checkUser(User user) {

        if (SessionMap.inst().hasLogin(user)) {
            return false;
        }

        //校验用户,比较耗时的操作,需要100 ms以上的时间
        //方法1：调用远程用户restfull 校验服务
        //方法2：调用数据库接口校验

        return true;

    }

}
```

​	用户密码验证的逻辑，在checkUser()方法中完成。<u>在实际的生产场景中，LoginProcesser进行用户验证的方式比较多：</u>

+ 通过RESTful接口验证用户
+ 通过数据库去验证用户
+ 通过验证(Auth)服务器去验证用户

​	总之，验证用户是一个耗时间的操作。为了尽量地简化流程，示例代码省去了通过账号和密码验证地过程，而checkUser()方法直接返回true，也就是所有的登录都是成功的。

​	<u>服务器端校验通过之后，可以完成服务器端会话（Session）的绑定工作</u>。

​	服务器端的ServerSession会话也是一个胶水类，与客户端的ClientSession会话类似。<u>每一个ServerSession对应一个客户端连接。一个ServerSession拥有一个Channel成员实例、一个User成员实例。Channel成员代表与客户端连接的子通道；User成员代表用户信息。</u>稍后，会对ServerSession进行详细介绍。

​	<u>在用户校验成功之后，就需要向客户端发送登陆响应</u>。具体的方法是：调用登录响应的Protobuf消息构造器loginResponseBuilder，构造一个登录响应POJO，设置好校验成功的标志位，调用会话（Session）的writeAndFlush()方法写到客户端。

#### 9.4.4 EventLoop线程和业务线程相互隔离

​	前面章节中，已经埋下一个疑问：为什么在服务器整个登录处理需要分成两个模块：一个是Netty Handler业务处理器，另一个是Processer业务逻辑处理器；而不是像客户端一样，在入站处理器Handler中统一完成呢？

​	答案是：**在服务器端需要隔离EventLoop（Reactor）线程和业务线程**。基本的方法是，<u>使用独立的、异步的线程任务去执行用户验证的逻辑；而不在EventLoop线程中去执行用户验证的逻辑。</u>

​	实际上，Reactor反应器线程和业务线程相互隔离，在服务器端非常重要。

​	首先，以读通道channelRead为例，看下一次普通的入站处理的基本步骤：

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception{
    //1 判断消息是否需要处理
    //2 取得消息，并判断类型
    //3 耗时的业务处理操作
    //4 把结果写入到连接通道
}
```

​	其中第三步，通常会涉及到一些比较耗时的业务处理操作，例如：

​	（1）数据库操作，百毫秒级，一般查询都在100ms以上。

​	（2）远程接口调用，百毫秒级，一般都在200ms以上，慢的在1000ms以上。

​	再看Netty内部的IO读写操作，通常都是毫秒级。也就是说，**Netty内部的IO操作和业务处理操作在时间上不在一个数量级**。

​	问题来了：<u>在大量（成千上万）的子通道复用一个EventLoop线程的应用场景中，一旦耗时的业务处理操作也会执行在EventLoop线程上，就会导致其他子通道的IO操作会发生严重的性能问题。</u>

​	大家都知道，**默认情况下，Netty的一个EventLoop实例会开启2倍CPU内核数的内部线程**。通常情况下，一个Netty服务器端会有几万或者几十万的连接通道。也就是说，一个EventLoop内部线程会负责处理几万或者上十万个通道连接的IO处理。

​	**在一个EventLoop内部线程上任务是串行的**。如果一个Handler业务处理器中的channelRead()入站处理方法执行1000ms或者几秒钟，最终的结果是，阻塞了EventLoop内部线程其他几十万个通道的出站和入站处理，阻塞时长为1000ms或者几秒钟。而耗时的入站/出站处理越多，就越会拖慢整个线程的其他IO处理，最终导致严重的性能问题。

​	<u>解决方法是，业务操作和EventLoop线程相隔离。具体来说，就是专门开辟一个独立的线程池，负责一个独立的异步任务处理队列。对于耗时的业务操作封装成异步任务，并放入异步任务队列中去处理。这样的化，服务器端的性能会提升很多。</u>

### 9.5 详解ServerSession服务器会话

​	无论是客户端还是服务器端，为了让通道（Channel——连接和用户(User)的管理和使用变得方便，引入了一个非常重要的概念——会话(Session)）。有点类似Tomcat的服务器会话，只是在实现上比较简单。

​	<u>由于客户端和服务器分别都有各自的通道，并且相关的参数与一些也不一致，因此这里使用了两个会话类型：客户端会话ClientSession、服务器端会话ServerSession</u>。

​	**会话和通道直接的导航关系有两个方向，一个是正向导航：通过会话导航到通道，主要用于出站处理的场景，通过会话将数据包写出到通道；另一个是反向导航，通过通道导航到会话，主要用于入站处理的场景，通过通道获取会话，以便进一步进行业务处理。**

​	如果进行反向导航呢？需要用到**通道的容器属性**。

#### 9.5.1 通道的容器属性

​	<u>Netty中的Channel通道类，有类似于Map的容器功能，可以通过“key-value”键值对的形式来保存任何Java Object类型的值</u>。一般来说，可以存放一些与通道实例相关联的属性，比如说服务器端的ServerSession会话实例。

​	**除了Channel通道类，Netty中的HandlerContext处理器上下文类，也具备了类似的容器功能，可以绑定key-value键值对**。

​	问题是：Channel和HandlerContext的容器功能，具体是如何实现的呢？

​	Netty没有实现Map接口，而是定义了一个类似的接口，叫作AttributeMap，<u>它有且只有一个方法"\<T\> Attribute\<T\> attr(AttributeKey\<T\> key)；"，此方法接受一个AttributeKey类型的key，返回一个Attribute类型的值</u>。这里特别说明一下：

​	（1）这里的AttributeKey也不是原始的key（例如，放在Map中的key），而是一个key的包装类。AttributeKey确保了key的唯一性，在单个Netty应用中，key值必须唯一。

​	（2）这里的值Attribute不是原始的value，也是value的包装类。原始的value就放置在Attribute包装类中，可以通过Attribute包装类实现value的读取（get）和设置（set），取值和设值。

key -> {AttributeMap : AttribubteKey -> Attribute} -> value

​	在Netty中，接口AttributeMap的源代码如下：

```java
public interface AttributeMap {
    <T> Attribute<T> attr(AttributeKey<T> key);
}
```

​	AttributeMap只是一个接口，Netty提供了默认的实现。**AttributeMap的实现要求是线程安全的**。可以通过AttributeMap的attr(...)方法，根据key取得Attribute类型的value；然后通过Attribute类型的value实例完成最终的两个重要操作：设值（set）、取值（get）。

1. Attribute的设值

   ​	Attribute设值的方法，举例如下：

   ```java
   // 定义key
   public static final AttributeKey<ServerSession> SESSION_KEY = AttributeKey.valueOf("SESSION_KEY");
   // ...
   // 通过设置将会话绑定到通道
   channel.attr(SESSION_KEY).set(session);
   ```

   ​	AttributeKey的创建，需要用到静态方法AttributeKey.valueOf(String)方法。该方法的返回值为一个AttributeKey实例，它的泛型类型为实际上的key-value键值对value的实际类型。这里的value实际是ServerSession类型，所以返回值类型为AttributeKey\<ServerSession\>。

   ```java
   public static <T> AttributeKey<T> valueOf(String name) {
       ObjectUtil.checkNotNull(name, "name");
       AttributeKey<T> option = (AttributeKey)names.get(name);
       if (option == null) {
           option = new AttributeKey(name);
           AttributeKey<T> old = (AttributeKey)names.putIfAbsent(name, option);
           if (old != null) {
               option = old;
           }
       }
       return option;
   }
   ```

   ​	再强调一句：这里的AttributeKey一般定义为一个常量，需要提前定义：它的那个泛型的参数是Attribute最终的包装值value的数据类型。

   ```java
   // key 的泛型形参是设置的value的类型
   public static final AttributeKey<ServerSession> SESSION_KEY = AttributeKey.valueOf("SESSION_KEY");
   ```

2. Attribute取值

   ​	Attribute取值的方法，举例如下：

   ```java
   ServerSession session = ctx.channel().attr(SESSION_KEY).get();
   ```

   ​	这里使用了链式调用，首先通过通道的attr(AttributeKey)方法，取得键(key)所对应的值(value)的包装类Attribute实例。然后通过Attribute 的get()方法，获取真正的值——前面所设置的会话session实例。

#### 9.5.2 ServerSession服务器端会话类

​	<u>在登录成功之后，服务器端会为每一个新连接通道创建一个ServerSession案例，表示用户与服务器端的会话信息</u>。

​	**每个SerevrSession实例都有一个唯一标识，为sessionId。注意，唯一标识ServerSession实例的不是userid。为什么呢?主要原因是：同一个用户可从网页端、手机端、电脑桌桌面，同时登录IM服务器端。微信、QQ都是典型的案例。另外，同一个用户的消息需要在手机端、网页端、桌面端进行同步，也就是说同时接受消息、同时发送消息。**

```java
@Data
@Slf4j
public class ServerSession {

    public static final AttributeKey<String> KEY_USER_ID =
        AttributeKey.valueOf("key_user_id");

    public static final AttributeKey<ServerSession> SESSION_KEY =
        AttributeKey.valueOf("SESSION_KEY");

    /**
     * 用户实现服务端会话管理的核心
     */
    //通道
    private Channel channel;
    //用户
    private User user;

    //session唯一标示
    private final String sessionId;

    //登录状态
    private boolean isLogin = false;

    /**
     * session中存储的session 变量属性值
     */
    private Map<String, Object> map = new HashMap<String, Object>();

    public ServerSession(Channel channel) {
        this.channel = channel;
        this.sessionId = buildNewSessionId();
    }

    //反向导航
    public static ServerSession getSession(ChannelHandlerContext ctx) {
        Channel channel = ctx.channel();
        return channel.attr(ServerSession.SESSION_KEY).get();
    }

    //关闭连接
    public static void closeSession(ChannelHandlerContext ctx) {
        ServerSession session =
            ctx.channel().attr(ServerSession.SESSION_KEY).get();

        if (null != session && session.isValid()) {
            session.close();
            SessionMap.inst().removeSession(session.getSessionId());
        }
    }

    //和channel 通道实现双向绑定
    public ServerSession bind() {
        log.info(" ServerSession 绑定会话 " + channel.remoteAddress());
        channel.attr(ServerSession.SESSION_KEY).set(this);
        SessionMap.inst().addSession(getSessionId(), this);
        isLogin = true;
        return this;
    }

    public ServerSession unbind() {
        isLogin = false;
        SessionMap.inst().removeSession(getSessionId());
        this.close();
        return this;
    }

    public String getSessionId() {
        return sessionId;
    }

    private static String buildNewSessionId() {
        String uuid = UUID.randomUUID().toString();
        return uuid.replaceAll("-", "");
    }

    public synchronized void set(String key, Object value) {
        map.put(key, value);
    }

    public synchronized <T> T get(String key) {
        return (T) map.get(key);
    }

    public boolean isValid() {
        return getUser() != null ? true : false;
    }

    //写Protobuf数据帧
    public synchronized void writeAndFlush(Object pkg) {
        channel.writeAndFlush(pkg);
    }

    //关闭连接
    public synchronized void close() {
        ChannelFuture future = channel.close();
        future.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    log.error("CHANNEL_CLOSED error ");
                }
            }
        });
    }

    public User getUser() {
        return user;
    }

    public void setUser(User user) {
        this.user = user;
        user.setSessionId(sessionId);
    }

}
```

​	从功能上说，ServerSession类与ClientSession类似，也是一个很重要的胶水类。

（1）通过ServerSession实例可以导航到通道，以发送消息。

（2）在通道收到消息时，通过ServerSession实例，反向导航到用户，以处理业务逻辑。

​	需要注意的是sessionId的类型，它是String类型而不是数字类型，为什么呢？**主要是为了方便后期的分布式扩展**。在后期，通信服务器需要从单体服务器扩展到多节点服务集群。**为了在分布式集群的应用场景下，在后期sessionId可以确保全局唯一，使用String类型就比较方便**。

#### 9.5.3 SessionMap会话管理器

​	一台服务器需要接受几万/几十万的客户端连接。每一条连接都对应到一个ServerSession实例，服务器需要对这些大量的ServerSession实例进行管理。

​	**SessionMap负责管理服务器端所有的ServerSession。它有一个<u>线程安全的ConcurrentHashMap类型的成员map</u>，保持sessionId到服务器端ServerSession的映射。**

```java
@Slf4j
@Data
public final class SessionMap {
    private SessionMap() {
    }

    private static SessionMap singleInstance = new SessionMap();

    //会话集合
    private ConcurrentHashMap<String, ServerSession> map =
            new ConcurrentHashMap<String, ServerSession>();

    public static SessionMap inst() {
        return singleInstance;
    }

    /**
     * 增加session对象
     */
    public void addSession(
            String sessionId, ServerSession s) {
        map.put(sessionId, s);
        log.info("用户登录:id= " + s.getUser().getUid()
                + "   在线总数: " + map.size());

    }

    /**
     * 获取session对象
     */
    public ServerSession getSession(String sessionId) {
        if (map.containsKey(sessionId)) {
            return map.get(sessionId);
        } else {
            return null;
        }
    }

    /**
     * 根据用户id，获取session对象
     */
    public List<ServerSession> getSessionsBy(String userId) {

        List<ServerSession> list = map.values()
                .stream()
                .filter(s -> s.getUser().getUid().equals(userId))
                .collect(Collectors.toList());
        return list;
    }

    /**
     * 删除session
     */
    public void removeSession(String sessionId) {
        if (!map.containsKey(sessionId)) {
            return;
        }
        ServerSession s = map.get(sessionId);
        map.remove(sessionId);
        Print.tcfo("用户下线:id= " + s.getUser().getUid()
                + "   在线总数: " + map.size());
    }


    public boolean hasLogin(User user) {
        Iterator<Map.Entry<String, ServerSession>> it =
                map.entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<String, ServerSession> next = it.next();
            User u = next.getValue().getUser();
            if (u.getUid().equals(user.getUid())
                    && u.getPlatform().equals(user.getPlatform())) {
                return true;
            }
        }

        return false;
    }
}
```

​	在调用ServerSessioni.bind()绑定时，在SessionMap中增加一次映射：

```java
// public class ServerSession
// 和channel 通道实现双向绑定
public ServerSession bind() {
    log.info(" ServerSession 绑定会话 " + channel.remoteAddress());
    channel.attr(ServerSession.SESSION_KEY).set(this);
    SessionMap.inst().addSession(getSessionId(), this);
    isLogin = true;
    return this;
}
```

​	在调用ServerSession.unbind()解除绑定时，在SessionMap中减少一次映射：

```java
public ServerSession unbind() {
    isLogin = false;
    SessionMap.inst().removeSession(getSessionId());
    this.close();
    return this;
}
```

​	<u>通过SessionMap，可以实现在线用户的统计。除此之外，当用户与用户之间进行单聊时，消息需要通过服务器端，在不同的用户之间进行转发，这时也需要用到SessionMap。</u>

### 9.6 点对点单聊的实践案例

​	单聊的业务非常简单，就像微信的聊天功能。这里强调一下，对单聊的业务流程进行具体介绍：

​	（1）当用户A登录成功之后，按照单聊的消息格式，发送所要的消息。

​	这里的消息格式设定为——userId：content。其中的userId，就是消息接收方目标用户B的userId；其中的content，表示聊天的内容。

​	（2）服务器端收到消息后，根据目标userID进行消息的转发，发送到用户B所在的客户端。

​	（3）客户端用户B收到用户A发来的消息，在自己的控制台显示出来。

​	<u>理论上来说，单聊业务是十分的简单。不过，在这里有问题，为很么服务器端的路由转发不是根据sessionID，而是根据userID呢？</u>

​	<u>原因是：用户B可能登录了多个会话（桌面会话、移动端会话、网页端会话），这时发给用户B的聊天信息必须转发到多个会话，所以需要根据userID进行转发</u>。

#### 9.6.1 简述单聊的端到端流程

​	单聊的端到端流程，从大的角度来说，包括以下环节：

​	client --发数据--> childChannel -> Server路由转发 -> childChannel --收数据--> client

​	（1）用户A发送单聊Protobuf数据包。

​	（2）服务器端接收到用户A的单聊数据包。

​	（3）服务器端转发单聊数据包到用户B。

​	（4）最终用户B接收到来自用户A的单聊数据包。

#### 9.6.2 客户端的ChatConsoleCommand收集聊天内容

​	ChatConsoleCommand命令收集类负责从Scanner控制台实例获取聊天的消息（格式为 id:message）。代码如下：

```java
@Data
@Service("ChatConsoleCommand")
public class ChatConsoleCommand implements BaseCommand {

    private String toUserId;
    private String message;
    public static final String KEY = "2";

    @Override
    public void exec(Scanner scanner) {
        System.out.print("请输入聊天的消息(id:message)：");
        String[] info = null;
        while (true) {
            String input = scanner.next();
            info = input.split(":");
            if (info.length != 2) {
                System.out.println("请输入聊天的消息(id:message):");
            } else {
                break;
            }
        }
        toUserId = info[0];
        message = info[1];
    }


    @Override
    public String getKey() {
        return KEY;
    }

    @Override
    public String getTip() {
        return "聊天";
    }

}
```

#### 9.6.3 客户端的CommandController发送POJO

​	ChatConsoleCommand的调用者是CommandController命令控制类。CommandController在完成聊天内容和目标用户信息的收集后，在自己的startOneChat（...）方法中调用chatSender发送器实例，将聊天消息组装成Protobuf数据包，通过客户端的通道发往服务器端。

```java
@Slf4j
@Data
@Service("CommandController")
public class CommandController {
    //聊天命令收集类
    @Autowired
    ChatConsoleCommand chatConsoleCommand;
    //...省略其他成员
    
    public void startCommandThread()
            throws InterruptedException {
        Thread.currentThread().setName("命令线程");

        while (true) {
            //建立连接
            while (connectFlag == false) {
                //开始连接
                startConnectServer();
                waitCommandThread();

            }
            //处理命令
            while (null != session) {

                Scanner scanner = new Scanner(System.in);
                clientCommandMenu.exec(scanner);
                String key = clientCommandMenu.getCommandInput();
                BaseCommand command = commandMap.get(key);

                if (null == command) {
                    System.err.println("无法识别[" + command + "]指令，请重新输入!");
                    continue;
                }


                switch (key) {
                    case ChatConsoleCommand.KEY:
                        command.exec(scanner);
                        startOneChat((ChatConsoleCommand) command);
                        break;

                    case LoginConsoleCommand.KEY:
                        command.exec(scanner);
                        startLogin((LoginConsoleCommand) command);
                        break;

                    case LogoutConsoleCommand.KEY:
                        command.exec(scanner);
                        startLogout(command);
                        break;

                }
            }
        }
    }
    
    //发送单聊消息
    private void startOneChat(ChatConsoleCommand c) {
        //登录
        if (!isLogin()) {
            log.info("还没有登录，请先登录");
            return;
        }
        chatSender.setSession(session);
        chatSender.setUser(user);
        chatSender.sendChatMsg(c.getToUserId(), c.getMessage());

//        waitCommandThread();

    }
    
    //...省略其他的命令处理
}
```

#### 9.6.4 服务器端的ChatRedrectHandler

​	服务器端的消息转发处理器ChatRedirectHandler类，主要的工作如下：

​	（1）对消息类型进行判断：判断是否为聊天请求Protobuf数据包。如果不是，通过super.channelRead(ctx, msg)将消息交给流水线的下一站。

​	（2）对用户登录进行判断：如果没有登录，则不能发送消息。

​	（3）开启**异步**的消息转发，由负责转发的chatRedirectProcesser实例完成消息转发。

```java
@Slf4j
@Service("ChatRedirectHandler")
@ChannelHandler.Sharable
public class ChatRedirectHandler extends ChannelInboundHandlerAdapter {

    @Autowired
    ChatRedirectProcesser chatRedirectProcesser;

    /**
     * 收到消息
     */
    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        //判断消息实例
        if (null == msg || !(msg instanceof ProtoMsg.Message)) {
            super.channelRead(ctx, msg);
            return;
        }

        //判断消息类型
        ProtoMsg.Message pkg = (ProtoMsg.Message) msg;
        ProtoMsg.HeadType headType = ((ProtoMsg.Message) msg).getType();
        if (!headType.equals(chatRedirectProcesser.type())) {
            super.channelRead(ctx, msg);
            return;
        }

        //判断是否登录
        ServerSession session = ServerSession.getSession(ctx);
        if (null == session || !session.isLogin()) {
            log.error("用户尚未登录，不能发送消息");
            return;
        }

        //异步处理IM消息转发的逻辑
        FutureTaskScheduler.add(() ->
        {
            chatRedirectProcesser.action(session, pkg);
        });

    }

}
```

#### 9.6.5 服务器端的ChatRedirectProcesser异步处理

​	编写负责异步消息转发的ChatRedirectProcesser类，功能如下：

​	（1）根据目标用户ID，找出所有的服务器端会话列表。

​	（2）然后为每一个会话转发一份消息。

​	大致的代码如下：

```java
@Slf4j
@Service("ChatRedirectProcesser")
public class ChatRedirectProcesser extends AbstractServerProcesser {
    @Override
    public ProtoMsg.HeadType type() {
        return ProtoMsg.HeadType.MESSAGE_REQUEST;
    }

    @Override
    public boolean action(ServerSession fromSession, ProtoMsg.Message proto) {
        // 聊天处理
        ProtoMsg.MessageRequest msg = proto.getMessageRequest();
        Print.tcfo("chatMsg | from="
                + msg.getFrom()
                + " , to=" + msg.getTo()
                + " , content=" + msg.getContent());
        // 获取接收方的chatID
        String to = msg.getTo();
        // int platform = msg.getPlatform();
        List<ServerSession> toSessions = SessionMap.inst().getSessionsBy(to);
        if (toSessions == null) {
            //接收方离线
            Print.tcfo("[" + to + "] 不在线，发送失败!");
        } else {

            toSessions.forEach((session) -> {
                // 将IM消息发送到接收方
                session.writeAndFlush(proto);
            });
        }
        return true;
    }

}
```

​	由于一个用户可能有多个会话，因此需要通过调用SessionMap会话管理器的SessionMap.inst().getSessionBy(to)方法来取得这个用户的所有会话。

```java
@Slf4j
@Data
public final class SessionMap {
    //会话集合
    private ConcurrentHashMap<String, ServerSession> map =
        new ConcurrentHashMap<String, ServerSession>();

    /**
     * 根据用户id，获取session对象
     */
    public List<ServerSession> getSessionsBy(String userId) {

        List<ServerSession> list = map.values()
            .stream()
            .filter(s -> s.getUser().getUid().equals(userId))
            .collect(Collectors.toList());
        return list;
    }
    // ...
}
```

#### 9.6.6 客户端的ChatMsgHandler接收POJO

​	客户端的ChatMsgHandler聊天消息处理器很简单，主要的工作如下：

​	（1）对消息类型进行判断：判断是否为聊天请求Protobuf数据包。如果不是，则通过super.channelRead(ctx, msg)将消息交给流水线的下一站。

​	（2）如果是聊天信息，则将聊天消息显示在控制台。

```java
@ChannelHandler.Sharable
@Service("ChatMsgHandler")
public class ChatMsgHandler extends ChannelInboundHandlerAdapter {


    public ChatMsgHandler() {

    }


    /**
     * 业务逻辑处理
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //判断消息实例
        if (null == msg || !(msg instanceof ProtoMsg.Message)) {
            super.channelRead(ctx, msg);
            return;
        }

        //判断类型
        ProtoMsg.Message pkg = (ProtoMsg.Message) msg;
        ProtoMsg.HeadType headType = pkg.getType();
        if (!headType.equals(ProtoMsg.HeadType.MESSAGE_REQUEST)) {
            super.channelRead(ctx, msg);
            return;
        }

        ProtoMsg.MessageRequest req = pkg.getMessageRequest();
        String content = req.getContent();
        String uid = req.getFrom();

        System.out.println(" 收到消息 from uid:" + uid + " -> " + content);
    }
    
}
```

### 9.7 详解心跳检测

​	客户端的心跳检测对于任何长连接的应用来说，都是一个非常基础的功能。要理解心跳的重要性，首先需要从**网络连接假死**的现象说起。

#### 9.7.1 网络连接的假死现象

​	**什么是连接假死呢？如果底层的TCP连接已经断开，但是服务器端并没有正常地关闭套接字，服务器端认为这条TCP连接仍然是存在的**。

​	连接假死的具体表现如下：

​	（1）在服务器端，会有一些处于TCP_ESTABLISHED状态的”正常“连接。

​	（2）但在客户端，TCP客户端已经显示连接已经断开。

​	（3）**客户端此时虽然可以进行断线重连操作，但是上一次的连接状态依然被服务器端认为有效，并且服务器端的资源得不到正确释放，包括套接字上下文以及接口/发送缓冲区**。

​	<u>连接假死的情况虽然不常见，但是确实存在。服务器端长时间运行后，会面临大量假死连接得不到正常释放的情况。由于每个连接都会耗费CPU和内存资源，因此大量假死的连接会逐渐消耗光服务器的资源，使得服务器越来越慢，IO处理效率越来越低，最终导致服务器崩溃。</u>

​	连接假死通常是由以下多个原因造成的，例如：

​	（1）应用程序出现线程阻塞，无法进行数据的读写。

​	（2）网络相关的设备出现故障，例如网卡、机房故障。

​	（3）**网络丢包。公网环境非常容易出现丢包和网络抖动等现象**。

​	**解决假死的有效手段是：客户端定时进行心跳检测，服务器端定时进行空闲检测。**

> [ping、网络抖动与丢包](https://www.cnblogs.com/zhangnan35/p/11128388.html)
>
> [记一次对网络抖动经典案例的分析](https://yq.aliyun.com/articles/697773)

#### 9.7.2 服务器端的空闲检测

​	想解决假死问题，服务器端的有效手段就是——空闲检测。

​	**何为空闲检测？就是每隔一段时间，检测子通道是否有数据读写，如果有，则子通道是正常的；如果没有，则子通道被判定为假死，关掉子通道**。

​	服务器端如何实现空闲检测呢？**使用Netty自带的IdleStateHandler空闲状态处理器就可以实现这个功能**。下面的示例程序继承自IdleStateHandler，定义一个假死处理类：

```java
@Slf4j
public class HeartBeatServerHandler extends IdleStateHandler {

    private static final int READ_IDLE_GAP = 150;

    public HeartBeatServerHandler() {
        super(READ_IDLE_GAP, 0, 0, TimeUnit.SECONDS);

    }

    public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
        //判断消息实例
        if (null == msg || !(msg instanceof ProtoMsg.Message)) {
            super.channelRead(ctx, msg);
            return;
        }

        ProtoMsg.Message pkg = (ProtoMsg.Message) msg;
        //判断消息类型
        ProtoMsg.HeadType headType = pkg.getType();
        if (headType.equals(ProtoMsg.HeadType.HEART_BEAT)) {
            //异步处理,将心跳包，直接回复给客户端
            FutureTaskScheduler.add(() -> {
                if (ctx.channel().isActive()) {
                    ctx.writeAndFlush(msg);
                }
            });

        }
        super.channelRead(ctx, msg);

    }

    @Override
    protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
        System.out.println(READ_IDLE_GAP + "秒内未读到数据，关闭连接");
        ServerSession.closeSession(ctx);
    }
}
```

​	在HeartBeatServerHandler的构造函数中，调用了基类IdleStateHandler的构造函数，传递了四个参数：

```java
public HeartBeatServerHandler() {
    super(READ_IDLE_GAP, 0, 0, TimeUnit.SECONDS);
}
```

​	其中，第一个参数表示入站空闲检测时长，指的是一段时间内如果没有数据入站，就判定连接假死；第二个参数是出站空闲检测时长，指的是一段时间内如果没有数据出站，就判定连接假死；第三个参数是出/入站检测时长，表示在一段时间内如果没有出站或者入站，就判定连接假死。最后一个参数表示时间单位，TimeUnit.SECONDS表示秒。

​	<u>判定假死后，IdleStateHanler类会回调自己的channelIdle()方法。在这个子类的重写版本中，重写了这个空闲回调方法，手动关闭连接。</u>

```java
@Override
protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
    System.out.println(READ_IDLE_GAP + "秒内未读到数据，关闭连接");
    ServerSession.closeSession(ctx);
}
```

​	HeartBeatServerHandler实现的主要功能是空闲检测，需要客户端定时发送心跳数据包（或报文、消息）进行配合。而且**客户端发送心跳数据包的时间间隔需要远远小于服务器端的空闲检测时间间隔。**

​	HeartBeatServerHandler收到客户端的心跳数据包之后，可以直接回复到客户端，让客户端也能进行类似的空闲检测。由于IdleStateHandler本身是一个入站处理器，只需要重写这个子类HeartBeatServerHandler的channelRead方法，然后将心跳数据包直接回复客户端即可。

​	<u>如果HeartBeatServerHandler要重写channelRead方法，一定要记得调用基类的"super.channelRead(ctx,msg);"方法，不然IdleStateHandler的入站空闲检测会无效。</u>

ps:这里的意思应该就是不要截断流水线的handler调用吧。

#### 9.7.3 客户端的心跳报文

​	**与服务器端的空闲检测相配合，客户端需要定期发送数据包到服务器端，通常这个数据包称为心跳数据包**。接下来，定义一个Handler业务处理器定期发送心跳数据包给服务器端。

```java
@Slf4j
@ChannelHandler.Sharable
@Service("HeartBeatClientHandler")
public class HeartBeatClientHandler extends ChannelInboundHandlerAdapter {
    //心跳的时间间隔，单位为s
    private static final int HEARTBEAT_INTERVAL = 100;

    //在Handler被加入到Pipeline时，开始发送心跳
    @Override
    public void handlerAdded(ChannelHandlerContext ctx)
            throws Exception {
        ClientSession session = ClientSession.getSession(ctx);
        User user = session.getUser();
        HeartBeatMsgBuilder builder =
                new HeartBeatMsgBuilder(user, session);

        ProtoMsg.Message message = builder.buildMsg();
        //发送心跳
        heartBeat(ctx, message);
    }

    //使用定时器，发送心跳报文
    public void heartBeat(ChannelHandlerContext ctx,
                          ProtoMsg.Message heartbeatMsg) {
        ctx.executor().schedule(() -> {

            if (ctx.channel().isActive()) {
                log.info(" 发送 HEART_BEAT  消息 to server");
                ctx.writeAndFlush(heartbeatMsg);

                //递归调用，发送下一次的心跳
                heartBeat(ctx, heartbeatMsg);
            }

        }, HEARTBEAT_INTERVAL, TimeUnit.SECONDS);
    }

    /**
     * 接受到服务器的心跳回写
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        //判断消息实例
        if (null == msg || !(msg instanceof ProtoMsg.Message)) {
            super.channelRead(ctx, msg);
            return;
        }

        //判断类型
        ProtoMsg.Message pkg = (ProtoMsg.Message) msg;
        ProtoMsg.HeadType headType = pkg.getType();
        if (headType.equals(ProtoMsg.HeadType.HEART_BEAT)) {

            log.info(" 收到回写的 HEART_BEAT  消息 from server");

            return;
        } else {
            super.channelRead(ctx, msg);
        }
    }

}
```

​	<u>在HeartBeatClientHandler实例被加入到流水线时，它重写的handlerAdded方法被回调。在handlerAdded(...)方法中，开始调用heartBeat()方法，发送心跳数据包。heartBeat是一个不断递归调用的方法，它的递归调用的方式比较特别：使用了executor()获取当前通道绑定的Reactor反应器NIO线程，然后通过NIO线程的schedule()定时调度方法，每隔一段时间(50s)执行一次回调，向服务器端发送一个心跳数据包</u>。

​	**客户端的心跳间隔，要比服务器端的空闲检测时间间隔要短，一般来说，要比它的一半要短一些，可以直接定义为空闲时间间隔的1/3。这样做的目的就是防止公网偶发的秒级抖动。**

​	<u>HeartBeatClientHandler实例并不是一开始就装配到了流水线中的，它装配的时机是在登录成功之后。</u>

```java
@Slf4j
@ChannelHandler.Sharable
    @Service("LoginResponceHandler")
    public class LoginResponceHandler extends ChannelInboundHandlerAdapter {
        /**
     * 业务逻辑处理
     */
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg)
            throws Exception {
            // ...
            //判断返回是否成功
            ProtoMsg.LoginResponse info = pkg.getLoginResponse();

            ProtoInstant.ResultCodeEnum result =
                ProtoInstant.ResultCodeEnum.values()[info.getCode()];

            if (!result.equals(ProtoInstant.ResultCodeEnum.SUCCESS)) {
                //登录失败
                log.info(result.getDesc());
            } else {
                //登录成功
                ClientSession.loginSuccess(ctx, pkg);
                ChannelPipeline p = ctx.pipeline();
                //移除登录响应处理器
                p.remove(this);

                //在编码器后面，动态插入心跳处理器
                p.addAfter("encoder", "heartbeat", new HeartBeatClientHandler());
            }
        }
    }
```

​	在登录成功之后，在ChannelPipeline通道流水线上，HeartBeatClientHandler心跳客户端处理器实例被动态插入到了解码器之后。

​	服务器端的空闲检测处理器在收到客户端的心跳数据包之后，会进行回写。在HeartBeatClientHandler的channelRead方法中，对回写的数据包进行了简单的处理。这个地方还藏有另外的一个玄机，那就是HeartBeatClientHandler在完成心跳处理的同时，还能和服务器的空闲检测处理器一样，继承IdleStateClient类，在客户端进行空闲检测。这样，客户端也可以对服务器进行假死判定。在服务器端假死的情况下，客户端可以发起重连。客户端的空闲检测的实战留给大家自行实验。

​	运行以上代码，查看运行效果:

```java
//ServerApplication
2020-01-16 03:48:16 INFO (ChatServer.java:81) - 疯狂创客圈 CrazyIM 服务启动, 端口 /0:0:0:0:0:0:0:0:8081
150秒内未读到数据，关闭连接
150秒内未读到数据，关闭连接
2020-01-16 03:52:11 INFO (User.java:53) - 登录中: User{uid='laowang', devId='1111', token='123456', nickName='nickName', platform=WINDOWS}
2020-01-16 03:52:11 INFO (ServerSession.java:74) -  ServerSession 绑定会话 /127.0.0.1:56276
2020-01-16 03:52:11 INFO (SessionMap.java:36) - 用户登录:id= laowang   在线总数: 1
2020-01-16 03:52:11 INFO (LoginRequestHandler.java:60) - 登录成功:User{uid='laowang', devId='1111', token='123456', nickName='nickName', platform=WINDOWS}
2020-01-16 03:52:23 INFO (User.java:53) - 登录中: User{uid='panjinlian', devId='1111', token='123456', nickName='nickName', platform=WINDOWS}
2020-01-16 03:52:23 INFO (ServerSession.java:74) -  ServerSession 绑定会话 /127.0.0.1:56307
2020-01-16 03:52:23 INFO (SessionMap.java:36) - 用户登录:id= panjinlian   在线总数: 2
2020-01-16 03:52:23 INFO (LoginRequestHandler.java:60) - 登录成功:User{uid='panjinlian', devId='1111', token='123456', nickName='nickName', platform=WINDOWS}
[pool-2-thread-1|ChatRedirectProcesser:action]：chatMsg | from=laowang , to=panjinlian , content=你好骚啊
[pool-2-thread-2|ChatRedirectProcesser:action]：chatMsg | from=panjinlian , to=laowang , content=会不会说话啊？


//ClientApplication
2020-01-16 03:48:20 INFO (NettyClient.java:102) - 客户端开始连接 [疯狂创客圈IM]
2020-01-16 03:48:21 INFO (CommandController.java:96) - 疯狂创客圈 IM 服务器 连接成功!
请输入某个操作指令：
[menu] 0->show 所有命令 | 1->登录 | 2->聊天 | 10->退出 | 
0
请输入某个操作指令：
[menu] 0->show 所有命令 | 1->登录 | 2->聊天 | 10->退出 | 
1
请输入用户信息(id:password)  
laowang:123456
2020-01-16 03:52:10 INFO (LoginSender.java:18) - 构造登录消息
2020-01-16 03:52:10 INFO (LoginSender.java:21) - 发送登录消息
请输入某个操作指令：
[menu] 0->show 所有命令 | 1->登录 | 2->聊天 | 10->退出 | 
2020-01-16 03:52:11 INFO (BaseSender.java:81) - 发送成功
2020-01-16 03:52:11 INFO (ClientSession.java:61) - 登录成功
2
请输入聊天的消息(id:message)：panjinlian:你好骚啊
2020-01-16 03:52:36 INFO (ChatSender.java:14) - 发送消息 startConnectServer
请输入某个操作指令：
[menu] 0->show 所有命令 | 1->登录 | 2->聊天 | 10->退出 | 
2020-01-16 03:52:36 INFO (ChatSender.java:28) - 发送成功:你好骚啊
 收到消息 from uid:panjinlian -> 会不会说话啊？


//ClientApplication(1)
2020-01-16 03:48:25 INFO (NettyClient.java:102) - 客户端开始连接 [疯狂创客圈IM]
2020-01-16 03:48:25 INFO (CommandController.java:96) - 疯狂创客圈 IM 服务器 连接成功!
请输入某个操作指令：
[menu] 0->show 所有命令 | 1->登录 | 2->聊天 | 10->退出 | 
1
请输入用户信息(id:password)  
panjinlian:123456
2020-01-16 03:52:22 INFO (LoginSender.java:18) - 构造登录消息
2020-01-16 03:52:22 INFO (LoginSender.java:21) - 发送登录消息
请输入某个操作指令：
[menu] 0->show 所有命令 | 1->登录 | 2->聊天 | 10->退出 | 
2020-01-16 03:52:22 INFO (BaseSender.java:81) - 发送成功
2020-01-16 03:52:23 INFO (ClientSession.java:61) - 登录成功
 收到消息 from uid:laowang -> 你好骚啊
2
请输入聊天的消息(id:message)：laowang:会不会说话啊？
2020-01-16 03:52:55 INFO (ChatSender.java:14) - 发送消息 startConnectServer
请输入某个操作指令：
[menu] 0->show 所有命令 | 1->登录 | 2->聊天 | 10->退出 | 
2020-01-16 03:52:55 INFO (ChatSender.java:28) - 发送成功:会不会说话啊？
```



#### 9.8 本章小结

​	本章内容是Netty学习的一次综合性的检验，覆盖了非常全面的Netty知识：包括自定义编解码器的开发、半包的处理、流水线的装配、会话的使用等。