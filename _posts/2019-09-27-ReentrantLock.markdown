---
layout: post
category: "juc"
title:  "ReentrantLock源码解析"
tags: [JUC,lock]
---
## ReentrantLock源码解析
ReentrantLock是具有排他性质的可重入锁，提供公平/非公平的策略。
使用的范例如下
```
 * class X {
 *   private final ReentrantLock lock = new ReentrantLock();
 *   // ...
 *
 *   public void m() {
 *     lock.lock();  // block until condition holds
 *     try {
 *       // ... method body
 *     } finally {
 *       lock.unlock()
 *     }
 *   }
 * }
```
下面进入源码解析啦

{% highlight java %} 

public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Sync 这类实现了所有的同步机制*/
    private final Sync sync;

    /**
     * 基于AQS实现的同步工具
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * 与Lock接口的lock()方法相同的实现。具体看子类的实现，公平/非公平的区别
         */
        abstract void lock();

        /**
         * 非公平的TryAcquire实现，在NonfairSync中调用
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();//获取当前的状态
            if (c == 0) {//没有竞争
            	/**
            	* 通过CAS修改state值，并且设置当前拥有锁的线程
            	*/
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            /**
            如果是已经拥有锁的线程，再次进来，就直接增加state
            */
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
        释放资源，减少state数量，直到0释放成功
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

        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); 
        }
    }

    /**
     * NonfairSync继承自Sync，提供非公平的实现
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
        首先利用CAS去设置state，如果设置成功了就获取到锁了。
        否者就用AQS的acquire方法去参于竞争
        */
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        /**
        为什么说非公平，就表现在，当线程要来参与紧张，我不需要管AQS等待队列中有没有线程等住了
        */
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

    /**
     * 公平的Sync
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        /**
        于NonfairSync的差别就是
        有没有先进行一次cas
        if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
        */
        final void lock() {
            acquire(1);
        }

        /**
         * tryAcquire的公平实现版本
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
            	//hasQueuedPredecessors这个方法先判断有没有线程已经在AQS等待队列中了
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

    /**
    默认非公平的实现
    */
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

    /**
     * 去获取锁，由sync具体实现
     */
    public void lock() {
        sync.lock();
    }

    /**
    lock方法，接受Thread#interrupt的中断，抛InterruptedException
    */
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

   	/**
    如果当前锁没有被占用，返回成功
    如果获取失败，直接返回失败，没有入队操作
   	*/
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    /**
    参考AQS中的tryAcquireNanos源码
    进行一次获取锁，如果失败，进行入队，然后休眠一段timeout时间
    就是一个带有超时时间的lock方法
    */
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    //释放资源
    public void unlock() {
        sync.release(1);
    }

    /**
    调用了Sync向外提供的方法
    */

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

    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }


    public final boolean hasQueuedThread(Thread thread) {
        return sync.isQueued(thread);
    }

    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }

    public boolean hasWaiters(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    protected Collection<Thread> getWaitingThreads(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }
}

{% endhighlight %}

所以ReentrantLock整个执行方法还是比较清晰，加锁都是利用了state字段的+1的原子操作来控制。
