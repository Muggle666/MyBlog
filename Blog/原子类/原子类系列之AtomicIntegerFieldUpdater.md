之前写了几篇关于基本数据类型的原子类、数组类型的原子类和引用对象类型的原子类，重点介绍了AtomicInteger、AtmoicIntegerArray和AtomicStampedReference原子类。而接下来这篇文章，重点AtomicIntegerFieldUpdater原子类！

# 简介


**AtomicIntegerFieldUpdater**原子类字面意思可以理解为：整型字段原子更新器。可以原子地修改对象的属性，与其类似的还有**AtomicLongFieldUpdater**和**AtomicReferenceFieldUpdater**。

#### 思考：
>AtomicIntegerFieldUpdater和AtomicInteger有什么区别呢？AtomicInteger也可以原子化地修改对象的属性呀，那AtomicIntegerFieldUpdater的应用场景是什么？

首先回顾一下AtomicInteger原子类的使用，当我们声明一个变量为AtomicInteger原子类，累加操作等算法操作都是调用getXXX()、setXXX()之类的方法。

```java
AtomicInteger x = new AtomicInteger(1);
x.incrementAndGet();
x.set(10);
```


那如果在原有的代码中，一个对象的属性 x 只是用基本数据类型或者引用类型声明，通过getter、setter等方式修改变量的值，在并发情况下无法保证原子化，有可能导致数据异常。此时如果变量 x
 由int类型变成AtomicInteger原子类声明，虽然可以保证原子性，但在每一处使用到这个变量 x 的地方都要修改为AtomicInteger原子类相应的方法，这很明显违背了设计模式中的六大原则——“开闭原则”！

```java
//原有的代码
int x = 0;
public void setX(){...}
public int getX(){...}

//为了保证原子性，修改后的代码
AtomicInteger x = new AtomicInteger(0);
x.set(...);
x.get();
```

所以此时使用AtomicIntegerFieldUpdater原子类就大派用场啦！我认为使用AtomicIntegerFieldUpdater原子类的使用场景最主要的就是可以不用修改过多的代码就可以保证代码的原子性操作！


### AtomicIntegerFieldUpdater原子类的使用

使用AtomicIntegerFieldUpdater原子类有三点注意事项：
1.变量必须使用volatile关键字修饰。
>使用volatile是为了保证可见性，如果没有volatile关键字修饰，使用newUpdater()会抛出IllegalArgumentException异常。

2.不能使用static关键字。
>

3.不能使用final关键字。
>这个容易理解，因为需要修改变量嘛，设置final就不能修改变量啦。

4.变量的描述符类型必须与调用者一致。如果调用者能够调用变量就能够通过反射操作保证原子性。


示例：
```java
1. public class AtomicIntegerFieldUpdaterDemo {
2. 
3.     //设置100000个线程，模拟并发场景
4.     private static final int THREAD_NUM = 100000;
5. 
6.     //设置栅栏是为了防止循环还没结束就执行main线程输出自增的变量，导致误以为线程不安全
7.     private static CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUM);
8. 
9.     private static AtomicIntegerFieldUpdater atomicIntegerFieldUpdater = AtomicIntegerFieldUpdater.newUpdater(Score.class, "totalScore");
10. 
11.     public static void main(String[] args) throws InterruptedException {
12.         Score score = new Score();
13.         for (int j = 0; j < THREAD_NUM; j++) {
14.             new Thread(() -> {
15.                 atomicIntegerFieldUpdater.incrementAndGet(score);
16.                 countDownLatch.countDown();
17.             }).start();
18.         }
19.         countDownLatch.await();
20.         System.out.println("totalScore的值：" + atomicIntegerFieldUpdater.get(score));
21.     }
22. }
23. 
24. class Score {
25.     public volatile int totalScore = 0;
26. 
27.     public int getTotalScore() {
28.         return totalScore;
29.     }
30. 
31.     public void setTotalScore(int totalScore) {
32.         this.totalScore = totalScore;
33.     }
34. }
```
输出结果：
```java
totalScore的值：100000
```

第9行AtomicIntegerFieldUpdater调用newUpdater(Class&lt;U&gt tclass, String fieldName)静态方法创建AtomicIntegerFieldUpdater对象，通过AtomicIntegerFieldUpdater的对象就可以保证对象属性为参数fieldName的原子性，示例中为Score类中的totalScore属性。



先看下AtomicIntegerFieldUpdater原子类有哪些方法：

![AtomicIntegerFieldUpdater-UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Atomic/AtomicIntegerFieldUpdater/AtomicIntegerFieldUpdater-UML.png)


咦？为什么AtomicIntegerFieldUpdater原子类内部又有AtomicIntegerFieldUpdaterImpl私有类，并且继承AtomicIntegerFieldUpdater类？原来AtomicIntegerFieldUpdater是一个抽象类，内部通过AtomicIntegerFieldUpdaterImpl类实现父类。
```java
public abstract class AtomicIntegerFieldUpdater<T> {...}
private static final class AtomicIntegerFieldUpdaterImpl<T> extends AtomicIntegerFieldUpdater<T> {...}
```

AtomicIntegerFieldUpdater原子类的构造函数修饰符为protect，提供一个静态方法newUpdater()创建AtomicIntegerFieldUpdater的对象。

```java
    @CallerSensitive
    public static <U> AtomicIntegerFieldUpdater<U> newUpdater(Class<U> tclass,
                                                              String fieldName) {
        return new AtomicIntegerFieldUpdaterImpl<U>
            (tclass, fieldName, Reflection.getCallerClass());
    }
```

**@CallerSensitive** 这个是JVM的注释，但目前我还找不到明确的解释，但兴趣的朋友可以参考这位大佬写的博客【[JVM注解@CallerSensitive](https://blog.csdn.net/HEL_WOR/article/details/50199797)】

通过





总结

参考资料：