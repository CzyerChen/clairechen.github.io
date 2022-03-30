---
layout:     post
title:      minio+Springboot 高性能分布式存储
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

当前测试版本如下：

- springboot 2.3.4.RELEASE
- minio 8.3.7

## minio+Springboot 高性能分布式存储

### 一、安装

办法有很多，可以参照[官网说明](http://minio.org.cn/download.shtml#/linux)

可以区分部署的形式，分为Linux(Docker,wget)/K8S/MasOS/Windows/Source这些形式

以下通过docker进行部署测试

```bash
a@b:~$ docker run -d -p 9000:9000 -p 9001:9001  --name minioserver  -v ~/docker/minio/data:/data   -e "MINIO_ROOT_USER=user"   -e "MINIO_ROOT_PASSWORD=password"   minio/minio server /data  --console-address ":9001" --address ":9000"
```

console at port 9001 ,api at port 9000


## 二、结合SpringBoot进行文件获取和保存

### 坑点: 8.3.7版本对接后，出现异常

```bash

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.4.RELEASE)

engine: [Apache Tomcat/9.0.38]
2022-03-30 13:24:59.814  INFO 1605 --- [  restartedMain] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-03-30 13:24:59.814  INFO 1605 --- [  restartedMain] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1310 ms
2022-03-30 13:24:59.887  WARN 1605 --- [  restartedMain] ConfigServletWebServerApplicationContext : Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'testController': Unsatisfied dependency expressed through field 'minioService'; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'minioService': Unsatisfied dependency expressed through field 'minioClient'; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'minioClient' defined in class path resource [com/learning/minio/config/MinioConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [io.minio.MinioClient]: Factory method 'minioClient' threw exception; nested exception is java.lang.ExceptionInInitializerError
2022-03-30 13:24:59.889  INFO 1605 --- [  restartedMain] o.apache.catalina.core.StandardService   : Stopping service [Tomcat]
2022-03-30 13:24:59.903  INFO 1605 --- [  restartedMain] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2022-03-30 13:24:59.911 ERROR 1605 --- [  restartedMain] o.s.b.d.LoggingFailureAnalysisReporter   : 

***************************
APPLICATION FAILED TO START
***************************

Description:

An attempt was made to call a method that does not exist. The attempt was made from the following location:

    io.minio.S3Base.<clinit>(S3Base.java:98)

The following method did not exist:

    okhttp3.RequestBody.create([BLokhttp3/MediaType;)Lokhttp3/RequestBody;

The method's class, okhttp3.RequestBody, is available from the following locations:

    jar:file:~/.m2/repository/com/squareup/okhttp3/okhttp/3.14.9/okhttp-3.14.9.jar!/okhttp3/RequestBody.class

The class hierarchy was loaded from the following locations:

    okhttp3.RequestBody: file:~/.m2/repository/com/squareup/okhttp3/okhttp/3.14.9/okhttp-3.14.9.jar


Action:

Correct the classpath of your application so that it contains a single, compatible version of okhttp3.RequestBody


Process finished with exit code 0
```

看到提示是OKhttp的问题，由于minio底层要依赖okhttp进行和minio服务端进行通信，考虑是多个okhttp版本造成冲突

#### 解决一 降低minio版本

此处是因为是测试，如果是生产所需可以考虑降低minio版本至8.2.X，具体需要测试下（看到8.2.2是正常的

#### 解决二 剔除冲突的okhttp版本，添加适配的版本

`Unsupported OkHttp library found. Must use okhttp >= 4.8`

okhttp-4.8及以上版本均可

```xml
  <dependency>
            <groupId>io.minio</groupId>
            <artifactId>minio</artifactId>
            <version>8.3.7</version>
            <exclusions>
                <exclusion>
                    <groupId>com.squareup.okhttp3</groupId>
                    <artifactId>okhttp</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

 <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>4.9.0</version>
        </dependency>

```

### 实战

根据自定义配置，配置客户端对象

```java
    @Bean
    public MinioClient minioClient() {
        return MinioClient.builder()
                .endpoint(url)
                .credentials(accessKey, secretKey)
                .build();
    }

```

[具体demo参考代码，赶快have a try【Github:CzyerChen/springboot-forall/boot-for-minio】](https://github.com/CzyerChen/springboot-forall/tree/master/boot-for-minio)
