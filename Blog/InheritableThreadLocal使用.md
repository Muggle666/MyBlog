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