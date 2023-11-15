---
layout:     post
title:      K8S学习笔记|15-K8S杂谈
subtitle:   Kubernetes
date:       2023-09-07
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
    - K8S
---

## 云计算

云计算作为一个新兴领域，从狭义上，是指IT基础设施的佳偶和使用模式，从广义上是指服务的交付和使用模式

云计算提供服务资源的类型分为三类：

### 1.基础设施即服务

infrastructure-as-a-service IAAS，面向网络工程师

通过虚拟化与分布式存储技术，对服务器、存储、网络等物理资源进行抽象，形成可扩展、可按需分配的虚拟资源池。例如虚拟机、磁盘以及主机互联的网络，Amazon AWS 提供了虚拟机EC2和云存储等

### 2.平台即服务

Platform-as-a-service PAAS，面向应用开发者

自动化应用的部署与运维，是开发者集中精力于业务开发，极大提升开发效率，K8S可以说在PAAS的定义范畴中

- 第一代PAAS 主要为GAE SAE，当时没有Paas的概念
- 第二代PAAS，包括Cloud Foundry\华为云\dell云服务等
- 第三代PAAS就到了Docker的时代，K8S\ServiceMesh\Istio等概念应运而生，成为PAAS的主力，Docker是一种Linux容器工具集，包含构建、交付、运行

### 3.软件即服务

software-as-a-service SAAS，面向终端用户

SAAS将软件服务以特定接口的形式进行发布，终端用户通过浏览器就可以使用软件，在线使用的邮箱系统和各种管理系统可以认为是SAAS范畴
