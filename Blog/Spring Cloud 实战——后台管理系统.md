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
Spring Cloud (Finchley)、JDK8、Maven、Mysql 5.6、Redis5 、Rabbitmq、Docker

├─CommonModel
│  ├─src
│  │  ├─main
│  │  │  └─java
│  │  │      └─com
│  │  │          └─hao
│  │  │              └─commonmodel
│  │  │                  ├─common
│  │  │                  ├─log
│  │  │                  │  └─constants
│  │  │                  ├─Mail
│  │  │                  │  └─constants
│  │  │                  ├─oauth
│  │  │                  └─user
│  │  │                      └─constants
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─commonmodel
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─commonmodel
│      │              ├─common
│      │              ├─log
│      │              │  └─constants
│      │              ├─mail
│      │              │  └─constants
│      │              ├─oauth
│      │              └─user
│      │                  └─constants
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      └─maven-status
│          └─maven-compiler-plugin
│              └─compile
│                  └─default-compile
├─CommonUnits
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─commonunits
│  │  │  │              ├─constant
│  │  │  │              └─utils
│  │  │  └─resources
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─commonunits
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─commonunits
│      │              ├─constant
│      │              └─utils
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      └─maven-status
│          └─maven-compiler-plugin
│              └─compile
│                  └─default-compile
├─ConfigCenter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─configcenter
│  │  │  └─resources
│  │  │      └─configs
│  │  │          └─dev
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─configcenter
│  └─target
│      ├─classes
│      │  ├─com
│      │  │  └─hao
│      │  │      └─configcenter
│      │  └─configs
│      │      └─dev
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
├─FileCenter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─filecenter
│  │  │  │              ├─config
│  │  │  │              ├─controller
│  │  │  │              ├─mapper
│  │  │  │              ├─model
│  │  │  │              ├─service
│  │  │  │              └─utils
│  │  │  └─resources
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─filecenter
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─filecenter
│      │              ├─config
│      │              ├─controller
│      │              ├─mapper
│      │              ├─model
│      │              ├─service
│      │              └─utils
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
├─GatewayCenter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─gatewaycenter
│  │  │  │              ├─config
│  │  │  │              ├─controller
│  │  │  │              ├─feign
│  │  │  │              └─filter
│  │  │  └─resources
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─gateway
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─gatewaycenter
│      │              ├─config
│      │              ├─controller
│      │              ├─feign
│      │              └─filter
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
├─LogCenter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─logcenter
│  │  │  │              ├─config
│  │  │  │              ├─consumer
│  │  │  │              ├─controller
│  │  │  │              ├─mapper
│  │  │  │              └─service
│  │  │  └─resources
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─logcenter
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─logcenter
│      │              ├─config
│      │              ├─consumer
│      │              ├─controller
│      │              ├─mapper
│      │              └─service
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
├─LogStarter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─logstarter
│  │  │  │              └─autoconfigure
│  │  │  └─resources
│  │  │      └─META-INF
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─logstarter
│  └─target
│      ├─classes
│      │  ├─com
│      │  │  └─hao
│      │  │      └─logstarter
│      │  │          └─autoconfigure
│      │  └─META-INF
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      └─maven-status
│          └─maven-compiler-plugin
│              └─compile
│                  └─default-compile
├─ManageBackend
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─managebackend
│  │  │  │              ├─config
│  │  │  │              ├─consumer
│  │  │  │              ├─controller
│  │  │  │              ├─mapper
│  │  │  │              ├─model
│  │  │  │              └─service
│  │  │  └─resources
│  │  │      └─static
│  │  │          ├─css
│  │  │          │  ├─font-awesome
│  │  │          │  │  ├─css
│  │  │          │  │  └─fonts
│  │  │          │  ├─treetable
│  │  │          │  └─ztree
│  │  │          │      └─zTreeStyle
│  │  │          │          └─img
│  │  │          │              └─diy
│  │  │          ├─fonts
│  │  │          ├─img
│  │  │          │  ├─avatars
│  │  │          │  ├─login
│  │  │          │  └─logo
│  │  │          ├─js
│  │  │          │  ├─bootstrap
│  │  │          │  ├─libs
│  │  │          │  ├─my
│  │  │          │  └─plugin
│  │  │          │      ├─bootstraptree
│  │  │          │      ├─bootstrapvalidator
│  │  │          │      ├─datatable-responsive
│  │  │          │      ├─datatables
│  │  │          │      │  └─swf
│  │  │          │      ├─jquery-nestable
│  │  │          │      └─jquery-validate
│  │  │          ├─layui
│  │  │          │  ├─css
│  │  │          │  │  └─modules
│  │  │          │  │      ├─laydate
│  │  │          │  │      │  └─default
│  │  │          │  │      └─layer
│  │  │          │  │          └─default
│  │  │          │  ├─font
│  │  │          │  ├─images
│  │  │          │  │  └─face
│  │  │          │  └─lay
│  │  │          │      └─modules
│  │  │          └─pages
│  │  │              ├─blackIP
│  │  │              ├─client
│  │  │              ├─file
│  │  │              ├─log
│  │  │              ├─mail
│  │  │              ├─menu
│  │  │              ├─permission
│  │  │              ├─role
│  │  │              ├─sms
│  │  │              ├─swagger
│  │  │              ├─user
│  │  │              └─wechat
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─managebackend
│  └─target
│      ├─classes
│      │  ├─com
│      │  │  └─hao
│      │  │      └─managebackend
│      │  │          ├─config
│      │  │          ├─consumer
│      │  │          ├─controller
│      │  │          ├─mapper
│      │  │          ├─model
│      │  │          └─service
│      │  └─static
│      │      ├─css
│      │      │  ├─font-awesome
│      │      │  │  ├─css
│      │      │  │  └─fonts
│      │      │  ├─treetable
│      │      │  └─ztree
│      │      │      └─zTreeStyle
│      │      │          └─img
│      │      │              └─diy
│      │      ├─fonts
│      │      ├─img
│      │      │  ├─avatars
│      │      │  ├─login
│      │      │  └─logo
│      │      ├─js
│      │      │  ├─bootstrap
│      │      │  ├─libs
│      │      │  ├─my
│      │      │  └─plugin
│      │      │      ├─bootstraptree
│      │      │      ├─bootstrapvalidator
│      │      │      ├─datatable-responsive
│      │      │      ├─datatables
│      │      │      │  └─swf
│      │      │      ├─jquery-nestable
│      │      │      └─jquery-validate
│      │      ├─layui
│      │      │  ├─css
│      │      │  │  └─modules
│      │      │  │      ├─laydate
│      │      │  │      │  └─default
│      │      │  │      └─layer
│      │      │  │          └─default
│      │      │  ├─font
│      │      │  ├─images
│      │      │  │  └─face
│      │      │  └─lay
│      │      │      └─modules
│      │      └─pages
│      │          ├─blackIP
│      │          ├─client
│      │          ├─file
│      │          ├─log
│      │          ├─mail
│      │          ├─menu
│      │          ├─permission
│      │          ├─role
│      │          ├─sms
│      │          ├─swagger
│      │          ├─user
│      │          └─wechat
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
├─MonitorCenter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─monitorcenter
│  │  │  └─resources
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─monitorcenter
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─monitorcenter
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
├─NotificationCenter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─notificationcenter
│  │  │  │              ├─config
│  │  │  │              ├─controller
│  │  │  │              ├─mapper
│  │  │  │              ├─model
│  │  │  │              ├─service
│  │  │  │              └─utils
│  │  │  └─resources
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─notificationcenter
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─notificationcenter
│      │              ├─config
│      │              ├─controller
│      │              ├─mapper
│      │              ├─model
│      │              ├─service
│      │              └─utils
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
├─OauthCenter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─oauthcenter
│  │  │  │              ├─config
│  │  │  │              ├─controller
│  │  │  │              ├─feigin
│  │  │  │              └─service
│  │  │  └─resources
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─oauthcenter
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─oauthcenter
│      │              ├─config
│      │              ├─controller
│      │              ├─feigin
│      │              └─service
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
├─RegisterCenter
│  ├─src
│  │  ├─main
│  │  │  ├─java
│  │  │  │  └─com
│  │  │  │      └─hao
│  │  │  │          └─registercenter
│  │  │  └─resources
│  │  └─test
│  │      └─java
│  │          └─com
│  │              └─hao
│  │                  └─registercenter
│  └─target
│      ├─classes
│      │  └─com
│      │      └─hao
│      │          └─registercenter
│      ├─docker
│      ├─generated-sources
│      │  └─annotations
│      ├─maven-archiver
│      ├─maven-status
│      │  └─maven-compiler-plugin
│      │      └─compile
│      │          └─default-compile
│      └─test-classes
└─UserCenter
    ├─src
    │  ├─main
    │  │  ├─java
    │  │  │  └─com
    │  │  │      └─hao
    │  │  │          └─usercenter
    │  │  │              ├─config
    │  │  │              ├─controller
    │  │  │              ├─feign
    │  │  │              ├─mapper
    │  │  │              └─service
    │  │  └─resources
    │  └─test
    │      └─java
    │          └─com
    │              └─hao
    │                  └─usercenter
    └─target
        ├─classes
        │  └─com
        │      └─hao
        │          └─usercenter
        │              ├─config
        │              ├─controller
        │              ├─feign
        │              ├─mapper
        │              └─service
        ├─docker
        ├─generated-sources
        │  └─annotations
        ├─maven-archiver
        ├─maven-status
        │  └─maven-compiler-plugin
        │      └─compile
        │          └─default-compile
        └─test-classes

