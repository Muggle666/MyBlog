上一篇文章详细讲解了AtomicInteger原子类，还有和AtomicInteger原子类实现原理基本一样的AtomicLong和AtomicBoolean原子类。这些都是基本数据类型的原子类，在并发情景下可以保证基本数据类型变量的原子性。但是对于引用类型，这些基本类型的原子类就无能为力了，所以就出现**对象引用类型的原子类**。

对象引用类型的原子类包括：**AtomicReference、AtomicStampedReference、AtomicMarkableReference**

AtomicReference原子类与基本数据类型的原子类实现过程相似，故不再赘述。

不过值得注意的是，使用CAS会有ABA的隐患！[什么是ABA？](https://en.wikipedia.org/wiki/ABA_problem)[知乎用户对ABA相关的提问](https://www.zhihu.com/question/23281499)


所以AtomicStampedReference、AtomicMarkableReference两个原子类就大派用场啦！

## AtomicStampedReference 的使用

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
                expectedReference == current.reference && // 如果返回true则说明期待的原始对象与Pair的reference对象一样
                        expectedStamp == current.stamp &&  // 如果返回true说明期待原始对象标志版本与Pair的stamp对象一样
                        ((newReference == current.reference &&
                                newStamp == current.stamp) || // 如果期待更新的对象和标志版本与Pair的reference和stamp一样的话直接返回true，否则执行CAS操作
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
            NoSuchFieldError error = new NoSuchFieldError(field);
            error.initCause(e);
            throw error;
        }
    }
}
```

AtomicStampedReference 实现的 CAS 方法**增加版本号参数stamp**，通过版本号就能够解决“ABA”问题。AtomicStampedReference类中大部分方法都可以根据方法名推测方法是有什么用。getXXX()方法无非就是从Pair对象属性中获取值；set方法将不同的对象或者不同的版本号设置给Pair对象。因此对AtomicStampedReference 的源码不作过多的解释。

示例：线程A和线程B同时访问一个对象，假设线程A执行CAS操作的时候，判断期待的原始对象和

```java
public class AtomicStampedReferenceDemo {

    // 注意：如果引用类型是Long、Integer、Short、Byte、Character一定一定要注意值的缓存区间！
    // 比如Long、Integer、Short、Byte缓存区间是在-128~127，会直接存在常量池中，而不在这个区间内对象的值则会每次都new一个对象，那么即使两个对象的值相同，CAS方法都会返回false
    // 先声明初始值，修改后的值和临时的值是为了保证使用CAS方法不会因为对象不一样而返回false
    private static final Integer INIT_NUM = 1000;
    private static final Integer UPDATE_NUM = 100;
    private static final Integer TEM_NUM = 200;

    private static AtomicStampedReference atomicStampedReference = new AtomicStampedReference(INIT_NUM, 1);

    public static void main(String[] args) {
        new Thread(() -> {
            int value = (int) atomicStampedReference.getReference();
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + " : 当前值为：" + value + " 版本号为：" + stamp);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            if(atomicStampedReference.compareAndSet(value, UPDATE_NUM, stamp, stamp + 1)){
                System.out.println(Thread.currentThread().getName() + " : 当前值为：" + atomicStampedReference.getReference() + " 版本号为：" + atomicStampedReference.getStamp());
            }else{
                System.out.println("版本号不同，更新失败！");
            }
        }, "线程A").start();

