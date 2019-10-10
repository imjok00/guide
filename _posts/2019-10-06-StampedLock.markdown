---
layout: post
category: "juc"
title:  "StampedLock源码解析"
tags: [StampedLock,源码解析]
---
# StampedLock源码解析

StampedLock是在jdk1.8新增进来的类，他的功能也是读写锁，那么在拥有ReentrantReadWriteLock的情况下，为什么还要有新的类呢？
那么让我们带着问题开始接下里的源码分析

先看下这个类的用法
```
 * class Point {
 *   private double x, y;
 *   private final StampedLock sl = new StampedLock();
 *
 *   void move(double deltaX, double deltaY) { // an exclusively locked method
 *     long stamp = sl.writeLock();
 *     try {
 *       x += deltaX;
 *       y += deltaY;
 *     } finally {
 *       sl.unlockWrite(stamp);
 *     }
 *   }
 *
 *   double distanceFromOrigin() { // A read-only method
 *     long stamp = sl.tryOptimisticRead();
 *     double currentX = x, currentY = y;
 *     if (!sl.validate(stamp)) {
 *        stamp = sl.readLock();
 *        try {
 *          currentX = x;
 *          currentY = y;
 *        } finally {
 *           sl.unlockRead(stamp);
 *        }
 *     }
 *     return Math.sqrt(currentX * currentX + currentY * currentY);
 *   }
 *
 *   void moveIfAtOrigin(double newX, double newY) { // upgrade
 *     // Could instead start with optimistic, not read mode
 *     long stamp = sl.readLock();
 *     try {
 *       while (x == 0.0 && y == 0.0) {
 *         long ws = sl.tryConvertToWriteLock(stamp);
 *         if (ws != 0L) {
 *           stamp = ws;
 *           x = newX;
 *           y = newY;
 *           break;
 *         }
 *         else {
 *           sl.unlockRead(stamp);
 *           stamp = sl.writeLock();
 *         }
 *       }
 *     } finally {
 *       sl.unlock(stamp);
 *     }
 *   }
 * }
```
在调用long stamp = sl.writeLock(); long stamp = sl.readLock();时我们看到有个关键字stamp，这个又是什么东西呢？

{% highlight java %} 
public class StampedLock implements java.io.Serializable {
 
    private static final long serialVersionUID = -6001602636862214147L;

    /** cpu的数量 */
    private static final int NCPU = Runtime.getRuntime().availableProcessors();

    /** 入队前最大自选重试次数 */
    private static final int SPINS = (NCPU > 1) ? 1 << 6 : 0;

    /** 队列头节点阻塞前最大自旋重试次数 */
    private static final int HEAD_SPINS = (NCPU > 1) ? 1 << 10 : 0;

    /** 队列头节点重新进入阻塞前最大自旋重试次数 */
    private static final int MAX_HEAD_SPINS = (NCPU > 1) ? 1 << 16 : 0;

    /** 等待自旋到期挂起的时间周期 */
    private static final int OVERFLOW_YIELD_RATE = 7; // must be power 2 - 1

    /** 用于计算线程数量超限制的位数值 */
    private static final int LG_READERS = 7;

    // 用于计算state and stamp的一些常量
    private static final long RUNIT = 1L;                 //1单位的读锁 0000 0001
    private static final long WBIT  = 1L << LG_READERS;   //写锁标志位  1000 0000
    private static final long RBITS = WBIT - 1L;          //溢出保护 0111 1111
    private static final long RFULL = RBITS - 1L;         //读锁最大值  0111 1110
    private static final long ABITS = RBITS | WBIT;       //用于计算读写状态位掩码 1111 1111
    private static final long SBITS = ~RBITS;             // 掩码 24个1+1000 0000

    // 锁状态的初始值
    private static final long ORIGIN = WBIT << 1;

    // INTERRUPT 值
    private static final long INTERRUPTED = 1L;

    // node的两个状态值
    private static final int WAITING   = -1;
    private static final int CANCELLED =  1;

    //node的两个模式，读模式/写模式
    private static final int RMODE = 0;
    private static final int WMODE = 1;

    /** 等待Node节点对象 */
    static final class WNode {
        volatile WNode prev;
        volatile WNode next;
        volatile WNode cowait;    // 读时使用该节点形成栈
        volatile Thread thread;   // non-null while possibly parked
        volatile int status;      // 0, WAITING, or CANCELLED
        final int mode;           // RMODE or WMODE
        WNode(int m, WNode p) { mode = m; prev = p; }
    }

    /** CLH队列头节点 */
    private transient volatile WNode whead;
    /** CLH队列尾节点 */
    private transient volatile WNode wtail;

