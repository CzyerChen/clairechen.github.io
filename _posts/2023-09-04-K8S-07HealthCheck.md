---
layout:     post
title:      K8S学习笔记|07- Liveness & Readiness 探测
subtitle:   Kubernetes
date:       2023-09-04
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
    - K8S
    - Liveness & Readiness
    - 服务探测
---

- [Liveness探测](#liveness探测)
- [Readiness探测](#readiness探测)
- [Liveness和Readiness](#liveness和readiness)
- [HealthCheck的功能](#healthcheck的功能)

关于健康检测这边，会具体讲述K8S自身如何对Pod进行健康存活检测，如何可以对应用进行存活和健康检测。让应用在K8S上实现完美的无缝切换，实现更为安全、零停服的版本滚动升级

K8S默认的健康检测，主要监测Pod启动的进程，进程退出返回码非0就是代表故障，需要根据重启策略执行
restartPolicy： 默认Always，可选OnFailure

但是经常应用程序的故障场景是，比如内存溢出，系统进程还在但无法对外提供服务了，这种情况K8S的默认规则就无法识别，需要使用
Liveness探测

## Liveness探测

使用livenessProbe指定自定义健康的条件:

- initialDelaySeconds是程序初始化后多久开始做Liveness探测
- periodSeconds 探测检测周期

## Readiness探测

- Liveness是告诉容器什么情况pod需要通过重启后自愈
- Readiness是告诉容器什么情况下应用已经启动完成具备对外提供服务的能力
- 使用ReadinessProbe指定自定义健康的条件
  - initialDelaySeconds是程序初始化后多久开始做Readiness探测
  - periodSeconds 探测检测周期
- Pod的状态会从not ready到ready，ready后就代表可以对外提供服务

## Liveness和Readiness

采用一致的配置格式，可以使用exec:command 或者httpGet 进行配置，httpGet返回值在200~400之间是正常的，如果非200~400就是非就绪，连续3次失败就会将服务从负载均衡中移除，无法再访问，直到下次成功再加入

```yml
readinessProbe:
  httpGet:
    schema: HTTP
    path: /health
    port:8080
  initialDelaySeconds: 10
  periodSeconds: 5  
```

springboot 2.3.X 自动兼容K8S的Liveness & Readiness探测

## HealthCheck的功能

在**ScaleUp**中，当一个Pod符合Readiness验证，才加入负载均衡中对外提供服务，避免无效的请求

在**RollingUpdate**中，可以避免服务更新期间应用尚未启动加载完全就认为Pod启动完成，持续替换，导致可能最终无法对外提供服务。Redisness探测可以检测到应用真实能提供服务后，再向后逐步替换，达到真正的不停服。异常的副本无法加入负载均衡，不会对外提供服务。

**maxSurge**:
参数控制滚动更新过程中副本总数超过DESIRED的上限
因为DESIRED是目标可用值，滚动更新中最终可用的Pod数，在感动更新的过程中，会新增并行更新的Pod数。向上取整，最大默认是Desired数量的*25%，控制初始创建的新副本数
资源充足可以配置多一些，那整体资源占用会多，但更新流程变快

**maxUnavailable:**
参数控制滚动更新过程中不可用副本占DESIRED的最大比例，向下取整，最大默认是Desired*25%，控制初始销毁的旧副本数

```yml
strategy:
  rollingUpdate:
    maxSurge: 35%
    maxUnavailable: 35%
```
