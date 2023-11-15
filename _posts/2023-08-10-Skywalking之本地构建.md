---
layout:     post
title:      Skywalking|本地源码构建
subtitle:   Skywalking
date:       2023-08-10
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Skywalking
    - 源码构建
---

Skywalking相信有很多人使用过，通过容器或者下载安装包进行安装的，今天从源代码角度，拉取、构建、启动。
官方文档步骤简洁明了，我这边会结合自己遇到的一些问题做出总结。

当前构建资源版本：

- MAC 10.15.7
- IDEA 2021.2
- node v16.14.0 (据代码要求安装了一个新版本)
- npm 8.3.1
- Skywalking 9.6.0-SNAPSHOT
- JDK 11 (最新的代码要看最新的要求，注意不能使用JDK8，用错的话构建时也会有提示)
- Maven 3.6.0

**官方指南(必看)**：https://github.com/apache/skywalking/blob/master/docs/en/guides/How-to-build.md

快速阅览：

- [流程1：源码获取](#流程1源码获取)
- [流程2：submodule模块初始化](#流程2submodule模块初始化)
- [或者流程1\&2：二合一](#或者流程12二合一)
- [流程3：构建项目](#流程3构建项目)
- [流程4：构建产生的源代码数据加入项目索引](#流程4构建产生的源代码数据加入项目索引)
- [流程5：启动前后端代码](#流程5启动前后端代码)

## 流程1：源码获取

方法一：fork代码后，在自己的仓库下 git clone源代码
方法二：下载源码包后解压

【下载失败，网络问题，只能自行解决】

## 流程2：submodule模块初始化

.gitsubmodules文件

```text
[submodule "apm-protocol/apm-network/src/main/proto"]
	path = apm-protocol/apm-network/src/main/proto
	url = https://github.com/apache/skywalking-data-collect-protocol.git
[submodule "oap-server/server-query-plugin/query-graphql-plugin/src/main/resources/query-protocol"]
	path = oap-server/server-query-plugin/query-graphql-plugin/src/main/resources/query-protocol
	url = https://github.com/apache/skywalking-query-protocol.git
[submodule "test/e2e-v2/java-test-service/e2e-protocol/src/main/proto"]
	path = test/e2e-v2/java-test-service/e2e-protocol/src/main/proto
	url = https://github.com/apache/skywalking-data-collect-protocol.git
[submodule "skywalking-ui"]
	path = skywalking-ui
	url = https://github.com/apache/skywalking-booster-ui.git
```

以上可以看出关联的仓库数据主要有：

-  `https://github.com/apache/skywalking-data-collect-protocol.git`
-  `https://github.com/apache/skywalking-query-protocol.git`
-  `https://github.com/apache/skywalking-data-collect-protocol.git`
-  `https://github.com/apache/skywalking-booster-ui.git`

执行以下命令，来获取这些代码数据：

```bash
git submodule init //初始化仓库目录位置
git submodule update //**这步骤至关重要**
```

这个环节由于涉及四个仓库的代码拉取，此时如果网络状况不佳，很有可能存在【空文件夹】的情况，所以可以这样解决：

**方法一**：保障网络稳定，删除拉取的空文件夹，再重新`git submodule init` AND `git submodule update` ，重复尝试直至拉取成功
关注这些项目文件夹:

- apm-protocol/apm-network/src/main/proto
- oap-server/server-query-plugin/query-graphql-plugin/src/main/resources/query-protocol
- test/e2e-v2/java-test-service/e2e-protocol/src/main/proto
- skywalking-ui

**方法二**： 去除.gitsubmodules文件中的内容，按照需求取拉取对应的代码，参照【gitsubmodules】文件目录要求放到指定的文件夹内

以上，可能存在的问题主要就是网络方面，如果以上项目目录未拉全就去执行下一步构建，会产生很多报错，**这步必须确认好**


## 或者流程1&2：二合一

```bash
git clone --recurse-submodules https://github.com/apache/skywalking.git
```

也就是clone代码，并且把submodules一起拉下来

## 流程3：构建项目

前提确认一下**JDK 11+** & **Maven 3.6+** 的环境，最好提前安装node要求的版本

项目根目录下执行`./mvnw clean package -Dmaven.test.skip`，构建项目较多，可能需要几分钟时间

这个步骤遇到比较多的问题，基本都是卡在**node版本的下载和npm依赖包的拉取**上，最终skywalking-ui的项目始终build不成功

1. node版本的下载，本地有8和18的版本，当前构建需要16的版本，下载网速出了问题

    当前用到的v16.14.0，还可以确认一下node和npm的版本映射关系(`https://nodejs.org/zh-cn/download/releases`)，我当前在npm8.3.1，是可以兼容

2. npm install拉取依赖包也存在网速问题，可以替换镜像源

    skywalking/apm-webapp/pom.xml `<build>`中，找到

    ```xml
        <arguments>install --registry=https://registry.npmjs.org/</arguments>
         <!--修改为淘宝镜像-->
        <arguments>install --registry=https://registry.npm.taobao.org/</arguments>
    ```

3. 【qiang】的问题，以为会变快，其实反作用
4. 整体流程构建如果最终都是卡在skywalking-ui前端项目的构建上，可以单独使用对应的node版本去这个项目下单独`npm install` & `npm run build` 确认下，避免要重头再来，时间比较长

最终：

```bash
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for apm 9.6.0-SNAPSHOT:
[INFO] 
[INFO] apm ................................................ SUCCESS [  5.195 s]
[INFO] oap-protocol ....................................... SUCCESS [  0.335 s]
[INFO] apm-network ........................................ SUCCESS [ 28.284 s]
[INFO] oap-server ......................................... SUCCESS [  0.446 s]
[INFO] server-library ..................................... SUCCESS [  0.280 s]
[INFO] library-util ....................................... SUCCESS [  4.413 s]
[INFO] library-module ..................................... SUCCESS [  1.693 s]
[INFO] server-telemetry ................................... SUCCESS [  0.187 s]
[INFO] telemetry-api ...................................... SUCCESS [  1.580 s]
[INFO] server-testing ..................................... SUCCESS [  1.385 s]
[INFO] server-configuration ............................... SUCCESS [  0.372 s]
[INFO] configuration-api .................................. SUCCESS [  1.714 s]
[INFO] library-elasticsearch-client ....................... SUCCESS [  7.733 s]
[INFO] library-client ..................................... SUCCESS [  2.882 s]
[INFO] library-server ..................................... SUCCESS [  1.853 s]
[INFO] library-datacarrier-queue .......................... SUCCESS [  0.902 s]
[INFO] ai-pipeline ........................................ SUCCESS [  2.719 s]
[INFO] server-core ........................................ SUCCESS [ 27.428 s]
[INFO] library-kubernetes-support ......................... SUCCESS [  1.696 s]
[INFO] analyzer ........................................... SUCCESS [  0.509 s]
[INFO] meter-analyzer ..................................... SUCCESS [  4.938 s]
[INFO] agent-analyzer ..................................... SUCCESS [  4.672 s]
[INFO] log-analyzer ....................................... SUCCESS [  3.668 s]
[INFO] event-analyzer ..................................... SUCCESS [  1.611 s]
[INFO] server-receiver-plugin ............................. SUCCESS [  0.982 s]
[INFO] skywalking-sharing-server-plugin ................... SUCCESS [  1.546 s]
[INFO] skywalking-trace-receiver-plugin ................... SUCCESS [  2.440 s]
[INFO] zipkin-receiver-plugin ............................. SUCCESS [  1.786 s]
[INFO] skywalking-mesh-receiver-plugin .................... SUCCESS [  1.946 s]
[INFO] skywalking-management-receiver-plugin .............. SUCCESS [  1.607 s]
[INFO] skywalking-jvm-receiver-plugin ..................... SUCCESS [  1.963 s]
[INFO] receiver-proto ..................................... SUCCESS [ 17.060 s]
[INFO] envoy-metrics-receiver-plugin ...................... SUCCESS [  3.926 s]
[INFO] skywalking-clr-receiver-plugin ..................... SUCCESS [  1.542 s]
[INFO] skywalking-profile-receiver-plugin ................. SUCCESS [  1.549 s]
[INFO] otel-receiver-plugin ............................... SUCCESS [  2.132 s]
[INFO] skywalking-meter-receiver-plugin ................... SUCCESS [  1.976 s]
[INFO] skywalking-browser-receiver-plugin ................. SUCCESS [  2.540 s]
[INFO] skywalking-log-receiver-plugin ..................... SUCCESS [  2.236 s]
[INFO] configuration-discovery-receiver-plugin ............ SUCCESS [  2.265 s]
[INFO] skywalking-event-receiver-plugin ................... SUCCESS [  2.095 s]
[INFO] skywalking-zabbix-receiver-plugin .................. SUCCESS [  3.999 s]
[INFO] skywalking-ebpf-receiver-plugin .................... SUCCESS [  2.799 s]
[INFO] skywalking-telegraf-receiver-plugin ................ SUCCESS [  2.467 s]
[INFO] aws-firehose-receiver .............................. SUCCESS [  7.095 s]
[INFO] server-cluster-plugin .............................. SUCCESS [  0.148 s]
[INFO] cluster-zookeeper-plugin ........................... SUCCESS [  1.676 s]
[INFO] cluster-standalone-plugin .......................... SUCCESS [  1.344 s]
[INFO] cluster-kubernetes-plugin .......................... SUCCESS [  1.972 s]
[INFO] cluster-consul-plugin .............................. SUCCESS [  1.811 s]
[INFO] cluster-etcd-plugin ................................ SUCCESS [  1.955 s]
[INFO] cluster-nacos-plugin ............................... SUCCESS [  1.982 s]
[INFO] server-storage-plugin .............................. SUCCESS [  0.120 s]
[INFO] storage-jdbc-hikaricp-plugin ....................... SUCCESS [  5.366 s]
[INFO] storage-elasticsearch-plugin ....................... SUCCESS [  4.129 s]
[INFO] storage-banyandb-plugin ............................ SUCCESS [  4.208 s]
[INFO] oal-grammar ........................................ SUCCESS [  1.128 s]
[INFO] oal-rt ............................................. SUCCESS [  2.740 s]
[INFO] server-query-plugin ................................ SUCCESS [  0.134 s]
[INFO] zipkin-query-plugin ................................ SUCCESS [  2.032 s]
[INFO] server-fetcher-plugin .............................. SUCCESS [  0.111 s]
[INFO] kafka-fetcher-plugin ............................... SUCCESS [  3.037 s]
[INFO] server-health-checker .............................. SUCCESS [  1.661 s]
[INFO] mqe-grammar ........................................ SUCCESS [  0.735 s]
[INFO] mqe-rt ............................................. SUCCESS [  2.202 s]
[INFO] query-graphql-plugin ............................... SUCCESS [  3.807 s]
[INFO] promql-plugin ...................................... SUCCESS [  3.402 s]
[INFO] logql-plugin ....................................... SUCCESS [  1.966 s]
[INFO] server-alarm-plugin ................................ SUCCESS [  5.635 s]
[INFO] telemetry-prometheus ............................... SUCCESS [  1.485 s]
[INFO] exporter ........................................... SUCCESS [  3.927 s]
[INFO] grpc-configuration-sync ............................ SUCCESS [  2.922 s]
[INFO] configuration-apollo ............................... SUCCESS [  1.624 s]
[INFO] configuration-zookeeper ............................ SUCCESS [  1.661 s]
[INFO] configuration-etcd ................................. SUCCESS [  1.129 s]
[INFO] configuration-consul ............................... SUCCESS [  1.498 s]
[INFO] configuration-k8s-configmap ........................ SUCCESS [  1.837 s]
[INFO] configuration-nacos ................................ SUCCESS [  1.452 s]
[INFO] server-starter ..................................... SUCCESS [  7.200 s]
[INFO] server-tools ....................................... SUCCESS [  0.096 s]
[INFO] profile-exporter ................................... SUCCESS [  0.091 s]
[INFO] tool-profile-snapshot-server-mock .................. SUCCESS [  1.363 s]
[INFO] tool-profile-snapshot-bootstrap .................... SUCCESS [  2.893 s]
[INFO] tool-profile-snapshot-exporter ..................... SUCCESS [  3.118 s]
[INFO] data-generator ..................................... SUCCESS [  4.334 s]
[INFO] microbench ......................................... SUCCESS [  8.785 s]
[INFO] oap-server-bom ..................................... SUCCESS [  0.059 s]
[INFO] apm-webapp ......................................... SUCCESS [03:32 min]
[INFO] apache-skywalking-apm .............................. SUCCESS [ 37.877 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  08:42 min
[INFO] Finished at: 2023-08-09T18:59:37+08:00
[INFO] ------------------------------------------------------------------------
```

## 流程4：构建产生的源代码数据加入项目索引

IDEA中，可以找到对应的目录，右键 **"Mark Dictory as"** -> **"Generated Sources Root"**，将构建产生的数据加入编辑器的索引中

- apm-protocol/apm-network/target/generated-sources/protobuf
- oap-server/server-core/target/generated-sources/protobuf
- oap-server/server-receiver-plugin/receiver-proto/target/generated-sources/fbs
- oap-server/server-receiver-plugin/receiver-proto/target/generated-sources/protobuf
- oap-server/exporter/target/generated-sources/protobuf
- oap-server/server-configuration/grpc-configuration-sync/target/generated-sources/protobuf
- oap-server/server-alarm-plugin/target/generated-sources/protobuf
- oap-server/oal-grammar/target/generated-sources

## 流程5：启动前后端代码

1.启动OAP核心服务，接收数据上报与数据查询

- /skywalking/oap-server/server-starter/src/main/java/org/apache/skywalking/oap/server/starter/OAPServerStartUp.java 
- 后端项目启动，存储的数据默认是H2，也可以自己根据标准版本部署对应的elastcisearch版本进行配置
- 由于整体是插件化的架构风格，在组件的选择上使用 `selector` 即可快速锁定选择的组件。默认存储是h2，修改为es的只要 `selector: ${SW_STORAGE:elasticsearch}` 并修改下方elasticsearch对应的配置即可
  
```xml
storage:
  selector: ${SW_STORAGE:h2} #--> 修改
  elasticsearch:
    namespace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:localhost:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    connectTimeout: ${SW_STORAGE_ES_CONNECT_TIMEOUT:3000}
    socketTimeout: ${SW_STORAGE_ES_SOCKET_TIMEOUT:30000}
    responseTimeout: ${SW_STORAGE_ES_RESPONSE_TIMEOUT:15000}
    numHttpClientThread: ${SW_STORAGE_ES_NUM_HTTP_CLIENT_THREAD:0}
    user: ${SW_ES_USER:""}
    password: ${SW_ES_PASSWORD:""}
    trustStorePath: ${SW_STORAGE_ES_SSL_JKS_PATH:""}
    trustStorePass: ${SW_STORAGE_ES_SSL_JKS_PASS:""}
    secretsManagementFile: ${SW_ES_SECRETS_MANAGEMENT_FILE:""} # Secrets management file in the properties format includes the username, password, which are managed by 3rd party tool.
    dayStep: ${SW_STORAGE_DAY_STEP:1} # Represent the number of days in the one minute/hour/day index.
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1} # Shard number of new indexes
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:1} # Replicas number of new indexes
    # Specify the settings for each index individually.
    # If configured, this setting has the highest priority and overrides the generic settings.
    specificIndexSettings: ${SW_STORAGE_ES_SPECIFIC_INDEX_SETTINGS:""}
    # Super data set has been defined in the codes, such as trace segments.The following 3 config would be improve es performance when storage super size data in es.
    superDatasetDayStep: ${SW_STORAGE_ES_SUPER_DATASET_DAY_STEP:-1} # Represent the number of days in the super size dataset record index, the default value is the same as dayStep when the value is less than 0
    superDatasetIndexShardsFactor: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_SHARDS_FACTOR:5} #  This factor provides more shards for the super data set, shards number = indexShardsNumber * superDatasetIndexShardsFactor. Also, this factor effects Zipkin traces.
    superDatasetIndexReplicasNumber: ${SW_STORAGE_ES_SUPER_DATASET_INDEX_REPLICAS_NUMBER:0} # Represent the replicas number in the super size dataset record index, the default value is 0.
    indexTemplateOrder: ${SW_STORAGE_ES_INDEX_TEMPLATE_ORDER:0} # the order of index template
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:5000} # Execute the async bulk record data every ${SW_STORAGE_ES_BULK_ACTIONS} requests
    batchOfBytes: ${SW_STORAGE_ES_BATCH_OF_BYTES:10485760} # A threshold to control the max body size of ElasticSearch Bulk flush.
    # flush the bulk every 5 seconds whatever the number of requests
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:5}
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
    resultWindowMaxSize: ${SW_STORAGE_ES_QUERY_MAX_WINDOW_SIZE:10000}
    metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:10000}
    scrollingBatchSize: ${SW_STORAGE_ES_SCROLLING_BATCH_SIZE:5000}
    segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
    profileTaskQueryMaxSize: ${SW_STORAGE_ES_QUERY_PROFILE_TASK_SIZE:200}
    profileDataQueryBatchSize: ${SW_STORAGE_ES_QUERY_PROFILE_DATA_BATCH_SIZE:100}
    oapAnalyzer: ${SW_STORAGE_ES_OAP_ANALYZER:"{\"analyzer\":{\"oap_analyzer\":{\"type\":\"stop\"}}}"} # the oap analyzer.
    oapLogAnalyzer: ${SW_STORAGE_ES_OAP_LOG_ANALYZER:"{\"analyzer\":{\"oap_log_analyzer\":{\"type\":\"standard\"}}}"} # the oap log analyzer. It could be customized by the ES analyzer configuration to support more language log formats, such as Chinese log, Japanese log and etc.
    advanced: ${SW_STORAGE_ES_ADVANCED:""}
    # Enable shard metrics and records indices into multi-physical indices, one index template per metric/meter aggregation function or record.
    logicSharding: ${SW_STORAGE_ES_LOGIC_SHARDING:false}
    # Custom routing can reduce the impact of searches. Instead of having to fan out a search request to all the shards in an index, the request can be sent to just the shard that matches the specific routing value (or values).
    enableCustomRouting: ${SW_STORAGE_ES_ENABLE_CUSTOM_ROUTING:false}
  h2:
    properties:
      jdbcUrl: ${SW_STORAGE_H2_URL:jdbc:h2:mem:skywalking-oap-db;DB_CLOSE_DELAY=-1;DATABASE_TO_UPPER=FALSE}
      dataSource.user: ${SW_STORAGE_H2_USER:sa}
    metadataQueryMaxSize: ${SW_STORAGE_H2_QUERY_MAX_SIZE:5000}
    maxSizeOfBatchSql: ${SW_STORAGE_MAX_SIZE_OF_BATCH_SQL:100}
    asyncBatchPersistentPoolSize: ${SW_STORAGE_ASYNC_BATCH_PERSISTENT_POOL_SIZE:1}
```

- 启动后端服务跑在 127.0.0.1:11800

2.启动web页面项目，供页面操作

- /skywalking/apm-webapp/src/main/java/org/apache/skywalking/oap/server/webapp/ApplicationStartUp.java
- 启动服务跑在 127.0.0.1:8080
- 浏览器中访问，可以展示菜单页

3.选择一个springBoot项目

- 可以选择下载skywalking-agent.jar，或者也走源码构建的流程
- 下载地址：`https://skywalking.apache.org/downloads/#JavaAgent`
- 页面选择Java Agent，可以选择源码包构建，或者打包好的distribution的包
- 这边直接下载打包好的包：`https://www.apache.org/dyn/closer.cgi/skywalking/java-agent/8.16.0/apache-skywalking-java-agent-8.16.0.tgz`
- 在springboot启动参数中添加skywalking追踪：-javaagent:~/Downloads/skywalking-agent/skywalking-agent.jar -Dskywalking.agent.service_name=demo -Dskywalking.collector.backend_service=127.0.0.1:11800
- 应用启动后能够查看到DB\REDIS相关的链接请求，访问后端服务接口，几秒后页面能够刷新报表。关注service,instance,endpoint相关指标

以上源码构建，本地部署Skywalking项目就算告一段落啦
