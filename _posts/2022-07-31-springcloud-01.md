---
layout:     post
title:      springcloud 初体验
subtitle:   redis
date:       2022-07-31
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - springcloud
    - springcloud Alibaba
---

什么是微服务，为什么要用微服务，它单体架构的服务有什么区别，能解决什么问题？

单体架构架构简单、开发部署测试简单，但是在工程代码复杂性高、相互兼顾一个代码变动可能影响大，项目部署慢效率低、扩展性不足，对新技术的支持有限

springcloud的使用，现需要理解springBoot的脚手架
微服务是将单体架构按照功能、区域进行拆分，模块小而精，单独部署，独立版本，减少项目模块间的耦合依赖。能够单独发布、灵活敏捷
可全自动部署，可水平扩展，提升整体项目的吞吐

微服务的适用场景
-  大型、复杂的项目
- 服务压力大
- 变更较快的服务

微服务不适用的场景
- 业务稳定
- 迭代周期长

如何添加ali的spring starter?
[spring官网可查看版本](https://spring.io/projects/spring-boot#learn)
在IDEA先配置Preferences配置HTTP Proxy,添加https://start.aliyun.com/,添加后check Connection进行检测，然后保存
在New Project SpringInitilizer自动选择spring initializer，即可添加相关依赖

如何做微服务的拆分
- 领域驱动，通过领域分析，将领域对象抽离，进行项目模块的组织
- 面向对象设计的方式，通过传统面向对象思维将对象拆分，进行模块功能的组织
- 另外也可根据自己的分析合理拆分


- 单体转微服务
- 做好领域分析、面向对象分析，产出项目架构图
- 进行数据库设计，设计后产生初始化数据库脚本
- 根据项目设计文档，编写API设计文档


SpringCloudAlibaba是什么？快速构建分布式系统的工具集
