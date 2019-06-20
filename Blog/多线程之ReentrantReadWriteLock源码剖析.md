在并发场景中，程序读多写少是非常普遍的。针对这种情况，JDK1.5提高了并发类**ReentrantReadWriteLock**。

## 简介：

ReentrantReadWriteLock类内部有**共享锁**和**排他锁**，在并发情况下，多个线程读操作使用共享锁，可以有多个线程同时执行读操作；而多个线程写操作是使用到排他锁，只允许一个线程执行写操作。所以在读多写少的场景下，使用ReentrantReadWriteLock类可以提高性能。

先写个示例使用ReentrantReadWriteLock类先吧！

```java
public class ReentrantReadWriteLockDemo {

    private static int threadNum = 100000;

    // 设置栅栏是为了防止循环还没结束就执行main线程输出自增的变量，导致误以为线程不安全
    private static CountDownLatch countDownLatch = new CountDownLatch(threadNum);

    public static void main(String[] args) {
        Score score = new Score();
        score.setScore(0);

        for (int i = 0; i < threadNum; i++) {
            new Thread(() -> {
                score.add(1);
                countDownLatch.countDown();
            }).start();
        }
        try {
            countDownLatch.await();
            System.out.println(score.getScore());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}

class Score {
    // 共享变量：累加的分数
    private int score;

    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    // 读锁
    private ReentrantReadWriteLock.ReadLock readLock = lock.readLock();

    // 写锁
    private ReentrantReadWriteLock.WriteLock writeLock = lock.writeLock();

    public void add(int score) {
        try {
            writeLock.lock();
            this.score = this.score + score;
        } finally {
            writeLock.unlock();
        }
    }

    public int getScore() {
        try {
            readLock.lock();
            return score;
        } finally {
            readLock.unlock();
        }
    }

    public void setScore(int score) {
        try {
            writeLock.lock();
            this.score = score;
        } finally {
            writeLock.unlock();
        }
    }
}
```
输出结果：
```java
100000
```
使用ReentrantReadWriteLock类可以保证线程安全，但为什么上面说在读多写少的场景下可以优化性能呢？接下来，从源码的角度进行剖析！

### ReentrantReadWriteLock源码剖析

先看下ReentrantReadWriteLock的类图：

![ReentrantReadWriteLock类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ReentrantReadWriteLock%E7%B1%BB%E5%9B%BE.jpg)

emmm...好像好复杂，ReentrantReadWriteLock类有那么多内部类。


从ReentrantReadWriteLock类图中，我可以知道以下几个信息：
1. ReentrantReadWriteLock实现ReadWriteLock接口和Serializable接口；
2. Sync 是 FairSync 和 NonfairSync 的父类；Sync 抽象类内部有 ThreadLocalHoldCounter 和 HoldCounter 这两个内部类；
3.ReentrantReadWriteLock类有 WriteLock 、 ReadLock 、FairSync 、 NonfairSync 和 Sync 这5个内部类。

接下来我就按照这3个信息，一步一步的剖析ReentrantReadWriteLock类源码吧！


#### 1.ReentrantReadWriteLock实现ReadWriteLock接口和Serializable接口

```java
public class ReentrantReadWriteLock implements ReadWriteLock, java.io.Serializable {
    private static final long serialVersionUID = -6992448646407690164L;

    // 内部类提供的读锁
    private final ReentrantReadWriteLock.ReadLock readerLock;

    // 内部类提供的写锁
    private final ReentrantReadWriteLock.WriteLock writerLock;

    // 执行同步的内部类
    final Sync sync;

    // 默认执行非公平锁
    public ReentrantReadWriteLock() {
        this(false);
    }

    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    // 实现ReadWriteLock接口的writeLock()
    public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
    // 实现ReadWriteLock接口的readLock()
    public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
}
```
###### ps.我在看源码的过程中，看到writeLock()和readLock()两个实现方法没有@Override注解，不禁提出了疑问，@Override这个注解可以省略的吗？后来经过实践和看大佬们的博客得知，@Override注解在程序中是可以省略的，@Override注解的作用可以理解为两个作用，第一是提高可读性，因为加上这个注解之后就可以知道是实现父类或者接口的抽象方法；第二就是可以在编译期间，IDE可以检验加上@Override的方法是否与父类或者接口方法一样。对Java注解不熟呀，抽时间出来系统学一下Java注解才行...


上面的代码只保留ReentrantReadWriteLock类的实现方法和构造器，方便我们了解创建ReentrantReadWriteLock对象时，在底层是怎样运作的。

![创建ReentrantReadWriteLock对象的流程图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ReentrantReadWriteLock/ReentrantReadWriteLock-InitUML.png)

#### 2. Sync 是 FairSync 和 NonfairSync 的父类；Sync 抽象类内部有 ThreadLocalHoldCounter 和 HoldCounter 这两个内部类。

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 6317671515068378041L;

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

        static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }

        static final class ThreadLocalHoldCounter
                extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }

        private transient ThreadLocalHoldCounter readHolds;

        private transient HoldCounter cachedHoldCounter;

        private transient Thread firstReader = null;
        private transient int firstReaderHoldCount;

        Sync() {
            readHolds = new ThreadLocalHoldCounter();
            setState(getState()); 
        }

        abstract boolean readerShouldBlock();

        abstract boolean writerShouldBlock();

        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }

        protected final boolean tryAcquire(int acquires) {
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                    !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }

        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }

        private IllegalMonitorStateException unmatchedUnlockException() {
            return new IllegalMonitorStateException(
                    "attempt to unlock read lock, not locked by current thread");
        }

        protected final int tryAcquireShared(int unused) {
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                    r < MAX_COUNT &&
                    compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }

        final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
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

        final boolean tryReadLock() {
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0 &&
                        getExclusiveOwnerThread() != current)
                    return false;
                int r = sharedCount(c);
                if (r == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
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
            setState(0);
        }

        final int getCount() { return getState(); }
    }
```



总结