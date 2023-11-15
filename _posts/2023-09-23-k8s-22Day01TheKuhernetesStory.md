---
layout:     post
title:      Kubernetes-从一个故事来了解
subtitle:   SpringBoot
date:       2023-09-21
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringBoot
---

## Day01 从一个故事进入K8S世界

下面的故事将围绕Phippy和他的朋友们，这个概念来自Cloud Native Computing Foundation. 他们希望以通俗易懂的故事情节，来解开云原生计算的神秘面纱。[原文介绍](https://www.cncf.io/blog/2021/10/19/phippy-and-friends-bringing-cloud-native-concepts-to-the-masses/#:~:text=Today%2C%20Phippy%20and%20Friends%E2%80%99%20mission%20is%20to%20demystify,manner%2C%20through%20books%2C%20presentations%20and%20even%20Twitter%20threads.)

## Phippy and Friends

是什么？为什么诞生？有什么用？`Phippy goes to zoo`

她的侄女Zee无精打采地躺在沙发上，Phippy就带她去了动物园

### 【Pods】

首先看到的是Pods，Phippy把他们形容为松鼠大小、毛茸茸的、蓝色的小动物，每人背着一个小盒子
是K8S中运行容器最基本的单元，能够设置环境变量、挂载存储、将其他信息传入容器，一个Pod至少包含一个container容器，能够控制容器的运行。container容器一旦退出了，pod也就销毁了

### 【ReplicaSets】

往前走，看到玻璃窗内的ReplicaSets，一排满脸笑容的小猫鼬
每次一个副本消失了，另一个就立刻会出现
在K8S中，ReplicaSets是一个低优先级类型，k8s的用户通常操作高优先级的组件类似Deployments 和 DaemonSets
一个ReplicaSet保障一组相同配置的Pod能够按照指定副本数运行，如果一个Pod下线了，ReplicaSet就会立刻新建一个新的来填补

### 【Secrets】

继续向前，Zee指向一堆洞穴和巢穴。种种迹象表明，此处是有人居住的，但是他们并没有看到任何动静。Phippy 说需要带上特殊眼睛才能看到，Zee带上眼镜后，能够看到了，他们继续向前。

这边要讲的是Secrets, secrets需要使用base64加密配置，在获取的时候能够自动解密，secrets可以通过文件或者环境变量的方式进行配置，通常用于存储非公开的一些信息，比如token\账密\证书等，通过加密的形式安全的存储在集群中，Pods可以在运行时获取这些配置的敏感数据。

### 【Deployments】

一群鬣蜥聚集在池塘岸边的大弹弓附近，一个小岛屿在一个水池中央，一个鬣蜥将它自己放在弹弓上，另一个人将它发射到小岛上。这些Deployments 将它们自己发射到小岛上，如果发射失败他们就继续尝试直到小岛上数量满足需求。

Deployments支持滚动更新和回滚，滚动的动作可以被暂停。
Deployment是一种高阶抽象，能够控制部署和管理一组Pods。背后依靠ReplicaSet来使这些Pod运行，他主要是提供复杂的部署逻辑，包括更新、扩缩容

### 【DaemonSets】

几根石柱从草地上拔地而起，每个石柱顶端都栖息着一只秃鹫。当Zee和Phippy在看的时候，一只秃鹫拍打着翅膀飞向远方，没多久另一只就取代了他的位置。他们就是DaemonSets，他们确保占领每一根石柱，风雨无阻，日夜守护。
DaemonSets有很多用法，一个在每个Node上都要频繁要使用的设计就可以使用Daemonset来安装和配置。主要提供一个能够确保Pod的副本能够在集群的每一个Node上都运行，当集群扩展或是收缩的时候，这些特殊标签的Pod就会分布在所有的node上

### 【Ingress】

当他们继续往前走，他们看到了一个巨大的礁石，小鱼从岩石的一侧到另一侧，快速穿越。这就是Ingress
从集群中将流量路由，为多个应用提供一个单一的SSL端点，ingress的多种不同的实现可以让你定制化你的平台。

### 【CronJob】

Zee指了指围栏内一动不动的浣熊，突然跳起来，又坐下来小憩。Phippy说这些就是CronJob，大部分时间他们都是睡觉，但是周期性地，他们会做一些特殊的任务。就说话的间隙，又有一只站起来，抓起扫帚扫了扫地，然后又睡下了。

使用简单的cron语法来调度任务，cronJobs是batchApi的一部分，用于创建短期服务。Cronjobs 提供了调度Pod的方法，对于运行像备份、报告、自动化测试这种周期性的任务很不错。

### 【CRD】

Zee 看到在远处，有一个黑色的围栏，上面标记着CRD的字样，在围栏里面有一些奇特的动物：长着河马头的长颈鹿。一条有浣熊耳朵的蛇。一只长着海狸尾巴的狮子。没有角的独角兽。Phippy突然发现，是午饭时间了，我们要回家了。Zee送了一口气，提出要在出去的路上在Kube的店进去看一下。

CRD定义一种新的资源类型，并同步给Kubernetes. 一旦有新的资源类型增加了，这个资源的实例就被会创建。如何处理CRD的变化是取决于你的。通常，会创建一个本地的controller，用来监控新的CRD实例，并做出相应的响应。CRD=CustomeResourceDefinitions,提供集群操作员和开发人员可以使用 创建自己的资源类型。
