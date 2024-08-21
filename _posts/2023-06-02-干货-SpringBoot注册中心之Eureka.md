---
layout:     post
title:      SpringCloud-Eureka注册中心、服务提供者与消费者
subtitle:   SpringCloud
date:       2023-05-30
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringCloud
---

- [一、关于注册中心、服务注册、服务发现](#一关于注册中心服务注册服务发现)
  - [1问：为什么需要注册中心？](#1问为什么需要注册中心)
  - [2问：什么是服务注册？](#2问什么是服务注册)
  - [3问：什么是服务发现？](#3问什么是服务发现)
- [二、关于 Eureka 实现服务注册与服务发现](#二关于-eureka-实现服务注册与服务发现)
  - [1.Eureka的特点](#1eureka的特点)
  - [2.Eureka注册中心](#2eureka注册中心)
  - [3.Eureka客户端](#3eureka客户端)

### 一、关于注册中心、服务注册、服务发现

#### 1问：为什么需要注册中心？

随着微服务框架的兴起，服务节点越来越多，服务间的调用会像蜘蛛网一样越来越密集，简单的文件配置形式将无法满足日渐增加的调用关系，所以需要一个职能独立、高可用的中间商-注册中心，来维系这种调用和被调用的关系。

#### 2问：什么是服务注册？

服务提供者将自己的信息写入到注册中心的服务清单，注册中心会定期监测上面的服务，确定服务是否存活，及时排除故障服务。

#### 3问：什么是服务发现？

服务治理框架下，服务消费者向注册中心表明需要调用的服务名，注册中心在服务清单上获取真实的访问信息给到服务消费者。服务提供者可以被服务消费者发现，服务消费者可以主动获取服务提供者的信息。

### 二、关于 Eureka 实现服务注册与服务发现

#### 1.Eureka的特点

- Eureka包含服务端和客户端组件，均采用Java开发，所以对Java实现的分布式系统适配良好
- 其他较为流行的开发平台也有对Eureka的封装
- Eureka提供了Restful API，非Java语言能够通过API方式来实现自己的客户端程序
- Eureka是典型的AP型注册中心，服务端依赖其强一致性，提供了良好的服务实例可用性。但是AP有一个典型的情况就是当集群自身出现问题的时候，为保证数据的一致性，会无法对外提供服务，进入自我保护模式。当集群数据同步一致后，又能对外提供服务。
- Eureka客户端会将自身服务注册到注册中心，并通过服务心跳进行续约，并且能够定期拉取注册中心的服务清单到本地，并进行周期性缓存，便于本地服务的真实IP调用。

以下测试使用版本信息

```xml
java11
gradle
springboot:2.3.12.RELEASE
springcloud:Hoxton.SR12
```

#### 2.Eureka注册中心

加入依赖：

```xml
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
}
```

在服务端主类上，开启Eureka Server，@EnableEurekaServer一个注解开启Eureka注册中心模式

```java
@EnableEurekaServer //一个注解开启Eureka注册中心模式
@SpringBootApplication
public class EurekaserverApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaserverApplication.class, args);
    }

}
```

Eureka Server默认也想自己注册自己的服务，可以使用配置进行关闭

```java
public class EurekaClientConfigBean{
private boolean registerWithEureka = true;
private boolean fetchRegistry = true;
...
}
```

SpringBoot中配置如下：

```xml
server:
  port: 8801
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false #关闭自己注册
    fetch-registry: false #关闭拉取注册
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

将服务启动后，能够看到Eureka Server的管理页面

服务端启动后，能够看到客户端的注册信息，也能够在页面上查看到注册实例的信息与状态

```log
 ......
2023-06-08 10:31:40.186  INFO 29553 --- [           main] DiscoveryClientOptionalArgsConfiguration : Eureka HTTP Client uses Jersey
2023-06-08 10:31:40.205  WARN 29553 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2023-06-08 10:31:40.256  INFO 29553 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2023-06-08 10:31:40.285  INFO 29553 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2023-06-08 10:31:40.285  INFO 29553 --- [           main] com.netflix.discovery.DiscoveryClient    : Client configured to neither register nor query for data.
2023-06-08 10:31:40.294  INFO 29553 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1686191500293 with initial instances count: 0
2023-06-08 10:31:40.321  INFO 29553 --- [           main] c.n.eureka.DefaultEurekaServerContext    : Initializing ...
2023-06-08 10:31:40.323  WARN 29553 --- [           main] c.n.eureka.cluster.PeerEurekaNodes       : The replica size seems to be empty. Check the route 53 DNS Registry
2023-06-08 10:31:40.338  INFO 29553 --- [           main] c.n.e.registry.AbstractInstanceRegistry  : Finished initializing remote region registries. All known remote regions: []
2023-06-08 10:31:40.339  INFO 29553 --- [           main] c.n.eureka.DefaultEurekaServerContext    : Initialized
2023-06-08 10:31:40.351  INFO 29553 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 2 endpoint(s) beneath base path '/actuator'
2023-06-08 10:31:40.410  INFO 29553 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application UNKNOWN with eureka with status UP
2023-06-08 10:31:40.412  INFO 29553 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : Setting the eureka configuration..
2023-06-08 10:31:40.413  INFO 29553 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : Eureka data center value eureka.datacenter is not set, defaulting to default
2023-06-08 10:31:40.414  INFO 29553 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : Eureka environment value eureka.environment is not set, defaulting to test
2023-06-08 10:31:40.425  INFO 29553 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : isAws returned false
2023-06-08 10:31:40.426  INFO 29553 --- [      Thread-10] o.s.c.n.e.server.EurekaServerBootstrap   : Initialized server context
2023-06-08 10:31:40.426  INFO 29553 --- [      Thread-10] c.n.e.r.PeerAwareInstanceRegistryImpl    : Got 1 instances from neighboring DS node
2023-06-08 10:31:40.426  INFO 29553 --- [      Thread-10] c.n.e.r.PeerAwareInstanceRegistryImpl    : Renew threshold is: 1
2023-06-08 10:31:40.426  INFO 29553 --- [      Thread-10] c.n.e.r.PeerAwareInstanceRegistryImpl    : Changing status to UP
2023-06-08 10:31:40.435  INFO 29553 --- [      Thread-10] e.s.EurekaServerInitializerConfiguration : Started Eureka Server
2023-06-08 10:31:40.446  INFO 29553 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8901 (http) with context path ''
2023-06-08 10:31:40.447  INFO 29553 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 8901
2023-06-08 10:31:40.470  INFO 29553 --- [           main] c.l.e.EurekaserverApplication            : Started EurekaserverApplication in 3.977 seconds (JVM running for 4.644)
2023-06-08 10:39:40.450  INFO 29553 --- [a-EvictionTimer] c.n.e.registry.AbstractInstanceRegistry  : Running the evict task with compensationTime 1ms
2023-06-08 10:40:40.455  INFO 29553 --- [a-EvictionTimer] c.n.e.registry.AbstractInstanceRegistry  : Running the evict task with compensationTime 4ms
2023-06-08 10:40:43.326  INFO 29553 --- [nio-8901-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-06-08 10:40:43.326  INFO 29553 --- [nio-8901-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-06-08 10:40:43.350  INFO 29553 --- [nio-8901-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 24 ms
2023-06-08 10:40:43.651  INFO 29553 --- [nio-8901-exec-2] c.n.e.registry.AbstractInstanceRegistry  : Registered instance CLIENT1/192.168.2.180:client1:8902 with status UP (replication=false)
2023-06-08 10:41:40.336  WARN 29553 --- [eerNodesUpdater] c.n.eureka.cluster.PeerEurekaNodes       : The replica size seems to be empty. Check the route 53 DNS Registry
2023-06-08 10:41:40.459  INFO 29553 --- [a-EvictionTimer] c.n.e.registry.AbstractInstanceRegistry  : Running the evict task with compensationTime 2ms
2023-06-08 10:42:40.464  INFO 29553 --- [a-EvictionTimer] c.n.e.registry.AbstractInstanceRegistry  : Running the evict task with compensationTime 4ms
```

#### 3.Eureka客户端

加入依赖：

```xml
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
}
```

客户端配置：

```yml
server:
  port: 8902
spring:
  application:
    name: client1 #默认使用服务名称进行访问，所以必须要起一个有指向意义的服务名字
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8901/eureka/
    fetch-registry: true  #默认true
    register-with-eureka: true #默认true
```

在客户端主类上，通过EnableDiscoveryClient注解开启服务注册与发现

```java
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaclientApplication {

    public static void main(String[] args){
        SpringApplication.run(EurekaclientApplication.class,args);
    }
}
```

使用DiscoveryClient获取服务清单，获取服务实例明细

```java
@Slf4j
@RestController
@RequestMapping("test")
public class TestController {
    @Autowired
    private DiscoveryClient discoveryClient;

    @RequestMapping("/service-instances/{applicationName}")
    public List<ServiceInstance> serviceInstancesByApplicationName(@PathVariable String applicationName) {
        return this.discoveryClient.getInstances(applicationName);
    }

    @GetMapping("/discovery")
    public String discovery() {
        List<String> serviceNames = discoveryClient.getServices();
        for (String serviceName : serviceNames) {
            log.info("***** element:" + serviceName);
            List<ServiceInstance> instances = discoveryClient.getInstances(serviceName);
            for (ServiceInstance instance : instances) {
                log.info(instance.toString());
            }
        }
        return "success";
    }
}
```

服务实例的信息，包含后续服务调用所需的IP与端口等必须信息

```json
[
  {
    "uri": "http://192.168.2.180:8902",
    "metadata": {
      "management.port": "8902"
    },
    "port": 8902,
    "host": "192.168.2.180",
    "scheme": "http",
    "secure": false,
    "instanceId": "192.168.2.180:client1:8902",
    "serviceId": "CLIENT1",
    "instanceInfo": {
      "instanceId": "192.168.2.180:client1:8902",
      "app": "CLIENT1",
      "appGroupName": null,
      "ipAddr": "192.168.2.180",
      "sid": "na",
      "homePageUrl": "http://192.168.2.180:8902/",
      "statusPageUrl": "http://192.168.2.180:8902/actuator/info",
      "healthCheckUrl": "http://192.168.2.180:8902/actuator/health",
      "secureHealthCheckUrl": null,
      "vipAddress": "client1",
      "secureVipAddress": "client1",
      "countryId": 1,
      "dataCenterInfo": {
        "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
        "name": "MyOwn"
      },
      "hostName": "192.168.2.180",
      "status": "UP",
      "overriddenStatus": "UNKNOWN",
      "leaseInfo": {
        "renewalIntervalInSecs": 30,
        "durationInSecs": 90,
        "registrationTimestamp": 1686192043567,
        "lastRenewalTimestamp": 1686192193487,
        "evictionTimestamp": 0,
        "serviceUpTimestamp": 1686192043567
      },
      "isCoordinatingDiscoveryServer": false,
      "metadata": {
        "management.port": "8902"
      },
      "lastUpdatedTimestamp": 1686192043568,
      "lastDirtyTimestamp": 1686192043464,
      "actionType": "ADDED",
      "asgName": null
    }
  }
]
```

客户端启动后能够看到客户端主动注册的日志

```log
.......
2023-06-08 10:40:42.607  INFO 29923 --- [           main] DiscoveryClientOptionalArgsConfiguration : Eureka HTTP Client uses Jersey
2023-06-08 10:40:42.664  WARN 29923 --- [           main] ockingLoadBalancerClientRibbonWarnLogger : You already have RibbonLoadBalancerClient on your classpath. It will be used by default. As Spring Cloud Ribbon is in maintenance mode. We recommend switching to BlockingLoadBalancerClient instead. In order to use it, set the value of `spring.cloud.loadbalancer.ribbon.enabled` to `false` or remove spring-cloud-starter-netflix-ribbon from your project.
2023-06-08 10:40:42.750  INFO 29923 --- [           main] o.s.c.n.eureka.InstanceInfoFactory       : Setting initial instance status as: STARTING
2023-06-08 10:40:42.782  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Initializing Eureka in region us-east-1
2023-06-08 10:40:42.935  INFO 29923 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON encoding codec LegacyJacksonJson
2023-06-08 10:40:42.935  INFO 29923 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using JSON decoding codec LegacyJacksonJson
2023-06-08 10:40:43.056  INFO 29923 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML encoding codec XStreamXml
2023-06-08 10:40:43.056  INFO 29923 --- [           main] c.n.d.provider.DiscoveryJerseyProvider   : Using XML decoding codec XStreamXml
2023-06-08 10:40:43.144  INFO 29923 --- [           main] c.n.d.s.r.aws.ConfigClusterResolver      : Resolving eureka endpoints via configuration
2023-06-08 10:40:43.161  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2023-06-08 10:40:43.161  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2023-06-08 10:40:43.161  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2023-06-08 10:40:43.161  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Application is null : false
2023-06-08 10:40:43.161  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2023-06-08 10:40:43.161  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Application version is -1: true
2023-06-08 10:40:43.161  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2023-06-08 10:40:43.457  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : The response status is 200
2023-06-08 10:40:43.458  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Starting heartbeat executor: renew interval is: 30
2023-06-08 10:40:43.460  INFO 29923 --- [           main] c.n.discovery.InstanceInfoReplicator     : InstanceInfoReplicator onDemand update allowed rate per min is 4
2023-06-08 10:40:43.463  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Discovery Client initialized at timestamp 1686192043462 with initial instances count: 0
2023-06-08 10:40:43.464  INFO 29923 --- [           main] o.s.c.n.e.s.EurekaServiceRegistry        : Registering application CLIENT1 with eureka with status UP
2023-06-08 10:40:43.464  INFO 29923 --- [           main] com.netflix.discovery.DiscoveryClient    : Saw local status change event StatusChangeEvent [timestamp=1686192043464, current=UP, previous=STARTING]
2023-06-08 10:40:43.466  INFO 29923 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_CLIENT1/192.168.2.180:client1:8902: registering service...
2023-06-08 10:40:43.492  INFO 29923 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8902 (http) with context path ''
2023-06-08 10:40:43.493  INFO 29923 --- [           main] .s.c.n.e.s.EurekaAutoServiceRegistration : Updating port to 8902
2023-06-08 10:40:43.504  INFO 29923 --- [           main] c.l.e.EurekaclientApplication            : Started EurekaclientApplication in 2.806 seconds (JVM running for 4.028)
2023-06-08 10:40:43.653  INFO 29923 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_CLIENT1/192.168.2.180:client1:8902 - registration status: 204
2023-06-08 10:41:13.464  INFO 29923 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2023-06-08 10:41:13.464  INFO 29923 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2023-06-08 10:41:13.464  INFO 29923 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2023-06-08 10:41:13.465  INFO 29923 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Application is null : false
2023-06-08 10:41:13.465  INFO 29923 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2023-06-08 10:41:13.465  INFO 29923 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Application version is -1: false
2023-06-08 10:41:13.465  INFO 29923 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2023-06-08 10:41:13.515  INFO 29923 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : The response status is 200
2023-06-08 10:41:54.752  INFO 29923 --- [nio-8902-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2023-06-08 10:41:54.752  INFO 29923 --- [nio-8902-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2023-06-08 10:41:54.758  INFO 29923 --- [nio-8902-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 6 ms
2023-06-08 10:42:07.625  INFO 29923 --- [nio-8902-exec-3] c.l.e.controller.TestController          : ***** element:client1
2023-06-08 10:42:07.628  INFO 29923 --- [nio-8902-exec-3] c.l.e.controller.TestController          : [EurekaDiscoveryClient.EurekaServiceInstance@64ca6245 instance = InstanceInfo [instanceId = 192.168.2.180:client1:8902, appName = CLIENT1, hostName = 192.168.2.180, status = UP, ipAddr = 192.168.2.180, port = 8902, securePort = 443, dataCenterInfo = com.netflix.appinfo.MyDataCenterInfo@5288f575]
```
默认是使用appname的方式，要求必须配置spring.application.name，如果不希望使用appname来访问服务，可以配置使用IP形式。

```yml
eureka:
  instance:
    prefer-ip-address: true #强制使用IP
```

以上是最基础的服务注册中心与服务提供者的搭建过程，能够启动一个服务注册中心，启动一个服务客户端，客户端向注册中心注册自己，且能够获取注册中心上的服务清单。

此外，还有高可用版本的服务注册中心，即两个及以上的eurekaserver节点，他们相互注册，服务提供者可以向两个节点注册自己的信息，任何一个服务注册节点宕机不会影响各服务节点间的调用。

以及服务消费者请求多服务提供者之间的负载均衡策略，依靠Ribbon实现。

