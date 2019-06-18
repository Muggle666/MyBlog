## 简介
在日常开发中，基本数据类型的原子类最常用的可能就是AtomicInteger类了。话说回来，为什么有Integer类还需要有AtomicInteger类呢？先来看看AtomicInteger类的包是啥：**java.util.concurrent.atomic** 。看到没有，这是并发包下的类，所以AtomicInteger类肯定是在并发环境下使用的。

### AtomicInteger原子类的使用

先看一条常见的面试题：x 作为全局变量，使用 x++ 线程安全吗？

```java
public class Test {

    private static int x = 0;

    private static final Integer THREAD_NUM = 100000;

    //设置栅栏是为了防止循环还没结束就执行main线程输出自增的变量，导致误以为线程不安全
    private static CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUM);

    public static void main(String[] args) throws InterruptedException {
        for (int j = 0; j < THREAD_NUM; j++) {
            new Thread(() -> {
                add();
            }).start();
        }
        countDownLatch.await();
        System.out.println("X的值：" + x);
    }

    public static void add(){
        x++;
        countDownLatch.countDown();
    }
}
```
输出结果：
```java
X的值：99993
```

输出的结果具有随机性，有可能是100000也有可能小于100000。由此可以看出，使用 x++ 操作在多线程环境下导致数据错误，也就是说使用 x++ 线程不安全！

那如果使用AtomicInteger原子类呢？
```java
public class Test {

    static AtomicInteger x = new AtomicInteger(0);

    private static final Integer THREAD_NUM = 100000;

    //设置栅栏是为了防止循环还没结束就执行main线程输出自增的变量，导致误以为线程不安全
    private static CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUM);

    public static void main(String[] args) throws InterruptedException {
        for (int j = 0; j < THREAD_NUM; j++) {
            new Thread(() -> {
                add();
            }).start();
        }
        countDownLatch.await();
        System.out.println("X的值：" + x);
    }

    public static void add(){
        x.getAndIncrement();
        countDownLatch.countDown();
    }
}
```
输出结果：
```java
X的值：100000
```

为什么使用 x 作为全局变量，使用 x++ 线程不安全，而使用AtomicInteger会线程安全呢？

以上一直强调 x 是作为全局变量，这是因为全局变量相当于共享变量，在多线程环境下多个线程访问同一个变量会出现线程不安全。x++ 相当于 x = x + 1 ，虽然在高级语言中是一行代码，但对于计算机而言，所有的操作都是指令，而 x++ 这一操作在计算机转化为指令可以理解为有3个步骤：
>1.获取变量的当前值
2.变量当前值加1
3.将新的值赋值给变量

假设现在只有A、B两个线程同时访问共享变量 x ，并发执行 x++ 操作。当线程A访问变量 x 的值为0，并加1，但没来得及将新的值赋值给共享变量 x ，线程B就访问共享变量 x ，此时 x 的值依然为0，然后线程A、线程B分别执行剩下的指令后发现，线程A和线程B的两次 x++ 操作的结果为共享变量x 的值为1。解决这个线程安全的问题，可以使用sychronized或者使用Lock显式加锁解决，也可以使用AtomicInteger类更加优雅的解决，因为java.util.concurrent.atomic包下的类都能够保证原子性,而且最大的好处就是性能，因为使用AtomicInteger是无锁的，对计算机而言，避免了加锁、解锁和线程上下文切换的操作。

好了，经过前面的铺垫，接下来开始对AtomicInteger类进一步的探索！

先了解AtomicInteger原子类有哪些方法和变量吧！
![AtomicInteger类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Atomic/AtomicInteger/AtomicInteger-UML.jpg)

