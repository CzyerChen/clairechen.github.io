---
layout:     post
title:      IDEA + 本地Tomcat 发布与调试
subtitle:   IDEA 本地Tomcat
date:       2021-04-09
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - IDEA
    - Tomcat
    - 项目发布与调试
---

## IDEA + 本地Tomcat 实现项目的发布与调试

- IDEA 2018.3
- Apache Tomcat 8.5.4
- JDK1.8
- MAC

### 一、配置Tomcat

- catalina.sh 添加以下内容，用于开启调试功能

```bash
CATALINA_OPTS="-agentlib:jdwp=transport=dt_socket,server=n,suspend=y,address=127.0.0.1:55948"
```


