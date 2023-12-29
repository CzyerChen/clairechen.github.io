---
layout:     post
title:      SpringCloud实战之Feign 2.x 迁移到 4.x
subtitle:   SpringCloud Feign 
date:       2023-12-29
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringCloud
    - Feign
---

从官方文档中去发现变化的点：

[关于Okhttp from 3.x to 4.x](https://square.github.io/okhttp/changelogs/upgrading_to_okhttp_4/)
[关于Spring-cloud+feign from 2.2.x](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.x/reference/html/)
[关于Spring-cloud+feign from 4.1.x](https://docs.spring.io/spring-cloud-openfeign/docs/4.1.x/reference/html/)

- [spring-cloud-openfeign 变化对比](#spring-cloud-openfeign-变化对比)
  - [如何引入feign?](#如何引入feign)
  - [如何加载feign?](#如何加载feign)
  - [如何申明一个client？](#如何申明一个client)
  - [关于load-balancer](#关于load-balancer)
  - [关于资源加载](#关于资源加载)
  - [关于启动默认的httpclient和okhttp的配置](#关于启动默认的httpclient和okhttp的配置)
  - [关于 SpringEncoder 配置](#关于-springencoder-配置)
  - [关于熔断的配置](#关于熔断的配置)
  - [关于请求/响应的压缩](#关于请求响应的压缩)
  - [关于SpringData的支持](#关于springdata的支持)
- [其他 4.1.x的新特性](#其他-41x的新特性)
- [SpringBoot + SpringCloud + Feign + Okhttp 实践示例](#springboot--springcloud--feign--okhttp-实践示例)

## spring-cloud-openfeign 变化对比

### 如何引入feign?

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
```

### 如何加载feign?

通过注解：`@EnableFeignClients`，开始feign的注入

### 如何申明一个client？

```java
@FeignClient("stores")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    Page<Store> getStores(Pageable pageable);

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);

    @RequestMapping(method = RequestMethod.DELETE, value = "/stores/{storeId:\\d+}")
    void delete(@PathVariable Long storeId);
}
```

### 关于load-balancer

**2.2.x中**
`@FeignClient` 会创建一个`Ribbon load-balancer` 或 `Spring Cloud LoadBalancer`. 取决于`spring.cloud.loadbalancer.ribbon.enabled` 配置项，如果true则使用`Ribbon load-balancer` ，如果false则使用`Spring Cloud LoadBalancer`

**4.1.x中**
`@FeignClient` 直接创建一个 `Spring Cloud LoadBalancer client`

### 关于资源加载

**4.1.x中**
`@FeignClient` 注解被解析的时间更早，便于很多逻辑的处理，如果想要懒加载需要显式指定
`spring.cloud.openfeign.lazy-attributes-resolution=true`

springcloud 会通过 `FeignClientsConfiguration` ，将feignclient加入应用上下文
配置包含三部分，一个 `feign.Decoder`, 一个 `feign.Encoder`, 和 一个 `feign.Contract`
可以通过 `@FeignClient` 的contextId来覆盖这几个默认的
springcloud支持不同的client指定不同配置，`@FeignClient(name = "stores", configuration = FooConfiguration.class)`
FooConfiguration 不需要用 @Configuration 进行注释，否则他将成为client的默认配置

`spring-cloud-starter-openfeign` 支持 `spring-cloud-starter-loadbalancer`，去除了`spring-cloud-starter-netflix-ribbon` 的默认支持

### 关于启动默认的httpclient和okhttp的配置

配置项的label上有比较大的区别，从`feign`的配置迁移到了`spring.cloud.openfeign`的命名上，主要还是源于feign自身的调整

|配置项|2.2.x|4.1.x|
|--|--|--|
| ApacheHttpClient|feign.httpclient.enabled=true|spring.cloud.openfeign.httpclient.enabled=true|
|ApacheHC5|feign.httpclient.hc5.enabled=true|spring.cloud.openfeign.httpclient.hc5.enabled=true
|OkHttpClient|feign.okhttp.enabled=true|spring.cloud.openfeign.okhttp.enabled=true|
| Http2Client|-|spring.cloud.openfeign.http2client.enabled=true|

从`Spring Cloud OpenFeign 4`开始，`HttpClient 4`将不再支持，建议使用`HttpClient 5`

**2.2.x** `application.yaml` 示例：

```yml
feign:
  client:
    config:
      feignName:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: full
        errorDecoder: com.example.SimpleErrorDecoder
        retryer: com.example.SimpleRetryer
       defaultQueryParameters:
          query: queryValue
        defaultRequestHeaders:
          header: headerValue
        requestInterceptors:
          - com.example.FooRequestInterceptor
          - com.example.BarRequestInterceptor
        decode404: false
        encoder: com.example.SimpleEncoder
        decoder: com.example.SimpleDecoder
        contract: com.example.SimpleContract

```

**4.1.x** `application.yaml` 示例：

```yml
spring:
    cloud:
        openfeign:
            client:
                config:
                    feignName:
                        url: http://remote-service.com
                        connectTimeout: 5000
                        readTimeout: 5000
                        loggerLevel: full
                        errorDecoder: com.example.SimpleErrorDecoder
                        retryer: com.example.SimpleRetryer
                        defaultQueryParameters:
                            query: queryValue
                        defaultRequestHeaders:
                            header: headerValue
                        requestInterceptors:
                            - com.example.FooRequestInterceptor
                            - com.example.BarRequestInterceptor
                        responseInterceptor: com.example.BazResponseInterceptor
                        dismiss404: false
                        encoder: com.example.SimpleEncoder
                        decoder: com.example.SimpleDecoder
                        contract: com.example.SimpleContract
                        capabilities:
                            - com.example.FooCapability
                            - com.example.BarCapability
                        queryMapEncoder: com.example.SimpleQueryMapEncoder
                        micrometer.enabled: false
```

### 关于 SpringEncoder 配置

|配置项|2.2.x|4.1.x|
|--|--|--|
|改变默认utf-8的配置|feign.encoder.charset-from-content-type=true|spring.cloud.openfeign.encoder.charset-from-content-type=true|

### 关于熔断的配置

|配置项|2.2.x|4.1.x|
|--|--|--|
|熔断|`feign.hystrix.enabled=true`|`spring.cloud.openfeign.circuitbreaker.enabled=true`|

从2020.0.2开始，熔断器的命名规则从 `<feignClientName>_<calledMethod>` 变为 `<feignClientClassName>#<calledMethod>(<parameterTypes>)`，比如`FooClient#bar()`

### 关于请求/响应的压缩

|配置项|2.2.x|4.1.x|
|--|--|--|
|配置压缩|feign.compression.request.enabled=true <br/> feign.compression.response.enabled=true|spring.cloud.openfeign.compression.request.enabled=true <br/>spring.cloud.openfeign.compression.response.enabled=true|

**4.1.x**
由于 OkHttpClient 使用“透明”压缩，如果存在 content-encoding 或 accept-encoding 标头，则禁用该压缩，因此当类路径上存在 feign.okhttp.OkHttpClient 并且 spring.cloud.openfeign.okhttp.enabled 设置为 true 时，我们不会启用压缩

### 关于SpringData的支持

**2.2.x中**
`feign.autoconfiguration.jackson.enabled=true` 开启

**4.1.x中**
只要Jackson Databind 和 Spring Data Commons 被引入了，就会自动开启，如果想禁用，主动配置 `spring.cloud.openfeign.autoconfiguration.jackson.enabled=false`

## 其他 4.1.x的新特性

- 支持Spring @RefreshScope `spring.cloud.openfeign.client.refresh-enabled=true`
- 支持 OAuth2 `spring.cloud.openfeign.oauth2.enabled=true`
- 转换负载均衡的 HTTP 请求，定义并实现LoadBalancerFeignRequestTransformer
- 支持X-Forwarded Headers `spring.cloud.loadbalancer.x-forwarded.enabled=true`

## SpringBoot + SpringCloud + Feign + Okhttp 实践示例

基础对象：

```java
public class PostInfo {
    private Integer id;
    private Integer userId;
    private String title;
    private String body;

    public Integer getId() {
        return id;
    }

    public void setId(final Integer id) {
        this.id = id;
    }

    public Integer getUserId() {
        return userId;
    }

    public void setUserId(final Integer userId) {
        this.userId = userId;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(final String title) {
        this.title = title;
    }

    public String getBody() {
        return body;
    }

    public void setBody(final String body) {
        this.body = body;
    }

    @Override
    public String toString() {
        return "PostInfo{" +
            "id=" + id +
            ", userId=" + userId +
            ", title='" + title + '\'' +
            ", body='" + body + '\'' +
            '}';
    }
}

@FeignClient(value = "jplaceholder", url = "https://jsonplaceholder.typicode.com/")
public interface PostService {

    @RequestMapping(method = RequestMethod.GET, value = "/posts")
    List<PostInfo> getPosts();

    @RequestMapping(method = RequestMethod.GET, value = "/posts/{postId}", produces = "application/json")
    PostInfo getPostById(@PathVariable("postId") Long postId);

    @RequestMapping(method = RequestMethod.POST, value = "/posts", produces = "application/json")
    PostInfo savePost(@RequestBody PostInfo postInfo);

    @RequestMapping(method = RequestMethod.PATCH, value = "/posts", produces = "application/json")
    PostInfo updatePost(@RequestBody PostInfo postInfo);

    @RequestMapping(method = RequestMethod.DELETE, value = "/posts/{postId}", produces = "application/json")
    void deletePost(@PathVariable("postId") Long postId);
}

@RestController
@RequestMapping("/v1")
public class TestController {
    @Autowired
    private PostService postService;

    @GetMapping("/feign/post")
    public String feignPostRequest(){
        return postService.getPostById(1L).toString();
    }
}
```

差异基本就在配置项上

**springboot2.x** pom.xml

```xml
   <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>2.3.12.RELEASE</spring-boot.version>
        <spring-cloud.version>Hoxton.SR12</spring-cloud.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-okhttp</artifactId>
            <version>10.12</version>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot.version}</version>
                <configuration>
                    <mainClass>com.learning.cloudforfeignokhttp3.CloudForFeignOkhttp3Application</mainClass>
                    <skip>true</skip>
                </configuration>
                <executions>
                    <execution>
                        <id>repackage</id>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

**springboot2.x** application.yaml

```yml
server:
  port: 8088
feign:
  okhttp:
    enabled: true
  httpclient:
    enabled: false
debug: true
```

**springboot3.x** pom.xml

```xml
 <properties>
        <java.version>17</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>3.2.0</spring-boot.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-okhttp</artifactId>
            <version>13.0</version>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot.version}</version>
                <configuration>
                    <mainClass>com.learning.cloudforfeign4.CloudForFeignOkhttp4Application</mainClass>
                    <skip>true</skip>
                </configuration>
                <executions>
                    <execution>
                        <id>repackage</id>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
```

**springboot3.x** application.yaml

```yml
server:
  port: 8089
spring:
  cloud:
    openfeign:
      okhttp:
        enabled: true
      httpclient:
        enabled: false
debug: true
```
