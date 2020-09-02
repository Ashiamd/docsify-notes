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
>
> [jvm学习：轻量级锁，偏向锁，重量级锁，重偏量锁流程及测试](https://juejin.im/post/6844904071850098701) <==  还不错，推荐阅读。下面部分图片，文字也摘自于该文章。

​	显而易见，需要上下文切换的重量级锁效率较低。轻量级锁使用JVM层面的CAS操作（Compare And Swap/Set），无需和操作系统交互，效率更高。CAS的底层实现由CPU提供原子性机械原语`cmpxchg`，就像其他机械码一样被执行。

​	轻量级锁，适用于线程并发量小且线程任务量小的场景。在并发量不高且线程任务量小的情况下，线程A占有锁的时间不长，线程B、C、D等完全可以自旋等待一小会，这样短时间内线程A、B、C、D都能执行完同步代码块的代码，中途无需退出RUNNING状态（就绪态和运行态）。

![img](https://img-blog.csdnimg.cn/20181207170638115.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhbmdlMTk5MQ==,size_16,color_FFFFFF,t_70)

​	轻两级锁上锁过程（CAS）如下：

1. 预进入`synchronized`的线程，在线程独享的栈空间**新建一条锁记录（Lock Record）对象**。

   ![img](https://user-gold-cdn.xitu.io/2020/2/25/1707a4d82d286632?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

   <small>(上图右侧，就是充当锁的java对象，最上面是Mark Word，往下是类元信息`union _metadata `中的指针,指向JVM方法区Method Area的类信息对象上；最下面Object body就是普通的需对象成员变量的。类内定义的方法当然也是存放在方法区的类信息中。)</small>


2. 让锁记录中的Object reference指向锁对象，并尝试用CAS**替换**Object的Mark Word，将Mark Word的值存入锁记录

   ![img](https://user-gold-cdn.xitu.io/2020/2/25/1707a4e43929ee69?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
   
   + 如果 **CAS替换成功**，对象头中存储了**锁记录地址和状态00**<small>（前面Mark Word表格可知，最后2bit为00表示轻量级锁）</small>，表示由该线程给对象加锁。
   
     ![img](https://user-gold-cdn.xitu.io/2020/2/25/1707a4f764023858?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
   
     （这里CAS替换了对象的Mark Word信息，所以前面查表时可看到无锁or偏向锁转换成自旋锁时明显Mark Word存储的信息不一样）
   
   + **CAS替换失败**，需要分两种情况考虑
   
     + 如果是其它线程已经持有了该 Object 的轻量级锁，这时表明有竞争，进入**锁膨胀过程(流程转重量级锁)**
   
     + **如果是自己执行了`synchronized`锁重入，那么再添加一条Lock Record作为重入的计数**
   
       （这里因为已经加过锁了，所以CAS打算加第二次锁时，会发现锁对象Object的Mark Word后两bit已是00即已被加锁，接着对比前面的信息发现lock record地址和自身线程拥有的lock record地址相同，所以不再替换Mark Word信息=>CAS失败，但是新增一条仅用来计数的Lock Record，其Object reference同样指向锁对象Object）
   
     ![img](https://user-gold-cdn.xitu.io/2020/2/25/1707a5066cdd602f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
   
3. 当拥有轻量级锁（自旋锁）的线程退出`synchronized`代码块（解锁时）如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重入计数减一。

   如果退出`synchronized`代码块时，栈中的锁记录Lock Record的Mark Word存储值不为null，则通过CAS将Mark Word恢复给Object锁对象。

   + 成功，则解锁成功

   + **失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程**

     (前面查表可知，64bit版本的Mark Word中，重量级锁前62bit用来标识锁对象Object向操作系统申请到的管程Monitor对象指针。)

     (轻量级锁前62bit标识锁对象的占有者线程的Lock Record地址指针)

   ![img](https://user-gold-cdn.xitu.io/2020/2/25/1707a51519432368?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

> [Java synchronized中轻量级锁LockRecord存在栈帧中的哪里？](https://www.zhihu.com/question/409046921/answer/1359657467)

#### 1.2.2.4 锁膨胀（轻量级锁升级为重量级锁）

> [并发编程（JAVA版）-------------（四)](https://blog.csdn.net/weixin_44350891/article/details/104733661) 
>
> [并发编程之线程第二篇](https://blog.csdn.net/zhao1299002788/article/details/104215554#t8) <= 图片出处

​	如果在尝试加轻量级锁的过程中，CAS操作无法成功，这时一种情况就是有其他线程为此对象加上了轻量级锁（有竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。

​	轻量级锁升级为重量级锁的2种默认界限（满足其一就升级为重量级锁）：

+ 存在线程为争夺轻量级锁已自旋10次
+ 争夺轻量级锁的线程数 > CPU总核数的 1/2

当Thread-1准备进行轻量级加锁时，Thread-0已经对该对象加了轻量级锁，Thread-1的CAS操作失败，继续自旋反复进行CAS操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208215534845.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW8xMjk5MDAyNzg4,size_16,color_FFFFFF,t_70)

当Thread-1的CAS自旋超过10次时（或当前争夺轻量级锁的线程总数 > CPU核数的1/2）时，Thread-1明确轻量级加锁失败，进入锁膨胀流程。

+ 即**为Object锁对象申请Monitor锁，让Object指向重量级锁地址**

+ 然后自己（Thread-1）进入Monitor的Entry List （BLOCKED状态，不再占用CPU时间片）

  （注意，这里原本占用轻量级锁的Thread-0的Lock Record中的Mark Word仍然存储的是锁对象Object最初的Mark Word）

  （由于升级为重量级锁，锁对象Object的Mark Word变为指向管程对象Monitor的指针，最后2bit变为10表示重量级锁）

+ **之后当Thread-0退出同步块解锁时，使用CAS将Mark Word的值恢复给对象头，失败。这时会进入重量级解锁流程，即按照Monitor地址找到Monitor对象，设置Owner为null，唤醒EntryList中BLOCKED线程。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208220009273.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW8xMjk5MDAyNzg4,size_16,color_FFFFFF,t_70)

#### 1.2.2.5 偏向锁（Biased Locking）

> [并发编程（JAVA版）-------------（四)](https://blog.csdn.net/weixin_44350891/article/details/104733661) 
>
> [jvm学习：轻量级锁，偏向锁，重量级锁，重偏量锁流程及测试](https://juejin.im/post/6844904071850098701) 

​	轻量级锁在没有竞争时（就自己这个线程），每次重入仍然需要执行CAS操作。Jdk6中引入了偏向锁来进一步优化：只有第一次使用CAS将线程ID设置到对象的Mark Word头，之后发现这个线程ID是自己的就表示没有竞争，不用重新CAS，以后只要不发生竞争，这个对象就归该线程所有。

​	在实际开发场景中，只有某几个时间段真正存在高并发，其他时间段比如凌晨3-4点等时间段，JVM中的方法其实很少被用户访问，很有可能一个锁对象被GC回收且重建创建后，在某1-3秒内只有一个在线用户访问了`synchronized`修饰的该方法，这时候完全可以只用偏向锁，因为不存在多线程竞争。

一个对象创建时：

- **如果开启了偏向锁（默认开启，且默认JVM初始化的4s后才启动->延迟启动）**，那么对象创建后，Mark Word值为0x05即最后3位为101，这时它的thread、epoch、age都为0（匿名偏向锁，因为对象新建时，并非一定拿来当作偏向锁使用）

  （通过命令行`java -XX:+PrintFlagsFinal -version | grep BiasedLocking`查看偏向锁相关的JVM非标准参数)

- **偏向锁是默认是延迟的**(j**dk1.8是JVM初始化后延迟4s**，因为JVM初始化时新建的GC等线程本身需要加锁同步互斥，已知多个线程抢占锁，则没必要用偏向锁。偏向锁的锁撤销本身也需要消耗资源。)，不会在程序启动时立即生效如果想避免延迟，可以加VM参数 `- xx:BiasedLockingStartupDelay=0`来禁用延迟。

- 如果没有开启偏向锁，那么对象创建后，Mark Word值为0x01即最后三位为001，这时它的Hash Code、age都为0，**第一次用到Hash Code时才会赋值**。

```shell
## Linux查看JDK关于 偏向锁Biased Locking的 非标准参数（凡是带X的都是非标准参数），正好主力机是Arch Linux。
## JDK1.8
./java -XX:+PrintFlagsFinal -version | grep BiasedLocking
     intx BiasedLockingBulkRebiasThreshold          = 20                                  {product} ## 偏向锁批量重偏向的阈值20，以class为单位，该类的偏向锁对象被执行第20次偏向撤销操作时，JVM假设当前偏向的线程不合适，重新把该class下所有对象锁对象实例偏向于新的线程。
     intx BiasedLockingBulkRevokeThreshold          = 40                                  {product} ## 偏向锁批量撤销的阈值40,当执行过偏向锁批量重偏向后，锁对象对应的class类上的锁撤销次数继续累加达到第40次时，JVM认为当前锁存在多线程争夺，将所有该class下的锁对象标记为不可偏向，即Mark Word后3bit从101变成000，直接走轻量级锁的逻辑。
     intx BiasedLockingDecayTime                    = 25000                               {product}
     intx BiasedLockingStartupDelay                 = 4000                                {product}   ## 这个4000 表示4000毫秒后延迟启动 偏向锁，
     bool TraceBiasedLocking                        = false                               {product}
     bool UseBiasedLocking                          = true                                {product}
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)

## OpenJDK 14
java -XX:+PrintFlagsFinal -version | grep BiasedLocking 
     intx BiasedLockingBulkRebiasThreshold         = 20                                        {product} {default}
     intx BiasedLockingBulkRevokeThreshold         = 40                                        {product} {default}
     intx BiasedLockingDecayTime                   = 25000                                     {product} {default}
     intx BiasedLockingStartupDelay                = 0                                         {product} {default}
     bool UseBiasedLocking                         = true                                      {product} {default}
openjdk version "14.0.2" 2020-07-14
OpenJDK Runtime Environment (build 14.0.2+12)
OpenJDK 64-Bit Server VM (build 14.0.2+12, mixed mode)
```

轻量级锁和偏向锁对比：

```java
static final Object obj = new Object();
public static void m1() {
    synchronized( obj ) {
        // 同步块 A
        m2();
    }
}
public static void m2() {
    synchronized( obj ) {
        // 同步块 B
        m3();
    }
}
public static void m3() {
    synchronized( obj ) {
        // 同步块 C
    }
}
```

![img](https://user-gold-cdn.xitu.io/2020/2/25/1707a5b930bf65b0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

![img](https://user-gold-cdn.xitu.io/2020/2/25/1707a5c5810fe154?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

​	使用轻量级锁，每次CAS操作前都需要暂时生成一个Lock Reord锁记录，用于和轻量级锁对象的Mark Word信息比对，只有成功替换时，线程才能占有锁。

​	使用偏向锁时，线程同样CAS操作替换对象的Mark Word信息。当已有线程占有锁时，其他占用偏向锁失败的线程只需在之后的自旋CAS中比对锁对象MarkWord中的Thread ID。

​	只要存在2个or2个以上的线程抢占偏向锁，就会触发偏向锁撤销的操作。例如Thread-0占有偏向锁，Thread-1抢占偏向锁时，Thread-0在安全点（无执行指令时）暂停，锁对象的Mark Word中的Thread ID被清零（即偏向锁撤销），Thread-0和Thread-1重新抢占锁（抢夺偏向锁的过程中，锁对象已经升级为轻量级锁，但是可偏向标志位还是1,表示之后抢占到锁的线程仍然得到偏向锁。一般原本占有偏向锁的线程有更大概率再次占有锁。）

```java
public class ThreadTest005 {

    public static void main(String[] args) throws InterruptedException {

        // 偏向锁延迟4s的干扰（jdk8延迟4s后启动偏向锁，这之后创建的对象可偏向标志位为1）
        Thread.sleep(4100);

        Object lock = new Object();
        ClassLayout classLayout = ClassLayout.parseInstance(lock);
        new Thread(() -> {
            System.out.println(classLayout.toPrintable());
            synchronized (lock) {
                System.out.println(classLayout.toPrintable());
            }
            for(int i = 0;i<10000_00000;++i); // 确保 同步代码块执行结束后，线程不再占用锁
            System.out.println(classLayout.toPrintable());
        }, "t1").start();
    }
}
```

输出如下：（只截取 3 次 输出的Mark Word）

可以看出来，即使**占有偏向锁的线程已经退出`synchronized`同步代码块，但偏向锁的Thread ID仍然不会被清除。**

```none
// 偏向锁枷锁前
OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)

// 偏向锁加锁后
OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 f1 45 28 (00000101 11110001 01000101 00101000) (675672325)
      4     4        (object header)                           b2 7f 00 00 (10110010 01111111 00000000 00000000) (32690)
      
// 偏向锁解锁后
OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 f1 45 28 (00000101 11110001 01000101 00101000) (675672325)
      4     4        (object header)                           b2 7f 00 00 (10110010 01111111 00000000 00000000) (32690)
```

#### 1.2.2.6 偏向锁撤销

触发偏向锁撤销的方式主要有以下几种：

+ 调用偏向锁对象的`hashCode()`方法<small>（正常对象一开始没有hashCode，第一次调用`hashCode()`时才生成）</small>。（升级为轻量级锁）

  *(轻量级锁会在锁记录中记录hashCode)*

  *(重量级锁会在Monitor中记录hashCode)*

+ 存在其他线程抢占偏向锁（升级为轻量级锁）
+ 线程使用`wait()`方法（`wait()`需要Monitor管程才能正常工作）。（升级为重量级锁）

----

1. 调用`hashCode()`方法

   ```java
   public class ThreadTest005 {
       // 运行前添加虚拟机参数，取消偏向锁延迟 -XX:BiasedLockingStartupDelay=0
       public static void main(String[] args) throws InterruptedException {
   
           Object lock = new Object();
           ClassLayout classLayout = ClassLayout.parseInstance(lock);
           // 调用hashCode()之前
           System.out.println(classLayout.toPrintable());
           lock.hashCode();
           new Thread(() -> {
               // 调用hashCode()之后，执行同步代码块之前
               System.out.println(classLayout.toPrintable());
               synchronized (lock) {
                   System.out.println(classLayout.toPrintable());
               }
               System.out.println(classLayout.toPrintable());
           }, "t1").start();
       }
   
   }
   ```

   执行结果如下:(只摘取每次输出的Mark Word)

   很明显，调用`hashCode()`之前，对象还是正常的匿名偏向锁。调用了`hashCode()`之后，偏向锁标志位变为0，变成普通的无锁对象。且之后的加锁解锁都是轻量级锁的流程。

   ```none
   // 调用hashCode()之前
   OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
         4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
         
   // 调用hashCode()之后，执行同步代码块之前
   OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           01 84 08 20 (00000001 10000100 00001000 00100000) (537428993)
         4     4        (object header)                           3f 00 00 00 (00111111 00000000 00000000 00000000) (63)
         
   OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           70 78 0b 3d (01110000 01111000 00001011 00111101) (1024161904)
         4     4        (object header)                           97 7f 00 00 (10010111 01111111 00000000 00000000) (32663)
   
   OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           01 84 08 20 (00000001 10000100 00001000 00100000) (537428993)
         4     4        (object header)                           3f 00 00 00 (00111111 00000000 00000000 00000000) (63)
   ```

2. 其他线程抢占偏向锁

   + 代码1:（偏向锁-> 轻量级锁 -> 重量级锁）

     ```java
     public class ThreadTest005 {
         // 运行前添加虚拟机参数，取消偏向锁延迟 -XX:BiasedLockingStartupDelay=0
         public static void main(String[] args) throws InterruptedException {
     
             Object lock = new Object();
             ClassLayout classLayout = ClassLayout.parseInstance(lock);
     
             new Thread(() -> {
                 System.out.println("t1 -- before sync:\n"+classLayout.toPrintable());
                 synchronized (lock) {
                     System.out.println("t1 -- sync:\n"+classLayout.toPrintable());
                 }
                 System.out.println("t1 -- after sync:\n"+classLayout.toPrintable());
             }, "t1").start();
     
             new Thread(() -> {
                 System.out.println("t2 -- before sync:\n"+classLayout.toPrintable());
                 synchronized (lock) {
                     System.out.println("t2 -- sync:\n"+classLayout.toPrintable());
                 }
                 System.out.println("t2 -- after sync:\n"+classLayout.toPrintable());
             }, "t2").start();
         }
     
     }
     ```

     输出（只截取部分内容）：

     可以看出来这种代码写法，一开始两个线程正式抢占锁之前，对象的后3bit101表示匿名偏向锁状态。之后由于偏向锁抢占，t1输出的信息表明锁升级为轻量级锁，而t2执行时更是升级为重量级锁。（我的电脑才4核，猜测是因为轻量级锁竞争线程数 >= CPU核数的1/2导致之后轻量级锁升级为重量级锁）

     ```java
     t1 -- before sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
           4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
          
     t2 -- before sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
           4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
          
     t1 -- sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           70 18 25 a4 (01110000 00011000 00100101 10100100) (-1541072784)
           4     4        (object header)                           f1 7f 00 00 (11110001 01111111 00000000 00000000) (32753)
     
     t2 -- sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           0a 41 00 88 (00001010 01000001 00000000 10001000) (-2013249270)
           4     4        (object header)                           f1 7f 00 00 (11110001 01111111 00000000 00000000) (32753)
          
     t1 -- after sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           0a 41 00 88 (00001010 01000001 00000000 10001000) (-2013249270)
           4     4        (object header)                           f1 7f 00 00 (11110001 01111111 00000000 00000000) (32753)
          
     t2 -- after sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           0a 41 00 88 (00001010 01000001 00000000 10001000) (-2013249270)
           4     4        (object header)                           f1 7f 00 00 (11110001 01111111 00000000 00000000) (32753)
     ```

   + 代码2：使用`LockSupport`的`park`和`unpark`，保证线程t2在线程t1执行完偏向锁锁定的同步代码块后，才执行`synchronized`修饰的代码块

     ```java
     public class ThreadTest005 {
     
         // 运行前添加虚拟机参数，取消偏向锁延迟 -XX:BiasedLockingStartupDelay=0
         public static void main(String[] args){
     
             Object lock = new Object();
             ClassLayout classLayout = ClassLayout.parseInstance(lock);
     
             Thread t2 = new Thread(() -> {
     
                 LockSupport.park(ThreadTest005.class); // 确保t2线程在t1退出同步方法块后运行
     
                 System.out.println("t2 -- before sync:\n"+classLayout.toPrintable());
     
                 synchronized (lock) {
                     System.out.println("t2 -- sync:\n"+classLayout.toPrintable());
                 }
                 System.out.println("t2 -- after sync:\n"+classLayout.toPrintable());
             }, "t2");
     
             Thread t1 = new Thread(() -> {
     
                 System.out.println("t1 -- before sync:\n"+classLayout.toPrintable());
                 synchronized (lock) {
                     System.out.println("t1 -- sync:\n"+classLayout.toPrintable());
                 }
                 System.out.println("t1 -- after sync:\n"+classLayout.toPrintable());
     
                 LockSupport.unpark(t2);
     
             }, "t1");
     
             t1.start();
             t2.start();
         }
     
     }
     ```

     输出结果（只提取关键信息）：

     ​	这里t1和t2线程实际运行时不存在偏向锁竞争，因为t2等待t1执行结束后才执行。偏向锁占用者在执行完同步方法后，并不会主动再CAS一次来修改锁对象的Mark Word。所以t2线程CAS准备占用偏向锁时，发现偏向锁已经有Thread ID记录且与自身记录不同，则偏向锁加锁失败，由于原本占有锁的线程已经不再占用该偏向锁对象，所以t2撤消偏向锁，并且让锁对象升级为轻量级锁。可以看到，<u>t2升级轻量级锁后，将对象Mark Word的可偏向标志位置0，并且轻量级锁释放后，会恢复对象的Mark Word（但是原本修改的可偏向位改成0不会重新变为1）</u>。

     ```none
     t1 -- before sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
           4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     
     t1 -- sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           05 38 43 ec (00000101 00111000 01000011 11101100) (-331139067)
           4     4        (object header)                           6a 7f 00 00 (01101010 01111111 00000000 00000000) (32618)
     
     t1 -- after sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           05 38 43 ec (00000101 00111000 01000011 11101100) (-331139067)
           4     4        (object header)                           6a 7f 00 00 (01101010 01111111 00000000 00000000) (32618)
           
     t2 -- before sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           05 38 43 ec (00000101 00111000 01000011 11101100) (-331139067)
           4     4        (object header)                           6a 7f 00 00 (01101010 01111111 00000000 00000000) (32618)
     
     t2 -- sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           70 a8 7b d4 (01110000 10101000 01111011 11010100) (-730093456)
           4     4        (object header)                           6a 7f 00 00 (01101010 01111111 00000000 00000000) (32618)
     
     t2 -- after sync:
      OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
           0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
           4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     ```

3. 在同步代码块中使用`wait`操作

   代码：前面说过了，`synchronized`中使用wait，将导致锁直接升为重量级锁。（因为`wait`和`notify/notifyAll`依赖Monitor，而Monitor管程使用的是操作系统级别的重量级锁。

   ```java
   // 运行前添加虚拟机参数，取消偏向锁延迟 -XX:BiasedLockingStartupDelay=0
   public static void main(String[] args){
       Object lock = new Object();
       ClassLayout classLayout = ClassLayout.parseInstance(lock);
   
       Thread t1 = new Thread(() -> {
           System.out.println("t1 -- before sync:\n"+classLayout.toPrintable());
           synchronized (lock) {
               System.out.println("t1 -- sync:\n"+classLayout.toPrintable());
               try {
                   lock.wait();
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
           System.out.println("t1 -- after sync:\n"+classLayout.toPrintable());
       }, "t1");
   
       Thread t2 = new Thread(() -> {
           System.out.println("t2 -- before sync:\n"+classLayout.toPrintable());
           synchronized (lock) {
               System.out.println("t2 -- sync:\n"+classLayout.toPrintable());
               lock.notifyAll();
           }
           System.out.println("t2 -- after sync:\n"+classLayout.toPrintable());
       }, "t2");
       
       t1.start();
       t2.start();
   }
   ```

   输出如下（同样只截取关键信息）：明显升级成了重量级锁（Mark Word最后2bit为00）

   ```java
   t1 -- before sync:
    OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
         4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
   
   t1 -- sync:
    OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           05 61 41 6c (00000101 01100001 01000001 01101100) (1816224005)
         4     4        (object header)                           3a 7f 00 00 (00111010 01111111 00000000 00000000) (32570)
        
   t2 -- before sync:
    OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           8a 46 00 34 (10001010 01000110 00000000 00110100) (872433290)
         4     4        (object header)                           3a 7f 00 00 (00111010 01111111 00000000 00000000) (32570)
   
   t2 -- sync:
    OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           8a 46 00 34 (10001010 01000110 00000000 00110100) (872433290)
         4     4        (object header)                           3a 7f 00 00 (00111010 01111111 00000000 00000000) (32570)
   
   t1 -- after sync:
    OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           8a 46 00 34 (10001010 01000110 00000000 00110100) (872433290)
         4     4        (object header)                           3a 7f 00 00 (00111010 01111111 00000000 00000000) (32570)
   
   t2 -- after sync:
    OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
         0     4        (object header)                           8a 46 00 34 (10001010 01000110 00000000 00110100) (872433290)
         4     4        (object header)                           3a 7f 00 00 (00111010 01111111 00000000 00000000) (32570)
   ```

----

> [【转载】Java中的锁机制 synchronized & 偏向锁 & 轻量级锁 & 重量级锁 & 各自优缺点及场景 & AtomicReference](https://www.cnblogs.com/charlesblc/p/5994162.html) <== 以下内容出至该文章

![img](https://images2015.cnblogs.com/blog/899685/201610/899685-20161025102843468-151954717.png)

上图中只讲了偏向锁的释放，其实还涉及偏向锁的抢占，其实就是两个进程对锁的抢占，在`synchronized`锁下表现为**轻量锁方式进行抢占**。

注：也就是说**一旦偏向锁冲突，双方都会升级为轻量级锁**。（这一点与轻量级->重量级锁不同，那时候失败一方直接升级，成功一方在释放时候notify，加下文后面详细描述）

如下图。之后会进入到轻量级锁阶段，两个线程进入锁竞争状态（注，我理解仍然会遵守先来后到原则；注2，的确是的，下图中提到了mark word中的lock record指向堆栈中最近的一个线程的lock record），一个具体例子可以参考`synchronized`锁机制。（图后面有介绍）

![img](https://images2015.cnblogs.com/blog/899685/201610/899685-20161025103757531-1715338870.jpg)

> [JAVA虚拟机锁机制的升级流程](https://www.iteye.com/blog/xly1981-1766224)
>
> 每一个线程在准备获取共享资源时：
> 第一步，检查MarkWord里面是不是放的自己的ThreadId ,如果是，表示当前线程是处于 “偏向锁”
> 第二步，如果MarkWord不是自己的ThreadId,锁升级，这时候，用CAS来执行切换，新的线程根据MarkWord里面现有的ThreadId，通知之前线程暂停，之前线程将Markword的内容置为空。
> 第三步，两个线程都把对象的HashCode复制到自己新建的用于存储锁的记录空间，接着开始通过CAS操作，把共享对象的MarKword的内容修改为自己新建的记录空间的地址的方式竞争MarkWord,
> 第四步，第三步中成功执行CAS的获得资源，失败的则进入自旋
> 第五步，自旋的线程在自旋过程中，成功获得资源(即之前获的资源的线程执行完成并释放了共享资源)，则整个状态依然处于 轻量级锁的状态，如果自旋失败
> 第六步，进入重量级锁的状态，这个时候，自旋的线程进行阻塞，等待之前线程执行完成并唤醒自己


#### 1.2.2.7 偏向锁批量重偏向和批量撤销

> **[synchronized原理和锁优化策略(偏向/轻量级/重量级)](https://my.oschina.net/lscherish/blog/3117851)** <=== **推荐阅读，很详细**。以下内容出至该文

​	只有一个线程反复进入同步块时，偏向锁带来的性能开销基本可以忽略，但是当有其他线程尝试获得锁时，就需要等到safe point时将偏向锁撤销为无锁状态或升级为轻量级/重量级锁。**safe point这个词我们在GC中经常会提到，其代表了一个状态，在该状态下所有线程都是暂停的**。总之，偏向锁的撤销是有一定成本的，**如果说运行时的场景本身存在多线程竞争的，那偏向锁的存在不仅不能提高性能，而且会导致性能下降**。因此，JVM中增加了一种批量重偏向/撤销的机制。

存在如下两种情况：

1. 一个线程创建了大量对象并执行了初始的同步操作，之后在另一个线程中将这些对象作为锁进行之后的操作。这种case下，会导致大量的偏向锁撤销操作。
2. 存在明显多线程竞争的场景下使用偏向锁是不合适的，例如生产者/消费者队列。 批量重偏向（bulk rebias）机制是为了解决第一种场景。批量撤销（bulk revoke）则是为了解决第二种场景。

​	其做法是：**以class为单位，为每个class维护一个偏向锁撤销计数器，每一次该class的对象发生偏向撤销操作时，该计数器+1，当这个值达到重偏向阈值（默认20，jvm参数`BiasedLockingBulkRebiasThreshold`控制）时，JVM就认为该class的偏向锁有问题，因此会进行批量重偏向。**

​	当**达到重偏向阈值后，假设该class计数器继续增长，当其达到批量撤销的阈值后（默认40，jvm参数`BiasedLockingBulkRevokeThreshold`控制），JVM就认为该class的使用场景存在多线程竞争，执行批量撤销，会标记该class为不可偏向，之后，对于该class的锁，直接走轻量级锁的逻辑。**

​	`BiasedLockingDecayTime`是开启一次新的批量重偏向距离上次批量重偏向的后的延迟时间，默认25000。也就是开启批量重偏向后，如果经过了一段较长的时间（>=`BiasedLockingDecayTime`），撤销计数器才超过阈值，那我们会重置计数器。

#### 1.2.2.8 偏向锁批量锁重偏向

> **[synchronized原理和锁优化策略(偏向/轻量级/重量级)](https://my.oschina.net/lscherish/blog/3117851)** <=== **推荐阅读，很详细**

​	介绍完偏向，我们发现如果锁先偏向了线程B，那么等另外任何一个线程来竞争的时候，都会导致进入偏向锁的撤销流程，在撤销流程里，才会判断线程B是否还活着，如果已经不活动了，则重偏向。

​	但**偏向锁的撤销流程需要等到全局安全点**，这是一个极大的消耗，为了能够让许多本应该重偏向的偏向锁无须等到全局安全点时才被重偏向，jvm引入了批量重偏向的逻辑。(默认阈值是20,即进行第20次某class的偏向锁撤销时触发批量锁重偏向。)

该机制的主要工作原理如下：

- 引入一个概念epoch，其本质是一个时间戳，代表了偏向锁的有效性，epoch存储在可偏向对象的MarkWord中。除了对象中的epoch,对象所属的类class信息中，也会保存一个epoch值
- 每当遇到一个全局安全点时，比如要对class C 进行批量再偏向，则首先对 class C中保存的epoch进行增加操作，得到一个新的epoch_new
- 然后扫描所有持有 class C 实例的线程栈，根据线程栈的信息判断出该线程是否锁定了该对象，仅将epoch_new的值赋给被锁定的对象中。（也就是现在偏向锁还在被使用的对象才会被赋值epoch_new）
- 退出安全点后，当有线程需要尝试获取偏向锁时，直接检查 class C 中存储的 epoch 值是否与目标对象中存储的 epoch 值相等， 如果不相等，则说明该对象的偏向锁已经无效了，可以尝试对此对象重新进行偏向操作。

测试代码如下：

```java
package concurrency;
import org.openjdk.jol.info.ClassLayout;
import java.util.Vector;

public class ThreadTest006 {

    static class Tmp{}

    public static void main(String[] args) {

        Vector<Tmp> locks = new Vector<>();

        Thread t1 = new Thread(()->{
            for(int i = 0;i<25;++i){
                System.out.println("  -------  "+i+"  -------  ");
                Tmp lock = new Tmp();
                locks.add(lock);
                synchronized (lock){
                    System.out.println(i+ClassLayout.parseInstance(lock).toPrintable());
                }
            }
            synchronized (locks){
                locks.notify();
            }
            /*try {
                Thread.sleep(500000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }*/
        },"t1");

        t1.start();

        Thread t2 = new Thread(()->{

            synchronized (locks){
                try {
                    locks.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            System.out.println("==================================================");

            for(int i = 0;i<25;++i){
                System.out.println("  -------  "+i+"  -------  ");
                Object lock = locks.get(i);

                System.out.println(i+ClassLayout.parseInstance(lock).toPrintable());
                synchronized (lock){
                    System.out.println(i+ClassLayout.parseInstance(lock).toPrintable());
                }
                System.out.println(i+ClassLayout.parseInstance(lock).toPrintable());

            }

        },"t2");

        t2.start();

        try {
            t2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

       // 验证偏向锁 的 批量重偏向是class级别的(并且只对重偏向阈值之后的对象锁生效,因为前19次偏向锁撤销后，对应的对象锁Mark Word已标识为不可偏向)
        for (int i = 0;i<25;++i){
            System.out.println(i+ClassLayout.parseInstance(locks.get(i)).toPrintable());
        }

        Tmp newOne = new Tmp();
        System.out.println("newOne:\n"+ClassLayout.parseInstance(newOne).toPrintable());
    }
}
```

最后输出可以看到：

​	t1线程使用的25个对象的偏向锁MarkWord一致。

​	而t2线程，0～18的这前19次偏向锁撤销，都是走的CAS轻量级锁的流程;但是19～24，后面这6次偏向锁竞争，变成了锁重新偏向了t2线程。

​	验证了JDK8环境下，偏向锁重偏向的默认阈值为20。当同一个对象锁发生第20次锁撤销时，触发偏向锁重定向。这个偏向锁重定向是class级别的，所有该class下的对象中Mark Word的可偏向标志位为1的对象，将重偏向到线程t2。新建的该class下的锁对象，并不受影响（不管是否发生过批量重偏向，这个把代码的循环25次改成循环15次，最后输出的newOne对象的Mark Word依旧是后3bit为101）

```none
// 只截取部分内容

  -------  0  -------  
0concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 a8 2c 1c (00000101 10101000 00101100 00011100) (472688645)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)
      
      
      .........


  -------  24  -------  
24concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 a8 2c 1c (00000101 10101000 00101100 00011100) (472688645)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)
      
==================================================
  -------  0  -------  
0concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 a8 2c 1c (00000101 10101000 00101100 00011100) (472688645)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)

0concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           68 18 db 00 (01101000 00011000 11011011 00000000) (14358632)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)

0concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      
      
      ..........


  -------  18  -------  
18concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 a8 2c 1c (00000101 10101000 00101100 00011100) (472688645)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)

18concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           68 18 db 00 (01101000 00011000 11011011 00000000) (14358632)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)

18concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      
  -------  19  -------  
19concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 a8 2c 1c (00000101 10101000 00101100 00011100) (472688645)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)

19concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 c9 2c 1c (00000101 11001001 00101100 00011100) (472697093)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)

19concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 c9 2c 1c (00000101 11001001 00101100 00011100) (472697093)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)
      
      
      ........
      

  -------  24  -------  
24concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 a8 2c 1c (00000101 10101000 00101100 00011100) (472688645)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)

24concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 c9 2c 1c (00000101 11001001 00101100 00011100) (472697093)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)

24concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 c9 2c 1c (00000101 11001001 00101100 00011100) (472697093)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)
      
      ..........
      
0concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      
      ..........
      
18concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)

19concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 c9 2c 1c (00000101 11001001 00101100 00011100) (472697093)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)
      
      ...........
      
24concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 c9 2c 1c (00000101 11001001 00101100 00011100) (472697093)
      4     4        (object header)                           06 7f 00 00 (00000110 01111111 00000000 00000000) (32518)
      
      .............
      
newOne:
concurrency.ThreadTest006$Tmp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 01 00 00 (00000101 00000001 00000000 00000000) (261)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
```

#### 1.2.2.9 偏向锁批量锁撤销

> **[synchronized原理和锁优化策略(偏向/轻量级/重量级)](https://my.oschina.net/lscherish/blog/3117851)** <=== **推荐阅读，很详细**

偏向锁的批量锁撤销的阈值是40。当某个class下的对象偏向锁撤销达到第40次时，JVM会将该class下的所有对象都变成不可偏向的。之后新建的对象也都是不可偏向的。

**同一个类class，批量锁重定向和批量锁撤销 只会发生一次。**（设置成不可偏向是整个class而言的，非单独几个对象）

- 将类的偏向标记关闭，之后当该类已存在的实例获得锁时，就会升级为轻量级锁；该类新分配的对象的mark word则是无锁模式。
- 处理当前正在被使用的锁对象，通过遍历所有存活线程的栈，找到所有正在使用的偏向锁对象，然后撤销偏向锁。

测试代码：循环39和38次。3个线程。

```java
package concurrency;

import org.openjdk.jol.info.ClassLayout;

import java.util.Vector;

/**
 * 偏向锁 批量撤销
 */
public class ThreadTest007 {

    static class Temp {
    }

    static Thread t1, t2, t3;

    public static void main(String[] args) {

        Vector<Temp> locks = new Vector<>();

        int round = 39; // 38

        t1 = new Thread(() -> {
            for (int i = 0; i < round; ++i) {
                System.out.println("  -------  " + Thread.currentThread().getName() + "   " + i + "  -------  \n");
                Temp lock = new Temp();
                locks.add(lock);
                synchronized (lock) {
                    System.out.println(i + ClassLayout.parseInstance(lock).toPrintable());
                }
            }
        }, "t1");

        t1.start();

        t2 = new Thread(() -> {

            try {
                t1.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("==================================================");

            for (int i = 0; i < round; ++i) {
                System.out.println("  -------  " + Thread.currentThread().getName() + "   " + i + "  -------  \n");
                Temp lock = locks.get(i);

                System.out.println(i + ClassLayout.parseInstance(lock).toPrintable());
                synchronized (lock) {
                    System.out.println(i + ClassLayout.parseInstance(lock).toPrintable());
                }
                System.out.println(i + ClassLayout.parseInstance(lock).toPrintable());

            }


        }, "t2");

        t2.start();

        t3 = new Thread(() -> {

            try {
                t2.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("==================================================");

            for (int i = 0; i < round; ++i) {
                System.out.println("  -------  " + Thread.currentThread().getName() + "   " + i + "  -------  \n");
                Temp lock = locks.get(i);

                System.out.println(i + ClassLayout.parseInstance(lock).toPrintable());
                synchronized (lock) {
                    System.out.println(i + ClassLayout.parseInstance(lock).toPrintable());
                }
                System.out.println(i + ClassLayout.parseInstance(lock).toPrintable());

            }

        }, "t3");

        t3.start();

        try {
            t3.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("+-++-+-+-+-+-+-+-+---+--+-+-+-+-+--+-");

       
        for (int i = 0; i < round; ++i) {
            System.out.println(i + " ---- \n" + ClassLayout.parseInstance(locks.get(i)).toPrintable());
        }

        System.out.println("+-++-+-+-+-+-+-+-+---+--+-+-+-+-+--+-");

        Temp newOne = new Temp();
        System.out.println("newOne:\n" + ClassLayout.parseInstance(newOne).toPrintable());
    }

}

```



输出结果：

​	（t1、t2、t3。t2在偏向锁撤销第20次时触发过class的批量重定向后，下次t3在第20次锁撤销时并不会在触发批量锁重定向，但是t3在第39次批量锁撤销时触发批量锁撤销，之后整个类class的新对象也会被标志成不可偏向。

**设置成不可偏向，是针对整个class而言的**。

+ round设置39，则最后新建出来的对象，被标志为“不可偏向” （Mark Word后3bit 001）
+ round设置38，则最后新建出来的对象，被标志为“可偏向的” （Mark Word后3bit 101）

```none
...... t1都是 偏向自己的锁，没什么好看的
  -------  t1   38  -------  

38concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 38 31 e4 (00000101 00111000 00110001 11100100) (-466536443)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)
      
      
      .........
      
==================================================
  -------  t2   0  -------  

0concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 38 31 e4 (00000101 00111000 00110001 11100100) (-466536443)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)
      
      ..........
      
  -------  t2   18  -------  

18concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 38 31 e4 (00000101 00111000 00110001 11100100) (-466536443)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)

18concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           60 38 55 d0 (01100000 00111000 01010101 11010000) (-799721376)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)

18concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      
 -------  t2   19  -------  

19concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 38 31 e4 (00000101 00111000 00110001 11100100) (-466536443)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

19concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 51 31 e4 (00000101 01010001 00110001 11100100) (-466530043)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)

19concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 51 31 e4 (00000101 01010001 00110001 11100100) (-466530043)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)
      
      
     ........
     
       -------  t2   38  -------  

38concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 38 31 e4 (00000101 00111000 00110001 11100100) (-466536443)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)

38concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 51 31 e4 (00000101 01010001 00110001 11100100) (-466530043)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)

38concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 51 31 e4 (00000101 01010001 00110001 11100100) (-466530043)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)
      
      
      ..........
      
      ==================================================
  -------  t3   0  -------  

0concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      
0concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           60 28 45 d0 (01100000 00101000 01000101 11010000) (-800774048)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)

0concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      
     ......
     
       -------  t3   38  -------  

38concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 51 31 e4 (00000101 01010001 00110001 11100100) (-466530043)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)

38concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           60 28 45 d0 (01100000 00101000 01000101 11010000) (-800774048)
      4     4        (object header)                           42 7f 00 00 (01000010 01111111 00000000 00000000) (32578)

38concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      
      
      ........
      
38 ---- 
concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

+-++-+-+-+-+-+-+-+---+--+-+-+-+-+--+-
newOne:
concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total


// 如果循环round 设置38,最后一小块输出如下：
// 如果循环round 设置38,最后一小块输出如下：
// 如果循环round 设置38,最后一小块输出如下：

37 ---- 
concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

+-++-+-+-+-+-+-+-+---+--+-+-+-+-+--+-
newOne:
concurrency.ThreadTest007$Temp object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 01 00 00 (00000101 00000001 00000000 00000000) (261)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           43 c1 00 f8 (01000011 11000001 00000000 11111000) (-134168253)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

### 1.2.3 synchronized

#### 1.2.3.1 synchronized概述和注意点

> [synchronized 是可重入锁吗？为什么？](https://www.cnblogs.com/incognitor/p/9894604.html)
>
> **[synchronized原理和锁优化策略(偏向/轻量级/重量级)](https://my.oschina.net/lscherish/blog/3117851)** <=== **推荐阅读，很详细**

​	`synchronized`是可重入锁，非公平锁。`synchronized`在JDK6之前使用重量级锁，系统开销大。而JDK6之后，有了完善的锁升级机制，使得`synchronized`的性能大大提升。`synchronized`底层也是依靠CAS操作。

*<small>（纵使java还有AQS、原子类等各种花样的类可用于同步互斥，但是其实平时开发还是以使用`synchronized`为主。锁升级使得`synchronized`在很多时候性能并不亚于其他同步互斥操作。建议非底层部件开发or非中间件开发，如果真有需要同步互斥，用`synchronized`即可。虽然大多数时候还是直接借助Spring的事务机制完成数据库的同步互斥操作保证ACID就够了。）</small>*

​	`synchronized`的用法很简单，一般就2种写法：

1. 同步代码块

   （使用我们指定的对象作为"锁"）

   ```java
   public class ThreadTest001 {
       @Test
       public void test1(){
           synchronized (this){
               System.out.println(this); // 输出concurrency.ThreadTest001@67784306
           }
           synchronized (new Object()){
   
           }
       }
   }
   ```

2. 同步"方法"

   （相当于使用this作为"锁"的同步代码块，this即类的java.lang.Class对象。所有java类的class字节码文件加载到内存中，都会有对应的java.lang.Class类对象）

   （所有Java类在初始化时经历3步骤：内存分配、根据默认构造函数初始化内存区域内的成员变量、Class指针指向初始化后的内存）

   ```java
   public class ThreadTest001 {
       @Test
       public synchronized void test(){ // 相当于使用synchronized(this){}代码块
           System.out.println(this);  // 输出concurrency.ThreadTest001@67784306
       }
   }
   ```

使用`synchronized`进行互斥同步操作时，需要注意以下几点：

+ `synchronized`是非公平、可重入锁

+ 充当锁的对象，不要使用String常量、基本类型包装类（Byte、Short、Integer、Long、Float、Double、Boolean、Character）

  <small>因为String常量有常量池易被混淆，且有些jar包里面用的就是String常量作为锁，容易导致难以发现的错误。</small>

  <small>包装类的话，除了浮点数Float和Double没缓存，其他数值类型（Character是0～127）会缓存-128～127的数值，而Boolean缓存了value为true和false的两个对象。**包装类的自动拆箱和自动装箱是编译器默认认可的，自动装箱会导致包装类对象引用指向新的包装类对象**。</small> ==> 建议阅读 [[Java synchronized 中使用Integer对象遇到的问题](https://www.jianshu.com/p/b7c5c8bd9702)]

  <small>java的锁是根据对象的hashCode来区分的，hashCode不一样则认为不是同一个锁对象。</small>

+ `synchronized`在JDK6开始有完善的锁升级机制

+ 锁对象的hashCode**只有调用了Object的`hashCode()`方法后，hashCode才会写入对象头的Mark Word中**。（继承后重写的`hashCode()`不会触发写入对象头的事件）

+ **带有锁的线程异常退出时，默认情况会释放锁**

+ **同一个对象的对象头Mark Word中的hashCode只会计算一次**。

  OpenJDK8 默认hashCode的计算方法是通过**和当前线程有关的一个随机数+三个确定值**，运用Marsaglia's xorshift scheme随机数算法得到的一个随机数。**和对象内存地址无关**。<u>生成的hashcode会放在对象的头部，已生成对象的hashcode不会变化，所以虽然不在同一个线程，但是是同一个对象，hashcode不会变化。</u> ===> 出至[[Java Object.hashCode()返回的是对象内存地址？](https://www.jianshu.com/p/be943b4958f4)]



----

##### 调用Object.hashCode()才会更新对象头MarkWord的hashCode

> **[Java 对象头中你可能不知道的事](https://blog.csdn.net/L__ear/article/details/105486403)** <== 建议阅读
>
> **[Java Object.hashCode()返回的是对象内存地址？](https://www.jianshu.com/p/be943b4958f4)** <== 推荐阅读
>
> OpenJDK8 默认hashCode的计算方法是通过**和当前线程有关的一个随机数+三个确定值**，运用Marsaglia's xorshift scheme随机数算法得到的一个随机数。**和对象内存地址无关**。<u>生成的hashcode会放在对象的头部，已生成对象的hashcode不会变化，所以虽然不在同一个线程，但是是同一个对象，hashcode不会变化。</u>



测试代码如下：

```java
package concurrency;

import org.junit.Test;
import org.openjdk.jol.info.ClassLayout;

public class ThreadTest008 {

    class Temp_001 { // 没有重写Object的hashCode，默认调用Object.hashCode()

    }

    class Temp_002 {// 重写Object的hashCode()，但是内部调用了Object.hashCode()

        @Override
        public int hashCode() {
            return super.hashCode();
        }
    }

    class Temp_003 {// 重写Object的hashCode()，但是内部没有调用Object.hashCode()

        @Override
        public int hashCode() {
            //return super.hashCode();
            return 0xFFFF;
        }
    }

    @Test
    public void test1() {

        // 预期因为调用了Object.hashCode()，将hashCode写入对象头MarkWord
        Temp_001 tmp_001 = new Temp_001();
        System.out.println(ClassLayout.parseInstance(tmp_001).toPrintable());
        tmp_001.hashCode();
        System.out.println(ClassLayout.parseInstance(tmp_001).toPrintable());
        System.out.println(tmp_001);

        System.out.println("---------------------------------------------------");

        // 预期因为重写的方法hashCode()中调用了Object.hashCode()，将hashCode写入对象头MarkWord
        Temp_002 tmp_002 = new Temp_002();
        System.out.println(ClassLayout.parseInstance(tmp_002).toPrintable());
        tmp_002.hashCode();
        System.out.println(ClassLayout.parseInstance(tmp_002).toPrintable());
        System.out.println(tmp_002);

        System.out.println("---------------------------------------------------");

        // 预期因为没有调用Object.hashCode()，所以不会修改对象头的MarkWord
        Temp_003 tmp_003 = new Temp_003();
        System.out.println(ClassLayout.parseInstance(tmp_003).toPrintable());
        tmp_003.hashCode();
        System.out.println(ClassLayout.parseInstance(tmp_003).toPrintable());
        System.out.println(tmp_003);
    }
}
```

输出结果：

```java
concurrency.ThreadTest008$Temp_001 object internals:
 OFFSET  SIZE                        TYPE DESCRIPTION                               VALUE
      0     4                             (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                             (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                             (object header)                           38 14 01 f8 (00111000 00010100 00000001 11111000) (-134147016)
     12     4   concurrency.ThreadTest008 Temp_001.this$0                           (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

concurrency.ThreadTest008$Temp_001 object internals:
 OFFSET  SIZE                        TYPE DESCRIPTION                               VALUE
      0     4                             (object header)                           01 73 9c e3 (00000001 01110011 10011100 11100011) (-476286207)
      4     4                             (object header)                           13 00 00 00 (00010011 00000000 00000000 00000000) (19)
      8     4                             (object header)                           38 14 01 f8 (00111000 00010100 00000001 11111000) (-134147016)
     12     4   concurrency.ThreadTest008 Temp_001.this$0                           (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

concurrency.ThreadTest008$Temp_001@13e39c73
---------------------------------------------------
concurrency.ThreadTest008$Temp_002 object internals:
 OFFSET  SIZE                        TYPE DESCRIPTION                               VALUE
      0     4                             (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                             (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                             (object header)                           8c 16 01 f8 (10001100 00010110 00000001 11111000) (-134146420)
     12     4   concurrency.ThreadTest008 Temp_002.this$0                           (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

concurrency.ThreadTest008$Temp_002 object internals:
 OFFSET  SIZE                        TYPE DESCRIPTION                               VALUE
      0     4                             (object header)                           01 52 56 22 (00000001 01010010 01010110 00100010) (576082433)
      4     4                             (object header)                           09 00 00 00 (00001001 00000000 00000000 00000000) (9)
      8     4                             (object header)                           8c 16 01 f8 (10001100 00010110 00000001 11111000) (-134146420)
     12     4   concurrency.ThreadTest008 Temp_002.this$0                           (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

concurrency.ThreadTest008$Temp_002@9225652
---------------------------------------------------
concurrency.ThreadTest008$Temp_003 object internals:
 OFFSET  SIZE                        TYPE DESCRIPTION                               VALUE
      0     4                             (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                             (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                             (object header)                           4d 16 01 f8 (01001101 00010110 00000001 11111000) (-134146483)
     12     4   concurrency.ThreadTest008 Temp_003.this$0                           (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

concurrency.ThreadTest008$Temp_003 object internals:
 OFFSET  SIZE                        TYPE DESCRIPTION                               VALUE
      0     4                             (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                             (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                             (object header)                           4d 16 01 f8 (01001101 00010110 00000001 11111000) (-134146483)
     12     4   concurrency.ThreadTest008 Temp_003.this$0                           (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

concurrency.ThreadTest008$Temp_003@ffff
Disconnected from the target VM, address: '127.0.0.1:47895', transport: 'socket'

Process finished with exit code 0

```

**很明显，只有当调用了`Object.hashCode()`方法时，对象头的MarkWord中的hashCode才会被更新**。



> [java八种基本数据类型及包装类详解](https://blog.csdn.net/qq_37688023/article/details/85106894)
>
> jdk1.5之后包装类的自动拆箱和装箱，该特性是**编译器**认可的。
>
> 1. **自动拆箱 包装类——>基本数据类型** (原理是调用了xxxValue方法)  
> 2. **自动装箱** **基本数据类型——>包装类** (原理是调用了valueOf方法)
>
> ```java
> @Test
> public void d() {
>     /*自动装箱：valueOf*/
>     Integer i=123;//原理是 Integer i=Integer.valueOf(123);
> 
>     /*自动拆箱*/
>     int i1=i+1;//原理是	int i1=i.intValue()+1;
> 
>     Integer a=123;
>     Integer b=123;
>     Integer c=a+b;
>     /*原理为Integer c=Integer.valueOf(a.intValue()+b.intValue());*/
> }
> ```
>
> [在java中==和equals()的区别](https://blog.csdn.net/lcsy000/article/details/82782864)
>
> ==比较对象的堆内存地址，equals默认Object的实现就是==。但是我们可以自己重写equals方法。
>
> （一般自定义的类，都需要重写hashCode方法和equals方法）
>
> **[Java 对象头中你可能不知道的事](https://blog.csdn.net/L__ear/article/details/105486403)**  <=== 文章到底层C++代码分析，建议阅读
>
> 1. **对象创建完毕后，对象头中的 hashCode 为 0**。
> 2. **只有对象调用了从 Object 继承下来的 hashCode 方法，HotSpot 才会把对象 hashCode 写入对象头，否则不会写入**。
>
> 
>
> **[Java equals() and hashCode()](https://www.journaldev.com/21095/java-equals-hashcode)** <= 外国人的优质博客，推荐阅读原文（下面直接搬运内容过来，省得次次翻墙。中间去掉了一些简单代码演示的内容）
>
> Java equals()
>
> ```java
> public boolean equals(Object obj) {
>         return (this == obj);
> }
> ```
>
> According to java documentation of equals() method, any implementation should adhere to following principles.
>
> - For any object x, `x.equals(x)` should return `true`.
> - For any two object x and y, `x.equals(y)` should return `true` if and only if `y.equals(x)` returns `true`.
> - For multiple objects x, y, and z, if `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` should return `true`.
> - Multiple invocations of `x.equals(y)` should return same result, unless any of the object properties is modified that is being used in the `equals()` method implementation.
> - Object class equals() method implementation returns `true` only when both the references are pointing to same object.
>
> 
>
> Java hashCode()
>
> Java Object hashCode() is a native method and returns the integer hash code value of the object. The general contract of hashCode() method is:
>
> - Multiple invocations of hashCode() should return the same integer value, unless the object property is modified that is being used in the equals() method.
> - An object hash code value can change in multiple executions of the same application.
> - If two objects are equal according to equals() method, then their hash code must be same.
> - **If two objects are unequal according to equals() method, their hash code are not required to be different. Their hash code value may or may-not be equal.**
>
> Importance of equals() and hashCode() method
>
> Java hashCode() and equals() method are used in Hash table based implementations in java for storing and retrieving data. I have explained it in detail at [How HashMap works in java?](https://www.journaldev.com/11560/java-hashmap#how-hashmap-works-in-java)
>
> The implementation of equals() and hashCode() should follow these rules.
>
> - **If `o1.equals(o2)`, then `o1.hashCode() == o2.hashCode()` should always be `true`.**
> - **If `o1.hashCode() == o2.hashCode` is true, it doesn’t mean that `o1.equals(o2)` will be `true`.**
>
> 
>
> When to override equals() and hashCode() methods?
>
> When we override equals() method, it’s almost necessary to override the hashCode() method too so that their contract is not violated by our implementation.
>
> **Note that your program will not throw any exceptions if the equals() and hashCode() contract is violated, if you are not planning to use the class as Hash table key, then it will not create any problem.**
>
> If you are planning to use a class as Hash table key, then it’s must to override both equals() and hashCode() methods.
>
> Let’s see what happens when we rely on default implementation of equals() and hashCode() methods and use a custom class as HashMap key.
>
> 
>
> Implementing equals() and hashCode() method
>
> We can define our own equals() and hashCode() method implementation but if we don’t implement them carefully, it can have weird issues at runtime. Luckily most of the IDE these days provide ways to implement them automatically and if needed we can change them according to our requirement.
>
> We can use Eclipse to auto generate equals() and hashCode() methods.
>
> Here is the auto generated equals() and hashCode() method implementations.
>
> ```java
> @Override
> public int hashCode() {
>     final int prime = 31;
>     int result = 1;
>     result = prime * result + id;
>     result = prime * result + ((name == null) ? 0 : name.hashCode());
>     return result;
> }
> 
> @Override
> public boolean equals(Object obj) {
>     if (this == obj)
>         return true;
>     if (obj == null)
>         return false;
>     if (getClass() != obj.getClass())
>         return false;
>     DataKey other = (DataKey) obj;
>     if (id != other.id)
>         return false;
>     if (name == null) {
>         if (other.name != null)
>             return false;
>     } else if (!name.equals(other.name))
>         return false;
>     return true;
> }
> ```
>
> Notice that both equals() and hashCode() methods are using same fields for the calculations, so that their contract remains valid.
>
> We can also use [Project Lombok](https://www.journaldev.com/18124/java-project-lombok) to auto generate equals and hashCode method implementations.
>
> 
>
> What is Hash Collision
>
> In very simple terms, Java Hash table implementations uses following logic for get and put operations.
>
> 1. First identify the “Bucket” to use using the “key” hash code.
> 2. If there are no objects present in the bucket with same hash code, then add the object for put operation and return null for get operation.
> 3. **If there are other objects in the bucket with same hash code, then “key” equals method comes into play.**
>    - **If equals() return true and it’s a put operation, then object value is overridden.**
>    - **If equals() return false and it’s a put operation, then new entry is added to the bucket.**
>    - **If equals() return true and it’s a get operation, then object value is returned.**
>    - **If equals() return false and it’s a get operation, then null is returned.**
>
> Below image shows a bucket items of HashMap and how their equals() and hashCode() are related.
>
> ![java hashmap, how hashmap works in java](https://cdn.journaldev.com/wp-content/uploads/2013/01/java-hashmap-entry-impl.png)
>
> **The phenomenon when two keys have same hash code is called hash collision**. If hashCode() method is not implemented properly, there will be higher number of hash collision and map entries will not be properly distributed causing slowness in the get and put operations. <u>This is the reason for prime number usage in generating hash code so that map entries are properly distributed across all the buckets.</u>
>
> 
>
> What if we don’t implement both hashCode() and equals()?
>
> We have already seen above that if hashCode() is not implemented, we won’t be able to retrieve the value because HashMap use hash code to find the bucket to look for the entry.
>
> If we only use hashCode() and don’t implement equals() then also value will be not retrieved because equals() method will return false.
>
> 
>
> Best Practices for implementing equals() and hashCode() method
>
> - **Use same properties in both equals() and hashCode() method implementations, so that their contract doesn’t violate when any properties is updated.**
> - **It’s better to use immutable objects as Hash table key so that we can cache the hash code rather than calculating it on every call. That’s why String is a good candidate for Hash table key because it’s immutable and cache the hash code value.**
> - Implement hashCode() method so that least number of hash collision occurs and entries are evenly distributed across all the buckets.



#### 1.2.3.2 synchronized内部实现

##### java字节码层面

> [jvms-6--monitorenter](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorenter) <= 官方定义、介绍monitorenter
>
> The *objectref* must be of type `reference`.
>
> Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes *monitorenter* attempts to gain ownership of the monitor associated with *objectref*, as follows:
>
> - If the entry count of the monitor associated with *objectref* is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
> - If the thread already owns the monitor associated with *objectref*, it reenters the monitor, incrementing its entry count.
> - If another thread already owns the monitor associated with *objectref*, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.
>
> .....（还有其他内容，建议直接点击上面链接阅读官方介绍文章）
>
> <u>下面monitorenter和monitorexit的可能抛出的异常，这里没有写出来，可以直接到官方原文看</u>。

`synchronized`在java字节码中，以`monitorenter`和`monitorexit`的形式出现。

1. monitorenter

   ​	根据官方对`monitorenter`的定义和介绍可知，每个Java对象都有一个与之关联的monitor。Monitor当且仅当有一个对应的`owner`时，才会被锁住。执行`monitorenter`的线程尝试通过引用关联monitor，获取monitor的所有权（也就是成为Monitor数据结构中的`owner`）。

   + 如果当前线程是第一个与monitor产生关联的，那么将monitor的entry count数值设置为0，并且成为monitor的`owner`

   + 如果当前线程已经拥有与该monitor关联的引用（指针），那么线程重入monitor时，将entry count的计数+1

   + 如果其他线程已经通过对象引用和monitor产生关联，那么该尝试访问monitor的线程阻塞，直到monitor的entry count计数为0时，该后来者线程才能尝试竞争获取monitor的所有权

   注意：

   ​	在java的`synchronized`实现中，**1个`monitorenter`可能对应1个或n个`monitorexit`**。尽管`monitorenter`和`monitorexit`不会在同步方法的java实现代码中直接出现，但是他们俩能够提供等效于锁的语义。<small>（换言之就是java中没有直接使用monitorenter和monitorexit的方法，但是他们确实存在，且能提供等效于锁的功能）</small>JVM隐式地在同步代码块/方法前使用`monitorenter`，并且在同步代码块/方法结束return前使用`monitorexit`。<small>（使用`synchronized`时，JVM帮我们自动在字节码层面添加`monitorenter`和`monitorexit`，使得往后实际的汇编代码执行能够同步互斥）</small>

   ​	<small>The association of a monitor with an object may be managed in various ways that are beyond the scope of this specification. For instance, the monitor may be allocated and deallocated at the same time as the object. Alternatively, it may be dynamically allocated at the time when a thread attempts to gain exclusive access to the object and freed at some later time when no thread remains in the monitor for the object.</small>

   ​	关联monitor的对象以及如何管控该对象，JVM没有严格要求实现该遵守什么标准。比如，关联monitor的对象可能同时被分配和回收。另外，monitor对象可能在某个线程尝试独占关联monitor的对象时动态创建（也就是说可能`synchronized(Object){}`锁定的对象Object所关联的Monitor可能在线程预备独占该对象（重量级锁）时，才临时创建与充当锁的对象Object关联的monitor对象），并且在之后的某个时间段内如果没有线程保留关联monitor的对象（即不占用充当锁的Object）再回收monitor的内存。

   ​	java关于同步功能的数据结构设计，除了支持monitor的entry和exit操作之外，还需要对等待monitor中的线程、唤醒monitor中的线程操作提供支持（`Object.wait`、`Object.notify/notifyAll`）。wait和nontify/notifyAll在`java.lang`包中，需要我们显式在java代码中使用，而非JVM隐式添加monitorenter等来实现。（自己试试看新建两个线程分别用wait和notify，其在java字节码中的体现就是` invokevirtual `，这里不多介绍了，感兴趣看看上面文档）

2. monitorexit

   ​	执行`monitorexit`的线程必须是和monitor关联的对象的拥有者`owner`。执行`monitorexit`后，与monitor关联的对象上记录的entry count计数-1，当值为0时，该线程不再是monitor的`owner`占有者。其他原本想进入monitor而blocking的线程在entry count值为0时，能够再尝试抢占monitor。

   注意：

   ​	JVM支持在`synchronized`同步代码块和同步方法内抛出不同的异常：

   + Monitor exit on normal `synchronized` method completion is handled by the Java Virtual Machine's return instructions. Monitor exit on abrupt `synchronized` method completion is handled implicitly by the Java Virtual Machine's *athrow* instruction.
   + When an exception is thrown from within a `synchronized` statement, exit from the monitor entered prior to the execution of the `synchronized` statement is achieved using the Java Virtual Machine's exception handling mechanism ([§3.14](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-3.html#jvms-3.14))



示例代码1如下：

```java
public class ThreadTest002 {
    public static void main(String[] args) {
        Object o = new Object();
        synchronized (o){
        }
    }
}
```

使用IDEA的Jclasslib方便查看字节码等信息：

```none
0 new #2 <java/lang/Object>
 3 dup
 4 invokespecial #1 <java/lang/Object.<init>>
 7 astore_1
 8 aload_1
 9 dup
10 astore_2
11 monitorenter  // (1) 进入sync代码块
12 aload_2
13 monitorexit  // (2) sync代码块正常退出，则goto return那行（字节码等于抽象的汇编语言了）
14 goto 22 (+8)
17 astore_3
18 aload_2
19 monitorexit  // (3) sync代码块异常退出
20 aload_3
21 athrow
22 return
```

----

##### Java的native方法对应的C++代码

> **[JVM源码分析之synchronized实现](https://www.jianshu.com/p/c5058b6fe8e5) <== 强烈建议阅读**
>
> **强烈建议阅读**
>
> **强烈建议阅读**

上面这篇文章讲解还不错了。建议阅读。这里就简单讲讲。

网上下载OpenJDK代码，然后进到`OpenJDK/hotspot-37240c1019fd/src/share/vm/interpreter/`目录，查看`InterpreterRuntime.cpp`就可以看到monitorenter和monitorexit的调用了。

```c++
//%note monitor_1
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
    #ifdef ASSERT
    thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
if (PrintBiasedLockingStatistics) {
    Atomic::inc(BiasedLocking::slow_path_entry_count_addr());
}
Handle h_obj(thread, elem->obj());
assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
       "must be NULL or an object");
if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
} else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
}
assert(Universe::heap()->is_in_reserved_or_null(elem->obj()),
       "must be NULL or an object");
#ifdef ASSERT
thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
IRT_END
```

上面可以看到java中`synchronized`修饰的代码，底层C++实现，首先判断是否启用偏向锁，如果启动用则使用偏向锁（`ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);`），否则使用自旋锁（` ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);`）

```c++
//%note monitor_1
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorexit(JavaThread* thread, BasicObjectLock* elem))
    #ifdef ASSERT
    thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
Handle h_obj(thread, elem->obj());
assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
       "must be NULL or an object");
if (elem == NULL || h_obj()->is_unlocked()) {
    THROW(vmSymbols::java_lang_IllegalMonitorStateException());
}
ObjectSynchronizer::slow_exit(h_obj(), elem->lock(), thread);
// Free entry. This must be done here, since a pending exception might be installed on
// exit. If it is not cleared, the exception handling code will try to unlock the monitor again.
elem->set_obj(NULL);
#ifdef ASSERT
thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
IRT_END
```



1.2.3.3 留着之后可能补充

1.2.3.4 留着之后可能补充

### 1.2.4 volatile

#### 1.2.4.1 volatile概述



#### 1.2.4.2

#### 1.2.4.3

#### 1.2.4.4





无锁-》偏向锁-》轻量级锁-》重量级锁

`synchronized`一开始偏向锁，就是没有锁，只是一个指针标识，因为往往加锁的方法其实只有一个线程在执行。然后遇到其他线程时，就升级成轻量级锁。轻量级锁，自旋锁，还是用户运行态，占用CPU资源。自旋超过10次or线程数超过CPU内核1/2数量，升级重量级锁，也就是 需要内核态提供的底层Lock。（底层Lock有自旋写法，也有非自旋使用等待队列->阻塞的写法，这里重量级指的是阻塞的写法。而轻量级锁是自旋的，还在运行态or就绪态。）

如果直接执行`wait`，那偏向锁，执行升级重量级锁。

对应[cpp代码]([https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/interpreterRuntime.cpp])`InterpreterRuntime::monitorenter`

> [openjdk中的同步代码](https://blog.csdn.net/iteye_16780/article/details/81620174)
>
> [volatile底层实现原理](https://www.cnblogs.com/wildwolf0/p/11449506.html)



# 2. Java虚拟机(JVM)





