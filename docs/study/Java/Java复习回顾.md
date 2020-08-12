> 当时最早是课外提前学的Java，看的《java疯狂讲义》。然后平时有事没事看到优质Java、SQL、Redis等优质技术文章，都会阅读+收藏。但是时间长了，阅读过的东西还是会忘记，俗话好记性不如烂笔头。所以这里再回顾下一些知识点，然后整理下笔记吧。
>
> 这里先更新 ==> 并发、虚拟机，这两个在实际开发解决问题时比较有帮助。
>
> <small>（其他的不是说不重要，只是如果不是平时做优化or底层开发的，感觉死记意义不大。比如HashMap什么时候链表变红黑树，阈值多少，ArrayList初始容量多少等。这些其实我觉得死记意义不大，知道什么场景用什么就好了，不一定非得像背书一样去记这些东西。<=做题也是有些需要挑选合适的集合类等使用，只不过有时候快排等自己实现的速度会更快。）</small>

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
2. 线程池`ExecutorService`提交线程任务<small>(池内有空闲则直接复用，无线程可用且未达到线程池上限容量，则新建并启动</small>

线程启动，即进入"就绪态"，是否"运行态"还看CPU调度情况。

#### 1.1.1.3 线程-就绪、阻塞

yield、sleep、join

> [java sleep 和 wait线程是处于 阻塞 还是 就绪](https://zhidao.baidu.com/question/1818162835626221268.html)
>
> yield是就绪
>
> 是的，wait 放对象锁 sleep不放。sleep不出让系统资源；wait是进入线程等待池等待，出让系统资源，其他线程可以占用CPU



### 锁升级

无锁-》偏向锁-》轻量级锁-》重量级锁

`synchronized`一开始偏向锁，就是没有锁，只是一个指针标识，因为往往加锁的方法其实只有一个线程在执行。然后遇到其他线程时，就升级成轻量级锁。轻量级锁，自旋锁，还是用户运行态，占用CPU资源。自旋超过10次or线程数超过CPU内核1/2数量，升级重量级锁，也就是 需要内核态提供的底层Lock。（底层Lock有自旋写法，也有非自旋使用等待队列->阻塞的写法，这里重量级指的是阻塞的写法。而轻量级锁是自旋的，还在运行态or就绪态。）

如果直接执行`wait`，那偏向锁，执行升级重量级锁。

对应[cpp代码]([https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683/src/share/vm/interpreter/interpreterRuntime.cpp])`InterpreterRuntime::monitorenter`

> [openjdk中的同步代码](https://blog.csdn.net/iteye_16780/article/details/81620174)

# 2. Java虚拟机(JVM)





