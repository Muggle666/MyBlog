上一篇文章详细讲解了AtomicInteger原子类，还有和AtomicInteger原子类实现原理基本一样的AtomicLong和AtomicBoolean原子类。这些都是基本数据类型的原子类，在并发情景下可以保证基本数据类型变量的原子性。但是对于引用类型，这些基本类型的原子类就无能为力了，所以就出现**对象引用类型的原子类**。

对象引用类型的原子类包括：**AtomicReference、AtomicStampedReference、AtomicMarkableReference**

AtomicReference原子类与基本数据类型的原子类实现过程相似，故不再赘述。

不过值得注意的是，使用CAS会有ABA的隐患！[什么是ABA？](https://en.wikipedia.org/wiki/ABA_problem)[知乎用户对ABA相关的提问](https://www.zhihu.com/question/23281499)


所以AtomicStampedReference、AtomicMarkableReference两个原子类就大派用场啦！

先看下AtomicStampedReference 原子类的核心源码：



```java
public class AtomicStampedReference<V> {
    // Pair对象维护对象的引用和对象标志的版本，通过Pair对象解决“ABA”问题
    private static class Pair<T> {
        final T reference;// 对象的引用
        final int stamp;//  对象标志的版本

        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }

        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }

    private volatile Pair<V> pair;

    public AtomicStampedReference(V initialRef, int initialStamp) { pair = Pair.of(initialRef, initialStamp); }

    public V getReference() { return pair.reference; }

    public int getStamp() { return pair.stamp; }

    public V get(int[] stampHolder) {
        Pair<V> pair = this.pair;
        stampHolder[0] = pair.stamp;
        return pair.reference;
    }

    public boolean weakCompareAndSet(V expectedReference,
                                     V newReference,
                                     int expectedStamp,
                                     int newStamp) {
        return compareAndSet(expectedReference, newReference,
                expectedStamp, newStamp);
    }

    /**
     * @param expectedReference 期待的原始对象
     * @param newReference      将要更新的对象
     * @param expectedStamp     期待原始对象的标志版本
     * @param newStamp          将要更新对象的标志版本
     */
    public boolean compareAndSet(V expectedReference,
                                 V newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
                expectedReference == current.reference && // 如果返回true则说明当前的
                        expectedStamp == current.stamp &&
                        ((newReference == current.reference &&
                                newStamp == current.stamp) ||
                                casPair(current, Pair.of(newReference, newStamp)));
    }

    public void set(V newReference, int newStamp) {
        Pair<V> current = pair;
        if (newReference != current.reference || newStamp != current.stamp)
            this.pair = Pair.of(newReference, newStamp);
    }

    public boolean attemptStamp(V expectedReference, int newStamp) {
        Pair<V> current = pair;
        return
                expectedReference == current.reference &&
                        (newStamp == current.stamp ||
                                casPair(current, Pair.of(expectedReference, newStamp)));
    }

    private static final sun.misc.Unsafe UNSAFE = sun.misc.Unsafe.getUnsafe();
    private static final long pairOffset =
            objectFieldOffset(UNSAFE, "pair", AtomicStampedReference.class);

    private boolean casPair(Pair<V> cmp, Pair<V> val) {
        return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
    }

    static long objectFieldOffset(sun.misc.Unsafe UNSAFE,
                                  String field, Class<?> klazz) {
        try {
            return UNSAFE.objectFieldOffset(klazz.getDeclaredField(field));
        } catch (NoSuchFieldException e) {
            // Convert Exception to corresponding Error
            NoSuchFieldError error = new NoSuchFieldError(field);
            error.initCause(e);
            throw error;
        }
    }
}
```

AtomicStampedReference 实现的 CAS 方法增加了版本号参数


参考资料：

[https://en.wikipedia.org/wiki/ABA_problem](https://en.wikipedia.org/wiki/ABA_problem)

[https://www.zhihu.com/question/23281499](https://www.zhihu.com/question/23281499)