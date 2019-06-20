在并发场景中，程序读多写少是非常普遍的。针对这种情况，Doug Lea 大佬写出并发类ReentrantReadWriteLock。

## 简介：

ReentrantReadWriteLock类内部有共享锁和排它锁，在并发情况下，多个线程读操作使用共享锁，可以有多个线程同时执行都；而多个线程写操作是使用到排它锁，只允许一个线程执行写操作