上一篇文章详细讲解了AtomicInteger原子类，还有和AtomicInteger原子类实现原理基本一样的AtomicLong和AtomicBoolean原子类。这些都是基本数据类型的原子类，在并发情景下可以保证基本数据类型变量的原子性。但是对于引用类型，这些基本类型的原子类就无能为力了，所以就出现**对象引用类型的原子类**。

对象引用类型的原子类包括：**AtomicReference、AtomicStampedReference、AtomicMarkableReference**

AtomicReference原子类与基本数据类型的原子类实现过程相似，故不再赘述。

不过值得注意的是，使用CAS会有ABA的隐患！[什么是ABA？](https://en.wikipedia.org/wiki/ABA_problem)[知乎用户对ABA相关的提问](https://www.zhihu.com/question/23281499)


所以AtomicStampedReference、AtomicMarkableReference两个原子类就大派用场啦！


AtomicStampedReference 实现的 CAS 方法增加了版本号参数


```java
public boolean compareAndSet(V expectedReference, V newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }
```




参考资料：

[https://en.wikipedia.org/wiki/ABA_problem](https://en.wikipedia.org/wiki/ABA_problem)

[https://www.zhihu.com/question/23281499](https://www.zhihu.com/question/23281499)