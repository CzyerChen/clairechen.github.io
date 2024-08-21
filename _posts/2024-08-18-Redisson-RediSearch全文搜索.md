---
layout:     post
title:      RediSearch-Redis的高性能全文搜索

subtitle:   RediSearch
date:       2024-08-18
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Redis
    - Redisson
    - RediSearch
---

- [什么是RediSearch](#什么是redisearch)
- [能解决什么问题](#能解决什么问题)
- [加入RediSearch模块有什么好处](#加入redisearch模块有什么好处)
- [如何部署](#如何部署)
- [命令行如何使用](#命令行如何使用)
- [Redisson如何使用](#redisson如何使用)
- [The end](#the-end)

## 什么是RediSearch

RediSearch是Redis Labs开发的一个模块，它为Redis添加了高性能的全文搜索功能。

## 能解决什么问题

RediSearch 的出现给 Redis 带来比较大热度。我们通常使用文档查询最为常见的就是 Elasticsearch，它的搜索开始的很早，风很大，企业实践也是如雨后春笋。Redis 作为缓存介的霸主，已经开始不满足于局限的业务范畴，RediSearch的出现极大地扩展了Redis的功能边界，使其不仅仅是一个简单的键值存储系统，而是成为了一个可以用于多种应用场景的强大平台。

## 加入RediSearch模块有什么好处

1、增强搜索能力：
RediSearch使Redis具备了全文搜索的能力，这对于需要在大量文本数据中进行高效搜索的应用非常有用。它支持复杂的查询语法，例如布尔查询、短语匹配、模糊匹配等，这使得Redis成为一个更加强大的数据处理工具。

2、简化架构：
对于需要搜索功能的应用程序，使用RediSearch可以减少对额外搜索服务（如Elasticsearch或Solr）的依赖，从而简化整体架构并降低运维成本。

3、灵活性和扩展性：
RediSearch支持在Redis集群环境中使用，这意味着它可以随着数据量的增长而水平扩展。它还可以与其他Redis模块结合使用，比如RedisJSON，以提供更复杂的数据处理和搜索功能。

## 如何部署

部署的方式有很多，就是源代码编译、预编译文件、docker方式

非常快速的试用方式就是docker部署一个包含search功能的redis，生产环境就编译模块后加入。

- Redis Stack 大礼包

```bash
$ docker run -p 6379:6379 redis/redis-stack-server:latest
```

`redislabs/redisearch` 的镜像已经过期，老版本和老demo中还有使用这个镜像，但是新更新不会再继续，需要下载 `redis-stack` 的docker镜像。

- 连接客户端

|Project|	Language|	
|--|--|
|jedis⁠	|Java|	
|redis-py|⁠	Python|
|node-redis|⁠	Node.js|
|nredisstack|⁠	.NET|
|redis-om|⁠	Python|⁠	
|lettusearch|⁠	Java|
|spring-redisearch|⁠	Java|
|redis-om-spring|⁠	Java|
|redisearch-go|⁠	Go|
|rueidis|⁠	Go|
|Redis-om⁠|	JavaScript	|
|Redis.OM|⁠	.NET|
|redisearch-php|⁠	PHP|
|php-redisearch|⁠	PHP|
|redisearch-api-rs⁠|	Rust|
|redi_search_rails|⁠	Ruby|
|redisearch-rb|⁠	Ruby|
|redi_search|⁠ Ruby|

## 命令行如何使用

- 创建索引

```bash
> FT.CREATE idx:movie ON hash PREFIX 1 "movie:" SCHEMA title TEXT SORTABLE release_year NUMERIC SORTABLE rating NUMERIC SORTABLE genre TAG SORTABLE

OK
```

- 查看索引

```bash
> FT.INFO idx:movie

 1) index_name
 2) idx:movie
... 
46) 1) global_idle
    2) (integer) 0
...
```

- 修改索引

```bash
> HSET "movie:11005" title "Star Wars: Episode VI - Return of the Jedi"  plot "The Rebels destroy the Empire's Death Star." release_year 1983 genre "Action" rating 8.3 votes 906260 

(integer) 6
```

- 删除索引

```bash
> EXPIRE "movie:11002" 15

(integer) 1
```

- 查询索引

```bash
> FT.SEARCH idx:movie "war" RETURN 3 title release_year rating

1) (integer) 1
2) "movie:11002"
3) 1) "title"
   1) "Star Wars: Episode V - The Empire Strikes Back"
   2) "release_year"
   3) "1980"
   4) "rating"
   5) "8.7"

> FT.SEARCH idx:movie "@title:war" RETURN 3 title release_year rating

> FT.SEARCH idx:movie "emp*" RETURN 3 title release_year rating

> FT.SEARCH idx:movie "%gdfather%" RETURN 3 title release_year rating

> FT.SEARCH idx:movie "war |  %gdfather% " RETURN 3 title release_year rating

> FT.SEARCH idx:movie "@genre:{Drama}" RETURN 3 title release_year rating

1) (integer) 1
2) "movie:11003"
3) 1) "title"
   2) "The Godfather"
   3) "release_year"
   4) "1972"
   5) "rating"
   6) "9.2"

```

所有的指令参考[官方文档](https://redis.io/docs/latest/commands/?group=search)

## Redisson如何使用

构建client

```java
Config config = new Config();
config.useSingleServer().setAddress("redis://" + address);
RedissonClient client = Redisson.create(config);

```

构建查询内容，并根据文本内容搜素

```java
RMap<String, SimpleObject> m = client.getMap(
            "doc:1", new CompositeCodec(StringCodec.INSTANCE, client.getConfig().getCodec()));
m.put("v1", new SimpleObject("name1"));
m.put("v2", new SimpleObject("name2"));
RMap<String, SimpleObject> m2 = client.getMap(
            "doc:2", new CompositeCodec(StringCodec.INSTANCE, client.getConfig().getCodec()));
m2.put("v1", new SimpleObject("name3"));
 m2.put("v2", new SimpleObject("name4"));

RSearch s = client.getSearch();
s.createIndex("idx", IndexOptions.defaults()
                                         .on(IndexType.HASH)
                                         .prefix(Arrays.asList("doc:")),
                      FieldIndex.text("v1"),
                      FieldIndex.text("v2")
        );

SearchResult r1 = s.search("idx", "*", QueryOptions.defaults()
                                                           .returnAttributes(
                                                                        new ReturnAttribute("v1"), new ReturnAttribute("v2")));

SearchResult r2 = s.search("idx", "*", QueryOptions.defaults()
                                                           .filters(QueryFilter.geo("field")
                                                                               .from(1, 1)
                                                                               .radius(10, GeoUnit.FEET)));
```

删除索引

```java                                                                               
s.dropIndex("idx");

```

修改配置

```java
Map<String, String> cfg = s.getConfig("TIMEOUT");
s.setConfig("TIMEOUT", "42");
```

Redisson官方也给到一些[样例](https://github.com/redisson/redisson/wiki/9.-distributed-services/#96-redisearch-service)： 包括普通查询、对象查询、索引查询和语法校验

- [完整官方说明文档](https://redis.io/docs/latest/develop/interact/search-and-query/)

## The end 

虽然这个组件目前迭代更新是很快，但是在国内实践并不算多，毕竟Elasticsearch的选型，多年已经在人们的内心根深蒂固，如果不是RediSearch足够好或者外部动力催生，还是比较难铺开市场的。企业对于选型Elasticsearch 也不是单纯只做一个搜索，还结合日志、技术栈、数据库等多方面的考虑。

希望这个组件能够继续持续迸发活力，我们也有一天能选型用上这样的功能。