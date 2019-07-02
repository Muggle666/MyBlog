前提：看ReentrantReadWriteLock源码的时候，发现其内部声明了一个内部类ThreadLocalHoldCounter，而这个内部类继承ThreadLocal类，后来粗读了ReentrantReadWriteLock的源码，发现ThreadLocalHoldCounter这个类发挥及其重要的作用，因此我决定将ThreadLocal类好好研究一番~


### 什么是ThreadLocal


用ThreadLocal声明的变量可以在线程内部提供变量副本，线程彼此之间修改ThreadLocal声明的变量互不影响，这就不存在并发的情况了。


先看下ThreadLocal类的





Why：为什么使用ThreadLocal


Where：在哪里使用ThreadLocal


How：怎么使用ThreadLocal
