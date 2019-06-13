## 简介

原子化数组包括：AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray。在并发环境下，数组的操作都是原子化。

有趣的是，AtomicIntegerArray、AtomicLongArray 和 AtomicReferenceArray这三个原子类的实现基本一样，许多方法也和基本数据类型的原子类相似，因此也不作过多的解释。如果对基本数据类型的原子类不太熟悉，请先浏览我对基本数据类型的原子类解释。【传送门~】

不过有一处地方与基本数据类型的原子类很不一样的地方就是数组的原子类都对数组的存储进行优化，通过


总结




参考资料

