---
layout:     post
title:      REST_IN_PRACTICE-MS-WebAPI
subtitle:  Azure-应用程序-WebAPI设计
date:       2020-08-01
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Web
---

# Azure-应用程序-WebAPI设计

- 上一篇[ REST_IN_PRACTICE_Paper ]有描述，2000年Fielding提出的REST理念-表现层状态转移，是用于[ 通过有原则的使用架构约束来理解和评估基于网络的应用程序软件的架构设计，从而获得架构所需的功能、性能和社会属性 ]

- REST独立于任何底层协议，但是基于网络的使用HTTP作为应用协议的REST架构设计应用广泛

## 1.目的

精心设计WebAPI,旨在平台独立性和服务演进


## 2.使用HTTP的RESTful API的主要设计原则

- API设计是围绕资源的，资源是客户端可以访问的对象、数据或服务

- 资源具有标识符，URI是其唯一的标识符

- 客户端通过交换资源与服务交互，许多WebApi使用JSON作为交换格式

- REST API 使用统一的接口，有助于客户端与服务分离。常见操作GET POST PUT PATCH DELETE

- REST API 无状态请求模型。HTTP请求式独立的，可以以任何顺序发生，每个请求是原子性的，此约束是的Web服务具有高度可伸缩性
