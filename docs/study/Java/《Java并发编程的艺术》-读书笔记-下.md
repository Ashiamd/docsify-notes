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

​	上述代码主要完成了同步状态获取、节点构造、加入同步队列以及在同步队列中自旋等待的相关工作，其主要逻辑是：首先调用自定义同步器实现的tryAcquire(int arg)方法，该方法保证线程安全的获取同步状态，如果同步状态获取失败，则构造同步节点（独占式Node.EXCLUSIVE，同一时刻只能有一个线程成功获取同步状态）并通过addWaiter(Node node)方法将该节点加入到同步队列的尾部，最后调用acquireQueued(Node node,int arg)方法，使得该节点以“死循环”的方式获取同步状态。<u>如果获取不到则阻塞节点中的线程，而被阻塞线程的唤醒主要依靠前驱节点的出队或阻塞线程被中断来实现。</u>

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

​	之前提到锁（如Mutex和ReentrantLock）基本都是排他锁，这些锁在同一时刻只允许一个线程进行访问，而读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过分离读锁和写锁，使得并发性相比一般的排他锁有了很大提升。

​	除了保证写操作对读操作的可见性以及并发性的提升之外，读写锁能够简化读写交互场景的编程方式。假设在程序中定义一个共享的用作缓存数据结构，它大部分时间提供读服务（例如查询和搜索），而写操作占有的时间很少，但是写操作完成之后的更新需要对后续的读服务可见。

​	在没有读写锁支持的（Java 5之前）时候，如果需要完成上述工作就要使用Java的等待通知机制，就是当写操作开始时，所有晚于写操作的读操作均会进入等待状态，只有写操作完成并进行通知之后，所有等待的读操作才能继续执行（写操作之间依靠synchronized关键进行同步），这样做的目的是使读操作能读取到正确的数据，不会出现脏读。改用读写锁实现上述功能，只需要在读操作时获取读锁，写操作时获取写锁即可。当写锁被获取到时，后续（非当前写操作线程）的读写操作都会被阻塞，写锁释放之后，所有操作继续执行，编程方式相对于使用等待通知机制的实现方式而言，变得简单明了。

​	一般情况下，读写锁的性能都会比排它锁好，因为大多数场景读是多于写的。在读多于写的情况下，读写锁能够提供比排它锁更好的并发性和吞吐量。Java并发包提供读写锁的实现是ReentrantReadWriteLock，它提供的特性如下表所示。

| 特性       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 公平性选择 | 支持非公平（默认）和公平的锁获取方式，吞吐量还是非公平优于公平 |
| 重进入     | 该锁支持重进入，以读写线程为例：读线程在获取了读锁之后，能够再次获取读锁。而写线程在获取了写锁之后能够再次获取写锁，同时也可以获取读锁 |
| 锁降级     | 遵循获取写锁、获取读锁再释放写锁的次序，写锁能够降级成为读锁 |

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

