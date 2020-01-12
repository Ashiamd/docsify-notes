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

