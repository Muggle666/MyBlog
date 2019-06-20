在并发场景中，程序读多写少是非常普遍的。针对这种情况，JDK1.5提高了并发类**ReentrantReadWriteLock**。

## 简介：

ReentrantReadWriteLock类内部有**共享锁**和**排他锁**，在并发情况下，多个线程读操作使用共享锁，可以有多个线程同时执行读操作；而多个线程写操作是使用到排他锁，只允许一个线程执行写操作。所以在读多写少的场景下，使用ReentrantReadWriteLock类可以提高性能。

先写个示例使用ReentrantReadWriteLock类先吧！

```java
public class ReentrantReadWriteLockDemo {

    private static int threadNum = 100000;

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
    // 累加的分数
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



先看下ReentrantReadWriteLock的类图：

![ReentrantReadWriteLock类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ReentrantReadWriteLock%E7%B1%BB%E5%9B%BE.jpg)

emmm...好像好复杂，ReentrantReadWriteLock类有那么多内部类。