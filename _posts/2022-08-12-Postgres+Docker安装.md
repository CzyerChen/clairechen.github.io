---
layout:     post
title:      Docker Postgres 安装部署
subtitle:   redis
date:       2022-08-12
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Docker
    - Postgres
---

以下为实验版本：

> Docker version：18.09.2
> 
> Postgres： 11.4

内容目录:
- [1.确定需要安装的版本](#1确定需要安装的版本)
- [2.获取指定版本镜像](#2获取指定版本镜像)
- [3.指定数据挂载目录](#3指定数据挂载目录)
- [4.启动Postgres服务](#4启动postgres服务)
- [5.创建数据库、用户](#5创建数据库用户)
  - [5.1 进入容器内部](#51-进入容器内部)
  - [5.2 切换超级用户创建用户](#52-切换超级用户创建用户)
  - [5.3 切换超级用户创建数据库](#53-切换超级用户创建数据库)

---------

### 1.确定需要安装的版本

版本不同可能还是会存在差异，这边没有追新，选择了11.4的版本进行测试

### 2.获取指定版本镜像

```bash
docker search postgres
docker pull postgres:11.4
```

### 3.指定数据挂载目录

为了镜像停止后数据还存在，一般都会将数据目录挂载出来。（其他容器化的部署都是这样操作的）

```bash
#创建挂载目录
mkdir ~/docker/postgres/data
```

### 4.启动Postgres服务

```bash
#测试场景直接将5432的默认端口代理出来了,绑定了数据持久化目录，指定了11.4的版本
docker run --name postgresql -e POSTGRES_PASSWORD=YOUR_PASSWORD -p 5432:5432 -v ~/docker/postgres/data:/var/lib/postgresql/data -d postgres:11.4
```

### 5.创建数据库、用户

#### 5.1 进入容器内部

```bash
docker exec -it postgresql /bin/sh
```

#### 5.2 切换超级用户创建用户

```bash
# su postgres
postgres@a:/$ createuser -P -s -e test
Enter password for new role: 
Enter it again: 
SELECT pg_catalog.set_config('search_path', '', false)
CREATE ROLE test PASSWORD 'md***************e' SUPERUSER CREATEDB CREATEROLE INHERIT LOGIN;
postgres@a:/$ psql
psql (11.4 (Debian 11.4-1.pgdg90+1))
Type "help" for help.
```

#### 5.3 切换超级用户创建数据库

```bash
postgres=# create database sms owner=test;
CREATE DATABASE
postgres=# \l
                                 List of databases
   Name    |   Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+-----------+----------+------------+------------+-----------------------
 postgres  | postgres  | UTF8     | en_US.utf8 | en_US.utf8 | 
 sms       | test      | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres  | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |           |          |            |            | postgres=CTc/postgres
 template1 | postgres  | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |           |          |            |            | postgres=CTc/postgres
(4 rows)


```

----------
至此，从外部就可通过 test@127.0.0.1:5432/sms 的方式访问DB了

