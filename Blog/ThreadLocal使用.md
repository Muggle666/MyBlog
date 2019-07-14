### <a href="#ThreadLocalUse">ThreadLocal的基本使用</a>

### <a href="#ThreadLocalSource">ThreadLocal源码剖析</a>

- #### <a href="#ThreadLocalSet">set()相关的源码</a>
- 
- #### <a href="#ThreadLocalGet">get()相关的源码</a>
- 
- #### <a href="#ThreadLocalRemove">remove()相关的源码</a>
- 
- #### <a href="#ThreadLocalWithInital">withInitial()相关的源码</a>
- 
- #### <a href="#ThreadLocalInheritableThreadLocal">InheritableThreadLocal对象</a>

### <a href="#summarize">总结</a>

**前提：** 看ReentrantReadWriteLock源码的时候，发现其内部声明了一个内部类ThreadLocalHoldCounter，而这个内部类继承ThreadLocal类，后来粗读了ReentrantReadWriteLock的源码，发现ThreadLocalHoldCounter这个类发挥及其重要的作用，因此我决定将ThreadLocal类好好研究一番~


## <span style="color:red">什么是ThreadLocal</span>


用ThreadLocal声明的变量可以在线程内部提供变量副本，线程修改ThreadLocal声明的变量互不影响，这就不存在并发的情况了。

从 JVM 角度来说，每个线程对应的线程对象实例都在 JVM 堆中，而 ThreadLocal 声明的变量副本就存在各自线程对象中。如下图所示：

![Sample](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal_sample.png)

## <span style="color:red">ThreadLocal的使用场景有哪些呢？</span>

不需要共享的变量。比如Web中，每个用户的requestid不一样，使用ThreadLocal声明就可以有效的避免线程之间的竞争，无需采取同步措施，因此可以简单的理解为用空间换时间。


## <a name="ThreadLocalUse"><span style="color:red">ThreadLocal的基本使用</span></a>

先看下ThreadLocal的API有哪些方法。
|方法名|用法|
|-|-|
|T get()|返回当前线程局部变量副本的变量值|
|void set(T value)|设置当前线程局部变量副本的变量值为指定值|
|void remove()|删除当前线程局部变量副本的变量值|
|static \<S> ThreadLocal\<S> withInitial(Supplier<? extends S> supplier)|返回当前线程局部变量副本的变量初始值。|

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

## <a name="ThreadLocalSource"><span style="color:red">ThreadLocal源码剖析</span></a>

先看下ThreadLocal类的类图：

![ThreadLocal类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal-UML.jpg)

可以看出ThreadLocal有两个静态内部类，分别是**SuppliedThreadLocal**和**ThreadLocalMap**。实际上，ThreadLocal 类的核心就是 ThreadLocalMap 这个内部类。当创建线程的时候，线程对象都会有 ThreadLocalMap 类型的成员变量。

##### Thread类的部分源码：
```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

实际上，ThreadLocalMap是一个数组，而数组内的元素都是由key和value组成的Entry对象。ThreadLocalMap的key就是经过哈希算法计算出来的ThreadLocal对象。神奇的是，ThreadLocal的哈希算法可以保证只要在ThreadLocalMap数组长度为2的 N 次方的时候，哈希值能平均的分布,避免键冲突。【[涉及的数学思想比较多，对ThreadLocal哈希算法感兴趣的可以参考](https://blog.csdn.net/y4x5M0nivSrJaY3X92c/article/details/81124944)】

## <a name="ThreadLocalSet"><span style="color:green">1.set()相关的源码：</span></a>

接下来看下ThreadLocal类的set()相关源码，值得注意的是，由于Entry对象是继承**WeakReference**，所以Entry对象是弱引用的，简单来说就是很容易被GC回收，所以在ThreadLocal类中，大部分方法都涉及判断对象是否为null，如果为null就要从数组中移除，避免内存溢出。虽然set方法涉及的源码很多，但理解核心的源码就可以了，就是**要知道ThreadLocal变量是在哪里保存，如何保存的**。

为方便理解，结合以下源码画出流程图（背景色为绿色的是set方法主要的流程图，紫色背景是涉及方法的流程图）：
![ThreadLocal 类 set 方法的 UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal_set-UML.png)
```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);// 获取ThreadLocalMap对象
        // 如果ThreadLocalMap对象不为null，则执行内部类ThreadLocalMap的set方法，否则执行createMap初始化ThreadLocalMap对象
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    // 获取ThreadLocalMap对象，这个对象是在Thread类的局部变量，所以通过Thread类获取对象
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    // 创建ThreadLocalMap对象
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

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

        // ThreadLocalMap对象实际上由Entry数组记录ThreadLocal变量
        private Entry[] table;

        // Entry数组元素的个数
        private int size = 0;

        // Entry扩容的阀值
        private int threshold;

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

        // ThreadLocal.set()主要核心方法
        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len - 1);// ThreadLocal对象经过哈希算法确定元素索引 i

            // 如果数组索引对应的Entry对象不是null，则进入for循环
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
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
            if (!cleanSomeSlots(i, sz) && sz >= threshold)// 清除一些过期的值并且判断是否需要扩容
                rehash();
        }

	// 将新元素放进陈旧的元素
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            int slotToExpunge = staleSlot;
            // 向前查找被弃用的索引
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // 向后查找key或者value为null的元素
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // 如果存在则清除被弃用的Entry对象
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

            // 如果还有其它被弃用的Entry对象，执行cleanSomeSlots方法清除他们
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
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


