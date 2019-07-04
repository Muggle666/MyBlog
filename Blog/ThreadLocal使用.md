**前提：** 看ReentrantReadWriteLock源码的时候，发现其内部声明了一个内部类ThreadLocalHoldCounter，而这个内部类继承ThreadLocal类，后来粗读了ReentrantReadWriteLock的源码，发现ThreadLocalHoldCounter这个类发挥及其重要的作用，因此我决定将ThreadLocal类好好研究一番~


## <span style="color:red">什么是ThreadLocal</span>


用ThreadLocal声明的变量可以在线程内部提供变量副本，线程彼此之间修改ThreadLocal声明的变量互不影响，这就不存在并发的情况了。

也就是说，每创建一个线程，栈帧都会保存ThreadLocal声明的变量副本，随着线程的结束而销毁。如下图所示：

![Demo](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal-demo.png)


## <span style="color:red">ThreadLocal的使用场景有哪些呢？</span>

不需要共享的变量。比如Web中，每个用户的requestid不一样，使用ThreadLocal声明就可以有效的避免线程之间的竞争，无需采取同步措施，因此可以简单的理解为用空间换时间。


## <span style="color:red">ThreadLocal的基本使用</span>

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
    // 初始化ThreadLocal的值————第一种方法：实现抽象方法
//    private static ThreadLocal threadLocal = ThreadLocal.withInitial(new Supplier<String>() {
//        @Override
//        public String get() {
//            return "Initial value";
//        }
//    });

    // 初始化ThreadLocal的值————第二种方法：使用Lambda表达式
    private static ThreadLocal threadLocal = ThreadLocal.withInitial(()->{return "Initial value";});

    // 初始化ThreadLocal的值————第三种方式重写initialValue()方法
//    private static ThreadLocal threadLocal = new ThreadLocal(){
//        @Override
//        protected Object initialValue() {
//            return "Initial value";
//        }
//    };

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

## <span style="color:red">ThreadLocal源码剖析</span>

先看下ThreadLocal类的类图：

![ThreadLocal类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal-UML.jpg)

可以看出ThreadLocal有两个静态内部类，分别是**SuppliedThreadLocal**和**ThreadLocalMap**。实际上，ThreadLocal类的核心就是ThreadLocalMap这个内部类。当创建线程的时候，栈帧都会有threadLocals这一个局部变量。

##### Thread类的部分源码：
```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

也就是说，每个线程内部都会有ThreadLocalMap类型的局部变量，用于记录ThreadLocal声明的变量，而且栈帧的局部变量都是ThreadLocal声明的变量的副本，所以线程内部改变ThreadLocal的变量也不会影响其他线程，也不存在并发。如下图所示：

![栈帧内的ThreadLocalMap示意图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/%E6%A0%88%E5%B8%A7%E5%86%85%E7%9A%84ThreadLocalMap%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

实际上，ThreadLocalMap是一个数组，而数组内的元素都是由key和value组成的Entry对象。ThreadLocalMap的key就是经过哈希算法计算出来的ThreadLocal对象。神奇的是，ThreadLocal的哈希算法可以保证只要在ThreadLocalMap数组长度为2的 N 次方的时候，哈希值能平均的分布,避免键冲突。【[涉及的数学思想比较多，对ThreadLocal哈希算法感兴趣的可以参考](https://blog.csdn.net/y4x5M0nivSrJaY3X92c/article/details/81124944)】

接下来看下ThreadLocal类的set()相关源码：
```java

```




















参考资料：

[https://blog.csdn.net/y4x5M0nivSrJaY3X92c/article/details/81124944](https://blog.csdn.net/y4x5M0nivSrJaY3X92c/article/details/81124944)







