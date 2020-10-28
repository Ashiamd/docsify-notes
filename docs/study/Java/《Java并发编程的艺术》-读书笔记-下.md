# 《Java并发编程的艺术》读书笔记-下

> 找到的[书本源码](http://ifeve.com/wp-content/uploads/2015/08/ArtConcurrentBook.zip)下载链接

# 第5章 Java中的锁

​	本章将介绍Java并发包中与锁相关的API和组件，以及这些API和组件的使用方式和实现细节。内容主要围绕两个方面：使用，通过示例演示这些组件的使用方法以及详细介绍与锁相关的API；实现，通过分析源码来剖析实现细节，因为理解实现的细节方能更加得心应手且正确地使用这些组件。希望通过以上两个方面的讲解使开发者对锁的使用和实现两个层面有一定的了解。

## 5.1 Lock接口

​	锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时访问共享资源（但是有些锁可以允许多个线程并发的访问共享资源，比如读写锁）。在Lock接口出现之前，Java程序是靠synchronized关键字实现锁功能的，而**Java SE 5之后，并发包中新增了Lock接口（以及相关实现类）用来实现锁功能，它提供了与synchronized关键字类似的同步功能，只是在使用时需要显式地获取和释放锁**。虽然它缺少了（通过synchronized块或者方法所提供的）隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的可操作性、可中断的获取锁以及超时获取锁等多种synchronized关键字所不具备的同步特性。

​	使用synchronized关键字将会隐式地获取锁，但是它将锁的获取和释放固化了，也就是先获取再释放。当然，这种方式简化了同步的管理，可是扩展性没有显示的锁获取和释放来的好。例如，针对一个场景，手把手进行锁获取和释放，先获得锁A，然后再获取锁B，当锁B获得后，释放锁A同时获取锁C，当锁C获得后，再释放B同时获取锁D，以此类推。这种场景下，synchronized关键字就不那么容易实现了，而使用Lock却容易许多。

​	Lock的使用也很简单，（代码清单5-1 LockUseCase.java）是Lock的使用的方式。

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
} finally {
  lock.unlock();
}
```

​	**在finally块中释放锁，目的是保证在获取到锁之后，最终能够被释放**。

​	**不要将获取锁的过程写在try块中，因为如果在获取锁（自定义锁的实现）时发生了异常，异常抛出的同时，也会导致锁无故释放**。

​	Lock接口提供的`synchronized`关键字所不具备的主要特性如下表所示。

| 特性               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| 尝试非阻塞地获取锁 | 当前线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁 |
| 能被中断地获取锁   | **与synchronized不同，获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常将会被抛出，同时锁将会被释放** |
| 超时获取锁         | 在指定的截止时间之前获取锁，如果截止时间到了仍旧无法获取锁，则返回 |

​	Lock是一个接口，它定义了锁获取和释放的基本操作，Lock的API如下表所示

| 方法名称                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| void lock()                                                  | 获取锁，调用该方法当前线程将会获取锁，当锁获得后，从该方法返回 |
| void lockInterruptibly() throws InterruptedException         | 可中断地获取锁，和lock()方法不同之处在于该方法会响应中断，即在锁释放中可以中断当前线程 |
| boolean trylock()                                            | 尝试非阻塞式的获取锁，调用该方法后立刻返回，如果能够获取则返回true，否则返回false |
| boolean tryLock(Long time,TimeUnit unit)throws InterruptedException | 超时获取锁，当线程在以下3种情况会返回：1. 当前线程在超时时间内获得了锁 2. 当前线程在超时时间内被中断 3. 超时时间结束，返回false |
| void unLock()                                                | 释放锁                                                       |
| Condition newCondition()                                     | 获取等待通知组件，该组件和当前的锁绑定，当前线程只有获得了锁，才能调用该组件wait()方法，而调用后，当前线程释放锁。 |

​	这里先简单介绍一下Lock接口的API，随后的章节会详细介绍**同步器**`AbstractQueuedSynchronizer`以及常用Lock接口的实现ReentrantLock。**Lock接口的实现基本都是通过聚合了一个同步器的子类来完成线程访问控制的**。

## 5.2 队列同步器

​	队列同步器`AbstractQueuedSynchronizer`（以下简称同步器），是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作，并发包的作者（Doug Lea）期望它能够成为实现大部分同步需求的基础。

​	<u>同步器的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态</u>，在抽象方法的实现过程中免不了要对同步状态进行更改，这时就需要使用同步器提供的3个方法（getState()、setState(int newState)和compareAndSetState(int expect,int update)）来进行操作，因为它们能够保证状态的改变是安全的。**子类推荐被定义为自定义同步组件的静态内部类**，同步器自身没有实现任何同步接口，它仅仅是定义了若干同步状态获取和释放的方法来供自定义同步组件使用，同步器既可以支持独占式地获取同步状态，也可以支持共享式地获取同步状态，这样就可以方便实现不同类型的同步组件（ReentrantLock、ReentrantReadWriteLock和CountDownLatch等）。

​	**同步器是实现锁（也可以是任意同步组件）的关键，在锁的实现中聚合同步器，利用同步器实现锁的语义**。可以这样理解二者之间的关系：**锁是面向使用者的，它定义了使用者与锁交互的接口（比如可以允许两个线程并行访问），隐藏了实现细节；同步器面向的是锁的实现者，它简化了锁的实现方式，屏蔽了同步状态管理、线程的排队、等待与唤醒等底层操作。锁和同步器很好地隔离了使用者和实现者所需关注的领域**。

### 5.2.1 队列同步器的接口与示例

​	**同步器的设计是基于模板方法模式**的，也就是说，使用者需要继承同步器并重写指定的方法，随后将同步器组合在自定义同步组件的实现中，并调用同步器提供的模板方法，而这些模板方法将会调用使用者重写的方法。

​	重写同步器指定的方法时，需要使用同步器提供的如下3个方法来访问或修改同步状态。

+ `getState()`：获取当前同步状态。

+ `setState(int newState)`：设置当前同步状态。

+ `compareAndSetState(int expect,int update)`：使用CAS设置当前状态，该方法能够保证状态

  设置的原子性。

同步器可重写的方法与描述如下表所示。

| 方法名称                                    | 描述                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| protected boolean tryAcquire(int arg)       | 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再进行CAS设置同步状态 |
| protected Boolean tryRelease(int arg)       | 独占式释放同步状态，等待获取同步状态的线程将有机会获取同步状态 |
| protected int tryAcquireShared(int arg)     | 共享式获取同步状态，返回大于等于0的值，表示获取成功，反之获取失败 |
| protected boolean tryReleaseShared(int arg) | 共享式释放同步状态                                           |
| ptorotected boolean isHeldExclusively()     | 当前同步器是否在独占模式下被线程占用，一般该方法表示是否被**当前**线程所独占 |

​	实现自定义同步组件时，将会调用同步器提供的模板方法，这些（部分）模板方法与描述如下表所示

| 方法名称                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| public final void acquire(int arg)                           | 独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入**同步队列**等待，<u>该方法将会调用重写的tryAcquire（int arg）方法</u> |
| public final void acquireInterruptibly(int arg)              | 与acquire（int arg）相同，但是该方法响应中断，当前线程未获取到同步状态而进入同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException并返回 |
| public final boolean tryAcquireNanos(int arg, long nanosTimeout) | 在acquireInterruptibly(int arg)基础上增加了超时限制，如果当前线程在超过时间内没有获取到同步状态，那么将返回false，如果获取到了返回true |
| public final void acquireShared(int arg)                     | 共享式的获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式获取的主要区别是在同一时刻可以有多个线程获取到同步状态 |
| public final void acquireSharedInterruptibly(int arg)        | 与acquireShared(int arg)相同，该方法响应中断                 |
| public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout) | 在acquireSharedInterruptibly（int arg）基础上增加了超时限制  |
| public final boolean release(int arg)                        | 独占式的释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒。 |
| public final boolean releaseShared(int arg)                  | 共享式的释放同步状态                                         |
| public final Collection\<Thread\> getQueuedThreads()         | 获取等待在同步队列上的线程集合                               |

​	同步器提供的模板方法基本上分为3类：

+ **独占式**获取与释放同步状态
+ **共享式**获取与释放同步状态
+ 查询同步队列中的等待线程情况。

​	自定义同步组件将使用同步器提供的模板方法来实现自己的同步语义。

​	只有掌握了同步器的工作原理才能更加深入地理解并发包中其他的并发组件，所以下面通过一个独占锁的示例来深入了解一下同步器的工作原理。

​	顾名思义，独占锁就是在同一时刻只能有一个线程获取到锁，而其他获取锁的线程只能处于同步队列中等待，只有获取锁的线程释放了锁，后继的线程才能够获取锁，如（代码清单5-2 Mutex.java）所示。

```java
class Mutex implements Lock {
  // 静态内部类，自定义同步器
  private static class Sync extends AbstractQueuedSynchronizer {
    // 是否处于占用状态
    protected boolean isHeldExclusively() {
      return getState() == 1;
    }
    // 当状态为0的时候获取锁
    public boolean tryAcquire(int acquires) {
      if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
      }
      return false;
    }
    // 释放锁，将状态设置为0
    protected boolean tryRelease(int releases) {
      if (getState() == 0) throw new
        IllegalMonitorStateException();
      setExclusiveOwnerThread(null);
      setState(0);
      return true;
    }
    // 返回一个Condition，每个condition都包含了一个condition队列
    Condition newCondition() { return new ConditionObject(); }
  }
  // 仅需要将操作代理到Sync上即可
  private final Sync sync = new Sync();
  public void lock() { sync.acquire(1); }
  public boolean tryLock() { return sync.tryAcquire(1); }
  public void unlock() { sync.release(1); }
  public Condition newCondition() { return sync.newCondition(); }
  public boolean isLocked() { return sync.isHeldExclusively(); }
  public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
  public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
  }
  public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
  }
}
```

​	上述示例中，独占锁Mutex是一个自定义同步组件，它在同一时刻只允许一个线程占有锁。Mutex中定义了一个静态内部类，该内部类继承了同步器并实现了独占式获取和释放同步状态。在tryAcquire(int acquires)方法中，如果经过CAS设置成功（同步状态设置为1），则代表获取了同步状态，而在tryRelease(int releases)方法中只是将同步状态重置为0。用户使用Mutex时并不会直接和内部同步器的实现打交道，而是调用Mutex提供的方法，在Mutex的实现中，以获取锁的lock()方法为例，只需要在方法实现中调用同步器的模板方法acquire(int args)即可，当前线程调用该方法获取同步状态失败后会被加入到同步队列中等待，这样就大大降低了实现一个可靠自定义同步组件的门槛。

### 5.2.2 队列同步器的实现分析

## 5.3 重入锁

## 5.4 读写锁

### 5.4.1 读写锁的接口与示例

### 5.4.2 读写锁的实现分析

## 5.5 LockSupport工具

## 5.6 Condition接口

### 5.6.1 Condition接口与示例

### 5.6.2 Condition的实现分析

## 5.7 本章小结

# 第6章 Java并发容器和框架

# 第7章 Java中的13个原子操作类

# 第8章 Java中的并发工具类

# 第9章 Java中的线程池

# 第10章 Executor框架

# 第11章 Java并发编程实战

