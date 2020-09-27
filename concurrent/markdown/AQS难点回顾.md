# AQS源码分析

# 先看看共享锁的获取流程



> Sets head of queue, and checks if successor may be waiting in shared mode, if so propagating if either propagate > 0 or PROPAGATE status was set.

此函数被共享锁操作而使用。这个函数用来将传入参数设为队列的新节点，如果传参的后继是共享模式且现在要么 共享锁有剩余（propagate > 0） 要么 PROPAGATE状态被设置，那么调用doReleaseShared。

当调用doAcquireShared

```java
 /**
     * Acquires in shared uninterruptible mode.
     * @param arg the acquire argument
     */
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
                    p.next = null; // help GC
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

- 执行doAcquireShared的当前线程想要获取到共享锁
- addWaiter将当前线程包装成一个共享模式的node放到队尾去。

for循环的过程分析

- 执行到tryAcquireShared后可能有两种情况
  - 如果tryAcquired返回值 >= 0，说明线程获取共享锁成功了，那么调用setHeadAndPropagate，然后函数返回
  - 如果tryAcquired返回值 < 0，说明线程获取共享锁失败，调用shouldParkAfterFailedAcquire。
    - 这个shouldParkAfterFailAcquire一般来说，得至少执行两遍才能返回true：第一次吧前驱设置为SIGNAL状态，第二次检测代SIGNAL才返回true
    - 也有可能第二遍的时候，发现自己的前驱突然变成了head并且获取共享锁成功，又或者本来第一遍的前驱就是head单第二遍获取共享锁成功了。设置SIGNAL的目的是为了唤醒自己。
    - 线程在执行了parkAndCheckInterrupt后，便阻塞了，之后智能等别人唤醒自己，以后自己被唤醒了，也会去唤醒别人

## setHeadAndPropagate分析

```java
/**
     * Sets head of queue, and checks if successor may be waiting
     * in shared mode, if so propagating if either propagate > 0 or
     * PROPAGATE status was set.
     *
     * @param node the node
     * @param propagate the return value from a tryAcquireShared
     */
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```



## 再看看doReleaseShared

doReleaseShared方法比较晦涩难懂，在自己抓破脑袋想了半天之后依然看不出其门道，遂上网看帖子，翻了几篇之后，发现有一个无名大神的分析总结甚合孤意，他的**唤醒风暴**的说法对我来说，好比醍醐灌顶，如梦方醒

先上源码再加注释

```java
/**
     * Release action for shared mode -- signals successor and ensures
     * propagation. (Note: For exclusive mode, release just amounts
     * to calling unparkSuccessor of head if it needs signal.)
	 * 翻译：对于共享锁的Release动作——唤醒后继并且确保传播。相对于 独占锁来说，相对应的函数就是unparkSuccessor。
     */
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                //在队列中的节点对应的线程阻塞之前，将前驱节点的waitStatus状态设置为SIGNAL
                //所以这里ws == 0，其实是当前线程通过第一次循环将状态设置为0
                //第二次循环进入的时候头结点还没有被改变
                //cas操作失败的化会直接continue，为什么会失败
                //可能是唤醒的其他节点在唤醒后续节点的时候已经进行了修改
                //修改失败前，头结点已经修改，进入下一次循环
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

## 使用场景

`doReleaseShared`的作用唤醒其后后继节点，具体的说是需要唤醒到下一个尝试获取锁的节点之间所有尝试获取共享锁的线程

在`AQS`中一共有两处使用到了`doReleaseShared()`方法，分别是

- 在`setHeadAndPropagate()`中，改方法用于同步等待队列中获取共享锁的节点在成功获取共享锁之后判断其是否有后继节点，以及后继节点是否是尝试获取共享锁，如果有则调用`doReleaseShared()`方法唤醒后继线程
- 在`releaseShared()`中当前线程释放完度锁后，同步器状态归0则调用`doReleaseShared()`方法唤醒后继节点

总的来说，该方法就是用来唤醒后继节点的，但是这个方法是一个死循环，而出口条件却不是很好理解

```java
if (h == head)                   // loop if head changed
    break;
```

## 唤醒风暴

假设有这样一个队列

![](D:\study\kkb_workspace\daily_notes\concurrent\markdown\AQS回顾.assets\未命名文件.png)

这时分两种情况

**共享锁1判断头节点之前，共享锁2替换头节点成功**

共享锁线程1会在这个循环里不能退出，第二次循环的时候h字段会变成曾经共享锁2对应的节点

- 共享锁2此时是被唤醒的，共享锁2也会调用`setHeadAndPropagate()`方法去唤醒共享锁3线程。假设是共享锁2唤醒了共享锁3，共享锁3线程会将头结点设置为自身节点，而共享锁1线程的`h`字段保存的头结点还没更改依然是共享锁2曾经的节点，CAS更换头结点的waitStatus状态操作会失败，进入下一次循环
- 假设共享锁3对应的线程由共享锁2唤醒，共享锁3完成了设置头结点的操作，此时共享锁1刚好进入又一次循环且没有竞争，那么共享锁1可以立刻唤醒共享锁4

假设队列足够长，那么久会产生一个唤醒的风暴，前面的线程都在唤醒后面的线程，这样可以快速的唤醒起队列中下一个排它锁之前的所有申请共享锁的线程

这样的风暴会在碰到一个申请排他锁的线程或者一直到队列尾部都没有排他锁，唤醒了所有的线程之后结束，当然中间一部分线程可能已经结束了唤醒操作（在判断h == head 之前，头结点没有被替换）

- 碰到排它锁：由于共享锁已经被获取，唤醒一个排他锁后，发现头节点是排他锁，所以不会替换头结点，唤醒风暴结束。
- 到达队尾，风暴结束。

**在排他锁1判断头结点完成之前，头结点没有被替换**

线程1退出循环，但是此时线程2已经入坑



## 总结

`doReleaseShared()`方法会以一种风暴的形式唤醒后续排它锁之前的所有节点