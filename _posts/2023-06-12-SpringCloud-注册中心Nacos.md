---
layout:     post
title:      SpringCloud|注册中心Nacos
subtitle:   SpringCloud注册中心
date:       2023-08-17
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringCloud
    - Nacos
---


## 1、部署Nacos Server

部署Nacos Server，方式与模式（单机、集群、多集群）有很多，根据自己的情况任选。这边为了测试方便直接部署的单机编译包。

- 源码打包 `git clone https://github.com/alibaba/nacos.git`
- 直接下载编译后的压缩包 `https://github.com/alibaba/nacos/releases`
- Docker部署 `git clone https://github.com/nacos-group/nacos-docker.git`
- K8S部署 `git clone https://github.com/nacos-group/nacos-k8s.git`

官方已经提供了保姆级别的入门部署指南，可以参考流程操作：https://nacos.io/zh-cn/docs/quick-start.html

选择合适的端口，部署启动后，快速后台运行 ` sh startup.sh -m standalone`

```bash
        ,--.
       ,--.'|
   ,--,:  : |                                           Nacos 2.2.3
,`--.'`|  ' :                       ,---.               Running in stand alone mode, All function modules
|   :  :  | |                      '   ,'\   .--.--.    Port: 8848
:   |   \ | :  ,--.--.     ,---.  /   /   | /  /    '   Pid: 79969
|   : '  '; | /       \   /     \.   ; ,. :|  :  /`./   Console: http://192.168.2.180:8848/nacos/index.html
'   ' ;.    ;.--.  .-. | /    / ''   | |: :|  :  ;_
|   | | \   | \__\/: . ..    ' / '   | .; : \  \    `.      https://nacos.io
'   : |  ; .' ," .--.; |'   ; :__|   :    |  `----.   \
|   | '`--'  /  /  ,.  |'   | '.'|\   \  /  /  /`--'  /
'   : |     ;  :   .'   \   :    : `----'  '--'.     /
;   |.'     |  ,     .-./\   \  /            `--'---'
'---'        `--`---'     `----'

2023-06-12 10:40:01,692 INFO Tomcat initialized with port(s): 8848 (http)

2023-06-12 10:40:01,836 INFO Root WebApplicationContext: initialization completed in 3677 ms

2023-06-12 10:40:08,212 INFO Adding welcome page: class path resource [static/index.html]

2023-06-12 10:40:08,696 WARN You are asking Spring Security to ignore Ant [pattern='/**']. This is not recommended -- please use permitAll via HttpSecurity#authorizeHttpRequests instead.

2023-06-12 10:40:08,696 INFO Will not secure Ant [pattern='/**']

```
