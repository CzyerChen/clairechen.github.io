---
layout:     post
title:      InfluxDB基础
subtitle:   redis
date:       2022-03-04
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - InfluxDB
---

- [一、InfluxDB是什么？](#一influxdb是什么)
- [二、与Java交互中有哪些对象？](#二与java交互中有哪些对象)
- [三、Docker安装Influxdb](#三docker安装influxdb)
- [四、SpringBoot + InfluxDB 实战监控数据统计](#四springboot--influxdb-实战监控数据统计)
- [五、实际监控](#五实际监控)

## 一、InfluxDB是什么？

InfluxDB是一种时间序列数据库，用于处理带时间标签的数据，用于一些实时检测、检查分析数据采集、高效存储查询统计等方面，在工业数据监控上发挥显著作用。

## 二、与Java交互中有哪些对象？

- database  对应数据库database
- measurement  对应数据库中的表table
- points 对应数据库表中的一行数据row
  - time 时间戳
  - fields 记录的值
  - tags 索引属性


## 三、Docker安装Influxdb

`ps:influxdb 1.x 和2.x有比较大的变化，包含一些指令、安装方式、javaclient的方式，此处使用1.6.1进行实现。`

这边就是描述一个非常容易上手部署的方式，Docker部署

1.第一步，查询镜像，docker search

```bash
a@b:~$ docker search influxdb
NAME                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
influxdb                               InfluxDB is an open source time series datab…   1517                [OK]                
telegraf                               Telegraf is an agent for collecting metrics …   558                 [OK]                
chronograf                             Chronograf is a visualization tool for time …   304                 [OK]                
tutum/influxdb                         InfluxDB image - DEPRECATED. See https://doc…   221                                     [OK]
arm32v7/influxdb                       InfluxDB is an open source time series datab…   25                                      
bitnami/influxdb                                                                       9                                       
sillydong/influxdb-ui                  web ui for influxdb query                       7                                       
prom/influxdb-exporter                 A server that accepts InfluxDB metrics via t…   7                                       
influxdb/influxdb                                                                      6                                       
arm64v8/influxdb                       InfluxDB is an open source time series datab…   6                                       
appcelerator/influxdb                  InfluxDB Docker image                           2                                       [OK]
bitnami/influxdb-relay                 Bitnami InfluxDB Relay Docker Image             1                                       
matisq/influxdb                        TIG Stack - InfluxDB                            1                                       [OK]
hassioaddons/influxdb-amd64                                                            1                                       
dcsg/influxdb-grafana-telegraf         Docker images with Influxdb, Grafana, and Te…   1                                       
amd64/influxdb                         InfluxDB is an open source time series datab…   0                                       
map3/influxdb                                                                          0                                       
dustinbarnes/influxdb-operator         Operator work for influxdb                      0                                       
ibmcom/influxdb-ppc64le                InfluxDB is an open source time series datab…   0                                       
digitalwonderland/influxdb             Latest InfluxDB - clusterable                   0                                       [OK]
monasca/influxdb-init                                                                  0                                       
rsethu/influxdb                        Influxdb                                        0                                       
drpsychick/influxdb                    Influxdb multi-arch docker image based on al…   0                                       
illusiveman/influxdb-to-s3             Backups influxdb to s3                          0                                       
forestscribe/influxdb-docker-collect   collect docker stats from mesos and transmit…   0                                       [OK]
```

2.第二步，拉取镜像，docker pull

```bash
a@b:~$ docker pull influxdb
Using default tag: latest
latest: Pulling from library/influxdb
1c9a8b42b578: Pull complete 
163066942b43: Pull complete 
cf70e03a8272: Pull complete 
e7bb129ae757: Pull complete 
fe41ad6b0f4f: Pull complete 
fc005f6ab372: Pull complete 
a073cbfbfc8b: Pull complete 
6b1db19b0053: Pull complete 
6846a86de2d3: Pull complete 
6e0be9a6fc2b: Pull complete 
Digest: sha256:36a79a82f1d22c5be51365a24fd31d2f2e74a06855bc34d4227d6dd4331de1e6
Status: Downloaded newer image for influxdb:latest

```

3.第三步，查看镜像，docker images

```bash
a@b:~$ docker images influxdb
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
influxdb            latest              201d069e915c        3 weeks ago         346MB
influxdb            1.6.1               34de2bdc2d7f        3 years ago         213MB
```

4.第四步，简单启动influxdb，docker run

```bash
docker run -d -p 8083:8083 -p 8086:8086 --name influxdbserver influxdb
```

5.第五步，确认进程启动完成

```bash
a@b:~$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                    PORTS                                            NAMES
d9672fecc4ca        influxdb            "/entrypoint.sh infl…"   3 seconds ago       Up 2 seconds              0.0.0.0:8083->8083/tcp, 0.0.0.0:8086->8086/tcp   influxdbserver

```

6.第六步，进入容器内部，使用命令行连接influxdb并操作

```bash
a@b:~$ docker exec -it influxdbserver bash
root@d9672fecc4ca:/# cd /usr/bin/
root@d9672fecc4ca:/usr/bin# ls -l 
root@d9672fecc4ca:/# influx
Connected to http://localhost:8086 version 1.6.1
InfluxDB shell version: 1.6.1
> use big_sms
Using database big_sms
> select * from test_analysis;
name: test_analysis
time                count url
----                ----- ---
1646360529255000000 0     /demo
1646360534255000000 0     /demo
1646360539259000000 0     /demo
1646360544255000000 0     /demo
1646360549255000000 0     /demo
1646360554256000000 0     /demo
1646360559257000000 0     /demo
1646360564255000000 0     /demo
1646360569256000000 0     /demo
1646360574257000000 0     /demo
1646360579259000000 0     /demo

```

即可进行相关命令行操作

## 四、SpringBoot + InfluxDB 实战监控数据统计

> 版本说明：
> 
> java 1.8
> 
> springboot 2.3.4.RELEASE
> 
> influxdb 1.6.1

一个简单的Pom文件（看好官方注释的描述，influxdb1.7以及更早版本使用influxdb-java，1.8以及以上使用influxdb-client-java）

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>springboot-forall</artifactId>
        <groupId>com.learning</groupId>
        <version>1.0.0</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>boot-for-influxdb</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

<!--        influxdb-client-java-->
<!--        <dependency>-->
<!--            <groupId>com.influxdb</groupId>-->
<!--            <artifactId>influxdb-client-java</artifactId>-->
<!--            <version>4.3.0</version>-->
<!--        </dependency>-->
<!--        Use this client library with InfluxDB 2.x and InfluxDB 1.8+ (see details).-->
<!--        For connecting to InfluxDB 1.7 or earlier instances, use the influxdb-java client library.-->
        
        <dependency>
            <groupId>org.influxdb</groupId>
            <artifactId>influxdb-java</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

    </dependencies>
</project>
```

一个简单的主类

```java
//主类
@EnableScheduling
@SpringBootApplication
public class InfluxdbTestApplication {
    public static void main(String[] args){
        SpringApplication.run(InfluxdbTestApplication.class,args);
    }
}

```

一个简单的本地版influxdb的配置

```java
spring.influx.url=http://127.0.0.1:8086
spring.influx.user=
spring.influx.password=
```

写一个测试类

```java
@Service
@AllArgsConstructor
@Slf4j
public class MonitorTest {

    private InfluxDB influxDB;

    @Scheduled(fixedRate = 5000)
    public void writeQps(){
        int count = (int)Math.random()*10;
        Point point = Point.measurement("test_analysis")//   test_analysis 数据表
                .tag("url","/demo") //url字段
                .addField("count",count) //统计内容
                .time(System.currentTimeMillis(),TimeUnit.MILLISECONDS) //时间
                .build();

        influxDB.write("big_sms","autogen",point);
        log.info("数据内容：{}",count);

    }
}
```

## 五、实际监控

- 常见的结合influxdb的监控有：chronograf、grafana，主要将指标可视化，使业务数据富有意义

- 比如grafana，可以直接对接influxdb的数据源，在Configuration中直接选择influxDB的数据源就可以直接对接

```
InfluxDB Datasource - Native Plugin
Grafana ships with built in support for InfluxDB (> 0.9.x).

There are currently two separate datasources for InfluxDB in Grafana: InfluxDB 0.8.x and the latest InfluxDB release. The API and capabilities of latest (> 0.9.x) InfluxDB are completely different from InfluxDB 0.8.x which is why Grafana handles them as different data sources.

InfluxDB 0.8 is no longer maintained by InfluxDB Inc, but we provide support as a convenience to existing users. You can find it here.

Read more about InfluxDB here:

http://docs.grafana.org/datasources/influxdb/
```

- 在grafana的analysis中，直接书写influxdb指标的统计公式，保存放入dashhboard即可获得你想要的周期性图表

```
SELECT sum("length") FROM "mea_sms_queue_info" WHERE ("queue" =~ /^$queue$/) AND $timeFilter GROUP BY time(5m) fill(0)
```




