## 简介

原子化数组包括：AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray。在并发环境下，数组的操作都是原子化。

有趣的是，AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray这三个原子类的实现基本一样，许多方法也和基本数据类型的原子类相似，因此也不作过多的解释。如果对基本数据类型的原子类不太熟悉，请先浏览我对基本数据类型的原子类解释。【传送门~】

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
	// 当scale为 4 ，Integer.numberOfLeadingZeros(scale) == 29 ，所以shift的值为 2
	// numberOfLeadingZeros
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }

    // 
    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);

        return byteOffset(i);
    }

    private static long byteOffset(int i) {
        return ((long) i << shift) + base;
    }
```



总结




参考资料

