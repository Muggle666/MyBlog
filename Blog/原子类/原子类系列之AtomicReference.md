上一篇文章详细讲解了AtomicInteger原子类，还有和AtomicInteger原子类实现原理基本一样的AtomicLong和AtomicBoolean原子类。这些都是基本数据类型的原子类，在并发情景下可以保证基本数据类型变量的原子性。但是对于引用类型，这些基本类型的原子类就无能为力了，所以就出现**对象引用类型的原子类**。

对象引用类型的原子类包括：**AtomicReference、AtomicStampedReference、AtomicMarkableReference**

AtomicReference原子类与基本数据类型的原子类实现过程相似，故不再赘述。

不过值得注意的是，使用CAS会有ABA的隐患！[什么是ABA？](https://en.wikipedia.org/wiki/ABA_problem)[知乎用户对ABA相关的提问](https://www.zhihu.com/question/23281499)

举个栗子：

分别有两个线程A、B同时访问一个共享变量，线程A比较变量的值，如果为1就改为2，而线程B比较变量值，如果变量值为1则先改为100再改为1（ABA）。

```java
public class ABA {
    // 注意：如果引用类型是Long、Integer、Short、Byte、Character一定一定要注意包装类的缓存区间！
    // 比如Long、Integer、Short、Byte缓存区间是在-128~127，会直接存在常量池中，而不在这个区间内对象的值则会每次都new一个对象，那么即使两个对象的值相同，CAS方法都会返回false
    // 先声明初始值，修改后的值和临时的值是为了保证使用CAS方法不会因为对象不一样而返回false
    private static final Integer INIT_NUM = 1;
    private static final Integer SET_NUM = 2;
    private static final Integer TEM_NUM = 100;

    private static AtomicReference<Integer> atomicReference = new AtomicReference<>(INIT_NUM);

    public static void main(String[] args) {

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " 初始值为：" + atomicReference.get());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " CAS操作结果：" + atomicReference.compareAndSet(INIT_NUM, SET_NUM));
            System.out.println(Thread.currentThread().getName() + " 修改后值为：" + atomicReference.get());
        }, "线程A").start();

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " 初始值为：" + atomicReference.get());
            Thread.yield();//确保线程A先执行
            atomicReference.set(TEM_NUM);
            System.out.println(Thread.currentThread().getName() + " 修改后值为：" + atomicReference.get());
            atomicReference.set(INIT_NUM);
            System.out.println(Thread.currentThread().getName() + " 修改后值为：" + atomicReference.get());
        }, "线程B").start();
    }
}
```


输出结果：











参考资料：

[https://en.wikipedia.org/wiki/ABA_problem](https://en.wikipedia.org/wiki/ABA_problem)

[https://www.zhihu.com/question/23281499](https://www.zhihu.com/question/23281499)