---
layout:     post
title:      SpringBoot 配置热加载
subtitle:   SpringCloudKubernetes
date:       2023-09-15
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringBoot
    - 热加载
---

- [1、从actuator端点刷新应用属性](#1从actuator端点刷新应用属性)
  - [1.1 @ConfigurationProperties + /actuator/refresh](#11-configurationproperties--actuatorrefresh)
  - [1.2 @Value + @RefreshScope + /actuator/refresh](#12-value--refreshscope--actuatorrefresh)
- [2、从外部文件加载和更新应用属性](#2从外部文件加载和更新应用属性)

springboot的配置通常是在以下文件中进行配置，在打包时就涵盖，或者在发布时指定外部配置文件：

- application.properties
- application.yaml
- application-{env}.properties
- application-{env}.yaml

通常一旦程序发布就不能修改配置，那有什么办法能够修改这上下文里的参数呢？
以下方法涉及：

- SpringBoot SpringCloud
- 动态代理
- 类加载
- ContextRefresher.refresh()上下文刷新

以下测试版本说明：

- springcloud: 4.0.4
- springboot: 3.1.0

## 1、从actuator端点刷新应用属性

主要用 SpringCloud 的上下文参数刷新的功能，接口 /actuator/refresh 的端点，实现应用属性的实时刷新。

### 1.1 @ConfigurationProperties + /actuator/refresh

1.引入springboot框架
2.引入 springcloud 依赖，springboot-actuator 依赖，开启 refresh 端口

application.yaml:

```yml
server:
  port: 8080
spring:
  application:
    name: springboot-config-refresh
management:
  endpoints:
    web:
      exposure:
        include: refresh
demo:
  message: files
```

pom.xml:

```xml
 <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter</artifactId>
            <version>${spring-cloud.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.22</version>
        </dependency>
    </dependencies>
```

3.书写配置类 MyConfig ，结合 @ConfigurationProperties 这个注解是扫描生效的重点

MyConfig.class

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "demo")
public class MyConfig {

    private String message ="default message";

}
```

4.书写值查询的接口进行验证

```java
@RestController
@RequestMapping
public class ValueController {
    @Autowired
    private MyConfig myConfig;

    @GetMapping
    public String value(){
     return myConfig.getMessage();
    }
}
```

5.请求API

请求API获取初始配置

```bash
curl --location --request GET '127.0.0.1:8080/'

files
```

- war包部署或本地测试，可以修改class下配置文件application.yaml
- jar包部署，需要启动时指定外部的文件，`--spring.config.location=/Users/chenzy/application.yaml`，此时修改外部文件保存即可

```yml
server:
  port: 8080
spring:
  application:
    name: springboot-config-refresh
management:
  endpoints:
    web:
      exposure:
        include: refresh
  endpoint:
    refresh:
      enabled: true
demo:
  message: newfiles
```

请求 `/actuator/refresh` 触发旧bean的销毁，并返回修改项

```bash
curl --location --request POST '127.0.0.1:8080/actuator/refresh' \
--header 'Content-Type: application/json'

[
    "demo.message"
]
```

请求API获取新配置

```bash
curl --location --request GET '127.0.0.1:8080/'

newfiles
```

配置被刷新，符合预期

### 1.2 @Value + @RefreshScope + /actuator/refresh

以下演示 @Value 在指定的配置类中，而非 Controller 中直接引入
1.2步骤同（1.1 @ConfigurationProperties + /actuator/refresh）

3.书写 MyValue ，使用 @Value 的注解获取配置文件参数值

```java
@Data
@Configuration
@RefreshScope
public class MyValue {
    @Value("${demo.value}")
    private String demoValue;
}

```

application.yaml

```yml
server:
  port: 8080
spring:
  application:
    name: springboot-config-refresh
management:
  endpoints:
    web:
      exposure:
        include: refresh
  endpoint:
    refresh:
      enabled: true
demo:
  message: files
  value: files
```

4.书写值查询的接口进行验证

```java
@RestController
@RequestMapping
public class ValueController {
    @Autowired
    private MyConfig myConfig;
    @Autowired
    private MyValue myValue;

    @GetMapping
    public String value(){
     return myConfig.getMessage();
    }

    @GetMapping("demo")
    private String demoValue(){
        return myValue.getDemoValue();
    }
}
```

5.请求API

请求API获取初始配置

```bash
curl --location --request GET '127.0.0.1:8080/demo'

files
```

参照 上一个方法去修改对应配置的值
调用 /actuator/refresh 方法后，能够获取到最新值

```bash
curl --location --request POST '127.0.0.1:8080/actuator/refresh' \
--header 'Content-Type: application/json'

[
    "demo.value"
]
```

```bash
curl --location --request GET '127.0.0.1:8080/demo'

newfiles
```

## 2、从外部文件加载和更新应用属性

这种方法目前用处不多，可以自行[参考demo](https://www.baeldung.com/spring-reloading-properties)
