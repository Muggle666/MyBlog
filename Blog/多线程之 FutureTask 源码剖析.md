系统通过多线程优化性能，实际上就是将串行操作转换为并行操作，也就是说将同步操作转换为异步操作。在众多并发类中，FutureTask 类可以接受线程返回的结果，并且可以取消或者中断线程。

先看下 FutureTask 类的类图结构：
![FutureTask类图](https://raw.githubusercontent.com/MuggleLee/PicGo/master/FutureTask/FutureTask%20%E7%B1%BB%E5%9B%BE%E7%BB%93%E6%9E%84.jpg)

由类图可以知道，FutureTask 类是 Runnable 的实现类，所以可以通过线程池 submit() 或者直接 new Thread() 启动线程。

举个栗子：

```java

```

# 总结

参考资料: