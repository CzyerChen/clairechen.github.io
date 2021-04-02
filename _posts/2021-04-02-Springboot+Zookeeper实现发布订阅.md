---
layout:     post
title:      Springboot Zookeeper 
subtitle:   Springboot Zookeeper 
date:       2021-04-02
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Springboot
    - Zookeeper 
---

## Zookeeper & Springboot 使用介绍

基础环境说明：

- SpringBoot : 2.3.4.RELEASE
- curator: 5.1.0

具体功能内容：

- 锁
- 选举
- Barrier
- 缓存
- 持久化节点
- 队列

### 一、Zookeeper基本介绍

### 二、基本操作

- create
- ls
- get
- set
- delete

```java
public class CuratorClientTest {
    private static final String ZK_ADDRESS = "127.0.0.1:2181";
    private static final String ZK_PATH = "/zktest1";

    public static void main(String[] args) throws Exception {
        // Connect to zk
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                ZK_ADDRESS,
                new RetryNTimes(10, 5000)
        );
        client.start();
        System.out.println("zk client start successfully!");

        // Client API test
        //Create node
        String content = "hello";
        print("create", ZK_PATH, content);
        client.create().creatingParentsIfNeeded().forPath(ZK_PATH, content.getBytes());

        // Get node and data
        print("ls", "/");
        print(client.getChildren().forPath("/"));
        print("get", ZK_PATH);
        print(client.getData().forPath(ZK_PATH));

        // Modify data
        String content2 = "world";
        print("set", ZK_PATH, content2);
        client.setData().forPath(ZK_PATH, content2.getBytes());
        print("get", ZK_PATH);
        print(client.getData().forPath(ZK_PATH));

        //Remove node
        print("delete", ZK_PATH);
        client.delete().forPath(ZK_PATH);
        print("ls", "/");
        print(client.getChildren().forPath("/"));
    }

    private static void print(String... cmds) {
        StringBuilder text = new StringBuilder("$ ");
        for (String cmd : cmds) {
            text.append(cmd).append(" ");
        }
        System.out.println(text.toString());
    }

    private static void print(Object result) {
        System.out.println(
                result instanceof byte[]
                        ? new String((byte[]) result)
                        : result);
    }

}
```

### 三、监听器

- Node Cache：监听当前节点的操作
- Path Cache：监听其子节点的操作
- Tree Cache：Node Cache+Path Cache，监听当前节点+及其子节点的操作

以下展示与SpringBoot的组合

1. pom.xml，引入Curator客户端依赖

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

    <artifactId>boot-for-zookeeper</artifactId>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>5.1.0</version>
        </dependency>
  
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>5.1.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>5.1.0</version>
        </dependency>
        
    </dependencies>

</project>
```

2. application.properties 配置连接参数

```xml
server.port=2182
spring.application.name=spring-boot-zookeeper
zookeeper.connectString=localhost:2181
zookeeper.maxRetries=3 
zookeeper.baseSleepTimeMs=1000
```

3. 步骤2对应的配置类

```java
@Configuration
@ConfigurationProperties(prefix = "zookeeper")
public class ZookeeperConfig {
    private String connectString;
    private Integer maxRetries;
    private Integer baseSleepTimeMs;

    public String getConnectString() {
        return connectString;
    }

    public void setConnectString(String connectString) {
        this.connectString = connectString;
    }

    public Integer getMaxRetries() {
        return maxRetries;
    }

    public void setMaxRetries(Integer maxRetries) {
        this.maxRetries = maxRetries;
    }

    public Integer getBaseSleepTimeMs() {
        return baseSleepTimeMs;
    }

    public void setBaseSleepTimeMs(Integer baseSleepTimeMs) {
        this.baseSleepTimeMs = baseSleepTimeMs;
    }
}

```

4. Client工厂类

```java
@Component
public class ZookeeperFactory implements FactoryBean<CuratorFramework> {
    private ZookeeperConfig zookeeperConfig;
    private CuratorFramework curatorClient;

    @PostConstruct
    public void init() {
        ExponentialBackoffRetry backoffRetry = new ExponentialBackoffRetry(zookeeperConfig.getMaxRetries(), zookeeperConfig.getBaseSleepTimeMs());
        curatorClient = CuratorFrameworkFactory.builder().connectString(zookeeperConfig.getConnectString()).retryPolicy(backoffRetry).build();
        curatorClient.start();
    }

    @Override
    public CuratorFramework getObject() throws Exception {
        return curatorClient;
    }

    @Override
    public Class<?> getObjectType() {
        return CuratorFramework.class;
    }

    @Autowired
    public void setZookeeperConfig(ZookeeperConfig zookeeperConfig) {
        this.zookeeperConfig = zookeeperConfig;
    }
}
```

5. 开启三种监听器

```java
@Component
public class ZkListener {
    @Resource
    private CuratorFramework curatorClient;

