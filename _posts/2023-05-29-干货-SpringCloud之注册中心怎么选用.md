---
layout:     post
title:      SpringCloud怎么选注册中心
subtitle:   SpringCloud
date:       2023-05-30
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringCloud
---

SpringCloud的框架并不陌生了，在业内微服务领域的扛把子。今天来看一看如何根据业务需要，来选择合适的注册中心？

注册中心是微服务管理节点通信、核心配置的关键组件，从分布式多节点的前提下最主要要解决是就是分布式下的一致性问题，不同业务场景有不同AP、CP模型的需求。

## 1、注册中心背景知识-CAP理论

CAP是对分布式系统的三个重要衡量指标：

- C - Consistency一致性：在多个分布式节点上，读取同一个key内容都是一样的。
  - 如果在A节点设置key为value1，A节点发生故障，切换另一个节点B查询应该查到key为value1，如果存在值不一致或不存在，则有一致性问题。
  - 当服务存在数据不一致问题，意味着如果外部查询，可能响应数据是旧版本，可能响应数据是新版本，或者说并发查询响应的内容都不同。
  - 那这个时候系统应该采取不返回任何数据，来避免数据的不一致。等恢复后，再提供准确数据。
- A - Availability-可用性：在多个分布式节点上，部分节点故障，或者部分节点间通信中断，在业务容许的响应时间内，能够正常提供服务，则服务具有较高的可用性。
  - 可以通过服务集群快速剔除故障节点，依旧对外正常提供服务。
  - 或者正常与故障节点均能对外提供服务，但正常与故障节点间不能通信，所以可能出现一致性问题。
  - 或者节点能正常接收请求，但是发现节点内部数据有问题，就是存在数据一致性，也要对外提供服务。
- P - Partition tolerance-分区容错性：分布式系统往往具备多个节点。如果多个节点突然其中几个节点间无法通信了，依旧能对外提供服务，则具备分区容错性。
  - 分布式系统中，节点间因网络、机器等原因出现通信中断问题，则称分布式系统出现了分区。
  - 此时即使出现了分区故障，系统还能正常运行，各服务节点还能对外提供服务。

在分布式存储中，CAP理论是肯定不能都满足，定然有所取舍，所以出现了AP/CP的选择。

从反复理解研读上面C\A两个理论，可以很明显地看出CP、AP系统的区别了：

- CP系统：要求数据的一致性，节点一旦出现故障，数据一致前，客户端请求都卡死或拒绝，直到节点能够提供一致的数据。
	- 典型的组件有Zookeeper。
	- 通常金融系统会要求数据强一致性。
- AP系统：要求服务的可用性，节点即使出现故障，服务依旧能对外提供服务，但是可能提供的数据不一致。
	- 典型的组件有Eureka。
	- 通常部分互联网系统接受最终一致性，而选择服务的可用性。

在一个完整的系统中，也要根据使用场景，来选择AP或CP，看是更注重一致性还是更关心可用性。

## 2、注册中心的选型分类

常见的CP注册中心有：Zookeeper\Consul\Nacos；
常见的AP注册中心有：Nacos\Eureka；

- Zookeeper：经常是Zookeeper + Doubbo这个组合；
- Consul：是一种服务网格解决方案，提供具有服务发现、配置和分段功能的全功能控制平面；
- Nacos：兼容AP\CP两种类型，更敏捷和容易地构建、交付和管理微服务平台；
- Eureka：Spring Cloud默认的注册中心，每个节点都可被视为其他节点的副本，从而对外提供较高的可用性；


既然Nacos既有AP版本又有CP版本，那我们就从SpringCloud+Nacos了解一下注册中心的流程吧。

为什么Nacos配置中心要使用CP模型？

答：CP模型牺牲了一定的延时去换取数据的强一致，但应用的外部化配置并不要求配置成功的延迟要多低，更多的是要保证配置信息的一致性（我配置什么信息A=1，不要给我弄丢了或者等下查 A 不等于 1，这些都是数据不一致，这肯定是不行的），所以在这种配置场景下是十分适合做CP模型的
为什么Nacos注册中心要使用AP模型？

