---
layout:     post
title:      Docker部署Apache Ignite
subtitle:   Ignite & Docker
date:       2021-04-02
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Apache Ignite
    - Docker
    - 单点+集群
---

## 使用Docker如何开启Ignite体验

[这边仅使用Docker进行搭建，其余请查看官网](https://ignite.apache.org/docs/latest/installation/installing-using-docker)

### 获取镜像

```bash
sudo docker search apacheignite/ignite
```

### 部署单点

```bash
sudo docker pull apacheignite/ignite

##获取一个指定版本
sudo docker pull apacheignite/ignite:{ignite-version}
```

默认情况，docker将暴露以下端口：11211, 47100, 47500, 49112

#### 形式一：In Memory

 借用容器的文件系统，但是如果没有持久卷支持，容器一旦删除，数据也将丢失

```bash
docker run -d -p 10800:10800 apacheignite/ignite

sudo docker run -it --net=host \
-e "CONFIG_URI=$CONFIG_URI" \  ---Ignite配置文件地址
-e "OPTION_LIBS=$OPTION_LIBS" \ --- Ignite可选的依赖包
-e "JVM_OPTS=$JVM_OPTS" \ ---虚拟机参数
apacheignite/ignite
```

#### 形式二：Persistent

挂载一个持久卷或者一个本地目录

##### 1.通过持久卷持久化数据

1. 创建一个持久卷：sudo docker volume create persistence-volume
2. 通过IGNITE_WORK_DIR系统属性指定，或通过节点配置文件指定

```bash
docker run -d -p 10800:10800 -v persistence-volume:/persistence -e IGNITE_WORK_DIR=/persistence apacheignite/ignite
```

##### 2.通过本地目录持久化数据

1. 准备一个本地目录： ~/docker/ignite/data
2. cd  ~/docker/ignite

```bash
docker run -d -p 10800:10800 -v ${PWD}/data:/persistence -e IGNITE_WORK_DIR=/persistence apacheignite/ignite
```

### 使用简单Java代码进行访问

```java
public class IgniteConnectionTest {

    public static void main(String[] args){
        ClientConfiguration cfg = new ClientConfiguration().setAddresses("127.0.0.1:10800");
        try (IgniteClient client = Ignition.startClient(cfg)) {
            //=========================Get data from the cache=========================//
            ClientCache<Integer, String> cache = client.createCache("myCache1");

            Map<Integer, String> data = new HashMap<>();
            for (int i = 1; i <= 100; i++) {
                Integer integer = i;
                if (data.put(integer, integer.toString()) != null) {
                    throw new IllegalStateException("Duplicate key");
                }
            }

            cache.putAll(data);

            assert !cache.replace(1, "2", "3");
            assert "1".equals(cache.get(1));
            assert cache.replace(1, "1", "3");
            assert "3".equals(cache.get(1));

            cache.put(101, "101");

            cache.removeAll(data.keySet());
            assert cache.size() == 1;
            assert "101".equals(cache.get(101));

            cache.removeAll();
            assert 0 == cache.size();


            ClientCache<Integer, Person> personCache = client.getOrCreateCache("persons");

            Query<Cache.Entry<Integer, Person>> qry = new ScanQuery<Integer, Person>(
                    (i, p) -> p.getFirstName().contains("Joe"));

            try (QueryCursor<Cache.Entry<Integer, Person>> cur = personCache.query(qry)) {
                for (Cache.Entry<Integer, Person> entry : cur) {
                    // Process the entry ...
                    Integer key = entry.getKey();
                    Person value = entry.getValue();
                    System.out.println(key);
                    System.out.println(value);
                }
            }
            //=============Working with Binary Objects================//
            IgniteBinary binary = client.binary();
            BinaryObject val = binary.builder("Person").setField("id", 1, int.class).setField("name", "Joe", String.class)
                    .build();

            ClientCache<Integer, BinaryObject> cache = client.getOrCreateCache("persons").withKeepBinary();

            cache.put(1, val);

            BinaryObject value = cache.get(1);
            System.out.println(value);

            ClientTransactions tx = client.transactions();
            try (ClientTransaction t = tx.txStart(TransactionConcurrency.OPTIMISTIC, TransactionIsolation.REPEATABLE_READ)) {
                ClientCache<Integer, String> cache = client.getOrCreateCache("transCache");
                cache.put(1, "new value");
                t.commit();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

[更多样例代码 with 2.10.0](https://github.com/CzyerChen/ignite-gridgain-examples)

### 使用命令行+UI工具进行访问

[具体可以参看官方文档的Tools](https://ignite.apache.org/docs/latest/tools/control-script)

- control.sh命令
- ignitevisorcmd命令
- gridgain

### ignite集群

Ignite 中，通过DiscoverySpi节点可以彼此发现对方，可以配置成基于多播的或者基于静态 IP 的。
在此还没有太了解，正在继续学习，[更多说明可以看InfoQ](https://www.infoq.cn/article/apache-ignite-part05)