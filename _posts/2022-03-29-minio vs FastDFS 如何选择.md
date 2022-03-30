---
layout:     post
title:      minio VS fastDFS
subtitle:   redis
date:       2022-03-30
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - minio
    - springboot
    - 分布式存储
---

我们比较常见的会使用FastDFS作为轻型、免费开源、快速交付的小型文件存储系统（当然还有GFS、HDFS、Lustre 、Ceph 、GridFS 、mogileFS、TFS，适用于不同领域和场景）

近期出现的minio作为开源的、文档详尽的、GO语言编写的又一文件存储系统，无疑成为各项选型的对比对象

|序号|对比项|minio|FastDFS|
|--|--|--|--|
|1|安装部署|适配大数据形式，配置简单，可集群化|配置相对复杂，生产集群化配置步骤较多|
|2|文档|简单、干净、清楚|没有官方化文档|
|3|社区环境|有专业维护，社区活跃|个人性项目|
|4|UI操作|有良好页面供查看|无自带页面|
|5|性能|||
|6|容器支持|||
|7|SDK支持|||
