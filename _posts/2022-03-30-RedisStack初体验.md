---
layout:     post
title:      Redis-stack 初体验
subtitle:   redis
date:       2022-03-31
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Redis-stack
    - docker
---

- [一、安装方式](#一安装方式)
- [二、Docker安装流程](#二docker安装流程)
  - [1.选择镜像&获取镜像](#1选择镜像获取镜像)
  - [2.启动容器](#2启动容器)
  
## 一、安装方式

1. 通过源码安装redis-stack
2. 通过docker安装redis-stack
3. 在Linux上安装redis-stack
4. 在MasOS上安装redis-stack

[详情参照官方文档](https://redis.io/docs/stack/)

## 二、Docker安装流程

### 1.选择镜像&获取镜像

在开始使用docker安装redis-stack之前，需要选择一个镜像

```bash
a@b:~$ docker search redis-stack
NAME                               DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
redis/redis-stack                                                                  1                                       
admiralobvious/redis-stackdriver   Docker Redis image with the Stackdriver agen…   0                                       [OK]
redis/redis-stack-server                                                           0                                       
```

以上能够看到两个官方镜像，redis/redis-stack 和 redis/

- redis/redis-stack 同时包含 Redis Stack server 和 RedisInsight。这个镜像会比较是和本地开发环境，可以同时使用 RedisInsight 来查看数据。
- redis/redis-stack-server 仅提供了 Redis Stack server。这个镜像比较适合生产环境。

下面就是安装本地开发环境版本 redis/redis-stack

```bash
a@b:~$ docker pull redis/redis-stack
Using default tag: latest
latest: Pulling from redis/redis-stack
4d32b49e2995: Pull complete 
59db42264c42: Pull complete 
6823dc0c7365: Pull complete 
adc416e0b552: Pull complete 
972811a9db55: Pull complete 
d84df654fab1: Pull complete 
4c53537c30f9: Pull complete 
a731a195d4fb: Pull complete 
475d8d5bf1a2: Pull complete 
f242ebf98bc6: Pull complete 
64827bf73809: Pull complete 
1a1591a03a07: Pull complete 
91a9bd50d46e: Pull complete 
44fd7da8c8ca: Pull complete 
b568ddb1639f: Pull complete 
Digest: sha256:27666e8e1b632cc02bfb926bf9cbbda650aed2b818444c58613379167e12369e
Status: Downloaded newer image for redis/redis-stack:latest
```

### 2.启动容器



```bash
#针对redis/redis-stack，需要同时启动RedisInsight
docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest

#redis/redis-stack-server
docker run -d --name redis-stack -p 6379:6379 redis/redis-stack:latest

```

当然此时使用本地测试的 redis/redis-stack 版本，确定启动后，可以通过 http://localhost:8001 来可视化查看

```bash
a@b:~$ docker ps -a
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS                       PORTS                                            NAMES
f4e4c99042d8        redis/redis-stack:latest   "/entrypoint.sh"         3 seconds ago       Up 2 seconds                 0.0.0.0:6379->6379/tcp, 0.0.0.0:8001->8001/tcp   redis-stack


a@b:~$ docker logs redis-stack
9:C 30 Mar 2022 10:47:43.125 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
9:C 30 Mar 2022 10:47:43.125 # Redis version=6.2.6, bits=64, commit=00000000, modified=0, pid=9, just started
9:C 30 Mar 2022 10:47:43.125 # Configuration loaded
9:M 30 Mar 2022 10:47:43.126 * monotonic clock: POSIX clock_gettime
9:M 30 Mar 2022 10:47:43.126 * Running mode=standalone, port=6379.
9:M 30 Mar 2022 10:47:43.126 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
9:M 30 Mar 2022 10:47:43.126 # Server initialized
9:M 30 Mar 2022 10:47:43.127 * <search> Redis version found by RedisSearch : 6.2.6 - oss
9:M 30 Mar 2022 10:47:43.127 * <search> RediSearch version 2.2.10 (Git=HEAD-3d0701f6)
9:M 30 Mar 2022 10:47:43.127 * <search> Low level api version 1 initialized successfully
9:M 30 Mar 2022 10:47:43.127 * <search> concurrent writes: OFF, gc: ON, prefix min length: 2, prefix max expansions: 200, query timeout (ms): 500, timeout policy: return, cursor read size: 1000, cursor max idle (ms): 300000, max doctable size: 1000000, max number of search results:  10000, search pool size: 20, index pool size: 8, 
9:M 30 Mar 2022 10:47:43.127 * <search> Initialized thread pool!
9:M 30 Mar 2022 10:47:43.127 * <search> Enabled diskless replication
9:M 30 Mar 2022 10:47:43.127 * Module 'search' loaded from /opt/redis-stack/lib/redisearch.so
9:M 30 Mar 2022 10:47:43.129 * <graph> Starting up RedisGraph version 2.8.9.
9:M 30 Mar 2022 10:47:43.129 * <graph> Thread pool created, using 4 threads.
9:M 30 Mar 2022 10:47:43.129 * <graph> Maximum number of OpenMP threads set to 4
9:M 30 Mar 2022 10:47:43.129 * Module 'graph' loaded from /opt/redis-stack/lib/redisgraph.so
9:M 30 Mar 2022 10:47:43.130 * <timeseries> RedisTimeSeries version 10609, git_sha=f36e5a703dc9a2487880087a34f6cb0e56d9a459
9:M 30 Mar 2022 10:47:43.131 * <timeseries> Redis version found by RedisTimeSeries : 6.2.6 - oss
9:M 30 Mar 2022 10:47:43.131 * <timeseries> loaded default CHUNK_SIZE_BYTES policy: 4096
9:M 30 Mar 2022 10:47:43.131 * <timeseries> loaded server DUPLICATE_POLICY: block
9:M 30 Mar 2022 10:47:43.131 * <timeseries> Setting default series ENCODING to: compressed
9:M 30 Mar 2022 10:47:43.131 * <timeseries> Detected redis oss
9:M 30 Mar 2022 10:47:43.131 * <timeseries> Enabled diskless replication
9:M 30 Mar 2022 10:47:43.131 * Module 'timeseries' loaded from /opt/redis-stack/lib/redistimeseries.so
9:M 30 Mar 2022 10:47:43.131 * <ReJSON> version: 20007 git sha: e51b585 branch: HEAD
9:M 30 Mar 2022 10:47:43.131 * <ReJSON> Exported RedisJSON_V1 API
9:M 30 Mar 2022 10:47:43.131 * <ReJSON> Enabled diskless replication
9:M 30 Mar 2022 10:47:43.131 * <ReJSON> Created new data type 'ReJSON-RL'
9:M 30 Mar 2022 10:47:43.131 * Module 'ReJSON' loaded from /opt/redis-stack/lib/rejson.so
9:M 30 Mar 2022 10:47:43.131 * <search> Acquired RedisJSON_V1 API
9:M 30 Mar 2022 10:47:43.131 * <graph> Acquired RedisJSON_V1 API
9:M 30 Mar 2022 10:47:43.131 * Module 'bf' loaded from /opt/redis-stack/lib/redisbloom.so
9:M 30 Mar 2022 10:47:43.131 * Ready to accept connections
```

启动完成即可照常类似使用redis，使用redis-stack 了

```bash
$ docker exec -it redis-stack redis-cli
```

进入容器，使用redis-cli就能够访问redis

`此外一样可以通过docker -v参数 挂载容器数据，-e参数 传递命令参数`

```bash
$ docker run -v /local-data/:/data redis/redis-stack:latest

$ docker run -e REDIS_ARGS="--requirepass redis-stack" redis/redis-stack:latest

```