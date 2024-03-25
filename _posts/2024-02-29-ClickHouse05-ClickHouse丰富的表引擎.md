---
layout:     post
title:      ClickHouse05-ClickHouse表引擎
subtitle:   ClickHouse
date:       2024-03-18
author:     Claire
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - ClickHouse
---

ClickHouse有丰富的表引擎，可以让数据的处理和操作合为一体，用SQL的方式查看和处理来自不同形式的数据

## MergeTree

最通用、功能最强大的表引擎，适用于高负载任务

- MergeTree**
- ReplacingMergeTree
- SummingMergeTree
- AggregatingMergeTree
- CollapsingMergeTree
- VersionedCollapsingMergeTree
- GraphiteMergeTree

## Log

功能较少的轻量级引擎

- TinyLog
- StripeLog
- Log

常规属性：

- 数据存储在磁盘上
- 将数据追加到文件的末尾
- 支持锁住并发的数据访问
- 不支持对表的删除和更新Alter操作
- 不支持索引

## 迁移类的引擎

源数据类型：

- ODBC
- JDBC**
- MySQL**
- MongoDB
- Redis**
- HDFS**
- S3
- Kafka**
- EmbeddedRocksDB
- RabbitMQ**
- PostgreSQL**
- S3Queue

支持建立表引擎后，从用户的角度来看，配置的集成看起来像一个普通表，但对它的查询被代理到外部系统

## 特殊的表引擎

- Distributed
- Dictionary
- Merge
- File**
- Null
- Set
- Join
- URL
- View
- Memory**
- Buffer
- KeeperMap

## 输入输出数据格式支持多种

CSV,JSON,Avro

[输入输出数据格式](https://clickhouse.com/docs/en/sql-reference/formats)

----
如果喜欢我的文章的话，可以去[GitHub上给一个免费的关注](https://github.com/CzyerChen/)吗？
