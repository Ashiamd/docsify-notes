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
> [LockSupport（park/unpark）源码分析](https://www.jianshu.com/p/e3afe8ab8364)

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

    <small>unpark(t1)不叠加，多次unpark(t1)和一次的效果相同，内部都是信号量计数置为1。unpark可以先于park执行，只要使得计数为1,park就无需进入WAITING直接往下执行。</small>

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

​	*学过操作系统，你会知道，操作系统中的进程、CPU等对锁的实现，本质上就是对同一块区域进行数值判断（比如判断同一内存地址的当前值是0还是1，只有读取到值为1的CPU核、操作系统进程等实体才能继续工作。当然读取到值为1的实体会把值替换成0，使得其他实体无法继续工作。）*

​	java的对象锁也不例外。`synchronized`指定用于"充当锁"的对象，其对象头信息中的"Mark Word"信息记录了"锁类型"、"锁拥有者"等信息。这样子，多线程下哪个线程拥有锁，拥有的是什么锁，JVM根据充当锁的对象的头信息即可知晓。

> [Java对象结构与锁实现原理及MarkWord详解](https://blog.csdn.net/scdn_cp/article/details/86491792)

### 锁升级

无锁-》偏向锁-》轻量级锁-》重量级锁

`synchronized`一开始偏向锁，就是没有锁，只是一个指针标识，因为往往加锁的方法其实只有一个线程在执行。然后遇到其他线程时，就升级成轻量级锁。轻量级锁，自旋锁，还是用户运行态，占用CPU资源。自旋超过10次or线程数超过CPU内核1/2数量，升级重量级锁，也就是 需要内核态提供的底层Lock。（底层Lock有自旋写法，也有非自旋使用等待队列->阻塞的写法，这里重量级指的是阻塞的写法。而轻量级锁是自旋的，还在运行态or就绪态。）

如果直接执行`wait`，那偏向锁，执行升级重量级锁。

对应[cpp代码]([https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/interpreterRuntime.cpp])`InterpreterRuntime::monitorenter`

> [openjdk中的同步代码](https://blog.csdn.net/iteye_16780/article/details/81620174)
>
> [volatile底层实现原理](https://www.cnblogs.com/wildwolf0/p/11449506.html)



# 2. Java虚拟机(JVM)





