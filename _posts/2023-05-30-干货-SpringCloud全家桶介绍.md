---
layout:     post
title:      SpringCloud全家桶初识
subtitle:   SpringCloud
date:       2023-05-30
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringCloud
---

- SpringCloud的从整体架构上看，相对来说是完整的、庞大的。
- 它不仅仅是一个基础性架构工具，它为微服务架构提供了一个“全家桶”的套餐。
- 每一个模块关注各自的职能，并且能够很好地配合与协作，能够帮助入门者快速搭建起一套微服务架构的服务。

内容快速一览

- [什么是微服务？Microservices](#什么是微服务microservices)
- [如何实施微服务？](#如何实施微服务)
- [微服务界有哪些组件可选择？](#微服务界有哪些组件可选择)
- [SpringCloud包含哪些功能模块？](#springcloud包含哪些功能模块)

## 什么是微服务？Microservices

- 将原本一个独立的系统，拆分成若干小服务
- 小服务之间通过REST API进行交互通信
- 每个小服务各自部署、各自维护、独立扩展
- 易于控制整体服务，降低整体耦合性，各部分根据各自需求开发与上线
- “微”也不是拆的越碎越好，也要考虑自身服务的完整性、服务部署与维护的复杂性等

## 如何实施微服务？

- 微服务表明存在多个独立的小服务，但这么多独立的服务需要能够互相通信实现调用，保持整体的可用性
- 将服务进行组件化。原本使用方法直接调用的，需要转化为REST API的方式进行调用
- CI/CD流程自动化。更多的服务，代表更大的运维复杂度，也代表更大的服务风险，所以需要自动化的CI/CD流程来降低运维复杂度

## 微服务界有哪些组件可选择？

虽说SpringCloud已经提供了一整套实现微服务的流程，但是民间也有不少因不能满足某些场景而自行实现或优化的微服务组件，所以也并不是一定需要选用全家桶内的组件，也要聚焦于当前应用的场景。

- 服务治理：
  - Dubbo(Alibaba)
  - DubboX(dangdang)
  - Eureka(Netflix)
  - Consul(Apache)
- 分布式配置管理：
  - Config(SpringCloud)
  - Nacos(alibaba)
  - Disconf(baidu)
  - QConf(360)
  - Diamond(taobao)
  - Zookeeper(Apache)
- 批量任务：
  - Task(SpringCloud)
  - Azkaban(LinkedIn)
  - Elastic-Job(dangdang)
  - XXL-Job(个人)
  - DolphinScheduler(Apache)
- 服务追踪：
  - Sleuth(SpringCloud)
  - Zipkin(Twitter)
  - Skywalking(Apache)

除了SpringCloud原生组件，民间也是涌现了不少基于不同场景进行优化的各类组件。

## SpringCloud包含哪些功能模块？

- SpringCloud是基于SpringBoot实现的微服务架构，能够保留一切SpringBoot可以带给你的丝滑和便利，
- 包含有配置管理、服务治理、断路器、智能路由、为代理、控制总线、全局锁、决策精选、分布式会话、集群状态管理等

官方全家桶所包含的子组件有：

- Spring Cloud Confg: 配置管理工具，可以配置的外部化存储，客户端刷新、加密/解密等。
- Spring Cloud Netflix: 核心组件。
  - Eureka: 服务治理组件，包含服务注册中心、服务注册与发现
  - Hystrix: 容错管理组件，实现断路器模式。
  - Ribbon: 客户端负载均衡的服务调用组件
  - Feign: 基于 Ribbon 和 Hystrix 的声明式服务调用组件。
  - Zuul: 网关组件，实现智能路由、访问过滤。
  - Archaius:外部化配置组件。
- Spring Cloud Bus: 事件、消息总线。
- Spring Cloud Cluster: 针对 ZooKeeper、Redis、Hazelcast、Consul 的选举算法和通用状态模式的实现。
- Spring Cloud Cloudfoundry: 服务发现与配置管理工具。
- Spring Cloud Stream: 通过 Redis、Rabbit 或者 Kafka 实现的消费微服务。
- Spring Cloud Sleuth: 整合 Zipkin，实现较完整的链路追踪。
- Spring Cloud ZooKeeper: 基于 ZooKeeper 的服务发现与配置管理组件。
- Spring Cloud Starters: Spring Cloud 的基础组件，基础依赖模块。
- Spring Cloud CLI: CLI插件。