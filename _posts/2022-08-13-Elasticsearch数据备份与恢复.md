---
layout:     post
title:      ElasticSearch 数据备份与恢复
subtitle:   redis
date:       2022-08-12
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Elasticsearch
    - 数据备份
    - 数据恢复
---

以下为背景

> Elasticsearch 7.6.2单点，8.3.3单点
> 
> Docker 部署
> 
> 当前使用场景：部分index，数据量较大，需要在跨版本的ES之间进行迁移

- [一、前提说明](#一前提说明)
  - [1. Elasticsearch备份](#1-elasticsearch备份)
  - [2. 备份恢复方案](#2-备份恢复方案)
- [二、Elasticsearch 环境准备](#二elasticsearch-环境准备)
  - [1.查看Elasticsearch所有版本](#1查看elasticsearch所有版本)
  - [2.部署2个Elasticsearch单点服务](#2部署2个elasticsearch单点服务)
- [三、数据备份](#三数据备份)
  - [1. 增加仓库配置项](#1-增加仓库配置项)
  - [2. API指令在原始ES集群创建仓库](#2-api指令在原始es集群创建仓库)
  - [3. API查看原始ES集群里创建的仓库](#3-api查看原始es集群里创建的仓库)
  - [4. API验证原始ES集群备份仓库是否创建成功](#4-api验证原始es集群备份仓库是否创建成功)
  - [5. API创建快照备份](#5-api创建快照备份)
  - [6. API删除备份](#6-api删除备份)
- [四、数据恢复](#四数据恢复)
  - [1. 必做步骤](#1-必做步骤)
  - [2. API 恢复备份](#2-api-恢复备份)
  - [3. API 确认数据](#3-api-确认数据)


## 一、前提说明

### 1. Elasticsearch备份

通常Elasticsearch备份有几种情况：

- 大量/全量数据的迁移
- 增量数据的双写或异步准实时/定时同步
- 临时部分数据的导出导入

### 2. 备份恢复方案

那么与之对应的解决方案有哪些呢？

什么样的方法适合什么样的场景呢？

- **Elasticsearch** 原生自带的备份Snapshot与恢复Restore的功能，能够同步部分index、数据量级可允许GB\TB\PB的场景，原生自带的相对稳定性有保障，离线操作
- **Logstash**的管道同步方案，elk这套中我们也有使用到logstash作为一个重要的中间解析环节，我们这边作为数据同步的场景，需要中间对源数据（可来自ES或其他渠道）字段进行转换、处理、新增、删除的场景，还有异步增量同步场景
- **Elasticsearch-dump** 它是一款一款开源的 ES 数据迁移工具，node.js 开发的，需要自行从github下载下来部署后使用，可指定index\mapping等，类似于mysqldump，最终是将数据导出为类似insert的语句后写入，适合数据量小的场景
- 其他开源工具：Kettle（是关系型到关系型/非关系型的数据同步，ES仅写入），DataX（比较完备完整的关系型到关系型/非关系型的数据同步组件，ES仅写入），Flinkx（ES可读取和写入），这种通常涉及关系型到非关系型，或者流程复杂需要二次包装的场景再考虑

基于以上说明，本地实验主要采取**原生自带的备份snapshot与恢复restore的功能**进行数据的迁移测试

## 二、Elasticsearch 环境准备

### 1.查看Elasticsearch所有版本

```bash
curl https://registry.hub.docker.com/v1/repositories/elasticsearch/tags | tr -d '[\[\]" ]' | tr '}' '\n' | awk -F: -v image='elasticsearch' '{if(NR!=NF && $3 != ""){printf("%s:%s\n",image,$3)}}'
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  8250    0  8250    0     0   8620      0 --:--:-- --:--:-- --:--:--  8611
elasticsearch:1
elasticsearch:1-alpine
elasticsearch:1.3.7
elasticsearch:1.3.8
elasticsearch:1.3.9
elasticsearch:1.4
elasticsearch:1.4.2
........
elasticsearch:8.1.2
elasticsearch:8.1.3
elasticsearch:8.2.0
elasticsearch:8.2.1
elasticsearch:8.2.2
elasticsearch:8.2.3
elasticsearch:8.3.1
elasticsearch:8.3.2
```

### 2.部署2个Elasticsearch单点服务

获取镜像

```bash
docker pull elasticsearch:7.6.2
docker pull elasticsearch:8.3.3

8.3.3: Pulling from library/elasticsearch
d7bfe07ed847: Pull complete 
46789798a785: Pull complete 
17d53e557d11: Pull complete 
1044b0d3c7dc: Pull complete 
6d3e6541b7e9: Pull complete 
e56c78671a82: Pull complete 
b3b71c18a4f1: Pull complete 
5208b7309c64: Pull complete 
067f1fbe449c: Pull complete 
Digest: sha256:966.....2e1
Status: Downloaded newer image for elasticsearch:8.3.3
```

启动服务

```bash
docker run --name elasticsearch1 -p 9200:9201 -p 9300:9301 \
> -e  "discovery.type=single-node" \
> -e ES_JAVA_OPTS="-Xms64m -Xmx256m" \
> -v ~/docker/elasticsearch1/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
> -v ~/docker/elasticsearch1/data:/usr/share/elasticsearch/data \
> -v  ~/docker/elasticsearch1/plugins:/usr/share/elasticsearch/plugins \
> -v  ~/docker/elasticsearch/backup:/usr/share/elasticsearch/backup \
> -d elasticsearch:7.6.2


docker run --name elasticsearch2 -p 9200:9201 -p 9300:9301 \
> -e  "discovery.type=single-node" \
> -e ES_JAVA_OPTS="-Xms64m -Xmx256m" \
> -v ~/docker/elasticsearch2/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
> -v ~/docker/elasticsearch2/data:/usr/share/elasticsearch/data \
> -v  ~/docker/elasticsearch2/plugins:/usr/share/elasticsearch/plugins \
> -v  ~/docker/elasticsearch/backup:/usr/share/elasticsearch/backup \
> -d elasticsearch:8.3.3
```

确认服务

```bash
a@~:~$ docker ps -a
CONTAINER ID        IMAGE                             COMMAND                  CREATED             STATUS                      PORTS                                             NAMES
9....b1        elasticsearch:8.3.3              "/usr/local/bin/dock…"   9 seconds ago       Up 7 seconds                0.0.0.0:9201->9200/tcp, 0.0.0.0:9301->9300/tcp    elasticsearch2
75....83        elasticsearch:7.6.2             "/usr/local/bin/dock…"   25 minutes ago      Up 25 minutes               0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp    elasticsearch1
```

## 三、数据备份

备份与恢复的版本说明

```text
备份恢复时的版本兼容性：当前版本支持当前版本及升高一个的版本进行恢复不支持跨版本直接恢复。当然如果你想跨版本恢复，可以尝试版本递增滚动升级来达到目的。

反之则不可行，不能递减版本来恢复备份的数据。
```

### 1. 增加仓库配置项

在每个Elastcisearch节点的elasticsearch.yml的配置文件内加入：

```bash
path:
  repo:
    - /usr/share/elasticsearch/backup
```

### 2. API指令在原始ES集群创建仓库

```bash
a@~:~$ curl -H "Content-Type:application/json" -XPUT 'http://127.0.0.1:9200/_snapshot/{mybackup:这是备份的名称}' -d '{
>   "type": "fs",
>   "settings": {
>     "location": "backupdata", #是对于elasticsearc.yml中地址的相对地址，也可以填完整地址
>     "compress": "true"  #压缩
>   }
> }'
```

此外还可以选择的参数有：
  
- location: 仓库地址
- compress: 是否启用压缩，默认为true
- chunk_size: 是否将文件切块，并指定文件块大小，默认：null(不切分)
- max_restore_bytes_per_sec: Snapshot从仓库恢复时的限速，默认：无限制
- max_snapshot_bytes_per_sec: 节点创建Snapshot进入仓库时的限速，默认：40mb/s
- readonly: Snapshot是否只读，默认：false
- 更多参数可以参考：[官方 Repository API](https://www.elastic.co/guide/en/elasticsearch/reference/current/put-snapshot-repo-api.html?spm=a2c6h.12873639.article-detail.8.288077fcjYK26J)


出现以下内容则说明创建成功

```bash
{"acknowledged":true}
```

### 3. API查看原始ES集群里创建的仓库

查看刚才建立的名为mybackup的仓库是否成功了

```bash
a@~:~$ curl -XGET 'http://127.0.0.1:9200/_snapshot'
{
    "mybackup": { 
    "type":"fs",
    "settings":{
        "compress":"true","location":"backupdata"
        }
        }
}
```

以上查询到了就是成功了， 此外可以在es创建的时候外挂的数据目录内看到新建的“backupdata”目录

### 4. API验证原始ES集群备份仓库是否创建成功

通过verify验证节点仓库是否在所有节点已生效

```bash
a@~:~$ curl -XPOST 'http://127.0.0.1:9200/_snapshot/mybackup/_verify'
{
    "nodes":{
        "VsZZX8i9Qk-o2YLwQbNPQQ":{
            "name":"75707f972883"
            }
    }
}
```

### 5. API创建快照备份


```bash
a@~:~$ curl -H "Content-Type:application/json" -XPUT 'http://127.0.0.1:9200/_snapshot/{mybackup:备份仓库的名字}/{snapshot_1:备份的名字}?wait_for_completion=true' -d '{
>   "indices": "sms_send_history_202208,users", #需要备份的index 
>   "ignore_unavailable": true,
>   "include_global_state": false,
>   "metadata": {
>     "taken_by": "testuser",
>     "taken_because": "test for backup."
>   }
> }'
```

我们API选择参数等到其完成，因而API的结果最终会返回

```bash
{"snapshot":{
    "snapshot":"snapshot_1",
    "uuid":"8wrJ_ODOTQeiXN9uNamQRw",
    "version_id":7060299,
    "version":"7.6.2",
    "indices":["sms_send_history_202208","users"],"include_global_state":false,
    "metadata":{
        "taken_by":"testuser",
        "taken_because":"test for backup."
        },
        "state":"SUCCESS",
        "start_time":"2022-08-08T09:43:24.082Z","start_time_in_millis":1659951804082,"end_time":"2022-08-08T09:43:24.484Z","end_time_in_millis":1659951804484,
        "duration_in_millis":402,"failures":[],
        "shards":{
            "total":6,
            "failed":0,
            "successful":6
            }
        }
}
```

### 6. API删除备份

如果导出配置错误\使用完毕，可以将备份删除

```bash
DELETE /_snapshot/my_fs_backup/snapshot_1
DELETE /_snapshot/my_fs_backup/snapshot_*
```

能够指定一个、多个备份名称，或者通配符的方式将匹配的全部删除

## 四、数据恢复

前面配置两个Elasticsearch节点的时候我们将数据备份的目录挂载在了同一个

所以，我们只需要在Elasticsearch2的节点上也新建一下同名的仓库，路径名称与上面Elasticsearch1的一致，恢复的时候即可直接一键恢复，如果无法实现同一目录挂载，可能就要涉及数据的转移

### 1. 必做步骤

```bash
a@~:~$ curl -H "Content-Type:application/json" -XPUT 'http://127.0.01:9201/_snapshot/{mybackup:与原仓库一致}' -d '{
>   "type": "fs",
>   "settings": {
>     "location": "backupdata-与原仓库一致",
>     "compress": "true"
>   }
> }'
{"acknowledged":true}
```

否则会有报错：

```bash
{"error":{"root_cause":[{"type":"repository_missing_exception","reason":"[mybackup] missing"}],"type":"repository_missing_exception","reason":"[mybackup] missing"},"status":404}

```

### 2. API 恢复备份

直接全量恢复

```bash
curl -XPOST 'http://127.0.0.1:9201/_snapshot/mybackup/snapshot_1/_restore'
```

选择部分恢复：

```bash
curl -H "Content-Type:application/json" -XPOST 'http://127.0.0.1:9201/_snapshot/mybackup/snapshot_1/_restore' -d '

{
  "indices": "user",
  "ignore_unavailable": true,   
  "include_global_state": false, # include_global_state默认为true，是设置集群全局状态 
  "rename_pattern": "user_(.+)",  # 重命名索引匹配规则，如： user_1  
  "rename_replacement": "re_user_$1",# 重命名索引为新的规则，如： re_user_1
  "include_aliases": false
}'
```

正常返回值：

```bash
{
    "accepted":true
}
```

### 3. API 确认数据

通过原始指令查看index 和 索引数据

```bash
a@~:~$ curl http://127.0.0.1:9201/_cat/indices
yellow open users                   jSc-YVNhREu_at5b-Nm7vA 5 1 2 0 12.1kb 12.1kb
yellow open sms_send_history_202208 lefrl7cAQkmO69J2r-xO1g 1 1 9 0   63kb   63kb

a@~:~$ curl http://127.0.0.1:9201/sms_send_history_202208/_search?pretty
{
  "took" : 58,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 9,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "sms_send_history_202208",
        "_type" : "sms_send_history",
        "_id" : "efb44600dbbe446789371ae146df654d",
        "_score" : 1.0,
        "_source" : {
          "acctime" : 1659715200000,
          "channelid" : 3,
          "msgcount" : 1,
          "msgbody" : "友情提示：1111111222222222220",
          "sendcode" : "001",
          "@timestamp" : "2022-08-08T0"
          }
      }]
  }
}
```

后续可以再通过页面或其他渠道验证下数据