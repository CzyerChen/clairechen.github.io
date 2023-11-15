---
layout:     post
title:      SpringCloudKubernetes-02 ConfigMap&Secret
subtitle:   SpringCloudKubernetes
date:       2023-09-15
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
    - SpringCloud
    - ConfigMap
    - Secret
---

K8S作为当下火热的微服务交付方式，SpringBoot作为目前Java领域最受欢迎的框架，两者的结合是必然的

今天就想讨论一下SpringBoot利用k8s的ConfigMap和Secret将部分配置外挂，将敏感信息脱敏和部分配置动态化

主要涉及到的内容：

- springboot
- k8s configmap
- k8s secret
