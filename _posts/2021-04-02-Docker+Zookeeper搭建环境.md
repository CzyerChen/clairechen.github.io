---
layout:     post
title:      Docker Zookeeper 单节点+集群
subtitle:   Dokcer Zookeeper 
date:       2021-04-02
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Docker
    - Zookeeper 单节点
    - Zookeeper 集群
---

## Dokcer 一键快速部署Zookeeper

首先要具备Docker环境,当前实验环境：

- Mac
- Docker: 18.09.2

### 一、Zookeeper单节点模式

- docker search zookeeper
- docker pull zookeeper
- 创建一个数据挂载目录： ~/docker/zookeeper/data，具体自行创建
- cd  ~/docker/zookeeper
- docker run -d -e TZ="Asia/Shanghai" -p 2181:2181 -v $PWD/data:/data --name zookeeper --restart always zookeeper
- docker ps -a ,能够看到zookeeper是处于【Up】的状态

```bash
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                       PORTS                                                  NAMES
f293af3f118a        zookeeper                    "/docker-entrypoint.…"   3 hours ago         Up 2 hours                   2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, 8080/tcp   zookeeper
```

- 测试客户端连接：docker run -it --rm --link zookeeper:zookeeper zookeeper zkCli.sh -server zookeeper
- 进入后能够进行:ls /,create,get, set,delete等操作

```bash
[zk: zookeeper(CONNECTED) 2] ls  /
[zookeeper]
[zk: zookeeper(CONNECTED) 3] create /zktest
Created /zktest
[zk: zookeeper(CONNECTED) 4] ls /
[zktest, zookeeper]
[zk: zookeeper(CONNECTED) 5] get /zktest
null
[zk: zookeeper(CONNECTED) 6] set /zktest data1
[zk: zookeeper(CONNECTED) 7] get /zktest
data1
[zk: zookeeper(CONNECTED) 8] ls /
[zktest, zookeeper]
[zk: zookeeper(CONNECTED) 9] get /zktest
data1
[zk: zookeeper(CONNECTED) 10] set /zktest data2
[zk: zookeeper(CONNECTED) 11] create /zktest/innerpath data1
Created /zktest/innerpath
[zk: zookeeper(CONNECTED) 12] create /zktest/innerpath data2
Node already exists: /zktest/innerpath
[zk: zookeeper(CONNECTED) 13] get /zktest/innerpath 
data1
[zk: zookeeper(CONNECTED) 14] set /zktest/innerpath data2
[zk: zookeeper(CONNECTED) 15] set /zktest/innerpath data3
[zk: zookeeper(CONNECTED) 16] create /zktest/innerpath2
Created /zktest/innerpath2
[zk: zookeeper(CONNECTED) 17] create /zktest/innerpath3
Created /zktest/innerpath3
[zk: zookeeper(CONNECTED) 18] set /zktest/innerpath3 data3
[zk: zookeeper(CONNECTED) 19] get /zktest/innerpath3
data3
[zk: zookeeper(CONNECTED) 20] set /zktest/innerpath2 data2
[zk: zookeeper(CONNECTED) 21] get /zktest/innerpath3
data3
[zk: zookeeper(CONNECTED) 22] set /zktest/innerpath2 data2
[zk: zookeeper(CONNECTED) 23] create /zktest/innerpath4
Created /zktest/innerpath4
```

### 二、Zookeeper集群模式

使用docker-compose进行一键编排

- 选择一个目录：cd  ~/docker/zookeeper
- vi docker-compose.yml
- 填充以下脚本：

```yml
# zk集群的通信网络zk-net
networks:
  zk-net:
    name: zk-net

# 配置zk集群的
services:
  zk1:
    image: zookeeper
    hostname: zk1
    container_name: zk1
    # 配置docker container和宿主机的端口映射
    ports:
      - 2181:2181
      - 8081:8080
    # 配置docker container的环境变量
    environment:
      # 当前zk实例的id,注意唯一性
      ZOO_MY_ID: 1
      # 整个zk集群的机器、端口列表
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=zk3:2888:3888;2181
    # 将docker container上的路径挂载到宿主机上 实现宿主机和docker container的数据共享
    volumes:
      - ./zk1/data:/data
      - ./zk1/datalog:/datalog
    # 当前docker container加入名为zk-net的隔离网络
    networks:
      - zk-net

  zk2:
    image: zookeeper
    hostname: zk2
    container_name: zk2
    ports:
      - 2182:2181
      - 8082:8080
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zk3:2888:3888;2181
    volumes:
      - ./zk2/data:/data
      - ./zk2/datalog:/datalog
    networks:
      - zk-net

  zk3:
    image: zookeeper
    hostname: zk3
    container_name: zk3
    ports:
      - 2183:2181
      - 8083:8080
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
    volumes:
      - ./zk3/data:/data
      - ./zk3/datalog:/datalog
    networks:
      - zk-net

```

- 脚本保存后，在当前目录执行：docker-compose up -d ,将后台启动三个zk节点

```bash
:zookeeper xxx$ docker-compose up -d
Creating network "zookeeper_default" with the default driver
Creating zk3 ... done
Creating zk2 ... done
Creating zk1 ... done
```

- docker-compose ps 或者 docker ps -a 两个命令都能看到三个节点顺利启动
- docker exec -it zk1 /bin/bash 进入zk1节点
- 能够正常进行create ls delete get set等操作
- 查看建立的隔离网络:docker network ls

```bash
xxx:zookeeper xxx$ docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
d1d3225ba249        bridge                bridge              local
2735294840d2        elasticsearch_esnet   bridge              local
16ba5525c054        host                  host                local
c6e9d4238f9b        none                  null                local
1bcc311baa3b        zk-net                bridge              local  -----> Here
```

- 查看某一节点的配置：docker inspect zk1


### 三、其他基础命令

```bash
# 停止docker-compose服务 不删除容器
docker-compose stop

# 停止docker-compose服务 删除容器
docker-compose down

# 启动docker-compose服务
docker-compose start 

# 重启docker-compose服务
docker-compose restart
```