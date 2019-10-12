系统通过多线程优化性能，实际上就是将串行操作转换为并行操作，也就是说将同步操作转换为异步操作。在众多并发类中，**FutureTask 类可以接收线程返回的结果，并且可以取消或者中断线程。**

先看下 FutureTask 类的类图结构：
![FutureTask类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/FutureTask/FutureTask%20%E7%B1%BB%E5%9B%BE%E7%BB%93%E6%9E%84.jpg)

由类图可以知道，FutureTask 类是 Runnable 的实现类，所以可以通过线程池 submit() 或者直接 new Thread() 启动线程。

举个栗子：

```language
public class FutureTaskDemo {
    public static void main(String[] args) {
        MyCallableDemo demo = new MyCallableDemo();
//        // 1.直接通过new Thread()启动线程
//        FutureTask task = new FutureTask(demo);
//        new Thread(task).start();
//        try {
//            System.out.println(task.get());
//        } catch (Exception e) {
//            e.printStackTrace();
//        }

        // 2.通过线程池启动线程
        ExecutorService service = Executors.newSingleThreadExecutor();

        Future<String> result = service.submit(demo);
        try {
            System.out.println(result.get());
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            service.shutdown();
        }
    }
}

class MyCallableDemo implements Callable {
    @Override
    public String call() throws Exception {
        return "MuggleLee";
    }
}
```

输出结果：
```language
MuggleLee
```

通过例子可以看出，使用FutureTask类可以接收线程完成后返回的结果。如果使用场景是需要接收线程执行的结果（无论是成功执行的结果还是异常返回的信息），实现Callable接口结合FutureTask实现类接收返回数据是比较常见的一种做法。更为常见的做法是通过使用线程池submit()方法接收返回的结果。

线程池的实现类 ThreadPoolExecutor 提供了3个 submit() 方法支持获取线程返回的结果。
```java
Future<?> submit(Runnable task);
<T> Future<T> submit(Runnable task, T result);
<T> Future<T> submit(Callable<T> task);
```

可以发现，返回值类型都是 Future 接口。那继续看下 Future 接口有哪些抽象方法。
```java
boolean cancel(boolean mayInterruptIfRunning);
boolean isCancelled();
boolean isDone();
V get() throws InterruptedException, ExecutionException;
V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
```
通过方法名很明显可以知道各个抽象方法的作用。

实际上，submit() 方法将线程的执行结果封装成 FutureTask 对象返回的。
```java
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

    // 将 Callable 接口对象封装成 FutureTask 对象
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
```

接下来，重点剖析 FutureTask 类的源码。

```java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;
    }
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;
    }
```
第一个构造方法将参数的Callable对象赋值给 FutureTask 对象的



# 总结

参考资料: