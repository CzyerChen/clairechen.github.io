---
layout:     post
title:      有限状态机 State Machine 入门01
subtitle:   statemachine
date:       2020-12-27
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - 有限状态机 State Machine
    - statemachine
---

## 有限状态机 State Machine 

## 一、简单程序带你入门

- 使用SpringBoot简单搭建状态机的使用，此处参照程序猿DD
- [Claire's Blog-参考代码地址]()

### 1.引入依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>springboot-forall</artifactId>
        <groupId>com.learning</groupId>
        <version>1.0.0</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>boot-for-statemachine</artifactId>

    <properties>
       <statemachine.version>2.2.0.RELEASE</statemachine.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-core</artifactId>
            <version>${statemachine.version}</version>
    </dependency>
    </dependencies>
</project>
```

### 2. 确定状态+事件的类型

- 此处以程序猿DD入门中提到的简单订单模式来DEMO

```java
public enum OrderStateEnum {
    /**
     * 待支付
     */
    UNPAID,
    /**
     * 待收货
     */
    WAITING_FOR_RECEIVE,
    /**
     * 结束
     */
    DONE
}

public enum OrderEvents {
    /**
     * 支付
     */
    PAY,
    /**
     * 收货
     */
    RECEIVE
}

```

### 3.1 statemachine配置类，手动添加listener处理

```java
@Configuration
@EnableStateMachine
public class StatemachineConfigurer extends EnumStateMachineConfigurerAdapter<OrderStateEnum, OrderEvents> {
    private Logger log = LoggerFactory.getLogger(getClass());

//定义初始状态
    @Override
    public void configure(StateMachineStateConfigurer<OrderStateEnum, OrderEvents> states)
            throws Exception {
        states.withStates()
                .initial(OrderStateEnum.UNPAID)
                .states(EnumSet.allOf(OrderStateEnum.class));
    }
//绑定其他状态变化与事件
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderStateEnum,OrderEvents> transactions) throws Exception {
        transactions
                .withExternal()
                .source(OrderStateEnum.UNPAID).target(OrderStateEnum.WAITING_FOR_RECEIVE)
                .event(OrderEvents.PAY)
                .and()
                .withExternal()
                .source(OrderStateEnum.WAITING_FOR_RECEIVE).target(OrderStateEnum.DONE)
                .event(OrderEvents.RECEIVE);
    }
//配置事件监听
    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderStateEnum,OrderEvents> config) throws Exception {
          config.withConfiguration().listener(listener());
    }
//处理事件监听
    public StateMachineListener<OrderStateEnum,OrderEvents> listener(){
        return new StateMachineListenerAdapter<OrderStateEnum, OrderEvents>(){
          @Override
          public void transition(Transition<OrderStateEnum,OrderEvents> transition){
              if(transition.getTarget().getId() == OrderStateEnum.UNPAID){
                 log.info("订单创建，待支付");
                 return;
              }
              if(transition.getSource().getId() == OrderStateEnum.UNPAID && transition.getTarget().getId() == OrderStateEnum.WAITING_FOR_RECEIVE){
                  log.info("订单完成支付，待收货");
                  return;
              }
              if(transition.getSource().getId() ==  OrderStateEnum.WAITING_FOR_RECEIVE && transition.getTarget().getId() == OrderStateEnum.DONE){
                  log.info("用户已收货，订单完成");
                  return;
              }
          }
        };
    }
}

```

### 3.2 statemachine配置类，配置事件类

```java
@Configuration
@EnableStateMachine
public class StatemachineConfigurer extends EnumStateMachineConfigurerAdapter<OrderStateEnum, OrderEvents> {
    private Logger log = LoggerFactory.getLogger(getClass());


    @Override
    public void configure(StateMachineStateConfigurer<OrderStateEnum, OrderEvents> states)
            throws Exception {
        states.withStates()
                .initial(OrderStateEnum.UNPAID)
                .states(EnumSet.allOf(OrderStateEnum.class));
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<OrderStateEnum,OrderEvents> transactions) throws Exception {
        transactions
                .withExternal()
                .source(OrderStateEnum.UNPAID).target(OrderStateEnum.WAITING_FOR_RECEIVE)
                .event(OrderEvents.PAY)
                .and()
                .withExternal()
                .source(OrderStateEnum.WAITING_FOR_RECEIVE).target(OrderStateEnum.DONE)
                .event(OrderEvents.RECEIVE);
    }
}

