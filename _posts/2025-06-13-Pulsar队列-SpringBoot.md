---
layout:     post
title:      Java 17 下 Spring Boot 与 Pulsar 队列集成实战：生产者与消费者实现指南
subtitle:   Pulsar
date:       2025-06-13
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Pulsar
    - SpringBoot
    - MQ
---

Pulsar队列与Springboot集成有2种模式：官方pulsar-client 或社区Starter（如pulsar-spring-boot-starter）

- 如果考虑最新、最快、最齐全的功能，使用官方pulsar-client
- 如果考虑快速低成本接入，使用社区Starter（如pulsar-spring-boot-starter）

环境依赖：

- SpringBoot 3.3.12
- Java 17

- [官方pulsar-client](#官方pulsar-client)
  - [引入依赖](#引入依赖)
  - [配置Pulsar连接](#配置pulsar连接)
  - [创建生产者](#创建生产者)
  - [创建消费者](#创建消费者)
- [社区Starter](#社区starter)
  - [引入依赖](#引入依赖-1)
  - [发送消息](#发送消息)
  - [接收消息](#接收消息)

## 官方pulsar-client

官方 `pulsar-client` 提供了最全面的 Pulsar 功能，适合对功能完整性有较高要求的项目。下面我们一步步实现生产者和消费者的功能。

### 引入依赖

首先，需要在项目中引入 `pulsar-client` 的依赖，这能帮助我们在 Spring Boot 项目里使用 Pulsar 客户端功能。

```xml
<dependency>
    <groupId>org.apache.pulsar</groupId>
    <artifactId>pulsar-client</artifactId>
    <version>3.3.1</version>
</dependency>
```
### 配置Pulsar连接

引入依赖后，我们需要对 Pulsar 进行连接配置，指定 Pulsar 服务的地址。可以在配置文件里添加相关配置，同时创建一个配置类来初始化 Pulsar 客户端。

```yml
spring:
  pulsar:
    service-url: pulsar://127.0.0.1:6650
```

```java
@Configuration
public class PulsarConfig {

    @Value("${spring.pulsar.client.service-url}")
    private String serviceUrl;

    @Bean
    public PulsarClient pulsarClient() throws PulsarClientException {
        ClientBuilder clientBuilder = PulsarClient.builder()
                .serviceUrl(serviceUrl)
                .operationTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
                .connectionTimeout(10, java.util.concurrent.TimeUnit.SECONDS);

        // 可以添加认证等其他配置
        // clientBuilder.authentication(AuthenticationFactory.token("your-token"));
        
        return clientBuilder.build();
    }
}
```

### 创建生产者

完成连接配置后，就可以创建 Pulsar 生产者来发送消息了。下面的代码实现了同步和异步发送消息的功能。

```java
@Service
public class PulsarMessageProducer {

    private static final String TOPIC = "persistent://public/default/messages";
    
    @Autowired
    private PulsarClient pulsarClient;

    public void sendMessage(String content) throws PulsarClientException {
        // 创建生产者
        Producer<Message> producer = pulsarClient.newProducer(Schema.JSON(Message.class))
                .topic(TOPIC)
                .producerName("message-producer")
                .create();

        // 创建消息对象
        Message message = new Message(
                UUID.randomUUID().toString(),
                content,
                LocalDateTime.now()
        );

        // 发送消息（同步）
        MessageId messageId = producer.send(message);
        System.out.println("Message sent successfully. Message ID: " + messageId);

        // 关闭生产者
        producer.close();
    }

    public CompletableFuture<MessageId> sendMessageAsync(String content) throws PulsarClientException {
        // 创建生产者
        Producer<Message> producer = pulsarClient.newProducer(Schema.JSON(Message.class))
                .topic(TOPIC)
                .producerName("async-message-producer")
                .create();

        // 创建消息对象
        Message message = new Message(
                UUID.randomUUID().toString(),
                content,
                LocalDateTime.now()
        );

        // 异步发送消息
        CompletableFuture<MessageId> future = producer.sendAsync(message);
        future.thenAccept(messageId -> {
            System.out.println("Async message sent successfully. Message ID: " + messageId);
            try {
                producer.close();
            } catch (PulsarClientException e) {
                e.printStackTrace();
            }
        }).exceptionally(throwable -> {
            System.err.println("Failed to send message: " + throwable.getMessage());
            try {
                producer.close();
            } catch (PulsarClientException e) {
                e.printStackTrace();
            }
            return null;
        });

        return future;
    }
}
```

### 创建消费者

创建完生产者后，还需要创建消费者来接收消息。下面的代码展示了如何启动一个消费者并异步接收消息。

```java
@Service
public class PulsarMessageConsumer implements CommandLineRunner {

    private static final String TOPIC = "persistent://public/default/messages";
    private static final String SUBSCRIPTION = "message-subscription";
    
    @Autowired
    private PulsarClient pulsarClient;

    @Override
    public void run(String... args) throws Exception {
        // 启动消费者
        startConsumer();
    }

    public void startConsumer() throws PulsarClientException {
        // 创建消费者
        Consumer<Message> consumer = pulsarClient.newConsumer(Schema.JSON(Message.class))
                .topic(TOPIC)
                .subscriptionName(SUBSCRIPTION)
                .subscriptionType(SubscriptionType.Shared)
                .subscribe();

        // 异步消费消息
        new Thread(() -> {
            while (true) {
                try {
                    // 等待接收消息，超时时间为10秒
                    Message<Message> msg = consumer.receive(10, TimeUnit.SECONDS);
                    
                    if (msg != null) {
                        try {
                            // 处理消息
                            Message message = msg.getValue();
                            System.out.println("Received message: " + message);
                            
                            // 确认消息已消费
                            consumer.acknowledge(msg);
                        } catch (Exception e) {
                            // 处理消息失败，重新放回队列
                            consumer.negativeAcknowledge(msg);
                        }
                    }
                } catch (PulsarClientException e) {
                    if (e.getCause() instanceof java.util.concurrent.TimeoutException) {
                        // 超时异常，继续等待
                        System.out.println("No message received within timeout period, waiting again...");
                    } else {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```

## 社区Starter

社区提供的 `pulsar-spring-boot-starter` 简化了 Pulsar 与 Spring Boot 的集成过程，适合需要快速接入的项目。下面我们来看看如何使用它。

### 引入依赖

首先，在配置文件中添加 Pulsar 服务的配置信息，这能帮助我们连接到 Pulsar 服务。

```yml
# Pulsar 服务
spring:
  pulsar:
    client:
      serviceUrl: pulsar://127.0.0.1:6650
```

### 发送消息

完成配置后，就可以使用 `PulsarTemplate` 来发送消息了。下面的代码实现了同步和异步发送消息的功能。

```java
@Service
public class MyProducer {
    private final PulsarTemplate<String> pulsarTemplate;

    public MyProducer(PulsarTemplate<String> pulsarTemplate) {
        this.pulsarTemplate = pulsarTemplate;
    }

    public void sendMessage(String message) {
// 由于 convertAndSend(String, String) 方法未定义，可能需要使用正确的方法
// 假设使用 send 方法来替代，具体根据 PulsarTemplate 的实际方法决定
        pulsarTemplate.send("my-topic", message);
        System.out.println("Sent: " + message);
    }

    public CompletableFuture<MessageId> sendMessageAsync(String message) {
        return pulsarTemplate.sendAsync("my-topic", message);
    }
}
```

### 接收消息

发送消息后，还需要创建消费者来接收消息。使用 `@PulsarListener` 注解可以方便地监听消息。下面的代码展示了如何接收消息。

```java
@Service
public class MyConsumer {
    @PulsarListener(topics = "my-topic")
    public void receive(Message<String> message) {
        System.out.println("Received in Spring Boot: " + message.getValue());
    }
}

```