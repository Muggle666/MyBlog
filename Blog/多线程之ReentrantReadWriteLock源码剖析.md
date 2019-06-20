在并发场景中，程序读多写少是非常普遍的。针对这种情况，JDK1.5提高了并发类**ReentrantReadWriteLock**。

## 简介：

ReentrantReadWriteLock类内部有**共享锁**和**排他锁**，在并发情况下，多个线程读操作使用共享锁，可以有多个线程同时执行读操作；而多个线程写操作是使用到排他锁，只允许一个线程执行写操作。所以在读多写少的场景下，使用ReentrantReadWriteLock类可以提高性能。

先看下ReentrantReadWriteLock的类图：

![ReentrantReadWriteLock类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ReentrantReadWriteLock%E7%B1%BB%E5%9B%BE.jpg)

emmm...好像好复杂，ReentrantReadWriteLock类有那么多内部类