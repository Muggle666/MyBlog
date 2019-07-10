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
1. public class InheritableThreadLocalDemo {
2. 
3.     private static InheritableThreadLocal inheritableThreadLocal = new InheritableThreadLocal();
4. 
5.     private static ThreadLocal threadLocal = new ThreadLocal();
6. 
7.     public static void main(String[] args) {
8.         threadLocal.set("ThreadLocal变量");
9.         inheritableThreadLocal.set("InheritableThreadLocal变量");
10. 
11.         System.out.println(Thread.currentThread().getName() + "线程  " + threadLocal.get());
12.         System.out.println(Thread.currentThread().getName() + "线程  " + inheritableThreadLocal.get());
13. 
14.         new Thread(() -> {
15.             System.out.println(Thread.currentThread().getName() + "  " + threadLocal.get());
16.             System.out.println(Thread.currentThread().getName() + "  " + inheritableThreadLocal.get());
17.         },"子线程").start();
18.     }
19. }
```
输出结果：
```java
main线程  ThreadLocal变量
main线程  InheritableThreadLocal变量
子线程  null
子线程  InheritableThreadLocal变量
```

由例子可以看出，使用 ThreadLocal 类的变量在子线程中是无法访问的，而
 InheritableThreadLocal 类的变量的确如上面所说，在子线程中可以访问父线程的变量。

接下来，通过上面的例子结合 InheritableThreadLocal 源码分析。

在第 2 行创建 InheritableThreadLocal 变量之后，接着在第 9 行调用父类 set() 方法设置 InheritableThreadLocal 的变量。

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

进入父类的 set() 方法，第三行实际上是执行子类的getMap()方法，而子类的 getMap() 方法返回的是 Thread.inheritableThreadLocals ，也就是Thread类的局部变量 inheritableThreadLocals 。由于 Thread.inheritableThreadLocals 未被初始化，所以值为 null ；接下来 set() 方法继续往下走进入第 7 行执行 createMap() 方法创建 ThreadLocalMap 对象，实际上执行的是子类 createMap() 方法，方法的实现是创建 ThreadLocalMap 对象赋值给 Thread.inheritableThreadLocals 。这时，InheritableThreadLocal 变量被初始化。

但是 InheritableThreadLocal 变量只是在 main 线程被初始化，那子线程怎么能访问父线程的局部变量呢？答案就是：**初始化子线程的时候，init()方法会获取父线程的局部变量并保存在本线程的栈帧。**