    @PostConstruct
    public void init(){
        String path = "/zktest";

        //当前节点
        CuratorCache curatorCache = CuratorCache.builder(curatorClient, path).build();
        CuratorCacheListener listener = CuratorCacheListener
                .builder()
                .forNodeCache(new NodeCacheListener(){
                    @Override
                    public void nodeChanged() throws Exception{
                        Optional<ChildData> childData = curatorCache.get(path);
                        if(childData.isPresent()) {
                            String data = new String(childData.get().getData());
                            System.out.println("--------【NodeCacheListener--------】---------nodePath:" + path + " data:" + data);
                        }
                    }
                })
                .build();
        curatorCache.listenable().addListener(listener);

        //监听子节点，不监听当前节点
        CuratorCacheListener pathCacheListener = CuratorCacheListener
                .builder()
                .forPathChildrenCache(path, curatorClient, new PathChildrenCacheListener() {
                    @Override
                    public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
                        String type = event.getType().name();
                        System.out.println("--------【PathChildrenCacheListener】-------- type:" + type);
                        Optional.ofNullable(event.getData()).ifPresent(childata -> {
                            String nodePath = childata.getPath();
                            String data = "";
                            if(Objects.nonNull(childata.getData())) {
                                data = new String(childata.getData());
                            }
                            System.out.println("--------【PathChildrenCacheListener】-------- nodePath:" + nodePath + " data:" + data + " type:" + type);
                        });
                    }
                })
                .build();
        curatorCache.listenable().addListener(pathCacheListener);

        CuratorCacheListener treeListener = CuratorCacheListener
                .builder()
                .forTreeCache(curatorClient, new TreeCacheListener() {
                    @Override
                    public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
                        String type = event.getType().name();
                        System.out.println("--------【TreeCacheListener--------】-------- type:" + type);
                        Optional.ofNullable(event.getData()).ifPresent(childata -> {
                            String nodePath = childata.getPath();
                            String data = "";
                            if(Objects.nonNull(childata.getData())) {
                               data = new String(childata.getData());
                            }
                            System.out.println("--------【TreeCacheListener--------】-------- nodePath:" + nodePath + " data:" + data + " type:" + type);
                        });
                    }
                })
                .build();
        curatorCache.listenable().addListener(treeListener);

        curatorCache.start();
    }
}
```

6. 最终测试，关于zk的操作可以使用上面的基本操作类进行测试，或者直接通过zkCli连接后进行操作

```text
Connected to the target VM, address: '127.0.0.1:49879', transport: 'socket'

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.4.RELEASE)

