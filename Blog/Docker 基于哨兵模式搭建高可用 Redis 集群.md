最近在学习Redis中间件，了解Redis为什么如此高性能高可用。强烈推荐一本书《Redis设计与实现》，这对于我学习Redis带来很对帮助和启迪！

Redis为什么如此高可用呢？这一篇文章就来介绍基于Docker容器搭建Redis集群，并使用哨兵(Sentinel)机制保证Redis集群的高可用。


## 哨兵(Sentinel)机制

哨兵机制就是开启一个或多个进程作为哨兵（Sentinel），监控着所有的主从服务器，定时的ping服务器。当主服务器宕机或者网络中断，Sentinel实例无法ping成功，Sentinel实例会**选举**一个从服务器作为主服务器；如果宕机后的服务器重新上线或者原先的主服务器网络恢复，会作为新的主服务器的从服务器。很绕口是吧？看下面的图就清楚了。

图片...

使用Sentinel的好处有以下几点：
> 监控（Monitoring）： Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。
提醒（Notification）： 当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。
自动故障迁移（Automatic failover）：当一个主服务器不能正常工作时，Sentinel 会开始一次自动故障迁移操作，它会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效主服务器的其他从服务器改为复制新的主服务器；当客户端试图连接失效的主服务器时，集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

### 使用Docker搭建Redis集群


##### 1.首先使用docker下载最新版本的Redis
>docker pull redis


##### 2.启动一主二从（一个master，两个slave），为了启动方便，都省略了密码
>docker run -p 8080:8080 --name master -d -v /root/redis/master/data:/data -v /root/redis/master/conf/redis.conf:/etc/redis/redis.conf redis --port 8080 --appendonly yes
docker run -p 8081:8081 --name slave1 -d -v /root/redis/slave1/data:/data -v /root/redis/slave1/conf/redis.conf:/etc/redis/redis.conf redis --port 8081 --appendonly yes --slave-read-only yes --slaveof 主服务器ip 8080
docker run -p 8082:8082 --name slave2 -d -v /root/redis/slave2/data:/data -v /root/redis/slave2/conf/redis.conf:/etc/redis/redis.conf redis --port 8082 --appendonly yes --slave-read-only yes --slaveof 主服务器ip 8080

图片...

简单解释docker命令吧。
>-p：Redis 服务端口映射到外网可访问的端口（譬如 slave1 服务设置的端口是8081，所以设置对外服务的端口命令为 对外的端口:8081）
-d：后台运行
-v：挂载宿主机的目录
--port：Redis 的启动命令，设置 Redis 服务的端口
--appendonly yes：开启 Redis 持久化方式为 AOF

我在学习Sentinel集群时候的一个误区，我看到很多博客，文章提到启动Redis设置的对外端口命令为 **对外端口:6379**，这是在没有使用Redis的--port命令的情况下，默认使用Redis服务的6379端口。一旦手动设置 Redis 服务端口，在 Docker 容器设置端口映射的时候要注意设置为你改后的端口！

3.下载redis.conf文件，用来配置Sentinel服务
>mkdir /redis
cd /redis
wget https://raw.githubusercontent.com/antirez/redis/5.0/redis.conf -O redis.conf

4.配置redis.conf文件：执行vim redis.conf后在最下面添加以下配置
>sentinel monitor 主机别名 主机ip 主机端口 2
sentinel down-after-milliseconds 主机别名 30000
sentinel parallel-syncs 主机别名 1
sentinel failover-timeout 主机别名 180000

**sentinel monitor**：指当前监控主节点，最后的“2”代表当主服务器宕机后至少需要两个sentinel服务投票同意
**sentinel down-after-milliseconds**：如果当前主服务器超过设定的时间没有回复，则判断主服务器不可用
**sentinel parallel-syncs**：限制从服务器向新的主服务器发起复制的个数。若发起复制的从服务器过多，那么可能会造成主服务器阻塞。若发起复制的从服务器过少，可能会造成数据在复制期间不一致的情况。
**sentinel failover-timeout**：故障转移的时间

**除了上面的配置之外，redis.conf还有几项配置需要更改！**
###### ps.一定要知道为什么要更改这些配置，否则参照别人的配置胡乱更改，启动后有可能出现各种情况！

4.1.注释bind配置
打开redis.conf文件，查找到一项配置为 ***bind 127.0.0.1*** ，这个配置说明只可以在本机访问Redis服务；注释后可以接收任意ip的连接，这样的话，可以接收到监控的节点回复。
> #bind 127.0.0.1

4.2.修改protected-mode配置
默认的protected-mode为yes,将yes设置为no，此时外部网络可以直接访问
>protected-mode no

4.3.修改daemonize配置
默认daemonize为no，将daemonize注释可以让Redis服务后台运行。
>#daemonize no

5.使用 -v 命令将Sentinel容器配置文件挂载到自定义的redis.conf文件，启动Sentinel服务后就可以监控Redis的服务啦！（注意redis.conf的权限，权限不够的话会报Permission denied）
>docker run --name sentinel -p 9001:6379 -v /redis/redis.conf:/etc/redis/redis.conf -v /root/redis/sentinel/data:/data -d redis redis-server /etc/redis/redis.conf --appendonly yes --sentinel

查看sentinel容器日志，通过日志可以知道sentinel启动后监控了主从服务器。

图片...

再次打开挂载的redis.conf文件，可以发现配置文件的底部记录了主从服务器的信息，也就是说成功的监控各个服务器！
图片...

进入sentinel容器执行sentinel masters可以看到监控的主服务器信息；执行sentinel slaves  
 your master name 可以看到从服务器的信息。
>docker exec -it sentinel redis-cli
sentinel masters
sentinel slaves master #譬如我的主服务器的名称是master

图片...


模拟主服务器故障，从服务器成为主服务器的操作。
>docker stop master

关闭master容器，模拟主服务器宕机，再次进入sentinel容器执行sentinel masters查看主服务器的信息，经过设定的故障转移时间，你会发现slave1或者slave2成为了主服务器。

再次启动master容器，进入sentinel容器执行sentinel masters查看主服务器的信息可以发现

参考资料
[https://www.cnblogs.com/kevingrace/p/9004460.html](https://www.cnblogs.com/kevingrace/p/9004460.html)
