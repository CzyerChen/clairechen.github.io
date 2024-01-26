---
layout:     post
title:      ActiveMQ-Artemis服务部署
subtitle:   ActiveMQ-Artemis
date:       2024-01-25
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - ActiveMQ
    - Artemis
    - 部署
---

对于ActiveMQ的部署方式网络上有很多的文章描述，这边简单介绍一下如何搭建Artemis

官方网站也有[Start Up指导](https://activemq.apache.org/components/artemis/documentation/)

- [Linux环境下部署](#linux环境下部署)
  - [下载源码包](#下载源码包)
  - [解压、配置并启动](#解压配置并启动)
  - [创建Broker实例](#创建broker实例)
  - [启动实例](#启动实例)
  - [修改配置](#修改配置)
  - [管理后台](#管理后台)
- [Docker环境下部署](#docker环境下部署)
  - [获取镜像](#获取镜像)
  - [启动服务](#启动服务)

## Linux环境下部署

### 下载源码包

具体版本可以自行修改

```bash
 wget https://dlcdn.apache.org/activemq/activemq-artemis/2.31.2/apache-artemis-2.31.2-bin.zip
 ```

 `.tar.gz`压缩包和`.zip`的压缩包均可

### 解压、配置并启动

关于目录结构

```bash
apache-artemis-2.31.2]# tree -L 1
.
├── bin
├── examples
├── lib
├── LICENSE
├── licenses
├── NOTICE
├── README.html
├── schema
└── web
```

- bin：是可执行脚本区域
- examples：是所有特性的样例
- lib：是启动所需的依赖包
- web：是broker启动web context会执行的目录

artemis 是程序和数据目录分离的结构，并且官方并不推荐将数据目录放置在程序目录下
数据可以轻松地挂载到其他目录，在应用升级的时候，程序目录升级不影响存量数据

### 创建Broker实例

```bash
$ cd ${ARTEMIS_HOME}
$ ${ARTEMIS_HOME}/bin/artemis create jms-data
Creating ActiveMQ Artemis instance at: /data/activemq-artemis/brokerdata/jms-data

--user:
What is the default username?
admin

--password: is mandatory with this configuration:
What is the default password?


--allow-anonymous | --require-login:
Allow anonymous access?, valid values are Y, N, True, False
N

Auto tuning journal ...
done! Your system can make 2.07 writes per millisecond, your journal-buffer-timeout will be 484000

You can now start the broker by executing:  

   "/data/activemq-artemis/brokerdata/jms-data/bin/artemis" run

Or you can run the broker in the background using:

   "/data/activemq-artemis/brokerdata/jms-data/bin/artemis-service" start
```

以上实例创建完毕，对于用户名密码不指定的话默认就是artemis/artemis

不想交互式输入，也可以在创建的时候直接指定

```bash
./bin/artemis create jms-data --user admin --password password --role admin
```

其他artemis指令方法：

```bash
$ ./artemis help create
NAME
        artemis create - creates a new broker instance

SYNOPSIS
        artemis create [--addresses <addresses>] [--aio] [--allow-anonymous]
                [--autocreate] [--blocking] [--cluster-password <clusterPassword>]
                [--cluster-user <clusterUser>] [--clustered] [--data <data>]
                [--default-port <defaultPort>] [--disable-persistence]
                [--encoding <encoding>] [--etc <etc>] [--failover-on-shutdown] [--force]
                [--global-max-size <globalMaxSize>] [--home <home>] [--host <host>]
                [--http-host <httpHost>] [--http-port <httpPort>]
                [--java-options <javaOptions>] [--jdbc]
                [--jdbc-bindings-table-name <jdbcBindings>]
                [--jdbc-connection-url <jdbcURL>]
                [--jdbc-driver-class-name <jdbcClassName>]
                [--jdbc-large-message-table-name <jdbcLargeMessages>]
                [--jdbc-lock-expiration <jdbcLockExpiration>]
                [--jdbc-lock-renew-period <jdbcLockRenewPeriod>]
                [--jdbc-message-table-name <jdbcMessages>]
                [--jdbc-network-timeout <jdbcNetworkTimeout>]
                [--jdbc-node-manager-table-name <jdbcNodeManager>]
                [--jdbc-page-store-table-name <jdbcPageStore>]
                [--journal-device-block-size <journalDeviceBlockSize>] [--mapped]
                [--max-hops <maxHops>] [--message-load-balancing <messageLoadBalancing>]
                [--name <name>] [--nio] [--no-amqp-acceptor] [--no-autocreate]
                [--no-autotune] [--no-fsync] [--no-hornetq-acceptor]
                [--no-mqtt-acceptor] [--no-stomp-acceptor] [--no-web] [--paging]
                [--password <password>] [--ping <ping>] [--port-offset <portOffset>]
                [--queues <queues>] [--relax-jolokia] [--replicated] [--require-login]
                [--role <role>] [--security-manager <securityManager>] [--shared-store]
                [--silent] [--slave] [--ssl-key <sslKey>]
                [--ssl-key-password <sslKeyPassword>] [--ssl-trust <sslTrust>]
                [--ssl-trust-password <sslTrustPassword>] [--static-cluster <staticNode>]
                [--use-client-auth] [--user <user>] [--verbose] [--] <directory>

OPTIONS
        --addresses <addresses>
            Comma separated list of addresses

        --aio
            Sets the journal as asyncio.

        --allow-anonymous
            Enables anonymous configuration on security, opposite of
            --require-login (Default: input)

        --autocreate
            Auto create addresses. (default: true)

        --blocking
            Block producers when address becomes full, opposite of --paging
            (Default: false)

        --cluster-password <clusterPassword>
            The cluster password to use for clustering. (Default: input)

        --cluster-user <clusterUser>
            The cluster user to use for clustering. (Default: input)

        --clustered
            Enable clustering

        --data <data>
            Directory where ActiveMQ data are stored. Paths can be absolute or
            relative to artemis.instance directory ('data' by default)

        --default-port <defaultPort>
            The port number to use for the main 'artemis' acceptor (Default:
            61616)

        --disable-persistence
            Disable message persistence to the journal

        --encoding <encoding>
            The encoding that text files should use

        --etc <etc>
            Directory where ActiveMQ configuration is located. Paths can be
            absolute or relative to artemis.instance directory ('etc' by
            default)

        --failover-on-shutdown
            Valid for shared store: will shutdown trigger a failover? (Default:
            false)

        --force
            Overwrite configuration at destination directory

        --global-max-size <globalMaxSize>
            Maximum amount of memory which message data may consume (Default:
            Undefined, half of the system's memory)

        --home <home>
            Directory where ActiveMQ Artemis is installed

        --host <host>
            The host name of the broker (Default: 0.0.0.0 or input if clustered)

        --http-host <httpHost>
            The host name to use for embedded web server (Default: localhost)

        --http-port <httpPort>
            The port number to use for embedded web server (Default: 8161)

        --java-options <javaOptions>
            Extra java options to be passed to the profile

        --jdbc
            It will activate jdbc

        --jdbc-bindings-table-name <jdbcBindings>
            Name of the jdbc bindings table

        --jdbc-connection-url <jdbcURL>
            The connection used for the database

        --jdbc-driver-class-name <jdbcClassName>
            JDBC driver classname

        --jdbc-large-message-table-name <jdbcLargeMessages>
            Name of the large messages table

        --jdbc-lock-expiration <jdbcLockExpiration>
            Lock expiration

        --jdbc-lock-renew-period <jdbcLockRenewPeriod>
            Lock Renew Period

        --jdbc-message-table-name <jdbcMessages>
            Name of the jdbc messages table

        --jdbc-network-timeout <jdbcNetworkTimeout>
            Network timeout

        --jdbc-node-manager-table-name <jdbcNodeManager>
            Name of the jdbc node manager table

        --jdbc-page-store-table-name <jdbcPageStore>
            Name of the page store messages table

        --journal-device-block-size <journalDeviceBlockSize>
            The block size by the device, default at 4096.

        --mapped
            Sets the journal as mapped.

        --max-hops <maxHops>
            Number of hops on the cluster configuration

        --message-load-balancing <messageLoadBalancing>
            Load balancing policy on cluster. [ON_DEMAND (default) | STRICT |
            OFF]

        --name <name>
            The name of the broker (Default: same as host)

        --nio
            Sets the journal as nio.

        --no-amqp-acceptor
            Disable the AMQP specific acceptor.

        --no-autocreate
            Disable Auto create addresses.

        --no-autotune
            Disable auto tuning on the journal.

        --no-fsync
            Disable usage of fdatasync (channel.force(false) from java nio) on
            the journal

        --no-hornetq-acceptor
            Disable the HornetQ specific acceptor.

        --no-mqtt-acceptor
            Disable the MQTT specific acceptor.

        --no-stomp-acceptor
            Disable the STOMP specific acceptor.

        --no-web
            Remove the web-server definition from bootstrap.xml

        --paging
            Page messages to disk when address becomes full, opposite of
            --blocking (Default: true)

        --password <password>
            The user's password (Default: input)

        --ping <ping>
            A comma separated string to be passed on to the broker config as
            network-check-list. The broker will shutdown when all these
            addresses are unreachable.

        --port-offset <portOffset>
            Off sets the ports of every acceptor

        --queues <queues>
            Comma separated list of queues with the option to specify a routing
            type. (ex: --queues myqueue,mytopic:multicast)

        --relax-jolokia
            disable strict checking on jolokia-access.xml

        --replicated
            Enable broker replication

        --require-login
            This will configure security to require user / password, opposite of
            --allow-anonymous

        --role <role>
            The name for the role created (Default: amq)

        --security-manager <securityManager>
            Which security manager to use - jaas or basic (Default: jaas)

        --shared-store
            Enable broker shared store

        --silent
            It will disable all the inputs, and it would make a best guess for
            any required input

        --slave
            Valid for shared store or replication: this is a slave server?

        --ssl-key <sslKey>
            The key store path for embedded web server

        --ssl-key-password <sslKeyPassword>
            The key store password

        --ssl-trust <sslTrust>
            The trust store path in case of client authentication

        --ssl-trust-password <sslTrustPassword>
            The trust store password

        --static-cluster <staticNode>
            Cluster node connectors list, separated by comma: Example
            "tcp://server:61616,tcp://server2:61616,tcp://server3:61616"

        --use-client-auth
            If the embedded server requires client authentication

        --user <user>
            The username (Default: input)

        --verbose
            Adds more information on the execution

        --
            This option can be used to separate command-line options from the
            list of argument, (useful when arguments might be mistaken for
            command-line options

        <directory>
            The instance directory to hold the broker's configuration and data.
            Path must be writable.
```

### 启动实例

前台启动broker可以执行 ` "/data/activemq-artemis/brokerdata/jms-data/bin/artemis" run`
前台关闭，`CRTL+C`即可退出

```bash
[~bin]# ./artemis run
     _        _               _
    / \  ____| |_  ___ __  __(_) _____
   / _ \|  _ \ __|/ _ \  \/  | |/  __/
  / ___ \ | \/ |_/  __/ |\/| | |\___ \
 /_/   \_\|   \__\____|_|  |_|_|/___ /
 Apache ActiveMQ Artemis 2.31.2


2024-01-26 09:16:02,824 INFO  [org.apache.activemq.artemis.integration.bootstrap] AMQ101000: Starting ActiveMQ Artemis Server version 2.31.2
2024-01-26 09:16:02,888 INFO  [org.apache.activemq.artemis.core.server] AMQ221000: live Message Broker is starting with configuration Broker Configuration (clustered=false,journalDirectory=data/journal,bindingsDirectory=data/bindings,largeMessagesDirectory=data/large-messages,pagingDirectory=data/paging)
2024-01-26 09:16:02,948 INFO  [org.apache.activemq.artemis.core.server] AMQ221013: Using NIO Journal
2024-01-26 09:16:03,018 INFO  [org.apache.activemq.artemis.core.server] AMQ221057: Global Max Size is being adjusted to 1/2 of the JVM max size (-Xmx). being defined as 268435456
2024-01-26 09:16:03,049 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-server]. Adding protocol support for: CORE
2024-01-26 09:16:03,050 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-amqp-protocol]. Adding protocol support for: AMQP
2024-01-26 09:16:03,050 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-hornetq-protocol]. Adding protocol support for: HORNETQ
2024-01-26 09:16:03,051 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-mqtt-protocol]. Adding protocol support for: MQTT
2024-01-26 09:16:03,051 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-openwire-protocol]. Adding protocol support for: OPENWIRE
2024-01-26 09:16:03,051 INFO  [org.apache.activemq.artemis.core.server] AMQ221043: Protocol module found: [artemis-stomp-protocol]. Adding protocol support for: STOMP
2024-01-26 09:16:03,101 INFO  [org.apache.activemq.artemis.core.server] AMQ221034: Waiting indefinitely to obtain live lock
2024-01-26 09:16:03,101 INFO  [org.apache.activemq.artemis.core.server] AMQ221035: Live Server Obtained live lock
2024-01-26 09:16:03,546 INFO  [org.apache.activemq.artemis.core.server] AMQ221080: Deploying address DLQ supporting [ANYCAST]
2024-01-26 09:16:03,546 INFO  [org.apache.activemq.artemis.core.server] AMQ221003: Deploying ANYCAST queue DLQ on address DLQ
2024-01-26 09:16:03,552 INFO  [org.apache.activemq.artemis.core.server] AMQ221080: Deploying address ExpiryQueue supporting [ANYCAST]
2024-01-26 09:16:03,552 INFO  [org.apache.activemq.artemis.core.server] AMQ221003: Deploying ANYCAST queue ExpiryQueue on address ExpiryQueue
2024-01-26 09:16:03,954 INFO  [org.apache.activemq.artemis.core.server] AMQ221020: Started EPOLL Acceptor at 0.0.0.0:61616 for protocols [CORE,MQTT,AMQP,STOMP,HORNETQ,OPENWIRE]
2024-01-26 09:16:03,956 INFO  [org.apache.activemq.artemis.core.server] AMQ221020: Started EPOLL Acceptor at 0.0.0.0:5445 for protocols [HORNETQ,STOMP]
2024-01-26 09:16:03,961 INFO  [org.apache.activemq.artemis.core.server] AMQ221020: Started EPOLL Acceptor at 0.0.0.0:5672 for protocols [AMQP]
2024-01-26 09:16:03,963 INFO  [org.apache.activemq.artemis.core.server] AMQ221020: Started EPOLL Acceptor at 0.0.0.0:1883 for protocols [MQTT]
2024-01-26 09:16:03,965 INFO  [org.apache.activemq.artemis.core.server] AMQ221020: Started EPOLL Acceptor at 0.0.0.0:61613 for protocols [STOMP]
2024-01-26 09:16:03,968 INFO  [org.apache.activemq.artemis.core.server] AMQ221007: Server is now live
2024-01-26 09:16:03,968 INFO  [org.apache.activemq.artemis.core.server] AMQ221001: Apache ActiveMQ Artemis Message Broker version 2.31.2 [0.0.0.0, nodeID=7d0b6dda-b694-11ee-8041-00163e39ecb9] 
2024-01-26 09:16:03,977 INFO  [org.apache.activemq.artemis] AMQ241003: Starting embedded web server
2024-01-26 09:16:04,257 INFO  [org.apache.activemq.hawtio.branding.PluginContextListener] Initialized activemq-branding plugin
2024-01-26 09:16:04,302 INFO  [org.apache.activemq.hawtio.plugin.PluginContextListener] Initialized artemis-plugin plugin
2024-01-26 09:16:04,568 INFO  [io.hawt.HawtioContextListener] Initialising hawtio services
2024-01-26 09:16:04,574 INFO  [io.hawt.system.ConfigManager] Configuration will be discovered via system properties
2024-01-26 09:16:04,576 INFO  [io.hawt.jmx.JmxTreeWatcher] Welcome to Hawtio 2.17.6
2024-01-26 09:16:04,583 INFO  [io.hawt.web.auth.AuthenticationConfiguration] Starting hawtio authentication filter, JAAS realm: "activemq" authorized role(s): "amq" role principal classes: "org.apache.activemq.artemis.spi.core.security.jaas.RolePrincipal"
2024-01-26 09:16:04,590 INFO  [io.hawt.web.auth.LoginRedirectFilter] Hawtio loginRedirectFilter is using 1800 sec. HttpSession timeout
2024-01-26 09:16:04,602 INFO  [io.hawt.web.proxy.ProxyServlet] Proxy servlet is disabled
2024-01-26 09:16:04,609 INFO  [io.hawt.web.servlets.JolokiaConfiguredAgentServlet] Jolokia overridden property: [key=policyLocation, value=file:/data/activemq-artemis/brokerdata/jms-data/etc/jolokia-access.xml]
2024-01-26 09:16:04,693 INFO  [org.apache.activemq.artemis] AMQ241001: HTTP Server started at http://0.0.0.0:8161
2024-01-26 09:16:04,694 INFO  [org.apache.activemq.artemis] AMQ241002: Artemis Jolokia REST API available at http://0.0.0.0:8161/console/jolokia
2024-01-26 09:16:04,694 INFO  [org.apache.activemq.artemis] AMQ241004: Artemis Console available at http://0.0.0.0:8161/console
```

后台启动broker可以执行 `"/data/activemq-artemis/brokerdata/jms-data/bin/artemis-service" start`
后台关闭，`"/data/activemq-artemis/brokerdata/jms-data/bin/artemis stop"`

### 修改配置

```bash
jms-data]# tree -L 1
.
├── bin
├── data
├── etc
├── lib
├── log
└── tmp
```

broker的配置文件，都在etc的目录下

- artemis.profile：系统配置和JVM参数
- artemis-roles.properties： 用户/角色的映射关系
- artemis-users.properties：用户名/密码板块
- bootstrap.xml： 内嵌的WEB服务配置，比如端口、访问地址
- broker.xml： broker的配置
- jolokia-access.xml： Jolokia的安全配置
- log4j2.properties： 日志等级配置等
- login.config： JAAS配置
- management.xml：远程连接配置

### 管理后台

默认情况下，ActiveMQ Artemis的管理控制台将在localhost的8161端口开启，可以通过浏览器访问 `http://localhost:8161/console` 来进行管理和监控。如果在服务器外访问，注意开放8161的端口，8161的端口也可以在web服务配置中修改。

## Docker环境下部署

docker的部署方式相对比较简单

### 获取镜像

```bash
docker pull apache/activemq-artemis:latest
```

具体版本可以自行选择，关于版本信息可在dockerHub查询可知

### 启动服务

暴露61616的broker默认端口和8161的web默认端口

```bash
$ docker run --detach --name jmsdata -p 61616:61616 -p 8161:8161 --rm apache/activemq-artemis:latest
```

不指定用户名密码的的话，默认是artemis/artemis

可以使用shell指令结合用户名密码进行操作

```bash
$ docker exec -it jmsdata /var/lib/artemis-instance/bin/artemis shell --user artemis --password artemis
```

其余配置参照Linux，可以正常进入到容器去修改并重启来生效
查看日志和暂停服务可以直接通过docker指令来实现

欢迎[Star我的博客](https://github.com/CzyerChen/clairechen.github.io)
欢迎[访问我的博客](https://zy5999.cn)