如果你看完上面的源码，我相信您很容易回答“**ThreadLocal变量是如何保存？在哪里保存？**”这个问题。

首先，ThreadLocal变量都是通过Entry这一个键值对对象保存，ThreadLocal对象作为Entry的key，而value就是ThreadLocal变量的值。而Entry对象是作为元素保存在线程成员变量ThreadLocalMap这个数组中。

理解上面set方法相关的源码，剩下的get()、remove()、withInitial()就容易了，因为大部分源码是相同或者是相似的。

## <a name="ThreadLocalGet"><span style="color:green">2.get()相关的源码：</span></a>

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        // 如果ThreadLocalMap为null则执行setInitialValue方法初始化
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);// 当前ThreadLocal对象作为参数传给getEntry方法
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    // 获取Entry元素
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);// 通过哈希算法计算数组索引
        // 通过索引获取Entry对象
        Entry e = table[i];
        // 如果索引对应的元素不是null（有可能发生GC，导致Entry对象被弃用），
        // 而且参数key与Entry对象的Key相同（有可能hash冲突的情况）则返回 e ，
        // 否则执行getEntryAfterMiss方法再去确认key对应的value值
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }

    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;

        // 如果Entry对象为null，则返回null
        while (e != null) {
            ThreadLocal<?> k = e.get();// 获取Entry对象的key
            if (k == key)// 
                return e;
            if (k == null)
                expungeStaleEntry(i);// 清除索引为i的键值对
            else
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }

    //如果ThreadLocalMap对象存在，则设置当前线程的ThreadLocal值为null
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```


根据上面的源码，画出 get 方法的流程图如下（背景色为绿色的是get方法的主要流程，粉红背景色则是涉及到的方法流程图）：
![ThreadLocal类get方法的UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal_get-UML.png)

## <a name="ThreadLocalRemove"><span style="color:green">3.remove()相关的源码：</span></a>

```java
    public void remove() {
        // 获取当前线程局部变量的ThreadLocalMap对象
        ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null)// 如果对象不是null则执行ThreadLocalMap内部类的remove方法
            m.remove(this);
    }

    private void remove(ThreadLocal<?> key) {
        Entry[] tab = table;
        int len = tab.length;
        // 通过哈希算法计算数组索引
        int i = key.threadLocalHashCode & (len-1);
        for (Entry e = tab[i];// 通过索引获取Entry对象
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            if (e.get() == key) {// 参数key与Entry对象的Key相同
                e.clear();// 设置 e 对象为null
                expungeStaleEntry(i);// 清除key为空的Entry,并将不为空的元素放到合适的位置
                return;
            }
        }
    }
```
根据上面的源码，画出 remove 方法的流程图如下（背景色为绿色的是remove方法的主要流程，紫色背景色则是涉及到的方法流程图）：

![ThreadLocal类remove方法的UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Concurrent/ThreadLocal/ThreadLocal_remove-UML.png)


## <a name="ThreadLocalWithInital"><span style="color:green">4.withInitial()相关的源码：</span></a>

首先通过一个例子深入withInitial方法：
```java
1.         ThreadLocal threadLocal = ThreadLocal.withInitial(new Supplier<String>() {
2.             @Override
3.             public String get() {
4.                 return "Init Value";
5.             }
6.         });
7. 
8.         System.out.println(threadLocal.get());
9. 
10.         // 使用lambda表示式简化
11.         // ThreadLocal threadLocal = ThreadLocal.withInitial(()->"Init value");
```
第八行的输出结果就是：Init Value。

那在ThreadLocal源码中，是怎么实现初始化ThreadLocal的值呢？

```java
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }

    static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

        private final Supplier<? extends T> supplier;

        SuppliedThreadLocal(Supplier<? extends T> supplier) {
            this.supplier = Objects.requireNonNull(supplier);
        }

        @Override
        protected T initialValue() {
            return supplier.get();
        }
    }