乍一看，好多方法呢！其实大多数的方法都是类似的，难度都不大。
```java
1. public class AtomicIntegerDemo {
2.     static AtomicInteger x = new AtomicInteger(0);
3. 
4.     public static void main(String[] args) {
5. 
6.         System.out.println("get()返回当前AtomicInteger变量的值：" + x.get());
7. 
8.         System.out.println("getAndIncrement()返回x++的值：" + x.getAndIncrement() + " ，X的当前值为" + x.get());
9.         System.out.println("incrementAndGet()返回++x的值：" + x.incrementAndGet() + " ，X的当前值为" + x.get());
10. 
11.         System.out.println("getAndDecrement()返回x--的值：" + x.getAndDecrement() + " ，X的当前值为" + x.get());
12.         System.out.println("decrementAndGet()返回--x的值：" + x.decrementAndGet() + " ，X的当前值为" + x.get());
13. 
14.         System.out.println("getAndAdd()返回x+=10前的值：" + x.getAndAdd(10) + " ，X的当前值为" + x.get());
15.         System.out.println("addAndGet()返回x+=10后的值：" + x.addAndGet(10) + " ，X的当前值为" + x.get());
16. 
17.         System.out.println("getAndUpdate()函数的结果更新当前值，返回更新前的值：" + x.getAndUpdate(t -> 100) + " ，X的当前值为" + x.get());
18.         System.out.println("updateAndGet()函数的结果更新当前值，返回更新后的值：" + x.updateAndGet(t -> 666) + " ，X的当前值为" + x.get());
19. 
20.         System.out.println("getAndAccumulate()使用IntBinaryOperator对当前值和第一个参数进行计算，并更新当前值，返回计算前的值：" + x.getAndAccumulate(100, (s1, s2) -> s1 + s2) + " ，X的当前值为" + x.get());
21.         System.out.println("accumulateAndGet()使用IntBinaryOperator对当前值和第一个参数进行计算，并更新当前值，返回计算后的值：" + x.accumulateAndGet(100, (s1, s2) -> s1 + s2) + " ，X的当前值为" + x.get());
22. 
23.         System.out.println("getAndSet()设定当前的值，返回旧值：" + x.getAndSet(10) + " ，X的当前值为" + x.get());
24.         x.set(30);
25.         System.out.println("set()设定X的当前值为" + x.get());
26.         x.lazySet(20);
27.         System.out.println("lazySet()设定X的当前值为" + x.get());
28. 
29.         //CAS操作，如果x当前值为20则设置x值为10并返回true，否则返回false
30.         System.out.println("compareAndSet()判断x当前值是否为20：" + x.compareAndSet(20, 10) + " ，X的当前值为" + x.get());
31.         //CAS操作，如果x当前值为10则设置x值为20并返回true，否则返回false
32.         System.out.println("weakCompareAndSet()判断x当前值是否为10：" + x.weakCompareAndSet(10, 20) + " ，X的当前值为" + x.get());
33. 
34.         Object shortObj = x.shortValue();;
35.         System.out.println("shortValue()将当前值转为short类型：" + (shortObj instanceof Short));
36.         Object intObj = x.intValue();
37.         System.out.println("intValue()将当前值转为int类型：" + (intObj instanceof Integer));
38.         Object longObj = x.longValue();
39.         System.out.println("longValue()将当前值转为long类型：" + (longObj instanceof Long));
40.         Object doubleObj = x.doubleValue();
41.         System.out.println("doubleValue()将当前值转为double类型：" + (doubleObj instanceof Double));
42.         Object floatObj = x.floatValue();
43.         System.out.println("floatValue()将当前值转为float类型：" + (floatObj instanceof Float));
44.     }
45. }
```
输出结果：
```
get()返回当前AtomicInteger变量的值：0
getAndIncrement()返回x++的值：0 ，X的当前值为1
incrementAndGet()返回++x的值：2 ，X的当前值为2
getAndDecrement()返回x--的值：2 ，X的当前值为1
decrementAndGet()返回--x的值：0 ，X的当前值为0
getAndAdd()返回x+=10前的值：0 ，X的当前值为10
addAndGet()返回x+=10后的值：20 ，X的当前值为20
getAndUpdate()函数的结果更新当前值，返回更新前的值：20 ，X的当前值为100
updateAndGet()函数的结果更新当前值，返回更新后的值：666 ，X的当前值为666
getAndAccumulate()使用IntBinaryOperator对当前值和第一个参数进行计算，并更新当前值，返回计算前的值：666 ，X的当前值为766
accumulateAndGet()使用IntBinaryOperator对当前值和第一个参数进行计算，并更新当前值，返回计算后的值：866 ，X的当前值为866
getAndSet()设定当前的值，返回旧值：866 ，X的当前值为10
set()设定X的当前值为30
lazySet()设定X的当前值为20
compareAndSet()判断x当前值是否为20：true ，X的当前值为10
weakCompareAndSet()判断x当前值是否为10：true ，X的当前值为20
shortValue()将当前值转为short类型：true
intValue()将当前值转为int类型：true
longValue()将当前值转为long类型：true
doubleValue()将当前值转为double类型：true
floatValue()将当前值转为float类型：true
```


