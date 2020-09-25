# CountDownLatch

说完了ReentrantLock后再来看看同样适用了同步基础框架AQS的CountDownLatch。这个依然是借鉴了极客时间争哥的研发理论，测试驱动开发，那我们就通过高层实现去理解底层原理，通过不同的组件去揭开AQS的神秘面纱。

CountDownLatch的英文注释是：A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

翻译：一种同步辅助，允许一个或多个线程等待直到在其他线程中执行的一组操作完成。

首先CountDownLatch内部维护了一个被关键字`final`修饰的私有静态内部类Sync

```java
/**
     * Synchronization control For CountDownLatch.
     * Uses AQS state to represent count.
     * 私有，静态，不能被继承，非抽象，可实例化
     */
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
    //通过构造函数，实例化一个Sync对象并设置同步器状态值
    Sync(int count) {
        setState(count);
    }

    //查询同步器状态值
    int getCount() {
        return getState();
    }

    //尝试获取共享锁
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
	
    //尝试释放共享锁
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

上面的Sync有些简单，总共四个方法（见上面的代码），我们接着看CountDownLatch是如何利用这4个方法来完成同步辅助

```java
/**
     * Constructs a {@code CountDownLatch} initialized with the given count.
     *
     * @param count the number of times {@link #countDown} must be invoked
     *        before threads can pass through {@link #await}
     * @throws IllegalArgumentException if {@code count} is negative
     */
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

/**
     * Causes the current thread to wait until the latch has counted down to
     * zero, unless the thread is {@linkplain Thread#interrupt interrupted}.
     *
     * <p>If the current count is zero then this method returns immediately.
     *
     * <p>If the current count is greater than zero then the current
     * thread becomes disabled for thread scheduling purposes and lies
     * dormant until one of two things happen:
     * <ul>
     * <li>The count reaches zero due to invocations of the
     * {@link #countDown} method; or
     * <li>Some other thread {@linkplain Thread#interrupt interrupts}
     * the current thread.
     * </ul>
     *
     * <p>If the current thread:
     * <ul>
     * <li>has its interrupted status set on entry to this method; or
     * <li>is {@linkplain Thread#interrupt interrupted} while waiting,
     * </ul>
     * then {@link InterruptedException} is thrown and the current thread's
     * interrupted status is cleared.
     *
     * @throws InterruptedException if the current thread is interrupted
     *         while waiting
     */
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

 /**
     * Decrements the count of the latch, releasing all waiting threads if
     * the count reaches zero.
     *
     * <p>If the current count is greater than zero then it is decremented.
     * If the new count is zero then all waiting threads are re-enabled for
     * thread scheduling purposes.
     *
     * <p>If the current count equals zero then nothing happens.
     * AQS同步器状态减一，若状态为零，唤醒所有等待线程
     */
public void countDown() {
    sync.releaseShared(1);
}



```

