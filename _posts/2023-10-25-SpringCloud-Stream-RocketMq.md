---
layout:     post
title:      SpringCloudStreamRocketMq 实践
subtitle:   RocketMq
date:       2023-10-25
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringCloud
    - RocketMq
---

- [组件介绍](#组件介绍)
- [工作流程](#工作流程)
- [文本消息+自定义信道](#文本消息自定义信道)
- [多主题+文本消息+自定义信道](#多主题文本消息自定义信道)
- [标签过滤+获取头信息](#标签过滤获取头信息)
- [定向的异常处理与全局异常处理](#定向的异常处理与全局异常处理)
- [顺序消息](#顺序消息)
  - [全局顺序消息](#全局顺序消息)
  - [局部顺序消息](#局部顺序消息)
- [事务消息](#事务消息)

当在选取队列组件的时候，通常要结合实际情况，大数据场景Kafka可能是理想的选择，事务或延迟队列场景可能RocketMQ是较成熟的选择，其他常规业务高性能场景可能RabbitMQ是不错的选择。今天这里为了了解和使用事务和延迟队列的特性，选择研究RocketMQ。

## 组件介绍

Producer：生产者，支持分布式集群部署，支持快速产生消息并投递

Consumer：消费者，支持分布式集群部署，支持Push和Pull的模式消费数据，支持集群和广播方式消费数据

NameServer：topic注册中心，支持Broker的动态注册与发现

Broker： 负责消息的存储、投递、查询与服务高可用

其他名词：

- Topic: 主题，对消息分类
- Message: 消息体
- MessageID: 全局唯一标志，系统自动生产
- Tag: 二级消息类型，区分某个Topic下的消息分类
- Producer实例: 生产者的一个对象实例
- Consumer实例: 消费者的一个对象实例
- Group: 一类producer和consumer
- Group ID: group标识
- 队列: 每个Topic会对应一个或者多个队列来存储信息
- Exactly-Once 语义：一条消息之后能被consumer消费一次，即使重试也不会多次消费。消息队列 RocketMQ 的 Exactly-Once 投递语义适用于“接收消息 -> 处理消息 -> 结果持久化到数据库”的流程，能够保证您的每一条消息消费的最终处理结果写入到您的数据库一次且仅一次，保证消息消费的幂等。
- 集群消费：同一个groupId下的consumer平均消费，一个消息只被投递到某一个consumer中
- 广播消费：同一个groupId下的consumer各自消费，一个消息被投递到等多个consumer中
- 定时消息：指定时间将消息投递给consumer进行消费
- 延时消息：延后一段时间投递给consumer进行消费
- 事务消息：分布式事务最终一致性
- 顺序消息：按照顺序进行发布和消费
- 全局顺序消息：一种特殊的分区顺序消息，严格遵守先进先出进行发布会和消费
- 分区顺序消息：一个topic多个分区，通过shardingKey区分分区，同一个分区内遵守先进先出，多分区能够增加并发度提升性能
- 消息堆积：消费者未能在短时间内消费所有数据
- 消息过滤：消费者可以根据TAG过滤消息
- 消息轨迹：从生产者产出，到消费者消费的过程中，各个香干节点的时间、地点等数据汇聚而成的完整链路
- 重置消费位点：在消息持久化存储的时间范围内，重新设置消费进度，成功设置时间点后，由生产者发送到服务端的消息
- 死信队列：处理无法正常消费的消息，消息被初次消费失败后，会进行自动重试，重试达到上限依旧失败后，消息会被放入死信队列-Dead Letter Queue,存储死信消息的特殊队列
- 消息路由：不同地域之间的消息同步

## 工作流程

- 启动 NameServer
- 启动 Broker
- Broker/Producer/Consumer 注册至 NameServer，并彼此获取Topic等信息
- 发送消息前，创建Topic
- Producer启动，与NameServer建立长链接。从NameServere获取Broker信息，与Broker建立长链接，向Broker发送消息
- Consumer启动，与NameServer建立长链接。从NameServere获取Broker信息，与Broker建立长链接，从Broker获取消息

Spring-Cloud-Stream: 2.2.10-C1
Spring-Boot: 2.3.12.RELEASE

## 文本消息+自定义信道

```yml
server:
  port: 8090

spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
          group: rmq-grop
      bindings:
        textoutput:
          destination: text-topic
          contentType: text/plain
          group: text-group
        textinput:
          destination: text-topic
          contentType: text/plain
          group: text-group
# 日志级别
logging:
  level:
    com.alibaba.cloud.stream.binder.rocketmq: info
```

```java
public interface SelfSink {
    public static String TEXT_INPUT="textinput";

    @Input(TEXT_INPUT)
    SubscribableChannel textInput();
}

@Configuration
@EnableBinding(SelfSink.class)
public class RmqConsumerConfig {
    
}

@Service
public class ReceiveService {

    @StreamListener(value = SelfSink.TEXT_INPUT)
    public void textInput(String message) {
        System.out.println("receive content:" + message);
    }
}

@Configuration
@EnableBinding(SelfSource.class)
public class RmqProducerConfig {

}

public interface SelfSource {
    public static String TEXT_OUTPUT="textoutput";

    @Output(TEXT_OUTPUT)
    MessageChannel textOutput();
}

@Service
public class SendService {

    @Autowired
    private SelfSource source;

    public void sendText(String msg){
        Message<String> message = (Message<String>) MessageBuilder.withPayload(msg).build();
        source.textOutput().send(message);
    }
}

```

## 多主题+文本消息+自定义信道


```yml
server:
  port: 8090

spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
          group: rmq-grop
      bindings:
        textoutput:
          destination: text-topic
          contentType: text/plain
          group: text-group
        textoutput2:
          destination: text-topic2
          contentType: text/plain
          group: text-group2
        textinput:
          destination: text-topic
          contentType: text/plain
          group: text-group
        textinput2:
          destination: text-topic2
          contentType: text/plain
          group: text-group2
# 日志级别
logging:
  level:
    com.alibaba.cloud.stream.binder.rocketmq: info
```

```java
public interface SelfSource {
    public static String TEXT_OUTPUT="textoutput";
    public static String TEXT_OUTPUT2="textoutput2";

    @Output(TEXT_OUTPUT)
    MessageChannel textOutput();

    @Output(TEXT_OUTPUT2)
    MessageChannel textOutput2();
}

@Service
public class SendService {

    @Autowired
    private SelfSource source;

    public void sendText(String msg){
        Message<String> message = (Message<String>) MessageBuilder.withPayload(msg).build();
        source.textOutput().send(message);
    }

    public void sendText2(String msg){
        Message<String> message = (Message<String>) MessageBuilder.withPayload(msg).build();
        source.textOutput2().send(message);
    }
}

public interface SelfSink {
    public static String TEXT_INPUT="textinput";
    public static String TEXT_INPUT2="textinput2";

    @Input(TEXT_INPUT)
    SubscribableChannel textInput();

    @Input(TEXT_INPUT2)
    SubscribableChannel textInput2();
}

@Service
public class ReceiveService {

    @StreamListener(value = SelfSink.TEXT_INPUT)
    public void textInput(String message) {
        System.out.println("receive group1 content:" + message);
    }

    @StreamListener(value = SelfSink.TEXT_INPUT2)
    public void textInput2(String message) {
        System.out.println("receive group2 content:" + message);
    }
}


```

## 标签过滤+获取头信息

```yml
server:
  port: 8090

spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
          group: rmq-group
        bindings:
          textinput3:
            consumer:
              subscription: tagfilter
      bindings:
        textoutput2:
          destination: text-topic2
          contentType: text/plain
          group: text-group2
        textinput3:
          destination: text-topic2
          contentType: text/plain
          group: text-group2
# 日志级别
logging:
  level:
    com.alibaba.cloud.stream.binder.rocketmq: info
```

```java
public interface SelfSource {
    public static String TEXT_OUTPUT2="textoutput2";

    @Output(TEXT_OUTPUT2)
    MessageChannel textOutput2();
}

@Service
public class SendService {

    @Autowired
    private SelfSource source;

    public void sendText(String msg) {
        Message<String> message = (Message<String>) MessageBuilder.withPayload(msg).build();
        source.textOutput().send(message);
    }

    public void sendText2(String msg) {
        Message<String> message = (Message<String>) MessageBuilder.withPayload(msg).build();
        source.textOutput2().send(message);
    }

    public void sendWithTag(String msg, String tag) {
        Message<String> message = MessageBuilder.withPayload(msg)
                .setHeader(MessageConst.PROPERTY_TAGS, tag)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build();
        source.textOutput2().send(message);
    }
}

public interface SelfSink {
    public static String TEXT_INPUT="textinput";
    public static String TEXT_INPUT2="textinput2";
    public static String TEXT_INPUT3="textinput3";

    @Input(TEXT_INPUT3)
    SubscribableChannel textInput3();
}

@Service
public class ReceiveService {

    @StreamListener(value = SelfSink.TEXT_INPUT3,condition = "headers['ROCKET_TAGS'] == 'tagfilter'")
    public void textInput3WithTag(String message, @Headers Map headers, @Header(name = "ROCKET_TAGS")String name) {
        System.out.println("receive group2 tagfilter content:" + message +", headers="+headers +", name="+name);
    }
}
```

```log
send tagfilter group2 index:0
send tagfilter group2 index:1
send tagfilter group2 index:2
send tagfilter group2 index:3
receive group2 tagfilter content:group2-index:0, headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698235938769, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001C9F42C13DA157FEE87CD0000, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=c13ac4ee-f498-a558-c6b7-30ed7563220f, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=3, timestamp=1698235938796}, name=tagfilter
receive group2 tagfilter content:group2-index:1, headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698235938793, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001C9F42C13DA157FEE87E90002, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=df97a997-8437-c3de-7802-80313be691ed, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=0, timestamp=1698235938818}, name=tagfilter
send tagfilter group2 index:4
receive group2 tagfilter content:group2-index:4, headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698235938816, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001C9F42C13DA157FEE88000009, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=020522c8-6c29-b4db-c3ca-9987bd108625, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=3, timestamp=1698235938825}, name=tagfilter
receive group2 tagfilter content:group2-index:3, headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698235938808, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001C9F42C13DA157FEE87F80007, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=aa7d5d49-4178-f621-38ed-70b77bbfffc3, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=2, timestamp=1698235939676}, name=tagfilter
receive group2 tagfilter content:group2-index:2, headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698235938801, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001C9F42C13DA157FEE87F10005, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=76d60399-c127-adab-e262-59d01f50baab, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=1, timestamp=1698235939677}, name=tagfilter
```

## 定向的异常处理与全局异常处理

```java
  @StreamListener(value = SelfSink.TEXT_INPUT)
    public void textInput(String message) {
        System.out.println("receive group1 content:" + message);
        throw new IllegalStateException("异常信息1");
    }

    @StreamListener(value = SelfSink.TEXT_INPUT3,condition = "headers['ROCKET_TAGS'] == 'tagfilter'")
    public void textInput3WithTag(String message, @Headers Map headers, @Header(name = "ROCKET_TAGS")String name) {
        System.out.println("receive group2 tagfilter content:" + message +", headers="+headers +", name="+name);
        throw new IllegalArgumentException("异常信息3");
    }

    @ServiceActivator(inputChannel = "text-topic.text-group.errors")
    public void handleError(ErrorMessage errorMessage){
        Throwable throwable = errorMessage.getPayload();
        System.out.println("定向异常："+throwable);
        Message<?> originalMessage = errorMessage.getOriginalMessage();
        System.out.println(Objects.nonNull(originalMessage)?originalMessage:"空");
    }

    @StreamListener("errorChannel")
    public void error(Message<?> message){
        ErrorMessage errorMessage = (ErrorMessage)message;
        System.out.println("其他异常:"+errorMessage);
    }
```

```log
send group1 index:0
receive group1 content:group1-index:0
send tagfilter group2 index:0
receive group2 tagfilter content:group2-index:0, headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698236994357, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001D1122C13DA157FFEA3350003, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=ff93d975-c880-fbf0-0168-6b6e0fab9653, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=0, timestamp=1698236995233}, name=tagfilter
receive group1 content:group1-index:0
receive group2 tagfilter content:group2-index:0, headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698236994357, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001D1122C13DA157FFEA3350003, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=ff93d975-c880-fbf0-0168-6b6e0fab9653, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=0, timestamp=1698236995233}, name=tagfilter
receive group1 content:group1-index:0
INFO 53522 --- [ad_text-group_1] o.s.i.h.s.MessagingMethodInvokerHelper   : Overriding default instance of MessageHandlerMethodFactory with provided one.
定向异常：org.springframework.messaging.MessagingException: Exception thrown while invoking ReceiveService#textInput[1 args]; nested exception is java.lang.IllegalStateException: 异常信息1, failedMessage=GenericMessage [payload=byte[14], headers={ROCKET_MQ_BORN_TIMESTAMP=1698236994326, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001D1122C13DA157FFEA3130000, ROCKET_MQ_TOPIC=text-topic, ROCKET_MQ_BORN_HOST=172.17.0.1, id=5672d0c2-04ae-6012-ae06-5cd84c2781fb, ROCKET_MQ_SYS_FLAG=0, contentType=text/plain, ROCKET_MQ_QUEUE_ID=3, timestamp=1698236994353}]
空
receive group2 tagfilter content:group2-index:0, headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698236994357, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001D1122C13DA157FFEA3350003, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=ff93d975-c880-fbf0-0168-6b6e0fab9653, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=0, timestamp=1698236995233}, name=tagfilter
其他异常:ErrorMessage [payload=org.springframework.messaging.MessagingException: Exception thrown while invoking ReceiveService#textInput3WithTag[3 args]; nested exception is java.lang.IllegalArgumentException: 异常信息3, failedMessage=GenericMessage [payload=byte[14], headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698236994357, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001D1122C13DA157FFEA3350003, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=ff93d975-c880-fbf0-0168-6b6e0fab9653, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=0, timestamp=1698236995233}], headers={id=266dcc54-fb75-0a90-e456-917b8743ff5b, timestamp=1698236998247}]
ERROR 53522 --- [d_text-group2_1] o.s.integration.handler.LoggingHandler   : org.springframework.messaging.MessagingException: Exception thrown while invoking ReceiveService#textInput3WithTag[3 args]; nested exception is java.lang.IllegalArgumentException: 异常信息3, failedMessage=GenericMessage [payload=byte[14], headers={ROCKET_TAGS=tagfilter, ROCKET_MQ_BORN_TIMESTAMP=1698236994357, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001D1122C13DA157FFEA3350003, ROCKET_MQ_TOPIC=text-topic2, ROCKET_MQ_BORN_HOST=172.17.0.1, id=ff93d975-c880-fbf0-0168-6b6e0fab9653, ROCKET_MQ_SYS_FLAG=0, contentType=application/json, ROCKET_MQ_QUEUE_ID=0, timestamp=1698236995233}]
	at org.springframework.cloud.stream.binding.StreamListenerMessageHandler.handleRequestMessage(StreamListenerMessageHandler.java:64)
	at org.springframework.integration.handler.AbstractReplyProducingMessageHandler.handleMessageInternal(AbstractReplyProducingMessageHandler.java:134)
	at org.springframework.integration.handler.AbstractMessageHandler.handleMessage(AbstractMessageHandler.java:69)
	at org.springframework.cloud.stream.binding.DispatchingStreamListenerMessageHandler.handleRequestMessage(DispatchingStreamListenerMessageHandler.java:96)
```

获取异常消息体内容：
尝试关闭springcloudstream自带的重试机制，能够实现。
上面步骤没有特殊配置，默认遇到异常会进行重新投递，导致`System.out.println(Objects.nonNull(originalMessage)?originalMessage:"空");`始终为空，无法获取原始信息。调整如下配置后：

```yml
server:
  port: 8090

spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
          group: rmq-group
        bindings:
          textinput3:
            consumer:
              subscription: tagfilter
      bindings:
        textoutput:
          destination: text-topic
          contentType: text/plain
          group: text-group
        textoutput2:
          destination: text-topic2
          contentType: text/plain
          group: text-group2
        textinput:
          destination: text-topic
          contentType: text/plain
          group: text-group
          consumer:
            maxAttempts: 1 #--> 默认是3,1表示不重试
        textinput2:
          destination: text-topic2
          contentType: text/plain
          group: text-group2
        textinput3:
          destination: text-topic2
          contentType: text/plain
          group: text-group2
# 日志级别
logging:
  level:
    com.alibaba.cloud.stream.binder.rocketmq: info
```

可以获取原始异常

```java
 @ServiceActivator(inputChannel = "text-topic.text-group.errors")
    public void handleError(ErrorMessage errorMessage){
        Throwable throwable = errorMessage.getPayload();
        System.out.println("定向异常："+throwable);
        Message<?> originalMessage = errorMessage.getOriginalMessage();
//        System.out.println(Objects.nonNull(originalMessage)?"处理定向异常原始信息: "+ originalMessage :"无");
        assert originalMessage != null;
        System.out.println("处理定向异常原始信息："+new String((byte[])originalMessage.getPayload()));
    }
```

```log
定向异常：org.springframework.messaging.MessagingException: Exception thrown while invoking ReceiveService#textInput[1 args]; nested exception is java.lang.IllegalStateException: 异常信息1, failedMessage=GenericMessage [payload=byte[14], headers={ROCKET_MQ_BORN_TIMESTAMP=1698237547339, ROCKET_MQ_FLAG=0, ROCKET_MQ_MESSAGE_ID=C6120001D46F2C13DA15800713490000, ROCKET_MQ_TOPIC=text-topic, ROCKET_MQ_BORN_HOST=172.17.0.1, id=1d05982a-38f7-5208-076d-7b84a2555bf1, ROCKET_MQ_SYS_FLAG=0, contentType=text/plain, ROCKET_MQ_QUEUE_ID=0, timestamp=1698237547362}]
处理定向异常原始信息：group1-index:0
```

## 顺序消息

### 全局顺序消息

```yaml
server:
  port: 8090

spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876
          group: rmq-group
        bindings:
          pojooutput:
            producer:
              sendType: Sync
              group: pojo-group
          pojoinput:
            consumer:
              orderly: true #这个参数的写法和你当前使用的版本息息相关，具体见官方文档
      bindings:
        pojooutput:
          destination: pojo-topic
          contentType: application/json
          group: pojo-group
        pojoinput:
          destination: pojo-topic
          contentType: application/json
          group: pojo-group
# 日志级别
logging:
  level:
    com.alibaba.cloud.stream.binder.rocketmq: info
```

```java
public class MessageDto {
    private static final long serialVersionUID =1L;

    private String index;
    private String title;
    private String content;

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public String getIndex() {
        return index;
    }

    public void setIndex(String index) {
        this.index = index;
    }

    @Override
    public String toString() {
        return "MessageDto{" +
                "index='" + index + '\'' +
                ", title='" + title + '\'' +
                ", content='" + content + '\'' +
                '}';
    }
}

@Component
public class ProducerRunner implements CommandLineRunner {
    @Autowired
    private SendService sendService;

    @Override
    public void run(String... args) throws Exception {
        for (int i = 0; i < 5; i++) {
            MessageDto messageDto = new MessageDto();
            messageDto.setIndex("num:" + i);
            messageDto.setTitle("title");
            messageDto.setContent("content");
            sendService.sendPojoOrderly(messageDto);
        }

    }
}

public interface SelfSource {
    public static String POJO_OUTPUT="pojooutput";

    @Output(POJO_OUTPUT)
    MessageChannel pojoOutput();
}

@Service
public class SendService {

    @Autowired
    private SelfSource source;

    public void sendPojoOrderly(MessageDto messageDto){
        Message<MessageDto> message = MessageBuilder.withPayload(messageDto)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON)
                .build();
        System.out.println(message);
        source.pojoOutput().send(message);
    }
}

public interface SelfSink {
    public static String POJO_INPUT="pojoinput";

    @Input(POJO_INPUT)
    SubscribableChannel pojoInput();
}

@Service
public class ReceiveService {

    @StreamListener(value = SelfSink.POJO_INPUT)
    public void pojoInout(@Payload MessageDto messageDto){
        System.out.println("[onMessage][线程编号:{} 消息内容：{}]" + Thread.currentThread().getId()+", "+messageDto.toString());
    } 
}    
```

测试效果：正常发送，并且按照顺序消费(不加orderly就是乱序)

```log
GenericMessage [payload=MessageDto{index='num:0', title='title', content='content'}, headers={contentType=application/json, id=00641aae-71d2-3a01-35f4-b3ec42886e9c, timestamp=1698745880467}]
GenericMessage [payload=MessageDto{index='num:1', title='title', content='content'}, headers={contentType=application/json, id=4d4a8cc2-d861-cf76-f0f5-33fdc06b2942, timestamp=1698745880521}]
GenericMessage [payload=MessageDto{index='num:2', title='title', content='content'}, headers={contentType=application/json, id=009ebc60-de06-adda-c44e-82faaae155c7, timestamp=1698745880525}]
GenericMessage [payload=MessageDto{index='num:3', title='title', content='content'}, headers={contentType=application/json, id=cad3af6c-df76-bc54-49c5-61cd27ef5b08, timestamp=1698745880528}]
GenericMessage [payload=MessageDto{index='num:4', title='title', content='content'}, headers={contentType=application/json, id=9beea414-7a78-a2e6-58f8-27729a2c7300, timestamp=1698745880531}]
[onMessage][线程编号:{} 消息内容：{}]119, MessageDto{index='num:2', title='title', content='content'}
[onMessage][线程编号:{} 消息内容：{}]119, MessageDto{index='num:0', title='title', content='content'}
[onMessage][线程编号:{} 消息内容：{}]119, MessageDto{index='num:4', title='title', content='content'}
[onMessage][线程编号:{} 消息内容：{}]119, MessageDto{index='num:1', title='title', content='content'}
[onMessage][线程编号:{} 消息内容：{}]119, MessageDto{index='num:3', title='title', content='content'}
```

配置顺序消费后，rocketmq内部同一个topic有多个queue，默认4个，每个queue中的数据是有序的，从日志打印的表面上看还是乱的实则已经队列内有序
但是如果想看到依据某一个标准，看到绝对的有序，可以通过分区有序来观测

### 局部顺序消息

计划分配两个分区，依据index【0,1】字段来分流到不同的队列，从而看出队列内的有序情况

```yaml
server:
  port: 8090

spring:
  cloud:
    stream:
      bindings:
        pojooutput: 
          destination: pojo-topic 
          content-type: application/json 
          group: pojo-group 
          producer:
            partitionCount: 2 #消息生产需要广播的消费者数量。即消息分区的数量
            partitionKeyExpression: payload.index #分区 key 表达式。该表达式基于 Spring EL，从消息中获得分区 key
        pojoinput:
          destination: pojo-topic
          content-type: application/json 
          group: pojo-group 
          consumer:
            partitioned: true 

      rocketmq:
        binder:
          name-server: 127.0.0.1:9876 
          group: rmq-group
        bindings:
          pojooutput:
            producer:
              sendMsgTimeout: 3000 
              sendType: Sync 
          pojoinput:
            consumer:
              enabled: true 
              subscription: myTag||look 
              messageModel: CLUSTERING 
              push:
                orderly: true
      instance-count: 2 
      instance-index: 0 
# 日志级别
logging:
  level:
    com.alibaba.cloud.stream.binder.rocketmq: info
```

最终能看到数据被index分区，每一个区里面消费是顺序的

```log
GenericMessage [payload=MessageDto{index='0', title='title0', content='content'}, headers={id=e06aef14-e094-ab5e-32a6-c8ba574eb7cf, contentType=application/json, TAGS=myTag, timestamp=1698810537731}]
GenericMessage [payload=MessageDto{index='0', title='title1', content='content'}, headers={id=bdb2d72e-4502-380a-6324-e49af58e6082, contentType=application/json, TAGS=myTag, timestamp=1698810537790}]
GenericMessage [payload=MessageDto{index='0', title='title2', content='content'}, headers={id=5b2ea555-a3f5-648f-99e5-ac4183fd939c, contentType=application/json, TAGS=myTag, timestamp=1698810537794}]
GenericMessage [payload=MessageDto{index='0', title='title3', content='content'}, headers={id=ba9ff31d-68ef-bcdb-3e86-f92cb88804c2, contentType=application/json, TAGS=myTag, timestamp=1698810537796}]
GenericMessage [payload=MessageDto{index='1', title='title4', content='content'}, headers={id=1c4b47c3-3f65-cf65-fa8e-143d873602f3, contentType=application/json, TAGS=myTag, timestamp=1698810537800}]
GenericMessage [payload=MessageDto{index='1', title='title5', content='content'}, headers={id=030732b8-1792-5899-b704-90462774ee76, contentType=application/json, TAGS=myTag, timestamp=1698810537803}]
GenericMessage [payload=MessageDto{index='1', title='title6', content='content'}, headers={id=2e4cdcc9-1b46-fddc-9df8-1c000d66648c, contentType=application/json, TAGS=myTag, timestamp=1698810537808}]
GenericMessage [payload=MessageDto{index='1', title='title7', content='content'}, headers={id=9f45f277-bab6-9abe-1618-6755ec3715b4, contentType=application/json, TAGS=myTag, timestamp=1698810537811}]
[onMessage][线程编号:{} 消息内容：{}]120, MessageDto{index='1', title='title4', content='content'}
[onMessage][线程编号:{} 消息内容：{}]120, MessageDto{index='1', title='title5', content='content'}
[onMessage][线程编号:{} 消息内容：{}]120, MessageDto{index='1', title='title6', content='content'}
[onMessage][线程编号:{} 消息内容：{}]120, MessageDto{index='1', title='title7', content='content'}
[onMessage][线程编号:{} 消息内容：{}]120, MessageDto{index='0', title='title0', content='content'}
[onMessage][线程编号:{} 消息内容：{}]120, MessageDto{index='0', title='title1', content='content'}
[onMessage][线程编号:{} 消息内容：{}]120, MessageDto{index='0', title='title2', content='content'}
[onMessage][线程编号:{} 消息内容：{}]120, MessageDto{index='0', title='title3', content='content'}
```


## 事务消息

自定义listener，来处理事务的执行、事务的提交或回滚

```yaml
server:
  port: 8090

spring:
  cloud:
    stream:
      bindings:
        transoutput:
          destination: trans-topic
          content-type: application/json
          group: trans-group
        transinput:
          destination: trans-topic
          content-type: application/json
          group: trans-group
      rocketmq:
        binder:
          name-server: 127.0.0.1:9876 
          group: rmq-group 
        bindings:
          transoutput:
            producer:
              producerType: Trans 
              transactionListener: rocketMQTransactionListener 
              group: trans-group
          transinput:
            consumer:
              group: trans-group
# 日志级别
logging:
  level:
    com.alibaba.cloud.stream.binder.rocketmq: info
```

```java
public class TransData {
    private String param1;
    private String param2;

    public String getParam1() {
        return param1;
    }

    public void setParam1(String param1) {
        this.param1 = param1;
    }

    public String getParam2() {
        return param2;
    }

    public void setParam2(String param2) {
        this.param2 = param2;
    }

    @Override
    public String toString() {
        return "TransData{" +
                "param1='" + param1 + '\'' +
                ", param2='" + param2 + '\'' +
                '}';
    }
}

public interface SelfSource {
    public static String TRANS_OUTPUT="transoutput";

    @Output(TRANS_OUTPUT)
    MessageChannel transOutput();
}

@Service
public class SendService {

    @Autowired
    private SelfSource source;

 public void sendTrans(String transMessage) {
        TransData data = new TransData();
        data.setParam1("data1");
        data.setParam2("data2");
        Message<String> message = MessageBuilder.withPayload(transMessage)
                .setHeader("args", JSON.toJSONString(data))
                .build();
        source.transOutput().send(message);
    }
}

public interface SelfSink {
    public static String TRANS_INPUT="transinput";

    @Input(TRANS_INPUT)
    SubscribableChannel transInput();
}

@Component
public class RocketMQTransactionListener implements TransactionListener {

    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        System.out.println("=========本地事务开始执行=============");
        String message = new String(msg.getBody());
        System.out.println("原始消息体：{}"+message);
        String tid = msg.getTransactionId();
        System.out.println("事务消息id：{}"+ tid);

        //模拟执行本地事务begin=======
        System.out.println("本地事务执行参数，start......----------------------");
        TransData args = JSON.parseObject(msg.getProperty("args"), TransData.class);
        //rollback, commit or unknown
        System.out.println("[executeLocalTransaction][执行本地事务，消息：{} args：{}]"+ msg+"--"+ args.toString());
        System.out.println("本地事务执行参数，end......------------------------");
        //模拟执行本地事务end========
        //TODO 根据本地事务执行结果返回
        //LocalTransactionState.COMMIT_MESSAGE 二次确认消息，然后消费者可以消费
        //LocalTransactionState.ROLLBACK_MESSAGE 回滚消息，Broker端会删除半消息
        //LocalTransactionState.UNKNOW Broker端会进行回查消息
        return LocalTransactionState.UNKNOW;
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        System.out.println("==========回查接口=========");
        //rollback, commit or unknown
        System.out.println("[checkLocalTransaction][回查消息：{}]"+ msg);
        String tid = msg.getTransactionId();
        System.out.println("[checkLocalTransaction][事务消息id：{}]"+ tid);
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}

public interface SelfSink {
    public static String TRANS_INPUT="transinput";

    @Input(TRANS_INPUT)
    SubscribableChannel transInput();
}

@Service
public class ReceiveService {
    @StreamListener(value = SelfSink.TRANS_INPUT)
    public void transInput(String message,@Header(name = "args")String args){
        System.out.println("receive trans content:" + message +", args="+args);
    }
}
```

测试情况：

```log
=========本地事务开始执行=============
原始消息体：{}transdata
事务消息id：{}C6120001B9C9077556FD02F051300000
本地事务执行参数，start......----------------------
[executeLocalTransaction][执行本地事务，消息：{} args：{}]Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, TRAN_MSG=true, id=5283b92f-8e67-c398-a63b-fd90ee7b577e, UNIQ_KEY=C6120001B9C9077556FD02F051300000, WAIT=true, contentType=application/json, PGROUP=trans-group, timestamp=1698817303846}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F051300000'}--TransData{param1='data1', param2='data2'}
本地事务执行参数，end......------------------------
=========本地事务开始执行=============
原始消息体：{}transdata
事务消息id：{}C6120001B9C9077556FD02F051440002
本地事务执行参数，start......----------------------
[executeLocalTransaction][执行本地事务，消息：{} args：{}]Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, TRAN_MSG=true, id=0cebbc2e-f6af-b6ab-a013-bc19d3d1b670, UNIQ_KEY=C6120001B9C9077556FD02F051440002, WAIT=true, contentType=application/json, PGROUP=trans-group, timestamp=1698817303875}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F051440002'}--TransData{param1='data1', param2='data2'}
本地事务执行参数，end......------------------------
=========本地事务开始执行=============
原始消息体：{}transdata
事务消息id：{}C6120001B9C9077556FD02F051470004
本地事务执行参数，start......----------------------
[executeLocalTransaction][执行本地事务，消息：{} args：{}]Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, TRAN_MSG=true, id=1ba5b5ae-c041-558f-b97e-5eff2093f477, UNIQ_KEY=C6120001B9C9077556FD02F051470004, WAIT=true, contentType=application/json, PGROUP=trans-group, timestamp=1698817303879}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F051470004'}--TransData{param1='data1', param2='data2'}
本地事务执行参数，end......------------------------
=========本地事务开始执行=============
原始消息体：{}transdata
事务消息id：{}C6120001B9C9077556FD02F0514D0006
本地事务执行参数，start......----------------------
[executeLocalTransaction][执行本地事务，消息：{} args：{}]Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, TRAN_MSG=true, id=f03f772b-0a49-d85c-4a53-9a2dbaf91c17, UNIQ_KEY=C6120001B9C9077556FD02F0514D0006, WAIT=true, contentType=application/json, PGROUP=trans-group, timestamp=1698817303885}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F0514D0006'}--TransData{param1='data1', param2='data2'}
本地事务执行参数，end......------------------------
=========本地事务开始执行=============
原始消息体：{}transdata
事务消息id：{}C6120001B9C9077556FD02F051520008
本地事务执行参数，start......----------------------
[executeLocalTransaction][执行本地事务，消息：{} args：{}]Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, TRAN_MSG=true, id=3bc809a8-2f33-40cb-edaf-9b69c68282c8, UNIQ_KEY=C6120001B9C9077556FD02F051520008, WAIT=true, contentType=application/json, PGROUP=trans-group, timestamp=1698817303890}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F051520008'}--TransData{param1='data1', param2='data2'}
本地事务执行参数，end......------------------------
==========回查接口=========
[checkLocalTransaction][回查消息：{}]MessageExt [brokerName=null, queueId=1, storeSize=416, queueOffset=18, sysFlag=0, bornTimestamp=1698817303885, bornHost=/172.17.0.1:60840, storeTimestamp=1698817303907, storeHost=/127.0.0.1:10911, msgId=7F00000100002A9F000000000012DDF4, commitLogOffset=1236468, bodyCRC=147427201, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, REAL_TOPIC=trans-topic, TRANSACTION_CHECK_TIMES=1, TRAN_MSG=true, id=f03f772b-0a49-d85c-4a53-9a2dbaf91c17, UNIQ_KEY=C6120001B9C9077556FD02F0514D0006, CLUSTER=DefaultRmqCluster, contentType=application/json, PGROUP=trans-group, WAIT=false, timestamp=1698817303885, REAL_QID=1}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F0514D0006'}]
[checkLocalTransaction][事务消息id：{}]C6120001B9C9077556FD02F0514D0006
==========回查接口=========
[checkLocalTransaction][回查消息：{}]MessageExt [brokerName=null, queueId=3, storeSize=416, queueOffset=16, sysFlag=0, bornTimestamp=1698817303876, bornHost=/172.17.0.1:60840, storeTimestamp=1698817303897, storeHost=/127.0.0.1:10911, msgId=7F00000100002A9F000000000012DA9C, commitLogOffset=1235612, bodyCRC=147427201, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, REAL_TOPIC=trans-topic, TRANSACTION_CHECK_TIMES=1, TRAN_MSG=true, id=0cebbc2e-f6af-b6ab-a013-bc19d3d1b670, UNIQ_KEY=C6120001B9C9077556FD02F051440002, CLUSTER=DefaultRmqCluster, contentType=application/json, PGROUP=trans-group, WAIT=false, timestamp=1698817303875, REAL_QID=3}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F051440002'}]
[checkLocalTransaction][事务消息id：{}]C6120001B9C9077556FD02F051440002
==========回查接口=========
[checkLocalTransaction][回查消息：{}]MessageExt [brokerName=null, queueId=0, storeSize=416, queueOffset=17, sysFlag=0, bornTimestamp=1698817303879, bornHost=/172.17.0.1:60840, storeTimestamp=1698817303902, storeHost=/127.0.0.1:10911, msgId=7F00000100002A9F000000000012DC48, commitLogOffset=1236040, bodyCRC=147427201, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, REAL_TOPIC=trans-topic, TRANSACTION_CHECK_TIMES=1, TRAN_MSG=true, id=1ba5b5ae-c041-558f-b97e-5eff2093f477, UNIQ_KEY=C6120001B9C9077556FD02F051470004, CLUSTER=DefaultRmqCluster, contentType=application/json, PGROUP=trans-group, WAIT=false, timestamp=1698817303879, REAL_QID=0}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F051470004'}]
[checkLocalTransaction][事务消息id：{}]C6120001B9C9077556FD02F051470004
==========回查接口=========
[checkLocalTransaction][回查消息：{}]MessageExt [brokerName=null, queueId=2, storeSize=416, queueOffset=15, sysFlag=0, bornTimestamp=1698817303860, bornHost=/172.17.0.1:60840, storeTimestamp=1698817303887, storeHost=/127.0.0.1:10911, msgId=7F00000100002A9F000000000012D8F0, commitLogOffset=1235184, bodyCRC=147427201, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, REAL_TOPIC=trans-topic, TRANSACTION_CHECK_TIMES=1, TRAN_MSG=true, id=5283b92f-8e67-c398-a63b-fd90ee7b577e, UNIQ_KEY=C6120001B9C9077556FD02F051300000, CLUSTER=DefaultRmqCluster, contentType=application/json, PGROUP=trans-group, WAIT=false, timestamp=1698817303846, REAL_QID=2}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F051300000'}]
[checkLocalTransaction][事务消息id：{}]C6120001B9C9077556FD02F051300000
==========回查接口=========
[checkLocalTransaction][回查消息：{}]MessageExt [brokerName=null, queueId=2, storeSize=416, queueOffset=19, sysFlag=0, bornTimestamp=1698817303890, bornHost=/172.17.0.1:60840, storeTimestamp=1698817303912, storeHost=/127.0.0.1:10911, msgId=7F00000100002A9F000000000012DFA0, commitLogOffset=1236896, bodyCRC=147427201, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='trans-topic', flag=0, properties={args={"param1":"data1","param2":"data2"}, REAL_TOPIC=trans-topic, TRANSACTION_CHECK_TIMES=1, TRAN_MSG=true, id=3bc809a8-2f33-40cb-edaf-9b69c68282c8, UNIQ_KEY=C6120001B9C9077556FD02F051520008, CLUSTER=DefaultRmqCluster, contentType=application/json, PGROUP=trans-group, WAIT=false, timestamp=1698817303890, REAL_QID=2}, body=[116, 114, 97, 110, 115, 100, 97, 116, 97], transactionId='C6120001B9C9077556FD02F051520008'}]
[checkLocalTransaction][事务消息id：{}]C6120001B9C9077556FD02F051520008
receive trans content:transdata, args={"param1":"data1","param2":"data2"}
receive trans content:transdata, args={"param1":"data1","param2":"data2"}
receive trans content:transdata, args={"param1":"data1","param2":"data2"}
receive trans content:transdata, args={"param1":"data1","param2":"data2"}
receive trans content:transdata, args={"param1":"data1","param2":"data2"}
```
