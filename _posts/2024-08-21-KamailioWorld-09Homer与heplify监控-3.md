---
layout:     post
title:      Kamailio-基于Homer与heplify的SIP信令监控-3

subtitle:   Kamailio Homer
date:       2024-09-12
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kamailio
    - Homer
---

接上2篇文章，你已经顺利地安装并部署了Homer相关服务，配置好了服务并顺利启动了。这个时候你已经算是搭建完成了一个SIP监控、分析体系，那应该怎么去用呢？

跟着我，你将学会：

- [如何使用 Homer 查询会话信息](#如何使用-homer-查询会话信息)
  - [登录平台](#登录平台)
  - [首页看板](#首页看板)
  - [会话详情](#会话详情)
  - [具体某条信令](#具体某条信令)
  - [自定义查询页面](#自定义查询页面)

## 如何使用 Homer 查询会话信息

### 登录平台

使用浏览器访问地址：`http://ip:9080/`，输入默认的账号密码admin/sipcapture。

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-0.png)

### 首页看板

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-1.png)

### 会话详情

点击session_id，查看会话详情。

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-2.png)

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-3.png)

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-4.png)

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-5.png)

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-6.png)

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-7.png)

### 具体某条信令

不是点击session_id了，直接列表一行双击即可展示。
或者在会话中的message和flow中点具体一条信令，也可以查看到详情。

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-8.png)

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-9.png)

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-10.png)

### 自定义查询页面

右上角 `Dashboad` 内默认会选择 `Home`，这里选择 `Smart search`就可以到自定义筛选页面。

有一些内置的字段可供自行组装查询条件：

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-12.png)

![img](https://raw.githubusercontent.com/CzyerChen/clairechen.github.io/master/img/blog/Kamailio-homer-11.png)

当然也可以研究存储到数据库的数据，依托 Homer-UI 做更多你想要的功能与统计。
