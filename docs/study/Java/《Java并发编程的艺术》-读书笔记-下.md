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

​	上述示例中，独占锁Mutex是一个自定义同步组件，它在同一时刻只允许一个线程占有锁。Mutex中定义了一个静态内部类，该内部类继承了同步器并实现了独占式获取和释放同步状态。在`tryAcquire(int acquires)`方法中，如果经过CAS设置成功（同步状态设置为1），则代表获取了同步状态，而在`tryRelease(int releases)`方法中只是将同步状态重置为0。用户使用Mutex时并不会直接和内部同步器的实现打交道，而是调用Mutex提供的方法，在Mutex的实现中，以获取锁的lock()方法为例，只需要在方法实现中调用同步器的模板方法acquire(int args)即可，当前线程调用该方法获取同步状态失败后会被加入到同步队列中等待，这样就大大降低了实现一个可靠自定义同步组件的门槛。

### 5.2.2 队列同步器的实现分析

> [AbstractQueuedSynchronizer 功能介绍](https://blog.csdn.net/u010805617/article/details/87276802)

​	接下来将从实现角度分析同步器是如何完成线程同步的，主要包括：**同步队列**、**独占式同步状态获取与释放**、**共享式同步状态获取与释放**以及**超时获取同步状态**等同步器的核心数据结构与模板方法。

#### 1. 同步队列

​	同步器依赖内部的**同步队列（一个FIFO双向队列）**来完成同步状态的管理，**当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成为一个节点（Node）并将其加入同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点中的线程唤醒，使其再次尝试获取同步状态**。

​	同步队列中的节点（Node）用来保存获取同步状态失败的线程引用、等待状态以及前驱和后继节点，节点的属性类型与名称以及描述如下表所示。

<table>
  <tr>
  	<th>属性类型与名称</th>
    <th align="center">描述</th>
  </tr>
  <tr>
    <td>int waitStatus</td>
    <td>
      等待状态。<br/>
    	包括如下状态。<br/>
      1、CANCELLED，值为1，由于在同步队列中等待的线程等待超时或者被中断，需要从同步队列中取消等待，节点进入该状态将不会变化<br/>
      2、SIGNAL，值为-1，后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行<br/>
      3、CONDITION，值为-2，节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()方法后，该节点将会从等待队列中转移到同步队列中，加入到对同步状态的获取中<br/>
      4、PROPAGATE，值为-3，表示下一次共享式同步状态获取将会无条件地被传播下去<br/>
      5、INITIAL，值为0，初始状态
    </td>
  </tr>
  <tr>
    <td>Node prev</td>
    <td>前驱节点，当节点加入同步队列时被设置（尾部添加）</td>
  </tr>
  <tr>
    <td>Node next</td>
    <td>后继节点</td>
  </tr>
  <tr>
    <td>Node nextWaiter</td>
    <td>等待队列中的后继节点。如果当前节点是共享的，那么这个字段将是一个SHARED常量，也就是说节点类型（独占和共享）和等待队列中的后继节点共用同一个字段</td>
  </tr>
  <tr>
    <td>Thread thread</td>
    <td>获取同步状态的线程</td>
  </tr>
</table>

​	节点是构成同步队列（等待队列，在5.6节中将会介绍）的基础，同步器拥有首节点（head）和尾节点（tail），没有成功获取同步状态的线程将会成为节点加入该队列的尾部，同步队列的基本结构如下图所示。

![同步队列的基本结构](https://img-blog.csdnimg.cn/20190214150809925.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4MDU2MTc=,size_16,color_FFFFFF,t_70)

​	在上图中，同步器包含了两个节点类型的引用，一个指向头节点，而另一个指向尾节点。试想一下，当一个线程成功地获取了同步状态（或者锁），其他线程将无法获取到同步状态，转而被构造成为节点并加入到同步队列中，而这个加入队列的过程必须要保证线程安全，因此同步器提供了一个基于CAS的设置尾节点的方法：`compareAndSetTail(Node expect,Nodeupdate)`，它需要传递当前线程“认为”的尾节点和当前节点，只有设置成功后，当前节点才正式与之前的尾节点建立关联。

​	同步器将节点加入到同步队列的过程如下图所示。

![img](https://img-blog.csdn.net/2018082917283132?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMjY3ODE3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

​	**同步队列遵循FIFO，首节点是获取同步状态成功的节点，首节点的线程在释放同步状态时，将会唤醒后继节点，而后继节点将会在获取同步状态成功时将自己设置为首节点**，该过程如下图所示。

![img](https://img-blog.csdn.net/20180829173026740?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMjY3ODE3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

​	在上图中，**设置首节点是通过获取同步状态成功的线程来完成的**，由于只有一个线程能够成功获取到同步状态，因此设置头节点的方法并不需要使用CAS来保证，它只需要将首节点设置成为原首节点的后继节点并断开原首节点的next引用即可。

#### 2. 独占式同步状态获取与释放

​	<u>通过调用同步器的`acquire(int arg)`方法可以获取同步状态，该方法对中断不敏感，也就是由于线程获取同步状态失败后进入同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移出</u>，该方法代码如（代码清单5-3 同步器的acquire方法）所示。

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

​	上述代码主要完成了同步状态获取、节点构造、加入同步队列以及在同步队列中自旋等待的相关工作，其主要逻辑是：首先调用自定义同步器实现的`tryAcquire(int arg)`方法，该方法保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点（独占式Node.EXCLUSIVE，同一时刻只能有一个线程成功获取同步状态）并通过addWaiter(Node node)方法将该节点加入到同步队列的尾部，最后调用`acquireQueued(Node node,int arg)`方法，使得该节点以“死循环”的方式获取同步状态。<u>如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。</u>

​	下面分析一下相关工作。首先是节点的构造以及加入同步队列，如（代码清单5-4 同步器的addWaiter和enq方法）所示。

```java
private Node addWaiter(Node mode) {
  Node node = new Node(Thread.currentThread(), mode);
  // 快速尝试在尾部添加
  Node pred = tail;
  if (pred != null) {
    node.prev = pred;
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  enq(node);
  return node;
} 
private Node enq(final Node node) {
  for (;;) {
    Node t = tail;
    if (t == null) { // Must initialize
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      node.prev = t;
      if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
      }
    }
  }
}
```

​	上述代码通过使用`compareAndSetTail(Node expect,Node update)`方法来确保节点能够被线程安全添加。试想一下：如果使用一个普通的LinkedList来维护节点之间的关系，那么当一个线程获取了同步状态，而其他多个线程由于调用`tryAcquire(int arg)`方法获取同步状态失败而并发地被添加到LinkedList时，LinkedList将难以保证Node的正确添加，最终的结果可能是节点的数量有偏差，而且顺序也是混乱的。

​	在`enq(final Node node)`方法中，同步器通过“死循环”来保证节点的正确添加，在“死循环”中只有通过CAS将节点设置成为尾节点之后，当前线程才能从该方法返回，否则，当前线程不断地尝试设置。可以看出，`enq(final Node node)`方法将并发添加节点的请求通过CAS变得“串行化”了。

​	**节点进入同步队列之后，就进入了一个自旋的过程，每个节点（或者说每个线程）都在自省地观察，当条件满足，获取到了同步状态，就可以从这个自旋过程中退出，否则依旧留在这个自旋过程中（并会阻塞节点的线程）**，如（代码清单5-5 同步器的acquireQueued方法）所示。

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      final Node p = node.predecessor();
      if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
      }
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

​	在`acquireQueued(final Node node,int arg)`方法中，当前线程在“死循环”中尝试获取同步状态，而**只有前驱节点是头节点才能够尝试获取同步状态**，这是为什么？原因有两个，如下。

​	第一，**头节点是成功获取到同步状态的节点**，而头节点的线程释放了同步状态之后，将会唤醒其后继节点，后继节点的线程被唤醒后需要检查自己的前驱节点是否是头节点。

​	第二，**维护同步队列的FIFO原则**。该方法中，节点自旋获取同步状态的行为如下图所示。

![img](https://img-blog.csdn.net/20160816201210376?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

​	在上图中，由于非首节点线程前驱节点出队或者被中断而从等待状态返回，随后检查自己的前驱是否是头节点，如果是则尝试获取同步状态。可以看到节点和节点之间在循环检查的过程中基本不相互通信，而是简单地判断自己的前驱是否为头节点，这样就使得节点的释放规则符合FIFO，并且也便于对过早通知的处理（**过早通知是指前驱节点不是头节点的线程由于中断而被唤醒**）。

​	独占式同步状态获取流程，也就是`acquire(int arg)`方法调用流程，如下图所示。

![独占式同步状态获取流程图](https://img-blog.csdnimg.cn/20190214151630304.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4MDU2MTc=,size_16,color_FFFFFF,t_70)

​	在上图中，**前驱节点为头节点且能够获取同步状态的判断条件和线程进入等待状态是获取同步状态的自旋过程**。当同步状态获取成功之后，当前线程从`acquire(int arg)`方法返回，如果对于锁这种并发组件而言，代表着当前线程获取了锁。

​	<u>当前线程获取同步状态并执行了相应逻辑之后，就需要释放同步状态，使得后续节点能够继续获取同步状态</u>。通过调用同步器的`release(int arg)`方法可以释放同步状态，该方法在释放了同步状态之后，会唤醒其后继节点（进而使后继节点重新尝试获取同步状态）。该方法代码如（代码清单5-6 同步器的release方法）所示。

```java
public final boolean release(int arg) {
  if (tryRelease(arg)) {
    Node h = head;
    if (h != null && h.waitStatus != 0)
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```

​	该方法执行时，会唤醒头节点的后继节点线程，`unparkSuccessor(Node node)`方法使用LockSupport（在后面的章节会专门介绍）来唤醒处于等待状态的线程。

​	分析了独占式同步状态获取和释放过程后，适当做个总结：

+ 在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行**自旋**；
+ 移出队列（或停止自旋）的条件是**前驱节点为头节点**且**成功获取了同步状态**。
+ 在释放同步状态时，同步器调用`tryRelease(int arg)`方法释放同步状态，然后**唤醒头节点的后继节点**。

#### 3. 共享式同步状态获取与释放

​	**共享式获取与独占式获取最主要的区别在于同一时刻能否有多个线程同时获取到同步状态**。以文件的读写为例，如果一个程序在对文件进行读操作，那么这一时刻对于该文件的写操作均被阻塞，而读操作能够同时进行。写操作要求对资源的独占式访问，而读操作可以是共享式访问，两种不同的访问模式在同一时刻对文件或资源的访问情况，如下图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190214153051344.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4MDU2MTc=,size_16,color_FFFFFF,t_70)

​	在上图中，左半部分，共享式访问资源时，其他共享式的访问均被允许，而独占式访问被阻塞，右半部分是独占式访问资源时，同一时刻其他访问均被阻塞。

​	通过调用同步器的`acquireShared(int arg)`方法可以**共享式**地获取**同步状态**，该方法代码如（代码清单5-7 同步器的acquireShared和doAcquireShared方法）所示。

```java
public final void acquireShared(int arg) {
  if (tryAcquireShared(arg) < 0)
    doAcquireShared(arg);
}
private void doAcquireShared(int arg) {
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      final Node p = node.predecessor();
      if (p == head) {
        int r = tryAcquireShared(arg);
        if (r >= 0) {
          setHeadAndPropagate(node, r);
          p.next = null;
          if (interrupted)
            selfInterrupt();
          failed = false;
          return;
        }
      }
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

​	在`acquireShared(int arg)`方法中，同步器调用`tryAcquireShared(int arg)`方法尝试获取同步状态，`tryAcquireShared(int arg)`方法返回值为int类型，当返回值大于等于0时，表示能够获取到同步状态。因此，在共享式获取的自旋过程中，成功获取到同步状态并退出自旋的条件就是`tryAcquireShared(int arg)`方法返回值大于等于0。可以看到，在`doAcquireShared(int arg)`方法的自旋过程中，如果当前节点的前驱为头节点时，尝试获取同步状态，如果返回值大于等于0，表示该次获取同步状态成功并从自旋过程中退出。

​	与独占式一样，共享式获取也需要释放同步状态，通过调用`releaseShared(int arg)`方法可以释放同步状态，该方法代码如（代码清单5-8 同步器的releaseShared方法）所示。

```java
public final boolean releaseShared(int arg) {
  if (tryReleaseShared(arg)) {
    doReleaseShared();
    return true;
  }
  return false;
}
```

​	**该方法在释放同步状态之后，将会唤醒后续处于等待状态的节点**。对于能够支持多个线程同时访问的并发组件（比如Semaphore），它和独占式主要区别在于`tryReleaseShared(int arg)`方法必须确保同步状态（或者资源数）线程安全释放，一般是通过**循环和CAS**来保证的，因为释放同步状态的操作会同时来自多个线程。

#### 4. 独占式超时获取同步状态

​	通过调用同步器的`doAcquireNanos(int arg,long nanosTimeout)`方法可以超时获取同步状态，即在指定的时间段内获取同步状态，如果获取到同步状态则返回true，否则，返回false。该方法提供了传统Java同步操作（比如synchronized关键字）所不具备的特性。

​	在分析该方法的实现前，先介绍一下**响应中断的同步状态获取过程**。**在Java 5之前，当一个线程获取不到锁而被阻塞在synchronized之外时，对该线程进行中断操作，此时该线程的中断标志位会被修改，但线程依旧会阻塞在synchronized上，等待着获取锁。在Java 5中，同步器提供了`acquireInterruptibly(int arg)`方法，这个方法在等待获取同步状态时，如果当前线程被中断，会立刻返回，并抛出InterruptedException。**

​	超时获取同步状态过程可以被视作响应中断获取同步状态过程的“增强版”，`doAcquireNanos(int arg,long nanosTimeout)`方法**在支持响应中断的基础上，增加了超时获取的特性**。针对超时获取，主要需要计算出需要睡眠的时间间隔nanosTimeout，为了防止过早通知，nanosTimeout计算公式为：nanosTimeout-=now-lastTime，其中now为当前唤醒时间，lastTime为上次唤醒时间，如果nanosTimeout大于0则表示超时时间未到，需要继续睡眠nanosTimeout纳秒，反之，表示已经超时，该方法代码如（代码清单5-9 同步器的doAcquireNanos方法）所示。

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
  throws InterruptedException {
  long lastTime = System.nanoTime();
  final Node node = addWaiter(Node.EXCLUSIVE);
  boolean failed = true;
  try {
    for (;;) {
      final Node p = node.predecessor();
      if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return true;
      }
      if (nanosTimeout <= 0)
        return false;
      if (shouldParkAfterFailedAcquire(p, node)
          && nanosTimeout > spinForTimeoutThreshold)
        LockSupport.parkNanos(this, nanosTimeout);
      long now = System.nanoTime();
      //计算时间，当前时间now减去睡眠之前的时间lastTime得到已经睡眠
      //的时间delta，然后被原有超时时间nanosTimeout减去，得到了
      //还应该睡眠的时间
      nanosTimeout -= now - lastTime;
      lastTime = now;
      if (Thread.interrupted())
        throw new InterruptedException();
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

​	该方法在自旋过程中，当节点的前驱节点为头节点时尝试获取同步状态，如果获取成功则从该方法返回，这个过程和独占式同步获取的过程类似，但是在同步状态获取失败的处理上有所不同。如果当前线程获取同步状态失败，则判断是否超时（nanosTimeout小于等于0表示已经超时），如果没有超时，重新计算超时间隔nanosTimeout，然后使当前线程等待nanosTimeout纳秒（当已到设置的超时时间，该线程会从`LockSupport.parkNanos(Objectblocker,long nanos)`方法返回）。

​	<u>如果nanosTimeout小于等于spinForTimeoutThreshold（1000纳秒）时，将不会使该线程进行超时等待，而是进入快速的自旋过程。原因在于，非常短的超时等待无法做到十分精确，如果这时再进行超时等待，相反会让nanosTimeout的超时从整体上表现得反而不精确。因此，在超时非常短的场景下，同步器会进入无条件的快速自旋</u>。

​	独占式超时获取同步态的流程如下图所示

![独占式超时获取同步状态的流程](https://img-blog.csdnimg.cn/20190314162243495.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4MDU2MTc=,size_16,color_FFFFFF,t_70)

​	从上图中可以看出，独占式超时获取同步状态`doAcquireNanos(int arg,long nanosTimeout)`和独占式获取同步状态`acquire(int args)`在流程上非常相似，其主要区别在于未获取到同步状态时的处理逻辑。`acquire(int args)`在未获取到同步状态时，将会使当前线程一直处于等待状态，而`doAcquireNanos(int arg,long nanosTimeout)`会使当前线程等待nanosTimeout纳秒，如果当前线程在nanosTimeout纳秒内没有获取到同步状态，将会从等待逻辑中自动返回。

#### 5. 自定义同步组件——TwinsLock

​	在前面的章节中，对同步器AbstractQueuedSynchronizer进行了实现层面的分析，本节通过编写一个自定义同步组件来加深对同步器的理解。

​	设计一个同步工具：该工具在同一时刻，只允许至多两个线程同时访问，超过两个线程的访问将被阻塞，我们将这个同步工具命名为TwinsLock。

​	首先，**确定访问模式**。TwinsLock能够在同一时刻支持多个线程的访问，这显然是共享式访问，因此，需要使用同步器提供的`acquireShared(int args)`方法等和Shared相关的方法，这就要求TwinsLock必须重写`tryAcquireShared(int args)`方法和`tryReleaseShared(int args)`方法，这样才能保证同步器的共享式同步状态的获取与释放方法得以执行。

​	其次，**定义资源数**。TwinsLock在同一时刻允许至多两个线程的同时访问，表明同步资源数为2，这样可以设置初始状态status为2，当一个线程进行获取，status减1，该线程释放，则status加1，状态的合法范围为0、1和2，其中0表示当前已经有两个线程获取了同步资源，此时再有其他线程对同步状态进行获取，该线程只能被阻塞。在同步状态变更时，需要使用`compareAndSet(int expect,int update)`方法做原子性保障。

​	最后，**组合自定义同步器**。前面的章节提到，**自定义同步组件通过组合自定义同步器来完成同步功能，一般情况下自定义同步器会被定义为自定义同步组件的内部类**。

​	TwinsLock（部分）代码如（代码清单5-10 TwinsLock.java）所示。

```java
public class TwinsLock implements Lock {
  private final Sync sync = new Sync(2);

  private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -7889272986162341211L;

    Sync(int count) {
      if (count <= 0) {
        throw new IllegalArgumentException("count must large than zero.");
      }
      setState(count);
    }

    public int tryAcquireShared(int reduceCount) {
      for (;;) {
        int current = getState();
        int newCount = current - reduceCount;
        if (newCount < 0 || compareAndSetState(current, newCount)) {
          return newCount;
        }
      }
    }

    public boolean tryReleaseShared(int returnCount) {
      for (;;) {
        int current = getState();
        int newCount = current + returnCount;
        if (compareAndSetState(current, newCount)) {
          return true;
        }
      }
    }

    final ConditionObject newCondition() {
      return new ConditionObject();
    }
  }

  public void lock() {
    sync.acquireShared(1);
  }

  public void unlock() {
    sync.releaseShared(1);
  }

  public void lockInterruptibly() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
  }

  public boolean tryLock() {
    return sync.tryAcquireShared(1) >= 0;
  }

  public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(time));
  }

  @Override
  public Condition newCondition() {
    return sync.newCondition();
  }
}
```

​	在上述示例中，TwinsLock实现了**Lock接口，提供了面向使用者的接口，使用者调用lock()方法获取锁，随后调用unlock()方法释放锁**，而同一时刻只能有两个线程同时获取到锁。TwinsLock同时包含了一个**自定义同步器Sync，而该同步器面向线程访问和同步状态控制**。以共享式获取同步状态为例：同步器会先计算出获取后的同步状态，然后通过CAS确保状态的正确设置，当`tryAcquireShared(int reduceCount)`方法返回值大于等于0时，当前线程才获取同步状态，对于上层的TwinsLock而言，则表示当前线程获得了锁。

​	**同步器作为一个桥梁，连接线程访问以及同步状态控制等底层技术与不同并发组件（比如Lock、CountDownLatch等）的接口语义**。

​	下面编写一个测试来验证TwinsLock是否能按照预期工作。在测试用例中，定义了工作者线程Worker，该线程在执行过程中获取锁，当获取锁之后使当前线程睡眠1秒（并不释放锁），随后打印当前线程名称，最后再次睡眠1秒并释放锁，测试用例如（代码清单5-11 TwinsLockTest.java）所示。

```java
public class TwinsLockTest {
  @Test
  public void test() {
    final Lock lock = new TwinsLock();
    class Worker extends Thread {
      public void run() {
        while (true) {
          lock.lock();
          try {
            SleepUtils.second(1);
            System.out.println(Thread.currentThread().getName());
            SleepUtils.second(1);
          } finally {
            lock.unlock();
          }
        }
      }
    }
    // 启动10个线程
    for (int i = 0; i < 10; i++) {
      Worker w = new Worker();
      w.setDaemon(true);
      w.start();
    }
    // 每隔1秒换行
    for (int i = 0; i < 10; i++) {
      SleepUtils.second(1);
      System.out.println();
    }
  }
}
```

​	运行该测试用例，可以看到线程名称成对输出，也就是在同一时刻只有两个线程能够获取到锁，这表明TwinsLock可以按照预期正确工作。

## 5.3 重入锁

​	**重入锁ReentrantLock，顾名思义，就是支持重进入的锁，它表示该锁能够支持一个线程对资源的重复加锁**。除此之外，该锁的还支持获取锁时的**公平**和**非公平**性选择。

​	回忆在同步器一节中的示例（Mutex），同时考虑如下场景：当一个线程调用Mutex的lock()方法获取锁之后，如果再次调用lock()方法，则该线程将会被自己所阻塞，原因是Mutex在实现tryAcquire(int acquires)方法时没有考虑占有锁的线程再次获取锁的场景，而在调用`tryAcquire(int acquires)`方法时返回了false，导致该线程被阻塞。简单地说，**Mutex是一个不支持重进入的锁**。**而synchronized关键字隐式的支持重进入**，比如一个synchronized修饰的递归方法，在方法执行时，执行线程在获取了锁之后仍能连续多次地获得该锁，而不像Mutex由于获取了锁，而在下一次获取锁时出现阻塞自己的情况。

​	**ReentrantLock虽然没能像synchronized关键字一样支持隐式的重进入，但是在调用lock()方法时，已经获取到锁的线程，能够再次调用lock()方法获取锁而不被阻塞。**

​	**这里提到一个锁获取的公平性问题，如果在绝对时间上，先对锁进行获取的请求一定先被满足，那么这个锁是公平的，反之，是不公平的**。**公平的获取锁，也就是等待时间最长的线程最优先获取锁，也可以说锁获取是顺序的**。ReentrantLock提供了一个构造函数，能够控制锁是否是公平的。

​	事实上，**公平的锁机制往往没有非公平的效率高**，但是，并不是任何场景都是以TPS作为唯一的指标，<u>公平锁能够减少“饥饿”发生的概率，等待越久的请求越是能够得到优先满足</u>。

​	下面将着重分析ReentrantLock是如何实现重进入和公平性获取锁的特性，并通过测试来验证公平性获取锁对性能的影响。

### 1. 实现重进入

​	**重进入是指任意线程在获取到锁之后能够再次获取该锁而不会被锁所阻塞**，该特性的实现需要解决以下两个问题。

1. **线程再次获取锁**。锁需要去识别获取锁的线程是否为当前占据锁的线程，如果是，则再次成功获取。

2. **锁的最终释放**。线程重复n次获取了锁，随后在第n次释放该锁后，其他线程能够获取到该锁。锁的最终释放要求锁对于获取进行计数自增，计数表示当前锁被重复获取的次数，而锁被释放时，计数自减，当计数等于0时表示锁已经成功释放。

​	ReentrantLock是通过组合自定义同步器来实现锁的获取与释放，以非公平性（默认的）实现为例，获取同步状态的代码如（代码清单5-12 ReentrantLock的nonfairTryAcquire方法）所示。

```java
final boolean nonfairTryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    if (compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  } else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

​	该方法增加了再次获取同步状态的处理逻辑：<u>通过判断当前线程是否为获取锁的线程来决定获取操作是否成功，如果是获取锁的线程再次请求，则将同步状态值进行增加并返回true，表示获取同步状态成功</u>。

