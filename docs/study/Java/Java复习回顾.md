> 当时最早是课外提前学的Java，看的《java疯狂讲义》。然后平时有事没事看到优质Java、SQL、Redis等优质技术文章，都会阅读+收藏。但是时间长了，阅读过的东西还是会忘记，俗话好记性不如烂笔头。所以这里再回顾下一些知识点，然后整理下笔记吧。
>
> 这里先更新 ==> 并发、虚拟机，这两个在实际开发解决问题时比较有帮助。（不只是代码，而是底层思想理解上的帮助）
>

# 1. Java并发

## 1.1 基础概念

### 1.1.1 线程

#### 1.1.1.1 线程-创建

常用的几种线程创建方式如下：

1. `MyThread`继承`Thread`，重写`run()`方法，新建对象`new MyThread()`
   + `public Thread() {init(null, null, "Thread-" + nextThreadNum(), 0);}`
2. `MyRunnable`实现`Runnable`接口，重写`run()`方法，新建对象`new Thread(new MyRunnable())`  <= <small>(当然也可以写成lambda表达式)</small>
   + `public Thread(Runnable target) {init(null, target, "Thread-" + nextThreadNum(), 0);}`
3. `MyCallable`实现`Callable`接口，重写`call()`方法，新建对象`new Thread(new FutureTask<Integer>(()->1));` <small>(这里就是返回值1的异步线程，采用lambda表达式)</small>
4. 使用`Executors.newCachedThreadPool()`静态方法创建线程池`ExecutorService`，由线程池管理线程。
   + `Future<?> submit(Runnable task);`
   + `<T> Future<T> submit(Runnable task, T result);`
   + `<T> Future<T> submit(Callable<T> task);`

*ps ：实际开发中，一般用不到单独的线程。多数情况提交异步任务到线程池，由线程池自动挑选闲置线程对象来执行异步任务。*

*如果实在需要使用独立的线程对象，建议使用`Runnable`方案or`Callable`方案，不建议直接使用继承Thread的方案，因为Java不像C++允许多继承。*

#### 1.1.1.2 线程-启动

1. Thread对象调用`start()`
2. 线程池`ExecutorService`提交线程任务`submit（...)`<small>(池内有空闲则直接复用，无线程可用且未达到线程池上限容量，则新建并启动)</small>

线程启动，即进入"就绪态"，是否"运行态"还得看CPU调度情况。

#### 1.1.1.3 线程状态转换

