---
layout:     post
title:      SpringCloud|Kubernetes示例
subtitle:   SpringCloudKubernetes
date:       2023-08-17
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringCloud
    - Kubernetes
---

- [获取源码并加载环境](#获取源码并加载环境)
- [spring-cloud-kubernetes下的示例](#spring-cloud-kubernetes下的示例)
- [mac minikube 本地构建执行SpringCloud-Kubernetes-Example](#mac-minikube-本地构建执行springcloud-kubernetes-example)

背景：2019年开始 SpringCloud 开始向 K8S容器化流程演变，逐步诞生了【Spring-Cloud-Kubernetes】

## 获取源码并加载环境

从github仓库获取源码

```bash
git clone https://github.com/spring-cloud/spring-cloud-kubernetes.git 
```

导入IDEA后，确认以下构建环境

- maven 3.+
- jdk17
- 当前springcloudkubernetes版本：3.1.0-SNAPSHOT
- spring-cloud-build 4.1.0-SNAPSHOT

由于代码涉及snapshot库的代码拉取（体现Spring全家桶的协同），需要关注项目根目录下的.setting文件，作为一般本地maven文件的补充，主要关注仓库部分，构建的时候可以直接选定这个setting文件，消除全项目红波浪线的问题

确认以上环境后

```bash
./mvnw clean package -Dmaven.test.skip=true
```

保持网络通畅，期间会需要连接docker，现在打包工具，确定JRE版本，下载JRE版本，打包到镜像仓库等动作

## spring-cloud-kubernetes下的示例

使用spring-cloud-kubernetes下的example体验springcloud与kubernetes的融合与互动

```bash
/spring-cloud-kubernetes/spring-cloud-kubernetes-examples/kubernetes-leader-election-example

 mvn spring-boot:build-image -Dspring-boot.build-image.imageName=org/kubernetes-leader-election-example

:~$ docker images
REPOSITORY                                                                    TAG       IMAGE ID       CREATED         SIZE
busybox                                                                       latest    a416a98b71e2   4 weeks ago     4.26MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver            v1.24.1   e9f4b425f919   15 months ago   130MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler            v1.24.1   18688a72645c   15 months ago   51MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy                v1.24.1   beb86f5d8e6c   15 months ago   110MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager   v1.24.1   b4ea7e648530   15 months ago   119MB
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd                      3.5.3-0   aebe758cef4c   16 months ago   299MB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause                     3.7       221177c6082a   17 months ago   711kB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                   v1.8.6    a4ca41631cc7   22 months ago   46.8MB
k8s.gcr.io/pause                                                              3.6       6270bb605e12   24 months ago   683kB
registry.cn-hangzhou.aliyuncs.com/google_containers/storage-provisioner       v5        6e38f40d628d   2 years ago     31.5MB
org/kubernetes-leader-election-example      


eval $(minikube docker-env)
```

## mac minikube 本地构建执行SpringCloud-Kubernetes-Example

mac 安装 minikube此处不描述。本文主要参照官方流程，依据源码构建实践示例代码。

依赖版本说明：

- docker 20.10.20
- minikube v1.28.0
- k8s 1.24.1
- maven 3.6
- jdk17
- spring-cloud-kubernetes 3.1.0-SNAPSHOT
- spring-cloud-build 4.1.0-SNAPSHOT

以上maven和jdk是必要条件，注意版本的对应

基础条件准备

- 安装maven3+环境
- 安装JDK17环境
- 启动docker
- 启动minikube  `minikube start`

查看minikube访问的docker环境

`minikube docker-env`

```bash
:~$ minikube docker-env
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://127.0.0.1:54965"
export DOCKER_CERT_PATH="~/.minikube/certs"
export MINIKUBE_ACTIVE_DOCKERD="minikube"

# To point your shell to minikube's docker-daemon, run:
# eval $(minikube -p minikube docker-env)
```

如果本地docker没有habor等私仓环境,注意这行注释，需要让当前shell去访问minikube的docker 需要执行 `eval $(minikube -p minikube docker-env)`。官方文档也很细心地写明了这个指令哦，对于小白还是比较友好
指令执行后，即可在当前这个shell直接`docker images`看到仓库中的镜像了


以下开始执行示例项目的部署

进入example项目文件夹下，为项目创建role和role-binding

```xml
kubectl apply -f leader-role.yml

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: leader
  labels:
    app: kubernetes-leader-election-example
    group: org.springframework.cloud
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - watch
  - get
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - watch
  - get
  - update
  # resourceNames:
  #   - <config-map name>

----

kubectl apply -f leader-rolebinding.yml

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app: kubernetes-leader-election-example
    group: org.springframework.cloud
  name: leader
roleRef:
  apiGroup: ""
  kind: Role
  name: leader
subjects:
- kind: ServiceAccount
  name: default
  apiGroup: ""
```

在example目录下通过maven组件构建docker镜像

```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=org/kubernetes-leader-election-example
```

通过以下yaml构建服务

```xml
---
apiVersion: v1
kind: List
items:
  - apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: kubernetes-leader-election-example
      name: kubernetes-leader-election-example
    spec:
      ports:
        - name: http
          port: 80
          targetPort: 8080
      selector:
        app: kubernetes-leader-election-example
      type: ClusterIP
  - apiVersion: v1
    kind: ServiceAccount
    metadata:
      labels:
        app: kubernetes-leader-election-example
      name: kubernetes-leader-election-example
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      labels:
        app: kubernetes-leader-election-example
      name: kubernetes-leader-election-example:view
    roleRef:
      kind: Role
      apiGroup: rbac.authorization.k8s.io
      name: namespace-reader
    subjects:
      - kind: ServiceAccount
        name: kubernetes-leader-election-example
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      namespace: default
      name: namespace-reader
    rules:
      - apiGroups: ["", "extensions", "apps"]
        resources: ["configmaps", "pods", "services", "endpoints", "secrets"]
        verbs: ["get", "list", "watch"]
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kubernetes-leader-election-example
    spec:
      selector:
        matchLabels:
          app: kubernetes-leader-election-example
      template:
        metadata:
          labels:
            app: kubernetes-leader-election-example
        spec:
          serviceAccountName: kubernetes-leader-election-example
          containers:
            - name: kubernetes-leader-election-example
              image: org/kubernetes-leader-election-example:latest
              imagePullPolicy: IfNotPresent
              readinessProbe:
                httpGet:
                  port: 8080
                  path: /actuator/health/readiness
              livenessProbe:
                httpGet:
                  port: 8080
                  path: /actuator/health/liveness
              ports:
                - containerPort: 8080

```

将内部服务的8080映射到ClusterIP的80上，获取本机的代理地址，端口未指定所以每次启动是随机的

```bash

:~$ minikube service kubernetes-leader-election-example --url
😿  service default/kubernetes-leader-election-example has no node port
http://127.0.0.1:57562
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.

```

访问地址 `http://127.0.0.1:57562` 就能获取响应，由于只有一个节点，所以这个节点就是主节点

```bash
curl http://127.0.0.1:57562
I am 'kubernetes-leader-election-example-5fbf89948f-gf8gd' and I am the leader of the 'world'
```

增加实例数增加到2个

```bash
kubectl scale --replicas=2 deployment.apps/kubernetes-leader-election-example

NAMESPACE     NAME                                                  READY   STATUS    RESTARTS       AGE
default       kubernetes-leader-election-example-5fbf89948f-427xs   1/1     Running   4 (25m ago)    3d21h
default       kubernetes-leader-election-example-5fbf89948f-gf8gd   1/1     Running   4 (25m ago)    3d21h
```

由于节点数增加，两个服务也就开始存在负载均衡的效果，依赖k8s调度，此时请求目标地址 `http://127.0.0.1:57562`, 请求一般都是落在从节点上，反复请求节点会交替但都是到从节点

```bash
I am 'kubernetes-leader-election-example-5fbf89948f-427xs' but I am not a leader of the 'world'
I am 'kubernetes-leader-election-example-5fbf89948f-gf8gd' but I am not a leader of the 'world'
```
