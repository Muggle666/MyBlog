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




# 总结

参考资料: