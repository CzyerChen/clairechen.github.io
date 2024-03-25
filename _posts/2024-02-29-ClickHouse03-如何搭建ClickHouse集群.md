---
layout:     post
title:      ClickHouse03-小白如何快速搭建ClickHouse集群
subtitle:   ClickHouse
date:       2024-03-18
author:     Claire
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - ClickHouse
---

普通测试通常使用ClickHouse单节点就可以了，但是生产环境不免需要考虑多活、负载等高可用问题，集群就成了基础需求

ClickHouse在集群的选择上，作者已知的有两种： 使用ZooKeeper作为节点协调的组件，使用ClickHouse-Keeper作为节点协调的组件：

1. 在ZooKeeper中存储集群的元数据信息，如表结构、分片配置以及集群节点状态等，通过ZooKeeper，ClickHouse能够实现在分布式环境下的元数据管理和节点间通信的协调。ZooKeeper的部署和使用也是大家比较熟悉的了。
2. 基于已知的ZooKeeper在部分场景下响应不佳的前提下，ClickHouse Keeper基于Raft一致性算法开发的一款专门为ClickHouse设计的分布式一致性解决方案，旨在替代ZooKeeper作为ClickHouse集群的元数据存储与管理工具。它提供了高可用性和强一致性保证，简化了ClickHouse集群的部署和维护，并且针对ClickHouse的工作负载进行了优化。这个组件的部署还分为独立集群和嵌入式的。

两种选择均可。