2021-04-02 15:53:14.983  INFO 25462 --- [           main] com.learning.zookeeper.ZkApplication     : Starting ZkApplication on fiboMBP004 with PID 25462 (/Users/chenzy/files/gitFile/springboot-forall/boot-for-zookeeper/target/classes started by chenzy in /Users/chenzy/files/gitFile/springboot-forall)
2021-04-02 15:53:14.987  INFO 25462 --- [           main] com.learning.zookeeper.ZkApplication     : No active profile set, falling back to default profiles: default
2021-04-02 15:53:15.833  INFO 25462 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 2182 (http)
2021-04-02 15:53:15.841  INFO 25462 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-04-02 15:53:15.841  INFO 25462 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.38]
2021-04-02 15:53:15.905  INFO 25462 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-04-02 15:53:15.906  INFO 25462 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 872 ms
2021-04-02 15:53:15.954  WARN 25462 --- [           main] o.a.c.retry.ExponentialBackoffRetry      : maxRetries too large (1000). Pinning to 29
2021-04-02 15:53:16.009  INFO 25462 --- [           main] o.a.c.f.imps.CuratorFrameworkImpl        : Starting
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:zookeeper.version=3.6.0--b4c89dc7f6083829e18fae6e446907ae0b1f22d7, built on 02/25/2020 14:38 GMT
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:host.name=192.168.7.106
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:java.version=1.8.0_181
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:java.vendor=Oracle Corporation
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:java.home=/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:java.class.path=/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/lib/tools.jar:/Users/chenzy/files/gitFile/springboot-forall/boot-for-zookeeper/target/classes:/Users/chenzy/.m2/repository/org/springframework/boot/spring-boot-starter/2.3.4.RELEASE/spring-boot-starter-2.3.4.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/boot/spring-boot/2.3.4.RELEASE/spring-boot-2.3.4.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/spring-context/5.2.9.RELEASE/spring-context-5.2.9.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/2.3.4.RELEASE/spring-boot-autoconfigure-2.3.4.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/boot/spring-boot-starter-logging/2.3.4.RELEASE/spring-boot-starter-logging-2.3.4.RELEASE.jar:/Users/chenzy/.m2/repository/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar:/Users/chenzy/.m2/repository/ch/qos/logback/logback-core/1.2.3/logback-core-1.2.3.jar:/Users/chenzy/.m2/repository/org/apache/logging/log4j/log4j-to-slf4j/2.13.3/log4j-to-slf4j-2.13.3.jar:/Users/chenzy/.m2/repository/org/apache/logging/log4j/log4j-api/2.13.3/log4j-api-2.13.3.jar:/Users/chenzy/.m2/repository/org/slf4j/jul-to-slf4j/1.7.30/jul-to-slf4j-1.7.30.jar:/Users/chenzy/.m2/repository/jakarta/annotation/jakarta.annotation-api/1.3.5/jakarta.annotation-api-1.3.5.jar:/Users/chenzy/.m2/repository/org/springframework/spring-core/5.2.9.RELEASE/spring-core-5.2.9.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/spring-jcl/5.2.9.RELEASE/spring-jcl-5.2.9.RELEASE.jar:/Users/chenzy/.m2/repository/org/yaml/snakeyaml/1.26/snakeyaml-1.26.jar:/Users/chenzy/.m2/repository/org/springframework/boot/spring-boot-starter-web/2.3.4.RELEASE/spring-boot-starter-web-2.3.4.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/boot/spring-boot-starter-json/2.3.4.RELEASE/spring-boot-starter-json-2.3.4.RELEASE.jar:/Users/chenzy/.m2/repository/com/fasterxml/jackson/core/jackson-databind/2.11.2/jackson-databind-2.11.2.jar:/Users/chenzy/.m2/repository/com/fasterxml/jackson/core/jackson-annotations/2.11.2/jackson-annotations-2.11.2.jar:/Users/chenzy/.m2/repository/com/fasterxml/jackson/core/jackson-core/2.11.2/jackson-core-2.11.2.jar:/Users/chenzy/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jdk8/2.11.2/jackson-datatype-jdk8-2.11.2.jar:/Users/chenzy/.m2/repository/com/fasterxml/jackson/datatype/jackson-datatype-jsr310/2.11.2/jackson-datatype-jsr310-2.11.2.jar:/Users/chenzy/.m2/repository/com/fasterxml/jackson/module/jackson-module-parameter-names/2.11.2/jackson-module-parameter-names-2.11.2.jar:/Users/chenzy/.m2/repository/org/springframework/boot/spring-boot-starter-tomcat/2.3.4.RELEASE/spring-boot-starter-tomcat-2.3.4.RELEASE.jar:/Users/chenzy/.m2/repository/org/apache/tomcat/embed/tomcat-embed-core/9.0.38/tomcat-embed-core-9.0.38.jar:/Users/chenzy/.m2/repository/org/glassfish/jakarta.el/3.0.3/jakarta.el-3.0.3.jar:/Users/chenzy/.m2/repository/org/apache/tomcat/embed/tomcat-embed-websocket/9.0.38/tomcat-embed-websocket-9.0.38.jar:/Users/chenzy/.m2/repository/org/springframework/spring-web/5.2.9.RELEASE/spring-web-5.2.9.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/spring-beans/5.2.9.RELEASE/spring-beans-5.2.9.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/spring-webmvc/5.2.9.RELEASE/spring-webmvc-5.2.9.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/spring-aop/5.2.9.RELEASE/spring-aop-5.2.9.RELEASE.jar:/Users/chenzy/.m2/repository/org/springframework/spring-expression/5.2.9.RELEASE/spring-expression-5.2.9.RELEASE.jar:/Users/chenzy/.m2/repository/org/apache/curator/curator-framework/5.1.0/curator-framework-5.1.0.jar:/Users/chenzy/.m2/repository/org/apache/curator/curator-recipes/5.1.0/curator-recipes-5.1.0.jar:/Users/chenzy/.m2/repository/org/apache/curator/curator-client/5.1.0/curator-client-5.1.0.jar:/Users/chenzy/.m2/repository/org/apache/zookeeper/zookeeper/3.6.0/zookeeper-3.6.0.jar:/Users/chenzy/.m2/repository/commons-lang/commons-lang/2.6/commons-lang-2.6.jar:/Users/chenzy/.m2/repository/org/apache/zookeeper/zookeeper-jute/3.6.0/zookeeper-jute-3.6.0.jar:/Users/chenzy/.m2/repository/org/apache/yetus/audience-annotations/0.5.0/audience-annotations-0.5.0.jar:/Users/chenzy/.m2/repository/io/netty/netty-handler/4.1.52.Final/netty-handler-4.1.52.Final.jar:/Users/chenzy/.m2/repository/io/netty/netty-common/4.1.52.Final/netty-common-4.1.52.Final.jar:/Users/chenzy/.m2/repository/io/netty/netty-resolver/4.1.52.Final/netty-resolver-4.1.52.Final.jar:/Users/chenzy/.m2/repository/io/netty/netty-buffer/4.1.52.Final/netty-buffer-4.1.52.Final.jar:/Users/chenzy/.m2/repository/io/netty/netty-transport/4.1.52.Final/netty-transport-4.1.52.Final.jar:/Users/chenzy/.m2/repository/io/netty/netty-codec/4.1.52.Final/netty-codec-4.1.52.Final.jar:/Users/chenzy/.m2/repository/io/netty/netty-transport-native-epoll/4.1.52.Final/netty-transport-native-epoll-4.1.52.Final.jar:/Users/chenzy/.m2/repository/io/netty/netty-transport-native-unix-common/4.1.52.Final/netty-transport-native-unix-common-4.1.52.Final.jar:/Users/chenzy/.m2/repository/log4j/log4j/1.2.17/log4j-1.2.17.jar:/Users/chenzy/.m2/repository/com/google/guava/guava/27.0.1-jre/guava-27.0.1-jre.jar:/Users/chenzy/.m2/repository/com/google/guava/failureaccess/1.0.1/failureaccess-1.0.1.jar:/Users/chenzy/.m2/repository/com/google/guava/listenablefuture/9999.0-empty-to-avoid-conflict-with-guava/listenablefuture-9999.0-empty-to-avoid-conflict-with-guava.jar:/Users/chenzy/.m2/repository/com/google/code/findbugs/jsr305/3.0.2/jsr305-3.0.2.jar:/Users/chenzy/.m2/repository/org/checkerframework/checker-qual/2.5.2/checker-qual-2.5.2.jar:/Users/chenzy/.m2/repository/com/google/errorprone/error_prone_annotations/2.2.0/error_prone_annotations-2.2.0.jar:/Users/chenzy/.m2/repository/com/google/j2objc/j2objc-annotations/1.1/j2objc-annotations-1.1.jar:/Users/chenzy/.m2/repository/org/codehaus/mojo/animal-sniffer-annotations/1.17/animal-sniffer-annotations-1.17.jar:/Users/chenzy/.m2/repository/org/slf4j/slf4j-api/1.7.30/slf4j-api-1.7.30.jar:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar:/Users/chenzy/Library/Caches/IntelliJIdea2018.3/captureAgent/debugger-agent.jar
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:java.library.path=/Users/chenzy/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:java.io.tmpdir=/var/folders/99/2yt_482s7_qbl4fz3sdrc06s54bv98/T/
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:java.compiler=<NA>
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:os.name=Mac OS X
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:os.arch=x86_64
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:os.version=10.15.2
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:user.name=chenzy
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:user.home=/Users/chenzy
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:user.dir=/Users/chenzy/files/gitFile/springboot-forall
2021-04-02 15:53:16.016  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:os.memory.free=149MB
2021-04-02 15:53:16.017  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:os.memory.max=1820MB
2021-04-02 15:53:16.017  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Client environment:os.memory.total=165MB
2021-04-02 15:53:16.019  INFO 25462 --- [           main] org.apache.zookeeper.ZooKeeper           : Initiating client connection, connectString=localhost:2181 sessionTimeout=60000 watcher=org.apache.curator.ConnectionState@4554de02
2021-04-02 15:53:16.022  INFO 25462 --- [           main] org.apache.zookeeper.common.X509Util     : Setting -D jdk.tls.rejectClientInitiatedRenegotiation=true to disable client-initiated TLS renegotiation
2021-04-02 15:53:16.029  INFO 25462 --- [           main] org.apache.zookeeper.ClientCnxnSocket    : jute.maxbuffer value is 1048575 Bytes
2021-04-02 15:53:16.033  INFO 25462 --- [           main] org.apache.zookeeper.ClientCnxn          : zookeeper.request.timeout value is 0. feature enabled=false
2021-04-02 15:53:16.038  INFO 25462 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Opening socket connection to server localhost/0:0:0:0:0:0:0:1:2181.
2021-04-02 15:53:16.038  INFO 25462 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : SASL config status: Will not attempt to authenticate using SASL (unknown error)
2021-04-02 15:53:16.042  INFO 25462 --- [           main] o.a.c.f.imps.CuratorFrameworkImpl        : Default schema
2021-04-02 15:53:16.060  INFO 25462 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Socket connection established, initiating session, client: /0:0:0:0:0:0:0:1:49886, server: localhost/0:0:0:0:0:0:0:1:2181
2021-04-02 15:53:16.063  INFO 25462 --- [           main] org.apache.curator.utils.Compatibility   : Using org.apache.zookeeper.server.quorum.MultipleAddresses
2021-04-02 15:53:16.070  INFO 25462 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Session establishment complete on server localhost/0:0:0:0:0:0:0:1:2181, session id = 0x100000cada4000b, negotiated timeout = 40000
2021-04-02 15:53:16.077  INFO 25462 --- [ain-EventThread] o.a.c.f.state.ConnectionStateManager     : State change: CONNECTED
2021-04-02 15:53:16.089  INFO 25462 --- [ain-EventThread] o.a.c.framework.imps.EnsembleTracker     : New config event received: {}
2021-04-02 15:53:16.096  INFO 25462 --- [ain-EventThread] o.a.c.framework.imps.EnsembleTracker     : New config event received: {}
--------【TreeCacheListener--------】-------- type:NODE_ADDED
--------【TreeCacheListener--------】-------- nodePath:/zktest data:data2 type:NODE_ADDED
--------【NodeCacheListener--------】---------nodePath:/zktest data:data2
--------【TreeCacheListener--------】-------- type:NODE_ADDED
--------【TreeCacheListener--------】-------- nodePath:/zktest/innerpath2 data:data2 type:NODE_ADDED
--------【NodeCacheListener--------】---------nodePath:/zktest data:data2
--------【PathChildrenCacheListener】-------- type:CHILD_ADDED
--------【PathChildrenCacheListener】-------- nodePath:/zktest/innerpath2 data:data2 type:CHILD_ADDED
--------【TreeCacheListener--------】-------- type:NODE_ADDED
--------【TreeCacheListener--------】-------- nodePath:/zktest/innerpath3 data:data3 type:NODE_ADDED
--------【NodeCacheListener--------】---------nodePath:/zktest data:data2
--------【PathChildrenCacheListener】-------- type:CHILD_ADDED
--------【PathChildrenCacheListener】-------- nodePath:/zktest/innerpath3 data:data3 type:CHILD_ADDED
--------【TreeCacheListener--------】-------- type:NODE_ADDED
--------【TreeCacheListener--------】-------- nodePath:/zktest/innerpath data:data3 type:NODE_ADDED
--------【NodeCacheListener--------】---------nodePath:/zktest data:data2
--------【PathChildrenCacheListener】-------- type:CHILD_ADDED
--------【PathChildrenCacheListener】-------- nodePath:/zktest/innerpath data:data3 type:CHILD_ADDED
--------【TreeCacheListener--------】-------- type:NODE_ADDED
--------【TreeCacheListener--------】-------- nodePath:/zktest/innerpath4 data: type:NODE_ADDED
--------【NodeCacheListener--------】---------nodePath:/zktest data:data2
--------【PathChildrenCacheListener】-------- type:CHILD_ADDED
--------【PathChildrenCacheListener】-------- nodePath:/zktest/innerpath4 data: type:CHILD_ADDED
--------【TreeCacheListener--------】-------- type:INITIALIZED
--------【PathChildrenCacheListener】-------- type:INITIALIZED
2021-04-02 15:53:16.199  INFO 25462 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-04-02 15:53:16.336  INFO 25462 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 2182 (http) with context path ''
2021-04-02 15:53:16.347  INFO 25462 --- [           main] com.learning.zookeeper.ZkApplication     : Started ZkApplication in 1.661 seconds (JVM running for 2.235)
Disconnected from the target VM, address: '127.0.0.1:49879', transport: 'socket'
2021-04-02 15:53:20.610  INFO 25462 --- [extShutdownHook] o.s.s.concurrent.ThreadPoolTaskExecutor  : Shutting down ExecutorService 'applicationTaskExecutor'
```

### 四、其他功能

#### 1. 分布式锁

具体分布式锁的使用场景多样，可自行适配

```java
public class CuratorLockTest {
    private static final String ZK_ADDRESS = "127.0.0.1:2181";
    private static final String ZK_PATH = "/zktest1";

