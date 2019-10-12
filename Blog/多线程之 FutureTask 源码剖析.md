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

实际上，submit() 方法将线程的执行结果封装成 FutureTask 对象返回。
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

接下来，看下创建 FutureTask 对象的源码。

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
Executor类部分源码：
```java
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```

FutureTask 类的第一个构造方法将参数 Callable 对象赋值给 FutureTask 对象的 callable 属性，并设置 state 变量为 NEW（稍后再解释 callable 和 state 两个变量的作用）；有意思的是第二个构造方法，将第一个参数 Runnable 对象传给 Executor 类的 callable() 方法，再调用已经实现了 Callable 接口的 RunnableAdapter 适配器类，执行 Runnable 对象的 run() 方法。（设计模式中的适配器模式，不熟悉的可以参考我另外一篇拙作：[设计模式之适配器模式](https://www.jianshu.com/p/a993df57812b)）

emmmm...上面几个方法的确比较绕，画了一张流程图辅助理解吧。
![线程池submit方法流程图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/FutureTask/%E7%BA%BF%E7%A8%8B%E6%B1%A0submit%E6%96%B9%E6%B3%95%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## 接下来，重点剖析 FutureTask 类的源码：

### FutureTask 类声明了7种状态：
```java
    /**
     * Possible state transitions:（FutureTask状态的转换）
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    // 任务当前 FutureTask 对象的状态
    private volatile int state;
    // 新建任务（初始状态）
    private static final int NEW = 0;
    // 任务运行中
    private static final int COMPLETING = 1;
    // 任务正常完成
    private static final int NORMAL = 2;
    // 任务异常
    private static final int EXCEPTIONAL = 3;
    // 任务被取消
    private static final int CANCELLED = 4;
    // 任务中断中
    private static final int INTERRUPTING = 5;
    // 任务被中断
    private static final int INTERRUPTED = 6;

    // 提交的任务
    private Callable<V> callable;
    // 任务运行的结果
    private Object outcome;
    // 执行任务的线程
    private volatile Thread runner;
    // 等待结果的队列（单链表）
    private volatile WaitNode waiters;
```

![FutureTask类的7种状态](https://raw.githubusercontent.com/MuggleLee/PicGo/master/FutureTask/FutureTask%E7%8A%B6%E6%80%81.png)

除了记录 FutureTask 对象状态之外，还声明了 state、runner、waiters的内存偏移量:
```java
    private static final sun.misc.Unsafe UNSAFE;
    // FutureTask 对象状态在内存中的偏移量
    private static final long stateOffset;
    // 执行任务对象在内存中的偏移量
    private static final long runnerOffset;
    // 等待链表在内存中的偏移量
    private static final long waitersOffset;

    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            // 通过反射获取各对象在内存中的偏移量
            stateOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("state"));
            runnerOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("runner"));
            waitersOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```

当程序执行到线程池的execute(Runnable runnable)方法的时候，由于 execute() 方法接收的参数是 FutureTask 对象，所以肯定是执行 FutureTask 类的 run() 方法。

### FutureTask.run()方法剖析
```java
    public void run() {
        // 如果当前 FutureTask 对象的状态不是 NEW 或者执行 CAS 操作赋值给 runnerOffset 失败直接跳出 run 方法
        if (state != NEW ||
                !UNSAFE.compareAndSwapObject(this, runnerOffset,
                        null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // 设置 runner 为 null ，利于 GC
            runner = null;
            int s = state;
            // 如果有其它线程在中断任务，会调用 handlePossibleCancellationInterrupt 方法处理
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
    // 执行任务正常结束后调用此方法
    protected void set(V v) {
        // 执行 CAS 方法设置 FutureTask 对象的状态由 NEW -> COMPLETING -> NORMAL
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            // 将任务执行的结果赋值给 outcome 变量
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL);
            // 唤醒等待线程
            finishCompletion();
        }
    }
    // 执行任务异常会调用此方法
    protected void setException(Throwable t) {
        // 执行 CAS 方法设置 FutureTask 对象的状态由 NEW -> COMPLETING -> EXCEPTIONAL
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            // 将任务执行的结果赋值给 outcome 变量
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL);
            // 唤醒等待线程
            finishCompletion();
        }
    }
    /**
     * 唤醒等待线程
     */
    private void finishCompletion() {
        for (WaitNode q; (q = waiters) != null; ) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                // 自旋遍历单链表
                for (; ; ) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        // 唤醒线程
                        LockSupport.unpark(t);
                    }
                    // 获取下一个节点，直到节点为null
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null;
                    q = next;
                }
                break;
            }
        }
        // 这是一个空方法，可以让开发中扩展使用
        done();
        callable = null;
    }
    private void handlePossibleCancellationInterrupt(int s) {
        // 双重判断，这里困扰我很久。为什么需要两次判断状态是否为 INTERRUPTING 呢？
        // 考虑的场景是：需要确保其它线程执行 cencel(true) 是在执行 run() 或者 runAndReset()的过程中
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield();// 通过自旋，优先让其它线程执行，等待 cancel(true) 执行完成
    }
```

核心代码都加上了注释，结合源码，画了张流程图加深理解吧！

![run 方法流程图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/FutureTask/FutureTask.run%20%E6%96%B9%E6%B3%95%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

### FutureTask.get()方法剖析
```java
    /**
     * 返回执行结果
     */
    public V get() throws InterruptedException, ExecutionException {
        // 获取当前 FutureTask 对象的状态
        int s = state;
        // 如果任务的状态是新建(NEW)或者运行中()就执行 awaitDone 方法等待获取
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        // 返回执行结果
        return report(s);
    }

    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V) x;
        if (s >= CANCELLED)
            throw new CancellationException();// 被取消或者中断，直接抛出异常
        throw new ExecutionException((Throwable) x);// 运行过程中发生异常
    }

    private int awaitDone(boolean timed, long nanos)
            throws InterruptedException {
        // 如果设置超时，计算截止时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        // 代表当前等待结果线程的等待节点
        WaitNode q = null;
        // 记录是否把当前线程加入到了队列
        boolean queued = false;
        for (; ; ) {
            if (Thread.interrupted()) {
                removeWaiter(q);// 如果被中断了，删除当前线程节点，并抛出异常
                throw new InterruptedException();
            }
            int s = state;
            // 如果 state 状态大于 COMPLETING 则说明任务已经执行，直接返回状态值
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            } else if (s == COMPLETING) // 如果 state 状态等于 COMPLETING，说明正在设置结果，则放弃时间片轮询等待
                Thread.yield();
            else if (q == null) // 任务状态为 NEW ，构造等待节点
                q = new WaitNode();
            else if (!queued) // 状态为 NEW ，并且节点不为 null ，并且该节点没有加入到 waiter 队列中
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                        q.next = waiters, q);
            else if (timed) { // 如果设定超时，进行超时判断
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
	        // 阻塞到超时时间
                LockSupport.parkNanos(this, nanos);
            } else // 如果没有设置超时，会一直阻塞，直到被中断或者被唤醒
                LockSupport.park(this);
        }
    }    
    // 通过自旋链表删除指定节点
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            // 设置节点为空，通过自旋找出空节点并删除
            node.thread = null;
            // 自旋保证删除成功
            retry:
            for (; ; ) {
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)// 如果当前节点不是空，不需要删除
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null)
                            continue retry;
                    } else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                            q, s))
                        continue retry;
                }
                break;
            }
        }
    }
```



# 总结

参考资料: