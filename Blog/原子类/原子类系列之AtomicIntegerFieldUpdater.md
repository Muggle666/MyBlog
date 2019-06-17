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

使用AtomicIntegerFieldUpdater












总结

参考资料：