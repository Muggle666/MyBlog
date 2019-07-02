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

## ThreadLocal源码剖析

先看下ThreadLocal类的类图：
![ThreadLocal类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal-UML.jpg)

可以看出ThreadLocal有两个静态内部类，分别是**SuppliedThreadLocal**和**ThreadLocalMap**。

### SuppliedThreadLocal内部类
可以通过SuppliedThreadLocal内部类初始化ThreadLocal类的值，而SuppliedThreadLocal继承ThreadLocal和声明了一个Supplier变量，而Supplier变量是一个标志性接口，因此可以通过使用Lambda表达式初始化值。
```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}

static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() { return supplier.get(); }
}
```

初始化ThreadLocal类的值有两种方式，第一种方式就是实现Supplier接口的get()方法或者使用Lambda表达式，第二种方式就是重写initialValue()方法。

```java
    // 初始化ThreadLocal的值————第一种方法：实现抽象方法
    private static ThreadLocal threadLocal = ThreadLocal.withInitial(new Supplier<String>() {
        @Override
        public String get() {
            return "Initial value";
        }
    });

    // 使用Lambda表达式
    private static ThreadLocal threadLocal = ThreadLocal.withInitial(()->{return "Initial value";});

    // 初始化ThreadLocal的值————第二种方式重写initialValue()方法
    private static ThreadLocal threadLocal = new ThreadLocal(){
        @Override
        protected Object initialValue() {
            return "Initial value";
        }
    };
```

### ThreadLocalMap内部类

ThreadLocalMap是一个自定义的哈希映射，是用来维护线程本地变量的值，其中ThreadLocalMap的key使用WeakReference修饰，这有利于垃圾回收，提高系统的性能。


```java
    static class ThreadLocalMap {

        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        private static final int INITIAL_CAPACITY = 16;

        private ThreadLocal.ThreadLocalMap.Entry[] table;

        private int size = 0;

        private int threshold; // Default to 0

        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }

        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new ThreadLocal.ThreadLocalMap.Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new ThreadLocal.ThreadLocalMap.Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }

        private ThreadLocalMap(ThreadLocal.ThreadLocalMap parentMap) {
            ThreadLocal.ThreadLocalMap.Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new ThreadLocal.ThreadLocalMap.Entry[len];

            for (int j = 0; j < len; j++) {
                ThreadLocal.ThreadLocalMap.Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        ThreadLocal.ThreadLocalMap.Entry c = new ThreadLocal.ThreadLocalMap.Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }

        private ThreadLocal.ThreadLocalMap.Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            ThreadLocal.ThreadLocalMap.Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

        private ThreadLocal.ThreadLocalMap.Entry getEntryAfterMiss(ThreadLocal<?> key, int i, ThreadLocal.ThreadLocalMap.Entry e) {
            ThreadLocal.ThreadLocalMap.Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            ThreadLocal.ThreadLocalMap.Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new ThreadLocal.ThreadLocalMap.Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        private void remove(ThreadLocal<?> key) {
            ThreadLocal.ThreadLocalMap.Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }

        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            ThreadLocal.ThreadLocalMap.Entry[] tab = table;
            int len = tab.length;
            ThreadLocal.ThreadLocalMap.Entry e;

            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            tab[staleSlot].value = null;
            tab[staleSlot] = new ThreadLocal.ThreadLocalMap.Entry(key, value);

            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

        private int expungeStaleEntry(int staleSlot) {
            ThreadLocal.ThreadLocalMap.Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            ThreadLocal.ThreadLocalMap.Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }

        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            ThreadLocal.ThreadLocalMap.Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                ThreadLocal.ThreadLocalMap.Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }

        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }

        private void resize() {
            ThreadLocal.ThreadLocalMap.Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            ThreadLocal.ThreadLocalMap.Entry[] newTab = new ThreadLocal.ThreadLocalMap.Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                ThreadLocal.ThreadLocalMap.Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }

        private void expungeStaleEntries() {
            ThreadLocal.ThreadLocalMap.Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                ThreadLocal.ThreadLocalMap.Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
    }
```




















参考资料：









