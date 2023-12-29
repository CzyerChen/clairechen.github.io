---
layout:     post
title:      Skywalking之JavaAgent本地构建

subtitle:   Skywalking
date:       2023-11-01
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Skywalking
---

- MAC 10.15.7
- IDEA 2021.2
- node v16.14.0 (据代码要求安装了一个新版本)
- npm 8.3.1
- Skywalking 9.6.0-SNAPSHOT
- JDK 11 (最新的代码要看最新的要求，注意不能使用JDK8，会构建失败)
- Maven 3.6.0

## 关于本地构建JavaAgent源码

### 获取源码，加载submodule

分步执行：

```bash
git clone https://github.com/apache/skywalking-java.git
git submodule init //初始化仓库目录位置
git submodule update //**这步骤至关重要**
```

或者一气呵成：

```bash
git clone --recurse-submodules https://github.com/apache/skywalking.git
```

### maven构建

```bash
#注意JDK版本，JDK11
mvn clean package -Dmaven.test.skip=true
```

构建后产生编译的文件，需要加为源码，找到以下文件夹右键 **"Mark Dictory as"** -> **"Generated Sources Root"**

```xml
~/skywalking-java/apm-protocol/apm-network/target/generated-sources/protobuf/grpc-java
~/skywalking-java/apm-protocol/apm-network/target/generated-sources/protobuf/java
```

正常就可以进行本地调试了

### 4、skywalking-agent包

在{home}/skywalking-agent/下，就是我们常规下载的可执行包的内容了

- plugins: 是会被真实扫描的插件列表，可以按需取用。删除/移除就不会被扫描
- optional-plugins: 是补充的一些插件，有需要不在plugins中的可以自行粘贴过去
- bootstrap-plugins：通常默认不动
- optional-reporter-plugins: 按需加入
- activations: 通常默认不动
- config: agent的配置
- licenses:
- logs: agent的日志，通常排查问题可以先看这里有没有报错哦

