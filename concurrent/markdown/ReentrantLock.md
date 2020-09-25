# ReentrantLock

研究AQS为什么要从ReentrantLock说起呢，这个借鉴了极客时间的前谷歌资深工程师争哥关于测试驱动开发的理论，我们先看看优秀的框架时怎么运用AQS的，通过不同的组件去揭开AQS的神秘面纱（不同的组件通过AQS完成了不同的功能）。

首先ReentrantLock内部维护了一个抽象内部类Sync，看下代码和注释

```java
//抽象类不能被实例化，可以有抽象方法
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         * 抽象方法，子类必须实现，在ReentrantLock中体现为公平锁和非公平锁
         */
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         * 该方法被tryLock调用，而tryLock的含义是在没有其他线程持有锁的时候获取锁
         * 该动作不会涉及到入队出队（AQS内部维护的Node队列）
         * 注：tryLock是非公平锁的获取方式，lock阻塞等待，tryLock则直接返回
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

    	/**
    	 * 释放锁，若调用线程不是当前持有锁的线程抛出异常
    	 * 因为该组件为重入锁，当持有线程完全释放锁之后返回true
    	 * 否则状态值-1，返回false
    	 */
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
		
    	/**
    	 * 判断当前线程释放持有锁
    	 */
        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
		/**
		 * 创建一个Condition队列，至于Condition队列后面再聊
		 */
        final ConditionObject newCondition() {
            return new ConditionObject();
        }

		//获取锁持有者（线程）
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }
		
    	//获取当前线程获得锁的次数
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }
		//锁是否被占用
        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         * 这个方法可以单独总结
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
```

通过认真分析上面内部内的代码和注释已经有一定的理解，现在继续看Sync的子类

NonfairSync

```java
 /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         * 非公平锁的获取锁方式：直接尝试修改AQS状态（通过compareAndSetState），若修改成功
         * 说明获取锁成功，否则调用acquire(1)方法，在调用acquire方法时，会调用nonfairTryAcquire
         * 参考下面一个方法，所以由此得出，在非公平锁当中，获取锁（争夺锁）的过程与AQS内部维护的队列无关，实际上
         * 在非公平锁中也存在一个等待队列，当线程没有枪占锁成功时，会加入队列中，加入队列之后就和公平锁没什么区别了
         * 但是，当前锁释放后，排在等待队列的第一线程也不一定能获取锁，这是因为非公平锁在抢占锁的时候没有考虑等待队列的优先性
         * 举例：当前锁释放，会唤醒等待队列中的第一个线程，但是此时可能有新的线程来抢夺。。。
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```

Sync

```java
/**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        /**
         * 与非公平锁的区别是，不会立马去尝试获取锁，而是直接走流程去tryAcquire
         acquir方法是先去tryAcquire，没有获取到，进入等待队列并休眠
         */
        final void lock() {
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         * 翻译作者的注释：公平版的tryAcquire，只有在当前线程是等待队列中第一个，或者等待队列为空时才
         * 返回true
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //自己是不是头，队列为不为空
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

三个内部类看完了（实际上是两个，公平锁和非公平锁继承与Sync）

看看ReentrantLockde的源码

```java
/**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
public ReentrantLock() {
    sync = new NonfairSync();
}
/**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     * 默认非公平锁
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
//调用内部实现的lock，公平方式/非公平方式
public void lock() {
    sync.lock();
}
//相应中断的获取锁，在这里调用的是AQS的acquireInterruptibly(1)方法
//以互斥方式获取，如果被中断则中止。
public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
}
//Acquires the lock only if it is not held by another thread at the time of invocation.
//非公平的调用方式
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}

//Acquires the lock if it is not held by another thread within the given waiting time and the current thread has not been interrupted
public boolean tryLock(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}

//调用AQS的释放锁，释放成功之后，唤醒队列中第一个线程
public void unlock() {
    sync.release(1);
}
//生成一个Condition队列
public Condition newCondition() {
    return sync.newCondition();
}

public int getHoldCount() {
    return sync.getHoldCount();
}

public boolean isHeldByCurrentThread() {
    return sync.isHeldExclusively();
}

public boolean isLocked() {
    return sync.isLocked();
}

public final boolean isFair() {
    return sync instanceof FairSync;
}

protected Thread getOwner() {
    return sync.getOwner();
}
//是否有同步队列
public final boolean hasQueuedThreads() {
    return sync.hasQueuedThreads();
}
//当前线程是否在同步队列中
public final boolean hasQueuedThread(Thread thread) {
    return sync.isQueued(thread);
}

//获取同步队列长度
public final int getQueueLength() {
    return sync.getQueueLength();
}

//获取同步队列线程集合
protected Collection<Thread> getQueuedThreads() {
    return sync.getQueuedThreads();
}

//Queries whether any threads are waiting on the given condition associated with this lock. Note that because timeouts and interrupts may occur at any time, a {@code true} return does not guarantee that a future {@code signal} will awaken any threads.  This method is designed primarily for use in monitoring of the system state.
public boolean hasWaiters(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
}

 /**
     * Returns an estimate of the number of threads waiting on the
     * given condition associated with this lock. Note that because
     * timeouts and interrupts may occur at any time, the estimate
     * serves only as an upper bound on the actual number of waiters.
     * This method is designed for use in monitoring of the system
     * state, not for synchronization control.
     */
public int getWaitQueueLength(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
}

/**
     * Returns a collection containing those threads that may be
     * waiting on the given condition associated with this lock.
     * Because the actual set of threads may change dynamically while
     * constructing this result, the returned collection is only a
     * best-effort estimate. The elements of the returned collection
     * are in no particular order.  This method is designed to
     * facilitate construction of subclasses that provide more
     * extensive condition monitoring facilities.
     */
protected Collection<Thread> getWaitingThreads(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
}
```

好了，到此，ReentrantLock就已经搞定了，原来如此简单

任何技能都需要不断训练才能熟练掌握