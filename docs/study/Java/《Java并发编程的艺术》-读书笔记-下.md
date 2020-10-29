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

### 6.3.1 什么是阻塞队列

### 6.3.2 Java里的阻塞队列

### 6.3.3 阻塞队列的实现原理

## 6.4 Fork/Join框架

### 6.4.1 什么是Fork/Join框架

### 6.4.2 工作窃取算法

### 6.4.3 Fork/Join框架的设计

### 6.4.4 使用Fork/Join框架

### 6.4.5 Fork/Join框架的异常处理

### 6.4.6 Fork/Join框架的实现原理

## 6.5 本章小结

# 第7章 Java中的13个原子操作类

# 第8章 Java中的并发工具类

# 第9章 Java中的线程池

# 第10章 Executor框架

# 第11章 Java并发编程实战

