在日常开发中，基本数据类型的原子类最常用的可能就是AtomicInteger类了。话说回来，为什么有Integer类还需要有AtomicInteger类呢？先来看看AtomicInteger类的包是啥：**java.util.concurrent.atomic** 。看到没有，这是并发包下的类，所以AtomicInteger类肯定是在并发环境下使用的。

先看一条常见的面试题：x作为全局变量，使用 x++ 线程安全吗？

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
99993
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
100000
```

为什么使用 x 作为全局变量，使用 x++ 线程不安全，而使用AtomicInteger会线程安全呢？

以上一直强调x是作为全局变量，是因为全局变量相当于共享变量，在多线程环境下多个线程访问同一个变量会出现线程不安全。x++ 相当于 x = x + 1 ，虽然在高级语言中是一行代码，但对于计算机而言，所有的操作都是指令，而 x++ 这一操作在计算机转化为指令可以理解为有3个步骤：
>1.获取变量的当前值
2.变量当前值加1
3.将新的值赋值给变量

假设现在只有A、B两个线程同时访问共享变量 x ，并都执行 x++ 操作。当线程A访问变量 x 的值为0，并加1，但没来得及将新的值赋值给共享变量 x ，线程B就访问共享变量 x ，此时 x 的值依然为0，然后线程A、线程B分别执行剩下的指令后发现，线程A和线程B的两次 x++ 操作的结果为共享变量x 的值为1。解决这个线程安全的问题，可以使用sychronized或者使用Lock加锁解决，也可以使用AtomicInteger类更加优雅的解决，因为java.util.concurrent.atomic包下的类都能够保证原子性,而且最大的好处就是性能，因为使用AtomicInteger是无锁的，对计算机而言，避免了加锁、解锁和线程上下文切换的操作。

好了，经过前面的铺垫，接下来开始对AtomicInteger类进一步的探索！

先了解AtomicInteger有哪些方法和变量吧！
![AtomicInteger-UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Atomic/AtomicInteger/AtomicInteger-UML.jpg)

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
```java
返回当前AtomicInteger变量的值：0
返回x++的值：0 ，X的当前值为1
返回++x的值：2 ，X的当前值为2
返回x--的值：2 ，X的当前值为1
返回--x的值：0 ，X的当前值为0
返回x+=10前的值：0 ，X的当前值为10
返回x+=10后的值：20 ，X的当前值为20
函数的结果更新当前值，返回更新前的值：20 ，X的当前值为100
函数的结果更新当前值，返回更新后的值：666 ，X的当前值为666
使用IntBinaryOperator对当前值和第一个参数进行计算，并更新当前值，返回计算前的值：666 ，X的当前值为766
使用IntBinaryOperator对当前值和第一个参数进行计算，并更新当前值，返回计算后的值：866 ，X的当前值为866
设定当前的值，返回旧值：866 ，X的当前值为10
设定X的当前值为30
X的当前值为20
x当前值是否为20：true ，X的当前值为10
x当前值是否为10：true ，X的当前值为20
将当前值转为short类型：true
将当前值转为int类型：true
将当前值转为long类型：true
将当前值转为double类型：true
将当前值转为float类型：true
```


值得注意的是，20~32行的示例中的getAndAccumulate、accumulateAndGet、lazySet、compareAndSet和weakCompareAndSet方法。
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
getAndAccumulate()和accumulateAndGet()第一个参数是需要操作







原子化的基本数据类型相关的实现有AtomicInteger、AtomicLong和AtomicBoolean。
