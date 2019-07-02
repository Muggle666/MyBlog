**前提：** 看ReentrantReadWriteLock源码的时候，发现其内部声明了一个内部类ThreadLocalHoldCounter，而这个内部类继承ThreadLocal类，后来粗读了ReentrantReadWriteLock的源码，发现ThreadLocalHoldCounter这个类发挥及其重要的作用，因此我决定将ThreadLocal类好好研究一番~


## 什么是ThreadLocal


用ThreadLocal声明的变量可以在线程内部提供变量副本，线程彼此之间修改ThreadLocal声明的变量互不影响，这就不存在并发的情况了。

也就是说，每创建一个线程，栈帧都会保存ThreadLocal声明的变量副本，随着线程的结束而销毁。如下图所示：

![Demo](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal-demo.png)


## ThreadLocal的使用场景有哪些呢？

不需要共享的变量。比如Web中，每个用户的requestid不一样，使用ThreadLocal声明就可以有效的避免线程之间的竞争，无需采取同步措施，因此可以简单的理解为用空间换时间。


## ThreadLocal的基本使用

先看下ThreadLocal的API有哪些方法。
|方法名|用法|
|-|-|
|T get()|返回当前线程的局部变量副本的变量值|
|void set(T value)|设置当前线程的局部变量副本的变量值为指定值|
|void remove()|删除当前线程的局部变量副本的变量值|
|static \<S> ThreadLocal\<S> withInitial(Supplier<? extends S> supplier)|返回当前线程的局部变量副本的变量初始值。|

实例：
```java
public class ThreadLocalDemo {
    // 初始化ThreadLocal的值——第一种方法：实现抽象方法
//    private static ThreadLocal threadLocal = ThreadLocal.withInitial(new Supplier<String>() {
//        @Override
//        public String get() {
//            return "Initial value";
//        }
//    });

    // 初始化ThreadLocal的值——第二种方法：使用Lambda表达式
    private static ThreadLocal threadLocal = ThreadLocal.withInitial(()->{return "Initial value";});

    public static void main(String[] args) {
        System.out.println("ThreadLocal的初始值：" + threadLocal.get());
        threadLocal.set("Main方法");
        new Thread(() -> {
            System.out.println("子线程获取ThreadLocal的值：" + threadLocal.get());
            threadLocal.set("Thread线程");
            System.out.println("子线程执行set方法后，子线程获取ThreadLocal的值：" + threadLocal.get());
            threadLocal.remove();
            System.out.println("子线程执行remove方法后，子线程获取ThreadLocal的值：" + threadLocal.get());
        }).start();
        System.out.println("主线程执行set方法后，主线程获取ThreadLocal的值：" + threadLocal.get());
        threadLocal.remove();
        System.out.println("主线程执行remove方法后，主线程获取ThreadLocal的值：" + threadLocal.get());
    }
}
```

输出结果：
```java
ThreadLocal的初始值：Initial value
主线程执行set方法后，主线程获取ThreadLocal的值：Main方法
主线程执行remove方法后，主线程获取ThreadLocal的值：Initial value
子线程获取ThreadLocal的值：Initial value
子线程执行set方法后，子线程获取ThreadLocal的值：Thread线程
子线程执行remove方法后，子线程获取ThreadLocal的值：Initial value
```

上面例子使用了ThreadLocal类API的四个方法，可以看出来十分方便的声明ThreadLocal类和使用，接下来就开始探索源码是怎么实现的！

先看下ThreadLocal类的类图：
![ThreadLocal类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal-UML.jpg)