```

withInitial的参数类型是Supplier，这是一个标记接口，其作用就是返回一个特定类型的对象。
```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```
例子第2～5行实现了接口get方法，其返回值就是我们自定义的初始化值。这个时候，初始化值其实并没有保存在线程的ThreadLocalMap对象中的，而是在调用ThreadLocal.get()的时候才会把初始值存储起来。下面通过源码展示这一流程：

```java
1.     public T get() {
2.         Thread t = Thread.currentThread();
3.         ThreadLocalMap map = getMap(t);
4.         if (map != null) {
5.             ThreadLocalMap.Entry e = map.getEntry(this);
6.             if (e != null) {
7.                 @SuppressWarnings("unchecked")
8.                 T result = (T)e.value;
9.                 return result;
10.             }
11.         }
12.         return setInitialValue();
13.     }
14. 
15.     private T setInitialValue() {
16.         T value = initialValue();
17.         Thread t = Thread.currentThread();
18.         ThreadLocalMap map = getMap(t);
19.         if (map != null)
20.             map.set(this, value);
21.         else
22.             createMap(t, value);
23.         return value;
24.     }
25. 
26.     static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {
27. 
28.         private final Supplier<? extends T> supplier;
29. 
30.         SuppliedThreadLocal(Supplier<? extends T> supplier) {
31.             this.supplier = Objects.requireNonNull(supplier);
32.         }
33. 
34.         @Override
35.         protected T initialValue() {
36.             return supplier.get();
37.         }
38.     }	
```

当执行ThreadLocal.get()方法的时候，虽然ThreadLocalMap对象不是null，但由于当前线程还没保存ThreadLocal对象，所以并不会执行6～10行的代码，而是执行第12行setInitialValue()方法，其目的就是要初始化当前线程成员变量ThreadLocalMap；接下来就会执行第16行，值得注意的是，当前ThreadLocal对象其实是SuppliedThreadLocal内部类初始化的，所以实际上执行initialValue()方法是执行SuppliedThreadLocal.initialValue()的方法，也就是第35行；继续往下执行第36行的supplier.get()方法，返回的值就是例子中，实现的get()方法return的值；接下来继续往下执行17～23行，这里就是设置当前线程的ThreadLocal的初始化值。


## <a name="ThreadLocalInheritableThreadLocal"><span style="color:green">5.InheritableThreadLocal 对象：</span></a>

ThreadLocal的变量作为线程的成员变量，在线程之间互不联系，但是有一种场景下，子线程需要用到父线程的变量，此时ThreadLocal的变量就无法传递。所以 JDK 提供了InheritableThreadLocal类。这个类可以解决子线程无法获取父线程的变量的问题，首先看下InheritableThreadLocal类有哪些方法吧！

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    protected T childValue(T parentValue) {
        return parentValue;
    }

    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```
可以看出，InheritableThreadLocal类继承ThreadLocal类，并且重写了三个方法，在分析InheritableThreadLocal类之前，先通过例子看下子线程是不是真的可以获取父类的本地变量。

```java
1. public class InheritableThreadLocalDemo {
2. 
3.     private static InheritableThreadLocal inheritableThreadLocal = new InheritableThreadLocal();
4. 
5.     private static ThreadLocal threadLocal = new ThreadLocal();
6. 
7.     public static void main(String[] args) throws InterruptedException {
8.         threadLocal.set("ThreadLocal变量");
9.         inheritableThreadLocal.set("InheritableThreadLocal变量");
10. 
11.         System.out.println(Thread.currentThread().getName() + "线程  " + threadLocal.get());
12.         System.out.println(Thread.currentThread().getName() + "线程  " + inheritableThreadLocal.get());
13. 
14.         Thread thread = new Thread(() -> {
15.             System.out.println(Thread.currentThread().getName() + "  " + threadLocal.get());
16.             System.out.println(Thread.currentThread().getName() + "  " + inheritableThreadLocal.get());
17.         },"子线程");
18. 
19.         inheritableThreadLocal.set("修改 inheritableThreadLocal 变量值");
20.         System.out.println(Thread.currentThread().getName() + "线程  " + inheritableThreadLocal.get());
21.         thread.start();
22.     }
23. }
```
输出结果：
```java
main线程  ThreadLocal变量
main线程  InheritableThreadLocal变量
main线程  修改 inheritableThreadLocal 变量值
子线程  null
子线程  InheritableThreadLocal变量
```