- [ZooKeeper方式搭建CK集群](#zookeeper方式搭建ck集群)
  - [手动部署](#手动部署)
    - [手动部署ZK](#手动部署zk)
    - [手动部署CK](#手动部署ck)
  - [docker-compose部署](#docker-compose部署)
- [ClickHouse-Keeper方式搭建CK集群](#clickhouse-keeper方式搭建ck集群)
  - [ClickHouse-Keeper嵌入式](#clickhouse-keeper嵌入式)
  - [ClickHouse-Keeper独立集群](#clickhouse-keeper独立集群)
    - [手动部署模式](#手动部署模式)
    - [容器化部署模式](#容器化部署模式)
    - [Keeper自身服务监控](#keeper自身服务监控)

## ZooKeeper方式搭建CK集群

### 手动部署

#### 手动部署ZK

使用[官方指导](https://zookeeper.apache.org/doc/current/zookeeperStarted.html#getting-started-coordinating-distributed-applications-with-zooKeeper)快速部署一个单节点

正常单节点部署流程：

1. 根据自身环境，[下载一个安装包](http://zookeeper.apache.org/releases.html)
2. 解压安装包，并进入根目录
3. 修改`conf/zoo.cfg`配置文件

```conf
tickTime=2000               #描述票据的时间，用来处理心跳或者session过期，毫秒
dataDir=/var/lib/zookeeper  #本地用于存储内存数据快照的目录
clientPort=2181             #通信端口
```

4. 启动服务 `bin/zkServer.sh start`
5. 查看日志，确认启动成功

集群模式需要至少3个服务节点，2个服务节点本质上是不如单节点稳定，并不推荐
多节点集群的部署流程：

1. 在每个服务节点，下载安装包，如单节点
2. 在每个服务节点，解压安装包，进入根目录
3. 在每个服务节点，进行配置修改 `conf/zoo.cfg`

```conf
tickTime=2000               #描述票据的时间，用来处理心跳或者session过期，毫秒
dataDir=/var/lib/zookeeper  #本地用于存储内存数据快照的目录
clientPort=2181             #通信端口
initLimit=5                 #表示新节点必须连接到leader的时间限制 initLimit*tickTime=最终毫秒数
syncLimit=2                 #表示服务节点要leader间过期的时限 syncLimit*tickTime=最终毫秒数
server.1=zoo1:2888:3888     #服务节点1
server.2=zoo2:2888:3888     #服务节点2
server.3=zoo3:2888:3888     #服务节点3
```

如果3个服务节点位于一个服务器上，也就是端口无法相同，那么请区分开，比如 `2888:3888, 2889:3889, 2890:3890`
4. 分别启动那个服务，查看日志

#### 手动部署CK

在多个服务器上分别部署ClickHouse，下载和基础安装步骤均可参考[官方说明](https://clickhouse.com/docs/en/install)

单个服务节点都需要：

1. 根据自身环境，下载安装包
2. 解压安装服务节点
3. 修改配置文件 `/etc/clickhouse-server/config.xml`(默认文件)，如果你想调整文件的位置和名称，启动服务时需指定 `clickhouse-server --config-file=/etc/clickhouse-server/config.xml`
4. 增加 metrika.xml 配置ZK的地址，修改config.xml 引入metrika的配置，此外需要根据自身情况定义好分片和副本的数量。

> 关于什么是分片？
> 通过定义Distributed表引擎或使用Replicated表引擎结合MergeTree系列引擎可以实现分片功能。分片有助于水平扩展数据存储能力，并且可以根据需要在不同的物理服务器上进行负载均衡

> 关于什么是副本？
> 通过Replicated表引擎（如ReplicatedMergeTree）可以在集群中的不同节点上创建相同结构表的副本，这样即使某个节点发生故障，其他拥有副本的节点仍然可以提供服务，从而保证了数据的高可用性

/etc/clickhouse-server/metrika.xml:

```xml
<yandex>
    <clickhouse_remote_servers>
        <cluster_2s_2r>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>127.0.0.1</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
                <replica>
                    <host>127.0.0.1</host>
                    <port>9100</port>
                    <user>default</user>
                    <password></password>
                </replica>
            </shard>
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>127.0.0.1</host>
                    <port>9100</port>
                    <user>default</user>
                    <password></password>
                </replica>
                <replica>
                    <host>127.0.0.1</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
            </shard>
        </cluster_2s_2r>
    </clickhouse_remote_servers>

    <zookeeper-servers>
        <node index="1">
            <host>127.0.0.1</host>
            <port>2181</port>
        </node>
         <node index="2">
            <host>127.0.0.1</host>
            <port>2182</port>
        </node>
         <node index="3">
            <host>127.0.0.1</host>
            <port>2183</port>
        </node>
    </zookeeper-servers>

    <macros>
        <layer>01</layer>
        <shard>01</shard><!--分片的定义需要不同-->
        <replica>cluster01-01-1</replica> <!--副本的定义需要不同-->
    </macros>
    <networks>
        <ip>::/0</ip>
    </networks>

    <clickhouse_compression>
        <case>
            <min_part_size>10000000000</min_part_size>
            <min_part_size_ratio>0.01</min_part_size_ratio>
            <method>lz4</method>
        </case>
    </clickhouse_compression>
</yandex>

```

/etc/clickhouse-server/config.xml

```xml
<clickhouse>
 ...
 <!--引入配置-->
      <include_from>/etc/clickhouse-server/metrika.xml</include_from>
 ...
</clickhouse>
```

5. 正常启动所有服务节点，`systemctl start clickhouse-server`（启动 clickhouse-server 新旧版有几种方式，此种为最新推荐的形式）

### docker-compose部署

使用docker-compose编排部署

docker-compose.yml:

```yml
version: '3.8'
services: 
  zoo1:
    image: zookeeper:latest
    container_name: zoo1
    environment:
      - ZOO_MY_ID=1
      - ZOO_SERVERS=server.1=zoo1:2888:3888;server.2=zoo2:2888:3888;server.3=zoo3:2888:3888
    ports:
      - "2181:2181"
    volumes:
      - ./data/zoo1/data:/data
      - ./data/zoo1/datalog:/datalog
    networks:
      - ckcluster
  zoo2:
    image: zookeeper:latest
    container_name: zoo2
    environment:
      - ZOO_MY_ID=2
      - ZOO_SERVERS=server.1=zoo1:2888:3888;server.2=zoo2:2888:3888;server.3=zoo3:2888:3888
    ports:
      - "2182:2181"
    volumes:
      - ./data/zoo1/data:/data
      - ./data/zoo1/datalog:/datalog
    networks:
      - ckcluster 
  zoo3:
    image: zookeeper:latest
    container_name: zoo3
    environment:
      - ZOO_MY_ID=3
      - ZOO_SERVERS=server.1=zoo1:2888:3888;server.2=zoo2:2888:3888;server.3=zoo3:2888:3888
    ports:
      - "2183:2181"
    volumes:
      - ./data/zoo1/data:/data
      - ./data/zoo1/datalog:/datalog 
    networks:
      - ckcluster       
  cknode1:
    image: clickhouse/clickhouse-server
    container_name: cknode1
    hostname: cknode1
    volumes:
      - ./data/clickhousenode1/data:/var/lib/clickhouse
      - ./data/clickhousenode1/conf/clickhouse-server/:/etc/clickhouse-server/
    depends_on:
      - zoo1
      - zoo2
      - zoo3
    ports:
      - "9000:9000"
      - "8123:8123"
      - "9009:9009"
      - "9363:9363"
    networks:
      - ckcluster  
  cknode2:
    image: clickhouse/clickhouse-server
    container_name: cknode2
    hostname: cknode2
    volumes:
      - ./data/clickhousenode2/data:/var/lib/clickhouse
      - ./data/clickhousenode2/conf/clickhouse-server/:/etc/clickhouse-server/
    depends_on:
      - zoo1
      - zoo2
      - zoo3
    ports:
      - "9100:9100"
      - "8124:8124"
      - "9109:9109"
      - "9364:9364"    
    networks:
      - ckcluster
networks:
  ckcluster:
    external: true
```

在docker-compose.yml的根目录下，启动服务`docker-compose up -d`

查看日志 `docker-compose logs -f` 确认服务是否启动成功，如有问题就修复后重启

如需重启：

- `docker-compose restart` 全部重启
- `docker-compose restart cknode1` 仅重启cknode1服务
- `docker-compose down && docker-compose up -d` 全部暂停再启动

## ClickHouse-Keeper方式搭建CK集群

### ClickHouse-Keeper嵌入式

嵌入式模式代表不需要额外部署和启动服务，在ClickHouse中配置启用，启动ClickHouse就可以启动嵌入式Keeper

修改 `/etc/clickhouse-server/config.xml`

```xml
<clickhouse>
 <logger>
        <!-- Possible levels [1]:

          - none (turns off logging)
          - fatal
          - critical
          - error
          - warning
          - notice
          - information
          - debug
          - trace

            [1]: https://github.com/pocoproject/poco/blob/poco-1.9.4-release/Foundation/include/Poco/Logger.h#L105-L114
        -->
        <level>trace</level>
        <log>/var/log/clickhouse-keeper/clickhouse-keeper.log</log>
        <errorlog>/var/log/clickhouse-keeper/clickhouse-keeper.err.log</errorlog>
        <!-- Rotation policy
             See https://github.com/pocoproject/poco/blob/poco-1.9.4-release/Foundation/include/Poco/FileChannel.h#L54-L85
          -->
        <size>500M</size>
        <count>10</count>
        <!-- <console>1</console> --> <!-- Default behavior is autodetection (log to console if not daemon mode and is tty) -->
    </logger>
...
    <keeper_server>
            <tcp_port>9181</tcp_port>

            <!-- Must be unique among all keeper serves -->
            <server_id>1</server_id>

            <log_storage_path>/var/lib/clickhouse/coordination/logs</log_storage_path>
            <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

            <coordination_settings>
                <operation_timeout_ms>10000</operation_timeout_ms>
                <min_session_timeout_ms>10000</min_session_timeout_ms>
                <session_timeout_ms>100000</session_timeout_ms>
                <raft_logs_level>information</raft_logs_level>
                <compress_logs>false</compress_logs>
                <!-- All settings listed in https://github.com/ClickHouse/ClickHouse/blob/master/src/Coordination/CoordinationSettings.h -->
            </coordination_settings>

            <!-- enable sanity hostname checks for cluster configuration (e.g. if localhost is used with remote endpoints) -->
            <hostname_checks_enabled>true</hostname_checks_enabled>
            <raft_configuration>
                <server>
                    <id>1</id>

                    <!-- Internal port and hostname -->
                    <hostname>localhost</hostname>
                    <port>9234</port>
                </server>

                <!-- Add more servers here -->

            </raft_configuration>
    </keeper_server>
...
   <zookeeper>
        <node>
            <host>localhost</host>
            <port>9181</port>
        </node>
     </zookeeper>
</clickhouse>
```

以上为手动部署模式，如果是 docker 或 K8S 模式，请将 host 替换为 container name.

启动直接是采用 clickhouse 的启动方式，`systemctl start clickhouse-server`（启动 clickhouse-server 新旧版有几种方式均可）

此外同样的部署多个服务节点，需调整 `<zookeeper>` 和 `<macros>` 下的地址配置，设置合理的分片和副本

### ClickHouse-Keeper独立集群

独立集群模式，意思是独立于ClickHouse-server之外，可以有更多的灵活性，不需要与ClickHouse-server进行一一捆绑，会更像ZooKeeper集群，可以独立运作，支持单独的指标监控
[官方部署说明文档参考](https://clickhouse.com/docs/en/guides/sre/keeper/clickhouse-keeper)

#### 手动部署模式

`手动部署模式`下，ClickHouse-Keeper在ClickHouse-server部署完后就已经存在：

- 配置位于 `/etc/clickhouse-keeper/keeper_config.xml` 就是其配置文件，内容类似于嵌入式的配置，但是需要额外放开IPV6访问和SSL配置 `<listen_host>0.0.0.0</listen_host>`
- 启动可以通过 `systemctl start clickhouse-keeper`

这样你就开启了一个单独的ClickHouse-Keeper节点，如果要与ClickHouse-server绑定互动起来，就需要在 `/etc/clickhouse-server/config.xml` 中完善 `<zookeeper/>` 节点的配置

#### 容器化部署模式

`容器化部署模式`下，选取ClickHouse-Keeper独立的镜像，对于它所需的配置文件进行挂载然后启动，需要与ClickHouse-server进行互动的话，配置同手动部署

对于ClickHouse集群，此外就是多部署几个ClickHouse服务节点，将Keeper的配置同步配置到 `<zookeeper>`的节点中

#### Keeper自身服务监控

对于独立集群运作的Keeper集群，可以独立校验它的状态和监控指标

```bash
$:echo ruok | nc localhost 9181
imok

$:echo mntr | nc localhost 9181
zk_version      v24.2.1.2248-testing-891689a41506d00aa169548f5b4a8774351242c4
zk_avg_latency  0
zk_max_latency  0
zk_min_latency  0
zk_packets_received     0
zk_packets_sent 0
zk_num_alive_connections        0
zk_outstanding_requests 0
zk_server_state standalone
zk_znode_count  10
zk_watch_count  0
zk_ephemerals_count     0
zk_approximate_data_size        1570
zk_key_arena_size       0
zk_latest_snapshot_size 0
zk_open_file_descriptor_count   34
zk_max_file_descriptor_count    500000
zk_followers    0
zk_synced_followers     0

$:echo stat | nc localhost 9181
ClickHouse Keeper version: v24.2.1.2248-testing-891689a41506d00aa169548f5b4a8774351242c4
Clients:
 [::1]:44734(recved=0,sent=0)

Latency min/avg/max: 0/0/0
Received: 0
Sent: 0
Connections: 0
Outstanding: 0
Zxid: 0x5be
Mode: standalone
Node count: 10
```

作为这个独立的组件，也有自己的 Prometheus 端点，可供监控使用

----
如果喜欢我的文章的话，可以去[GitHub上给一个免费的关注](https://github.com/CzyerChen/)吗？