值得注意的是，示例中20~32行的getAndAccumulate、accumulateAndGet、lazySet、compareAndSet和weakCompareAndSet方法。
```java
    public final int getAndAccumulate(int x, IntBinaryOperator accumulatorFunction) {
        int prev, next;
        do {
            prev = get();
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSet(prev, next));
        return prev;
    }

    public final int accumulateAndGet(int x,IntBinaryOperator accumulatorFunction) {
        int prev, next;
        do {
            prev = get();
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSet(prev, next));
        return next;
    }

    public final void lazySet(int newValue) {
        unsafe.putOrderedInt(this, valueOffset, newValue);
    }

    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    public final boolean weakCompareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```
**getAndAccumulate()** 和**accumulateAndGet()** 第二个参数是通过传入function函数计算，我们可以自定义计算的过程。

譬如：
```java
public class MethodDemo {
    private static AtomicInteger atomic = new AtomicInteger(2);

    public static void main(String[] args) {
        atomic.getAndAccumulate(10, (s1, s2) -> (s1 + s2) * s2);
        System.out.println(atomic.get());//计算结果：120
    }
}
```


**lazySet()、compareAndSet()、weakCompareAndSet()** 方法都是调用Unsafe对象的方法。

>查看Atomic包下所有类的源码，很多的方法都有用到Unsafe对象，而这个对象内部大部分方法都是使用native关键字。Unsafe这个类可以简单的理解为是与操作系统交互的对象。原子类之所以能够保证原子性，无锁情况下保证线程安全就是得益于使用Unsafe对象与操作系统硬件交互，通过计算机硬件保证系统安全。

观察compareAndSet()、weakCompareAndSet()两个方法的源码，惊奇的发现，两个方法的实现都一样的！查看官方文档，各种的博客都找不到准确的说法，不过大多数的说法都是说使用weakCompareAndSet()方法可能存在happen-before的情况，也就是在执行weakCompareAndSet()方法的时候发生了指令重排序导致失败，返回false。【[歪果仁的答案](https://stackoverflow.com/questions/2443239/how-can-weakcompareandset-fail-spuriously-if-it-is-implemented-exactly-like-comp)】

#### lazySet()和set()有什么区别呢？

他们都是改变引用类型为AtomicInteger变量的值，但合理使用lazySet()方法可以性能优化！

```java
private volatile int value;
```

由源码可以知道，引用类型为AtomicInteger变量的值是用volatile关键字修饰，volatile的作用大致可以总结为：
>1.提供内存屏障。
2.防止重排序。
3.volatile关键字修饰的变量在写入的时候会强制将cpu写缓冲区刷新到内存；在读取的时候会强制从内存中读取最新的值。

所以使用set()方法一定会将最新的值设置到value，但使用lazySet()方法改变共享变量的值不一定会被其它线程“看到”。所以使用lazySet()方法是线程不安全的，但为什么上面说合理使用lazySet()可以优化程序呢？

举个栗子：
```java
public void Demo() {
        AtomicInteger atomicInteger = new AtomicInteger(10);

        Lock lock = new ReentrantLock();

        lock.lock();
        try {
//            atomicInteger.set(2333);//变量由volatile关键字修饰，会使用到内存屏障，在锁内其实是不必要的
            atomicInteger.lazySet(2333);
        } finally {
            lock.unlock();
        }
    }
```

在使用显式锁的情景下，合理使用lazySet()可能比使用set()更好！因为加锁一定是线程安全，保证了可见性，再使用volatile修饰变量其实多余了，而使用lazySet()就可以避免内存屏障，提高程序的执行效率！


### AtomicLong 原子类的使用

AtomicLong原子类与AtomicInteger原子类的作用类似，AtomicLong类可以保证long类型的操作原子性。提到Long类型，有一个需要知道的常识是，在Java中，long型和double型以外的基本类型和引用类型变量的写操作都是原子性操作。但为什么long型和double型变量写操作不保障原子性操作呢？【可参考我另外一篇博客：[https://www.jianshu.com/p/45f7e31c6792](https://www.jianshu.com/p/45f7e31c6792)】

>因为long和double类型都是64位(8字节)的存储空间。在32位的Java虚拟机下的写操作可能被分为两个步骤操作，先写入低32位，再写入高32位，这样在多线程环境下共享long或者double类型变量，可能出现A线程对变量执行低32位操作，B线程执行高32位操作，导致变量值出错。



# 总结
通过上面的讲解，相信大家对AtomicInteger原子类都有一定的了解。只有熟悉了AtomicInteger原子类，那么肯定也会使用AtomicLong和AtomicBoolean原子类，因为基本数据类型的原子类实现基本相同，因此对AtomicLong和AtomicBoolean原子类不再赘述。



参考资料：
[https://segmentfault.com/a/1190000015825207?utm_source=tag-newest](https://segmentfault.com/a/1190000015825207?utm_source=tag-newest)