    /**
    * 这些视图其实是对StamedLock方法的封装，便于习惯了ReentrantReadWriteLock的用户使用：
	* 例如，ReadLockView其实相当于ReentrantReadWriteLock.readLock()返回的读锁;
    */
    transient ReadLockView readLockView;
    transient WriteLockView writeLockView;
    transient ReadWriteLockView readWriteLockView;

    /** 记录锁的state字段 */
    private transient volatile long state;
    /** 读锁溢出以后的暂存字段 */
    private transient int readerOverflow;

    public StampedLock() {
        state = ORIGIN;
    }

    /**
     * 获取写锁
     */
    public long writeLock() {
        long s, next;  // bypass acquireWrite in fully unlocked case only
        /**
        * ((s = state) & ABITS) == 0L 读锁/写锁都没有被占用
        * U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) 高位cas为1
        * cas成功返回next 即stamp
        * cas失败acquireWrite 进入排队
        */
        return ((((s = state) & ABITS) == 0L &&
                 U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
                next : acquireWrite(false, 0L));
    }

    /**
     * 获取写锁-失败不进入排队
     */
    public long tryWriteLock() {
        long s, next;
        return ((((s = state) & ABITS) == 0L &&
                 U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
                next : 0L);
    }

    /**
     * 带有超时时间的获取写锁，一定时间内拿不到 走InterruptedException
     */
    public long tryWriteLock(long time, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(time);
        if (!Thread.interrupted()) {
            long next, deadline;
            if ((next = tryWriteLock()) != 0L)//先尝试获取一次
                return next;
            if (nanos <= 0L)
                return 0L;
            if ((deadline = System.nanoTime() + nanos) == 0L)//计算deadline
                deadline = 1L;
            if ((next = acquireWrite(true, deadline)) != INTERRUPTED)//把deadline传入acquireWrite做等待超时处理
                return next;
        }
        throw new InterruptedException();
    }

    /**
     * 获取写锁被中断抛InterruptedException
     */
    public long writeLockInterruptibly() throws InterruptedException {
        long next;
        if (!Thread.interrupted() &&
            (next = acquireWrite(true, 0L)) != INTERRUPTED)
            return next;
        throw new InterruptedException();
    }

    /**
     * 共享的方法获取读锁
     */
    public long readLock() {
        long s = state, next;  // bypass acquireRead on common uncontended case
        /**
        * whead == wtail 写锁等待队列头尾相同，也就是没有写锁在等待
        * (s & ABITS) < RFULL 读锁数量没有超过最大值RFULL
        * cas来尝试加锁，
        * 如果成功返回最新next值作为stamp
        * 如果失败进入读锁等待队列
        */
        return ((whead == wtail && (s & ABITS) < RFULL &&
                 U.compareAndSwapLong(this, STATE, s, next = s + RUNIT)) ?
                next : acquireRead(false, 0L));
    }

    /**
     * 参数获取读锁，当出现可用时。没有入队的操作
     */
    public long tryReadLock() {
        for (;;) {//这里有个for循环，无限监听
            long s, m, next;
            if ((m = (s = state) & ABITS) == WBIT)//state 的写锁被占用，直接结束
                return 0L;
            else if (m < RFULL) {
                if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))//cas做尝试获取锁
                    return next;
            }
            else if ((next = tryIncReaderOverflow(s)) != 0L)//如果读锁等待超限的情况
                return next;
        }
    }