```bash
 $tree
.
├── LICENSE
├── NOTICE
├── activations
│   ├── apm-toolkit-kafka-activation-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-log4j-1.x-activation-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-log4j-2.x-activation-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-logback-1.x-activation-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-logging-common-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-meter-activation-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-micrometer-activation-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-opentracing-activation-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-trace-activation-9.2.0-SNAPSHOT.jar
│   ├── apm-toolkit-webflux-6.x-activation-9.2.0-SNAPSHOT.jar
│   └── apm-toolkit-webflux-activation-9.2.0-SNAPSHOT.jar
├── bootstrap-plugins
│   ├── apm-jdk-forkjoinpool-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jdk-http-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jdk-threading-plugin-9.2.0-SNAPSHOT.jar
│   └── apm-jdk-threadpool-plugin-9.2.0-SNAPSHOT.jar
├── config
│   └── agent.config
├── licenses
│   └── LICENSE-asm.txt
├── logs
│   └── skywalking-api.log
├── optional-plugins
│   ├── apm-customize-enhance-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-ehcache-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-fastjson-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-gson-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-guava-cache-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jackson-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-kotlin-coroutine-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mybatis-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-nacos-client-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-netty-http-4.1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-quartz-scheduler-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-resttemplate-6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-sentinel-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-shenyu-2.4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-annotation-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-cloud-gateway-2.0.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-cloud-gateway-2.1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-cloud-gateway-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-tx-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-webflux-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-webflux-6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-springmvc-annotation-6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-trace-ignore-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-zookeeper-3.4.x-plugin-9.2.0-SNAPSHOT.jar
│   └── trace-sampler-cpu-policy-plugin-9.2.0-SNAPSHOT.jar
├── optional-reporter-plugins
│   ├── kafka-reporter-plugin-9.2.0-SNAPSHOT.jar
│   ├── lz4-java-1.6.0.jar
│   ├── snappy-java-1.1.7.3.jar
│   └── zstd-jni-1.4.3-1.jar
├── plugins
│   ├── apm-activemq-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-aerospike-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-armeria-0.84.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-armeria-0.85.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-armeria-1.0.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-asynchttpclient-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-avro-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-canal-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-cassandra-java-driver-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-clickhouse-0.3.1-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-clickhouse-0.3.2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-cxf-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-dubbo-2.7.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-dubbo-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-dubbo-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-elastic-job-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-elasticjob-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-elasticsearch-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-elasticsearch-6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-elasticsearch-7.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-feign-default-http-9.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-finagle-6.25.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-grizzly-2.x-4.x-work-threadpool-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-grizzly-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-grpc-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-guava-eventbus-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-h2-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-hbase-1.x-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-hikaricp-3.x-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-httpClient-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-httpasyncclient-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-httpclient-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-httpclient-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-httpclient-commons-9.2.0-SNAPSHOT.jar
│   ├── apm-hutool-http-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-hystrix-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-impala-jdbc-2.6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-influxdb-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jdbc-commons-9.2.0-SNAPSHOT.jar
│   ├── apm-jersey-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jersey-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jetty-client-11.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jetty-client-9.0-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jetty-client-9.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jetty-server-11.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jetty-server-9.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-jetty-thread-pool-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-kafka-commons-9.2.0-SNAPSHOT.jar
│   ├── apm-kafka-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-kylin-jdbc-2.6.x-3.x-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-lettuce-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-light4j-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mariadb-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mongodb-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mongodb-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mongodb-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mssql-commons-9.2.0-SNAPSHOT.jar
│   ├── apm-mssql-jdbc-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mssql-jtds-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mysql-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mysql-6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mysql-8.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-mysql-commons-9.2.0-SNAPSHOT.jar
│   ├── apm-neo4j-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-netty-socketio-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-nutz-http-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-nutz-mvc-annotation-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-okhttp-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-okhttp-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-okhttp-common-9.2.0-SNAPSHOT.jar
│   ├── apm-play-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-postgresql-8.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-pulsar-2.2-2.7-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-pulsar-2.8.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-pulsar-common-9.2.0-SNAPSHOT.jar
│   ├── apm-quasar-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-rabbitmq-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-redisson-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-resttemplate-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-resttemplate-4.3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-rocketMQ-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-rocketmq-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-rocketmq-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-rocketmq-client-java-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-servicecomb-java-chassis-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-sharding-sphere-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-sharding-sphere-4.1.0-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-shardingsphere-4.0.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-shardingsphere-5.0.0-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-solrj-7.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-async-annotation-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-cloud-feign-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-cloud-feign-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-concurrent-util-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-core-patch-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-kafka-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-kafka-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-scheduled-annotation-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-webflux-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-spring-webflux-6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-springmvc-annotation-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-springmvc-annotation-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-springmvc-annotation-5.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-springmvc-annotation-6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-springmvc-annotation-commons-9.2.0-SNAPSHOT.jar
│   ├── apm-spymemcached-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-struts2-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-tomcat-thread-pool-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-undertow-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-undertow-worker-thread-pool-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-vertx-core-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-vertx-core-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-xmemcached-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── apm-xxl-job-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── baidu-brpc-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── baidu-brpc-plugin-9.2.0-SNAPSHOT.jar
│   ├── dbcp-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── druid-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── dubbo-2.7.x-conflict-patch-9.2.0-SNAPSHOT.jar
│   ├── dubbo-3.x-conflict-patch-9.2.0-SNAPSHOT.jar
│   ├── dubbo-conflict-patch-9.2.0-SNAPSHOT.jar
│   ├── elasticsearch-common-9.2.0-SNAPSHOT.jar
│   ├── graphql-12.x-15.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── graphql-16plus-plugin-9.2.0-SNAPSHOT.jar
│   ├── graphql-8.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── graphql-9.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── jedis-2.x-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── jedis-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── jsonrpc4j-1.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── micronaut-http-client-plugin-9.2.0-SNAPSHOT.jar
│   ├── micronaut-http-server-plugin-9.2.0-SNAPSHOT.jar
│   ├── motan-plugin-9.2.0-SNAPSHOT.jar
│   ├── nats-2.14.x-2.15.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── okhttp-2.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── resteasy-server-3.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── resteasy-server-4.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── resteasy-server-6.x-plugin-9.2.0-SNAPSHOT.jar
│   ├── resttemplate-commons-9.2.0-SNAPSHOT.jar
│   ├── sofa-rpc-plugin-9.2.0-SNAPSHOT.jar
│   ├── spring-commons-9.2.0-SNAPSHOT.jar
│   ├── spring-webflux-5.x-webclient-plugin-9.2.0-SNAPSHOT.jar
│   ├── spring-webflux-6.x-webclient-plugin-9.2.0-SNAPSHOT.jar
│   ├── thrift-plugin-9.2.0-SNAPSHOT.jar
│   ├── tomcat-10x-plugin-8.11.0.jar
│   ├── tomcat-7.x-8.x-plugin-9.2.0-SNAPSHOT.jar
│   └── websphere-liberty-23.x-plugin-9.2.0-SNAPSHOT.jar
└── skywalking-agent.jar

8 directories, 196 files
```

在应用中就可以加这个路径直接使用，日志见logs文件夹
