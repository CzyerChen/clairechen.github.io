---
layout:     post
title:      Skywalking之JavaAgent本地构建
subtitle:   Skywalking
date:       2023-11-01
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Skywalking
---

- MAC 10.15.7
- IDEA 2021.2
- node v16.14.0 (据代码要求安装了一个新版本)
- npm 8.3.1
- Skywalking 9.6.0-SNAPSHOT
- JDK 11 (最新的代码要看最新的要求，注意不能使用JDK8，会构建失败)
- Maven 3.6.0

## 关于本地构建JavaAgent源码

### 获取源码，加载submodule

分步执行：

```bash
git clone https://github.com/apache/skywalking-java.git
git submodule init //初始化仓库目录位置
git submodule update //**这步骤至关重要**
```

或者一气呵成：

```bash
git clone --recurse-submodules https://github.com/apache/skywalking.git
```

### maven构建

```bash
#注意JDK版本，JDK11
mvn clean package -Dmaven.test.skip=true
```

构建后产生编译的文件，需要加为源码，找到以下文件夹右键 **"Mark Dictory as"** -> **"Generated Sources Root"**

```xml
~/skywalking-java/apm-protocol/apm-network/target/generated-sources/protobuf/grpc-java
~/skywalking-java/apm-protocol/apm-network/target/generated-sources/protobuf/java
```

正常就可以进行本地调试了
