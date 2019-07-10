```java
    private final ReentrantReadWriteLock.ReadLock readerLock;
    private final ReentrantReadWriteLock.WriteLock writerLock;

    public ReentrantReadWriteLock() {
        this(false);
    }

    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    public static class ReadLock implements Lock, java.io.Serializable {
        private final Sync sync;

        protected ReadLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
    }
    public static class WriteLock implements Lock, java.io.Serializable {
        private final Sync sync;

        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }
    } 
```
首先初始化 ReentrantReadWriteLock 对象，同时根据 ReentrantReadWriteLock 构造器传入的参数判断使用公平锁或者非公平锁创建读锁和写锁。

接下来获取读锁，并执行lock()方法获取读锁。
```java
    // 获取读锁
    public void lock() {
        sync.acquireShared(1);
    }

    // 执行AQS类的acquireShared方法，如果获取读锁失败后返回的值会小于0，往下执行doAcquireShared方法加入等待队列
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    // 获取读锁
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        // 如果exclusiveCount(c)不等于0代表写线程未释放锁，而且当前线程不是写锁的占有线程，则返回-1，获取锁失败
        if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
            return -1;
        // 读锁重入次数
        int r = sharedCount(c);
        // a.读锁阻塞
        // b.重入次数大于共享锁线程最大个数（65535）
        // c.并发情景下CAS失败
        // 符合这三个任意条件之一都会执行fullTryAcquireShared()自旋获取读锁
        if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {// 如果读锁重入次数为0代表当前没有线程占有读锁，设置当前线程为第一个获取读锁的线程
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {// 如果当前线程为第一个获取读锁的线程（重复获取读锁）则第一个firstReaderHoldCount加1
                firstReaderHoldCount++;
            } else {
                // 将缓存计数器赋值给HoldCounter对象
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))// 缓存计数器是否null或者缓存的线程ID是否为当前线程的ID
                    cachedHoldCounter = rh = readHolds.get();// 将readHolds计数器赋值给cachedHoldCounter计数器
                else if (rh.count == 0)// 如果缓存的计数器为0代表上次读锁unlock()操作的时候，执行readHolds.remove()把readHolds计数器count值设置为0
                    readHolds.set(rh);
                rh.count++;// HoldCounter对象的count加1，相当于读锁重入次数加1
            }
            return 1;
        }
        // 自旋获取读锁
        return fullTryAcquireShared(current);
    }

    final int fullTryAcquireShared(Thread current) {
        HoldCounter rh = null;
        for (;;) {
            int c = getState();// 获取锁个数
            if (exclusiveCount(c) != 0) {// 判断写锁的重入次数是否为0，如果不是0则代表写锁被线程占有
                if (getExclusiveOwnerThread() != current)// 判断当前线程是否为占有锁的写线程，返回-1代表获取锁失败，将线程放进队列
                    return -1;
            } else if (readerShouldBlock()) {// 读锁是否需要阻塞
                if (firstReader == current) {// 如果当前线程为第一个占有读锁的线程，继续往下执行
                    // assert firstReaderHoldCount > 0;
                } else {
                    if (rh == null) { // 如果HoldCounter对象为null，将缓存计算器赋值给HoldCounter对象
                        rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current)) {// 缓存计数器是否null或者缓存的线程ID是否为当前线程的ID
                            rh = readHolds.get();// 将本地readHolds计数器赋值给HoldCounter对象
                            if (rh.count == 0)// 如果HoldCounter对象为0，代表上次读锁unlock()操作的时候，执行readHolds.remove()把readHolds计数器count值设置为0
                                readHolds.remove();// 将readHolds的count值归0，所以接下来的读锁CAS操作是一定成功的，因为读锁没有被占有
                        }
                    }
                    if (rh.count == 0) // 如果readHolds计数器为0，返回-1，将线程放进队列
                        return -1;
                }
            }
            if (sharedCount(c) == MAX_COUNT) // 判断共享锁的重入次数是否为共享锁线程最大个数
                throw new Error("Maximum lock count exceeded");
            if (compareAndSetState(c, c + SHARED_UNIT)) {// CAS操作，如果失败继续自旋
                if (sharedCount(c) == 0) {// 如果读锁的重入值为0，将当前线程作为第一个占有读锁的线程
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {// 如果当前线程为第一个占有读锁的线程，则firstReaderHoldCount加1
                    firstReaderHoldCount++;
                } else {// 不是第一个占有读锁的线程
                    if (rh == null) // 执行readHolds.remove()后再次执行rh = readHolds.get()，所以rh有可能为null
                        rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) // 如果当前线程是附一个
                        rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                    cachedHoldCounter = rh;
                }
                return 1; // 代表获取锁成功
            }
        }
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
