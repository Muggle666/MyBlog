聊Redis线程模型之前，先思考一个问题：Redis是单线程的，但为什么可以支持高并发？


这问题的答案我认为可以从以下几个点回答：
1.纯内存操作——内存操作肯定比普通的操作快呀！
2.非阻塞IO多路复用程序——Redis核心
3.单线程避免上下文切换带来的CPU消耗

第二点，Redis核心是非阻塞IO多路复用程序是什么东东？

那就要引入这篇文章的主角————Redis线程模型

Redis线程模型实际上是Redis基于Reactor模式开发文件事件处理器（File Event Handler）。简单来说，文件事件处理器可以监听多个客户端的socket，并根据socket产生的事件选择不同的事件处理器处理客户端连接请求、命令请求和响应请求。

文件事件处理器的组成结构：

多个socket：Redis服务器会为每个发起连接Redis的客户端创建对应的socket
IO多路复用程序：可以同时监听多个socket，当socket产生事件，IO多路复用程序会将socket塞进队列。
文件事件分派器：接收队列的socket并分派到socket事件关联的事件处理器
事件处理器：包括连接应答处理器、命令请求处理器、命令回复处理器、复制处理器。

>连接应答处理器：对连接服务器的客户端进行应答。
命令请求处理器：接收客户端发过来的请求。
命令回复处理器：向客户端返回命令的执行结果。
复制处理器：当从服务器向主服务器进行复制的操作时，主从服务器都需要关联复制处理器。

通过上面的简单介绍文件事件处理器，再回顾“Redis是单线程的，但为什么可以支持高并发？”的答案，相信你会有不一样的感受！

Redis核心——文件事件处理器可以接收多个客户端的发过来的请求信息，请求信息会产生相应的事件（譬如：AE_READABLE、AE_WRITEABLE），然后非阻塞IO多路复用程序监听到socket产生的事件就会把socket塞进队列，再通过文件事件分派器选择事件对应的处理器处理。

什么？！还是不太明白Redis为什么效率那么高？文件事件处理器的各个组成也不太清楚有什么联系？接下来我通过画图，一步一图的方式尽可能的理清Redis线程模型的样貌。

### 模拟客户端和Redis服务器的通信过程

1.在Redis服务器初始化的时候，

![Redis线程模型-服务器初始化的时候](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Redis%E5%9B%BE/%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B/Redis%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B-%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%88%9D%E5%A7%8B%E5%8C%96%E7%9A%84%E6%97%B6%E5%80%99.png)

2.当客户端发起请求

![Redis线程模型-客户端发起请求](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Redis%E5%9B%BE/%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B/Redis%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B-%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%8F%91%E8%B5%B7%E8%AF%B7%E6%B1%82.png)

3.当客户端接收响应

![Redis线程模型-客户端接收响应](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Redis%E5%9B%BE/%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B/Redis%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B-%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%8E%A5%E6%94%B6%E5%93%8D%E5%BA%94.png)


参考资料：

《Redis 设计与实现》

中华石杉老师的课程——《Java工程师面试突击第一季》