> [多线程系列（一）------ 线程的状态及转换](https://blog.csdn.net/qq_35206261/article/details/88873820) <= 文章不错，图很形象了。
>
> [LockSupport（park/unpark）源码分析](https://www.jianshu.com/p/e3afe8ab8364) <= 推荐阅读

按照Thread类定义，java线程共有6个状态：(以下内容在官方Thread类上的注解都能看到->除了我的额外备注外)

+ NEW

  ​	新建的对象，仍未启动时。<small>（即未触发`start()`方法）</small>

+ RUNNABLE

  ​	处于RUNNABLE的线程在JVM中运行，但是其在操作系统层面不一定被执行（比如可用的处理器都被其他线程/进程占用）。

  ​	RUNNABLE的线程，对应操作系统的**就绪态or运行态**。<small>（具体看线程是否仍占有时间片，以及操作系统的调度实现）</small>

  <small>（Linux系统线程即进程，底层 C语言用的相同数据结构`task_struct`，线程大部分内存空间相关的指针指向线程组组长/进程组组长所在的空间，即属于同一线程组/进程组的线程共享内存空间，当然程序计数器、函数堆栈是线程需要另外申请的空间，与其他线程不共享。）</small>

+ BLOCKED

  ​	预进入/执行`synchronized`代码块/方法的线程，若**抢占不到锁**(monitor lock)，则进入阻塞态。<small>（操作系统的monitor管程机制就是学习的java锁机制，操作系统管程的wait和signal类似java的wait和notify）</small>

  ​	<small>（实际非底层部件开发，那其实用得最频繁的就是`synchronized`关键字。JDK1.6以及往后对`synchronized`进行了优化,采取“锁升级”策略,而非粗暴的重量级锁。）</small>

+ WAITING

  ​	调用`wait()`、`join()`、`LockSupport.park()`三者之一，线程进入WAITING等待状态。

  + 调用`Object.wait()`的线程进入队列，需要其他线程执行`Object.notify()`或`Object.notifyAll()`，才能退出WAITING状态，重返RUNNABLE状态。

    <small>`wait()`和`notidy()`必须在`synchronized`修饰区域使用，否则抛出异常`Exception in thread "XXX" java.lang.IllegalMonitorStateException`。这个</small>

  + 调用`Thread对象.join()`的线程，需要等待指定的线程进入TERMINATED状态后，才能退出WAITING状态。

  + 调用`LockSupport.park()`的线程t1,需要等待其他线程执行`LockSupport.unlock(t1)`之后，才能退出WAITING状态。

    <small>**unpark(t1)不叠加，多次unpark(t1)和一次的效果相同，内部都是二进制信号量计数置为1**。unpark可以先于park执行，只要使得计数为1,park就无需进入WAITING直接往下执行。park在监听的二进制信号量/条件变量>0时置0并继续往下执行;而upark不管二进制信号量/条件变量值为多少，将值置1。这里park和unpark相当于操作系统的条件变量机制，只有条件变量满足>0，park才无需等待，否则等待直到满足条件变量（有人对该线程执行unpark）or参数设置的至多等待时间到达。</small>

  ​	<small>(`park()`方法和`unpark()`方法类似`wait()`和`notify()/notifyAll()`，前者能够指定要"许可"的线程，后者唤醒具有随机性。`park()`和`unpark()`底层用到Unsafe类，用mutex二进制信号量、条件变量来实现。且`park`和`unpark`不要求在同步代码块内使用。)</small>

  ​	<small>(`wait()`方法，不建议使用。其实导致WAITING的方法，除非没办法不然这几个都尽量别用。)</small>

+ TIMED_WAITING

  ​	和WAITING类似，但是多了时间限制，即线程至多维持time时间的WAITING状态。

  + 调用`Thread.sleep(time)`，线程睡眠time时间。（注意，sleep方法不会释放线程原本占有的锁，如果原本线程进入`synchronized`方法并`sleep(time)`，那么其他线程即使获取CPU执行时间片，仍然没法进入`synchronized`修饰的区域）
  + 在同步方法块内执行`Object.wait(time)`至多等待time时间，如果期间没有被notify移出等待队列，则由JVM将其移出等待队列，重新进入RUNNABLE状态。
  + `Thread对象.join()`至多等待目标线程time时间，若time时间内目标线程仍没有terminate，那当前线程从TIME_WAITING转为RUNNABLE。
  + `LockSupport.parkNanos()`和`LockSupport.parkUntil()`，第一个设置至多等待多少纳秒，第二个设置至多等到什么时间点。

+ TERMINATED

  线程执行结束则进入该状态。



![img](https://img-blog.csdnimg.cn/20190329113203194.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1MjA2MjYx,size_16,color_FFFFFF,t_70)

​	图中还有`yield()`方法，用于将RUNNABLE状态的县城由运行态转为就绪态，实际开发中可以说完全见不到。说那么多，其实除了`synchronized`，其他的基本用不上。

Thread类中还有一个可改变线程状态的方法`interrupt()`。其可以提醒线程中断：

+ 如果线程非阻塞/等待状态，那么仅设置interrupt中断标志位为true，不影响线程正常运行（线程内当然可以自己编写逻辑，在中断标志位为true时采取某些动作）
+ 如果线程由于wait、sleep、join等方法而阻塞，则清理中断标志位,并抛出`InterruptedException`异常.
+ 在`java.nio.channels.*`进行数据传输，建立连接的时候发生阻塞时，若`interrupt()`人为提醒中断，同样设置标记位true，并且抛出异常（有多种，这里不列举）。

`interrupt()`方法常见于网络编程框架中，日常开发一般也少用到。（网络未知数大，常有未知问题导致阻塞，需要人为杀死线程/进程）

<small>其他的线程方法，用得更少了，还有的已经弃用->建议不要去可以了解已被弃用的方法，省得混淆。</small>

> [java sleep 和 wait线程是处于 阻塞 还是 就绪](https://zhidao.baidu.com/question/1818162835626221268.html)
>
> yield是就绪。wait 放对象锁 sleep不放。sleep不出让系统资源；wait是进入线程等待池等待，出让系统资源，其他线程可以占用CPU
>
> [Java中interrupt的使用](https://www.cnblogs.com/jenkov/p/juc_interrupt.html)
>
> [Java里一个线程调用了Thread.interrupt()到底意味着什么？](https://www.zhihu.com/question/41048032)
>
> 首先，一个线程不应该由其他线程来强制中断或停止，而是应该由线程自己自行停止。
> 所以，Thread.stop, Thread.suspend, Thread.resume 都已经被废弃了。
> 而 Thread.interrupt 的作用其实也不是中断线程，而是「通知线程应该中断了」，
> 具体到底中断还是继续运行，应该由被通知的线程自己处理。
>
> 具体来说，当对一个线程，调用 interrupt() 时，
> ① 如果线程处于被阻塞状态（例如处于sleep, wait, join 等状态），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常。仅此而已。
> ② 如果线程处于正常活动状态，那么会将该线程的中断标志设置为 true，仅此而已。被设置中断标志的线程将继续正常运行，不受影响。
>
> interrupt() 并不能真正的中断线程，需要被调用的线程自己进行配合才行。
> 也就是说，一个线程如果有被中断的需求，那么就可以这样做。
> ① 在正常运行任务时，经常检查本线程的中断标志位，如果被设置了中断标志就自行停止线程。
> ② 在调用阻塞方法时正确处理InterruptedException异常。（例如，catch异常后就结束线程。）
>
> ```java
> Thread thread = new Thread(() -> {
>     while (!Thread.interrupted()) {
>         // do more work.
>     }
> });
> thread.start();
> 
> // 一段时间以后
> thread.interrupt();
> ```
>
> 具体到你的问题，Thread.interrupted()清除标志位是为了下次继续检测标志位。
> 如果一个线程被设置中断标志后，选择结束线程那么自然不存在下次的问题，
> 而如果一个线程被设置中断标识后，进行了一些处理后选择继续进行任务，
> 而且这个任务也是需要被中断的，那么当然需要清除标志位了。

### 1.1.2 Executor

#### 1.1.2.1 Executor概述

*（建议遇到问题先看jdk注解，实在不了解再搜索资料，很多情况只是不清楚XX方法大致用法，注解对用法和执行流程一般都有介绍。当然如果连YY方法是哪个类的，或者连是否存在ZZ类、ZZ方法可以用都不清楚的话，当然还是先网上搜索有没有对应功能的java类or类库）*

​	根据jdk注解，**Executor目标就是分离任务提交和任务执行的过程，让用户无需关心提交的任务何时被哪个线程执行。**

​	<u>Executor并没有严格要求被提交的任务需要被异步执行，有些情况下任务简单，可以直接在调用Excutor方法的线程以同步方式执行任务。当然，最常见的情况还是希望提交的任务在新线程/其他线程里异步执行</u>。

​	尽管Executor对线程执行没有明确要求，但是不少Executor的实现类，对任务的执行时机、执行顺序等都有明确规定。且有些Executor实现类内部还会嵌套Executor对象，用于执行任务。

+ ` ExecutorService`继承`Executor`，是扩展性更高的接口。（`Executor`只有一个`void execute(Runnable command);`方法）
+ `ThreadPoolExecutor`提供可扩展的线程池实现。
+ `Executors`为这些Executor实现提供工厂方法。（即统一管理不同实现方式的Executor实现类）

> [**Java并发——Executor框架详解（Executor框架结构与框架成员）**](https://blog.csdn.net/tongdanping/article/details/79604637) <== **很详细，建议阅读。**
>
> [为什么类不能多继承,而接口可以多继承](https://blog.csdn.net/caidongxuan/article/details/107324427) <= RunnableFuture\<V\>继承Runnable和Future\<V\>，惭愧，忘记接口可以多继承。 
>
> **类不能多继承的原因是**：防止两个相同的方法被子类继承,如果是两个相同的继承 既不会知道重写哪个被继承的父类,又不是重载.且会导致方法体合并。
> **接口可以多继承的原因是**：当有相同的方法时候 二合一，因为接口里面的方法没有方法体。
>
> [java中的接口为什么可以多继承，其他类不能呢？](https://zhidao.baidu.com/question/1964506145037321940.html)
>
> java 在编译的时候就会检查 类是不是多继承，如果出现多继承编译不通过。但是在java语法中接口是可以多继承的。
>
> + java 如果出现多继承、父类中都有相同的属性和name 值 子类如果使用父类的属性和name 值 无法确定是哪一个父类的是 属性和name值。
>
> + 父类中如果相同的方法，并且子类并没有覆盖该方法。子类调用父类的时候 无法判断是那个父类的方法。
>
> + 接口是可以多继承的。接口（jdk 1.7 以下版本）里面的方法并有实现,即使接口之间具有相同的方法仍然是可以的 几个接口可以有想通的实现类和实现方法。而且接口 接口里面的成员变量都是 static   final的  有自己静态域 只能自己使用。
>
> + 接口的实现类可以有多个 。（java bean 注解注入） 一个接口（用多个实现类）被注入进来。调用方法的时候。会先依据bean 查找那个 一样的bean 。调用该实现类的方法。其次如过 实现类上都没有注解的 bean 会按照加载的先后顺序去调用的。

## 1.2 锁

### 1.2.1 java对象锁本质

> **[Java对象结构与锁实现原理及MarkWord详解](https://blog.csdn.net/scdn_cp/article/details/86491792)** <== **推荐,图文并茂,很详细**

​	*学过操作系统，你会知道，操作系统中的进程、CPU等对锁的实现，本质上就是对同一块区域进行数值判断（比如判断同一内存地址的当前值是0还是1，只有读取到值为1的CPU核、操作系统进程等实体才能继续工作。当然读取到值为1的实体会把值替换成0，使得其他实体无法继续工作。）*

​	<u>java的对象锁也不例外。`synchronized`指定用于"充当锁"的对象，其对象头信息中的"Mark Word"信息记录了"锁类型"、"锁拥有者"等信息。这样子，多线程下哪个线程拥有锁，拥有的是什么锁，JVM根据"充当锁"的对象的头信息即可知晓。</u>

​	通过阅读`OpenJDK/hotspot-37240c1019fd/src/share/vm/oops/oop.hpp`,我们可以看到主要的对象头信息如下:<small>（OpenJDK代码网上可以下载，是开源的）</small>

```hpp
// oopDesc is the top baseclass for objects classes.  The {name}Desc classes describe
// the format of Java objects so the fields can be accessed from C++.
// oopDesc is abstract.
// (see oopHierarchy for complete oop class hierarchy)

class oopDesc { 
    friend class VMStructs;
    private:
    volatile markOop  _mark;
    union _metadata {
        Klass*      _klass;
        narrowKlass _compressed_klass;
    } _metadata;

    // Fast access to barrier set.  Must be initialized.
    static BarrierSet* _bs;
    public:
    	//.... 各种函数
}
```

​	需要注意的是前三个private的成员变量:`markOop  _mark`,	` Klass*      _klass`,	`narrowKlass _compressed_klass`，这三个共同构成对象头信息，后两个都是还是类的元信息，存放在方法区（Method Area）中。这里元数据信息并非我们的关注点,我们主要关注`markOop _mark`,也被称为"Mark Word"，其主要记录锁信息和GC标记。

![img](https://img-blog.csdnimg.cn/20190115141050902.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDRE5fQ1A=,size_16,color_FFFFFF,t_70)

Java对象由对象头（锁信息、GC信息、类元信息）、对象体（存放成员变量）和对齐字节（使得组成Java对象的字节数能够被8整除）。

<small>*(方法，存放在方法区，而非在对象体中存储)*</small>

64bit的"Mark Word"信息，可以分成如下几种情况：<small>（信息涵盖：锁信息、HashCode、GC信息）</small>

+ 无锁状态（new）:最后3bit为001
+ 偏向锁：最后3bit为101
+ 轻量级锁（自旋锁）：最后2bit为00
+ 重量级锁：最后2ibt为10
+ GC标记信息：最后2bit为11

这里先不对各种锁的区别进行介绍，后面的"锁升级"再具体叙述。

![img](https://img-blog.csdnimg.cn/20190111092408622.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpdWR1bl9jb29s,size_16,color_FFFFFF,t_70)

通过java代码，同样可以查看锁信息（使用`org.openjdk.jol.info.ClassLayout`类的`ClassLayout.parseInstance(object).toPrintable()`来获取对象头信息。

```java
public class ThreadTest002 {
    public static void main(String[] args) {

        Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
        synchronized (o){
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
    }
}
```

输出信息如下：前两行为"Mark Word",可以看到对象o加锁前,后3bit为"001"（无锁状态），使用`synchronized`加锁后，后3bit变为"000"，根据前面各种锁的后几位判别，可以得知对象o上了轻量级锁（自旋锁）。

*（前8字节 为Mark Word，中间4字节为类指针->一般就4字节，指向该对象对应的方法区的类信息，最后4字节是对齐字节，补足保证java对象大小能被8字节整除）*

```none
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           c0 d8 9d 2c (11000000 11011000 10011101 00101100) (748542144)
      4     4        (object header)                           b3 7f 00 00 (10110011 01111111 00000000 00000000) (32691)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

下面再来看一个例子

```java
public class ThreadTest003 {

    public static void main(String[] args) {

        try {
            Thread.sleep(4500);
        } catch (InterruptedException e) {
            System.out.println("after sleep 4.5 s ...");
            e.printStackTrace();
        }
       // 这下面的代码和上面一样。
        Object o = new Object();
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
        synchronized (o){
            System.out.println(ClassLayout.parseInstance(o).toPrintable());
        }
    }
}
```

输出如下：可以看到初始后3bit为"101"（偏向锁），执行`synchronized`同步代码块后，o的Mark Word后3bit还是"101"，仍然是偏向锁。

```none
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 e9 00 a4 (00000101 11101001 00000000 10100100) (-1543444219)
      4     4        (object header)                           81 7f 00 00 (10000001 01111111 00000000 00000000) (32641)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

下面回顾"锁升级"流程，了解无锁、偏向锁、轻量级锁、重量级锁的区别。

> [Hotspot 垃圾回收之BarrierSet（三） 源码解析](https://blog.csdn.net/qq_31865983/article/details/103746959)
>
> [JAVA中类中的方法存储在什么地方？](https://zhidao.baidu.com/question/420152838.html)
>
> 类的实例方法在内存中是只有一份,不过肯定不会是第一个对象中,如果是第一个对象的话,那么当第一个对象被销毁的时候,那么后面的对象就永远无法调用了...
> 类的实例方法存在一个专门的区叫方法区,事实上类刚装载的时候就被装载好了,不过它们在"睡眠",只是这些方法必须当有对象产生的时候才会"苏醒".(比如,一个输出类的成员变量的方法,如果连对象都没有,何来的输出成员变量).所以,方法在装载的时候就有了,但是不可用,因为它没有指象任何一个对象。
>
> 类加载时 方法信息保存在一块称为方法区的内存中， 并不随你创建对象而随对象保存于堆中。可参考《深入java虚拟机》前几章。
> 另参考（他人文章）：
> 如果instance method也随着instance增加而增加的话，那内存消耗也太大了，为了做到共用一小段内存，Java 是根据this关键字做到的，比如：instance1.instanceMethod(); instance2.instanceMethod(); 在传递给对象参数的时候，Java 编译器自动先加上了一个this参数，它表示传递的是这个对象引用，虽然他们两个对象共用一个方法，但是他们的方法中所产生的数据是私有的，这是因为参数被传进来变成call stack内的entry，而各个对象都有不同call stack，所以不会混淆。其实调用每个非static方法时，Java 编译器都会自动的先加上当前调用此方法对象的参数，有时候在一个方法调用另一个方法，这时可以不用在前面加上this的，因为要传递的对象参数就是当前执行这个方法的对象。

### 1.2.2 锁升级

#### 1.2.2.1 无锁、偏向锁、轻两级锁（自旋锁）、重量级锁

> [**Java锁升级**](https://blog.csdn.net/pange1991/article/details/84877487) <= 强力推荐,图文并茂。1.2.2锁升级章节内，部分内容摘自该文章。

​	*前面"1.2.1 java对象锁本质"，我们得知java对象锁，本质上就是在对象头的"Mark Word"记录锁信息。（这和操作系统、CPU实现的锁策略类似，都是通过访问共享资源，根据值判断是否加锁、是否自己占有锁等。）*

​	在JDK1.6之前，`synchronized`直接申请重量级锁，而JDK1.6之后，添加了"锁升级"过程，提高了`synchronized`的综合效益。(注意：锁只能升级,不能降级,如果最后升级成了重量级锁,没法降级回之前的状态。当然你如果说重新新建了一个对象当作锁，那当然是从头升级了，不过这和锁升级就没关联了。)

​	介绍锁升级之前，我们需要了解每个锁的大致作用/区别。

![img](https://img-blog.csdnimg.cn/20181207170638115.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhbmdlMTk5MQ==,size_16,color_FFFFFF,t_70)

1. 无锁：字面意，不对对象加锁。(有两种无锁情况,一种是偏向锁标志位1可用,一种是偏向锁标志位0不可用。只有偏向锁标志位1的无锁对象才可能添加偏向锁)
2. 偏向锁：在Mark Word记录拥有该偏向锁的线程，不实际加"锁"。（因为即使是线上项目，也存在某些同步方法/代码块在一段时间内仅只有一个线程访问，此时如果直接加重量级锁，显然不合适。）
3. 轻量级锁：当同一锁被两个或两个以上的线程抢占时，偏向锁自动升级为轻量级锁。（也有可能一开始无锁直接升级成轻量级锁。上一小节的示例代码即存在该现象。）轻量级锁，又称自旋锁，因为当多个线程抢占轻两级锁时，内部使用while循环实现，循环判断是否能够抢占到锁。这里的锁，指JVM层面的对象锁，并非向操作系统申请的锁。
4. 重量级锁：需要向操作系统申请的锁，该锁是操作系统级别的，不再是JVM用户空间级别的（需要经过上下文切换，效率较低，在线程任务量小的时候，应该尽量避免使用重量级锁）。当抢占自旋锁的线程数量超过CPU核数的1/2，或者某线程自旋超过10次，自旋锁将自动升级为重量级锁。另外，当线程执行了wait方法，将直接升级为重量级锁。（所以不建议使用wait方法，其他显式阻塞线程的方法也不建议使用，除非代码逻辑不得不这么做，且对死锁等特殊情况必须有明确把握能处理。）

#### 1.2.2.2 重量级锁

> [通过openjdk源码ObjectMonitor底层实现分析wait/notify](https://blog.csdn.net/qq_33249725/article/details/104212364) <= 拿图当大纲看即可
>
> [深入分析wait/notify为什么要在同步块内](https://blog.csdn.net/lsgqjh/article/details/61915074) <== 推荐这篇
>
> [**调用了wait()的线程进入等待池，只有被notify唤醒之后才进入锁池，这两个池的内涵是什么？**](https://www.zhihu.com/question/64725629) <== 强烈推荐
>
> 下面用到的图片来自上述几篇文章。

​	JDK1.6之前`synchronized`修饰区域，需要向操作系统申请重量级锁之后才能访问。重量级锁不归JVM管控，由操作系统管理，即操作系统提供接口，java通过native方法实现（C/C++代码）。**重量级锁对应"管程"机制（Monitor），管程要求其管理的函数被访问前必须加锁，而函数执行完毕退出前必须释放锁，且同一时刻管程所管理的某函数只能被其中一个线程占有**。

​	管程的加锁、解锁由操作系统内核态完成，所以JVM中的java程序需要经历上下文切换（用户态与内核态之间切换，内核态的`task_struct`需要保存当前进程执行状态，然后完成锁操作，再将表示锁的信息传给用户态JVM进程）。*可想而知，本来用户态能完成的事情，现在需要经过操作系统中转，换来系统安全和进程间可靠执行的代价就是执行效率降低。*

​	关于Java管程的native实现，同样可以通过hotspot源码中查看。（`OpenJDK/hotspot-37240c1019fd/src/share/vm/runtime/objectMonitor.hpp`)

![[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-k3WKuRUO-1581065675036)(https://i.loli.net/2020/02/07/GQDUqBdIZnJoehY.jpg)]](https://img-blog.csdnimg.cn/20200207165547327.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMjQ5NzI1,size_16,color_FFFFFF,t_70)

​	![这里写图片描述](https://img-blog.csdn.net/20170313112310275?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHNncWpo/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```c++
// objectMonitor.hpp文件
ObjectWaiter * volatile _EntryList ;     // Threads blocked on entry or reentry.
```

​	假设`synchronized`的对象锁已经升级到重量级锁，此时多个线程抢占该锁，根据管程定义和底层代码实现，保证了只能有一个线程成为`Owner`，其余访问同一个管程所管理的函数的线程，进入`Entry Set`（处于阻塞状态BLOCKED，不会抢占CPU时间片）。当`Owner`执行了`wait`操作时，将释放锁，自身进入`Wait Set`（处于WAITING状态），此时`Entry Set`中随机一个BLOCKED的线程被唤醒(进入RUNNABLE状态)，和其他此刻本来就是RUNNABLE状态的线程抢占锁，流程和前面相同。如果`Owner`正常结束同步代码块并释放锁，同样会随机唤醒一个`Entry Set`的线程（BLOCKED状态转为RUNNABLE）。

​	当某一个`Owner`在`synchronized`代码块内执行了`notify()`或`notifyAll()`之后，如果`Wait Set`不为空，则唤醒第一个（Wait Set是双向链表），由于java线程被加入`Wait Set`的顺序不确定，所以对JVM来说就是随机一个线程被唤醒了。被唤醒的线程从WAITING进入到RUNNABLE状态，等待下次操作系统调度和其他RUNNABLE的线程抢占锁（`notify/notifyAll`唤醒，实际有两种处理策略，一种是直接加入`Entry Set`;另一种是先自旋尝试占锁，占锁未果则进入`Entry Set`）。

回顾操作系统中，管程的wait方法主要包含几个步骤：

+ 释放锁
+ 加入等待队列
+ 请求操作系统重新调度
+ 请求锁（这一步需要下次该线程被notify/notifyAll唤醒并成功占用CPU时间片）

----

小结：

+ 重量级锁，需要操作系统参与上下文切换。牺牲效率保证系统安全和线程同步互斥。
+ `wait`需要Monitor，所以需要重量级锁，即在`synchronized`代码块中使用。而`notify/notifyAll`只有和`wait`一起使用才有意义，同样需要在同步代码块中使用。
+ 管程Monitor主要组件包括3个：
  + WaitSet等待队列（执行过wait操作的旧`Owner`）
  + EntryList（可以理解为阻塞队列，阻塞条件即是否为锁的`Owner`）
  + `Owner`（锁的占有者-线程）。
+ 适用于线程任务耗时长or线程数量极多的情况<small>（此时重量级锁上下文切换的开销 < 占用CPU时间片的轻量级锁（反复自旋尝试占锁））</small>

#### 1.2.2.3 轻量级锁

> [深入理解CAS算法原理](https://www.jianshu.com/p/21be831e851e) <= 操作系统笔记里已经介绍够多了，这里java推荐看这篇就够了。
>
> [CMPXCHG - 比较并交换](https://www.hgy413.com/hgydocs/IA32/instruct32_hh/vc42.htm)
>
> | 操作码      | 指令                  | 说明                                                         |
> | ----------- | --------------------- | ------------------------------------------------------------ |
> | 0F B0/**r** | CMPXCHG **r/m8,r8**   | 比较 AL 与 **r/m8**。如果相等，则设置 ZF，并将 **r8** 加载到 **r/m8**。否则清除 ZF，并将 **r/m8** 加载到 AL。 |
> | 0F B1/**r** | CMPXCHG **r/m16,r16** | 比较 AX 与 **r/m16**。如果相等，则设置 ZF，并将 **r16** 加载到 **r/m16**。否则清除 ZF，并将 **r/m16** 加载到 AL。 |
> | 0F B1/**r** | CMPXCHG **r/m32,r32** | 比较 EAX 与 **r/m32**。如果相等，则设置 ZF，并将 **r32** 加载到 **r/m32**。否则清除 ZF，并将 **r/m32** 加载到 AL。 |
>
> [cpu cmpxchg 指令理解 (CAS)](https://blog.csdn.net/xiuye2015/article/details/53406432) <== 内含测试的汇编代码
>
> cmpxchg是汇编指令
> 作用：比较并交换操作数.
> 如：CMPXCHG r/m,r 将累加器AL/AX/EAX/RAX中的值与首操作数（目的操作数）比较，如果相等，第2操作数（源操作数）的值装载到首操作数，zf置1。如果不等， 首操作数的值装载到AL/AX/EAX/RAX并将zf清0
> 该指令只能用于486及其后继机型。第2操作数（源操作数）只能用8位、16位或32位寄存器。第1操作数（目地操作数）则可用寄存器或任一种存储器寻址方式。

​	显而易见，需要上下文切换的重量级锁效率较低。轻量级锁使用JVM层面的CAS操作（Compare And Swap/Set），无需和操作系统交互，效率更高。CAS的底层实现由CPU提供原子性机械原语`cmpxchg`，就像其他机械码一样被执行。

​	![img](https://img-blog.csdnimg.cn/20181207170638115.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhbmdlMTk5MQ==,size_16,color_FFFFFF,t_70)

​	轻两级

#### 1.2.2.4 偏向锁



### 1.2.3 synchronized

> [synchronized 是可重入锁吗？为什么？](https://www.cnblogs.com/incognitor/p/9894604.html)

synchronized是可重入锁，非公平锁。

### 1.2.4 volatile

​		



无锁-》偏向锁-》轻量级锁-》重量级锁

`synchronized`一开始偏向锁，就是没有锁，只是一个指针标识，因为往往加锁的方法其实只有一个线程在执行。然后遇到其他线程时，就升级成轻量级锁。轻量级锁，自旋锁，还是用户运行态，占用CPU资源。自旋超过10次or线程数超过CPU内核1/2数量，升级重量级锁，也就是 需要内核态提供的底层Lock。（底层Lock有自旋写法，也有非自旋使用等待队列->阻塞的写法，这里重量级指的是阻塞的写法。而轻量级锁是自旋的，还在运行态or就绪态。）

如果直接执行`wait`，那偏向锁，执行升级重量级锁。

对应[cpp代码]([https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/interpreterRuntime.cpp])`InterpreterRuntime::monitorenter`

> [openjdk中的同步代码](https://blog.csdn.net/iteye_16780/article/details/81620174)
>
> [volatile底层实现原理](https://www.cnblogs.com/wildwolf0/p/11449506.html)



# 2. Java虚拟机(JVM)





