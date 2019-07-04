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

```java
public class ThreadLocal<T> {

        // 防止哈希冲突，当数组大小为2的 N 次方的时候，哈希值能平均的分布
        private final int threadLocalHashCode = nextHashCode();

        private static AtomicInteger nextHashCode =
                new AtomicInteger();

        private static final int HASH_INCREMENT = 0x61c88647;

        private static int nextHashCode() {
            return nextHashCode.getAndAdd(HASH_INCREMENT);
        }
}
```





先看下ThreadLocal类的源码：

```java
public class ThreadLocal<T> {

        // 防止哈希冲突，当数组大小为2的 N 次方的时候，哈希值能平均的分布
        private final int threadLocalHashCode = nextHashCode();

        private static AtomicInteger nextHashCode =
                new AtomicInteger();

        private static final int HASH_INCREMENT = 0x61c88647;

        private static int nextHashCode() {
            return nextHashCode.getAndAdd(HASH_INCREMENT);
        }

        // 初始化的值为null
        protected T initialValue() {
            return null;
        }

        // 初始值
        public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
            return new SuppliedThreadLocal<>(supplier);
        }

        public ThreadLocal() { }

        public T get() {
            Thread t = Thread.currentThread();
            ThreadLocalMap map = getMap(t);
            // 如果ThreadLocalMap为null则执行setInitialValue方法初始化
            if (map != null) {
                Entry e = map.getEntry(this);
                if (e != null) {// 如果Entry对象为null，代表当前线程的ThreadLocal值没有被初始化，向下执行setInitialValue方法
                    @SuppressWarnings("unchecked")
                    T result = (T) e.value;
                    return result;
                }
            }
            return setInitialValue();
        }

        private T setInitialValue() {
            T value = initialValue();
            Thread t = Thread.currentThread();
            ThreadLocalMap map = getMap(t);
            //如果ThreadLocalMap对象存在，则设置当前线程的ThreadLocal值为null
            if (map != null)
                map.set(this, value);
            else
                createMap(t, value);// 初始化ThreadLocalMap对象
            return value;
        }

        public void set(T value) {
            Thread t = Thread.currentThread();
            ThreadLocalMap map = getMap(t);
            if (map != null)
                map.set(this, value);
            else
                createMap(t, value);
        }

        // 删去当前线程ThreadLocal对象存储的值
        public void remove() {
            ThreadLocalMap m = getMap(Thread.currentThread());
            if (m != null)
                m.remove(this);
        }

        // 获取ThreadLocalMap对象
        ThreadLocalMap getMap(Thread t) {
            return t.threadLocals;
        }

        // 创建ThreadLocalMap对象
        void createMap(Thread t, T firstValue) {
            t.threadLocals = new ThreadLocalMap(this, firstValue);
        }

        static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
            return new ThreadLocalMap(parentMap);
        }

        T childValue(T parentValue) {
            throw new UnsupportedOperationException();
        }

        // 下面两个内部类等会再介绍
        static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {...}

        static class ThreadLocalMap {...}
    }
```


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

        // Entry数组的初始容量
        private static final int INITIAL_CAPACITY = 16;

        // Entry数组
        private Entry[] table;

        // Entry数组元素的个数
        private int size = 0;

        // Entry扩容的阀值
        private int threshold; // Default to 0

        // 设置Entry数组的阀值，长度为 len 的 2/3 倍
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        // Entry数组的下一个索引
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        // Entry数组的上一个索引
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }

        // 初始化ThreadLocalMap对象
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);// 初始化Entry
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }

        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }

        // 获取Entry元素
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            // 如果索引对应的元素不是null，而且元素的索引与参数key相同
            // （为什么会存在不同的情况呢？这是因为有hash冲突的情况），否则执行getEntryAfterMiss方法再去确认key对应的value值
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;
            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);// 清除key为i的键值对
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            // 如果数组的索引为key的哈希对应的值不为null则进入for循环
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                if (k == key) {// 如果key存在就更新
                    e.value = value;
                    return;
                }
                // 如果key为null则说明该entry已经失效，执行replaceStaleEntry替换掉
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            // 向数组新增Entry对象元素
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold) // 清除一些过期的值并且判断是否需要扩容
                rehash();// 扩容
        }

        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;// 获取Entry数组
            int len = tab.length;// 获取数组长度
            int i = key.threadLocalHashCode & (len-1);// 获取数组元素索引
            // 如果索引对应的数组元素不为null，则循环Entry数组
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();// 当前元素设置为null，即删去当前元素
                    expungeStaleEntry(i);// 重组数组
                    return;
                }
            }
        }

        // 将新元素放进陈旧的元素
        private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

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
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }

        // 清除目标对象，并向后扫描清除被弃用的元素
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            Entry e;
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

        // 清除被弃用的元素
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }

        // 清除弃用元素并判断是否需要扩容
        private void rehash() {
            expungeStaleEntries();
            if (size >= threshold - threshold / 4)
                resize();
        }

        // 扩容
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
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

        // 清空被弃用的元素
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
    }
```




















参考资料：

[https://blog.csdn.net/y4x5M0nivSrJaY3X92c/article/details/81124944](https://blog.csdn.net/y4x5M0nivSrJaY3X92c/article/details/81124944)







