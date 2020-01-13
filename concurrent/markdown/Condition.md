# Condition详解

## 简介

在Java程序中，任意一个Java对象，都拥有一组监视器方法（定义在Object类上），主要包括wait(),wait(long),notify(),notifyAll()方法，这些方法与synchronized关键字配合，可以实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等待/通知模式，但是这两者在使用方式以及功能特性上还是有区别的。Object的监视器与Condition接口对比如下

| 对比项                                               | Object监视器方法   | Condition                                                    |
| ---------------------------------------------------- | ------------------ | ------------------------------------------------------------ |
| 前置条件                                             | 获取对象的监视器锁 | 调用Lock.lock()获取锁<br />调用Lock.newCondition()获取Condition对象 |
| 调用方法                                             | object.wait()      | condition.await()                                            |
| 等待队列个数                                         | 一个               | 多个                                                         |
| 当前线程释放锁并进入等待队列                         | 支持               | 支持                                                         |
| 当前线程释放锁并进入等待队列，在等待状态中不响应中断 | 不支持             | 支持                                                         |
| 当前线程释放锁并进入超时等待状态                     | 支持               | 支持                                                         |
| 当前线程释放锁并进入等待状态到将来的某个时间         | 不支持             | 支持                                                         |
| 唤醒等待队列中的一个线程                             | 支持               | 支持                                                         |
| 唤醒等待队列中的全部线程                             | 支持               | 支持                                                         |

Condition提供了一系列的方法来阻塞和唤醒线程：

```java
public interface Condition {
    void await() throws InterruptedException;
    void awaitUninterruptibly();
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
    void signal();
    void signalAll();
}
```

Condition的方法以及描述如下：

| 方法名称                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| void await() throws InterruptedException                     | 当前线程进入等待状态直到被通知（signal）或中断。             |
| void awaitUninterruptibly()                                  | 当前线程进入等待状态直到被通知，该方法不响应中断。           |
| long awaitNanos(long nanosTimeout) throws InterruptedException | 当前线程进入等待状态直到被通知、中断或者超时，返回值表示剩余超时时间。 |
| boolean awaitUntil(Date deadline) throws InterruptedException | 当前线程进入等待状态直到被通知、中断或者到某个时间。如果没有到指定时间就被通知，方法返回true，否则，表示到了指定时间，返回false。 |
| void signal()                                                | 唤醒一个等待在Condition上的线程，该线程从等待方法返回前必须获得与Condition相关联的锁。 |
| void signalAll()                                             | 唤醒所有等待在Condition上的线程，能够从等待方法返回的线程必须获得与Condition相关联的锁。 |

## Condition的实现分析

可以通过Lock.newCondition()方法获取Condition对象，而我们知道Lock对于同步状态的实现都是通过内部的自定义同步器实现的，newCondition()方法也不例外，所以，Condition接口的唯一实现类是同步器AQS的内部类ConditionObject，因为Condition的操作需要获取相关的锁，所以作为同步器的内部类也比较合理，该类定义如下：

```java
public class ConditionObject implements Condition, java.io.Serializable
```

每个Condition对象都包含着一个队列（以下称为等待队列），该队列是Condition对象实现等待/通知功能的关键。

### 等待队列

等待队列是一个FIFO队列，在队列中的每个节点都包含了一个线程引用，该线程就是在Condition对象上等待的线程，如果一个线程调用了Condition.await()方法，那么该线程将会释放锁、构造成节点加入等待队列并进入等待状态。事实上，节点的定义复用了AQS中Node节点的定义，也就是说，同步队列和等待队列中节点类型都是AQS的静态内部类AbstractQueuedSynchronized.Node。
一个Condition包含一个等待队列，Condition拥有首节点（firstWaiter）和尾节点（lastWaiter）。当前线程调用Condition.await()方法之后，将会以当前线程构造节点，并将节点从尾部加入等待队列，等待队列的基本结构如下图所示：

![](Condition.assets/Condition.png)

Condition拥有首尾节点的引用，而新增节点只需要将原来的尾节点nextWaiter指向它，并且更新尾节点即可。上述更新过程不需要CAS保证，原因在于调用await()方法的线程必定是获取了锁的线程，也就是说该过程是由锁来保证线程安全的。

