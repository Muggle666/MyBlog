import java.util.concurrent.*;
import java.util.concurrent.locks.LockSupport;

public class FutureTask<V> implements RunnableFuture<V> {

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

    /**
     * 返回执行结果
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V) x;
        if (s >= CANCELLED)
            throw new CancellationException();// 被取消或者中断，直接抛出异常
        throw new ExecutionException((Throwable) x);// 运行过程中发生异常
    }

    /**
     * 初始化FutureTask对象
     */
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;
    }

    /**
     * 初始化FutureTask对象
     */
    public FutureTask(Runnable runnable, V result) {
        // 调用 callable 方法将 Runnable 对象转换为 Callable 对象
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;
    }

    // 根据状态码判断任务是否被取消
    public boolean isCancelled() {
        return state >= CANCELLED;
    }

    // 根据状态码判断任务是否执行完成
    public boolean isDone() {
        return state != NEW;
    }

    /**
     * 任务取消
     * @param mayInterruptIfRunning true：代表中断线程 false：代表取消线程
     * @return
     */
    public boolean cancel(boolean mayInterruptIfRunning) {
        // 如果任务不在 NEW 状态或者执行 UNSAFE 操作失败直接返回 false
        if (!(state == NEW &&
                UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                        mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally {
                    // 设置最终的状态为中断状态
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            // 释放等待线程
            finishCompletion();
        }
        return true;
    }

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

    /**
     * 与上面get()相似，加上 timeout 设置过期时间，超时抛出异常
     *
     * @param timeout 过期时间
     * @param unit    时间单位
     * @return 返回执行结果
     */
    public V get(long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
                (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }

    protected void done() {
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
                    // 执行成功返回结果
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {// 运行异常执行setException()方法返回异常结果
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

    /**
     * 与 run() 方法类似，但该方法可以执行多次。
     * 不同点：1.不设置返回值 2.不设置 state 值（执行完任务后，FutureTask 对象状态还是 NEW ）
     */
    protected boolean runAndReset() {
        if (state != NEW ||
                !UNSAFE.compareAndSwapObject(this, runnerOffset,
                        null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    // 不设置返回值
                    c.call();
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            runner = null;
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        // 任务执行成功，且状态重置为NEW
        return ran && s == NEW;
    }

    private void handlePossibleCancellationInterrupt(int s) {
        // 双重判断，这里困扰我很久。为什么需要两次判断状态是否为 INTERRUPTING 呢？
        // 考虑的场景是：需要确保其它线程执行 cencel(true) 是在执行 run() 或者 runAndReset()的过程中
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield();// 通过自旋，优先让其它线程执行，等待 cancel(true) 执行完成
    }

    /**
     * 单链表。存储等待线程
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;

        WaitNode() {
            thread = Thread.currentThread();
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

    // Unsafe mechanics
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

}