​	成功获取锁的线程再次获取锁，只是增加了同步状态值，这也就要求ReentrantLock在释放同步状态时减少同步状态值，该方法的代码如（代码清单5-13 ReentrantLock的tryRelease方法）所示。

```java
protected final boolean tryRelease(int releases) {
  int c = getState() - releases;
  if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
  boolean free = false;
  if (c == 0) {
    free = true;
    setExclusiveOwnerThread(null);
  }
  setState(c);
  return free;
}
```

​	<u>如果该锁被获取了n次，那么前(n-1)次tryRelease(int releases)方法必须返回false，而只有同步状态完全释放了，才能返回true。可以看到，该方法将同步状态是否为0作为最终释放的条件，当同步状态为0时，将占有线程设置为null，并返回true，表示释放成功</u>。

### 2. 公平与非公平获取锁的区别

> [公平锁和非公平锁及读写锁](https://www.lmlphp.com/user/56/article/item/1049/)
>
> [Java并发笔记 （10）---- ReentrantLock](https://blog.csdn.net/weixin_44078008/article/details/106064240)

​	**公平性与否是针对获取锁而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是FIFO**。

​	回顾上一小节中介绍的`nonfairTryAcquire(int acquires)`方法，对于非公平锁，只要CAS设置同步状态成功，则表示当前线程获取了锁，而公平锁则不同，如（代码清单5-14 ReentrantLock的tryAcquire方法）所示。

```java
protected final boolean tryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  } else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

​	该方法与`nonfairTryAcquire(int acquires)`比较，唯一不同的位置为判断条件多了`hasQueuedPredecessors()`方法，即加入了同步队列中当前节点是否有前驱节点的判断，如果该方法返回true，则表示有线程比当前线程更早地请求获取锁，因此需要等待前驱线程获取并释放锁之后才能继续获取锁。

​	下面编写一个测试来观察公平和非公平锁在获取锁时的区别，在测试用例中定义了内部类ReentrantLock2，该类主要公开了`getQueuedThreads()`方法，该方法返回正在等待获取锁的线程列表，由于列表是逆序输出，为了方便观察结果，将其进行反转，测试用例（部分）如（代码清单5-15 FairAndUnfairTest.java）所示。

```java
public class FairAndUnfairTest {
  private static Lock fairLock = new ReentrantLock2(true);
  private static Lock unfairLock = new ReentrantLock2(false);
  @Test
  public void fair() {
    testLock(fairLock);
  }
  @Test
  public void unfair() {
    testLock(unfairLock);
  }
  private void testLock(Lock lock) {
    // 启动5个Job（略）
  }
  private static class Job extends Thread {
    private Lock lock;
    public Job(Lock lock) {
      this.lock = lock;
    }
    public void run() {
      // 连续2次打印当前的Thread和等待队列中的Thread（略）
    }
  }
  private static class ReentrantLock2 extends ReentrantLock {
    public ReentrantLock2(boolean fair) {
      super(fair);
    }
    public Collection<Thread> getQueuedThreads() {
      List<Thread> arrayList = new ArrayList<Thread>(super.
                                                     getQueuedThreads());
      Collections.reverse(arrayList);
      return arrayList;
    }
  }
}
```

分别运行fair()和unfair()两个测试方法，输出结果如下表所示。

![公平锁和非公平锁及读写锁-LMLPHP](https://c1.lmlphp.com/user/master/2018/10/03/71a7aeb2fd7d81d7611d0f2121c71ab7.jpg)

​	观察上表所示的结果（其中每个数字代表一个线程），公平性锁每次都是从同步队列中的第一个节点获取到锁，而非公平性锁出现了一个线程连续获取锁的情况。

​	为什么会出现线程连续获取锁的情况呢？回顾`nonfairTryAcquire(int acquires)`方法，**当一个线程请求锁时，只要获取了同步状态即成功获取锁。在这个前提下，刚释放锁的线程再次获取同步状态的几率会非常大，使得其他线程只能在同步队列中等待。**

​	非公平性锁可能使线程“饥饿”，为什么它又被设定成默认的实现呢？再次观察上表的结果，如果把每次不同线程获取到锁定义为1次切换，公平性锁在测试中进行了10次切换，而非公平性锁只有5次切换，这说明**非公平性锁的开销更小**。下面运行测试用例（测试环境：ubuntu server 14.04 i5-34708GB，测试场景：10个线程，每个线程获取100000次锁），通过vmstat统计测试运行时系统线程上下文切换的次数，运行结果如下表所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200511222409543.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDA3ODAwOA==,size_16,color_FFFFFF,t_70#pic_center)

​	在测试中公平性锁与非公平性锁相比，总耗时是其94.3倍，总切换次数是其133倍。可以看出，**公平性锁保证了锁的获取按照FIFO原则，而代价是进行大量的线程切换。非公平性锁虽然可能造成线程“饥饿”，但极少的线程切换，保证了其更大的吞吐量**。

## 5.4 读写锁

​	之前提到锁（如Mutex和ReentrantLock）基本都是排他锁，这些锁在同一时刻只允许一个线程进行访问，而**读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞**。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

​	除了保证写操作对读操作的可见性以及并发性的提升之外，读写锁能够简化读写交互场景的编程方式。假设在程序中定义一个共享的用作缓存数据结构，它大部分时间提供读服务（例如查询和搜索），而写操作占有的时间很少，但是写操作完成之后的更新需要对后续的读服务可见。

​	**在没有读写锁支持的（Java 5之前）时候，如果需要完成上述工作就要使用Java的等待通知机制，就是当写操作开始时，所有晚于写操作的读操作均会进入等待状态，只有写操作完成并进行通知之后，所有等待的读操作才能继续执行（写操作之间依靠synchronized关键进行同步），这样做的目的是使读操作能读取到正确的数据，不会出现脏读。改用读写锁实现上述功能，只需要在读操作时获取读锁，写操作时获取写锁即可。当写锁被获取到时，后续（非当前写操作线程）的读写操作都会被阻塞，写锁释放之后，所有操作继续执行，编程方式相对于使用等待通知机制的实现方式而言，变得简单明了**。

​	一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。**在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量**。Java并发包提供读写锁的实现是ReentrantReadWriteLock，它提供的特性如下表所示。

| 特性       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 公平性选择 | 支持非公平（默认）和公平的锁获取方式，吞吐量还是非公平优于公平 |
| 重进入     | **该锁支持重进入，以读写线程为例：读线程在获取了读锁之后，能够再次获取读锁。而写线程在获取了写锁之后能够再次获取写锁，同时也可以获取读锁** |
| **锁降级** | **遵循获取写锁、获取读锁再释放写锁的次序，写锁能够降级成为读锁** |

### 5.4.1 读写锁的接口与示例

​	ReadWriteLock仅定义了获取读锁和写锁的两个方法，即readLock()方法和writeLock()方法，而其实现——ReentrantReadWriteLock，除了接口方法之外，还提供了一些便于外界监控其内部工作状态的方法，这些方法以及描述如表下表所示。

| 方法名称                | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| int getReadLockCount()  | 返回当前读锁被获取的次数。该次数**不等于获取读锁的线程数**，例如，仅一个线程，它连续获取（重进入）了n次读锁，那么占据读锁的线程数是1，但该方法返回n |
| int getReadHoldCount()  | 返回当前线程获取读锁的次数。该方法在**Java6中加入到ReentranReadWriteLock中，使用ThreadLocal保存当前线程获取的次数，这也使得Java 6的实现变得更加复杂** |
| boolean isWrireLocked() | 判断写锁是否被获取                                           |
| int getWriteHoldCount() | 返回当前写锁被获取的次数                                     |

​	接下来，通过一个缓存示例说明读写锁的使用方式，示例代码如（代码清单5-16 Cache.java）所示。

```java
public class Cache {
  static Map<String, Object> map = new HashMap<String, Object>();
  static ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
  static Lock r = rwl.readLock();
  static Lock w = rwl.writeLock();
  // 获取一个key对应的value
  public static final Object get(String key) {
    r.lock();
    try {
      return map.get(key);
    } finally {
      r.unlock();
    }
  }
  // 设置key对应的value，并返回旧的value
  public static final Object put(String key, Object value) {
    w.lock();
    try {
      return map.put(key, value);
    } finally {
      w.unlock();
    }
  }
  // 清空所有的内容
  public static final void clear() {
    w.lock();
    try {
      map.clear();
    } finally {
      w.unlock();
    }
  }
}
```

​	上述示例中，Cache组合一个**非线程安全**的HashMap作为缓存的实现，同时使用读写锁的读锁和写锁来保证Cache是线程安全的。在读操作get(String key)方法中，需要获取读锁，这使得并发访问该方法时不会被阻塞。写操作put(String key,Object value)方法和clear()方法，<u>在更新HashMap时必须提前获取写锁，当获取写锁后，其他线程对于读锁和写锁的获取均被阻塞，而只有写锁被释放之后，其他读写操作才能继续</u>。Cache使用读写锁提升读操作的并发性，也保证每次写操作对所有的读写操作的可见性，同时简化了编程方式。

### 5.4.2 读写锁的实现分析

> [AbstractQueuedSynchronizer(AQS)深入分析](https://blog.csdn.net/wjs_1024/article/details/107172153)

​	接下来分析ReentrantReadWriteLock的实现，主要包括：**读写状态的设计**、**写锁的获取与释放**、**读锁的获取与释放**以及**锁降级**（以下没有特别说明读写锁均可认为是ReentrantReadWriteLock）。

#### 1. 读写状态的设计

​	读写锁同样依赖自定义同步器来实现同步功能，而**读写状态就是其同步器的同步状态**。回想ReentrantLock中自定义同步器的实现，同步状态表示锁被一个线程重复获取的次数，而读写锁的自定义同步器需要在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态，使得该状态的设计成为读写锁实现的关键。

​	**如果在一个整型变量上维护多种状态，就一定需要“按位切割使用”这个变量**，读写锁将变量切分成了两个部分，高16位表示读，低16位表示写，划分方式如下图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200707010009147.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dqc18xMDI0,size_16,color_FFFFFF,t_70#pic_center)

​	当前同步状态表示一个线程已经获取了写锁，且重进入了两次，同时也连续获取了两次读锁。读写锁是如何迅速确定读和写各自的状态呢？答案是通过位运算。假设当前同步状态值为S，写状态等于S&0x0000FFFF（将高16位全部抹去），读状态等于S>>>16（无符号补0右移16位）。当写状态增加1时，等于S+1，当读状态增加1时，等于S+(1<<16)，也就是S+0x00010000。

​	根据状态的划分能得出一个推论：**S不等于0时，当写状态（S&0x0000FFFF）等于0时，则读状态（S>>>16）大于0，即读锁已被获取。**

#### 2. 写锁的获取与释放

​	**写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（读状态不为0）或者该线程不是已经获取写锁的线程，则当前线程进入等待状态**，获取写锁的代码如（代码清单5-17 ReentrantReadWriteLock的tryAcquire方法）所示。

```java
protected final boolean tryAcquire(int acquires) {
  Thread current = Thread.currentThread();
  int c = getState();
  int w = exclusiveCount(c);
  if (c != 0) {
    // 存在读锁或者当前获取线程不是已经获取写锁的线程
    if (w == 0 || current != getExclusiveOwnerThread())
      return false;
    if (w + exclusiveCount(acquires) > MAX_COUNT)
      throw new Error("Maximum lock count exceeded");
    setState(c + acquires);
    return true;
  }
  if (writerShouldBlock() || !compareAndSetState(c, c + acquires)) {
    return false;
  }
  setExclusiveOwnerThread(current);
  return true;
}
```

​	该方法除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的判断。<u>如果存在读锁，则写锁不能被获取，原因在于：读写锁要确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞</u>。

​	写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对后续读写线程可见。

#### 3. 读锁的获取与释放

​	**读锁是一个支持重进入的共享锁，它能够被多个线程同时获取，在没有其他写线程访问（或者写状态为0）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态**。如果当前线程已经获取了读锁，则增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。获取读锁的实现从Java 5到Java 6变得复杂许多，主要原因是新增了一些功能，例如getReadHoldCount()方法，作用是返回当前线程获取读锁的次数。读状态是所有线程获取读锁次数的总和，而每个线程各自获取读锁的次数只能选择保存在ThreadLocal中，由线程自身维护，这使获取读锁的实现变得复杂。因此，这里将获取读锁的代码做了删减，保留必要的部分，如（代码清单5-18 ReentrantReadWriteLock的tryAcquireShared方法）所示。

```java
protected final int tryAcquireShared(int unused) {
  for (;;) {
    int c = getState();
    int nextc = c + (1 << 16);
    if (nextc < c)
      throw new Error("Maximum lock count exceeded");
    if (exclusiveCount(c) != 0 && owner != Thread.currentThread())
      return -1;
    if (compareAndSetState(c, nextc))
      return 1;
  }
}
```

​	在`tryAcquireShared(int unused)`方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠CAS保证）增加读状态，成功获取读锁。

​	读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的值是（1<<16）。

#### 4. 锁降级

​	**锁降级指的是写锁降级成为读锁**。如果当前线程拥有写锁，然后将其释放，最后再获取读锁，这种分段完成的过程不能称之为锁降级。**锁降级是指把持住（当前拥有的）写锁，再获取到读锁，随后释放（先前拥有的）写锁的过程。**

​	接下来看一个锁降级的示例。因为数据不常变化，所以多个线程可以并发地进行数据处理，当数据变更后，如果当前线程感知到数据变化，则进行数据的准备工作，同时其他处理线程被阻塞，直到当前线程完成数据的准备工作，如（代码清单5-19 processData方法）所示。

```java
public void processData() {
  readLock.lock();
  if (!update) {
    // 必须先释放读锁
    readLock.unlock();
    // 锁降级从写锁获取到开始
    writeLock.lock();
    try {
      if (!update) {
        // 准备数据的流程（略）
        update = true;
      }
      readLock.lock();
    } finally {
      writeLock.unlock();
    }
    // 锁降级完成，写锁降级为读锁
  }
  try {
    // 使用数据的流程（略）
  } finally {
    readLock.unlock();
  }
}
```

​	上述示例中，当数据发生变更后，update变量（布尔类型且volatile修饰）被设置为false，此时所有访问processData()方法的线程都能够感知到变化，但**只有一个线程能够获取到写锁，其他线程会被阻塞在读锁和写锁的lock()方法上**。当前线程获取写锁完成数据准备之后，再获取读锁，随后释放写锁，完成锁降级。

​	**锁降级中读锁的获取是否必要呢？答案是必要的。主要是为了保证数据的可见性，如果当前线程不获取读锁而是直接释放写锁，假设此刻另一个线程（记作线程T）获取了写锁并修改了数据，那么当前线程无法感知线程T的数据更新。如果当前线程获取读锁，即遵循锁降级的步骤，则线程T将会被阻塞，直到当前线程使用数据并释放读锁之后，线程T才能获取写锁进行数据更新。**

​	**RentrantReadWriteLock不支持<u>锁升级（把持读锁、获取写锁，最后释放读锁的过程）</u>。<u>目的也是保证数据可见性</u>，如果读锁已被多个线程获取，其中任意线程成功获取了写锁并更新了数据，则其更新对其他获取到读锁的线程是不可见的。**

## 5.5 LockSupport工具

> [LockSupport工具](https://blog.csdn.net/fristjcjdncg/article/details/107851377)

​	回顾5.2节，**当需要阻塞或唤醒一个线程的时候，都会使用LockSupport工具类来完成相应工作**。LockSupport定义了一组的公共静态方法，这些方法提供了最基本的线程阻塞和唤醒功能，而LockSupport也成为构建同步组件的基础工具。

​	LockSupport定义了一组**以park开头的方法用来阻塞当前线程，以及unpark(Thread thread)方法来唤醒一个被阻塞的线程**。Park有停车的意思，假设线程为车辆，那么park方法代表着停车，而unpark方法则是指车辆启动离开，这些方法以及描述如下表所示。

| 方法名称                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| void park()                   | 阻塞当前线程，如果调用unpark(Thread thread)方法或者当前线程被中断，才能从park()方法返回 |
| void parkNanos(long nanos)    | 阻塞当前线程，最长不超过nanos纳秒，返回条件在park()的基础上增加了超时返回 |
| void parkUntil(long deadline) | 阻塞当前线程，直到deadline时间（从1970年开始到deadline时间的毫秒数） |
| void unpack(Thread thread)    | 唤醒处于阻塞状态的线程thread                                 |

​	**在Java 6中，LockSupport增加了`park(Object blocker)`、`parkNanos(Object blocker,long nanos)`和`parkUntil(Object blocker,long deadline)`3个方法，用于实现阻塞当前线程的功能，其中参数blocker是用来标识当前线程在等待的对象（以下称为阻塞对象），该对象主要用于问题排查和系统监控**。

​	下面的示例中，将对比`parkNanos(long nanos)`方法和`parkNanos(Object blocker,long nanos)`方法来展示阻塞对象blocker的用处，代码片段和线程dump（部分）如表5-11所示。

​	从下表的线程dump结果可以看出，代码片段的内容都是阻塞当前线程10秒，但从线程dump结果可以看出，<u>有阻塞对象的parkNanos方法能够传递给开发人员更多的现场信息。这是由于在Java 5之前，当线程阻塞（使用synchronized关键字）在一个对象上时，通过线程dump能够查看到该线程的阻塞对象，方便问题定位，而Java 5推出的Lock等并发工具时却遗漏了这一点，致使在线程dump时无法提供阻塞对象的信息。因此，在Java 6中，LockSupport新增了上述3个含有阻塞对象的park方法，用以替代原有的park方法</u>。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200806223931366.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZyaXN0amNqZG5jZw==,size_16,color_FFFFFF,t_70)

## 5.6 Condition接口

​	任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括`wait()`、`wait(long timeout)`、`notify()`以及`notifyAll()`方法，这些方法与synchronized同步关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。

​	通过对比Object的监视器方法和Condition接口，可以更详细地了解Condition的特性，对比项与结果如下表所示。

| 对比项                                               | Object Monitor Methods      | Condition                                                    |
| ---------------------------------------------------- | --------------------------- | ------------------------------------------------------------ |
| 前置条件                                             | 获取对象的锁                | 调用Lock.lock()获取锁，调用Lock.newCondition()获取Condition对象 |
| 调用方式                                             | 直接调用，如：object.wait() | 直接调用，如：condition.await()                              |
| 等待队列个数                                         | 一个                        | 多个                                                         |
| 当前线程释放锁并进入等待状态                         | 支持                        | 支持                                                         |
| 当前线程释放锁并进去等待状态，在等待状态中不响应中断 | 不支持                      | 支持                                                         |
| 当前线程释放锁并进入超时等待状态                     | 支持                        | 支持                                                         |
| 当前线程释放锁并进入等待状态到将来的某个时间         | 不支持                      | 支持                                                         |
| 唤醒等待队列中的一个线程                             | 支持                        | 支持                                                         |
| 唤醒等待队列中的全部线程                             | 支持                        | 支持                                                         |

### 5.6.1 Condition接口与示例

​	Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，需要提前获取到Condition对象关联的锁。<u>Condition对象是由Lock对象（调用Lock对象的newCondition()方法）创建出来的</u>，换句话说，**Condition是依赖Lock对象的**。

​	Condition的使用方式比较简单，需要注意在调用方法前获取锁，使用方式如（代码清单5-20 ConditionUseCase.java）所示。

```java
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
public void conditionWait() throws InterruptedException {
  lock.lock();
  try {
    condition.await();
  } finally {
    lock.unlock();
  }
} public void conditionSignal() throws InterruptedException {
  lock.lock();
  try {
    condition.signal();
  } finally {
    lock.unlock();
  }
}
```

​	如示例所示，**一般都会将Condition对象作为成员变量。当调用await()方法后，当前线程会释放锁并在此等待，而其他线程调用Condition对象的signal()方法，通知当前线程后，当前线程才从await()方法返回，并且在返回前已经获取了锁。**

​	Condition定义的（部分）方法以及描述如下表所示。

| 方法名称                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| void await() throws InterruptedException                     | 当前线程进入等待状态直到**被通知(signal)或中断**，当前线程将进入运行状态且从await()方法返回的情况，包括：其他线程调用该Condition的signal()或signAll()方法，而当前线程被选中唤醒（1.其他线程（调用interrupt()方法）中断当前线程 2.如果当前线程从await()方法返回，那么表明该线程已经获取了Condition对象所对应的锁） |
| void awaitUninterruptibly()                                  | 当前线程进入等待状态直到被通知，从方法名称上可以看出该方法对中断不敏感 |
| long awaitNanos(long nanosTimeout) throws InterruptedException | 当前线程进入等待状态直到被通知、中断或者超时。返回值表示剩余的时间，如果在nanosTimeout纳秒之前被唤醒，那么返回值就是（nanosTimeout-实际耗时）如果返回值是0或者负数，那么可以认定已经超时了 |
| boolean awaitUntil(Date deadline) throws InterruptedException | 当前线程进入等待状态直到被通知、中断或者某个时间。如果没有到指定时间就被通知，方法返回true，否则，表示到了指定时间，方法返回false |
| void signal()                                                | 唤醒一个等待在Condition上的线程，该线程从等待方法返回前必须获得与Condition相关联的锁 |
| void signalAll()                                             | 唤醒所有等待在Condition上的线程，能够从等待方法返回的线程必须获得与Condition相关联的锁 |

​	获取一个Condition必须通过Lock的newCondition()方法。下面通过一个有界队列的示例来深入了解Condition的使用方式。有界队列是一种特殊的队列，当队列为空时，队列的获取操作将会阻塞获取线程，直到队列中有新增元素，当队列已满时，队列的插入操作将会阻塞插入线程，直到队列出现“空位”，如（代码清单5-21 BoundedQueue.java）所示。

```java
public class BoundedQueue<T> {
  private Object[] items;
  // 添加的下标，删除的下标和数组当前数量
  private int addIndex, removeIndex, count;
  private Lock lock = new ReentrantLock();
  private Condition notEmpty = lock.newCondition();
  private Condition notFull = lock.newCondition();
  public BoundedQueue(int size) {
    items = new Object[size];
  }
  // 添加一个元素，如果数组满，则添加线程进入等待状态，直到有"空位"
  public void add(T t) throws InterruptedException {
    lock.lock();
    try {
      while (count == items.length)
        notFull.await();
      items[addIndex] = t;
      if (++addIndex == items.length)
        addIndex = 0;
      ++count;
      notEmpty.signal();
    } finally {
      lock.unlock();
    }
  }
  // 由头部删除一个元素，如果数组空，则删除线程进入等待状态，直到有新添加元素
  @SuppressWarnings("unchecked")
  public T remove() throws InterruptedException {
    lock.lock();
    try {
      while (count == 0)
        notEmpty.await();
      Object x = items[removeIndex];
      if (++removeIndex == items.length)
        removeIndex = 0;
      --count;
      notFull.signal();
      return (T) x;
    } finally {
      lock.unlock();
    }
  }
}
```

​	上述示例中，BoundedQueue通过add(T t)方法添加一个元素，通过remove()方法移出一个元素。以添加方法为例。

​	**首先需要获得锁，目的是确保数组修改的可见性和排他性**。当数组数量等于数组长度时，表示数组已满，则调用`notFull.await()`，当前线程随之释放锁并进入等待状态。如果数组数量不等于数组长度，表示数组未满，则添加元素到数组中，同时通知等待在notEmpty上的线程，数组中已经有新元素可以获取。

​	在添加和删除方法中使用while循环而非if判断，目的是防止过早或意外的通知，只有条件符合才能够退出循环。回想之前提到的等待/通知的经典范式，二者是非常类似的。

### 5.6.2 Condition的实现分析

> [六、Lock的Condition（等待队列）接口](https://www.jianshu.com/p/08c9d3e1bea0)
>
> [AbstractQueuedSynchronizer(AQS)深入分析](AbstractQueuedSynchronizer(AQS)深入分析)

​	**ConditionObject是同步器AbstractQueuedSynchronizer的内部类，因为Condition的操作需要获取相关联的锁，所以作为同步器的内部类也较为合理。每个Condition对象都包含着一个队列（以下称为等待队列），该队列是Condition对象实现等待/通知功能的关键**。

​	下面将分析Condition的实现，主要包括：**等待队列**、**等待**和**通知**，下面提到的Condition如果不加说明均指的是ConditionObject。

#### 1. 等待队列

​	**等待队列是一个FIFO的队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态**。事实上，**节点的定义复用了同步器中节点的定义，也就是说，同步队列和等待队列中节点类型都是同步器的静态内部类AbstractQueuedSynchronizer.Node**。

​	一个Condition包含一个等待队列，Condition拥有首节点（firstWaiter）和尾节点（lastWaiter）。当前线程调用Condition.await()方法，将会以当前线程构造节点，并将节点从尾部加入等待队列，等待队列的基本结构如下图所示。

![](https://upload-images.jianshu.io/upload_images/7378149-9ea7544188cbf975.png)

​	如图所示，Condition拥有首尾节点的引用，而新增节点只需要将原有的尾节点nextWaiter指向它，并且更新尾节点即可。**上述节点引用更新的过程并没有使用CAS保证，原因在于调用await()方法的线程必定是获取了锁的线程，也就是说该过程是由锁来保证线程安全的**。

​	在Object的监视器模型上，一个对象拥有一个**同步队列**和**等待队列**，而并发包中的**Lock（更确切地说是同步器）拥有一个同步队列和多个等待队列**，其对应关系如下图所示。

![](https://upload-images.jianshu.io/upload_images/7378149-9613867df26d66d6.png)

​	如图所示，**Condition的实现是同步器的内部类，因此每个Condition实例都能够访问同步器提供的方法，相当于每个Condition都拥有所属同步器的引用**。

#### 2. 等待

​	**调用Condition的await()方法（或者以await开头的方法），会使当前线程进入等待队列并释放锁，同时线程状态变为等待状态**。当从await()方法返回时，当前线程一定获取了Condition相关联的锁。

​	**如果从队列（同步队列和等待队列）的角度看await()方法，当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中**。

​	Condition的await()方法，如（代码清单5-22 ConditionObject的await方法）所示。

```java
public final void await() throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  // 当前线程加入等待队列
  Node node = addConditionWaiter();
  // 释放同步状态，也就是释放锁
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  while (!isOnSyncQueue(node)) {
    LockSupport.park(this);
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null)
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```

​	调用该方法的线程成功获取了锁的线程，也就是同步队列中的首节点，<u>该方法会将当前线程构造成节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态</u>。

​	当等待队列中的节点被唤醒，则唤醒节点的线程开始尝试获取同步状态。如果不是通过其他线程调用`Condition.signal()`方法唤醒，而是对等待线程进行中断，则会抛出InterruptedException。

​	如果**从队列的角度去看，当前线程加入Condition的等待队列**，该过程如下图示。

![](https://upload-images.jianshu.io/upload_images/7378149-1c66869c2b6b4d21.png)

​	如图所示，**同步队列的首节点并不会直接加入等待队列，而是通过addConditionWaiter()方法把当前线程构造成一个新的节点并将其加入等待队列中**。

#### 3. 通知

​	调用Condition的signal()方法，将会唤醒在等待队列中等待时间最长的节点（首节点），**在唤醒节点之前，会将节点移到同步队列中**。

​	Condition的signal()方法，如（代码清单5-23 ConditionObject的signal方法）所示。

```java
public final void signal() {
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  Node first = firstWaiter;
  if (first != null)
    doSignal(first);
}
```

​	<u>调用该方法的前置条件是当前线程必须获取了锁，可以看到signal()方法进行了isHeldExclusively()检查，也就是当前线程必须是获取了锁的线程。接着获取等待队列的首节点，将其移动到同步队列并使用LockSupport唤醒节点中的线程。</u>

​	节点从等待队列移动到同步队列的过程如下图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020070701031919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dqc18xMDI0,size_16,color_FFFFFF,t_70#pic_center)

​	通过调用同步器的`enq(Node node)`方法，等待队列中的头节点线程安全地移动到同步队列。当节点移动到同步队列后，当前线程再使用LockSupport唤醒该节点的线程。

​	被唤醒后的线程，将从`await()`方法中的while循环中退出（`isOnSyncQueue(Node node)`方法返回true，节点已经在同步队列中），进而调用同步器的`acquireQueued()`方法加入到获取同步状态的竞争中。

​	成功获取同步状态（或者说锁）之后，被唤醒的线程将从先前调用的`await()`方法返回，此时该线程已经成功地获取了锁。

​	**Condition的`signalAll()`方法，相当于对等待队列中的每个节点均执行一次`signal()`方法，效果就是将等待队列中所有节点全部移动到同步队列中，并唤醒每个节点的线程。**

## 5.7 本章小结

​	本章介绍了Java并发包中与锁相关的API和组件，通过示例讲述了这些API和组件的使用方式以及需要注意的地方，并在此基础上详细地剖析了**队列同步器**、**重入锁**、**读写锁**以及**Condition**等API和组件的实现细节，只有理解这些API和组件的实现细节才能够更加准确地运用它们。

# 第6章 Java并发容器和框架

​	Java程序员进行并发编程时，相比于其他语言的程序员而言要倍感幸福，因为并发编程大师Doug Lea不遗余力地为Java开发者提供了非常多的并发容器和框架。本章让我们一起来见识一下大师操刀编写的并发容器和框架，并通过每节的原理分析一起来学习如何设计出精妙的并发程序。

## 6.1 ConcurrentHashMap的实现原理与使用

​	ConcurrentHashMap是线程安全且高效的HashMap。本节让我们一起研究一下该容器是如何在保证线程安全的同时又能保证高效的操作。

### 6.1.1 为什么要使用ConcurrentHashMap

​	在并发编程中使用HashMap可能导致程序死循环。而使用线程安全的HashTable效率又非常低下，基于以上两个原因，便有了ConcurrentHashMap的登场机会。

#### 1. 线程不安全的HashMap

​	在多线程环境下，使用HashMap进行put操作会引起死循环，导致CPU利用率接近100%，所以在并发情况下不能使用HashMap。例如，执行以下代码会引起死循环。

```java
final HashMap<String, String> map = new HashMap<String, String>(2);
Thread t = new Thread(new Runnable() {
  @Override
  public void run() {
    for (int i = 0; i < 10000; i++) {
      new Thread(new Runnable() {
        @Override
        public void run() {
          map.put(UUID.randomUUID().toString(), "");
        }
      }, "ftf" + i).start();
    }
  }
}, "ftf");
t.start();
t.join();
```

​	**HashMap在并发执行put操作时会引起死循环，是因为多线程会导致HashMap的Entry链表形成环形数据结构，一旦形成环形数据结构，Entry的next节点永远不为空，就会产生死循环获取Entry**。

> [并发的HashMap为什么会引起死循环？](https://blog.csdn.net/zhuqiuhui/article/details/51849692) <== 建议阅读
>
> JDK7用的头插法，JDK8用的尾插法。JDK7扩容后复制节点到新的HashMap上时，线程A的节点的next可能已经线程B提前修改，导致线程A在头插时，HashMap实现中的数组下的链表节点自成环。

#### 2. 效率低下的HashTable

​	HashTable容器使用synchronized来保证线程安全，但在线程竞争激烈的情况下HashTable的效率非常低下。因为当一个线程访问HashTable的同步方法，其他线程也访问HashTable的同步方法时，会进入阻塞或轮询状态。如线程1使用put进行元素添加，线程2不但不能使用put方法添加元素，也不能使用get方法来获取元素，所以竞争越激烈效率越低。

#### 3. ConcurrentHashMap的锁分段技术可有效提升并发访问率

​	HashTable容器在竞争激烈的并发环境下表现出效率低下的原因是所有访问HashTable的线程都必须竞争同一把锁，假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效提高并发访问效率，这就是**ConcurrentHashMap所使用的锁分段技术**。**首先将数据分成一段一段地存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问**。（JDK8直接是CAS，不再是JDK7分段锁）

### 6.1.2 ConcurrentHashMap的结构

> [Java并发编程的艺术之六----并发编程容器和框架](https://blog.csdn.net/huangwei18351/article/details/82975462)

​	通过ConcurrentHashMap的类图来分析ConcurrentHashMap的结构，如下图所示。

![img](https://img-blog.csdn.net/20181008232111253?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1YW5nd2VpMTgzNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

​	ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。Segment是一种可重入锁（ReentrantLock），在ConcurrentHashMap里扮演锁的角色；HashEntry则用于存储键值对数据。**一个ConcurrentHashMap里包含一个Segment数组。Segment的结构和HashMap类似，是一种数组和链表结构。一个Segment里包含一个HashEntry数组，每个HashEntry是一个链表结构的元素**，每个Segment守护着一个HashEntry数组里的元素，当对HashEntry数组的数据进行修改时，必须首先获得与它对应的Segment锁，如下图所示。

![img](https://img-blog.csdn.net/20181008232111409?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1YW5nd2VpMTgzNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

> [面试题：ConcurrentHashMap 1.7和1.8的区别](https://blog.csdn.net/u013374645/article/details/88700927)
>
> [七、Java并发容器ConcurrentHashMap](https://www.jianshu.com/p/0422f1601708)

### 6.1.3 ConcurrentHashMap的初始化

​	ConcurrentHashMap初始化方法是通过initialCapacity、loadFactor和concurrencyLevel等几个参数来初始化segment数组、段偏移量segmentShift、段掩码segmentMask和每个segment里的HashEntry数组来实现的。

#### 1. 初始化segments数组

​	让我们来看一下初始化segments数组的源代码。

```java
if (concurrencyLevel > MAX_SEGMENTS)
  concurrencyLevel = MAX_SEGMENTS;
int sshift = 0;
int ssize = 1;
while (ssize < concurrencyLevel) {
  ++sshift;
  ssize <<= 1;
}
segmentShift = 32 - sshift;
segmentMask = ssize - 1;
this.segments = Segment.newArray(ssize);
```

​	由上面的代码可知，segments数组的长度ssize是通过concurrencyLevel计算得出的。**为了能通过按位与的散列算法来定位segments数组的索引，必须保证segments数组的长度是2的N次方（power-of-two size），所以必须计算出一个大于或等于concurrencyLevel的最小的2的N次方值来作为segments数组的长度**。假如concurrencyLevel等于14、15或16，ssize都会等于16，即容器里锁的个数也是16。

​	注意：concurrencyLevel的最大值是65535，这意味着segments数组的长度最大为65536，对应的二进制是16位。

#### 2. 初始化segmentShift和segmentMask

​	这两个全局变量需要在定位segment时的散列算法里使用，sshift等于ssize从1向左移位的次数，在默认情况下concurrencyLevel等于16，1需要向左移位移动4次，所以sshift等于4。segmentShift用于定位参与散列运算的位数，segmentShift等于32减sshift，所以等于28，这里之所以用32是因为ConcurrentHashMap里的`hash()`方法输出的最大数是32位的，后面的测试中我们可以看到这点。segmentMask是散列运算的掩码，等于ssize减1，即15，掩码的二进制各个位的值都是1。因为ssize的最大长度是65536，所以segmentShift最大值是16，segmentMask最大值是65535，对应的二进制是16位，每个位都是1。

#### 3. 初始化每个segment

​	输入参数initialCapacity是ConcurrentHashMap的初始化容量，loadfactor是每个segment的负载因子，在构造方法里需要通过这两个参数来初始化数组中的每个segment。

```java
if (initialCapacity > MAXIMUM_CAPACITY)
  initialCapacity = MAXIMUM_CAPACITY;
int c = initialCapacity / ssize;
if (c * ssize < initialCapacity)
  ++c;
int cap = 1;
while (cap < c)
  cap <<= 1;
for (int i = 0; i < this.segments.length; ++i)
  this.segments[i] = new Segment<K,V>(cap, loadFactor);
```

​	上面代码中的变量cap就是segment里HashEntry数组的长度，它等于initialCapacity除以ssize的倍数c，如果c大于1，就会取大于等于c的2的N次方值，所以cap不是1，就是2的N次方。segment的容量threshold＝（int）cap*loadFactor，默认情况下initialCapacity等于16，loadfactor等于0.75，通过运算cap等于1，threshold等于零。

### 6.1.4 定位Segment

​	**既然ConcurrentHashMap使用分段锁Segment来保护不同段的数据，那么在插入和获取元素的时候，必须先通过散列算法定位到Segment**。可以看到ConcurrentHashMap会首先使用Wang/Jenkins hash的变种算法对元素的hashCode进行一次再散列。

```java
private static int hash(int h) {
  h += (h << 15) ^ 0xffffcd7d;
  h ^= (h >>> 10);
  h += (h << 3);
  h ^= (h >>> 6);
  h += (h << 2) + (h << 14);
  return h ^ (h >>> 16);
}
```

​	**之所以进行再散列，目的是减少散列冲突，使元素能够均匀地分布在不同的Segment上，从而提高容器的存取效率**。假如散列的质量差到极点，那么所有的元素都在一个Segment中，不仅存取元素缓慢，分段锁也会失去意义。笔者（本书作者）做了一个测试，不通过再散列而直接执行散列计算。

```java
System.out.println(Integer.parseInt("0001111", 2) & 15);
System.out.println(Integer.parseInt("0011111", 2) & 15);
System.out.println(Integer.parseInt("0111111", 2) & 15);
System.out.println(Integer.parseInt("1111111", 2) & 15);
```

​	计算后输出的散列值全是15，通过这个例子可以发现，如果不进行再散列，散列冲突会非常严重，因为只要低位一样，无论高位是什么数，其散列值总是一样。我们再把上面的二进制数据进行再散列后结果如下（为了方便阅读，不足32位的高位补了0，每隔4位用竖线分割下）。

```none
0100｜0111｜0110｜0111｜1101｜1010｜0100｜1110
1111｜0111｜0100｜0011｜0000｜0001｜1011｜1000
0111｜0111｜0110｜1001｜0100｜0110｜0011｜1110
1000｜0011｜0000｜0000｜1100｜1000｜0001｜1010
```

​	可以发现，每一位的数据都散列开了，**通过这种再散列能让数字的每一位都参加到散列运算当中，从而减少散列冲突**。ConcurrentHashMap通过以下散列算法定位segment。

```java
final Segment<K,V> segmentFor(int hash) {
  return segments[(hash >>> segmentShift) & segmentMask];
}
```

​	默认情况下segmentShift为28，segmentMask为15，再散列后的数最大是32位二进制数据，向右无符号移动28位，意思是让高4位参与到散列运算中，（hash>>>segmentShift）&segmentMask的运算结果分别是4、15、7和8，可以看到散列值没有发生冲突。

### 6.1.5 ConcurrentHashMap的操作

​	本节介绍ConcurrentHashMap的3种操作——get操作、put操作和size操作。

#### 1. get操作

​	**Segment的get操作实现非常简单和高效。先经过一次再散列，然后使用这个散列值通过散列运算定位到Segment，再通过散列算法定位到元素**，代码如下。

```java
public V get(Object key) {
  int hash = hash(key.hashCode());
  return segmentFor(hash).get(key, hash);
}
```

​	**get操作的高效之处在于整个get过程不需要加锁，除非读到的值是空才会加锁重读**。我们知道HashTable容器的get方法是需要加锁的，那么ConcurrentHashMap的get操作是如何做到不加锁的呢？原因是它的**get方法里将要使用的共享变量都定义成volatile类型**，如用于统计当前Segement大小的count字段和用于存储值的HashEntry的value。**定义成volatile的变量，能够在线程之间保持可见性，能够被多线程同时读，并且保证不会读到过期的值，但是只能被单线程写（有一种情况可以被多线程写，就是写入的值不依赖于原值），在get操作里只需要读不需要写共享变量count和value，所以可以不用加锁**。**之所以不会读到过期的值，是因为根据Java内存模型的happen-before原则，对volatile字段的写入操作先于读操作，即使两个线程同时修改和获取volatile变量，get操作也能拿到最新的值，这是用volatile替换锁的经典应用场景。**

```java
transient volatile int count;
volatile V value;
```

​	在定位元素的代码里我们可以发现，定位HashEntry和定位Segment的散列算法虽然一样，都与数组的长度减去1再相“与”，但是相“与”的值不一样，<u>定位Segment使用的是元素的hashcode通过再散列后得到的值的高位，而定位HashEntry直接使用的是再散列后的值</u>。其目的是避免两次散列后的值一样，虽然元素在Segment里散列开了，但是却没有在HashEntry里散列开。

```java
hash >>> segmentShift) & segmentMask　　 // 定位Segment所使用的hash算法
int index = hash & (tab.length - 1);　　 // 定位HashEntry所使用的hash算法
```

#### 2. put操作

​	**由于put方法里需要对共享变量进行写入操作，所以为了线程安全，在操作共享变量时必须加锁**。put方法首先定位到Segment，然后在Segment里进行插入操作。<u>插入操作需要经历两个步骤，第一步判断是否需要对Segment里的HashEntry数组进行扩容，第二步定位添加元素的位置，然后将其放在HashEntry数组里</u>。

1. **是否需要扩容**

   在插入元素前会先判断Segment里的HashEntry数组是否超过容量（threshold），如果超过阈值，则对数组进行扩容。值得一提的是，Segment的扩容判断比HashMap更恰当，因为<u>HashMap是在插入元素后判断元素是否已经到达容量的，如果到达了就进行扩容，但是很有可能扩容之后没有新元素插入，这时HashMap就进行了一次无效的扩容</u>。

2. 如何扩容

   在扩容的时候，首先会创建一个容量是原来容量**两倍**的数组，然后将原数组里的元素进行再散列后插入到新的数组里。为了高效，**ConcurrentHashMap不会对整个容器进行扩容，而只对某个segment进行扩容**。

#### 3. size操作

​	<u>如果要统计整个ConcurrentHashMap里元素的大小，就必须统计所有Segment里元素的大小后求和</u>。Segment里的全局变量count是一个volatile变量，那么在**多线程**场景下，是不是直接把所有Segment的count相加就可以得到整个ConcurrentHashMap大小了呢？不是的，虽然相加时可以获取每个Segment的count的最新值，但是可能累加前使用的count发生了变化，那么统计结果就不准了。所以，<u>最安全的做法是在统计size的时候把所有Segment的put、remove和clean方法全部锁住，但是这种做法显然非常低效</u>。

​	因为在累加count操作过程中，之前累加过的count发生变化的几率非常小，所以**ConcurrentHashMap的做法是先尝试2次通过不锁住Segment的方式来统计各个Segment大小，如果统计的过程中，容器的count发生了变化，则再采用加锁的方式来统计所有Segment的大小**。

​	<u>那么ConcurrentHashMap是如何判断在统计的时候容器是否发生了变化呢？使用modCount变量，在put、remove和clean方法里操作元素前都会将变量modCount进行加1，那么在统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化</u>。

## 6.2 ConcurrentLinkedQueue

​	在并发编程中，有时候需要使用线程安全的队列。如果要**实现一个线程安全的队列有两种方式：一种是使用阻塞算法，另一种是使用非阻塞算法**。使用阻塞算法的队列可以用一个锁（入队和出队用同一把锁）或两个锁（入队和出队用不同的锁）等方式来实现。非阻塞的实现方式则可以使用**循环CAS**的方式来实现。本节让我们一起来研究一下Doug Lea是如何使用非阻塞的方式来实现线程安全队列ConcurrentLinkedQueue的，相信从大师身上我们能学到不少并发编程的技巧。

​	ConcurrentLinkedQueue是一个基于链接节点的无界线程安全队列，它采用先进先出的规则对节点进行排序，当我们添加一个元素的时候，它会**添加到队列的尾部**；当我们获取一个元素时，它会返回队列头部的元素。它采用了**“wait-free”算法（即CAS算法）**来实现，该算法在Michael&Scott算法上进行了一些修改。

### 6.2.1 ConcurrentLinkedQueue的结构

> [ConcurrentLinkedQueue的实现原理分析](https://blog.csdn.net/zhao9tian/article/details/39613977)

​	通过ConcurrentLinkedQueue的类图来分析一下它的结构，如下图所示。

![img](http://ifeve.com/wp-content/uploads/2013/01/ConcurrentLinkedQueue%E7%B1%BB%E5%9B%BE.jpg)

​	ConcurrentLinkedQueue由head节点和tail节点组成，每个节点（Node）由节点元素（item）和指向下一个节点（next）的引用组成，节点与节点之间就是通过这个next关联起来，从而组成一张链表结构的队列。默认情况下head节点存储的元素为空，tail节点等于head节点。

```java
private transient volatile Node<E> tail = head;
```

### 6.2.2 入队列

> [ConcurrentLinkedQueue的实现原理分析](ConcurrentLinkedQueue的实现原理分析)

​	本节将介绍入队列的相关知识。

#### 1. 入队列的过程

​	入队列就是将入队节点**添加到队列的尾部**。为了方便理解入队时队列的变化，以及head节点和tail节点的变化，这里以一个示例来展开介绍。假设我们想在一个队列中依次插入4个节点，为了帮助大家理解，每添加一个节点就做了一个队列的快照图，如下图所示。

​	下图所示的过程如下。

+ 添加元素1。队列更新head节点的next节点为元素1节点。又因为tail节点默认情况下等于head节点，所以它们的next节点都指向元素1节点。

+ 添加元素2。队列首先设置元素1节点的next节点为元素2节点，然后更新tail节点指向元素2节点。
+ 添加元素3，设置tail节点的next节点为元素3节点。
+ 添加元素4，设置元素3的next节点为元素4节点，然后将tail节点指向元素4节点。

![img](http://ifeve.com/wp-content/uploads/2013/01/ConcurrentLinekedQueue%E9%98%9F%E5%88%97%E5%85%A5%E9%98%9F%E7%BB%93%E6%9E%84%E5%8F%98%E5%8C%96%E5%9B%BE.jpg)

​	通过调试入队过程并观察head节点和tail节点的变化，发现入队主要做两件事情：第一是将入队节点设置成当前队列尾节点的下一个节点；第二**是更新tail节点，如果tail节点的next节点不为空，则将入队节点设置成tail节点，如果tail节点的next节点为空，则将入队节点设置成tail的next节点，所以tail节点不总是尾节点**（理解这一点对于我们研究源码会非常有帮助）。

​	通过对上面的分析，我们从单线程入队的角度理解了入队过程，但是多个线程同时进行入队的情况就变得更加复杂了，因为可能会出现其他线程插队的情况。如果有一个线程正在入队，那么它必须先获取尾节点，然后设置尾节点的下一个节点为入队节点，但这时可能有另外一个线程插队了，那么队列的尾节点就会发生变化，这时当前线程要暂停入队操作，然后重新获取尾节点。让我们再通过源码来详细分析一下它是如何使用CAS算法来入队的。

```java
public boolean offer(E e) {
  if (e == null) throw new NullPointerException();
  // 入队前，创建一个入队节点
  Node<E> n = new Node<E>(e);
  retry:
  // 死循环，入队不成功反复入队。
  for (;;) {
    // 创建一个指向tail节点的引用
    Node<E> t = tail;
    // p用来表示队列的尾节点，默认情况下等于tail节点。
    Node<E> p = t;
    for (int hops = 0; ; hops++) {
      // 获得p节点的下一个节点。
      Node<E> next = succ(p);
      // next节点不为空，说明p不是尾节点，需要更新p后在将它指向next节点
      if (next != null) {
        // 循环了两次及其以上，并且当前节点还是不等于尾节点
        if (hops > HOPS && t != tail)
          continue retry;
        p = next;
      }
      // 如果p是尾节点，则设置p节点的next节点为入队节点。
      else if (p.casNext(null, n)) {
        /*如果tail节点有大于等于1个next节点，则将入队节点设置成tail节点，
更新失败了也没关系，因为失败了表示有其他线程成功更新了tail节点*/
        if (hops >= HOPS)
          casTail(t, n); // 更新tail节点，允许失败
        return true;
      }
      // p有next节点,表示p的next节点是尾节点，则重新设置p节点
      else {
        p = succ(p);
      }
    }
  }
}
```

​	**从源代码角度来看，整个入队过程主要做两件事情：第一是定位出尾节点；第二是使用CAS算法将入队节点设置成尾节点的next节点，如不成功则重试**。

#### 2. 定位尾节点

​	**tail节点并不总是尾节点，所以每次入队都必须先通过tail节点来找到尾节点。尾节点可能是tail节点，也可能是tail节点的next节点**。代码中循环体中的第一个if就是判断tail是否有next节点，有则表示next节点可能是尾节点。获取tail节点的next节点需要注意的是p节点等于p的next节点的情况，只有一种可能就是p节点和p的next节点都等于空，表示这个队列刚初始化，正准备添加节点，所以需要返回head节点。获取p节点的next节点代码如下。

```java
final Node<E> succ(Node<E> p) {
  Node<E> next = p.getNext();
  return (p == next) head : next;
}
```

#### 3. 设置入队节点为尾节点

​	p.casNext（null，n）方法用于将入队节点设置为当前队列尾节点的next节点，如果p是null，表示p是当前队列的尾节点，如果不为null，表示有其他线程更新了尾节点，则需要重新获取当前队列的尾节点。

#### 4. HOPS的设计意图

​	上面分析过对于先进先出的队列入队所要做的事情是将入队节点设置成尾节点，doug lea写的代码和逻辑还是稍微有点复杂。那么，我用以下方式来实现是否可行？

```java
public boolean offer(E e) {
  if (e == null)
    throw new NullPointerException();
  Node<E> n = new Node<E>(e);
  for (;;) {
    Node<E> t = tail;
    if (t.casNext(null, n) && casTail(t, n)) {
      return true;
    }
  }
}
```

​	让tail节点永远作为队列的尾节点，这样实现代码量非常少，而且逻辑清晰和易懂。但是，这么做有个缺点，每次都需要使用循环CAS更新tail节点。如果能减少CAS更新tail节点的次数，就能提高入队的效率，所以doug lea使用hops变量来控制并减少tail节点的更新频率，并不是每次节点入队后都将tail节点更新成尾节点，而是当tail节点和尾节点的距离大于等于常量HOPS的值（默认等于1）时才更新tail节点，tail和尾节点的距离越长，使用CAS更新tail节点的次数就会越少，但是距离越长带来的负面效果就是每次入队时定位尾节点的时间就越长，因为循环体需要多循环一次来定位出尾节点，但是这样仍然能提高入队的效率，因为**从本质上来看它通过增加对volatile变量的读操作来减少对volatile变量的写操作，而对volatile变量的写操作开销要远远大于读操作，所以入队效率会有所提升**。

```java
private static final int HOPS = 1;
```

**注意：入队方法永远返回true，所以不要通过返回值判断入队是否成功**。

### 6.2.3 出队列

> [Java并发编程的艺术之六----并发编程容器和框架](https://blog.csdn.net/huangwei18351/article/details/82975462)

​	出队列的就是从队列里返回一个节点元素，并清空该节点对元素的引用。让我们通过每个节点出队的快照来观察一下head节点的变化，如下图所示。

![img](https://img-blog.csdn.net/20181008232111730?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1YW5nd2VpMTgzNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

​	从图中可知，**并不是每次出队时都更新head节点，当head节点里有元素时，直接弹出head节点里的元素，而不会更新head节点。只有当head节点里没有元素时，出队操作才会更新head节点。这种做法也是通过hops变量来减少使用CAS更新head节点的消耗，从而提高出队效率**。让我们再通过源码来深入分析下出队过程。

```java
public E poll() {
  Node<E> h = head;
  // p表示头节点，需要出队的节点
  Node<E> p = h;
  for (int hops = 0;; hops++) {
    // 获取p节点的元素
    E item = p.getItem();
    // 如果p节点的元素不为空，使用CAS设置p节点引用的元素为null,
    // 如果成功则返回p节点的元素。
    if (item != null && p.casItem(item, null)) {
      if (hops >= HOPS) {
        // 将p节点下一个节点设置成head节点
        Node<E> q = p.getNext();
        updateHead(h, (q != null) q : p);
      }
      return item;
    }
    // 如果头节点的元素为空或头节点发生了变化，这说明头节点已经被另外
    // 一个线程修改了。那么获取p节点的下一个节点
    Node<E> next = succ(p);
    // 如果p的下一个节点也为空，说明这个队列已经空了
    if (next == null) {
      // 更新头节点。
      updateHead(h, p);
      break;
    }
    // 如果下一个元素不为空，则将头节点的下一个节点设置成头节点
    p = next;
  }
  return null;
}
```

​	**首先获取头节点的元素，然后判断头节点元素是否为空，如果为空，表示另外一个线程已经进行了一次出队操作将该节点的元素取走，如果不为空，则使用CAS的方式将头节点的引用设置成null，如果CAS成功，则直接返回头节点的元素，如果不成功，表示另外一个线程已经进行了一次出队操作更新了head节点，导致元素发生了变化，需要重新获取头节点**。

## 6.3 Java中的阻塞队列

​	本节将介绍什么是阻塞队列，以及Java中阻塞队列的4种处理方式，并介绍Java 7中提供的7种阻塞队列，最后分析阻塞队列的一种实现方式。

### 6.3.1 什么是阻塞队列

​	阻塞队列（BlockingQueue）是一个支持两个附加操作的队列。这两个附加的操作支持阻塞的插入和移除方法。

1. **支持阻塞的插入**方法：意思是当队列满时，队列会阻塞插入元素的线程，直到队列不满。

2. **支持阻塞的移除**方法：意思是在队列为空时，获取元素的线程会等待队列变为非空。**阻塞队列常用于生产者和消费者的场景**，生产者是向队列里添加元素的线程，消费者是从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的容器。

​	在阻塞队列不可用时，这两个附加操作提供了4种处理方式，如下表所示。

| 方法/处理方式 | 抛出异常  | 返回特殊值 | 一直阻塞 | 超时退出             |
| ------------- | --------- | ---------- | -------- | -------------------- |
| 插入方法      | add(e)    | offer(e)   | put(e)   | Offer(e, time, unit) |
| 移除方法      | remove()  | poll()     | take()   | poll(time, unit)     |
| 检查方法      | element() | peek()     | 不可用   | 不可用               |

+ 抛出异常：当队列满时，如果再往队列里插入元素，会抛出IllegalStateException（"Queuefull"）异常。当队列空时，从队列里获取元素会抛出NoSuchElementException异常。

+ 返回特殊值：当往队列插入元素时，会返回元素是否插入成功，成功返回true。如果是移除方法，则是从队列里取出一个元素，如果没有则返回null。

+ **一直阻塞**：当阻塞队列满时，如果生产者线程往队列里put元素，队列会一直阻塞生产者线程，直到队列可用或者响应中断退出。当队列空时，如果消费者线程从队列里take元素，队列会阻塞住消费者线程，直到队列不为空。

+ **超时退出**：当阻塞队列满时，如果生产者线程往队列里插入元素，队列会阻塞生产者线程一段时间，如果超过了指定的时间，生产者线程就会退出。

​	这两个附加操作的4种处理方式不方便记忆，所以我（书的作者）找了一下这几个方法的规律。put和take分别尾首含有字母t，offer和poll都含有字母o。

​	***注意：如果是无界阻塞队列，队列不可能会出现满的情况，所以使用put或offer方法永远不会被阻塞，而且使用offer方法时，该方法永远返回true。***

### 6.3.2 Java里的阻塞队列

JDK 7提供了7个阻塞队列，如下。

+ ArrayBlockingQueue：一个由数组结构组成的**有界**阻塞队列。

+ LinkedBlockingQueue：一个由链表结构组成的**有界**阻塞队列。

+ PriorityBlockingQueue：一个支持优先级排序的**无界**阻塞队列。

+ DelayQueue：一个使用优先级队列实现的**无界**阻塞队列。

+ SynchronousQueue：一个**不存储元素**的阻塞队列。

+ LinkedTransferQueue：一个由链表结构组成的**无界**阻塞队列。

+ LinkedBlockingDeque：一个由链表结构组成的**双向**阻塞队列。

#### 1. ArrayBlockingQueue

​	ArrayBlockingQueue是一个用数组实现的有界阻塞队列。此队列按照**先进先出（FIFO）**的原则对元素进行排序。

​	**默认情况下不保证线程公平的访问队列**，所谓公平访问队列是指阻塞的线程，可以按照阻塞的先后顺序访问队列，即先阻塞线程先访问队列。非公平性是对先等待的线程是非公平的，当队列可用时，阻塞的线程都可以争夺访问队列的资格，有可能先阻塞的线程最后才访问队列。**为了保证公平性，通常会降低吞吐量**。我们可以使用以下代码创建一个公平的阻塞队列。

```java
ArrayBlockingQueue fairQueue = new ArrayBlockingQueue(1000,true);
```

​	访问者的公平性是使用**可重入锁**实现的，代码如下。

```java
public ArrayBlockingQueue(int capacity, boolean fair) {
  if (capacity <= 0)
    throw new IllegalArgumentException();
  this.items = new Object[capacity];
  lock = new ReentrantLock(fair);
  notEmpty = lock.newCondition();
  notFull = lock.newCondition();
}
```

#### 2. LinkedBlockingQueue

​	LinkedBlockingQueue是一个用链表实现的**有界**阻塞队列。此队列的**默认和最大长度为Integer.MAX_VALUE**。此队列按照**先进先出**的原则对元素进行排序。

#### 3. PriorityBlockingQueue

​	PriorityBlockingQueue是一个**支持优先级**的**无界**阻塞队列。**默认情况下元素采取自然顺序升序排列**。也可以自定义类实现compareTo()方法来指定元素排序规则，或者初始化PriorityBlockingQueue时，指定构造参数Comparator来对元素进行排序。需要注意的是不能保证同优先级元素的顺序。

#### 4. DelayQueue

​	DelayQueue是一个**支持延时获取元素**的无界阻塞队列。队列使用PriorityQueue来实现。**队列中的元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素**。

​	DelayQueue非常有用，可以将DelayQueue运用在以下应用场景。

+ **缓存系统的设计**：可以用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。

+ **定时任务调度**：使用DelayQueue保存当天将会执行的任务和执行时间，一旦从DelayQueue中获取到任务就开始执行，比如**TimerQueue就是使用DelayQueue实现的**。

##### 1. 如何实现Delayed接口

​	DelayQueue队列的元素必须实现Delayed接口。我们可以参考ScheduledThreadPoolExecutor里ScheduledFutureTask类的实现，一共有三步。

​	第一步：在对象创建的时候，初始化基本数据。使用time记录当前对象延迟到什么时候可以使用，使用sequenceNumber来标识元素在队列中的先后顺序。代码如下。

```java
private static final AtomicLong sequencer = new AtomicLong(0);
ScheduledFutureTask(Runnable r, V result, long ns, long period) {
  super(r, result);
  this.time = ns;
  this.period = period;
  this.sequenceNumber = sequencer.getAndIncrement();
}
```

​	第二步：实现getDelay方法，该方法返回当前元素还需要延时多长时间，单位是纳秒，代码如下。

```java
public long getDelay(TimeUnit unit) {
  return unit.convert(time - now(), TimeUnit.NANOSECONDS);
}
```

​	通过构造函数可以看出延迟时间参数ns的单位是纳秒，<u>自己设计的时候最好使用纳秒</u>，因为实现getDelay()方法时可以指定任意单位，一旦以秒或分作为单位，而延时时间又精确不到纳秒就麻烦了。使用时请注意当time小于当前时间时，getDelay会返回负数。

​	第三步：实现compareTo方法来指定元素的顺序。例如，让延时时间最长的放在队列的末尾。实现代码如下。

```java
public int compareTo(Delayed other) {
  if (other == this)　　// compare zero ONLY if same object
    return 0;
  if (other instanceof ScheduledFutureTask) {
    ScheduledFutureTask<> x = (ScheduledFutureTask<>)other;
    long diff = time - x.time;
    if (diff < 0)
      return -1;
    else if (diff > 0)
      return 1;
    else if (sequenceNumber < x.sequenceNumber)
      return -1;
    else
      return 1;
  }
  long d = (getDelay(TimeUnit.NANOSECONDS) -
            other.getDelay(TimeUnit.NANOSECONDS));
  return (d == 0) 0 : ((d < 0) -1 : 1);
}
```

##### 2. 如何实现延时阻塞队列

​	**延时阻塞队列的实现很简单，当消费者从队列里获取元素时，如果元素没有达到延时时间，就阻塞当前线程**。

```java
long delay = first.getDelay(TimeUnit.NANOSECONDS);
if (delay <= 0)
  return q.poll();
else if (leader != null)
  available.await();
else {
  Thread thisThread = Thread.currentThread();
  leader = thisThread;
  try {
    available.awaitNanos(delay);
  } finally {
    if (leader == thisThread)
      leader = null;
  }
}
```

​	代码中的变量leader是一个等待获取队列头部元素的线程。如果leader不等于空，表示已经有线程在等待获取队列的头元素。所以，使用await()方法让当前线程等待信号。如果leader等于空，则把当前线程设置成leader，并使用awaitNanos()方法让当前线程等待接收信号或等待delay时间。

#### 5. SynchronousQueue

​	SynchronousQueue是一个**不存储元素**的阻塞队列。<u>每一个put操作必须等待一个take操作，否则不能继续添加元素</u>。

​	**它支持公平访问队列**。**默认情况下线程采用非公平性策略访问队列**。使用以下构造方法可以创建公平性访问的SynchronousQueue，如果设置为true，则等待的线程会采用先进先出的顺序访问队列。

```java
public SynchronousQueue(boolean fair) {
  transferer = fair new TransferQueue() : new TransferStack();
}
```

​	SynchronousQueue可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素，非常适合传递性场景。**SynchronousQueue的吞吐量高于LinkedBlockingQueue和ArrayBlockingQueue**。

#### 6. LinkedTransferQueue

​	LinkedTransferQueue是一个由链表结构组成的无界阻塞TransferQueue队列。相对于其他阻塞队列，LinkedTransferQueue多了tryTransfer和transfer方法。

##### 1. transfer方法

​	<u>如果当前有消费者正在等待接收元素（消费者使用take()方法或带时间限制的poll()方法时），transfer方法可以把生产者传入的元素立刻transfer（传输）给消费者。如果没有消费者在等待接收元素，transfer方法会将元素存放在队列的tail节点，并等到该元素被消费者消费了才返回</u>。transfer方法的关键代码如下。

```java
Node pred = tryAppend(s, haveData);
return awaitMatch(s, pred, e, (how == TIMED), nanos);
```

​	第一行代码是试图把存放当前元素的s节点作为tail节点。第二行代码是让CPU自旋等待消费者消费元素。**因为自旋会消耗CPU，所以自旋一定的次数后使用Thread.yield()方法来暂停当前正在执行的线程，并执行其他线程**。

##### 2. tryTransfer方法

​	**tryTransfer方法是用来试探生产者传入的元素是否能直接传给消费者。如果没有消费者等待接收元素，则返回false。和transfer方法的区别是tryTransfer方法无论消费者是否接收，方法立即返回，而transfer方法是必须等到消费者消费了才返回**。

​	对于带有时间限制的`tryTransfer（E e，long timeout，TimeUnit unit）`方法，试图把生产者传入的元素直接传给消费者，但是如果没有消费者消费该元素则等待指定的时间再返回，如果超时还没消费元素，则返回false，如果在超时时间内消费了元素，则返回true。

#### 7. LinkedBlockingDeque

​	LinkedBlockingDeque是一个由链表结构组成的双向阻塞队列。所谓双向队列指的是可以从队列的两端插入和移出元素。双向队列因为多了一个操作队列的入口，在多线程同时入队时，也就减少了一半的竞争。相比其他的阻塞队列，LinkedBlockingDeque多了addFirst、addLast、offerFirst、offerLast、peekFirst和peekLast等方法，以First单词结尾的方法，表示插入、获取（peek）或移除双端队列的第一个元素。以Last单词结尾的方法，表示插入、获取或移除双端队列的最后一个元素。另外，**插入方法add等同于addLast，移除方法remove等效于removeFirst**。**但是take方法却等同于takeFirst**，不知道是不是JDK的bug，使用时还是用带有First和Last后缀的方法更清楚。

​	在初始化LinkedBlockingDeque时可以设置容量防止其过度膨胀。另外，**双向阻塞队列可以运用在“工作窃取”模式中**。

### 6.3.3 阻塞队列的实现原理

​	如果队列是空的，消费者会一直等待，当生产者添加元素时，消费者是如何知道当前队列有元素的呢？如果让你来设计阻塞队列你会如何设计，如何让生产者和消费者进行高效率的通信呢？让我们先来看看JDK是如何实现的。

​	**使用通知模式实现**。所谓通知模式，就是当生产者往满的队列里添加元素时会阻塞住生产者，当消费者消费了一个队列中的元素后，会通知生产者当前队列可用。通过查看JDK源码发现ArrayBlockingQueue使用了Condition来实现，代码如下。

```java
private final Condition notFull;
private final Condition notEmpty;
public ArrayBlockingQueue(int capacity, boolean fair) {
  // 省略其他代码
  notEmpty = lock.newCondition();
  notFull = lock.newCondition();
}
public void put(E e) throws InterruptedException {
  checkNotNull(e);
  final ReentrantLock lock = this.lock;
  lock.lockInterruptibly();
  try {
    while (count == items.length)
      notFull.await();
    insert(e);
  } finally {
    lock.unlock();
  }
} 
public E take() throws InterruptedException {
  final ReentrantLock lock = this.lock;
  lock.lockInterruptibly();
  try {
    while (count == 0)
      notEmpty.await();
    return extract();
  } finally {
    lock.unlock();
  }
} 
private void insert(E x) {
  items[putIndex] = x;
  putIndex = inc(putIndex);
  ++count;
  notEmpty.signal();
}
```

​	当往队列里插入一个元素时，如果队列不可用，那么阻塞生产者主要通过`LockSupport.park（this）`来实现。

```java
public final void await() throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  Node node = addConditionWaiter();
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  while (!isOnSyncQueue(node)) {
    LockSupport.park(this);
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null) // clean up if cancelled
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```

​	继续进入源码，发现调用setBlocker先保存一下将要阻塞的线程，然后调用unsafe.park阻塞当前线程。

```java
public static void park(Object blocker) {
  Thread t = Thread.currentThread();
  setBlocker(t, blocker);
  unsafe.park(false, 0L);
  setBlocker(t, null);
}
```

​	unsafe.park是个native方法，代码如下。

```java
public native void park(boolean isAbsolute, long time);
```

​	park这个方法会阻塞当前线程，只有以下4种情况中的一种发生时，该方法才会返回。

+ 与park对应的unpark执行或已经执行时。“已经执行”是指unpark先执行，然后再执行park的情况。

+ 线程被中断时。

+ 等待完time参数指定的毫秒数时。

+ 异常现象发生时，这个异常现象没有任何原因。

​	继续看一下JVM是如何实现park方法：park在不同的操作系统中使用不同的方式实现，在Linux下使用的是系统方法`pthread_cond_wait`实现。实现代码在JVM源码路径src/os/linux/vm/os_linux.cpp里的`os::PlatformEvent::park`方法，代码如下。

```cpp
void os::PlatformEvent::park() {
  int v ;
  for (;;) {
    v = _Event ;
    if (Atomic::cmpxchg (v-1, &_Event, v) == v) break ;
  }
  guarantee (v >= 0, "invariant") ;
  if (v == 0) {
    // Do this the hard way by blocking ...
    int status = pthread_mutex_lock(_mutex);
    assert_status(status == 0, status, "mutex_lock");
    guarantee (_nParked == 0, "invariant") ;
    ++ _nParked ;
    while (_Event < 0) {
      status = pthread_cond_wait(_cond, _mutex);
      // for some reason, under 2.7 lwp_cond_wait() may return ETIME ...
      // Treat this the same as if the wait was interrupted
      if (status == ETIME) { status = EINTR; }
      assert_status(status == 0 || status == EINTR, status, "cond_wait");
    }
    -- _nParked ;
    // In theory we could move the ST of 0 into _Event past the unlock(),
    // but then we'd need a MEMBAR after the ST.
    _Event = 0 ;
    status = pthread_mutex_unlock(_mutex);
    assert_status(status == 0, status, "mutex_unlock");
  }
  guarantee (_Event >= 0, "invariant") ;
}
```

​	pthread_cond_wait是一个多线程的条件变量函数，cond是condition的缩写，字面意思可以理解为线程在等待一个条件发生，这个条件是一个全局变量。这个方法接收两个参数：一个共享变量_cond，一个互斥量_mutex。而unpark方法在Linux下是使用pthread_cond_signal实现的。park方法在Windows下则是使用WaitForSingleObject实现的。想知道pthread_cond_wait是如何实现的，可以参考glibc-2.5的nptl/sysdeps/pthread/pthread_cond_wait.c。

​	当线程被阻塞队列阻塞时，线程会进入WAITING（parking）状态。我们可以使用jstack dump阻塞的生产者线程看到这点，如下。

```none
"main" prio=5 tid=0x00007fc83c000000 nid=0x10164e000 waiting on condition [0x000000010164d000]
java.lang.Thread.State: WAITING (parking)
at sun.misc.Unsafe.park(Native Method)
- parking to wait for <0x0000000140559fe8> (a java.util.concurrent.locks.
AbstractQueuedSynchronizer$ConditionObject)
at java.util.concurrent.locks.LockSupport.park(LockSupport.java:186)
at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.
await(AbstractQueuedSynchronizer.java:2043)
at java.util.concurrent.ArrayBlockingQueue.put(ArrayBlockingQueue.java:324)
at blockingqueue.ArrayBlockingQueueTest.main(ArrayBlockingQueueTest.java:
```

## 6.4 Fork/Join框架

​	本节将会介绍Fork/Join框架的基本原理、算法、设计方式、应用与实现等。

### 6.4.1 什么是Fork/Join框架

> [Java中并行执行任务的框架Fork/Join](https://blog.csdn.net/apeopl/article/details/82591319)

​	**Fork/Join框架是Java 7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架**。

​	我们再通过Fork和Join这两个单词来理解一下Fork/Join框架。Fork就是把一个大任务切分为若干子任务并行的执行，Join就是合并这些子任务的执行结果，最后得到这个大任务的结果。比如计算1+2+…+10000，可以分割成10个子任务，每个子任务分别对1000个数进行求和，最终汇总这10个子任务的结果。Fork/Join的运行流程如下图所示。

![img](https://img-blog.csdn.net/20180910161235530?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FwZW9wbA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 6.4.2 工作窃取算法

> [工作窃取算法 work-stealing](https://blog.csdn.net/pange1991/article/details/80944797)

​	**工作窃取（work-stealing）算法是指某个线程从其他队列里窃取任务来执行**。那么，为什么需要使用工作窃取算法呢？假如我们需要做一个比较大的任务，可以把这个任务分割为若干互不依赖的子任务，为了减少线程间的竞争，把这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务，线程和队列一一对应。比如A线程负责处理A队列里的任务。但是，有的线程会先把自己队列里的任务干完，而其他线程对应的队列里还有任务等待处理。**干完活的线程与其等着，不如去帮其他线程干活，于是它就去其他线程的队列里窃取一个任务来执行。而在这时它们会访问同一个队列，所以为了减少窃取任务线程和被窃取任务线程之间的竞争，通常会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行**。

​	工作窃取的运行流程如下图所示。

![这里写图片描述](https://img-blog.csdn.net/20180707151815286?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3BhbmdlMTk5MQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

+ **工作窃取算法的优点：充分利用线程进行并行计算，减少了线程间的竞争**。
+ **工作窃取算法的缺点：在某些情况下还是存在竞争，比如双端队列里只有一个任务时。并且该算法会消耗了更多的系统资源，比如创建多个线程和多个双端队列**。

### 6.4.3 Fork/Join框架的设计

​	我们已经很清楚Fork/Join框架的需求了，那么可以思考一下，如果让我们来设计一个Fork/Join框架，该如何设计？这个思考有助于你理解Fork/Join框架的设计。

​	步骤1　**分割任务**。首先我们需要有一个fork类来把大任务分割成子任务，有可能子任务还是很大，所以还需要不停地分割，直到分割出的子任务足够小。

​	步骤2　**执行任务并合并结果**。分割的子任务分别放在双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。

​	Fork/Join使用两个类来完成以上两件事情。

1. ForkJoinTask：我们要使用ForkJoin框架，必须首先创建一个ForkJoin任务。它提供在任务中执行fork()和join()操作的机制。通常情况下，我们不需要直接继承ForkJoinTask类，只需要继承它的子类，Fork/Join框架提供了以下两个子类。

   + RecursiveAction：用于没有返回结果的任务。

   + RecursiveTask：用于有返回结果的任务。

2. ForkJoinPool：ForkJoinTask需要通过ForkJoinPool来执行。

​	**任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。**

### 6.4.4 使用Fork/Join框架

​	让我们通过一个简单的需求来使用Fork/Join框架，需求是：计算1+2+3+4的结果。

​	使用Fork/Join框架首先要考虑到的是如何分割任务，如果希望每个子任务最多执行两个数的相加，那么我们设置分割的阈值是2，由于是4个数字相加，所以Fork/Join框架会把这个任务fork成两个子任务，子任务一负责计算1+2，子任务二负责计算3+4，然后再join两个子任务的结果。因为是有结果的任务，所以必须继承RecursiveTask，实现代码如下。

```java
package fj;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Future;
import java.util.concurrent.RecursiveTask;
public class CountTask extends RecursiveTask<Integer> {
  private static final int THRESHOLD = 2;　　// 阈值
  private int start;
  private int end;
  public CountTask(int start, int end) {
    this.start = start;
    this.end = end;
  }
  @Override
  protected Integer compute() {
    int sum = 0;
    // 如果任务足够小就计算任务
    boolean canCompute = (end - start) <= THRESHOLD;
    if (canCompute) {
      for (int i = start; i <= end; i++) {
        sum += i;
      }
    } else {
      // 如果任务大于阈值，就分裂成两个子任务计算
      int middle = (start + end) / 2;
      CountTask leftTask = new CountTask(start, middle);
      CountTask rightTask = new CountTask(middle + 1, end);
      // 执行子任务
      leftTask.fork();
      rightTask.fork();
      // 等待子任务执行完，并得到其结果
      int leftResult=leftTask.join();
      int rightResult=rightTask.join();
      // 合并子任务
      sum = leftResult + rightResult;
    }
    return sum;
  }
  public static void main(String[] args) {
    ForkJoinPool forkJoinPool = new ForkJoinPool();
    // 生成一个计算任务，负责计算1+2+3+4
    CountTask task = new CountTask(1, 4);
    // 执行一个任务
    Future<Integer> result = forkJoinPool.submit(task);
    try {
      System.out.println(result.get());
    } catch (InterruptedException e) {
    } catch (ExecutionException e) {
    }
  }
}
```

​	通过这个例子，我们进一步了解ForkJoinTask，**ForkJoinTask与一般任务的主要区别在于它需要实现compute方法，在这个方法里，首先需要判断任务是否足够小，如果足够小就直接执行任务。如果不足够小，就必须分割成两个子任务，每个子任务在调用fork方法时，又会进入compute方法，看看当前子任务是否需要继续分割成子任务，如果不需要继续分割，则执行当前子任务并返回结果。使用join方法会等待子任务执行完并得到其结果。**

### 6.4.5 Fork/Join框架的异常处理

​	ForkJoinTask在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以ForkJoinTask提供了`isCompletedAbnormally()`方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过ForkJoinTask的getException方法获取异常。使用如下代码。

```java
if(task.isCompletedAbnormally())
{
  System.out.println(task.getException());
}
```

​	getException方法返回Throwable对象，如果任务被取消了则返回CancellationException。如果任务没有完成或者没有抛出异常则返回null。

### 6.4.6 Fork/Join框架的实现原理

​	**ForkJoinPool由ForkJoinTask数组和ForkJoinWorkerThread数组组成，ForkJoinTask数组负责存放程序提交给ForkJoinPool的任务，而ForkJoinWorkerThread数组负责执行这些任务。**

#### 1. ForkJoinTask的fork方法实现原理

​	当我们调用ForkJoinTask的fork方法时，程序会调用ForkJoinWorkerThread的pushTask方法**异步地执行这个任务，然后立即返回结果**。代码如下。

```java
public final ForkJoinTask<V> fork() {
  ((ForkJoinWorkerThread) Thread.currentThread())
  .pushTask(this);
  return this;
}
```

​	pushTask方法把当前任务存放在ForkJoinTask数组队列里。然后再调用ForkJoinPool的signalWork()方法**唤醒或创建一个工作线程来执行任务**。代码如下。

```java
final void pushTask(ForkJoinTask<> t) {
  ForkJoinTask<>[] q; 
  int s, m;
  if ((q = queue) != null) {　　　　// ignore if queue removed
    long u = (((s = queueTop) & (m = q.length - 1)) << ASHIFT) + ABASE;
    UNSAFE.putOrderedObject(q, u, t);
    queueTop = s + 1;　　　　　　// or use putOrderedInt
    if ((s -= queueBase) <= 2)
      pool.signalWork();
    else if (s == m)
      growQueue();
  }
}
```

#### 2. ForkJoinTask的join方法实现原理

​	Join方法的主要作用是**阻塞当前线程并等待获取结果**。让我们一起看看ForkJoinTask的join方法的实现，代码如下。

```java
public final V join() {
  if (doJoin() != NORMAL)
    return reportResult();
  else
    return getRawResult();
} private V
  reportResult() {
  int s;
  Throwable ex;
  if ((s = status) == CANCELLED)
    throw new CancellationException();
  if (s == EXCEPTIONAL && (ex = getThrowableException()) != null)
    UNSAFE.throwException(ex);
  return getRawResult();
}
```

​	首先，它调用了doJoin()方法，通过doJoin()方法得到当前任务的状态来判断返回什么结果，任**务状态有4种：已完成（NORMAL）、被取消（CANCELLED）、信号（SIGNAL）和出现异常（EXCEPTIONAL）**。

+ 如果任务状态是已完成，则直接返回任务结果。
+ 如果任务状态是被取消，则直接抛出CancellationException。
+ 如果任务状态是抛出异常，则直接抛出对应的异常。

​	让我们再来分析一下doJoin()方法的实现代码。

```java
private int doJoin() {
  Thread t; 
  ForkJoinWorkerThread w;
  int s;
  boolean completed;
  if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) {
    if ((s = status) < 0)
      return s;
    if ((w = (ForkJoinWorkerThread)t).unpushTask(this)) {
      try {
        completed = exec();
      } catch (Throwable rex) {
        return setExceptionalCompletion(rex);
      }
      if (completed)
        return setCompletion(NORMAL);
    }
    return w.joinTask(this);
  }
  else
    return externalAwaitDone();
}
```

​	<u>在doJoin()方法里，首先通过查看任务的状态，看任务是否已经执行完成，如果执行完成，则直接返回任务状态；如果没有执行完，则从任务数组里取出任务并执行。如果任务顺利执行完成，则设置任务状态为NORMAL，如果出现异常，则记录异常，并将任务状态设置为EXCEPTIONAL</u>。

## 6.5 本章小结

​	本章介绍了Java中提供的各种并发容器和框架，并分析了该容器和框架的实现原理，从中我们能够领略到大师级的设计思路，希望读者能够充分理解这种设计思想，并在以后开发的并发程序时，运用上这些并发编程的技巧。

# 第7章 Java中的13个原子操作类

​	当程序更新一个变量时，如果多线程同时更新这个变量，可能得到期望之外的值，比如变量i=1，A线程更新i+1，B线程也更新i+1，经过两个线程操作之后可能i不等于3，而是等于2。因为A和B线程在更新变量i的时候拿到的i都是1，这就是线程不安全的更新操作，通常我们会使用synchronized来解决这个问题，synchronized会保证多线程不会同时更新变量i。

​	而Java从JDK 1.5开始提供了`java.util.concurrent.atomic`包（以下简称Atomic包），这个包中的原子操作类提供了一种用法简单、性能高效、线程安全地更新一个变量的方式。

​	因为变量的类型有很多种，所以在Atomic包里一共提供了13个类，属于4种类型的原子更新方式，分别是：

+ 原子更新基本类型
+ 原子更新数组
+ 原子更新引用
+ 原子更新属性（字段）

​	**Atomic包里的类基本都是使用Unsafe实现的包装类**。

## 7.1 原子更新基本类型类

​	使用原子的方式更新基本类型，Atomic包提供了以下3个类。

+ AtomicBoolean：原子更新布尔类型。

+ AtomicInteger：原子更新整型。

+ AtomicLong：原子更新长整型。

​	以上3个类提供的方法几乎一模一样，所以本节仅以AtomicInteger为例进行讲解，AtomicInteger的常用方法如下。

+ int addAndGet（int delta）：以原子方式将输入的数值与实例中的值（AtomicInteger里的value）相加，并返回结果。

+ boolean compareAndSet（int expect，int update）：如果输入的数值等于预期值，则以原子方式将该值设置为输入的值。

+ int getAndIncrement()：以原子方式将当前值加1，注意，**这里返回的是自增前的值**。

+ void lazySet（int newValue）：最终会设置成newValue，使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。关于该方法的更多信息可以参考并发编程网翻译的一篇文章《AtomicLong.lazySet是如何工作的？》，[~~文章地址-已失效~~](http://ifeve.com/howdoes-atomiclong-lazyset-work/)。

+ int getAndSet（int newValue）：以原子方式设置为newValue的值，并返回旧值。

AtomicInteger示例代码如（代码清单7-1 AtomicIntegerTest.java）所示。

```java
import java.util.concurrent.atomic.AtomicInteger;
public class AtomicIntegerTest {
  static AtomicInteger ai = new AtomicInteger(1);
  public static void main(String[] args) {
    System.out.println(ai.getAndIncrement());
    System.out.println(ai.get());
  }
}
```

输出结果如下。

```none
1
2
```

​	那么getAndIncrement是如何实现原子操作的呢？让我们一起分析其实现原理，getAndIncrement的源码如（代码清单7-2 AtomicInteger.java）所示。

```java
public final int getAndIncrement() {
  for (;;) {
    int current = get();
    int next = current + 1;
    if (compareAndSet(current, next))
      return current;
  }
}
public final boolean compareAndSet(int expect, int update) {
  return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

​	源码中for循环体的第一步先取得AtomicInteger里存储的数值，第二步对AtomicInteger的当前数值进行加1操作，关键的第三步调用compareAndSet方法来进行原子更新操作，该方法先检查当前数值是否等于current，等于意味着AtomicInteger的值没有被其他线程修改过，则将AtomicInteger的当前数值更新成next的值，如果不等compareAndSet方法会返回false，程序会进入for循环重新进行compareAndSet操作。

​	Atomic包提供了3种基本类型的原子更新，但是Java的基本类型里还有char、float和double等。那么问题来了，如何原子的更新其他的基本类型呢？Atomic包里的类基本都是使用Unsafe实现的，让我们一起看一下Unsafe的源码，如（代码清单7-3 Unsafe.java）所示。

```java
/**
* 如果当前数值是expected，则原子的将Java变量更新成x
* @return 如果更新成功则返回true
*/
public final native boolean compareAndSwapObject(Object o,long offset,Object expected,Object x);
public final native boolean compareAndSwapInt(Object o, long offset,int expected,int x);
public final native boolean compareAndSwapLong(Object o, long offset,long expected,long x);
```

​	通过代码，我们发现**Unsafe只提供了3种CAS方法：compareAndSwapObject、compare-AndSwapInt和compareAndSwapLong，再看AtomicBoolean源码，发现它是先把Boolean转换成整型，再使用compareAndSwapInt进行CAS，所以原子更新char、float和double变量也可以用类似的思路来实现。**

## 7.2 原子更新数组

​	通过原子的方式更新数组里的某个元素，Atomic包提供了以下4个类。

+ AtomicIntegerArray：原子更新整型数组里的元素。

+ AtomicLongArray：原子更新长整型数组里的元素。

+ AtomicReferenceArray：原子更新引用类型数组里的元素。

+ AtomicIntegerArray类主要是提供原子的方式更新数组里的整型，其常用方法如下。
  + int addAndGet（int i，int delta）：以原子方式将输入值与数组中索引i的元素相加。
  + boolean compareAndSet（int i，int expect，int update）：如果当前值等于预期值，则以原子方式将数组位置i的元素设置成update值。

​	以上几个类提供的方法几乎一样，所以本节仅以AtomicIntegerArray为例进行讲解，AtomicIntegerArray的使用实例代码如（代码清单7-4 AtomicIntegerArrayTest.java）所示。

```java
public class AtomicIntegerArrayTest {
  static int[] value = new int[] { 1， 2 };
  static AtomicIntegerArray ai = new AtomicIntegerArray(value);
  public static void main(String[] args) {
    ai.getAndSet(0， 3);
    System.out.println(ai.get(0));
    System.out.println(value[0]);
  }
}
```

以下是输出的结果。

```none
3
1
```

​	**需要注意的是，数组value通过构造方法传递进去，然后AtomicIntegerArray会将当前数组复制一份，所以当AtomicIntegerArray对内部的数组元素进行修改时，不会影响传入的数组**。

## 7.3 原子更新引用类型

​	原子更新基本类型的AtomicInteger，只能更新一个变量，如果要原子更新多个变量，就需要使用这个原子更新引用类型提供的类。Atomic包提供了以下3个类。

+ AtomicReference：原子更新引用类型。
+ AtomicReferenceFieldUpdater：原子更新引用类型里的字段。
+ **AtomicMarkableReference：原子更新带有标记位的引用类型**。可以原子更新一个布尔类型的标记位和引用类型。构造方法是AtomicMarkableReference（V initialRef，boolean initialMark）。

​	以上几个类提供的方法几乎一样，所以本节仅以AtomicReference为例进行讲解，AtomicReference的使用示例代码如（代码清单7-5 AtomicReferenceTest.java）所示。

```java
public class AtomicReferenceTest {
  public static AtomicReference<user> atomicUserRef = new
    AtomicReference<user>();
  public static void main(String[] args) {
    User user = new User("conan"， 15);
    atomicUserRef.set(user);
    User updateUser = new User("Shinichi"， 17);
    atomicUserRef.compareAndSet(user， updateUser);
    System.out.println(atomicUserRef.get().getName());
    System.out.println(atomicUserRef.get().getOld());
  }
  static class User {
    private String name;
    private int old;
    public User(String name， int old) {
      this.name = name;
      this.old = old;
    }
    public String getName() {
      return name;
    }
    public int getOld() {
      return old;
    }
  }
}
```

​	代码中首先构建一个user对象，然后把user对象设置进AtomicReferenc中，最后调用compareAndSet方法进行原子更新操作，实现原理同AtomicInteger里的compareAndSet方法。代码执行后输出结果如下。

```none
Shinichi
17
```

## 7.4 原子更新字段类

​	如果需**原子地更新某个类里的某个字段时，就需要使用原子更新字段类**，Atomic包提供了以下3个类进行原子字段更新。

+ AtomicIntegerFieldUpdater：原子更新整型的字段的更新器。

+ AtomicLongFieldUpdater：原子更新长整型字段的更新器。

+ **AtomicStampedReference：原子更新带有版本号的引用类型**。该类将整数值与引用关联起来，可用于原子的更新数据和数据的版本号，**可以解决使用CAS进行原子更新时可能出现的ABA问题**。

​	要想原子地更新字段类需要两步。

1. 第一步，因为原子更新字段类都是抽象类，每次使用的时候必须使用静态方法newUpdater()创建一个**更新器**，并且需要设置想要更新的类和属性。

2. 第二步，**更新类的字段（属性）必须使用public volatile修饰符**。

​	以上3个类提供的方法几乎一样，所以本节仅以AstomicIntegerFieldUpdater为例进行讲解，AstomicIntegerFieldUpdater的示例代码如（代码清单7-6 AtomicIntegerFieldUpdaterTest.java）所示。

```java
public class AtomicIntegerFieldUpdaterTest {
  // 创建原子更新器，并设置需要更新的对象类和对象的属性
  private static AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater.
    newUpdater(User.class， "old");
  public static void main(String[] args) {
    // 设置柯南的年龄是10岁
    User conan = new User("conan"， 10);
    // 柯南长了一岁，但是仍然会输出旧的年龄
    System.out.println(a.getAndIncrement(conan));
    // 输出柯南现在的年龄
    System.out.println(a.get(conan));
  }
  public static class User {
    private String name;
    public volatile int old;
    public User(String name， int old) {
      this.name = name;
      this.old = old;
    }
    public String getName() {
      return name;
    }
    public int getOld() {
      return old;
    }
  }
}
```

代码执行后输出如下。

```java
10
11
```

## 7.5 本章小结

​	本章介绍了JDK中并发包里的13个原子操作类以及原子操作类的实现原理，读者需要熟悉这些类和使用场景，在适当的场合下使用它。

# 第8章 Java中的并发工具类

​	在JDK的并发包里提供了几个非常有用的并发工具类。<u>**CountDownLatch**、**CyclicBarrier**和**Semaphore**工具类提供了一种并发流程控制的手段</u>，<u>**Exchanger工具类**则提供了在线程间交换数据的一种手段</u>。本章会配合一些应用场景来介绍如何使用这些工具类。

## 8.1 等待多线程完成的CountDownLatch

​	假如有这样一个需求：我们需要解析一个Excel里多个sheet的数据，此时可以考虑使用多线程，每个线程解析一个sheet里的数据，等到所有的sheet都解析完之后，程序需要提示解析完成。在这个需求中，要实现主线程等待所有线程完成sheet的解析操作，最简单的做法是使用join()方法，如（代码清单8-1 JoinCountDownLatchTest.java）所示。

```java
public class JoinCountDownLatchTest {
  public static void main(String[] args) throws InterruptedException {
    Thread parser1 = new Thread(new Runnable() {
      @Override
      public void run() {
      }
    });
    Thread parser2 = new Thread(new Runnable() {
      @Override
      public void run() {
        System.out.println("parser2 finish");
      }
    });
    parser1.start();
    parser2.start();
    parser1.join();
    parser2.join();
    System.out.println("all parser finish");
  }
}
```

​	join用于让当前执行线程等待join线程执行结束。**其实现原理是不停检查join线程是否存活，如果join线程存活则让当前线程永远等待**。其中，wait（0）表示永远等待下去，代码片段如下。

```java
while (isAlive()) {
  wait(0);
}
```

​	直到join线程中止后，线程的this.notifyAll()方法会被调用，调用notifyAll()方法是在JVM里实现的，所以在JDK里看不到，大家可以查看JVM源码。

​	在JDK 1.5之后的并发包中提供的CountDownLatch也可以实现join的功能，并且比join的功能更多，如（代码清单8-2 CountDownLatchTest.java）所示。

```java
public class CountDownLatchTest {
  staticCountDownLatch c = new CountDownLatch(2);
  public static void main(String[] args) throws InterruptedException {
    new Thread(new Runnable() {
      @Override
      public void run() {
        System.out.println(1);
        c.countDown();
        System.out.println(2);
        c.countDown();
      }
    }).start();
    c.await();
    System.out.println("3");
  }
}
```

​	CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完成，这里就传入N。

​	当我们调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个CountDownLatch的引用传递到线程里即可。

​	如果有某个解析sheet的线程处理得比较慢，我们不可能让主线程一直等待，所以可以使用另外一个带指定时间的await方法——await（long time，TimeUnit unit），这个方法等待特定时间后，就会不再阻塞当前线程。join也有类似的方法。

​	**注意：计数器必须大于等于0，只是等于0时候，计数器就是零，调用await方法时不会阻塞当前线程。CountDownLatch不可能重新初始化或者修改CountDownLatch对象的内部计数器的值。一个线程调用countDown方法happen-before，另外一个线程调用await方法。**

## 8.2 同步屏障CyclicBarrier

​	<u>CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行</u>。

### 8.2.1 CyclicBarrier简介

​	CyclicBarrier默认的构造方法是CyclicBarrier（int parties），其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。示例代码如（代码清单8-3 CyclicBarrierTest.java）所示。

```java
public class CyclicBarrierTest {
  staticCyclicBarrier c = new CyclicBarrier(2);
  public static void main(String[] args) {
    new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          c.await();
        } catch (Exception e) {
        }
        System.out.println(1);
      }
    }).start();
    try {
      c.await();
    } catch (Exception e) {
    }
    System.out.println(2);
  }
}
```

​	因为主线程和子线程的调度是由CPU决定的，两个线程都有可能先执行，所以会产生两种输出，第一种可能输出如下。

```none
1
2
```

第二种可能输出如下。

```none
2
1
```

​	如果把new CyclicBarrier(2)修改成new CyclicBarrier(3)，则主线程和子线程会永远等待，因为没有第三个线程执行await方法，即没有第三个线程到达屏障，所以之前到达屏障的两个线程都不会继续执行。

​	<u>CyclicBarrier还提供一个更高级的构造函数CyclicBarrier（int parties，Runnable barrier-Action），用于**在线程到达屏障时，优先执行barrierAction**，方便处理更复杂的业务场景</u>，如（代码清单8-4 CyclicBarrierTest2.java）所示。

```java
import java.util.concurrent.CyclicBarrier;
public class CyclicBarrierTest2 {
  static CyclicBarrier c = new CyclicBarrier(2, new A());
  public static void main(String[] args) {
    new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          c.await();
        } catch (Exception e) {
        }
        System.out.println(1);
      }
    }).start();
    try {
      c.await();
    } catch (Exception e) {
    }
    System.out.println(2);
  }
  static class A implements Runnable {
    @Override
    public void run() {
      System.out.println(3);
    }
  }
}
```

​	因为CyclicBarrier设置了拦截线程的数量是2，所以必须等代码中的第一个线程和线程A都执行完之后，才会继续执行主线程，然后输出2，所以代码执行后的输出如下。

```none
3
1
2
```

### 8.2.2 CyclicBarrier的应用场景

​	CyclicBarrier可以用于多线程计算数据，最后合并计算结果的场景。例如，用一个Excel保存了用户所有银行流水，每个Sheet保存一个账户近一年的每笔银行流水，现在需要统计用户的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水，如（代码清单8-5 BankWaterService.java）所示。

```java
import java.util.Map.Entry;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;
/**
* 银行流水处理服务类
*
* @authorftf
*
*/
publicclass BankWaterService implements Runnable {
  /**
* 创建4个屏障，处理完之后执行当前类的run方法
*/
  private CyclicBarrier c = new CyclicBarrier(4, this);
  /**
* 假设只有4个sheet，所以只启动4个线程
*/
  private Executor executor = Executors.newFixedThreadPool(4);
  /**
* 保存每个sheet计算出的银流结果
*/
  private ConcurrentHashMap<String, Integer>sheetBankWaterCount = new
    ConcurrentHashMap<String, Integer>();
  privatevoid count() {
    for (inti = 0; i< 4; i++) {
      executor.execute(new Runnable() {
        @Override
        publicvoid run() {
          // 计算当前sheet的银流数据，计算代码省略
          sheetBankWaterCount
            .put(Thread.currentThread().getName(), 1);
          // 银流计算完成，插入一个屏障
          try {
            c.await();
          } catch (InterruptedException |
                   BrokenBarrierException e) {
            e.printStackTrace();
          }
        }
      });
    }
  }
  @Override
  publicvoid run() {
    intresult = 0;
    // 汇总每个sheet计算出的结果
    for (Entry<String, Integer>sheet : sheetBankWaterCount.entrySet()) {
      result += sheet.getValue();
    }
    // 将结果输出
    sheetBankWaterCount.put("result", result);
    System.out.println(result);
  }
  public static void main(String[] args) {
    BankWaterService bankWaterCount = new BankWaterService();
    bankWaterCount.count();
  }
}
```

​	使用线程池创建4个线程，分别计算每个sheet里的数据，每个sheet计算结果是1，再由BankWaterService线程汇总4个sheet计算出的结果，输出结果如下。

```none
4
```

### 8.2.3 CyclicBarrier和CountDownLatch的区别

​	**CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置**。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次。

​	CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得Cyclic-Barrier阻塞的线程数量。**isBroken()方法用来了解阻塞的线程是否被中断**。（代码清单8-5 BankWaterService.java）执行完之后会返回true，其中isBroken的使用代码如（代码清单8-6 CyclicBarrierTest3.java）所示。

```java
importjava.util.concurrent.BrokenBarrierException;
importjava.util.concurrent.CyclicBarrier;
public class CyclicBarrierTest3 {
  staticCyclicBarrier c = new CyclicBarrier(2);
  public static void main(String[] args) throws InterruptedException，
    BrokenBarrierException {
    Thread thread = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          c.await();
        } catch (Exception e) {
        }
      }
    });
    thread.start();
    thread.interrupt();
    try {
      c.await();
    } catch (Exception e) {
      System.out.println(c.isBroken());
    }
  }
}
```

输出如下所示。

```none
true
```

## 8.3 控制并发线程数的Semaphore

​	**Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源**。

​	多年以来，我（书籍作者）都觉得从字面上很难理解Semaphore所表达的含义，只能把它比作是控制流量的红绿灯。比如××马路要限制流量，只允许同时有一百辆车在这条路上行使，其他的都必须在路口等待，所以前一百辆车会看到绿灯，可以开进这条马路，后面的车会看到红灯，不能驶入××马路，但是如果前一百辆中有5辆车已经离开了××马路，那么后面就允许有5辆车驶入马路，这个例子里说的车就是线程，驶入马路就表示线程在执行，离开马路就表示线程执行完成，看见红灯就表示线程被阻塞，不能执行。

### 1. 应用场景

​	**Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接**。假如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。这个时候，就可以使用Semaphore来做流量控制，如（代码清单8-7 SemaphoreTest.java）所示。

```java
public class SemaphoreTest {
  private static final int THREAD_COUNT = 30;
  private static ExecutorServicethreadPool = Executors
    .newFixedThreadPool(THREAD_COUNT);
  private static Semaphore s = new Semaphore(10);
  public static void main(String[] args) {
    for (inti = 0; i< THREAD_COUNT; i++) {
      threadPool.execute(new Runnable() {
        @Override
        public void run() {
          try {
            s.acquire();
            System.out.println("save data");
            s.release();
          } catch (InterruptedException e) {
          }
        }
      });
    }
    threadPool.shutdown();
  }
}
```

​	在代码中，虽然有30个线程在执行，但是只允许10个并发执行。Semaphore的构造方法Semaphore（int permits）接受一个整型的数字，表示可用的许可证数量。Semaphore（10）表示允许10个线程获取许可证，也就是最大并发数是10。Semaphore的用法也很简单，首先线程使用Semaphore的`acquire()`方法获取一个许可证，使用完之后调用`release()`方法归还许可证。还可以用`tryAcquire()`方法尝试获取许可证。

### 2. 其他方法

Semaphore还提供一些其他方法，具体如下。

+ `int availablePermits()`：返回此信号量中当前可用的许可证数。

+ `int getQueueLength()`：返回正在等待获取许可证的线程数。

+ `boolean hasQueuedThreads()`：是否有线程正在等待获取许可证。

+ `void reducePermits（int reduction）`：减少reduction个许可证，是个protected方法。

+ `Collection getQueuedThreads()`：返回所有等待获取许可证的线程集合，是个protected方法。

## 8.4 线程间交换数据的Exchanger

​	**Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交换**。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过exchange方法交换数据，如果第一个线程先执行`exchange()`方法，它会一直等待第二个线程也执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产出来的数据传递给对方。

​	下面来看一下Exchanger的应用场景。

​	**Exchanger可以用于遗传算法**，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。**Exchanger也可以用于校对工作**，比如我们需要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行录入，录入到Excel之后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看看是否录入一致，代码如（代码清单8-8 ExchangerTest.java）所示。

```java
public class ExchangerTest {
  private static final Exchanger<String>exgr = new Exchanger<String>();
  private static ExecutorServicethreadPool = Executors.newFixedThreadPool(2);
  public static void main(String[] args) {
    threadPool.execute(new Runnable() {
      @Override
      public void run() {
        try {
          String A = "银行流水A";　　　　// A录入银行流水数据
          exgr.exchange(A);
        } catch (InterruptedException e) {
        }
      }
    });
    threadPool.execute(new Runnable() {
      @Override
      public void run() {
        try {
          String B = "银行流水B";　　　　// B录入银行流水数据
          String A = exgr.exchange("B");
          System.out.println("A和B数据是否一致：" + A.equals(B) + "，A录入的是："
                             + A + "，B录入是：" + B);
        } catch (InterruptedException e) {
        }
      }
    });
    threadPool.shutdown();
  }
}
```

​	<u>如果两个线程有一个没有执行exchange()方法，则会一直等待，如果担心有特殊情况发生，避免一直等待，可以使用exchange（V x，longtimeout，TimeUnit unit）设置最大等待时长</u>。

## 8.5 本章小结

​	本章配合一些应用场景介绍JDK中提供的几个并发工具类，大家记住这个工具类的用途，一旦有对应的业务场景，不妨试试这些工具类。

# 第9章 Java中的线程池

​	Java中的线程池是运用场景最多的并发框架，几乎所有需要异步或并发执行任务的程序都可以使用线程池。在开发过程中，合理地使用线程池能够带来3个好处。

+ 第一：**降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
+ 第二：**提高响应速度**。当任务到达时，任务可以不需要等到线程创建就能立即执行。
+ 第三：**提高线程的可管理性**。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。但是，要做到合理利用线程池，必须对其实现原理了如指掌。

## 9.1 线程池的实现

> [十二、Java中的线程池](https://www.jianshu.com/p/4df00eee82f6)

​	当向线程池提交一个任务之后，线程池是如何处理这个任务的呢？本节来看一下线程池的主要处理流程，处理流程图如下图所示。

​	从图中可以看出，当提交一个新任务到线程池时，线程池的处理流程如下。

![](https://upload-images.jianshu.io/upload_images/7378149-4656084f521a0113.png)

1. 线程池判断核心线程池里的线程是否都在执行任务。如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。

2. 线程池判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。

3. 线程池判断线程池的线程是否都处于工作状态。如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给饱和策略来处理这个任务。

​	ThreadPoolExecutor执行execute()方法的示意图，如下图所示。

![](https://upload-images.jianshu.io/upload_images/7378149-c8b3a07b3e404414.png)

​	ThreadPoolExecutor执行execute方法分下面4种情况。

1. **如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤需要获取全局锁）**。

2. 如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。

3. **如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（注意，执行这一步骤需要获取全局锁）**。

4. 如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用`RejectedExecutionHandler.rejectedExecution()`方法。

​	ThreadPoolExecutor采取上述步骤的总体设计思路，是为了在执行`execute()`方法时，尽可能地避免获取全局锁（那将会是一个严重的可伸缩瓶颈）。在ThreadPoolExecutor完成预热之后（当前运行的线程数大于等于corePoolSize），几乎所有的`execute()`方法调用都是执行步骤2，而步骤2不需要获取全局锁。

​	源码分析：上面的流程分析让我们很直观地了解了线程池的工作原理，让我们再通过源代码来看看是如何实现的，线程池执行任务的方法如下。

```java
public void execute(Runnable command) {
  if (command == null)
    throw new NullPointerException();
  // 如果线程数小于基本线程数，则创建线程并执行当前任务
  if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
    // 如线程数大于等于基本线程数或线程创建失败，则将当前任务放到工作队列中。
    if (runState == RUNNING && workQueue.offer(command)) {
      if (runState != RUNNING || poolSize == 0)
        ensureQueuedTaskHandled(command);
    } // 如果线程池不处于运行中或任务无法放入队列，并且当前线程数量小于最大允许的线程数量,
    // 则创建一个线程执行任务。
    else if (!addIfUnderMaximumPoolSize(command))
      // 抛出RejectedExecutionException异常
      reject(command); // is shutdown or saturated
  }
}
```

​	**工作线程：线程池创建线程时，会将线程封装成工作线程Worker，Worker在执行完任务后，还会循环获取工作队列里的任务来执行。我们可以从Worker类的run()方法里看到这点。**

```java
public void run() {
  try {
    Runnable task = firstTask;
    firstTask = null;
    while (task != null || (task = getTask()) != null) {
      runTask(task);
      task = null;
    }
  } finally {
    workerDone(this);
  }
}
```

​	ThreadPoolExecutor中线程执行任务的示意图如下图所示。

![](https://upload-images.jianshu.io/upload_images/7378149-845e5b2c4edbc279.png)

​	线程池中的线程执行任务分两种情况，如下。

1. 在execute()方法中创建一个线程时，会让这个线程执行当前任务。

2. **这个线程执行完上图中1的任务后，会反复从BlockingQueue获取任务来执行**。

## 9.2 线程池的使用

### 9.2.1 线程池的创建

​	我们可以通过ThreadPoolExecutor来创建一个线程池。

```java
new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime,
                       milliseconds,runnableTaskQueue, handler);
```

​	创建一个线程池时需要输入几个参数，如下。

1. corePoolSize（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。<u>如果调用了线程池的`prestartAllCoreThreads()`方法，线程池会提前创建并启动所有基本线程</u>。

2. runnableTaskQueue（任务队列）：用于保存等待执行的任务的阻塞队列。可以选择以下几个阻塞队列。

   + ArrayBlockingQueue：是一个基于数组结构的**有界**阻塞队列，此队列按FIFO（先进先出）原则对元素进行排序。
   + LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO排序元素，**吞吐量通常要高于ArrayBlockingQueue**。静态工厂方法`Executors.newFixedThreadPool()`使用了这个队列。
   + SynchronousQueue：一个不存储元素的阻塞队列。**每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue**，静态工厂方法`Executors.newCachedThreadPool`使用了这个队列。
   + PriorityBlockingQueue：一个具有优先级的**无限**阻塞队列。

3. maximumPoolSize（线程池最大数量）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。**值得注意的是，如果使用了无界的任务队列这个参数就没什么效果**。

4. ThreadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。**使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线程设置有意义的名字**，代码如下。

   ```java
   new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build();
   ```

5. RejectedExecutionHandler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。**这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常**。在JDK 1.5中Java线程池框架提供了以下4种策略。

   + AbortPolicy：直接抛出异常。
   + CallerRunsPolicy：只用调用者所在线程来运行任务。
   + DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
   + DiscardPolicy：不处理，丢弃掉。

   当然，也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化存储不能处理的任务。

   + **keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以，如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率**。
   + TimeUnit（线程活动保持时间的单位）：可选的单位有天（DAYS）、小时（HOURS）、分钟（MINUTES）、毫秒（MILLISECONDS）、微秒（MICROSECONDS，千分之一毫秒）和纳秒（NANOSECONDS，千分之一微秒）。

### 9.2.2 向线程池提交任务

​	可以使用两个方法向线程池提交任务，分别为`execute()`和`submit()`方法。

​	**`execute()`方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功**。通过以下代码可知`execute()`方法输入的任务是一个Runnable类的实例。

```java
threadsPool.execute(new Runnable() {
  @Override
  public void run() {
    // TODO Auto-generated method stub
  }
});
```

​	**submit()方法用于提交需要返回值的任务**。线程池会返回一个future类型的对象，通过这个future对象可以判断任务是否执行成功，并且可以通过future的get()方法来获取返回值，**get()方法会阻塞当前线程直到任务完成，而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完**。

```java
Future<Object> future = executor.submit(harReturnValuetask);
try {
  Object s = future.get();
} catch (InterruptedException e) {
  // 处理中断异常
} catch (ExecutionException e) {
  // 处理无法执行任务异常
} finally {
  // 关闭线程池
  executor.shutdown();
}
```

### 9.2.3 关闭线程池

​	**可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的原理是遍历线程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务可能永远无法终止**。但是它们存在一定的区别。

+ shutdownNow首先将线程池的状态设置成**STOP**，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表
+ shutdown只是将线程池的状态设置成**SHUTDOWN**状态，然后中断所有没有正在执行任务的线程。

​	**只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务都已关闭后，才表示线程池关闭成功，这时调用isTerminated方法会返回true**。至于应该调用哪一种方法来关闭线程池，应该由提交到线程池的任务特性决定，通常调用shutdown方法来关闭线程池，如果任务不一定要执行完，则可以调用shutdownNow方法。

### 9.2.4 合理地配置线程池

​	要想合理地配置线程池，就必须首先分析任务特性，可以从以下几个角度来分析。

+ **任务的性质：CPU密集型任务、IO密集型任务和混合型任务**。

+ 任务的优先级：高、中和低。

+ 任务的执行时间：长、中和短。

+ **任务的依赖性：是否依赖其他系统资源，如数据库连接**。

​	性质不同的任务可以用不同规模的线程池分开处理。

+ **CPU密集型任务应配置尽可能小的线程，如配置Ncpu+1个线程的线程池。**
+ **由于IO密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程，如2\*Ncpu。**
+ 混合型的任务，如果可以拆分，将其拆分成一个CPU密集型任务和一个IO密集型任务，只要这两个任务执行的时间相差不是太大，那么分解后执行的吞吐量将高于串行执行的吞吐量。如果这两个任务执行时间相差太大，则没必要进行分解。

​	**可以通过`Runtime.getRuntime().availableProcessors()`方法获得当前设备的CPU个数。**

​	优先级不同的任务可以使用优先级队列PriorityBlockingQueue来处理。它可以让优先级高的任务先执行。

​	**注意：如果一直有优先级高的任务提交到队列里，那么优先级低的任务可能永远不能执行**。

​	执行时间不同的任务可以交给不同规模的线程池来处理，或者可以使用优先级队列，让执行时间短的任务先执行。

​	**依赖数据库连接池的任务，因为线程提交SQL后需要等待数据库返回结果，等待的时间越长，则CPU空闲时间就越长，那么线程数应该设置得越大，这样才能更好地利用CPU**。

​	**建议使用有界队列**。有界队列能增加系统的稳定性和预警能力，可以根据需要设大一点儿，比如几千。有一次，我们系统里后台任务线程池的队列和线程池全满了，不断抛出抛弃任务的异常，通过排查发现是数据库出现了问题，导致执行SQL变得非常缓慢，因为后台任务线程池里的任务全是需要向数据库查询和插入数据的，所以导致线程池里的工作线程全部阻塞，任务积压在线程池里。如果当时我们设置成无界队列，那么线程池的队列就会越来越多，有可能会撑满内存，导致整个系统不可用，而不只是后台任务出现问题。当然，我们的系统所有的任务是用单独的服务器部署的，我们使用不同规模的线程池完成不同类型的任务，但是出现这样问题时也会影响到其他任务。

### 9.2.5 线程池的监控

​	如果在系统中大量使用线程池，则有必要对线程池进行监控，方便在出现问题时，可以根据线程池的使用状况快速定位问题。可以通过线程池提供的参数进行监控，在监控线程池的时候可以使用以下属性。

+ taskCount：线程池需要执行的任务数量。
+ completedTaskCount：线程池在运行过程中已完成的任务数量，小于或等于taskCount。
+ **largestPoolSize：线程池里曾经创建过的最大线程数量。通过这个数据可以知道线程池是否曾经满过。如该数值等于线程池的最大大小，则表示线程池曾经满过。**
+ **getPoolSize：线程池的线程数量。如果线程池不销毁的话，线程池里的线程不会自动销毁，所以这个大小只增不减。**
+ getActiveCount：获取活动的线程数。

​	<u>通过扩展线程池进行监控。可以通过继承线程池来自定义线程池，重写线程池的beforeExecute、afterExecute和terminated方法，也可以在任务执行前、执行后和线程池关闭前执行一些代码来进行监控。例如，监控任务的平均执行时间、最大执行时间和最小执行时间等。这几个方法在线程池里是空方法。</u>

```java
protected void beforeExecute(Thread t, Runnable r) { }
```

> [解决Java线程池任务执行完毕后线程回收问题](https://www.cnblogs.com/pengineer/p/5011965.html)
>
> 工作线程回收需要满足三个条件：
>
> 1.  参数allowCoreThreadTimeOut为true
>
> 2. 该线程在keepAliveTime时间内获取不到任务，即空闲这么长时间
>
> 3. 当前线程池大小 > 核心线程池大小corePoolSize。

## 9.3 本章小结

​	在工作中我经常发现，很多人因为不了解线程池的实现原理，把线程池配置错误，从而导致了各种问题。本章介绍了为什么要使用线程池、如何使用线程池和线程池的使用原理，相信阅读完本章之后，读者能更准确、更有效地使用线程池。

# 第10章 Executor框架

​	在Java中，**使用线程来异步执行任务**。Java线程的创建与销毁需要一定的开销，如果我们为每一个任务创建一个新线程来执行，这些线程的创建与销毁将消耗大量的计算资源。同时，为每一个任务创建一个新线程来执行，这种策略可能会使处于高负荷状态的应用最终崩溃。**Java的线程既是工作单元，也是执行机制**。**从JDK 5开始，把工作单元与执行机制分离开来**。**工作单元包括Runnable和Callable，而执行机制由Executor框架提供**。

## 10.1 Executor框架简介

### 10.1.1 Executor框架的两级调度模型

> [Java线程池Executor框架详解](https://blog.51cto.com/13981400/2314726)

​	**在HotSpot VM的线程模型中，Java线程（java.lang.Thread）被一对一映射为本地操作系统线程**。<u>Java线程启动时会创建一个本地操作系统线程；当该Java线程终止时，这个操作系统线程也会被回收。操作系统会调度所有线程并将它们分配给可用的CPU</u>。

​	**在上层，Java多线程程序通常把应用分解为若干个任务，然后使用用户级的调度器（Executor框架）将这些任务映射为固定数量的线程；在底层，操作系统内核将这些线程映射到硬件处理器上**。这种两级调度模型的示意图如下图所示。

![Java线程池Executor框架详解](https://s4.51cto.com/images/blog/201811/08/a68351402070370e097e18f4b11241a7.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

​	从图中可以看出，应用程序通过Executor框架控制上层的调度；而下层的调度由操作系统内核控制，下层的调度不受应用程序的控制。

### 10.1.2 Executor框架的结构与成员

#### 1. Executor框架的结构

​	Executor框架主要由3大部分组成如下。

+ **任务**。包括被执行任务需要实现的接口：Runnable接口或Callable接口。

+ **任务的执行**。包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。Executor框架有两个关键类实现了ExecutorService接口（<u>ThreadPoolExecutor和ScheduledThreadPoolExecutor</u>）。

+ **异步计算的结果**。包括**接口Future和实现Future接口的FutureTask类**。

Executor框架包含的主要的类与接口如下图所示。（这个图书上画错了，左上角应该是Runnable）

![Java线程池Executor框架详解](https://s4.51cto.com/images/blog/201811/08/1957d47aa75c83916724a4a999a93759.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

下面是这些类和接口的简介。

+ Executor是一个接口，它是Executor框架的基础，它将任务的提交与任务的执行分离开来。

+ ThreadPoolExecutor是线程池的核心实现类，用来执行被提交的任务。

+ **ScheduledThreadPoolExecutor是一个实现类，可以在给定的延迟后运行命令，或者定期执行命令。ScheduledThreadPoolExecutor比Timer更灵活，功能更强大**。

+ **Future接口和实现Future接口的FutureTask类，代表异步计算的结果**。

+ Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。

Executor框架的使用示意图如下图所示。

![Java线程池Executor框架详解](https://s4.51cto.com/images/blog/201811/08/57a05da02b69cbdbee2d3e3acded72a4.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

​	主线程首先要创建实现Runnable或者Callable接口的任务对象。工具类Executors可以把一个Runnable对象封装为一个Callable对象（`Executors.callable（Runnable task）`或`Executors.callable（Runnable task，Object result）`）。

​	然后可以把Runnable对象直接交给ExecutorService执行（`ExecutorService.execute（Runnable command）`）；或者也可以把Runnable对象或Callable对象提交给ExecutorService执行（`Executor-Service.submit（Runnable task`）或`ExecutorService.submit（Callable<T> task）`）。

​	如果执行`ExecutorService.submit（…）`，ExecutorService将返回一个实现Future接口的对象（到目前为止的JDK中，返回的是**FutureTask对象**）。**由于FutureTask实现了Runnable，程序员也可以创建FutureTask，然后直接交给ExecutorService执行**。

​	最后，**主线程可以执行`FutureTask.get()`方法来等待任务执行完成。主线程也可以执行`FutureTask.cancel（boolean mayInterruptIfRunning）`来取消此任务的执行**。

#### 2. Executor框架的成员

​	本节将介绍Executor框架的主要成员：ThreadPoolExecutor、ScheduledThreadPoolExecutor、Future接口、Runnable接口、Callable接口和Executors。

##### 1. ThreadPoolExecutor

​	ThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建3种类型的ThreadPoolExecutor：SingleThreadExecutor、FixedThreadPool和CachedThreadPool。

​	下面分别介绍这3种ThreadPoolExecutor。

###### 1. FixedThreadPool

下面是Executors提供的，创建使用固定线程数的FixedThreadPool的API。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
  return new ThreadPoolExecutor(nThreads, nThreads,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>());
}

public static ExecutorService newFixedThreadPool(int nThreads) {
  return new ThreadPoolExecutor(nThreads, nThreads,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>());
}
```

**FixedThreadPool适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景，它适用于负载比较重的服务器。**

###### 2. SingleThreadExecutor

下面是Executors提供的，创建使用单个线程的SingleThreadExecutor的API。

```java
public static ExecutorService newSingleThreadExecutor() {
  return new FinalizableDelegatedExecutorService
    (new ThreadPoolExecutor(1, 1,
                            0L, TimeUnit.MILLISECONDS,
                            new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
  return new FinalizableDelegatedExecutorService
    (new ThreadPoolExecutor(1, 1,
                            0L, TimeUnit.MILLISECONDS,
                            new LinkedBlockingQueue<Runnable>(),
                            threadFactory));
}
```

**SingleThreadExecutor适用于需要保证顺序地执行各个任务；并且在任意时间点，不会有多个线程是活动的应用场景**。

###### 3. CachedThreadPool

下面是Executors提供的，创建一个会根据需要创建新线程的CachedThreadPool的API。

```java
public static ExecutorService newCachedThreadPool() {
  return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                60L, TimeUnit.SECONDS,
                                new SynchronousQueue<Runnable>());
}
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
  return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                60L, TimeUnit.SECONDS,
                                new SynchronousQueue<Runnable>(),
                                threadFactory);
}
```

**CachedThreadPool是大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器。**

##### 2. ScheduledThreadPoolExecutor

​	ScheduledThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建2种类型的ScheduledThreadPoolExecutor，如下。

+ ScheduledThreadPoolExecutor。包含若干个线程的ScheduledThreadPoolExecutor。

+ SingleThreadScheduledExecutor。只包含一个线程的ScheduledThreadPoolExecutor。

下面分别介绍这两种ScheduledThreadPoolExecutor。

下面是工厂类Executors提供的，创建固定个数线程的ScheduledThreadPoolExecutor的API。

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
  return new ScheduledThreadPoolExecutor(corePoolSize);
}
public static ScheduledExecutorService newScheduledThreadPool(
  int corePoolSize, ThreadFactory threadFactory) {
  return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```

​	**ScheduledThreadPoolExecutor适用于需要多个后台线程执行周期任务，同时为了满足资源管理的需求而需要限制后台线程的数量的应用场景**。

下面是Executors提供的，创建单个线程的SingleThreadScheduledExecutor的API。

```java
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
  return new DelegatedScheduledExecutorService
    (new ScheduledThreadPoolExecutor(1));
}
public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
  return new DelegatedScheduledExecutorService
    (new ScheduledThreadPoolExecutor(1, threadFactory));
}
```

​	**SingleThreadScheduledExecutor适用于需要单个后台线程执行周期任务，同时需要保证顺序地执行各个任务的应用场景。**

##### 3. Future接口

​	Future接口和实现Future接口的FutureTask类用来表示异步计算的结果。当我们把Runnable接口或Callable接口的实现类提交（submit）给ThreadPoolExecutor或ScheduledThreadPoolExecutor时，ThreadPoolExecutor或ScheduledThreadPoolExecutor会向我们返回一个FutureTask对象。下面是对应的API。

```java
<T> Future<T> submit(Callable<T> task)
<T> Future<T> submit(Runnable task, T result)
Future<> submit(Runnable task)
```

​	有一点需要读者注意，**到目前最新的JDK 8为止，Java通过上述API返回的是一个FutureTask对象。但从API可以看到，Java仅仅保证返回的是一个实现了Future接口的对象**。在将来的JDK实现中，返回的可能不一定是FutureTask。

##### 4. Runnable接口和Callable接口

​	Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或Scheduled-ThreadPoolExecutor执行。它们之间的区别是**Runnable不会返回结果，而Callable可以返回结果**。

​	除了可以自己创建实现Callable接口的对象外，还**可以使用工厂类Executors来把一个Runnable包装成一个Callable**。

下面是Executors提供的，把一个Runnable包装成一个Callable的API。

```java
public static Callable<Object> callable(Runnable task) // 假设返回对象Callable1
```

下面是Executors提供的，把一个Runnable和一个待返回的结果包装成一个Callable的API。

```java
public static <T> Callable<T> callable(Runnable task, T result) // 假设返回对象Callable2
```

​	前面讲过，当我们把一个Callable对象（比如上面的Callable1或Callable2）提交给ThreadPoolExecutor或ScheduledThreadPoolExecutor执行时，submit（…）会向我们返回一个FutureTask对象。我们可以执行FutureTask.get()方法来等待任务执行完成。当任务成功完成后FutureTask.get()将返回该任务的结果。例如，如果提交的是对象Callable1，FutureTask.get()方法将返回null；如果提交的是对象Callable2，FutureTask.get()方法将返回result对象。

## 10.2 ThreadPoolExecutor详解

​	Executor框架最核心的类是ThreadPoolExecutor，它是线程池的实现类，主要由下列4个组件构成。

+ corePool：核心线程池的大小。

+ maximumPool：最大线程池的大小。

+ BlockingQueue：用来暂时保存任务的工作队列。

+ RejectedExecutionHandler：当ThreadPoolExecutor已经关闭或ThreadPoolExecutor已经饱和时（达到了最大线程池大小且工作队列已满），execute()方法将要调用的Handler。

通过Executor框架的工具类Executors，可以创建3种类型的ThreadPoolExecutor。

+ FixedThreadPool。

+ SingleThreadExecutor。

+ CachedThreadPool。

下面将分别介绍这3种ThreadPoolExecutor。

### 10.2.1 FixedThreadPool详解

> [《Java并发编程的艺术》第十章——Executor框架](https://blog.csdn.net/qq_24982291/article/details/79163378)

​	**FixedThreadPool被称为可重用固定线程数的线程池**。下面是FixedThreadPool的源代码实现。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
  return new ThreadPoolExecutor(nThreads, nThreads,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>());
}
```

​	FixedThreadPool的corePoolSize和maximumPoolSize都被设置为创建FixedThreadPool时指定的参数nThreads。

​	**当线程池中的线程数大于corePoolSize时，keepAliveTime为多余的空闲线程等待新任务的最长时间，超过这个时间后多余的线程将被终止。这里把keepAliveTime设置为0L，意味着多余的空闲线程会被立即终止。**

​	FixedThreadPool的execute()方法的运行示意图如下图所示。

![这里写图片描述](https://img-blog.csdn.net/20180125152123448?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjQ5ODIyOTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

对上图的说明如下。

1. 如果当前运行的线程数少于corePoolSize，则创建新线程来执行任务。

2. **在线程池完成预热之后（当前运行的线程数等于corePoolSize），将任务加入LinkedBlockingQueue。**

3. **线程执行完1中的任务后，会在循环中反复从LinkedBlockingQueue获取任务来执行**。

FixedThreadPool使用**无界队列**LinkedBlockingQueue作为线程池的工作队列（队列的容量为Integer.MAX_VALUE）。使用无界队列作为工作队列会对线程池带来如下影响。

1. 当线程池中的线程数达到corePoolSize后，新任务将在无界队列中等待，因此线程池中的线程数不会超过corePoolSize。

2. 由于1，使用无界队列时maximumPoolSize将是一个无效参数。

3. 由于1和2，使用无界队列时keepAliveTime将是一个无效参数。

4. **由于使用无界队列，运行中的FixedThreadPool（未执行方法shutdown()或shutdownNow()）不会拒绝任务（不会调用RejectedExecutionHandler.rejectedExecution方法）**。

### 10.2.2 SingleThreadExecutor详解

​	**SingleThreadExecutor是使用单个worker线程的Executor**。下面是SingleThreadExecutor的源代码实现。

```java
public static ExecutorService newSingleThreadExecutor() {
  return new FinalizableDelegatedExecutorService
    (new ThreadPoolExecutor(1, 1,
                            0L, TimeUnit.MILLISECONDS,
                            new LinkedBlockingQueue<Runnable>()));
}
```

​	SingleThreadExecutor的corePoolSize和maximumPoolSize被设置为1。其他参数与FixedThreadPool相同。SingleThreadExecutor使用无界队列LinkedBlockingQueue作为线程池的工作队列（队列的容量为Integer.MAX_VALUE）。SingleThreadExecutor使用无界队列作为工作队列对线程池带来的影响与FixedThreadPool相同，这里就不赘述了。

​	SingleThreadExecutor的运行示意图如下图所示。

![这里写图片描述](https://img-blog.csdn.net/20180125152123448?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjQ5ODIyOTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	对上图的说明如下。

1. 如果当前运行的线程数少于corePoolSize（即线程池中无运行的线程），则创建一个新线程来执行任务。

2. 在线程池完成预热之后（当前线程池中有一个运行的线程），将任务加入LinkedBlockingQueue。

3. 线程执行完1中的任务后，会在一个无限循环中反复从LinkedBlockingQueue获取任务来执行。

### 10.2.3 CachedThreadPool详解

> [（转）深入详解Java线程池——Executor框架](https://www.cnblogs.com/wangle1001986/p/11195562.html)

​	**CachedThreadPool是一个会根据需要创建新线程的线程池**。下面是创建CachedThreadPool的源代码。

```java
public static ExecutorService newCachedThreadPool() {
  return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                60L, TimeUnit.SECONDS,
                                new SynchronousQueue<Runnable>());
}
```

​	CachedThreadPool的corePoolSize被设置为0，即corePool为空；maximumPoolSize被设置为Integer.MAX_VALUE，即maximumPool是无界的。这里把keepAliveTime设置为60L，意味着CachedThreadPool中的空闲线程等待新任务的最长时间为60秒，空闲线程超过60秒后将会被终止。

​	FixedThreadPool和SingleThreadExecutor使用无界队列LinkedBlockingQueue作为线程池的工作队列。CachedThreadPool使用没有容量的SynchronousQueue作为线程池的工作队列，但CachedThreadPool的maximumPool是无界的。这意味着，如果主线程提交任务的速度高于maximumPool中线程处理任务的速度时，CachedThreadPool会不断创建新线程。**极端情况下，CachedThreadPool会因为创建过多线程而耗尽CPU和内存资源**。

​	CachedThreadPool的execute()方法的执行示意图如下图所示。

![这里写图片描述](https://img-blog.csdn.net/20180125154131750?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjQ5ODIyOTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​	对上图的说明如下。

1. 首先执行`SynchronousQueue.offer（Runnable task）`。如果当前maximumPool中有空闲线程正在执行`SynchronousQueue.poll（keepAliveTime，TimeUnit.NANOSECONDS）`，那么主线程执行offer操作与空闲线程执行的poll操作配对成功，主线程把任务交给空闲线程执行，execute()方法执行完成；否则执行下面的步骤2）。

2. 当初始maximumPool为空，或者maximumPool中当前没有空闲线程时，将没有线程执行`SynchronousQueue.poll（keepAliveTime，TimeUnit.NANOSECONDS）`。这种情况下，步骤1）将失败。此时CachedThreadPool会创建一个新线程执行任务，execute()方法执行完成。

3. 在步骤2. 中新创建的线程将任务执行完后，会执行`SynchronousQueue.poll（keepAliveTime，TimeUnit.NANOSECONDS）`。这个poll操作会让空闲线程最多在SynchronousQueue中等待60秒钟。如果60秒钟内主线程提交了一个新任务（主线程执行步骤1.）），那么这个空闲线程将执行主线程提交的新任务；否则，这个空闲线程将终止。<u>由于空闲60秒的空闲线程会被终止，因此长时间保持空闲的CachedThreadPool不会使用任何资源</u>。

​	前面提到过，**SynchronousQueue是一个没有容量的阻塞队列。每个插入操作必须等待另一个线程的对应移除操作，反之亦然**。CachedThreadPool使用SynchronousQueue，把主线程提交的任务传递给空闲线程执行。CachedThreadPool中任务传递的示意图如下图所示。

![img](https://static.oschina.net/uploads/space/2018/0512/224657_CRYa_3352298.png)

## 10.3 ScheduledThreadPoolExecutor详解

​	ScheduledThreadPoolExecutor继承自ThreadPoolExecutor。它主要用来在给定的延迟之后运行任务，或者定期执行任务。ScheduledThreadPoolExecutor的功能与Timer类似，但ScheduledThreadPoolExecutor功能更强大、更灵活。**Timer对应的是单个后台线程，而ScheduledThreadPoolExecutor可以在构造函数中指定多个对应的后台线程数**。

### 10.3.1 ScheduledThreadPoolExecutor的运行机制

​	ScheduledThreadPoolExecutor的执行示意图（本文基于JDK 6）如下图所示。

![img](https://static.oschina.net/uploads/space/2018/0512/234343_q14N_3352298.png)

​	DelayQueue是一个无界队列，所以ThreadPoolExecutor的maximumPoolSize在ScheduledThreadPoolExecutor中没有什么意义（设置maximumPoolSize的大小没有什么效果）。

​	ScheduledThreadPoolExecutor的执行主要分为两大部分。

1. 当调用ScheduledThreadPoolExecutor的scheduleAtFixedRate()方法或者scheduleWithFixedDelay()方法时，会向ScheduledThreadPoolExecutor的DelayQueue添加一个**实现了RunnableScheduledFutur接口的ScheduledFutureTask**。

2. 线程池中的线程从DelayQueue中获取ScheduledFutureTask，然后执行任务。

​	ScheduledThreadPoolExecutor为了实现周期性的执行任务，对ThreadPoolExecutor做了如下的修改。

+ 使用DelayQueue作为任务队列。

+ 获取任务的方式不同（后文会说明）。

+ 执行周期任务后，增加了额外的处理（后文会说明）。

### 10.3.2 ScheduledThreadPoolExecutor的实现

> [Java并发编程 - 第十章 Executor框架](https://blog.csdn.net/weixin_41105242/article/details/108535335)

​	前面我们提到过，ScheduledThreadPoolExecutor会把待调度的任务（ScheduledFutureTask）放到一个DelayQueue中。

​	ScheduledFutureTask主要包含3个成员变量，如下。

+ long型成员变量time，表示这个任务将要被执行的具体时间。

+ long型成员变量sequenceNumber，表示这个任务被添加到ScheduledThreadPoolExecutor中的序号。

+ long型成员变量period，表示任务执行的间隔周期。

​	**DelayQueue封装了一个PriorityQueue，这个PriorityQueue会对队列中的Scheduled-FutureTask进行排序。排序时，time小的排在前面（时间早的任务将被先执行）。如果两个ScheduledFutureTask的time相同，就比较sequenceNumber，sequenceNumber小的排在前面（也就是说，如果两个任务的执行时间相同，那么先提交的任务将被先执行）**。

​	首先，让我们看看ScheduledThreadPoolExecutor中的线程执行周期任务的过程。下图是ScheduledThreadPoolExecutor中的线程1执行某个周期任务的4个步骤。

![img](https://static.oschina.net/uploads/space/2018/0512/234708_Xhsw_3352298.png)

下面是对这4个步骤的说明。

1. 线程1从DelayQueue中获取已到期的ScheduledFutureTask（DelayQueue.take()）。到期任务是指ScheduledFutureTask的time大于等于当前时间。

2. 线程1执行这个ScheduledFutureTask。

3. 线程1修改ScheduledFutureTask的time变量为下次将要被执行的时间。

4. 线程1把这个修改time之后的ScheduledFutureTask放回DelayQueue中（`DelayQueue.add()`）。

接下来，让我们看看上面的步骤1. 获取任务的过程。下面是DelayQueue.take()方法的源代码实现。

```java
public E take() throws InterruptedException {
  final ReentrantLock lock = this.lock;
  lock.lockInterruptibly();　　　　　　　　　　　　 // 1
  try {
    for (;;) {
      E first = q.peek();
      if (first == null) {
        available.await();　　　　　　　　　　 // 2.1
      } else {
        long delay = first.getDelay(TimeUnit.NANOSECONDS);
        if (delay > 0) {
          long tl = available.awaitNanos(delay);　　 // 2.2
        } else {
          E x = q.poll();　　　　　　　　　　 // 2.3.1
          assert x != null;
          if (q.size() != 0)
            available.signalAll();　　　　　　　　 // 2.3.2
          return x;
        }
      }
    }
  } finally {
    lock.unlock();　　　　　　　　　　　　　　 // 3
  }
}
```

下图是DelayQueue.take()的执行示意图。

![ScheduledThreadPoolExecutor 获取任务的过程](https://img-blog.csdnimg.cn/20201001103404501.jpg#pic_center)

​	如图所示，获取任务分为3大步骤。

1. 获取Lock。

2. 获取周期任务。

   + 如果PriorityQueue为空，当前线程到Condition中等待；否则执行下面的2.2。

   + 如果PriorityQueue的头元素的time时间比当前时间大，到Condition中等待到time时间；否则执行下面的2.3。

   + 获取PriorityQueue的头元素（2.3.1）；如果PriorityQueue不为空，则唤醒在Condition中等待的所有线程（2.3.2）。

3. 释放Lock。

​	ScheduledThreadPoolExecutor在一个循环中执行步骤2，直到线程从PriorityQueue获取到一个元素之后（执行2.3.1之后），才会退出无限循环（结束步骤2）。

​	最后，让我们看看ScheduledThreadPoolExecutor中的线程执行任务的步骤4，把ScheduledFutureTask放入DelayQueue中的过程。下面是DelayQueue.offer()的源代码实现。

```java
public boolean offer(E e) {
  final ReentrantLock lock = this.lock;
  lock.lock();　　　　　　　　　　 // 1
  try {
    E first = q.peek();
    q.offer(e);　　　　　　　　 // 2.1
    if (first == null || e.compareTo(first) < 0)
      available.signalAll();　　　 // 2.2
    return true;
  } finally {
    lock.unlock();　　　　　　　　 // 3
  }
}
```

下图是DelayQueue.offer()的执行示意图。

![ScheduledThreadPoolExecutor 添加任务的过程](https://img-blog.csdnimg.cn/20201001110428874.jpg#pic_center)

如图所示，添加任务分为3大步骤。

1. 获取Lock。

2. 添加任务。

   + 向PriorityQueue添加任务。

   + 如果在上面2.1中添加的任务是PriorityQueue的头元素，唤醒在Condition中等待的所有线程。

3. 释放Lock。

## 10.4. FutureTask详解

​	**Future接口和实现Future接口的FutureTask类，代表异步计算的结果**。

### 10.4.1 FutureTask简介

​	**FutureTask除了实现Future接口外，还实现了Runnable接口**。因此，FutureTask可以交给Executor执行，也可以由调用线程直接执行（`FutureTask.run()`）。根据`FutureTask.run()`方法被执行的时机，FutureTask可以处于下面3种状态。

1. **未启动**。`FutureTask.run()`方法还没有被执行之前，FutureTask处于未启动状态。当创建一个FutureTask，且没有执行FutureTask.run()方法之前，这个FutureTask处于未启动状态。

2. **已启动**。`FutureTask.run()`方法被执行的过程中，FutureTask处于已启动状态。

3. **已完成**。`FutureTask.run()`方法执行完后正常结束，或被取消（`FutureTask.cancel（…）`），或执行`FutureTask.run()`方法时抛出异常而异常结束，FutureTask处于已完成状态。

下图是FutureTask的状态迁移的示意图。

![FutureTask 的状态迁移示意图](https://img-blog.csdnimg.cn/20201001124048826.jpg#pic_center)

+ **当FutureTask处于未启动或已启动状态时，执行`FutureTask.get()`方法将导致调用线程阻塞；**
+ **当FutureTask处于已完成状态时，执行`FutureTask.get()`方法将导致调用线程立即返回结果或抛出异常。**

---

+ 当FutureTask处于未启动状态时，执行`FutureTask.cancel()`方法将导致此任务永远不会被执行；
+ 当FutureTask处于已启动状态时，执行`FutureTask.cancel（true）`方法将以**中断**执行此任务线程的方式来试图停止任务；
+ 当FutureTask处于已启动状态时，执行`FutureTask.cancel（false）`方法将不会对正在执行此任务的线程产生影响（让正在执行的任务运行完成）；
+ 当FutureTask处于已完成状态时，执行`FutureTask.cancel（…）`方法将返回false。

​	下图是get方法和cancel方法的执行示意图。

![FutureTask 的 get 和 cancel 的执行示意图](https://img-blog.csdnimg.cn/2020100112484678.jpg#pic_center)

### 10.4.2 FutureTask的使用

​	可以把FutureTask交给Executor执行；也可以通过`ExecutorService.submit（…）`方法返回一个FutureTask，然后执行`FutureTask.get()`方法或`FutureTask.cancel（…）`方法。除此以外，还可以单独使用FutureTask。

​	**当一个线程需要等待另一个线程把某个任务执行完后它才能继续执行，此时可以使用FutureTask**。假设有多个线程执行若干任务，每个任务最多只能被执行一次。当多个线程试图同时执行同一个任务时，只允许一个线程执行任务，其他线程需要等待这个任务执行完后才能继续执行。下面是对应的示例代码。

```java
private final ConcurrentMap<Object, Future<String>> taskCache =
  new ConcurrentHashMap<Object, Future<String>>();
private String executionTask(final String taskName)
  throws ExecutionException, InterruptedException {
  while (true) {
    Future<String> future = taskCache.get(taskName);　　 // 1.1,2.1
    if (future == null) {
      Callable<String> task = new Callable<String>() {
        public String call() throws InterruptedException {
          return taskName;
        }
      }; 
      FutureTask<String> futureTask = new FutureTask<String>(task);
      future = taskCache.putIfAbsent(taskName, futureTask);　 // 1.3
      if (future == null) {
        future = futureTask;
        futureTask.run();　　　　　　　　 // 1.4执行任务
      }
    }
    try {
      return future.get();　　　　　　 // 1.5,2.2
    } catch (CancellationException e) {
      taskCache.remove(taskName, future);
    }
  }
}
```

上述代码的执行示意图如下图所示。

![代码的执行示意图](https://img-blog.csdnimg.cn/20201001133001526.jpg#pic_center)

​	当两个线程试图同时执行同一个任务时，如果Thread 1执行1.3后Thread 2执行2.1，那么接下来Thread 2将在2.2等待，直到Thread 1执行完1.4后Thread 2才能从2.2（`FutureTask.get()`）返回。

### 10.4.3 FutureTask的实现

​	FutureTask的实现基于AbstractQueuedSynchronizer（以下简称为AQS）。**java.util.concurrent中的很多可阻塞类（比如ReentrantLock）都是基于AQS来实现的**。AQS是一个同步框架，它提供通用机制来原子性管理同步状态、阻塞和唤醒线程，以及维护被阻塞线程的队列。**JDK 6中AQS被广泛使用，基于AQS实现的同步器包括：ReentrantLock、Semaphore、ReentrantReadWriteLock、CountDownLatch和FutureTask。**

​	每一个基于AQS实现的同步器都会包含两种类型的操作，如下。

+ **至少一个acquire操作**。这个操作阻塞调用线程，除非/直到AQS的状态允许这个线程继续执行。FutureTask的acquire操作为`get()/get（long timeout，TimeUnit unit）`方法调用。

+ **至少一个release操作**。这个操作改变AQS的状态，改变后的状态可允许一个或多个阻塞线程被解除阻塞。FutureTask的release操作包括`run()`方法和`cancel（…）`方法。

​	**基于“复合优先于继承”的原则，FutureTask声明了一个内部私有的继承于AQS的子类Sync，对FutureTask所有公有方法的调用都会委托给这个内部子类。**

​	AQS被作为“模板方法模式”的基础类提供给FutureTask的内部子类Sync，这个内部子类只需要实现**状态检查**和**状态更新**的方法即可，这些方法将控制FutureTask的获取和释放操作。具体来说，Sync实现了AQS的`tryAcquireShared（int）`方法和`tryReleaseShared（int）`方法，Sync通过这两个方法来检查和更新同步状态。

​	FutureTask的设计示意图如下图所示。

![FutureTask 的设计示意图](https://img-blog.csdnimg.cn/20201001133118533.jpg#pic_center)

​	如图所示，Sync是FutureTask的内部私有类，它继承自AQS。创建FutureTask时会创建内部私有的成员对象Sync，FutureTask所有的的公有方法都直接委托给了内部私有的Sync。

​	`FutureTask.get()`方法会调用`AQS.acquireSharedInterruptibly（int arg）`方法，这个方法的执行过程如下。

1. 调用`AQS.acquireSharedInterruptibly（int arg）`方法，这个方法首先会回调在子类Sync中实现的`tryAcquireShared()`方法来判断acquire操作是否可以成功。acquire操作可以成功的条件为：state为执行完成状态RAN或已取消状态CANCELLED，且runner不为null。

2. 如果成功则`get()`方法立即返回。如果失败则到线程等待队列中去等待其他线程执行release操作。

3. 当其他线程执行release操作（比如`FutureTask.run()或FutureTask.cancel（…）`）唤醒当前线程后，当前线程再次执行`tryAcquireShared()`将返回正值1，当前线程将离开线程等待队列并唤醒它的后继线程（这里会产生级联唤醒的效果，后面会介绍）。

4. 最后返回计算的结果或抛出异常。

`FutureTask.run()`的执行过程如下。

1. 执行在构造函数中指定的任务（`Callable.call()`）。

2. **以原子方式来更新同步状态**（调用`AQS.compareAndSetState（int expect，int update）`，设置state为执行完成状态RAN）。如果这个原子操作成功，就设置代表计算结果的变量result的值为`Callable.call()`的返回值，然后调用`AQS.releaseShared（int arg）`。

3. `AQS.releaseShared（int arg）`首先会回调在子类Sync中实现的`tryReleaseShared（arg）`来执行release操作（设置运行任务的线程runner为null，然会返回true）；`AQS.releaseShared（int arg）`，然后唤醒线程等待队列中的第一个线程。

4. 调用`FutureTask.done()`。当执行`FutureTask.get()`方法时，如果FutureTask不是处于执行完成状态RAN或已取消状态CANCELLED，当前执行线程将到AQS的线程等待队列中等待（见下图的线程A、B、C和D）。当某个线程执行`FutureTask.run()`方法或`FutureTask.cancel（...）`方法时，会唤醒线程等待队列的第一个线程（见下图所示的线程E唤醒线程A）。

![FutureTask 的级联唤醒示意图](https://img-blog.csdnimg.cn/20201001133152151.jpg#pic_center)

​	假设开始时FutureTask处于未启动状态或已启动状态，等待队列中已经有3个线程（A、B和C）在等待。此时，线程D执行get()方法将导致线程D也到等待队列中去等待。

​	当线程E执行run()方法时，会唤醒队列中的第一个线程A。线程A被唤醒后，首先把自己从队列中删除，然后唤醒它的后继线程B，最后线程A从get()方法返回。线程B、C和D重复A线程的处理流程。最终，在队列中等待的所有线程都被级联唤醒并从get()方法返回。

## 10.5 本章小结

​	本章介绍了Executor框架的整体结构和成员组件。希望读者阅读本章之后，能够对Executor框架有一个比较深入的理解，同时也希望本章内容有助于读者更熟练地使用Executor框架。

# 第11章 Java并发编程实战

​	当你在进行并发编程时，看着程序的执行速度在自己的优化下运行得越来越快，你会觉得越来越有成就感，这就是并发编程的魅力。但与此同时，并发编程产生的问题和风险可能也会随之而来。本章先介绍几个并发编程的实战案例，然后再介绍如何排查并发编程造成的问题。

## 11.1 生产者和消费者模式

​	在并发编程中使用**生产者和消费者模式**能够解决绝大多数并发问题。该模式通过平衡生产线程和消费线程的工作能力来提高程序整体处理数据的速度。

​	在线程世界里，生产者就是生产数据的线程，消费者就是消费数据的线程。在多线程开发中，如果生产者处理速度很快，而消费者处理速度很慢，那么生产者就必须等待消费者处理完，才能继续生产数据。同样的道理，如果消费者的处理能力大于生产者，那么消费者就必须等待生产者。为了解决这种生产消费能力不均衡的问题，便有了生产者和消费者模式。

​	**生产者和消费者模式是通过一个容器来解决生产者和消费者的强耦合问题。生产者和消费者彼此之间不直接通信，而是通过阻塞队列来进行通信，所以生产者生产完数据之后不用等待消费者处理，直接扔给阻塞队列，消费者不找生产者要数据，而是直接从阻塞队列里取，阻塞队列就相当于一个缓冲区，平衡了生产者和消费者的处理能力。**

​	<u>这个阻塞队列就是用来给生产者和消费者解耦的。纵观大多数设计模式，都会找一个第三者出来进行解耦，如工厂模式的第三者是工厂类，模板模式的第三者是模板类。在学习一些设计模式的过程中，先找到这个模式的第三者，能帮助我们快速熟悉一个设计模式。</u>

### 11.1.1 生产者消费者模式实战

​	我和同事一起利用业余时间开发的Yuna工具中使用了生产者和消费者模式。我先介绍下Yuna工具，在阿里巴巴很多同事都喜欢通过邮件分享技术文章，因为通过邮件分享很方便，大家在网上看到好的技术文章，执行复制→粘贴→发送就完成了一次分享，但是我发现技术文章不能沉淀下来，新来的同事看不到以前分享的技术文章，大家也很难找到以前分享过的技术文章。为了解决这个问题，我们开发了一个Yuna工具。

​	我们申请了一个专门用来收集分享邮件的邮箱，比如share@alibaba.com，大家将分享的文章发送到这个邮箱，让大家每次都抄送到这个邮箱肯定很麻烦，所以我们的做法是将这个邮箱地址放在部门邮件列表里，所以分享的同事只需要和以前一样向整个部门分享文章就行。Yuna工具通过读取邮件服务器里该邮箱的邮件，把所有分享的邮件下载下来，包括邮件的附件、图片和邮件回复。因为我们可能会从这个邮箱里下载到一些非分享的文章，所以我们要求分享的邮件标题必须带有一个关键字，比如“内贸技术分享”。下载完邮件之后，通过confluence的Web Service接口，把文章插入到confluence里，这样新同事就可以在confluence里看以前分享过的文章了，并且Yuna工具还可以自动把文章进行分类和归档。

​	为了快速上线该功能，当时我们花了3天业余时间快速开发了Yuna 1.0版本。在1.0版本中并没有使用生产者消费模式，而是使用单线程来处理，因为当时只需要处理我们一个部门的邮件，所以单线程明显够用，整个过程是串行执行的。在一个线程里，程序先抽取全部的邮件，转化为文章对象，然后添加全部的文章，最后删除抽取过的邮件。代码如下。

```java
public void extract() {
  logger.debug("开始" + getExtractorName() + "。。");
  // 抽取邮件
  List<Article> articles = extractEmail();
  // 添加文章
  for (Article article : articles) {
    addArticleOrComment(article);
  }
  // 清空邮件
  cleanEmail();
  logger.debug("完成" + getExtractorName() + "。。");
}
```

​	Yuna工具在推广后，越来越多的部门使用这个工具，处理的时间越来越慢，Yuna是每隔5分钟进行一次抽取的，而当邮件多的时候一次处理可能就花了几分钟，于是我在Yuna 2.0版本里使用了生产者消费者模式来处理邮件，首先生产者线程按一定的规则去邮件系统里抽取邮件，然后存放在阻塞队列里，消费者从阻塞队列里取出文章后插入到conflunce里。代码如下。

```java
public class QuickEmailToWikiExtractor extends AbstractExtractor {
  private ThreadPoolExecutor threadsPool;
  private ArticleBlockingQueue<ExchangeEmailShallowDTO> emailQueue;
  public QuickEmailToWikiExtractor() {
    emailQueue = new ArticleBlockingQueue<ExchangeEmailShallowDTO>();
    int corePoolSize = Runtime.getRuntime().availableProcessors() * 2;
    threadsPool = new ThreadPoolExecutor(corePoolSize, corePoolSize, 10l, TimeUnit.
                                         SECONDS,
                                         new LinkedBlockingQueue<Runnable>(2000));
  }
  public void extract() {
    logger.debug("开始" + getExtractorName() + "。。");
    long start = System.currentTimeMillis();
    // 抽取所有邮件放到队列里
    new ExtractEmailTask().start();
    // 把队列里的文章插入到Wiki
    insertToWiki();
    long end = System.currentTimeMillis();
    double cost = (end - start) / 1000;
    logger.debug("完成" + getExtractorName() + ",花费时间：" + cost + "秒");
  }
  /**
* 把队列里的文章插入到Wiki
*/
  private void insertToWiki() {
    // 登录Wiki,每间隔一段时间需要登录一次
    confluenceService.login(RuleFactory.USER_NAME, RuleFactory.PASSWORD);
    while (true) {
      // 2秒内取不到就退出
      ExchangeEmailShallowDTO email = emailQueue.poll(2, TimeUnit.SECONDS);
      if (email == null) {
        break;
      }
      threadsPool.submit(new insertToWikiTask(email));
    }
  }
  protected List<Article> extractEmail() {
    List<ExchangeEmailShallowDTO> allEmails = getEmailService().queryAllEmails();
    if (allEmails == null) {
      return null;
    }
    for (ExchangeEmailShallowDTO exchangeEmailShallowDTO : allEmails) {
      emailQueue.offer(exchangeEmailShallowDTO);
    }
    return null;
  }
  /**
* 抽取邮件任务
*
* @author tengfei.fangtf
*/
  public class ExtractEmailTask extends Thread {
    public void run() {
      extractEmail();
    }
  }
}
```

​	代码的执行逻辑是，生产者启动一个线程把所有邮件全部抽取到队列中，消费者启动CPU\*2个线程数处理邮件，从之前的单线程处理邮件变成了现在的多线程处理，并且抽取邮件的线程不需要等处理邮件的线程处理完再抽取新邮件，所以使用了生产者和消费者模式后，邮件的整体处理速度比以前要快了几倍。

### 11.1.2 多生产者和多消费者场景

> [生产者消费者模式[转]](https://www.jianshu.com/p/1bcb8465f617)

​	在多核时代，多线程并发处理速度比单线程处理速度更快，所以可以使用多个线程来生产数据，同样可以使用多个消费线程来消费数据。而更复杂的情况是，消费者消费的数据，有可能需要继续处理，于是消费者处理完数据之后，它又要作为生产者把数据放在新的队列里，交给其他消费者继续处理，如下图所示。

![img](https://upload-images.jianshu.io/upload_images/6383319-17c5c5c655500b4e.png)

​	我们在一个长连接服务器中使用了这种模式，生产者1负责将所有客户端发送的消息存放在阻塞队列1里，消费者1从队列里读消息，然后通过消息ID进行散列得到N个队列中的一个，然后根据编号将消息存放在到不同的队列里，每个阻塞队列会分配一个线程来消费阻塞队列里的数据。**如果消费者2无法消费消息，就将消息再抛回到阻塞队列1中，交给其他消费者处理**。

​	以下是消息总队列的代码。

```java
/**
* 总消息队列管理
*
* @author tengfei.fangtf
*/
public class MsgQueueManager implements IMsgQueue{
  private static final Logger LOGGER
    = LoggerFactory.getLogger(MsgQueueManager.class);
  /**
* 消息总队列
*/
  public final BlockingQueue<Message> messageQueue;
  private MsgQueueManager() {
    messageQueue = new LinkedTransferQueue<Message>();
  }
  public void put(Message msg) {
    try {
      messageQueue.put(msg);
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    }
  }
  public Message take() {
    try {
      return messageQueue.take();
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    }
    return null;
  }
}
```

启动一个消息分发线程。在这个线程里子队列自动去总队列里获取消息。

```java
/**
* 分发消息，负责把消息从大队列塞到小队列里
*
* @author tengfei.fangtf
*/
static class DispatchMessageTask implements Runnable {
  @Override
  public void run() {
    BlockingQueue<Message> subQueue;
    for (;;) {
      // 如果没有数据，则阻塞在这里
      Message msg = MsgQueueFactory.getMessageQueue().take();
      // 如果为空，则表示没有Session机器连接上来，
      // 需要等待，直到有Session机器连接上来
      while ((subQueue = getInstance().getSubQueue()) == null) {
        try {
          Thread.sleep(1000);
        } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
        }
      }
      // 把消息放到小队列里
      try {
        subQueue.put(msg);
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }
}
```

使用散列（hash）算法获取一个子队列，代码如下。

```java
/**
* 均衡获取一个子队列。
*
* @return
*/
public BlockingQueue<Message> getSubQueue() {
  int errorCount = 0;
  for (;;) {
    if (subMsgQueues.isEmpty()) {
      return null;
    }
    int index = (int) (System.nanoTime() % subMsgQueues.size());
    try {
      return subMsgQueues.get(index);
    } catch (Exception e) {
      // 出现错误表示，在获取队列大小之后，队列进行了一次删除操作
      LOGGER.error("获取子队列出现错误", e);
      if ((++errorCount) < 3) {
        continue;
      }
    }
  }
}
```

使用的时候，只需要往总队列里发消息。

```java
//往消息队列里添加一条消息
IMsgQueue messageQueue = MsgQueueFactory.getMessageQueue();
Packet msg = Packet.createPacket(Packet64FrameType.
                                 TYPE_DATA, "{}".getBytes(), (short) 1);
messageQueue.put(msg);
```

### 11.1.3 线程池与生产消费者模式

​	Java中的线程池类其实就是一种生产者和消费者模式的实现方式，但是我觉得其实现方式更加高明。生产者把任务丢给线程池，线程池创建线程并处理任务，如果将要运行的任务数大于线程池的基本线程数就把任务扔到阻塞队列里，这种做法比只使用一个阻塞队列来实现生产者和消费者模式显然要高明很多，因为消费者能够处理直接就处理掉了，这样速度更快，而生产者先存，消费者再取这种方式显然慢一些。

​	我们的系统也可以使用线程池来实现多生产者和消费者模式。例如，创建N个不同规模的Java线程池来处理不同性质的任务，比如线程池1将数据读到内存之后，交给线程池2里的线程继续处理压缩数据。线程池1主要处理IO密集型任务，线程池2主要处理CPU密集型任务。

​	本节讲解了生产者和消费者模式，并给出了实例。读者可以在平时的工作中思考一下哪些场景可以使用生产者消费者模式，我相信这种场景应该非常多，特别是需要处理任务时间比较长的场景，比如上传附件并处理，用户把文件上传到系统后，系统把文件丢到队列里，然后立刻返回告诉用户上传成功，最后消费者再去队列里取出文件处理。再如，**调用一个远程接口查询数据，如果远程服务接口查询时需要几十秒的时间，那么它可以提供一个申请查询的接口，这个接口把要申请查询任务放数据库中，然后该接口立刻返回。然后服务器端用线程轮询并获取申请任务进行处理，处理完之后发消息给调用方，让调用方再来调用另外一个接口取数据**。

## 11.2 线上问题定位

​	有时候，有很多问题只有在线上或者预发环境才能发现，而线上又不能调试代码，所以线上问题定位就只能看日志、系统状态和dump线程，本节只是简单地介绍一些常用的工具，以帮助大家定位线上问题。

1. 在Linux命令行下使用TOP命令查看每个进程的情况

   ```shell
   top - 22:27:25 up 463 days, 12:46, 1 user, load average: 11.80, 12.19, 11.79
   Tasks: 113 total, 5 running, 108 sleeping, 0 stopped, 0 zombie
   Cpu(s): 62.0%us, 2.8%sy, 0.0%ni, 34.3%id, 0.0%wa, 0.0%hi, 0.7%si, 0.2%st
   Mem: 7680000k total, 7665504k used, 14496k free, 97268k buffers
   Swap: 2096472k total, 14904k used, 2081568k free, 3033060k cached
   PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
   31177 admin 18 0 5351m 4.0g 49m S 301.4 54.0 935:02.08 java
   31738 admin 15 0 36432 12m 1052 S 8.7 0.2 11:21.05 nginx-proxy
   ```

   ​	我们的程序是Java应用，所以只需要关注COMMAND是Java的性能数据，COMMAND表示启动当前进程的命令，在Java进程这一行里可以看到CPU利用率是300%，不用担心，这个是当前机器所有核加在一起的CPU利用率。

2. 再使用top的交互命令数字1查看每个CPU的性能数据。

   ```shell
   top - 22:24:50 up 463 days, 12:43, 1 user, load average: 12.55, 12.27, 11.73
   Tasks: 110 total, 3 running, 107 sleeping, 0 stopped, 0 zombie
   Cpu0 : 72.4%us, 3.6%sy, 0.0%ni, 22.7%id, 0.0%wa, 0.0%hi, 0.7%si, 0.7%st
   Cpu1 : 58.7%us, 4.3%sy, 0.0%ni, 34.3%id, 0.0%wa, 0.0%hi, 2.3%si, 0.3%st
   Cpu2 : 53.3%us, 2.6%sy, 0.0%ni, 34.1%id, 0.0%wa, 0.0%hi, 9.6%si, 0.3%st
   Cpu3 : 52.7%us, 2.7%sy, 0.0%ni, 25.2%id, 0.0%wa, 0.0%hi, 19.5%si, 0.0%st
   Cpu4 : 59.5%us, 2.7%sy, 0.0%ni, 31.2%id, 0.0%wa, 0.0%hi, 6.6%si, 0.0%st
   Mem: 7680000k total, 7663152k used, 16848k free, 98068k buffers
   Swap: 2096472k total, 14904k used, 2081568k free, 3032636k cached
   ```

   命令行显示了CPU4，说明这是一个5核的虚拟机，平均每个CPU利用率在60%以上。如果这里显示CPU利用率100%，则很有可能程序里写了一个死循环。这些参数的含义，可以对比下表来查看。

   | 参数     | 描述                                          |
   | -------- | --------------------------------------------- |
   | us       | **用户空间占用CPU百分比**                     |
   | 1.0% sy  | 内核空间占用CPU百分比                         |
   | 0.0% ni  | 用户进程空间内改变过优先级的进程占用CPU百分比 |
   | 98.7% id | 空闲CPU百分比                                 |
   | 0.0% wa  | 等待输入输出的CPU时间百分比                   |

3. 使用top的交互命令H查看每个线程的性能信息。

   ```shell
   PID USER PR NI VIRT RES SHR S %CPU %MEM TIME+ COMMAND
   31558 admin 15 0 5351m 4.0g 49m S 12.2 54.0 10:08.31 java
   31561 admin 15 0 5351m 4.0g 49m R 12.2 54.0 9:45.43 java
   31626 admin 15 0 5351m 4.0g 49m S 11.9 54.0 13:50.21 java
   31559 admin 15 0 5351m 4.0g 49m S 10.9 54.0 5:34.67 java
   31612 admin 15 0 5351m 4.0g 49m S 10.6 54.0 8:42.77 java
   31555 admin 15 0 5351m 4.0g 49m S 10.3 54.0 13:00.55 java
   31630 admin 15 0 5351m 4.0g 49m R 10.3 54.0 4:00.75 java
   31646 admin 15 0 5351m 4.0g 49m S 10.3 54.0 3:19.92 java
   31653 admin 15 0 5351m 4.0g 49m S 10.3 54.0 8:52.90 java
   31607 admin 15 0 5351m 4.0g 49m S 9.9 54.0 14:37.82 java
   ```

   在这里可能会出现3种情况。

   + 第一种情况，**某个线程CPU利用率一直100%，则说明是这个线程有可能有死循环**，那么请记住这个PID。

   + 第二种情况，**某个线程一直在TOP 10的位置，这说明这个线程可能有性能问题**。

   + 第三种情况，CPU利用率高的几个线程在不停变化，说明并不是由某一个线程导致CPU偏高。

   **如果是第一种情况，也有可能是GC造成，可以用jstat命令看一下GC情况，看看是不是因为持久代或年老代满了，产生Full GC，导致CPU利用率持续飙高**，命令和回显如下。

   ```shell
   sudo /opt/java/bin/jstat -gcutil 31177 1000 5
   S0 S1 E O P YGC YGCT FGC FGCT GCT
   0.00 1.27 61.30 55.57 59.98 16040 143.775 30 77.692 221.467
   0.00 1.27 95.77 55.57 59.98 16040 143.775 30 77.692 221.467
   1.37 0.00 33.21 55.57 59.98 16041 143.781 30 77.692 221.474
   1.37 0.00 74.96 55.57 59.98 16041 143.781 30 77.692 221.474
   0.00 1.59 22.14 55.57 59.98 16042 143.789 30 77.692 221.481
   ```

   还可以把线程dump下来，看看究竟是哪个线程、执行什么代码造成的CPU利用率高。执行以下命令，把线程dump到文件dump17里。执行如下命令。

   ```shell
   sudo -u admin /opt/taobao/java/bin/jstack 31177 > /home/tengfei.fangtf/dump17
   ```

   dump出来内容的类似下面内容。

   ```shell
   "http-0.0.0.0-7001-97" daemon prio=10 tid=0x000000004f6a8000 nid=0x555e in Object.
   wait() [0x0000000052423000]
   java.lang.Thread.State: WAITING (on object monitor)
   at java.lang.Object.wait(Native Method)
   - waiting on (a org.apache.tomcat.util.net.AprEndpoint$Worker)
   at java.lang.Object.wait(Object.java:485)
   at org.apache.tomcat.util.net.AprEndpoint$Worker.await(AprEndpoint.java:1464)
   - locked (a org.apache.tomcat.util.net.AprEndpoint$Worker)
   at org.apache.tomcat.util.net.AprEndpoint$Worker.run(AprEndpoint.java:1489)
   at java.lang.Thread.run(Thread.java:662)
   ```

   dump出来的线程ID（nid）是十六进制的，而我们用TOP命令看到的线程ID是十进制的，所以要用printf命令转换一下进制。然后用十六进制的ID去dump里找到对应的线程。

   ```shell
   printf "%x\n" 31558
   ```

   输出：7b46。

## 11.3 性能测试

​	因为要支持某个业务，有同事向我们提出需求，希望系统的某个接口能够支持2万的QPS，因为我们的应用部署在多台机器上，要支持两万的QPS，我们必须先要知道该接口在单机上能支持多少QPS，如果单机能支持1千QPS，我们需要20台机器才能支持2万的QPS。需要注意的是，要支持的2万的QPS必须是峰值，而不能是平均值，比如一天当中有23个小时QPS不足1万，只有一个小时的QPS达到了2万，我们的系统也要支持2万的QPS。

​	我们先进行性能测试。我们使用公司同事开发的性能测试工具进行测试，该工具的原理是，用户写一个Java程序向服务器端发起请求，这个工具会启动一个线程池来调度这些任务，可以配置同时启动多少个线程、发起请求次数和任务间隔时长。将这个程序部署在多台机器上执行，统计出QPS和响应时长。我们在10台机器上部署了这个测试程序，每台机器启动了100个线程进行测试，压测时长为半小时。注意不能压测线上机器，我们压测的是开发服务器。

​	测试开始后，首先登录到服务器里查看当前有多少台机器在压测服务器，因为程序的端口是12200，所以使用netstat命令查询有多少台机器连接到这个端口上。命令如下。

```shell
$ netstat -nat | grep 12200 –c
10
```

​	通过这个命令可以知道已经有10台机器在压测服务器。QPS达到了1400，程序开始报错获取不到数据库连接，因为我们的数据库端口是3306，用netstat命令查看已经使用了多少个数据库连接。命令如下。

```shell
$ netstat -nat | grep 3306 –c
12
```

​	增加数据库连接到20，QPS没上去，但是响应时长从平均1000毫秒下降到700毫秒，使用TOP命令观察CPU利用率，发现已经90%多了，于是升级CPU，将2核升级成4核，和线上的机器保持一致。再进行压测，CPU利用率下去了达到了75%，QPS上升到了1800。执行一段时间后响应时长稳定在200毫秒。

​	增加应用服务器里线程池的核心线程数和最大线程数到1024，通过ps命令查看下线程数是否增长了，执行的命令如下。

```shell
$ ps -eLf | grep java -c
1520
```

​	再次压测，QPS并没有明显的增长，单机QPS稳定在1800左右，响应时长稳定在200毫秒。

​	我在性能测试之前先优化了程序的SQL语句。使用了如下命令统计执行最慢的SQL，左边的是执行时长，单位是毫秒，右边的是执行的语句，可以看到系统执行最慢的SQL是queryNews和queryNewIds，优化到几十毫秒。

```shell
$ grep Y /home/admin/logs/xxx/monitor/dal-rw-monitor.log |awk -F',' '{print $7$5}' |
sort -nr|head -20
1811 queryNews
1764 queryNews
1740 queryNews
1697 queryNews
679 queryNewIds
```

---

### 性能测试中使用的其他命令

#### 1. 查看网络流量

```shell
$ cat /proc/net/dev
Inter-| Receive | Transmit
face |bytes packets errs drop fifo frame compressed multicast|bytes packets
errs drop fifo colls carrier compressed
lo:242953548208 231437133 0 0 0 0 0 0 242953548208 231437133 0 0 0 0 0 0
eth0:153060432504 446365779 0 0 0 0 0 0 108596061848 479947142 0 0 0 0 0 0
bond0:153060432504 446365779 0 0 0 0 0 0 108596061848 479947142 0 0 0 0 0 0
```

#### 2. 查看系统平均负载

```shell
$ cat /proc/loadavg
0.00 0.04 0.85 1/1266 22459
```

#### 3. 查看系统内存情况

```shell
$ cat /proc/meminfo
MemTotal: 4106756 kB
MemFree: 71196 kB
Buffers: 12832 kB
Cached: 2603332 kB
SwapCached: 4016 kB
Active: 2303768 kB
Inactive: 1507324 kB
Active(anon): 996100 kB
部分省略
```

#### 4. 查看CPU的利用率

```shell
cat /proc/stat
cpu 167301886 6156 331902067 17552830039 8645275 13082 1044952 33931469 0
cpu0 45406479 1992 75489851 4410199442 7321828 12872 688837 5115394 0
cpu1 39821071 1247 132648851 4319596686 379255 67 132447 11365141 0
cpu2 40912727 1705 57947971 4418978718 389539 78 110994 8342835 0
cpu3 41161608 1211 65815393 4404055191 554651 63 112672 9108097 0
```

## 11.4 异步任务池

> [Java异步线程池](https://blog.csdn.net/weixin_41705493/article/details/102499497)

​	Java中的线程池设计得非常巧妙，可以高效并发执行多个任务，但是在某些场景下需要对线程池进行扩展才能更好地服务于系统。例如，如果一个任务仍进线程池之后，运行线程池的程序重启了，那么线程池里的任务就会丢失。另外，线程池只能处理本机的任务，在集群环境下不能有效地调度所有机器的任务。所以，需要结合线程池开发一个异步任务处理池。下图为异步任务池设计图。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019101113352191.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MTcwNTQ5Mw==,size_16,color_FFFFFF,t_70)

​	任务池的主要处理流程是，每台机器会启动一个任务池，每个任务池里有多个线程池，当某台机器将一个任务交给任务池后，任务池会先将这个任务保存到数据中，然后某台机器上的任务池会从数据库中获取待执行的任务，再执行这个任务。

​	每个任务有几种状态，分别是创建（NEW）、执行中（EXECUTING）、RETRY（重试）、挂起（SUSPEND）、中止（TEMINER）和执行完成（FINISH）。

+ 创建：提交给任务池之后的状态。

+ 执行中：任务池从数据库中拿到任务执行时的状态。

+ 重试：当执行任务时出现错误，程序显式地告诉任务池这个任务需要重试，并设置下一次执行时间。

+ 挂起：当一个任务的执行依赖于其他任务完成时，可以将这个任务挂起，当收到消息后，再开始执行。

+ 中止：任务执行失败，让任务池停止执行这个任务，并设置错误消息告诉调用端。

+ 执行完成：任务执行结束。

### 任务池的任务隔离

​	**异步任务有很多种类型，比如抓取网页任务、同步数据任务等**，不同类型的任务优先级不一样，但是系统资源是有限的，如果低优先级的任务非常多，高优先级的任务就可能得不到执行，所以必须对任务进行隔离执行。使用不同的线程池处理不同的任务，或者不同的线程池处理不同优先级的任务，**如果任务类型非常少，建议用任务类型来隔离，如果任务类型非常多，比如几十个，建议采用优先级的方式来隔离。**

### 任务池的重试策略

​	**根据不同的任务类型设置不同的重试策略**，有的任务对实时性要求高，那么每次的重试间隔就会非常短，如果对实时性要求不高，可以采用默认的重试策略，重试间隔随着次数的增加，时间不断增长，比如间隔几秒、几分钟到几小时。**每个任务类型可以设置执行该任务类型线程池的最小和最大线程数、最大重试次数。**

### 使用任务池的注意事项

​	**任务必须无状态**：任务不能在执行任务的机器中保存数据，比如某个任务是处理上传的文件，任务的属性里有文件的上传路径，如果文件上传到机器1，机器2获取到了任务则会处理失败，所以上传的文件必须存在其他的集群里，比如OSS或SFTP。

### 异步任务的属性

​	**包括任务名称、下次执行时间、已执行次数、任务类型、任务优先级和执行时的报错信息（用于快速定位问题）**。

## 11.5 本章小结

​	本章介绍了使用生产者和消费者模式进行并发编程、线上问题排查手段和性能测试实战，以及异步任务池的设计。并发编程的实战需要大家平时多使用和测试，才能在项目中发挥作用。