在Object的监视器模型上，一个对象拥有一个同步队列和等待队列，而并发包中的Lock（更确切地说是同步器）拥有一个同步队列和多个等待队列，其对应关系如下图所示：

![](Condition.assets/AQS.png)

### 等待

调用Condition的await()方法会使当前线程进入等待状态，同时线程状态变为等待状态，当从await()方法返回时，当前线程一定获取了Condition相关联的锁。

如果从队列（同步队列和等待队列）的角度看await()方法，当调用await()方法时，相当于同步队列的首节点（获取了锁的节点）移动到Condition的等待队列中。

```java
public final void await() throws InterruptedException {
    // 检测线程中断状态
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程包装为Node节点加入等待队列
    Node node = addConditionWaiter();
    // 释放同步状态，也就是释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 检测该节点是否在同步队中，如果不在，则说明该线程还不具备竞争锁的资格，则继续等待
    while (!isOnSyncQueue(node)) {
        // 挂起线程
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 竞争同步状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 清理条件队列中的不是在等待条件的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

调用该方法的线程是成功获取了锁的线程，也就是同步队列中的首节点，该方法会将当前线程构造节点并加入等待队列中，然后释放同步状态，唤醒同步队列中的后继节点，然后当前线程会进入等待状态。

加入等待队列是通过addConditionWaiter()方法来完成的：

```java
private Node addConditionWaiter() {
    // 尾节点
    Node t = lastWaiter;
    // 尾节点如果不是CONDITION状态，则表示该节点不处于等待状态，需要清理节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 根据当前线程创建Node节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 将该节点加入等待队列的末尾
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

如果从队列的角度看，当前线程加入到Condition的等待队列，如下图所示：

![](Condition.assets/AQS2.png)

将当前线程加入到等待队列之后，需要释放同步状态，该操作通过fullyRelease(Node)方法来完成:

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取同步状态
        int savedState = getState();
        // 释放锁
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

将当前线程加入到等待队列之后，需要释放同步状态，该操作通过fullyRelease(Node)方法来完成：

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取同步状态
        int savedState = getState();
        // 释放锁
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

线程释放锁之后，我们需要通过isOnSyncQueue(Node)方法不断自省地检查其对应节点是否在同步队列中：

```java
final boolean isOnSyncQueue(Node node) {
    // 节点状态为CONDITION，或者前驱节点为null，返回false
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 后继节点不为null，那么肯定在同步队列中
    if (node.next != null) // If has successor, it must be on queue
        return true;
    
    return findNodeFromTail(node);
}
```

若节点不在同步队列中，则挂起当前线程，若线程在同步队列中，且获取了同步状态，可能会调用unlinkCancelledWaiters()方法来清理等待队列中不为CONDITION 状态的节点：

```java
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
```



### 通知

调用Condition的signal()方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点移到同步队列中。Condition的signal()方法如下所示：

```java
public final void signal() {
    // 判断是否是当前线程获取了锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 唤醒等待队列的首节点
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

该方法最终调用doSignal(Node)方法来唤醒节点：

```java
private void doSignal(Node first) {
    do {
        // 把等待队列的首节点移除之后，要修改首结点
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}
```

将节点移动到同步队列是通过transferForSignal(Node)方法完成的：

```java
final boolean transferForSignal(Node node) {
    // 尝试将该节点的状态从CONDITION修改为0
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    
    // 将节点加入到同步队列尾部，返回该节点的前驱节点
    Node p = enq(node);
    int ws = p.waitStatus;
    // 如果前驱节点的状态为CANCEL或者修改waitStatus失败，则直接唤醒当前线程
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

节点从等待队列移动到同步队列的过程如下图所示：

![](Condition.assets/AQS-signal.png)

被唤醒后的线程，将从await()方法中的while循环中退出（因为此时isOnSyncQueue(Node)方法返回true），进而调用acquireQueued()方法加入到获取同步状态的竞争中。

成功获取了锁之后，被唤醒的线程将从先前调用的await()方法返回，此时，该线程已经成功获取了锁。

Condition的signalAll()方法，相当于对等待队列的每个节点均执行一次signal()方法，效果就是将等待队列中的所有节点移动到同步队列中。