    public static void main(String[] args){
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                ZK_ADDRESS,
                new RetryNTimes(10, 5000)
        );
        client.start();

        Thread thread1 = new Thread(() -> {
            accquireLock(client);
        });

        Thread thread2 = new Thread(() -> {
            accquireLock(client);
        });
        thread1.start();
        thread2.start();
    }

    private static void accquireLock(CuratorFramework client){
        InterProcessMutex lock = new InterProcessMutex(client, ZK_PATH);
        try{
            //等待60s,请求锁
           if(lock.acquire(60, TimeUnit.SECONDS)){
               System.out.println(Thread.currentThread().getName() + " get & hold the lock");
               //占用锁5s
               Thread.sleep(5000);
               System.out.println(Thread.currentThread().getName() + " release the lock");
           }
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            try{
                lock.release();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

测试如下：

```text
16:43:07.621 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got ping response for session id: 0x100000cada40011 after 3ms.
16:43:07.909 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification session id: 0x100000cada40011
16:43:07.910 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeDeleted path:/zktest1/_c_ce3a9167-6e7a-45f4-8b14-36f415775da7-lock-0000000039 for session id 0x100000cada40011
16:43:07.918 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40011, packet:: clientPath:null serverPath:null finished:false header:: 9,12  replyHeader:: 9,158,0  request:: '/zktest1,F  response:: v{'_c_ffcaa88f-2741-4b0c-a85a-f694deb37a22-lock-0000000041,'_c_a99c6f23-5fa0-4cd3-a5d8-fa74ff20f4c7-lock-0000000042,'_c_c57c9182-137b-48ec-a2bf-b9a213066082-lock-0000000040,'_c_add0c35f-918d-49d2-8dae-a09a33c34988-lock-0000000043,'_c_c2baf925-0d5c-474b-b6c1-e293bb364c82-lock-0000000044},s{70,70,1617352787210,1617352787210,0,85,0,0,13,5,158} 
Thread-2 get & hold the lock
Thread-2 release the lock
16:43:12.930 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification session id: 0x100000cada40011
16:43:12.930 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeDeleted path:/zktest1/_c_c57c9182-137b-48ec-a2bf-b9a213066082-lock-0000000040 for session id 0x100000cada40011
16:43:12.931 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40011, packet:: clientPath:null serverPath:null finished:false header:: 10,2  replyHeader:: 10,159,0  request:: '/zktest1/_c_c57c9182-137b-48ec-a2bf-b9a213066082-lock-0000000040,-1  response:: null
16:43:12.934 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40011, packet:: clientPath:null serverPath:null finished:false header:: 11,12  replyHeader:: 11,159,0  request:: '/zktest1,F  response:: v{'_c_ffcaa88f-2741-4b0c-a85a-f694deb37a22-lock-0000000041,'_c_a99c6f23-5fa0-4cd3-a5d8-fa74ff20f4c7-lock-0000000042,'_c_add0c35f-918d-49d2-8dae-a09a33c34988-lock-0000000043,'_c_c2baf925-0d5c-474b-b6c1-e293bb364c82-lock-0000000044},s{70,70,1617352787210,1617352787210,0,86,0,0,13,4,159} 
Thread-1 get & hold the lock
Thread-1 release the lock
16:43:17.947 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40011, packet:: clientPath:null serverPath:null finished:false header:: 12,2  replyHeader:: 12,160,0  request:: '/zktest1/_c_ffcaa88f-2741-4b0c-a85a-f694deb37a22-lock-0000000041,-1  response:: null

```

#### 2.Leader节点选举

用于多个节点为集群情况，某master节点失去心跳后，自动选举master节点


```java
public class CuratorLeaderTest {
    private static final String ZK_ADDRESS = "127.0.0.1:2181";
    private static final String ZK_PATH = "/zktest1";

    public static void main(String[] args) throws InterruptedException {


        LeaderSelectorListener selectorListener = new LeaderSelectorListener() {

            @Override
            public void stateChanged(CuratorFramework curatorFramework, ConnectionState connectionState) {

            }

            @Override
            public void takeLeadership(CuratorFramework client) throws Exception {
                System.out.println(Thread.currentThread().getName() + " take leadership");
                //保留5s
                Thread.sleep(5000);
                System.out.println(Thread.currentThread().getName() + " give up leadership");
            }
        };

        Thread thread1 = new Thread(() -> {
            registerListener(selectorListener);
        });

        Thread thread2 = new Thread(() -> {
            registerListener(selectorListener);
        });

        Thread thread3 = new Thread(() -> {
            registerListener(selectorListener);
        });

        thread1.start();
        thread2.start();
        thread3.start();

        Thread.sleep(5*60*1000);
    }

    private static void registerListener(LeaderSelectorListener listener) {
        //创建连接
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                ZK_ADDRESS,
                new RetryNTimes(10, 5000)
        );
        client.start();

        //确保path存在
        try{
            client.create().creatingParentContainersIfNeeded().forPath(ZK_PATH);
        } catch (Exception e) {
            e.printStackTrace();
        }

        //进行选举
        LeaderSelector leaderSelector = new LeaderSelector(client, ZK_PATH, listener);
        leaderSelector.autoRequeue();
        leaderSelector.start();
    }

```

测试如下：

```text
16:39:47.243 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000e, packet:: clientPath:null serverPath:null finished:false header:: 4,15  replyHeader:: 4,73,0  request:: '/zktest1/_c_419bd043-637c-475b-b649-96c3663a5d9b-lock-,#3139322e3136382e372e313036,v{s{31,s{'world,'anyone}}},3  response:: '/zktest1/_c_419bd043-637c-475b-b649-96c3663a5d9b-lock-0000000000,s{73,73,1617352787238,1617352787238,0,0,0,72057648490741774,13,0,73} 
16:39:47.244 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000f, packet:: clientPath:null serverPath:null finished:false header:: 4,15  replyHeader:: 4,74,0  request:: '/zktest1/_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-,#3139322e3136382e372e313036,v{s{31,s{'world,'anyone}}},3  response:: '/zktest1/_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001,s{74,74,1617352787240,1617352787240,0,0,0,72057648490741775,13,0,74} 
16:39:47.246 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40010, packet:: clientPath:null serverPath:null finished:false header:: 4,15  replyHeader:: 4,75,0  request:: '/zktest1/_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-,#3139322e3136382e372e313036,v{s{31,s{'world,'anyone}}},3  response:: '/zktest1/_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002,s{75,75,1617352787240,1617352787240,0,0,0,72057648490741776,13,0,75} 
16:39:47.250 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40010, packet:: clientPath:null serverPath:null finished:false header:: 5,12  replyHeader:: 5,75,0  request:: '/zktest1,F  response:: v{'_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001,'_c_419bd043-637c-475b-b649-96c3663a5d9b-lock-0000000000,'_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002},s{70,70,1617352787210,1617352787210,0,3,0,0,13,3,75} 
16:39:47.250 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000e, packet:: clientPath:null serverPath:null finished:false header:: 5,12  replyHeader:: 5,75,0  request:: '/zktest1,F  response:: v{'_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001,'_c_419bd043-637c-475b-b649-96c3663a5d9b-lock-0000000000,'_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002},s{70,70,1617352787210,1617352787210,0,3,0,0,13,3,75} 
16:39:47.251 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000f, packet:: clientPath:null serverPath:null finished:false header:: 5,12  replyHeader:: 5,75,0  request:: '/zktest1,F  response:: v{'_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001,'_c_419bd043-637c-475b-b649-96c3663a5d9b-lock-0000000000,'_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002},s{70,70,1617352787210,1617352787210,0,3,0,0,13,3,75} 
Curator-LeaderSelector-1 take leadership
16:39:47.259 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40010, packet:: clientPath:null serverPath:null finished:false header:: 6,4  replyHeader:: 6,75,0  request:: '/zktest1/_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001,T  response:: #3139322e3136382e372e313036,s{74,74,1617352787240,1617352787240,0,0,0,72057648490741775,13,0,74} 
16:39:47.259 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000f, packet:: clientPath:null serverPath:null finished:false header:: 6,4  replyHeader:: 6,75,0  request:: '/zktest1/_c_419bd043-637c-475b-b649-96c3663a5d9b-lock-0000000000,T  response:: #3139322e3136382e372e313036,s{73,73,1617352787238,1617352787238,0,0,0,72057648490741774,13,0,73} 
Curator-LeaderSelector-1 give up leadership
16:39:52.269 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification session id: 0x100000cada4000f
16:39:52.270 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeDeleted path:/zktest1/_c_419bd043-637c-475b-b649-96c3663a5d9b-lock-0000000000 for session id 0x100000cada4000f
16:39:52.270 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000e, packet:: clientPath:null serverPath:null finished:false header:: 6,2  replyHeader:: 6,76,0  request:: '/zktest1/_c_419bd043-637c-475b-b649-96c3663a5d9b-lock-0000000000,-1  response:: null
16:39:52.275 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000e, packet:: clientPath:null serverPath:null finished:false header:: 7,15  replyHeader:: 7,77,0  request:: '/zktest1/_c_e861209a-1520-4ac6-a0c5-5c9fd46f790d-lock-,#3139322e3136382e372e313036,v{s{31,s{'world,'anyone}}},3  response:: '/zktest1/_c_e861209a-1520-4ac6-a0c5-5c9fd46f790d-lock-0000000003,s{77,77,1617352792271,1617352792271,0,0,0,72057648490741774,13,0,77} 
16:39:52.277 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000e, packet:: clientPath:null serverPath:null finished:false header:: 8,12  replyHeader:: 8,77,0  request:: '/zktest1,F  response:: v{'_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001,'_c_e861209a-1520-4ac6-a0c5-5c9fd46f790d-lock-0000000003,'_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002},s{70,70,1617352787210,1617352787210,0,5,0,0,13,3,77} 
16:39:52.282 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000f, packet:: clientPath:null serverPath:null finished:false header:: 7,12  replyHeader:: 7,77,0  request:: '/zktest1,F  response:: v{'_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001,'_c_e861209a-1520-4ac6-a0c5-5c9fd46f790d-lock-0000000003,'_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002},s{70,70,1617352787210,1617352787210,0,5,0,0,13,3,77} 
Curator-LeaderSelector-2 take leadership
16:39:52.282 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000e, packet:: clientPath:null serverPath:null finished:false header:: 9,4  replyHeader:: 9,77,0  request:: '/zktest1/_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002,T  response:: #3139322e3136382e372e313036,s{75,75,1617352787240,1617352787240,0,0,0,72057648490741776,13,0,75} 
Curator-LeaderSelector-2 give up leadership
16:39:57.290 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification session id: 0x100000cada40010
16:39:57.290 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeDeleted path:/zktest1/_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001 for session id 0x100000cada40010
16:39:57.290 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000f, packet:: clientPath:null serverPath:null finished:false header:: 8,2  replyHeader:: 8,78,0  request:: '/zktest1/_c_0fb081c7-77d7-45c1-88a9-dd7c7719348c-lock-0000000001,-1  response:: null
16:39:57.292 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got ping response for session id: 0x100000cada40010 after 2ms.
16:39:57.295 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000f, packet:: clientPath:null serverPath:null finished:false header:: 9,15  replyHeader:: 9,79,0  request:: '/zktest1/_c_2b6e01ea-26dc-42d8-9643-c9f23be88431-lock-,#3139322e3136382e372e313036,v{s{31,s{'world,'anyone}}},3  response:: '/zktest1/_c_2b6e01ea-26dc-42d8-9643-c9f23be88431-lock-0000000004,s{79,79,1617352797291,1617352797291,0,0,0,72057648490741775,13,0,79} 
16:39:57.295 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40010, packet:: clientPath:null serverPath:null finished:false header:: 7,12  replyHeader:: 7,79,0  request:: '/zktest1,F  response:: v{'_c_e861209a-1520-4ac6-a0c5-5c9fd46f790d-lock-0000000003,'_c_2b6e01ea-26dc-42d8-9643-c9f23be88431-lock-0000000004,'_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002},s{70,70,1617352787210,1617352787210,0,7,0,0,13,3,79} 
Curator-LeaderSelector-0 take leadership
16:39:57.297 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000f, packet:: clientPath:null serverPath:null finished:false header:: 10,12  replyHeader:: 10,79,0  request:: '/zktest1,F  response:: v{'_c_e861209a-1520-4ac6-a0c5-5c9fd46f790d-lock-0000000003,'_c_2b6e01ea-26dc-42d8-9643-c9f23be88431-lock-0000000004,'_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002},s{70,70,1617352787210,1617352787210,0,7,0,0,13,3,79} 
16:39:57.299 [Thread-2-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000f, packet:: clientPath:null serverPath:null finished:false header:: 11,4  replyHeader:: 11,79,0  request:: '/zktest1/_c_e861209a-1520-4ac6-a0c5-5c9fd46f790d-lock-0000000003,T  response:: #3139322e3136382e372e313036,s{77,77,1617352792271,1617352792271,0,0,0,72057648490741774,13,0,77} 
Curator-LeaderSelector-0 give up leadership
16:40:02.302 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification session id: 0x100000cada4000e
16:40:02.303 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeDeleted path:/zktest1/_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002 for session id 0x100000cada4000e
16:40:02.303 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40010, packet:: clientPath:null serverPath:null finished:false header:: 8,2  replyHeader:: 8,80,0  request:: '/zktest1/_c_bcaaf80d-1450-419a-b8c5-e2ad72a09481-lock-0000000002,-1  response:: null
16:40:02.305 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Got ping response for session id: 0x100000cada4000e after 2ms.
16:40:02.306 [Thread-0-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada40010, packet:: clientPath:null serverPath:null finished:false header:: 9,15  replyHeader:: 9,81,0  request:: '/zktest1/_c_48b0f6f3-54f5-4565-8148-63a338ae1329-lock-,#3139322e3136382e372e313036,v{s{31,s{'world,'anyone}}},3  response:: '/zktest1/_c_48b0f6f3-54f5-4565-8148-63a338ae1329-lock-0000000005,s{81,81,1617352802304,1617352802304,0,0,0,72057648490741776,13,0,81} 
16:40:02.307 [Thread-1-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x100000cada4000e, packet:: clientPath:null serverPath:null finished:false header:: 10,12  replyHeader:: 10,81,0  request:: '/zktest1,F  response:: v{'_c_48b0f6f3-54f5-4565-8148-63a338ae1329-lock-0000000005,'_c_e861209a-1520-4ac6-a0c5-5c9fd46f790d-lock-0000000003,'_c_2b6e01ea-26dc-42d8-9643-c9f23be88431-lock-0000000004},s{70,70,1617352787210,1617352787210,0,9,0,0,13,3,81} 
Curator-LeaderSelector-1 take leadership
```

#### 3. 集群连接形式

- 就是针对以上connectString:localhost:2181,修改为：localhost:2181,localhost:2182,localhost:2183 即可,能够和单节点一样进行操作

```text
17:16:15.145 [main] INFO org.apache.zookeeper.ZooKeeper - Initiating client connection, connectString=127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183 sessionTimeout=60000 watcher=org.apache.curator.ConnectionState@754ba872
17:16:15.150 [main] INFO org.apache.zookeeper.common.X509Util - Setting -D jdk.tls.rejectClientInitiatedRenegotiation=true to disable client-initiated TLS renegotiation
17:16:15.160 [main] INFO org.apache.zookeeper.ClientCnxnSocket - jute.maxbuffer value is 1048575 Bytes
17:16:15.167 [main] INFO org.apache.zookeeper.ClientCnxn - zookeeper.request.timeout value is 0. feature enabled=false
17:16:15.172 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.SaslServerPrincipal - Canonicalized address to localhost
17:16:15.174 [main-SendThread(127.0.0.1:2181)] INFO org.apache.zookeeper.ClientCnxn - Opening socket connection to server localhost/127.0.0.1:2181.
17:16:15.174 [main-SendThread(127.0.0.1:2181)] INFO org.apache.zookeeper.ClientCnxn - SASL config status: Will not attempt to authenticate using SASL (unknown error)
17:16:15.178 [main] INFO org.apache.curator.framework.imps.CuratorFrameworkImpl - Default schema
zk client start successfully!
$ create /zktest1 hello 
17:16:15.195 [main-SendThread(127.0.0.1:2181)] INFO org.apache.zookeeper.ClientCnxn - Socket connection established, initiating session, client: /127.0.0.1:56136, server: localhost/127.0.0.1:2181
17:16:15.198 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Session establishment request sent on localhost/127.0.0.1:2181
17:16:15.235 [main-SendThread(127.0.0.1:2181)] INFO org.apache.zookeeper.ClientCnxn - Session establishment complete on server localhost/127.0.0.1:2181, session id = 0x10000a2880f0000, negotiated timeout = 40000
17:16:15.238 [main-EventThread] DEBUG org.apache.curator.ConnectionState - Negotiated session timeout: 40000
17:16:15.245 [main-EventThread] INFO org.apache.curator.framework.state.ConnectionStateManager - State change: CONNECTED
17:16:15.246 [main-EventThread] DEBUG org.apache.curator.framework.imps.CuratorFrameworkImpl - Clearing sleep for 1 operations
17:16:15.268 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x10000a2880f0000, packet:: clientPath:/zookeeper/config serverPath:/zookeeper/config finished:false header:: 1,4  replyHeader:: 1,4294967297,0  request:: '/zookeeper/config,T  response:: #7365727665722e313d302e302e302e303a323838383a333838383a7061727469636970616e743b302e302e302e303a32313831a7365727665722e323d7a6b323a323838383a333838383a7061727469636970616e743b302e302e302e303a32313831a7365727665722e333d7a6b333a323838383a333838383a7061727469636970616e743b302e302e302e303a32313831a76657273696f6e3d30,s{0,0,0,1617354951026,-1,0,-1,0,157,0,0} 
17:16:15.272 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x10000a2880f0000, packet:: clientPath:/zookeeper/config serverPath:/zookeeper/config finished:false header:: 2,4  replyHeader:: 2,4294967297,0  request:: '/zookeeper/config,T  response:: #7365727665722e313d302e302e302e303a323838383a333838383a7061727469636970616e743b302e302e302e303a32313831a7365727665722e323d7a6b323a323838383a333838383a7061727469636970616e743b302e302e302e303a32313831a7365727665722e333d7a6b333a323838383a333838383a7061727469636970616e743b302e302e302e303a32313831a76657273696f6e3d30,s{0,0,0,1617354951026,-1,0,-1,0,157,0,0} 
17:16:15.274 [main-EventThread] INFO org.apache.curator.framework.imps.EnsembleTracker - New config event received: {server.1=0.0.0.0:2888:3888:participant;0.0.0.0:2181, version=0, server.3=zk3:2888:3888:participant;0.0.0.0:2181, server.2=zk2:2888:3888:participant;0.0.0.0:2181}
17:16:15.280 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x10000a2880f0000, packet:: clientPath:null serverPath:null finished:false header:: 3,15  replyHeader:: 3,4294967298,0  request:: '/zktest1,#68656c6c6f,v{s{31,s{'world,'anyone}}},0  response:: '/zktest1,s{4294967298,4294967298,1617354975260,1617354975260,0,0,0,0,5,0,4294967298} 
$ ls / 
17:16:15.290 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x10000a2880f0000, packet:: clientPath:null serverPath:null finished:false header:: 4,12  replyHeader:: 4,4294967298,0  request:: '/,F  response:: v{'zookeeper,'zktest1},s{0,0,0,0,0,0,0,0,0,2,4294967298} 
[zookeeper, zktest1]
$ get /zktest1 
17:16:15.295 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x10000a2880f0000, packet:: clientPath:null serverPath:null finished:false header:: 5,4  replyHeader:: 5,4294967298,0  request:: '/zktest1,F  response:: #68656c6c6f,s{4294967298,4294967298,1617354975260,1617354975260,0,0,0,0,5,0,4294967298} 
hello
$ set /zktest1 world 
17:16:15.310 [main-SendThread(127.0.0.1:2181)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply session id: 0x10000a2880f0000, packet:: clientPath:null serverPath:null finished:false header:: 6,5  replyHeader:: 6,4294967299,0  request:: '/zktest1,#776f726c64,-1  response:: s{4294967298,4294967299,1617354975260,1617354975300,1,0,0,0,5,0,4294967298} 
$ get /zktest1 
```