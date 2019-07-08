ThreadLocal的变量作为线程的局部变量，在线程之间互不联系，但是有一种场景下，子线程需要用到父线程的值，此时ThreadLocal的变量就无法传递。所以 JDK 提供了InheritableThreadLocal类。这个类可以解决子线程无法访问父线程的变量的问题，首先看下InheritableThreadLocal类有哪些方法吧！

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
可以看出，InheritableThreadLocal类继承ThreadLocal类，并且重写了三个方法，在分析InheritableThreadLocal类之前，先通过例子看下子线程是不是真的可以访问父类的本地变量。

```java
public class InheritableThreadLocalDemo {

    private static InheritableThreadLocal inheritableThreadLocal = new InheritableThreadLocal();

    private static ThreadLocal threadLocal = new ThreadLocal();

    public static void main(String[] args) {
        threadLocal.set("ThreadLocal变量");
        inheritableThreadLocal.set("InheritableThreadLocal变量");

        System.out.println(Thread.currentThread().getName() + "线程  " + threadLocal.get());
        System.out.println(Thread.currentThread().getName() + "线程  " + inheritableThreadLocal.get());

        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "  " + threadLocal.get());
            System.out.println(Thread.currentThread().getName() + "  " + inheritableThreadLocal.get());
        },"子线程").start();
    }
}
```
输出结果：
```java
main线程  ThreadLocal变量
main线程  InheritableThreadLocal变量
子线程  null
子线程  InheritableThreadLocal变量
```

由例子可以看出，使用 ThreadLocal 类的变量在子线程中是无法访问的，而
 InheritableThreadLocal 类的变量的确

