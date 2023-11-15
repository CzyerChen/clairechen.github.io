---
layout:     post
title:      Kubernetes|MS学习02-K8S基础认知
subtitle:   Kubernetes
date:       2023-10-18
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
---

- [从几个方面了解Kubernetes](#从几个方面了解kubernetes)
- [Kubernetes 网络模型](#kubernetes-网络模型)
- [Kubernetes 数据存储](#kubernetes-数据存储)

## 从几个方面了解Kubernetes

Why you should care about containers

- Utilize underlying resources more efficiently with this lightweight, immutable infractructure for application deployment

Understand serverless with Kubernetes

- Learn the difference between serverleess Kubernetes and serverless on Kubernetes

Overview of common Kubernetes scenarios

- Use Kubernetes ffor purposes beyond microservice  delivery, such as barch analytics and workflows

How Kubernetes work

- Get to know key components including pods, deployments, replica sets, and the scheduler

How the Kubernetes scheduler works

- See how Scheduler uses predicates or hard constraints(such as nodeSelector, memory) and priorities or soft containers(such as !sick node) to make decisions
- sort(filter(Nodes))

How volumes and storage work in Kubernetes

- find out about emptyDir, persistent volume, and persistent volume claim

How Kubernetes deployments work

- Upgrade applications running in a Kubernetes cluster reliably and with zero downtime
- liveness check
- readiness check

Setting up a Kubernetes build pipeline

- Watch a demonstration of security and associated checks and balances
- admissions con troller / policy controller
- build pipeline has special privileges to push image and deploy pods.

The basics of stateful applications in Kubernetes

- Understand the role that replica sets, persistent volumes, and persistent volume claims play in stateful app development
- stateful applications may expect their hostnames to be constant
- stateful set,replica has indices, declare an initial leader, scale up and down may base on indices
- can use naming to each specific index in stateful set

Understand secrets management in Kubernetes

- Securely deploy and manage sensitive information such as passwords and certificates
- password to datebase, certs, api token
- key-value pairs, or contents of file
- secret volume in the list, ammount the path
- app can load from file or environment variables
- stored in etcd in an unencrypted form
- can be combined with other key vault tools with management so that keys can be safe to be automaticly stored into etcd
- resource-based access control on the Kubernetes API

Getting production ready in Kubernetes

- Put Kubernetes services into production, plus see why CI/CD, monitoring, and failover planning are important
- Cluster API, contents\machines in your cluster, security boundary in you app
- RBAC: developers, operators, CICD pipelines
- monitor solutions like SAS/ ELK stack
- DNS entry,IP address mapping, k8s cluster, health check, CICD

Getting started with monitoring and alerting

- Set up and customize alerts and monitoring for apps, and integrate metrics to operate apps more reliably
- CPU/MEMORY/NETWORK/DISK to metrics server to promoetheus
- service mesh, http latency, error codes
- Prometheus metrics +grafana
- alert based on metrics data, so that can understand what's going on
- when meet common causes, you can build agents or tools automaticly triggered

How Kubernetes and config management work

- Learn management practices and topics like ConfigMaps, rollout of configurations, and templating
- ConfigMap, key-value pairs, properties files
- accmount file as a volume， can be environment variables
- can be roll out
- high level template with little different keys can be changed, you can use helm

How service meshes work in Kubernetes

- Get a quick overview of services meshes, what they provide, and why you would want one in your application
- Load balancing, user & pod ingress in K8S, pod1 & pod2 service mesh
- side cars in mervice mesh
- service authorization, determined the service you allowed to talk to and the service can talk to you
- 1.canary testing, different version with service, divide the traffic
- 2.side car can do proxy
- 3.collect metrics, push to metrics server
- service mesh interface SMI that the collections of API that register themselves with K8S, api isn't tightly bound to any paticular implementations

How pods and the pod lifecycle work in Kubernetes

- Understand what happens when you create a pod—the atomic unit of scheduling
- Pending, k8s accept your pod, ready to go, the scheduler finding the CPU\MEM. If stuck in Pending, means VM are full, no resource left, Pending to waiting
- Creating, pull image down to the node, if pod already exist, the step may be skipped. If stuck in Creating, may be fail to pull images with 404\401\connections refused
- Running, the pod running up successfully
- CrashLoopBackOff, restart too many times
- /health /ready /post start hook/ pre stop hook can be called to know the state of your pod, init container that started successfully and then the other container start up 

Understand role-based access control in Kubernetes

- Ensure that people working on a project don’t interfere with each other’s work by setting up a proper RBAC system
- Cluster role binding, provide you with permissions for the entire cluster
- Role bindings,  provice you with permissions to the paticular namespaces

Simple app management on Kubernetes with operators

- Adopt a cloud-native paradigm for managing apps in clusters and simplify management with core operator concepts
- the desire state, the current state, driving from current state to match desire state, the operator to ensure the system is up-to-date and healthy

Customizing, extending API with admission controllers

- Add new and unique capabilities to your cluster by modifying how API objects are validated or created
- the person made API calls (RBAC fo authorize/authentication)to etcd datebase
- After authn & authz, admission controller will be called

by web hooks: validating, mutating

- validating admission controllers: policy engine, like gatekeepers, check specific values in requests
- mutating admission controllers: service mesh, change values in request, service mesh can inject side car without change any other configs

Clusters and workloads

- See how infrastructure components like the control plane, nodes, and node pools work in AKS—along with workload resources like pods and sets

Access and identity

- Authenticate and assign permissions in AKS using Kubernetes service accounts, AAD integration, role-based access control, Roles and ClusterRoles, and RoleBindings and ClusterRoleBindings

App and cluster security

- Safeguard your applications in AKS with master components security, node security, cluster upgrades, network security, and Kubernetes secrets

Network concepts for apps

- Provide networking to your applications in AKS, including services, Azure virtual networks, ingress controllers, and network policies

## Kubernetes 网络模型

在 Kubernetes 中：
服务以逻辑方式对 Pod 进行分组，以允许通过 IP 地址或 DNS 名称在特定端口上进行直接访问。

ServiceType 允许指定所需的服务类型。

可以使用负载均衡器分发流量。

应用程序流量的第 7 层路由也可以通过入口控制器来实现。

可以控制群集节点的出站（出口）流量。

使用网络策略可提供安全性，还可筛选 Pod 网络流量

可用的 ServiceType 如下：

- ClusterIP
  ClusterIP 创建在 AKS 群集中使用的内部 IP 地址。 此服务适用于支持群集中其他工作负荷的仅限内部使用的应用程序。 如果未为服务显式指定类型，则使用此默认值
- NodePort
  NodePort 在基础节点上创建端口映射，该映射允许使用节点 IP 地址和端口直接访问应用程序。
- LoadBalancer
  创建 Azure 负载均衡器资源、配置外部 IP 地址并将请求的 Pod 连接到负载均衡器后端池。 为允许客户流量发送到应用程序，要在所需端口上创建负载均衡规则。
- ExternalName
  创建特定的 DNS 条目，便于访问应用程序

Kubenet 和 Azure CNI 都为 AKS 群集提供网络连接。 不过，这两个模型各有优缺点。 从较高层面讲，需要考虑以下因素：

- kubenet
  - 节省 IP 地址空间。
  - 使用 Kubernetes 内部或外部负载均衡器可从群集外部访问 Pod。
  - 可手动管理和维护用户定义的路由 (UDR)。
  - 每个群集最多可包含 400 个节点。
- Azure CNI
  - Pod 建立了全面的虚拟网络连接，可以通过其专用 IP 地址直接从已连接的网络对其进行访问。
  - 需要更多的 IP 地址空间。

## Kubernetes 数据存储

Store applications in AKS using volumes, persistent volumes, storage classes, and persistent volume chains

- emptyDir
  - 删除 Pod 后，卷也会删除。 此卷通常使用基础本地节点磁盘存储，但它也可以仅存在于节点的内存中
  - secret：可以使用 secret 卷将敏感数据注入 Pod，例如密码
  - configMap：可以使用 configMap 将键值对属性注入 Pod，例如应用程序配置信息

- 永久性卷：
  - 作为 Pod 生命周期的一部分定义和创建的卷仅在删除 Pod 之前存在

- 存储类：若要定义不同的存储层（例如高级和标准），可创建 StorageClass。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_ZRS
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

- 永久性卷声明：PersistentVolumeClaim 会请求特定 StorageClass、访问模式和大小的存储。

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx
spec:
  containers:
    - name: myfrontend
      image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
      volumeMounts:
      - mountPath: "/mnt/azure"
        name: volume
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: azure-managed-disk
```