    /**
     * 带超时时间的获取读锁
     */
    public long tryReadLock(long time, TimeUnit unit)
        throws InterruptedException {
        long s, m, next, deadline;
        long nanos = unit.toNanos(time);
        if (!Thread.interrupted()) {
            if ((m = (s = state) & ABITS) != WBIT) { // 没有写锁占用
                if (m < RFULL) { // 等待未满
                    if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))
                        return next;
                }
                else if ((next = tryIncReaderOverflow(s)) != 0L) //满了进ReaderOverflow
                    return next;
            }
            if (nanos <= 0L)
                return 0L;
            if ((deadline = System.nanoTime() + nanos) == 0L)
                deadline = 1L;
            if ((next = acquireRead(true, deadline)) != INTERRUPTED)//acquireRead入队操作
                return next;
        }
        throw new InterruptedException();
    }

    /**
     * 带中断的读锁获取
     */
    public long readLockInterruptibly() throws InterruptedException {
        long next;
        if (!Thread.interrupted() &&
            (next = acquireRead(true, 0L)) != INTERRUPTED)
            return next;
        throw new InterruptedException();
    }

    /**
     * 存在写锁返回0
     * 否者返回最新的state作为stamp
     */
    public long tryOptimisticRead() {
        long s;
        return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
    }

    /**
     * 验证当前stamp 是持有锁的线程
     */
    public boolean validate(long stamp) {
        U.loadFence();
        return (stamp & SBITS) == (state & SBITS);
    }

    /**
     * stamp匹配的锁释放
     */
    public void unlockWrite(long stamp) {
        WNode h;
        //state != stamp 写锁不是当前占用直接抛IllegalMonitorStateException
        //(stamp & WBIT) == 0L stamp不是写锁获取成功返回的直接抛IllegalMonitorStateException
        if (state != stamp || (stamp & WBIT) == 0L)
            throw new IllegalMonitorStateException();
        //(stamp += WBIT) == 0L 高24位+1，第8位+1后为0也表示释放锁，然后加到溢出就还原回ORIGIN
        state = (stamp += WBIT) == 0L ? ORIGIN : stamp;
        //头节点不为空，状态不为0，唤起下一个等待节点
        if ((h = whead) != null && h.status != 0)
            release(h);
    }

    /**
     * 释放读锁-针对悲观的读锁
     */
    public void unlockRead(long stamp) {
        long s, m; WNode h;
        for (;;) {
        	/**
        	* ((s = state) & SBITS) != (stamp & SBITS) 不匹配版本
        	*  (stamp & ABITS) == 0L || (m = s & ABITS) == 0L || m == WBIT) stamp是获取锁失败的版本
        	* 直接走异常
        	*/
            if (((s = state) & SBITS) != (stamp & SBITS) ||
                (stamp & ABITS) == 0L || (m = s & ABITS) == 0L || m == WBIT)
                throw new IllegalMonitorStateException();
            if (m < RFULL) {
            	//版本-1 回退一个，然后唤醒后续节点
                if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                    if (m == RUNIT && (h = whead) != null && h.status != 0)
                        release(h);
                    break;
                }
            }
            else if (tryDecReaderOverflow(s) != 0L)//ReaderOverflow减1操作
                break;
        }
    }

    /**
    * 能释放读锁或者写锁
    */
    public void unlock(long stamp) {
        long a = stamp & ABITS, m, s; WNode h;
        while (((s = state) & SBITS) == (stamp & SBITS)) {//判断版本一致
            if ((m = s & ABITS) == 0L)//获取锁失败的版本
                break;
            else if (m == WBIT) {//state只有写锁
                if (a != m)//stamp取8位以后 与state不匹配
                    break;
                state = (s += WBIT) == 0L ? ORIGIN : s;//写锁互斥的 就直接高位+1，使写锁释放
                if ((h = whead) != null && h.status != 0)//判断whead不为空，唤醒后续节点
                    release(h);
                return;
            }
            else if (a == 0L || a >= WBIT)//读写锁都没有 或者同时存在
                break;
            else if (m < RFULL) {//只有读锁/并且没满
                if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {//cas成功，唤起后继节点
                    if (m == RUNIT && (h = whead) != null && h.status != 0)
                        release(h);
                    return;
                }
            }
            else if (tryDecReaderOverflow(s) != 0L)//减少ReaderOverflow
                return;
        }
        throw new IllegalMonitorStateException();
    }

    /**
     * 各种情况下转写锁
     */
    public long tryConvertToWriteLock(long stamp) {
        long a = stamp & ABITS, m, s, next;
        while (((s = state) & SBITS) == (stamp & SBITS)) {//判断版本一致
            if ((m = s & ABITS) == 0L) {//m 为state后8位 等于0 没读锁也没写锁
                if (a != 0L)//stamp后8位不为零，那么这种情况是状态的不一致
                    break;
                if (U.compareAndSwapLong(this, STATE, s, next = s + WBIT))//cas成功说明直接从无所加锁到写锁了
                    return next;
            }
            else if (m == WBIT) {//已经存在写锁
                if (a != m)//stamp不匹配，再看看
                    break;
                return stamp;
            }
            else if (m == RUNIT && a != 0L) {//只有一个读锁时，做写锁cas
                if (U.compareAndSwapLong(this, STATE, s,
                                         next = s - RUNIT + WBIT))
                    return next;
            }
            else
                break;
        }
        return 0L;
    }

    /**
    * 转读锁
    */
    public long tryConvertToReadLock(long stamp) {
        long a = stamp & ABITS, m, s, next; WNode h;
        while (((s = state) & SBITS) == (stamp & SBITS)) {//这里无限循环判断版本一致性，一种悲观的实现
            if ((m = s & ABITS) == 0L) {//m 为state后8位 等于0 没读锁也没写锁
                if (a != 0L)//stamp后8位不为零，那么有可能还没同步
                    break;
                else if (m < RFULL) {
                    if (U.compareAndSwapLong(this, STATE, s, next = s + RUNIT))//cas成功说明直接从无所加锁到读锁了
                        return next;
                }
                else if ((next = tryIncReaderOverflow(s)) != 0L)//读锁溢出加到ReaderOverflow上
                    return next;
            }
            else if (m == WBIT) {//存在写锁
                if (a != m)//不匹配，那就出现了state被改？
                    break;
                state = next = s + (WBIT + RUNIT);//读锁高位为0，读锁低位+1
                if ((h = whead) != null && h.status != 0)
                    release(h);//唤起后继节点
                return next;
            }
            else if (a != 0L && a < WBIT)//已经是读锁
                return stamp;
            else
                break;
        }
        return 0L;
    }

    /**
    * 转成乐观读
    */
    public long tryConvertToOptimisticRead(long stamp) {
        long a = stamp & ABITS, m, s, next; WNode h;
        U.loadFence();
        for (;;) {
            if (((s = state) & SBITS) != (stamp & SBITS))//版本不对重试
                break;
            if ((m = s & ABITS) == 0L) {//state上没有锁
                if (a != 0L)
                    break;
                return s;
            }
            else if (m == WBIT) {//存在写锁
                if (a != m)
                    break;
                state = next = (s += WBIT) == 0L ? ORIGIN : s;//直接更新成读锁
                if ((h = whead) != null && h.status != 0)
                    release(h);//唤起后续节点
                return next;
            }
            /**
            * a == 0L 这个 stamp 是没有拿到读锁/写锁的版本
            * a >= WBIT 这个stamp版本上同时存在读/写锁
            */
            else if (a == 0L || a >= WBIT)
                break;
            else if (m < RFULL) {//已经是读锁，没有超限
                if (U.compareAndSwapLong(this, STATE, s, next = s - RUNIT)) {
                    if (m == RUNIT && (h = whead) != null && h.status != 0)
                        release(h);
                    return next & SBITS;
                }
            }
            else if ((next = tryDecReaderOverflow(s)) != 0L)//超限
                return next & SBITS;
        }
        return 0L;
    }

    /**
    * 释放写锁，只做1次
	*/
    public boolean tryUnlockWrite() {
        long s; WNode h;
        if (((s = state) & WBIT) != 0L) {
            state = (s += WBIT) == 0L ? ORIGIN : s;//通过+WBIT让高位溢出
            if ((h = whead) != null && h.status != 0)
                release(h);
            return true;
        }
        return false;
    }

    /**
    * 释放读锁
    */
    public boolean tryUnlockRead() {
        long s, m; WNode h;
        while ((m = (s = state) & ABITS) != 0L && m < WBIT) {//存在读锁并且不存在写锁
            if (m < RFULL) {
                if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {//cas释放
                    if (m == RUNIT && (h = whead) != null && h.status != 0)
                        release(h);
                    return true;
                }
            }
            else if (tryDecReaderOverflow(s) != 0L)
                return true;
        }
        return false;
    }

    /**
     * 读锁数量
     */
    private int getReadLockCount(long s) {
        long readers;
        if ((readers = s & RBITS) >= RFULL)
            readers = RFULL + readerOverflow;
        return (int) readers;
    }

    /**
     * is存在写锁
     */
    public boolean isWriteLocked() {
        return (state & WBIT) != 0L;
    }

    /**
     * is存在读锁
     */
    public boolean isReadLocked() {
        return (state & RBITS) != 0L;
    }

    /**
     * 快照下读锁数量
     */
    public int getReadLockCount() {
        return getReadLockCount(state);
    }

    public String toString() {
        long s = state;
        return super.toString() +
            ((s & ABITS) == 0L ? "[Unlocked]" :
             (s & WBIT) != 0L ? "[Write-locked]" :
             "[Read-locks:" + getReadLockCount(s) + "]");
    }

    //读锁视图
    public Lock asReadLock() {
        ReadLockView v;
        return ((v = readLockView) != null ? v :
                (readLockView = new ReadLockView()));
    }

    /**
     * 写锁视图
     */
    public Lock asWriteLock() {
        WriteLockView v;
        return ((v = writeLockView) != null ? v :
                (writeLockView = new WriteLockView()));
    }

    public ReadWriteLock asReadWriteLock() {
        ReadWriteLockView v;
        return ((v = readWriteLockView) != null ? v :
                (readWriteLockView = new ReadWriteLockView()));
    }

    // 视图类-方便使用
    final class ReadLockView implements Lock {
        public void lock() { readLock(); }
        public void lockInterruptibly() throws InterruptedException {
            readLockInterruptibly();
        }
        public boolean tryLock() { return tryReadLock() != 0L; }
        public boolean tryLock(long time, TimeUnit unit)
            throws InterruptedException {
            return tryReadLock(time, unit) != 0L;
        }
        public void unlock() { unstampedUnlockRead(); }
        public Condition newCondition() {
            throw new UnsupportedOperationException();
        }
    }

    final class WriteLockView implements Lock {
        public void lock() { writeLock(); }
        public void lockInterruptibly() throws InterruptedException {
            writeLockInterruptibly();
        }
        public boolean tryLock() { return tryWriteLock() != 0L; }
        public boolean tryLock(long time, TimeUnit unit)
            throws InterruptedException {
            return tryWriteLock(time, unit) != 0L;
        }
        public void unlock() { unstampedUnlockWrite(); }
        public Condition newCondition() {
            throw new UnsupportedOperationException();
        }
    }

    final class ReadWriteLockView implements ReadWriteLock {
        public Lock readLock() { return asReadLock(); }
        public Lock writeLock() { return asWriteLock(); }
    }


    /**
    * 不检查版本的释放写锁
    */
    final void unstampedUnlockWrite() {
        WNode h; long s;
        if (((s = state) & WBIT) == 0L)
            throw new IllegalMonitorStateException();
        state = (s += WBIT) == 0L ? ORIGIN : s;
        if ((h = whead) != null && h.status != 0)
            release(h);
    }

	/**
    * 不检查版本的释放读锁
    */
    final void unstampedUnlockRead() {
        for (;;) {
            long s, m; WNode h;
            if ((m = (s = state) & ABITS) == 0L || m >= WBIT)
                throw new IllegalMonitorStateException();
            else if (m < RFULL) {
                if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                    if (m == RUNIT && (h = whead) != null && h.status != 0)
                        release(h);
                    break;
                }
            }
            else if (tryDecReaderOverflow(s) != 0L)
                break;
        }
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        state = ORIGIN; // reset to unlocked state
    }

    // internals

    /**
     * readerOverflow 增加
     */
    private long tryIncReaderOverflow(long s) {
        // assert (s & ABITS) >= RFULL;
        if ((s & ABITS) == RFULL) {
            if (U.compareAndSwapLong(this, STATE, s, s | RBITS)) {
                ++readerOverflow;
                state = s;
                return s;
            }
        }
        else if ((LockSupport.nextSecondarySeed() &
                  OVERFLOW_YIELD_RATE) == 0)
            Thread.yield();
        return 0L;
    }

    /**
     * readerOverflow减少
     */
    private long tryDecReaderOverflow(long s) {
        // assert (s & ABITS) >= RFULL;
        if ((s & ABITS) == RFULL) {
            if (U.compareAndSwapLong(this, STATE, s, s | RBITS)) {
                int r; long next;
                if ((r = readerOverflow) > 0) {
                    readerOverflow = r - 1;
                    next = s;
                }
                else
                    next = s - RUNIT;
                 state = next;
                 return next;
            }
        }
        else if ((LockSupport.nextSecondarySeed() &
                  OVERFLOW_YIELD_RATE) == 0)
            Thread.yield();
        return 0L;
    }

    /**
     * 唤醒head节点
     */
    private void release(WNode h) {
        if (h != null) {
            WNode q; Thread w;
            U.compareAndSwapInt(h, WSTATUS, WAITING, 0);//尝试修改状态
            if ((q = h.next) == null || q.status == CANCELLED) {//对h.next的判断，为空或状态为取消
                for (WNode t = wtail; t != null && t != h; t = t.prev)//从尾部遍历，找到状态对的点
                    if (t.status <= 0)
                        q = t;
            }
            if (q != null && (w = q.thread) != null)
                U.unpark(w);//unpark 唤醒操作
        }
    }

    /**
     * 写线程先自旋获取写锁，失败后入队
     */
    private long acquireWrite(boolean interruptible, long deadline) {
        WNode node = null, p;
        for (int spins = -1;;) { // spin while enqueuing
            long m, s, ns;
            if ((m = (s = state) & ABITS) == 0L) { // 无锁状态
                if (U.compareAndSwapLong(this, STATE, s, ns = s + WBIT))//cas来一次获取锁的尝试
                    return ns;
            }
            else if (spins < 0)//如果自旋次数小于0，则计算自旋的次数
                /**
                * m == WBIT 写锁存在，wtail == whead 没有再等待竞争的节点
                * 那就在for循环里等待，很快会轮到我的
                * 否者开始计数
                */
                spins = (m == WBIT && wtail == whead) ? SPINS : 0;
            else if (spins > 0) {
                if (LockSupport.nextSecondarySeed() >= 0)
                    --spins;
            }
            else if ((p = wtail) == null) { // initialize queue
            	//当前队列没有初始化，新建一个空节点，给头尾节点
                WNode hd = new WNode(WMODE, null);
                if (U.compareAndSwapObject(this, WHEAD, null, hd))
                    wtail = hd;
            }
            else if (node == null)//初始化节点
                node = new WNode(WMODE, p);
            else if (node.prev != p)//设置前驱节点为之前的尾节点
                node.prev = p;
            else if (U.compareAndSwapObject(this, WTAIL, p, node)) {//加到尾节点
                p.next = node;
                break;
            }
        }

        //这里又进入第二次自旋了，主要进行阻塞并等待唤醒
        for (int spins = -1;;) {
            WNode h, np, pp; int ps;
            if ((h = whead) == p) {//我的前置节点事头节点，那下个该轮到我了
                if (spins < 0)//自选次数初始化
                    spins = HEAD_SPINS;
                else if (spins < MAX_HEAD_SPINS)//自旋次数小于头结点最大自旋次数，则增加自旋次数
                    spins <<= 1;
                for (int k = spins;;) { // 进入这个循环，是无锁时尝试获取写锁
                    long s, ns;
                    if (((s = state) & ABITS) == 0L) {
                        if (U.compareAndSwapLong(this, STATE, s,
                                                 ns = s + WBIT)) {
                            whead = node;
                            node.prev = null;
                            return ns;
                        }
                    }
                    else if (LockSupport.nextSecondarySeed() >= 0 &&
                             --k <= 0)
                        break;
                }
            }
            else if (h != null) { // help release stale waiters
                WNode c; Thread w;
                while ((c = h.cowait) != null) {//这里是去唤醒所有等待的读线程？
                    if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                        (w = c.thread) != null)
                        U.unpark(w);
                }
            }
            if (whead == h) {//头节点没有改变
                if ((np = node.prev) != p) {
                    if (np != null)
                        (p = np).next = node;   // stale
                }
                else if ((ps = p.status) == 0)
                    U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
                else if (ps == CANCELLED) {
                    if ((pp = p.prev) != null) {
                        node.prev = pp;
                        pp.next = node;
                    }
                }
                else {
                    long time; // 0 argument to park means no timeout
                    if (deadline == 0L)
                        time = 0L;
                    else if ((time = deadline - System.nanoTime()) <= 0L)
                        return cancelWaiter(node, node, false);
                    Thread wt = Thread.currentThread();
                    U.putObject(wt, PARKBLOCKER, this);
                    node.thread = wt;
                    /**
                    * 1、当前节点前驱节点状态为WAITING；
                	* 2、当前节点的前驱节点不是头节点或有读写锁已经被获取；
                	* 3、头结点没改变；
                	* 4、当前节点的前驱节点没改变；
                	* 当以上4个条件都满足时将当前节点进行阻塞
                    */
                    if (p.status < 0 && (p != h || (state & ABITS) != 0L) &&
                        whead == h && node.prev == p)
                        U.park(false, time);  // emulate LockSupport.park
                    node.thread = null;
                    U.putObject(wt, PARKBLOCKER, null);
                    if (interruptible && Thread.interrupted())
                        return cancelWaiter(node, node, true);
                }
            }
        }
    }

    /**
     * 尝试自旋的获取读锁, 获取不到则加入等待队列, 并阻塞线程
     */
    private long acquireRead(boolean interruptible, long deadline) {
        WNode node = null, p;
        for (int spins = -1;;) {
            WNode h;
            if ((h = whead) == (p = wtail)) {//写锁等待队列头尾相同，没有等待中的线程
                for (long m, s, ns;;) {
                	/**
                	* 当读标志位没满，写锁也没有，就cas更新+1
                	* 当满了，就增加ReaderOverflow
                	*/
                    if ((m = (s = state) & ABITS) < RFULL ?
                        U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                        (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L))
                        return ns;
                    else if (m >= WBIT) {//存在写锁的情况
                        if (spins > 0) {
                            if (LockSupport.nextSecondarySeed() >= 0)
                                --spins;
                        }
                        else {
                            if (spins == 0) {//自选次数到了
                                WNode nh = whead, np = wtail;
                                //头尾节点没有改变，并且存在竞争，就打断当前自旋
                                if ((nh == h && np == p) || (h = nh) != (p = np))
                                    break;
                            }
                            spins = SPINS;
                        }
                    }
                }
            }
            if (p == null) { // 初始化队列，头尾节点赋值空node
                WNode hd = new WNode(WMODE, null);
                if (U.compareAndSwapObject(this, WHEAD, null, hd))
                    wtail = hd;
            }
            else if (node == null)//初始化当前
                node = new WNode(RMODE, p);
            else if (h == p || p.mode != RMODE) {//当前mode不是读，添加到队列尾部
                if (node.prev != p)
                    node.prev = p;
                else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
                    p.next = node;
                    break;
                }
            }
            //如果head!= tail说明队列中已经有线程在等待或者tail.mode是读状态RMODE，
        	//那么CAS方式将当前线程的节点node加入到tail节点的cowait链中
            else if (!U.compareAndSwapObject(p, WCOWAIT,
                                             node.cowait = p.cowait, node))//更新cowait
                node.cowait = null;
            else {
                for (;;) {
                    WNode pp, c; Thread w;
                    /**
                    * 尝试唤醒头结点的cowait中的第一个元素, 循环释放cowait链
                    */
                    if ((h = whead) != null && (c = h.cowait) != null &&
                        U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                        (w = c.thread) != null) // help release
                        U.unpark(w);
                    if (h == (pp = p.prev) || h == p || pp == null) {//快到我了
                        long m, s, ns;
                        /**
                        * 就尝试自旋获取锁
                        */
                        do {
                            if ((m = (s = state) & ABITS) < RFULL ?
                                U.compareAndSwapLong(this, STATE, s,
                                                     ns = s + RUNIT) :
                                (m < WBIT &&
                                 (ns = tryIncReaderOverflow(s)) != 0L))
                                return ns;
                        } while (m < WBIT);
                    }
                    if (whead == h && p.prev == pp) {//存在写锁，结构稳定的
                        long time;
                        if (pp == null || h == p || p.status > 0) {
                            node = null; // throw away
                            break;
                        }
                        if (deadline == 0L)
                            time = 0L;
                        else if ((time = deadline - System.nanoTime()) <= 0L)
                            return cancelWaiter(node, p, false);
                        Thread wt = Thread.currentThread();
                        U.putObject(wt, PARKBLOCKER, this);
                        node.thread = wt;
                        if ((h != pp || (state & ABITS) == WBIT) &&
                            whead == h && p.prev == pp)
                            U.park(false, time);
                        node.thread = null;
                        U.putObject(wt, PARKBLOCKER, null);
                        if (interruptible && Thread.interrupted())
                            return cancelWaiter(node, p, true);
                    }
                }
            }
        }

        for (int spins = -1;;) {
            WNode h, np, pp; int ps;
            if ((h = whead) == p) {
                if (spins < 0)
                    spins = HEAD_SPINS;
                else if (spins < MAX_HEAD_SPINS)
                    spins <<= 1;
                for (int k = spins;;) { // spin at head
                    long m, s, ns;
                    if ((m = (s = state) & ABITS) < RFULL ?
                        U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                        (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                        WNode c; Thread w;
                        whead = node;
                        node.prev = null;
                        while ((c = node.cowait) != null) {
                            if (U.compareAndSwapObject(node, WCOWAIT,
                                                       c, c.cowait) &&
                                (w = c.thread) != null)
                                U.unpark(w);
                        }
                        return ns;
                    }
                    else if (m >= WBIT &&
                             LockSupport.nextSecondarySeed() >= 0 && --k <= 0)
                        break;
                }
            }
            else if (h != null) {
                WNode c; Thread w;
                while ((c = h.cowait) != null) {
                    if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                        (w = c.thread) != null)
                        U.unpark(w);
                }
            }
            if (whead == h) {
                if ((np = node.prev) != p) {
                    if (np != null)
                        (p = np).next = node;   // stale
                }
                else if ((ps = p.status) == 0)
                    U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
                else if (ps == CANCELLED) {
                    if ((pp = p.prev) != null) {
                        node.prev = pp;
                        pp.next = node;
                    }
                }
                else {
                    long time;
                    if (deadline == 0L)
                        time = 0L;
                    else if ((time = deadline - System.nanoTime()) <= 0L)
                        return cancelWaiter(node, node, false);
                    Thread wt = Thread.currentThread();
                    U.putObject(wt, PARKBLOCKER, this);
                    node.thread = wt;
                    /**
                    * 1、当前节点前驱节点状态为WAITING；
                	* 2、当前节点的前驱节点不是头节点或有读写锁已经被获取；
                	* 3、头结点没改变；
                	* 4、当前节点的前驱节点没改变；
                	* 当以上4个条件都满足时将当前节点进行阻塞
                    */
                    if (p.status < 0 &&
                        (p != h || (state & ABITS) == WBIT) &&
                        whead == h && node.prev == p)
                        U.park(false, time);
                    node.thread = null;
                    U.putObject(wt, PARKBLOCKER, null);
                    if (interruptible && Thread.interrupted())
                        return cancelWaiter(node, node, true);
                }
            }
        }
    }

    /**
     * 取消操作
     */
    private long cancelWaiter(WNode node, WNode group, boolean interrupted) {
        if (node != null && group != null) {
            Thread w;
            node.status = CANCELLED;//设置成取消状态
            // 如果当前取消节点cowait 不为空，将当前节点从中拿掉
            for (WNode p = group, q; (q = p.cowait) != null;) {
                if (q.status == CANCELLED) {
                    U.compareAndSwapObject(p, WCOWAIT, q, q.cowait);
                    p = group; // restart
                }
                else
                    p = q;
            }
            if (group == node) {
                for (WNode r = group.cowait; r != null; r = r.cowait) {//唤醒状态没有取消的cowait队列中的节点
                    if ((w = r.thread) != null)
                        U.unpark(w);       // wake up uncancelled co-waiters
                }
                for (WNode pred = node.prev; pred != null; ) { // unsplice
                    WNode succ, pp;        // find valid successor
                    while ((succ = node.next) == null ||
                           succ.status == CANCELLED) {//while循环中不断的通过next找到CANCELLED节点
                        WNode q = null;    // find successor the slow way
                        for (WNode t = wtail; t != null && t != node; t = t.prev)
                            if (t.status != CANCELLED)//由tail节点开始往前找，不是取消的节点
                                q = t;     // don't link if succ cancelled
                        if (succ == q ||   // 当前取消节点的next节点 与找到的非取消状态节点相同
                            U.compareAndSwapObject(node, WNEXT,
                                                   succ, succ = q)) {//或者把找到的非取消状态节点 设置成当前取消节点的next 成功
                            if (succ == null && node == wtail)//判断当前节点为尾巴节点
                                U.compareAndSwapObject(this, WTAIL, node, pred);//把前驱pred当作尾节点
                            break;
                        }
                    }
                    if (pred.next == node) //当前取消节点的 pred节点的next节点为当前取消节点
                        U.compareAndSwapObject(pred, WNEXT, node, succ);//把pred的next设置为当前取消节点的next
                    if (succ != null && (w = succ.thread) != null) {
                        succ.thread = null;
                        U.unpark(w);       // 唤起succ节点，让其寻找新的前驱
                    }
                    //前驱节点不是取消或者前驱的前驱为空，中止循环
                    if (pred.status != CANCELLED || (pp = pred.prev) == null)
                        break;
                    node.prev = pp;        // 重新设置 当前取消节点的前置节点
                    U.compareAndSwapObject(pp, WNEXT, pred, succ);//将前驱节点的前驱节点的next设置为当前取消节点的next
                    pred = pp;//重设pred
                }
            }
        }
        WNode h; // Possibly release first waiter
        while ((h = whead) != null) {
            long s; WNode q; // similar to release() but check eligibility
            if ((q = h.next) == null || q.status == CANCELLED) {//判断状态是否正确，不正确从tail节点往前找
                for (WNode t = wtail; t != null && t != h; t = t.prev)
                    if (t.status <= 0)
                        q = t;
            }
            if (h == whead) {
            	/**
            	* 1.头节点的下一个有效节点不为空
            	* 2.头节点状态为0
            	* 3.不存在写锁
            	* 4.并且为读模式
            	* 然后release 唤醒下一个节点
            	*/
                if (q != null && h.status == 0 &&
                    ((s = state) & ABITS) != WBIT && // waiter is eligible
                    (s == 0L || q.mode == RMODE))
                    release(h);
                break;
            }
        }
        return (interrupted || Thread.interrupted()) ? INTERRUPTED : 0L;
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe U;
    private static final long STATE;
    private static final long WHEAD;
    private static final long WTAIL;
    private static final long WNEXT;
    private static final long WSTATUS;
    private static final long WCOWAIT;
    private static final long PARKBLOCKER;

    static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = StampedLock.class;
            Class<?> wk = WNode.class;
            STATE = U.objectFieldOffset
                (k.getDeclaredField("state"));
            WHEAD = U.objectFieldOffset
                (k.getDeclaredField("whead"));
            WTAIL = U.objectFieldOffset
                (k.getDeclaredField("wtail"));
            WSTATUS = U.objectFieldOffset
                (wk.getDeclaredField("status"));
            WNEXT = U.objectFieldOffset
                (wk.getDeclaredField("next"));
            WCOWAIT = U.objectFieldOffset
                (wk.getDeclaredField("cowait"));
            Class<?> tk = Thread.class;
            PARKBLOCKER = U.objectFieldOffset
                (tk.getDeclaredField("parkBlocker"));

        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
{% endhighlight %}

