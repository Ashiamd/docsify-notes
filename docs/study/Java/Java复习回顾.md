> 当时最早是课外提前学的Java，看的《java疯狂讲义》。然后平时有事没事看到优质Java、SQL、Redis等优质技术文章，都会阅读+收藏。但是时间长了，阅读过的东西还是会忘记，俗话好记性不如烂笔头。所以这里再回顾下一些知识点，然后整理下笔记吧。
>
> 这里先更新 ==> 并发、虚拟机，这两个在实际开发解决问题时比较有帮助。（不只是代码，而是底层思想理解上的帮助）
>
> 最近还没正式整理。==>还是老样子会在本地OneNote先粗略记录一遍，之后再整理到MD笔记里。（OneNote，还在时不时存一些文章链接，所以有点慢。因为不想懂个大概，要懂就尽量懂到暂时没法弄懂的地方为止。（当然，不会较真类似CPU从材料进口到加工的全过程之类的问题，哈哈。但是贴近底层语言的，像汇编这种级别的，还是可以了解下的=>虽然过去校内敲过8086汇编，不如实际现在用的CPU架构的汇编指令集复杂。但是没接触过不代表看不懂，配合查相关汇编指令表还是能看懂的。）



# 1. Java并发

## 1.1 基础概念

### 1.1.1 线程

#### 1.1.1.1 线程-创建

常用的几种线程创建方式如下：

1. `MyThread`继承`Thread`，重写`run()`方法，新建对象`new MyThread()`
   + `public Thread() {init(null, null, "Thread-" + nextThreadNum(), 0);}`
2. `MyRunnable`实现`Runnable`接口，重写`run()`方法，新建对象`new Thread(new MyRunnable())`  <= <small>(当然也可以写成lambda表达式)</small>
   + `public Thread(Runnable target) {init(null, target, "Thread-" + nextThreadNum(), 0);}`
3. 使用`Executors.newCachedThreadPool()`静态方法创建线程池`ExecutorService`，由线程池管理线程。
   + `Future<?> submit(Runnable task);`
   + `<T> Future<T> submit(Runnable task, T result);`
   + `<T> Future<T> submit(Callable<T> task);`

#### 1.1.1.2 线程-启动

1. Thread对象调用`start()`
2. 线程池`ExecutorService`提交线程任务<small>(池内有空闲则直接复用，无线程可用且未达到线程池上限容量，则新建并启动)</small>

线程启动，即进入"就绪态"，是否"运行态"还看CPU调度情况。

#### 1.1.1.3 线程-就绪、阻塞

yield、sleep、join

> [java sleep 和 wait线程是处于 阻塞 还是 就绪](https://zhidao.baidu.com/question/1818162835626221268.html)
>
> yield是就绪
>
> 是的，wait 放对象锁 sleep不放。sleep不出让系统资源；wait是进入线程等待池等待，出让系统资源，其他线程可以占用CPU
>
> [Java中interrupt的使用](https://www.cnblogs.com/jenkov/p/juc_interrupt.html)



### 锁升级

无锁-》偏向锁-》轻量级锁-》重量级锁

`synchronized`一开始偏向锁，就是没有锁，只是一个指针标识，因为往往加锁的方法其实只有一个线程在执行。然后遇到其他线程时，就升级成轻量级锁。轻量级锁，自旋锁，还是用户运行态，占用CPU资源。自旋超过10次or线程数超过CPU内核1/2数量，升级重量级锁，也就是 需要内核态提供的底层Lock。（底层Lock有自旋写法，也有非自旋使用等待队列->阻塞的写法，这里重量级指的是阻塞的写法。而轻量级锁是自旋的，还在运行态or就绪态。）

如果直接执行`wait`，那偏向锁，执行升级重量级锁。

对应[cpp代码]([https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/interpreterRuntime.cpp])`InterpreterRuntime::monitorenter`

> [openjdk中的同步代码](https://blog.csdn.net/iteye_16780/article/details/81620174)
>
> [volatile底层实现原理](https://www.cnblogs.com/wildwolf0/p/11449506.html)

# 2. Java虚拟机(JVM)