        new Thread(() -> {
            Thread.yield();// 确保线程A先执行
            int value = (int) atomicStampedReference.getReference();
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + " : 当前值为：" + value + " 版本号为：" + stamp);
            atomicStampedReference.compareAndSet(atomicStampedReference.getReference(), TEM_NUM, stamp, stamp + 1);
            System.out.println(Thread.currentThread().getName() + " : 当前值为：" + atomicStampedReference.getReference() + " 版本号为：" + atomicStampedReference.getStamp());
            atomicStampedReference.compareAndSet(atomicStampedReference.getReference(), INIT_NUM, stamp, stamp + 1);
            System.out.println(Thread.currentThread().getName() + " : 当前值为：" + atomicStampedReference.getReference() + " 版本号为：" + atomicStampedReference.getStamp());
        }, "线程B").start();
    }
}
```

输出结果：
```java
线程A : 当前值为：1000 版本号为：1
线程B : 当前值为：1000 版本号为：1
线程B : 当前值为：200 版本号为：2
线程B : 当前值为：200 版本号为：2
版本号不同，更新失败！
```

## AtomicMarkableReference 的使用

而AtomicMarkableReference原子类与AtomicStampedReference原子类源码实现相似，区别在于AtomicMarkableReference的标记是Boolean类型，只有两种状态true和false，适用在只需要知道
AtomicMarkableReference对象是否有被修改的情景。

```java
    // Pair对象维护对象的引用和对象标记
    private static class Pair<T> {
        final T reference;
        final boolean mark;// 通过标记的状态区分对象是否有更改

        private Pair(T reference, boolean mark) {
            this.reference = reference;
            this.mark = mark;
        }

        static <T> Pair<T> of(T reference, boolean mark) {
            return new Pair<T>(reference, mark);
        }
    }
    /**
     * @param expectedReference 期待的原始对象
     * @param newReference      将要更新的对象
     * @param expectedMark      期待原始对象的标记
     * @param newMark           将要更新对象的标记
     */
    public boolean compareAndSet(V expectedReference,
                                 V newReference,
                                 boolean expectedMark,
                                 boolean newMark) {
        Pair<V> current = pair;
        return
                expectedReference == current.reference && // 如果期待的原始对象与Pair的reference一样则返回true
                        expectedMark == current.mark && // 如果期待的原始对象的标记与Pair的mark一样则返回true
                        ((newReference == current.reference &&
                                newMark == current.mark) || // 如果要更新的对象和对象标记与Pair的refernce和mark一样的话直接返回true，否则执行CAS操作
                                casPair(current, Pair.of(newReference, newMark)));
    }
```

示例：
```java
public class AtomicMarkableReferenceDemo {

    private static final Integer INIT_NUM = 10;

    private static final Integer TEM_NUM = 20;

    private static final Integer UPDATE_NUM = 100;

    private static final Boolean INITIAL_MARK = Boolean.FALSE;

    private static AtomicMarkableReference atomicMarkableReference = new AtomicMarkableReference(INIT_NUM, INITIAL_MARK);

    public static void main(String[] args) {
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " ： 初始值为：" + INIT_NUM + " , 标记为： " + INITIAL_MARK);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            if (atomicMarkableReference.compareAndSet(INIT_NUM, UPDATE_NUM, atomicMarkableReference.isMarked(), Boolean.TRUE)) {
                System.out.println(Thread.currentThread().getName() + " ： 修改后的值为：" + atomicMarkableReference.getReference() + " , 标记为： " + atomicMarkableReference.isMarked());
            }else{
                System.out.println(Thread.currentThread().getName() +  " CAS返回false");
            }
        }, "线程A").start();

        new Thread(() -> {
            Thread.yield();
            System.out.println(Thread.currentThread().getName() + " ： 初始值为：" + atomicMarkableReference.getReference() + " , 标记为： " + INITIAL_MARK);
            atomicMarkableReference.compareAndSet(atomicMarkableReference.getReference(), TEM_NUM, atomicMarkableReference.isMarked(), Boolean.TRUE);
            System.out.println(Thread.currentThread().getName() + " ： 修改后的值为：" + atomicMarkableReference.getReference() + " , 标记为： " + atomicMarkableReference.isMarked());
        }, "线程B").start();
    }
}
```

输出结果：
```java
线程A ： 初始值为：10 , 标记为： false
线程B ： 初始值为：10 , 标记为： false
线程B ： 修改后的值为：20 , 标记为： true
线程A CAS返回false
```

由于线程B修改了对象，标记有false改为true，所以当上下文切换为线程A的时候，如果标记不一致CAS方法就会返回false。

参考资料：

[https://en.wikipedia.org/wiki/ABA_problem](https://en.wikipedia.org/wiki/ABA_problem)

[https://www.zhihu.com/question/23281499](https://www.zhihu.com/question/23281499)