@WithStateMachine
public class OrderEventConfig {
    private Logger log = LoggerFactory.getLogger(getClass());

    @OnTransition(target = "UNPAID")
    public void create() {
        log.info("订单创建，待支付");
    }

    @OnTransition(source = "UNPAID", target = "WAITING_FOR_RECEIVE")
    public void pay() {
        log.info("订单已支付，待收货");
    }

    @OnTransition(source = "WAITING_FOR_RECEIVE", target = "DONE")
    public void receive() {
       log.info("用户已收货，订单完成");
    }
}

```

### 4.开始测试

```java
@SpringBootApplication
public class StateMachineTestApplication implements CommandLineRunner {
    @Autowired
    private StateMachine<OrderStateEnum, OrderEvents> stateMachine;

    public static void main(String[] args){
        SpringApplication.run(StateMachineTestApplication.class,args);
    }

    public void run(String... args) throws Exception {
        stateMachine.start();
        stateMachine.sendEvent(OrderEvents.PAY);
        stateMachine.sendEvent(OrderEvents.RECEIVE);
    }
}

```

```text
Connected to the target VM, address: '127.0.0.1:57525', transport: 'socket'

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.4.RELEASE)

2020-12-28 18:03:51.603  INFO 44699 --- [           main] c.s.StateMachineTestApplication          : Starting StateMachineTestApplication on fiboMBP004 with PID 44699 (/Users/chenzy/files/gitFile/springboot-forall/boot-for-statemachine/target/classes started by chenzy in /Users/chenzy/files/gitFile/springboot-forall)
2020-12-28 18:03:51.605  INFO 44699 --- [           main] c.s.StateMachineTestApplication          : No active profile set, falling back to default profiles: default
2020-12-28 18:03:52.322  INFO 44699 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.statemachine.config.configuration.StateMachineAnnotationPostProcessorConfiguration' of type [org.springframework.statemachine.config.configuration.StateMachineAnnotationPostProcessorConfiguration$$EnhancerBySpringCGLIB$$293eccee] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2020-12-28 18:03:52.555  INFO 44699 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8888 (http)
2020-12-28 18:03:52.563  INFO 44699 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-12-28 18:03:52.564  INFO 44699 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.38]
2020-12-28 18:03:52.639  INFO 44699 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-12-28 18:03:52.639  INFO 44699 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 992 ms
2020-12-28 18:03:53.034  INFO 44699 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8888 (http) with context path ''
2020-12-28 18:03:53.043  INFO 44699 --- [           main] c.s.StateMachineTestApplication          : Started StateMachineTestApplication in 1.755 seconds (JVM running for 2.327)
2020-12-28 18:03:53.054  INFO 44699 --- [           main] tConfig$$EnhancerBySpringCGLIB$$31e521c1 : 订单创建，待支付
2020-12-28 18:03:53.057  INFO 44699 --- [           main] o.s.s.support.LifecycleObjectSupport     : started org.springframework.statemachine.support.DefaultStateMachineExecutor@3c87fdf2
2020-12-28 18:03:53.057  INFO 44699 --- [           main] o.s.s.support.LifecycleObjectSupport     : started UNPAID WAITING_FOR_RECEIVE DONE  / UNPAID / uuid=0a30fabc-4fa0-492e-89ae-b1901e0b3294 / id=null
2020-12-28 18:03:53.060  INFO 44699 --- [           main] tConfig$$EnhancerBySpringCGLIB$$31e521c1 : 订单已支付，待收货
2020-12-28 18:03:53.061  INFO 44699 --- [           main] tConfig$$EnhancerBySpringCGLIB$$31e521c1 : 用户已收货，订单完成
Disconnected from the target VM, address: '127.0.0.1:57525', transport: 'socket'
2020-12-28 18:03:56.668  INFO 44699 --- [extShutdownHook] o.s.s.support.LifecycleObjectSupport     : stopped org.springframework.statemachine.support.DefaultStateMachineExecutor@3c87fdf2
2020-12-28 18:03:56.669  INFO 44699 --- [extShutdownHook] o.s.s.support.LifecycleObjectSupport     : stopped UNPAID WAITING_FOR_RECEIVE DONE  /  / uuid=0a30fabc-4fa0-492e-89ae-b1901e0b3294 / id=null
2020-12-28 18:03:56.670  INFO 44699 --- [extShutdownHook] o.s.s.support.LifecycleObjectSupport     : destroy called
```

