最近在学习Redis中间件，了解Redis为什么如此高性能高可用。强烈推荐一本书《Redis设计与实现》，这对于我学习Redis带来很对帮助和启迪！

上一篇文章描述了[使用 Docker 基于哨兵模式搭建高可用的Redis](https://www.jianshu.com/p/91f87c2a2c61)，这一篇文章则使用Docker搭建Redis集群。

## Redis Cluster 简介

Redis 3.0提供的分布式数据库解决方案——Redis Cluster。不仅可以主从复制，还可以像Sentinel那样有故障转移机制。除此之外，集群还有几个独有的特点：去中心化和数据分片。

>去中心化：集群没有中心（核心）节点，就不会受到太大的影响，当某个主节点故障而不会影响整个集群的可用性。
分片：集群中的主节点可以水平扩展，使用哈希槽将键值对分配到不同的主节点，分担了单机Redis的压力。

接下来主要介绍如何使用 Docker 搭建Redis Cluster

#### 1.创建集群目录，存放各个节点的配置目录（/conf/redis.conf和数据目录（/data）

>for port in `seq 8081 8089`; do mkdir -p ./${port}/conf && PORT=${port} envsubst < ./redis-cluster.tmpl > ./${port}/conf/redis.conf  && mkdir -p ./${port}/data; done

#### 2.运行端口为8081~8089的容器

>for port in `seq 8081 8089`; do docker run -d -ti -p ${port}:${port} -p 1${port}:1${port} -v /cluster-docker/${port}/conf/redis.conf:/etc/redis/redis.conf -v /cluster-docker/${port}/data:/data --restart always --name redis-${port} --net redis-net --sysctl net.core.somaxconn=1024 redis:5.0 redis-server /etc/redis/redis.conf; done

![docker ps ](https://raw.githubusercontent.com/MuggleLee/PicGo/master/Redis%E5%9B%BE/%E9%9B%86%E7%BE%A4/docker%20ps.jpg)

#### 3.获取各个容器的内外ip

>for port in `seq 8081 8089`; do echo -n "$(docker inspect --format '{{ (index .NetworkSettings.Networks "bridge").IPAddress }}' "redis-${port}")":${port} ' ' ; done

复制打印出来的ip和端口




#### 4.创建集群

随便进入一个redis容器，譬如
>docker exec -it redis-8081 /bin/bash

再执行创建集群的命令
> redis-cli --cluster create 172.17.0.2:8081  172.17.0.3:8082  172.17.0.4:8083  172.17.0.5:8084  172.17.0.6:8085  172.17.0.7:8086  172.17.0.8:8087  172.17.0.9:8088  172.17.0.10:8089 --cluster-replicas 2


显示 Can I set the above configuration? (type 'yes' to accept): 的时候，输入yes

如无意外的话，最后显示 [OK] All 16384 slots covered. 则代表集群创建成功！


模拟主节点故障



