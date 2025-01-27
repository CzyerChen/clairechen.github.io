---
layout:     post
title:      OpenSIPS-由浅入深编译更多可选模块

subtitle:   OpenSIPS 
date:       2025-01-27
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - OpenSIPS
    - sip
---

接上篇[OpenSIPS-从安装部署开始认识一个组件](https://zy5999.cn/2025/01/27/OpensipsWorld-01%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2/)，是采取全默认的方式体验OpenSIPS，那么我需要额外的模块怎么办呢？可选的有哪些？

流程在第一篇文章中已经介绍了，这边主要是操作一下接入一个db_mysql的组件是如何操作的。

- [开启配置](#开启配置)
- [选择模块](#选择模块)
- [安装基础依赖](#安装基础依赖)
- [再次执行编译，加入额外模块](#再次执行编译加入额外模块)
- [选一个模块使用db\_mysql](#选一个模块使用db_mysql)

## 开启配置

```bash
make menuconfig
```

进入编译选项：

```bash
 --->  Configure Compile Options  
```

## 选择模块

选择db_mysql模块

```bash
 --->  Configure Excluded Modules
```

完成的可选模块列表如下：

- aaa_diameter
- aaa_radius
- auth_jwt
- cachedb_cassandra
- cachedb_couchbase
- cachedb_memcached
- cachedb_mongodb
- cachedb_redis
- carrierroute
- cgrates
- compression
- cpl_c
- db_berkeley
- db_http
- db_mysql
- db_oracle
- db_perlvdb
- db_postgres
- db_sqlite
- db_unixodbc
- dialplan
- emergency
- event_rabbitmq
- event_kafka
- h350
- regex
- identity
- jabber
- json
- ldap
- lua
- httpd
- mi_xmlrpc_ng
- mmgeoip
- osp
- perl
- pi_http
- rabbitmq
- rabbitmq_consumer
- proto_sctp
- proto_tls
- proto_wss
- presence
- presence_dialoginfo
- presence_mwi
- presence_xml
- presence_dfks
- pua
- pua_bla
- pua_dialoginfo
- pua_mi
- pua_usrloc
- pua_xmpp
- python
- rest_client
- rls
- sngtc
- siprec
- snmpstats
- tls_openssl
- tls_mgm
- tls_wolfssl
- xcap
- xcap_client
- xml
- xmpp
- uuid

根据自己的需求选择需要额外编译的模块，这边为了测试db_mysql数据库模块，因此将`db_mysql`选中

左键返回到列表，注意进行保存 ` --->  Save Changes `

此时你将获得如下提示

```bash
 You have enabled the 'db_mysql' module, so please install ' development libraries of mysql-client , typically libmysqlclient-dev'  
 Press any key to continue  
```

官方温馨提示到，引入的模块需要具备的基础依赖包，需要你自行确认现有服务器是否具备，否则需要你先具备后，再进行编译。

## 安装基础依赖

因此我们退出可视化界面，先安装 `mysql-client`，特别是 ` libmysqlclient-dev`

在目前CentOS7的机器上，非常容易实现这一点，官方的提示可能是基于 ubuntu系统的，那么安装libmysqlclient-dev，实际上需要安装的是mysql-devel包，mysql-devel包含了开发MySQL客户端所需的库和头文件，等同于libmysqlclient-dev。

1. 首先确保你的系统软件包是最新的：`sudo yum update -y`
2. 安装mysql-devel:  `sudo yum install mysql-devel -y`

以上安装完成后，就可以继续开启编译了

## 再次执行编译，加入额外模块

```bash
make menuconfig

-> Compile And Install OpenSIPS 
```

然后依旧是快速地编译，快速地结束，我们看一下modules模块下，对应的内容编译完成了吗？

```bash
# find / -name db_mysql.so
/usr/local/lib64/opensips/modules/db_mysql.so
```

## 选一个模块使用db_mysql

接下来我们就是需要测试，这个模块是否与我们的DB能够交互，版本兼容性上面是否存在异常

官方对于一些核心模块使用数据库，在安装菜单内还有两个专门的菜单，描述如何进行数据库表结构的[初始化](https://www.opensips.org/Documentation/Install-DBDeployment-3-5)，以及[数据库表结构](https://www.opensips.org/Documentation/Install-DBSchema-3-5)

我们可以选择使用官方脚本，进行数据库表的初始化。也可以从数据库结构中，配置需要接入数据库配置的部分表。

要创建您在上面预置的 database_name 数据库，请运行

```bash
opensips-cli -x database create
```

如果您决定添加新模块，例如 presence，只需调用：

```bash
opensips-cli -x database add presence
```

您还可以使用以下方法为数据库指定不同的名称，例如 opensips_test：

opensips-cli -x 数据库创建opensips_test

```bash
opensips-cli -x database create opensips_test
```

比如我们需要usrloc的模块，引入MySQL数据库存储配置

```config
modparam("usrloc", "db_url", "dbdriver://username:password@dbhost/dbname")
```

配置后启动服务，usrloc数据就从db进行初始化，DB中修改后，就通过reload http接口进行动态生效。
