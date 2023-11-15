---
layout:     post
title:      K8S学习笔记|04-Controller类型
subtitle:   Kubernetes
date:       2023-08-31
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
    - K8S
    - Deployment
    - ReplicaSet
    - StatefulSet
    - DaemonSet
    - Job
---

- [如何创建一个Deployment？](#如何创建一个deployment)
  - [命令创建](#命令创建)
  - [配置文件创建](#配置文件创建)
  - [其他指令](#其他指令)
- [ReplicaSet](#replicaset)
- [Pod](#pod)
- [Deployment 部署示例](#deployment-部署示例)
  - [YAML文件配置格式](#yaml文件配置格式)
- [DaemonSet](#daemonset)
- [StatefulSet](#statefulset)
- [Job](#job)

## 如何创建一个Deployment？

### 命令创建

`kubectl run` 数清晰直观、使用便捷，可用于测试

### 配置文件创建

`kubectl apply`，配置文件的形式，提供了创建资源的模板，有利于重复使用，可以形成版本，同代码一起进行管理，适合生产的交付流程

### 其他指令

```bash
kubectl get deployment <name>

kubectl describe deployment <name>
```

## ReplicaSet

创建了Deployment后就会隐式创建一个ReplicaSet
可以通过` kubectl describe replicaset `来查看

## Pod

查看的指令

```bash
kubectl get pod 
kubectl describe pod
```

Pod怎么诞生？

```bash
kubectl 创建Deploymnet
Deploymnet 创建ReplicaSet
ReplicaSet 创建Pod
```

## Deployment 部署示例

```bash
kubectl apply
kubectl create
kubectl replace
kubectl edit
kubectl patch
```

### YAML文件配置格式

Deployments 比较典型的就是通过 `kubectl create ` 或 ` kubectl apply` 来创建和管理
创建一个Deployment 可以通过定义YAML文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: mcr.microsoft.com/oss/nginx/nginx:1.15.2-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 256Mi
```

|声明|描述|
|--|--|
|.apiVersion|指定API组和API资源|
|.kind|指定资源类型|
|.metadata.name|定义deployment的名字|
|.spec.replicas|指定Pod的数量|
|.spec.selector|指定影响的Pods|
|.spec.selector.matchLabels|通过相同键值对定义的容器将被deployment管理|
|.spec.template.labels|指定的键值对绑定到这个对象|
|.spec.template.app|与【.spec.selector.matchLabels】对应|
|.spec.spec.containers|指定归属于这个Pod的一系列containers|
|.spec.spec.containers.name|指定容器的名称|
|.spec.spec.containers.image|指定容器镜像|
|.spec.spec.containers.ports|指定container暴露的端口|
|.spec.spec.containers.ports.containerPort|指定Pod要暴露的IP地址的端口，用于内网交互|
|.spec.spec.resources|指定container所需的计算资源|
|.spec.spec.resources.requests|指定所需的最小资源|
|.spec.spec.resources.requests.cpu|指定所需的最小CPU|
|.spec.spec.resources.requests.memory|指定所需的最小内存|
|.spec.spec.resources.limits|指定所需的最大资源|
|.spec.spec.resources.limits.cpu|指定所需的最大CPU|
|.spec.spec.resources.limits.memory|指定所需的最大内存|

以上参数是基本的定义YAML文件的参数，其中最大最小资源是建议指定的，以保障整体资源的可控性

可以通过yaml配置，轻松控制副本数

可以使用label控制Pod的位置，希望将指定Pod运行到指定Node上，使用nodeSelector通过label指定要挂载的node
对node节点添加label，`kubectl label node k8snode disktype=ssd`
对node节点删除label，`kubectl label node k8snode disktype-`

## DaemonSet

DaemonSet 不同于普通Deployment，它在每个Node上只能最多运行一个副本，应用场景有

1. 数据存储，例如ceph/glusterd
2. 日志收集，例如flunentd/logstash
3. 业务监控，例如Prometheus Node Exporter/collectd

k8s自身的DaemonSet服务类型主要有 `kube-proxy`，`kube-flannel-ds`

kind是DaemonSet
尝试使用prometheus node exporter来运行一个daemonset容器

## StatefulSet

容器分为服务类容器和工作类容器
服务类容器需要一直运行，Deployment\ReplicaSet\DaemonSet用于管理服务类容器

## Job

用于管理工作类容器，用完就可以销毁

Kind: Job、CronJob
restartPolicy: Never/OnFailure

`kubectl get job`

DESIRED 和 SUCCESSFUL 不符合时，

- restartPolicy: Never，job会不断新建来达到DESIRED的要求
- restartPolicy: OnFailure，job会不断重启来达到DESIRED的要求

可以通过parallelism 设置Job的并行度

CronJob 的schedule配置cron表达式

K8S默认不开启CronJob功能，可以通过kube-apiserver的配置文件打开 --runtime-config=batch/v2alphha1=true

重启kubelet.service服务来重启 kube-apiserver
