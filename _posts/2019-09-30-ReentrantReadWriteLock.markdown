---
layout: post
category: "juc"
title:  "ReentrantReadWriteLock源码解析"
tags: [lock,JUC]
---
## ReentrantReadWriteLock源码解析

ReentrantReadWriteLock是Java上关于读写锁的实现
首先，先让我们熟悉下读写锁的功能，再来看读写锁的源码
读锁：可以被多个线程同时反问，多个线程可以同时拥有写锁
写锁：具有排他性，只允许一个线程拥有写锁
使用范例
```
 * class RWDictionary {
 *   private final Map<String, Data> m = new TreeMap<String, Data>();
 *   private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 *   private final Lock r = rwl.readLock();
 *   private final Lock w = rwl.writeLock();
 *
 *   public Data get(String key) {
 *     r.lock();
 *     try { return m.get(key); }
 *     finally { r.unlock(); }
 *   }
 *   public String[] allKeys() {
 *     r.lock();
 *     try { return m.keySet().toArray(); }
 *     finally { r.unlock(); }
 *   }
 *   public Data put(String key, Data value) {
 *     w.lock();
 *     try { return m.put(key, value); }
 *     finally { w.unlock(); }
 *   }
 *   public void clear() {
 *     w.lock();
 *     try { m.clear(); }
 *     finally { w.unlock(); }
 *   }
 * }
```

{% highlight java %} 

