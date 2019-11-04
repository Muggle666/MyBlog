当我看完《深入了解 Spring Cloud 与微服务构建》的时候，我就打算将所学习到的 Spring Cloud 组件使用到项目中做成一个小项目，加深理解。

该项目是在个人 PC 运行，不一定长期启动...（出租房电费太贵了）。**正常运行时间为：8:00 ~ 22:00**
项目Demo：[http://mugglelee.nat300.top/api-b/login.html](http://mugglelee.nat300.top/api-b/login.html)
账号：Tourist  
密码：123456  
（此账号设置了权限，不能对显示的信息删除或者修改，如有疑问，可在评论区与我一起探讨）

## 项目简介：

#### 登录模块：
- 1.账号登录
- 2.微信授权登录（由于是个体测试者，只能通过关注我个人的测试公众号才能微信授权，如有需要，可通过微信关注以下公众号）
- 3.手机号码登录：使用阿里云短信服务。通过帐号登录后可在页面右上方绑定手机号码，以后登录的话可以直接通过 **手机号码+验证码** 登录

#### 管理后台模块：
- 1.菜单管理：菜单列表、添加菜单、修改菜单、删除菜单、菜单图标设置
- 2.角色管理：角色查询、添加角色、修改角色、删除角色、分配菜单、分配权限
- 3.权限管理：权限查询、添加权限、修改权限、删除权限
- 4.用户管理：用户查询、添加用户、修改用户、给用户分配角色、重置密码
- 5.文件管理：上传文件、文件列表、文件删除
- 6.邮件管理：发送邮件、查询邮件
- 7.注册中心：查看微服务注册情况
- 8.监控中心：监控微服务信息
- 9.接口swagger文档
- 10.ip黑名单：查询、添加、删除ip黑名单
- 11.日志查询
- 12.个人信息修改
- 13.修改密码
- 14.头像修改


### 开发环境：
系统：Windows10 —— 项目的子项目和运行环境都通过启动 Docker 镜像运行。详情可查看父项目的 docker-compose.yml 配置

### 项目结构：

├─CommonModel：基础Model
├─CommonUnits：工具包
├─ConfigCenter：配置中心
├─FileCenter：文件上传中心
├─GatewayCenter：网关中心
├─LogCenter：日志中心
├─LogStarter：日志配置
├─ManageBackend：门户中心
├─MonitorCenter：监控中心
├─NotificationCenter：通知中心
├─OauthCenter：权限中心
├─RegisterCenter：注册中心
├─UserCenter：用户中心
├─buildImage：将所有子项目都打包后构建成镜像
├─docker-compose.yml：构建容器
├─Starting Sequence.md：查看子项目的启动顺序

### 技术实现：
Spring Cloud (Finchley)、Spring Security、JDK8、Maven、Mysql 5.6、Mybatis-Plus、Redis5、Rabbitmq、Docker

## **技术选型：**

搭建框架：Spring Boot + Spring Cloud 2.0（Finchley版本）
好处：由于Spring Boot 默认大于配置的原则，可以快速搭建 Spring Cloud 微服务。

##### 注册中心：Eureka 
常用的注册中心有：Eureka，Zookeeper。那为什么选择使用 Eureka ？（没有实际项目支持说法，如有理解错误望帮助我改正...）
我觉得有以下两点：
```
1.Eureka是Spring Cloud首选推荐的服务注册和发现组件，可以和其它组件无缝对接；
2.分布式系统CAP（Consistency，Availability，Partition tolerance）理论指出三种特性只能选其二，而P（分区容错性）是分布式系统必须保证，因此注册中心就分为AP和CP。
Zookeeper属于CP，所以对于Zookeeper来说，一致性高于可用性，优点是可以保证当前的节点信息是最新的，但当master节点故障会导致重新选举，而选举过程太长可能会导致整个zk集群不可用，导致瘫痪；
Eureka属于AP，使用Eureka则不会因为几个节点挂掉而影响其他节点，但缺点也很明显，Eureka不保证强一致性，所以可能其他节点查询的信息并不是最新的。
```
ps.常见的服务注册组件还有Consul，其属于
当前项目我并没有部署集群，也不必要保证当前的节点信息是最新的，另外为了能够与其他Spring Cloud组件无缝对接，我选择使用Eureka。

##### 配置中心：Config
##### 网关中心：

##### 断路器：
##### 消息队列：
##### 数据库：
##### 持久层框架：



Spring Boot 结合 Spring Cloud 各个组件能够快速的搭建微服务项目。使用 Spring Boot 2.0（Finchley版本），项目运用到 Eureka 作为注册中心，Config 作为配置中心，Gateway 作为网关中心，