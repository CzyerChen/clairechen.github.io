---
layout:     post
title:      SpringBoot-Maven|Gradle多模块
subtitle:   SpringCloud
date:       2023-05-30
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringBoot
    - Maven
    - Gradle
---

- [1、IDEA中maven多模块](#1idea中maven多模块)
  - [常规搭建流程](#常规搭建流程)
  - [Maven常规指令](#maven常规指令)
- [2、IDEA中gradle多模块](#2idea中gradle多模块)
  - [搭建流程](#搭建流程)
  - [gradle常用指令](#gradle常用指令)


比较熟悉的模式是Springboot+Maven多模块的组织方式，由于近期Gradle势力很猛，据Gradle官网给出的5种压测场景的数据来看，Gradle的处理性能确实比Maven快，所以就来体验一下Gradle怎么做多模块搭建和各种依赖的引入和查看。

对Gradle官网提供的Maven与Gradle的对比感兴趣的，可以点：https://gradle.org/maven-vs-gradle/

材料：

- SpringBoot
- IDEA
- Maven
- Gadle

## 1、IDEA中maven多模块

### 常规搭建流程

- 新建一个项目，可以使用Spring Initializer进行初始化，选maven项目
- 删除src目录，仅保留外部的pom.xml
- 右击父项目，New->Module->Maven项目，填项目名就可以产生一个maven子项目
- 在父项目中pom.xml自动写入子模块名<modules>，子模块pom.xml中自动写入父项目名<parent>
- 在父项目中可以做版本的统一定义<properties>，依赖的统一申明<dependencyManagement> <dependencies>

### Maven常规指令

常规打包指令

```bash
#跳过单测打包
mvn clean package -Dmaven.test.skip=true
#跳过单测打包，并把打好的包上传到本地仓库
mvn clean install -Dmaven.test.skip=true
#跳过单测打包，并把打好的包上传到远程仓库
mvn clean deploy -Dmaven.test.skip=true
```

其他指令

```bash
#查看版本 
mvn -v 

#创建 Maven 项目 
mvn archetype:create 

#编译源代码 
mvn compile

#编译测试代码 
mvn test-compile 

#运行应用程序中的单元测试 
mvn test 

#生成项目相关信息的网站 
mvn site 

#依据项目生成 jar 文件 
mvn package 

#在本地 Repository 中安装 jar
mvn install  

#忽略测试文档编译
mvn -Dmaven.test.skip=true  

#清除目标目录中的生成结果
mvn clean  

#将.java类编译为.class文件
mvn clean compile  

#进行打包
mvn clean package  

#执行单元测试
mvn clean test 

#部署到版本仓库 
mvn clean deploy 

#使其他项目使用这个jar,会安装到maven本地仓库中 
mvn clean install 

#创建项目架构
mvn archetype:generate  

#查看已解析依赖 
mvn dependency:list 

#【*常用*】看到依赖树
mvn dependency:tree com.xx.xxx  

#查看依赖的工具 
mvn dependency:analyze 

#从中央仓库下载文件至本地仓库
mvn help:system  

#查看当前激活的profiles 
mvn help:active-profiles 

#查看所有profiles 
mvn help:all-profiles 

#查看完整的pom信息
mvn help:effective -pom 

```

## 2、IDEA中gradle多模块

### 搭建流程

基本流程是一致的，相对Maven的配置，gradle父项目中有一个subprojects的配置，需要手动调整。

- 新建一个项目，可以使用【Spring Initializer】进行初始化，选择gradle项目
- 删除src目录，保留外部的【build.gradle/gradlew/gradlew.bat/setting.gradle】
- 项目顶级目录下会自动建立gradle目录，【gradle->wrapper->gradle-wrapper.jar/gradle-wrapper.preperties】，如果是自建的项目需要注意引入（此处项目maven项目会多比较多文件）
- 右击父项目，【New->Module->Gradle】项目，填项目名就可以产生一个gradle子项目。子模块只有一个【build.gradle】文件
- 在父项目setting.gradle中自动写入子模块名【include "子模块"】，不同于maven，这边是将子模块的包含关系写在另外的文件中。子模块中没有父项目的痕迹。
- 在父项目中可以做版本的统一定义【ext】，依赖的统一申明subprojects{dependencies{} dependencyManagement{}} 
- gradle项目有一定差异的是，需要在父项目中将子模块的依赖单独写明

```bash
plugins {
    id 'org.springframework.boot' version '2.3.12.RELEASE'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group = 'com.learning'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

ext {
    set('springCloudVersion', "Hoxton.SR12")
}

subprojects {

    apply plugin: 'java'
    apply plugin: 'org.springframework.boot'  //引入外部定义的插件
    apply plugin: 'io.spring.dependency-management' //引入外部定义的插件

    ext {
        springVersion = "2.3.12.RELEASE" //统一定义版本，后面引用
        compileJava.options.encoding = 'UTF-8'
        compileTestJava.options.encoding = 'UTF-8'
        springCloudVersion = "Hoxton.SR12"
    }

    sourceCompatibility = 1.11
    targetCompatibility = 1.11

    tasks.withType(JavaCompile) {
        options.encoding = 'UTF-8'
    }

    //配置依赖
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter' //一次性定义子项目包含的依赖
        implementation 'com.alibaba:fastjson:2.0.33' //指定版本
        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation('org.springframework.boot:spring-boot-starter-test') {
            exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        }
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }

    tasks.named('test') {
        useJUnitPlatform()
    }
}
```

### gradle常用指令

- Gradle 是google开发的基于groovy语言 ，用于代替 ant 构建的一种配置型语言
- 基于gradle构建的工程都有一个gradle本地代理，叫做 gradle wrapper，就是前面步骤里面项目根目录下gradle子目录里面的内容

```bash
# 查看构建版本
./gradlew -v
# 清除build文件夹
./gradlew clean
a@b:~/files/gitFile/springcloudall$ ./gradlew  clean

BUILD SUCCESSFUL in 827ms
3 actionable tasks: 2 executed, 1 up-to-date

# 检查依赖并编译打包
./gradlew build

./gradlew dependencies

a@b:~/files/gitFile/springcloudall/eurekaclient$ ../gradlew dependencies

> Task :eurekaclient:dependencies

------------------------------------------------------------
Project ':eurekaclient'
------------------------------------------------------------

annotationProcessor - Annotation processors and their dependencies for source set 'main'.
\--- org.projectlombok:lombok -> 1.18.20

apiElements - API elements for main. (n)
No dependencies

archives - Configuration for archive artifacts. (n)
No dependencies

bootArchives - Configuration for Spring Boot archive artifacts.
No dependencies

compileClasspath - Compile classpath for source set 'main'.
+--- org.projectlombok:lombok -> 1.18.20
+--- org.springframework.boot:spring-boot-starter -> 2.3.12.RELEASE
|    +--- org.springframework.boot:spring-boot:2.3.12.RELEASE
|    |    +--- org.springframework:spring-core:5.2.15.RELEASE
|    |    |    \--- org.springframework:spring-jcl:5.2.15.RELEASE
|    |    \--- org.springframework:spring-context:5.2.15.RELEASE
|    |         +--- org.springframework:spring-aop:5.2.15.RELEASE
|    |         |    +--- org.springframework:spring-beans:5.2.15.RELEASE
|    |         |    |    \--- org.springframework:spring-core:5.2.15.RELEASE (*)
|    |         |    \--- org.springframework:spring-core:5.2.15.RELEASE (*)
|    |         +--- org.springframework:spring-beans:5.2.15.RELEASE (*)
|    |         +--- org.springframework:spring-core:5.2.15.RELEASE (*)
|    |         \--- org.springframework:spring-expression:5.2.15.RELEASE
|    |              \--- org.springframework:spring-core:5.2.15.RELEASE (*)
|    +--- org.springframework.boot:spring-boot-autoconfigure:2.3.12.RELEASE
|    |    \--- org.springframework.boot:spring-boot:2.3.12.RELEASE (*)
|    +--- org.springframework.boot:spring-boot-starter-logging:2.3.12.RELEASE
|    |    +--- ch.qos.logback:logback-classic:1.2.3
|    |    |    +--- ch.qos.logback:logback-core:1.2.3
|    |    |    \--- org.slf4j:slf4j-api:1.7.25 -> 1.7.30
|    |    +--- org.apache.logging.log4j:log4j-to-slf4j:2.13.3
|    |    |    +--- org.slf4j:slf4j-api:1.7.25 -> 1.7.30
|    |    |    \--- org.apache.logging.log4j:log4j-api:2.13.3
|    |    \--- org.slf4j:jul-to-slf4j:1.7.30
|    |         \--- org.slf4j:slf4j-api:1.7.30
|    +--- jakarta.annotation:jakarta.annotation-api:1.3.5
|    +--- org.springframework:spring-core:5.2.15.RELEASE (*)

# 或者模组的 依赖
./gradlew app:dependencies
# 检索依赖库
./gradlew app:dependencies | grep CompileClasspath
# windows 没有 grep 命令
./gradlew app:dependencies | findstr "CompileClasspath"
```


gradle的多模块打包和构建