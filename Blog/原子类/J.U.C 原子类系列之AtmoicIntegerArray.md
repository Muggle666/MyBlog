## 简介

原子化数组包括：AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray。在并发环境下，数组的操作都是原子化。

有趣的是，AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray这三个原子类的实现基本一样，许多方法也和基本数据类型的原子类相似，因此也不作过多的解释。如果对基本数据类型的原子类不太熟悉，请先浏览我对基本数据类型的原子类解释。【[](https://www.jianshu.com/p/5d87871b4bf9)】

不过有一处地方与基本数据类型的原子类很不一样的地方就是数组的原子类都对数组的存储进行优化，通过位运算提高程序的效率。

```java
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // 数组在内存中第一个元素的位置 即数组的起始位置的偏移量，base值为16
    private static final int base = unsafe.arrayBaseOffset(int[].class);
    private static final int shift;
    private final int[] array;

    static {
        //scale为每个元素的字节偏移量，int为4字节。scale代表数组每个元素占有的字节数
        int scale = unsafe.arrayIndexScale(int[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
	// Integer.numberOfLeadingZeros(int i)返回的是无符号整型 i 最高非零位前面 0 的个数
	// 当scale为 4 ，Integer.numberOfLeadingZeros(scale) == 29 ，所以shift的值为 2
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }

    // 执行get、set方法都会执行checkedByteOffset()方法，检查索引值是否超出数组长度，如果没超出就执行byteOffset()方法
    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);
        return byteOffset(i);
    }

    // 计算元素的偏移量
    private static long byteOffset(int i) {
        return ((long) i << shift) + base;
    }
```

示例：

```java
public class AtomicIntegerArrayDemo {
    public static void main(String[] args) {
        AtomicIntegerArray array = new AtomicIntegerArray(8);
        array.set(0, 00);
        array.set(1, 10);
        array.set(2, 20);
        array.set(3, 30);
        array.set(4, 40);
        array.set(5, 50);
        for (int i = 0; i < array.length(); i++) {
            System.out.println("索引： " + i + " ，值为：" + array.get(i));
        }

        System.out.println("getAndAdd()先获取指定索引元素的值再加参数，返回修改前的值：" + array.getAndAdd(5,5));
        System.out.println("get()返回指定索引元素的值：" + array.get(5));

        if(array.compareAndSet(5,55,500)){
            System.out.println("compareAndSet()期待的索引值与原数组指定的索引值相同，返回true，修改后的值为：" + array.get(5));
        }else{
            System.out.println("期待的索引值与原数组索引值不同，返回false");
        }

        System.out.println("getAndSet()先获取指定索引元素的值再更新，返回修改前的值：" + array.getAndSet(5,50));
        System.out.println("get()返回指定索引元素的值：" + array.get(5));

        System.out.println("getAndIncrement()自增指定索引的值，返回自增前的值：" + array.getAndIncrement(5));
        System.out.println("incrementAndGet()自增指定索引的值，返回自增后的值：" + array.incrementAndGet(5));
    }
}
```
输出结果：
```java
索引： 0 ，值为：0
索引： 1 ，值为：10
索引： 2 ，值为：20
索引： 3 ，值为：30
索引： 4 ，值为：40
索引： 5 ，值为：50
索引： 6 ，值为：0
索引： 7 ，值为：0
getAndAdd()先获取指定索引元素的值再加参数，返回修改前的值：50
get()返回指定索引元素的值：55
compareAndSet()期待的索引值与原数组指定的索引值相同，返回true，修改后的值为：500
getAndSet()先获取指定索引元素的值再更新，返回修改前的值：500
get()返回指定索引元素的值：50
getAndIncrement()自增指定索引的值，返回自增前的值：50
incrementAndGet()自增指定索引的值，返回自增后的值：52
```



## 总结

AtomicIntegerArray原子类通过数组可以快速的随机访问元素，内部的方法都调用Unsafe对象的方法保证原子性。在并发的场景下，数组应根据实际情况选择AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray 原子类。



