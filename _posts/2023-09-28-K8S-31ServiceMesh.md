---
layout:     post
title:      Kubernetes|ServiceMesh
subtitle:   Git
date:       2023-09-28
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Git
    - GitOps
---

## 什么是ServiceMesh?

**基础授权**
通过声明式的方式定义服务之间的通信

**服务发布分流**
测试版本和生产版本，也就是金丝雀的想法
边车模式可以很好地做99%生产1%测试的流量转换，不需要复杂的配置就可以实现

**便于收集大量生产指标**
服务网格边车可以灵活的获取指标的，可以将指标添加到Prometheus等开源项目中

**服务网格接口Service Mesh Interface SMI**
是开发较多的部分，是Kubernetes 注册的API集合
这些API接口可以做很多实现，Consul\Istio\LinkedIn ...
