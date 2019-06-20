在并发场景中，程序读多写少是非常普遍的。针对这种情况，Doug Lea 大佬写出并发类ReentrantReadWriteLock。

## 简介：

ReentrantReadWriteLock类内部有共享锁和排它锁