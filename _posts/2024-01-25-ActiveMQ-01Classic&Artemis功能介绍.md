---
layout:     post
title:      ActiveMQ-Classic&Artemis功能介绍
subtitle:   ActiveMQ-Classic/ActiveMQ-Artemis
date:       2024-01-25
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - ActiveMQ-Classic
    - ActiveMQ-Artemis
---

接上篇，ActiveMQ就是基于JMS消息服务规范的消息中间件组件，主要应用在分布式系统架构中，帮助构建高可用、 高性能、可伸缩的企业级面向消息服务的系统

- [JMS对象模型](#jms对象模型)
- [ActiveMQ的功能](#activemq的功能)
  - [产品优势](#产品优势)
    - [产品成熟且稳定](#产品成熟且稳定)
    - [协议支持广泛](#协议支持广泛)
    - [企业级特性](#企业级特性)
  - [产品的不足](#产品的不足)
- [Classic版本和Artemis版本有什么区别](#classic版本和artemis版本有什么区别)

## JMS对象模型

ConnectionFactory： 连接工厂，客户端他通过JNDO查找，通过工厂获取一个JMS连接

Connection： 表示客户端和服务端之间的一个连接

Session：表示客户端和服务端之间会话状态，会话建立在连接之上

Destination： 消息队列，有通信地址、类型等信息

Message Producer： 消息的生产者

Message Consumer： 消息的消费者

常用的两种通信模式： P2P点对点的队列模式，Publish/SubscribePublish/Subscribe发布订阅的广播模式

## ActiveMQ的功能

ActiveMQ作为消息队列的前驱探索者，虽然在历史的演进中前浪已经逐渐被后浪取代，但是它也始终在变化求存，依托Java语言门槛低占有一定份额。

### 产品优势

#### 产品成熟且稳定

作为先驱者，在历史的长河中经受住了实战的考验，曾经也在大量头部企业中投产，并发挥生产的作用。对于基本的消息队列功能有保障。

#### 协议支持广泛

支持多种消息协议，包括JMS（Java Message Service）、AMQP 1.0、STOMP、OpenWire等，提供了跨语言和平台的兼容性

#### 企业级特性

提供事务支持、持久化消息存储、集群部署、高可用性和安全性等功能，适用于企业级应用集成场景

### 产品的不足

通过消息队列在企业中不断使用，高并发、高性能、大数据量、多租户等业务场景的不断涌现，特定的产品解决特定的业务问题，ActiveMQ并没有一味追新，在业务功能特性上逐渐不突出

处理小消息和高并发场景时，对硬件资源要求相对较高

分布式环境下的扩展能力较弱

Apache社区的重点逐渐转向了ActiveMQ Artemis（基于不同的架构），而对ActiveMQ 5.x版本的维护越来越少

因此，在传统的服务架构对性能要求不高的情况下，可以使用ActiveMQ快速接入，但是对于大数据、高并发微服务的环境下，新生的RocketMQ/Kafka更能适应业务需求。

## Classic版本和Artemis版本有什么区别

大家更为熟悉的ActiveMQ是classic的版本，也就是经典版本，市场占有稳定的ActiveMQ5.x

2015年从HornetQ项目发展而来合并入ActiveMQ的新基线Artemis就不是我们广泛知晓的版本，就是为了摆脱传统架构，增加对高并发、大数据、微服务方向是适配，Artemis采用了全新的设计和实现，旨在提高性能、可扩展性和可靠性。

Artemis版本作为Classic版本的替代品而诞生，相较的提升点有：

- 高性能与低延迟：Artemis通过改进的设计实现了更高的吞吐量和更低的消息处理延迟。

- 存储机制：使用了不同的持久化策略和日志结构，比如Journal文件系统，以获得更好的写性能和恢复速度。

- 内存管理：内存使用效率更高，尤其是在处理大量小消息时表现更好。

- 集群和HA：提供了更先进的高可用性解决方案和更灵活的集群模式。

- 协议支持：除了原有的JMS之外，对AMQP 1.0的支持更加成熟和完善，并且也支持STOMP等多种协议。

- 架构更新：整体架构更为现代化，为云原生环境和大规模分布式部署进行了优化。

还是比较期待Artemis的发展的，毕竟在消息队列的演化进程中，它入场很早，一直在坚持开源，也服务于诸多业务，在大数据、云原生、AI时代下还在优化升级，也是希望自己继续发光发热
