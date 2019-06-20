在并发场景中，程序读多写少是非常普遍的。针对这种情况，Doug Lea 大佬写出并发类ReentrantReadWriteLock。

## 简介：

ReentrantReadWriteLock类内部有共享锁和排他锁，在并发情况下，多个线程读操作使用共享锁，可以有多个线程同时执行读操作；而多个线程写操作是使用到排他锁，只允许一个线程执行写操作。所以在读多写少的场景下，使用ReentrantReadWriteLock类可以提高性能。

先看下ReentrantReadWriteLock的类图：

![https://raw.githubusercontent.com/MuggleLee/PicGo/master/ReentrantReadWriteLock%E7%B1%BB%E5%9B%BE.jpg](https://raw.githubusercontent.com/MuggleLee/PicGo/master/ReentrantReadWriteLock%E7%B1%BB%E5%9B%BE.jpg)