答：这个问题也可以转换成为什么注册中心要使用AP模型。因为可用性比一致性更加重要，可用性体现在两个方面，第一是容许集群宕机数量，AP模型中集群全部机器宕机才会造成不可用，CP模型只容许一半的机器宕机。第二是分区是否可写，例如A机房和B机房发生分区，无法进行网络调用，在CP模型下部署在A机房的服务就无法部署，因为A机房无法与B机房通讯，所以无法写入IP地址，在AP模型下，可以去中心化，就像Eureka那样就算发生分区每台机器还都能写入，集群间无Master，而是互相复制各自的服务IP信息，这样，在分区时依然是支持写入的。不可用会造成微服务间调用拿不到IP地址，进而业务崩盘，所以不可用是不可接受的，不一致会造成不同微服务拿到的IP地址不一样，会造成负载不均衡，这是可以接受的。

## 3、Nacos注册中心

Nacos主要提供以下四大功能：

1. 服务发现和服务健康监测
2. 动态配置服务
3. 动态DNS服务
4. 服务及其元数据管理
  
Nacos有Server和Client：

- 服务端Server 提供服务注册、元数据存储。
	- 会将服务信息按照Namespace -> Group -> Service -> Cluster -> Instance 的映射关系来存储。
	- 服务端会对注册上来的客户端定期做心跳检测，以确定服务的健康状态。
	- 会对注册上来变更的服务提供者数据进行刷新和广播。
- 客户端Client 有分为Service Provider Client,Service Consumer Client
	- 服务提供者，将自己的访问地址注册到注册中心，定时任务发送心跳进行续约，自身如果有IP等变化主动向注册中心同步。
	- 服务消费者，有两种方式来实现服务的订阅，一种是主动向注册中心拉取服务列表，一种是向注册中心提交订阅任务定期处理消息。
		- 主动向注册中心拉取：就是使用openAPI或者SDK的方式，直接调用指定实例列表的查询，来获取具体实例信息
		- 提交订阅任务：通过 EventListener处理不同类型的实现，来动态调整对服务的调用

Nacos是在通过客户端进行服务发现的时候是默认开启订阅拉取的。

默认参数subcribe就是true，所以会使用getServiceInfo方法去获取到最新的服务实例，如果subcribe为false，就使用getServiceInfoDirectlyFromServer方法每次都会请求服务器获取最新的服务实例。

```java
public List<Instance> selectInstances(String serviceName, String groupName, List<String> clusters, boolean healthy,
        boolean subscribe) throws NacosException {

    ServiceInfo serviceInfo;
    String clusterString = StringUtils.join(clusters, ",");
    // 是否为订阅模式
    if (subscribe) {
        // 先从本地缓存获取服务信息
        serviceInfo = serviceInfoHolder.getServiceInfo(serviceName, groupName, clusterString);
        // 如果本地缓存不存在服务信息，则进行订阅
        if (null == serviceInfo) {
            serviceInfo = clientProxy.subscribe(serviceName, groupName, clusterString);
        }
    } else {
        // 如果未订阅服务信息，则直接从服务器进行查询
        serviceInfo = clientProxy.queryInstancesOfService(serviceName, groupName, clusterString, 0, false);
    }
    // 从服务信息中获取实例列表
    return selectInstances(serviceInfo, healthy);
}
```

关于Nacos的部署与基础使用在官网都有介绍与示例，可以跟随官网进行简单的Nacos服务端单节点部署，书写SpringCloud+Nacos的demo将自身服务进行注册，并且简单查询服务列表，来初步了解一下Nacos的功能。（官方文档：[https://nacos.io/zh-cn/docs/quick-start-spring-cloud.html](https://nacos.io/zh-cn/docs/quick-start-spring-cloud.html)）

目前Nacos在国内SpringCloud/SpringBoot微服务架构中也是受到较大欢迎，被广泛使用。

- 服务的高可用 
- 部署简易轻量
- 配置简单成本低
- 社区开放活跃
- 支持跨语言服务的开发与使用

还不赶快来试一试？