由例子可以看出，使用 ThreadLocal 类的变量在子线程中是无法获取的，而
 InheritableThreadLocal 类的变量的确如上面所说，在子线程中可以获取父线程的变量。

<h6 style="color:red">*值得注意的是，InheritableThreadLocal 类的变量在子线程中只是保存父线程的副本。初始化线程之后，成员变量都是在线程对象中，而线程对象之间的成员变量是互不干扰的，所以无论父线程怎么操作成员变量都不会影响子线程。</h6>

接下来，通过上面的例子结合 InheritableThreadLocal 源码分析。

在第 3 行创建 InheritableThreadLocal 变量之后，接着在第 9 行调用父类 set() 方法设置 InheritableThreadLocal 的变量。

```java
1.     public void set(T value) {
2.         Thread t = Thread.currentThread();
3.         ThreadLocalMap map = getMap(t);
4.         if (map != null)
5.             map.set(this, value);
6.         else
7.             createMap(t, value);
8.     }
```

进入父类的 set() 方法，第三行实际上是执行子类的getMap()方法，而子类的 getMap() 方法返回的是 Thread.inheritableThreadLocals ，也就是Thread类的成员变量 inheritableThreadLocals 。由于 Thread.inheritableThreadLocals 未被初始化，所以值为 null ；接下来 set() 方法继续往下走进入第 7 行执行 createMap() 方法创建 ThreadLocalMap 对象，实际上执行的是子类 createMap() 方法，方法的实现是创建 ThreadLocalMap 对象赋值给 Thread.inheritableThreadLocals 。这时，InheritableThreadLocal 变量被初始化。

但是 InheritableThreadLocal 变量只是在 main 线程被初始化，那子线程怎么能获取父线程的成员变量呢？答案就是：**==初始化子线程的时候，init()方法会获取父线程的成员变量并保存在本线程对象中。==**

```java
    public class Thread implements Runnable {
        ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;

        public Thread() {
            init(null, null, "Thread-" + nextThreadNum(), 0);
        }

        private void init(ThreadGroup g, Runnable target, String name,
                          long stackSize) {
            init(g, target, name, stackSize, null, true);
        }

        private void init(ThreadGroup g, Runnable target, String name,
                          long stackSize, AccessControlContext acc,
                          boolean inheritThreadLocals) {
            // 省略了一些代码

            Thread parent = currentThread();// 获取当前线程（由于子线程还没被初始化，所以获取的是父线程对象）

            // 如果父线程的 inheritableThreadLocals 局部变量不是 null ，就证明父线程有设置变量可以让子线程访问
            if (inheritThreadLocals && parent.inheritableThreadLocals != null)
                this.inheritableThreadLocals =
                        ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);// 创建 ThreadLocalMap 对象，赋值给子线程的局部变量 inheritableThreadLocals
        }
    }
```

由 Thread 类的源码可以看出，在初始化 Thread 对象的时候，会判断父线程是否有设置成员变量可以让子线程访问，如果有的话，创建 ThreadLocalMap 对象，赋值给子线程的成员变量 inheritableThreadLocals 。

```java
1.     public T get() {
2.         Thread t = Thread.currentThread();
3.         ThreadLocalMap map = getMap(t);
4.         if (map != null) {
5.             ThreadLocalMap.Entry e = map.getEntry(this);
6.             if (e != null) {
7.                 @SuppressWarnings("unchecked")
8.                 T result = (T)e.value;
9.                 return result;
10.             }
11.         }
12.         return setInitialValue();
13.     }
```

当例子中的代码执行ThreadLocal.get()方法第三行的时候，getMap()方法实际上是调用InheritableThreadLocal.getMap()，获取的是线程的 inheritableThreadLocals 变量。


## <a name="summarize">总结：</a>

1.ThreadLocal 设计的目的就是为了能够在当前线程中有属于自己的变量，并不是为了解决并发或者共享变量的问题。
2.每个线程都是Thread 类的对象，其中 ThreadLocalMap 作为成员变量保存 ThreadLocal的值；ThreadLocalMap 数据结构是数组 Entry ，数组的索引通过获取 ThreadLocal 对象开地址法实现的，可以在数组长度为 2 的 N 次方情况下保证索引唯一；而 Entry 对象内部结构是 key-value 键值对，其中 ThreadLocal 对象作为Entry 对象的 key。
3.子线程如果要获取父线程的成员变量，可以在父线程中使用 InheritableThreadLocal 声明成员变量。

参考资料：

[https://blog.csdn.net/y4x5M0nivSrJaY3X92c/article/details/81124944](https://blog.csdn.net/y4x5M0nivSrJaY3X92c/article/details/81124944)







 