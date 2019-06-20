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
2. ReentrantReadWriteLock类有 WriteLock 、 ReadLock 、FairSync 、 NonfairSync 和 Sync 这5个内部类；
3. Sync 是 FairSync 和 NonfairSync 的父类；Sync 抽象类内部有 ThreadLocalHoldCounter 和 HoldCounter 这两个内部类。


接下来我就按照这3个信息，一步一步的剖析ReentrantReadWriteLock类源码吧！


#### ReentrantReadWriteLock实现ReadWriteLock接口和Serializable接口

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
ps.我在看源码的过程中，看到writeLock()和readLock()两个实现方法没有@Override注解，不禁提出了疑问，@Override这个注解可以省略的吗？后来经过实践和看大佬们的博客得知，@Override注解在程序中是可以省略的，@Override注解的作用可以理解为两个作用，第一是提高可读性，因为加上这个注解之后就可以知道是实现父类或者接口的抽象方法；第二就是可以在编译期间，ide可以检验加上@Override的方法





总结