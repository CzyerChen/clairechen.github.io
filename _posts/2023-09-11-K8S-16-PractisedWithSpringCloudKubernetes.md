---
layout:     post
title:      SpringCloudKubernetes-01 简介
subtitle:   K8S
date:       2023-09-11
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringCloud
    - Kubernetes
---

SpringCloud的项目，自从与K8S生态结合之后，关于服务负载均衡、服务发现、配置中心就有了更多的选择，可以将一些功能从应用级别转化为系统级别

SpringCloudKubernetes项目就是对SpringCloud+Kubernetes的组合进行包装，使功能的切换更为丝滑

通常我们使用配置中心有几个要求：

- 将多个项目、多个环境的配置进行隔离，通过分组单独维护
- 将配置与敏感项抽离，保障配置的集中性与安全性
- 配置支持动态修改，在不中断服务的情况下可以动态修改配置信息，达到无缝切换
- 额外还有更多配置中心的使用场景...

以下分别从上面三个基础使用场景，结合SpringCloudKubernetes\k8s-ConfigMap 看看如何实现

## SpringCloudKubernetes\k8s-ConfigMap

通过SpringCloudKubernetes的相关组件，可以在原有SprirngCloud的基础上，结合K8S相应功能的组件，来替换原生的SpringCloud生态，使微服务融入云原生环境

SpringCloud对配置的实现是通过配置中心，比如基础的ConfigServer或者是额外的nacos-server