public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;
    //内部读锁
    private final ReentrantReadWriteLock.ReadLock readerLock;
    //内部写锁
    private final ReentrantReadWriteLock.WriteLock writerLock;
    //一个AQS同步对象
    final Sync sync;

    public ReentrantReadWriteLock() {
        this(false);
    }

    //它也具有公平/非公平的实现方式
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

    /**
    熟悉的AQS子类，所以这里也就是我们要理解的重点啦
    */
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 6317671515068378041L;

        /**
        SHARED_SHIFT   位的长度
        SHARED_UNIT    1左移16位=65536
        MAX_COUNT      65535 0x0000ffff 共享锁线程最大个数
        EXCLUSIVE_MASK 65535 排他锁的掩码
        */

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        //这里要先声明一下概念，读写锁采用一个state 但分高16位和低16位来表示读写锁的数量
 		/**  让c右移16位，这样就留下了高16位，高16位的值就是读锁的数量了  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** 让c和0x0000ffff做与运算，这样就保留下低16位，保留下的值就是写锁数量啦 */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

        static final class HoldCounter {
            int count = 0;
            final long tid = getThreadId(Thread.currentThread());
        }

        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }

        /**
        * 读锁的数量
        */
        private transient ThreadLocalHoldCounter readHolds;

        /**
        * 最新获取读锁的线程重入次数
        */
        private transient HoldCounter cachedHoldCounter;

        /**
         * 第一个获取读锁的线程
         */
        private transient Thread firstReader = null;
        private transient int firstReaderHoldCount;

        Sync() {
            readHolds = new ThreadLocalHoldCounter();
            /**
            * 这里需要扩展下volatile内存的语意
            * 当写一个 volatile 变量时，JMM 会把该线程对应的本地内存中的共享变量值刷新到主内存。
         	* 当读一个 volatile 变量时，JMM 会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。
         	* 所以这里利用一个setState 来达到ensures visibility of readHolds
            */
            setState(getState()); // ensures visibility of readHolds
        }

        /**
         * 当前持有读锁线程返回true，否者需要阻塞
         */
        abstract boolean readerShouldBlock();

        /**
         * 当前持有写锁线程返回true，否者需要阻塞
         */
        abstract boolean writerShouldBlock();

        /**
        * 释放资源，不存在竞争，所以大家看到代码比较简单。
        * 主要是针对写锁
        */
        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = 	exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }

        /**
        * 写线程尝试去获取资源
        */
        protected final boolean tryAcquire(int acquires) {
            
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);//写锁数量
            if (c != 0) { //有读锁或者写锁
                //写锁数量为0，那么说明存在读锁
                //写锁数量不为0，当前线程不是拥有锁的线程
                //直接返回 false
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //判断数量越界
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 重入的设置state
                setState(c + acquires);
                return true;
            }
            //写线程等待别人放开，然后用cas去设置state，如果成功设置持有锁线程
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }

        /**
        * 放开读锁资源
        * 先对当前线程拥有资源数量做一个数据预处理
        * 然后在for循环中 进行CAS
        */
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {//当前线程是第一个获取读锁的线程
                // 第一个读锁的线程获取的次数
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;//最新获取读锁的拥有资源的数量
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();//赋值为当前线程拥有资源的数量
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();//拿到state
                int nextc = c - SHARED_UNIT;//减掉65535的读锁占位数+1个资源位
                if (compareAndSetState(c, nextc))
                    //是否放光
                    return nextc == 0;
            }
        }

        private IllegalMonitorStateException unmatchedUnlockException() {
            return new IllegalMonitorStateException(
                "attempt to unlock read lock, not locked by current thread");
        }

        /**
        * 获取读锁资源
        */
        protected final int tryAcquireShared(int unused) {
            
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)//存在读锁，直接失败
                return -1;
            int r = sharedCount(c);//获取读锁数量

            /**
            * if里面包含了很多逻辑
            * 先用readerShouldBlock 如果存在竞争阻塞一下
            * 然后cas的进行读锁数量+1
            */
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {//还没读锁成功过，第一次赋值操作
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {//当前线程是第一次获取读锁的线程
                    firstReaderHoldCount++;
                } else {
                	//为了把当前持有资源数量++
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            //这里是为了保证cas失败啥的情况，进入fullTryAcquireShared
            return fullTryAcquireShared(current);
        }

        /**
        * 这个方法是通过for循环，来保证CAS一定会成功
        */
        final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }

        /**
         * 写锁的tryLock方法
         * 排他的写锁就比较好理解了，
         * 判断当前是否是拥有锁的线程
         * 然后进行一次cas
         */
        final boolean tryWriteLock() {
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {
                int w = exclusiveCount(c);
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }

        /**
         * 读线程的tryLock方法
         */
        final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)//存在写锁就结束
                    return false;
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {//进行一次cas+1
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                    	//从ThreadLocal里拿当前线程持有数量，然后+1
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }

        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        final Thread getOwner() {
            return ((exclusiveCount(getState()) == 0) ?
                    null :
                    getExclusiveOwnerThread());
        }

        final int getReadLockCount() {
            return sharedCount(getState());
        }

        final boolean isWriteLocked() {
            return exclusiveCount(getState()) != 0;
        }

        final int getWriteHoldCount() {
            return isHeldExclusively() ? exclusiveCount(getState()) : 0;
        }

        final int getReadHoldCount() {
            if (getReadLockCount() == 0)
                return 0;

            Thread current = Thread.currentThread();
            if (firstReader == current)
                return firstReaderHoldCount;

            HoldCounter rh = cachedHoldCounter;
            if (rh != null && rh.tid == getThreadId(current))
                return rh.count;

            int count = readHolds.get().count;
            if (count == 0) readHolds.remove();
            return count;
        }

        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            readHolds = new ThreadLocalHoldCounter();
            setState(0); // reset to unlocked state
        }

        final int getCount() { return getState(); }
    }

    /**
     * 非公平的实现Sync
     */
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
        final boolean readerShouldBlock() {
           // 判断读是否阻塞的实现
            return apparentlyFirstQueuedIsExclusive();
        }
    }

    /**
     * 公平的实现Sync
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
        	//队列是否有元素等待
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
        	//队列是否有元素等待
            return hasQueuedPredecessors();
        }
    }

    /**
     * 读锁
     */
    public static class ReadLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -5992448646407690164L;
        private final Sync sync;

        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        /**
         * 语意上就都好理解
         * 具体还是在读锁的Sync的tryAcquire实现上
         */
        public void lock() {
            sync.acquireShared(1);
        }

        /**
         * 响应中断的实现
         */
        public void lockInterruptibly() throws InterruptedException {
            sync.acquireSharedInterruptibly(1);
        }

        /**
         * 详看tryReadLock
         */
        public boolean tryLock() {
            return sync.tryReadLock();
        }

        /**
         * 带有超时的tryLock
         */
        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
        }

        /**
         * 释放lock
         */
        public void unlock() {
            sync.releaseShared(1);
        }

        public Condition newCondition() {
            throw new UnsupportedOperationException();
        }

        public String toString() {
            int r = sync.getReadLockCount();
            return super.toString() +
                "[Read locks = " + r + "]";
        }
    }

    /**
     * 写锁
     */
    public static class WriteLock implements Lock, java.io.Serializable {
        private static final long serialVersionUID = -4992448646407690164L;
        private final Sync sync;

        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        /**
         * 具体看写锁实现的tryAcquire方法的区别
         */
        public void lock() {
            sync.acquire(1);
        }

        /**
         * 带响应中断的
         */
        public void lockInterruptibly() throws InterruptedException {
            sync.acquireInterruptibly(1);
        }

        /**
         * tryWriteLock调用
         */
        public boolean tryLock( ) {
            return sync.tryWriteLock();
        }

        /**
         * 带超时时间的tryLock
         */
        public boolean tryLock(long timeout, TimeUnit unit)
                throws InterruptedException {
            return sync.tryAcquireNanos(1, unit.toNanos(timeout));
        }

        /**
         * 释放
         */
        public void unlock() {
            sync.release(1);
        }

        /**
         * Condition对象
         */
        public Condition newCondition() {
            return sync.newCondition();
        }

        public String toString() {
            Thread o = sync.getOwner();
            return super.toString() + ((o == null) ?
                                       "[Unlocked]" :
                                       "[Locked by thread " + o.getName() + "]");
        }

        public boolean isHeldByCurrentThread() {
            return sync.isHeldExclusively();
        }

        public int getHoldCount() {
            return sync.getWriteHoldCount();
        }
    }

    //是否公平
    public final boolean isFair() {
        return sync instanceof FairSync;
    }

    /**
     * 锁对象拥有者线程
     */
    protected Thread getOwner() {
        return sync.getOwner();
    }

    /**
     * 读锁数量
     */
    public int getReadLockCount() {
        return sync.getReadLockCount();
    }

    /**
     * 有没有写锁
     */
    public boolean isWriteLocked() {
        return sync.isWriteLocked();
    }

    /**
     * 是不是当前线程有写锁
     */
    public boolean isWriteLockedByCurrentThread() {
        return sync.isHeldExclusively();
    }

    /**
     * 写锁获取资源的次数，就是重入次数
     */
    public int getWriteHoldCount() {
        return sync.getWriteHoldCount();
    }

    /**
     * 读锁重入次数
     */
    public int getReadHoldCount() {
        return sync.getReadHoldCount();
    }

    protected Collection<Thread> getQueuedWriterThreads() {
        return sync.getExclusiveQueuedThreads();
    }

    protected Collection<Thread> getQueuedReaderThreads() {
        return sync.getSharedQueuedThreads();
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
        int c = sync.getCount();
        int w = Sync.exclusiveCount(c);
        int r = Sync.sharedCount(c);

        return super.toString() +
            "[Write locks = " + w + ", Read locks = " + r + "]";
    }

    static final long getThreadId(Thread thread) {
        return UNSAFE.getLongVolatile(thread, TID_OFFSET);
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long TID_OFFSET;//tid距当前对象头地址的offset
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> tk = Thread.class;
            TID_OFFSET = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("tid"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

}

{% highlight %} 

通过看读写锁的实现，把握核心的state被拆成前16位和后16位，再通过CAS操作，就很好理解了。
