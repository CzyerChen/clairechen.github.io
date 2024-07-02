---
layout:     post
title:      Skywalking之日志分析语言LAL的配置与解析

subtitle:   Skywalking
date:       2024-04-30
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Skywalking
    - LAL
---

提到Skywalking相比并不陌生，或多或少地听到过这个名词，如果你是JAVA开发者，那么可能就更为了解。作为国内甚至国际上热度比较高、社区比较活跃的APM(Application Performance Monitoring System)系统，它拥有众多的使用者，从传统Java服务，到云原生架构下的服务，甚至框架。

人们对于日志价值的挖掘也是源源不断，从打日志、存日志、同步日志到解析日志，更期望从规范的日志中提取有效的消息，用于检测或告警，那么今天要介绍的Skywalking中的LAL(Log Analysis Language)模块，就是协助用户从有效的日志数据中，结合自定义的pattern，抽离出有效的字段，用于应用的监控或告警。

全文概览
- [LAL 的背景知识](#lal-的背景知识)
- [SkyWalking LAL 的基础语法](#skywalking-lal-的基础语法)
  - [Layer](#layer)
  - [Filter](#filter)
    - [全局操作符 - abort](#全局操作符---abort)
    - [全局操作符 - tag](#全局操作符---tag)
    - [Filter:Parser模块](#filterparser模块)
    - [Filter:Extractor模块](#filterextractor模块)
    - [Filter:Sink模块](#filtersink模块)

## LAL 的背景知识

SkyWalking中的LAL的语法就能够实现，通过正则等形式，结合groovy语法，匹配出指定字段进行提取，实现从文本中摘出所需要的字段进行聚合统计。

处理的这个数据来源主要就是各类日志，比如Nginx日志、数据库日志、应用日志，由于应用日志是人为写入的，那么对于这个监控就有无限可能。

其实这个功能有很多类似的业内实现，比如说：

Filebeat：轻量级日志文件 shipper，用于转发和集中日志数据到 Elasticsearch 或 Logstash。 同样支持json和正则的形式，分为input/processors/output

```js
filebeat.inputs:
- type: stdin
  processors:
    - dissect:
        tokenizer: "%{logDate} %{logTime}  %{logLevel} %{pid} --- [%{thread}] %{logger} : %{message}"
        field: "message"
        target_prefix: "dissect"
output.console:
  enabled: true
  pretty: true

filebeat.inputs:
  - type: log
    paths:
      - "/data/logs/*/error.log"
    document_type: json
    json.message_key: log
    json.keys.under_root: true
    json.overwrite_keys: true
    fields:
      logType: errJson
    fields_under_root: true
```

Fluentd / Fluent Bit：高可用、高性能的日志采集器，支持多种输入输出插件，适用于多种数据源和目标。区分为input/parser/filter/output

```js
<parse>
  @type regexp
  expression /^\[(?<logtime>[^\]]*)\] (?<name>[^ ]*) (?<title>[^ ]*) (?<id>\d*)$/
  time_key logtime
  time_format %Y-%m-%d %H:%M:%S %z
  types id:integer
</parse>
```

Logstash：Elastic Stack 的一部分，强大的数据处理管道，能够从多个来源收集数据，转换并输出到像 Elasticsearch 这样的存储系统。区分为 input/filter/output

```js
input { stdin { } }

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}

output {
  elasticsearch { hosts => ["localhost:9200"] }
  stdout { codec => rubydebug }
}
```


如果对这些类似的产品有过使用经验的，那你肯定能非常快速的理解和上手LAL，主要包含2块概念：Layer, Filter

## SkyWalking LAL 的基础语法

对于语法官网也有给到[专门的说明](https://skywalking.apache.org/docs/main/next/en/concepts-and-designs/lal/#log-analysis-language)

基本框架：

```js
filter { // filter A: this is for persistence
    // ... parser

    sink {
        sampler {
            // .. sampler configs
        }
    }
}
filter { // filter B:
    // ... extractors to generate many metrics
    extractors {
        metrics {
            // ... metrics
        }
    }
    sink {
        dropper {} // drop all logs because they have been saved in "filter A" above.
    }
}

```

### Layer

对于你这块功能一个命名，需要是唯一的具有指向性的名称

你如果在lal的目录下，新增了一个日志解析方式，需要生效就需要在 `application.yml` 文件中，在 `log-analyzer:default:lalFiles` 的配置下，启用你自定义的layer，或者通过 `SW_LOG_LAL_FILES` 的环境变量来激活你的配置

### Filter

Filter模块是一组 parser, extractor 和 sink。

Filter允许做一些条件判断，其中可以结合log对象的参数值
log对象值有：

```java
 // [Optional] The timestamp of the log, in millisecond.
    // If not set, OAP server would use the received timestamp as log's timestamp, or relies on the OAP server analyzer.
    int64 timestamp = 1;
    // [Required] **Service**. Represents a set/group of workloads which provide the same behaviours for incoming requests.
    //
    // The logic name represents the service. This would show as a separate node in the topology.
    // The metrics analyzed from the spans, would be aggregated for this entity as the service level.
    //
    // If this is not the first element of the streaming, use the previous not-null name as the service name.
    string service = 2;
    // [Optional] **Service Instance**. Each individual workload in the Service group is known as an instance. Like `pods` in Kubernetes, it
    // doesn't need to be a single OS process, however, if you are using instrument agents, an instance is actually a real OS process.
    //
    // The logic name represents the service instance. This would show as a separate node in the instance relationship.
    // The metrics analyzed from the spans, would be aggregated for this entity as the service instance level.
    string serviceInstance = 3;
    // [Optional] **Endpoint**. A path in a service for incoming requests, such as an HTTP URI path or a gRPC service class + method signature.
    //
    // The logic name represents the endpoint, which logs belong.
    string endpoint = 4;
    // [Required] The content of the log.
    LogDataBody body = 5;
    // [Optional] Logs with trace context
    TraceContext traceContext = 6;
    // [Optional] The available tags. OAP server could provide search/analysis capabilities based on these.
    LogTags tags = 7;
    // [Optional] Since 9.0.0
    // The layer of the service and servce instance. If absent, the OAP would set `layer`=`ID: 2, NAME: general`
    string layer = 8;
```

#### 全局操作符 - abort

类似于for循环中的 `continue`， 已经达到满足的条件，提前跳出当前流程，后续步骤都不会被执行

```js
filter {
    if (log.service == "TestingService") { // Don't waste resources on TestingServices
        abort {} // all remaining components won't be executed at all
    }
    // ... parsers, extractors, sinks
}
```

#### 全局操作符 - tag

给数据打上标签，丰富最终的数据内容，对于后续步骤中也可以以此作为判断，比如给不同条件的数据做归类，后续进行不同处理等

```js
[
   {
      "tags":{
         "data":[
            {
               "key":"TEST_KEY",
               "value":"TEST_VALUE"
            }
         ]
      },
      "body":{
         ...
      }
      ...
   }
]
```

对增加的标签进行判断

```js
filter {
    if (tag("TEST_KEY") == "TEST_VALUE") {
         ...   
    }
}
```

#### Filter:Parser模块

这个解析器模块就和前面描述的组件的解析部分一样，就是什么样的数据结构需要做什么样的解析处理，包括有json类型数据、yaml类型数据和text文本类型数据

json和yaml类型的数据很好理解，都是有相对固定的格式要求，text文本类型就需要借助一些正则来实现匹配

给出规则进行匹配，匹配成功后就能够获取到对应解析出来的对象，那么这些数据如何获取呢？

这边会有一个 `parsed` 对象来承载解析成功的字段，可以通过 `parsed.xxx` 来读取到对应的值，是一个典型的map。 json和yaml由于都是固定的键值类型的数据，因此解析后 `parsed` 对象就会包含其所有值。对于text的文本类型，由于需要类似正则的方式进行匹配的，那么匹配成功就能够获取对应的上的字段值。

此处有一个 `abortOnFailure` 的参数，表示日志到解析这一步如果就失败了那后续匹配链上的步骤还要不要继续，默认就是遇到失败就跳过不继续往下走了。

- JSON格式

```js
filter {
    json {
        abortOnFailure true // this is optional because it's default behaviour
    }
}
```

- YAML格式

```js
filter {
    yaml {
        abortOnFailure true // this is optional because it's default behaviour
    }
}
```

- TEXT格式

纯粹的文本格式可以通过强大的正则表达式去适配所有的情况
其中regexp的参数必须使用前后括号包围： regexp ("xxxxxx")

```js
filter {
    text {
        abortOnFailure true // this is optional because it's default behaviour
        // this is just a demo pattern
        regexp "(?<timestamp>\\d{8}) (?<thread>\\w+) (?<level>\\w+) (?<traceId>\\w+) (?<msg>.+)"
    }
    extractor {
        tag level: parsed.level
        // we add a tag called `level` and its value is parsed.level, captured from the regexp above
        traceId parsed.traceId
        // we also extract the trace id from the parsed result, which will be used to associate the log with the trace
    }
    // ...
}
```

#### Filter:Extractor模块

提取器模块，就是将上一个解析步骤产生的字段罗列罗列，筛选出需要的字段，进行后续的聚合或存储，数据可以取自`log对象` 和 `parsed对象`

`parsed对象` 除了上面解析出来的一些字段，还有一些默认系统加入的字段，都是系统内链路上共有的值，为了日志能够更好地与服务、实例、链路关联起来

- service：parsed.service, 服务的名称，用于和链路和指标绑定
- instance：parsed.instance, 服务实例的名称，用于和链路和指标绑定
- endpoint：parsed.endpoint, 服务实例的名称，用于和链路和指标绑定
- traceId：parsed.traceId，用于和链路和指标绑定
- segmentId：parsed.segmentId，用于和链路和指标绑定
- spanId：parsed.spanId，用于和链路和指标绑定
- timestamp：提取后可以指定类型和format, 到毫秒级别

```js
filter {
    // ... parser

    extractor {
        timestamp parsed.time as String
    }
}

filter {
    // ... parser

    extractor {
        timestamp parsed.time as String, "yyyy-MM-dd HH:mm:ss"
    }
}
```

- layer：parsed.layer, 用于与服务绑定
- tag：手动打标签

```js
filter {
    // ... parser

    extractor {
        tag level: parsed.level, (parsed.statusCode): parsed.statusMsg
        tag anotherKey: "anotherConstantValue"
        layer 'GENERAL'
    }
}
```

- metrics：前面步骤我们通过parse和extractor前面的步骤，实现了从文本中解析出键值对，并根据条件打上了额外的标签，最终如何来使用呢？--> 使用MAL语言，将值做成指标

首先，在 `lal` 脚本中，组装好 metrics 的定义

```js
filter {
    // ...
    extractor {
        service parsed.serviceName
        metrics {
            name "log_count" //指标名称
            timestamp parsed.timestamp //增加一个时间戳
            labels level: parsed.level, service: parsed.service, instance: parsed.instance //抽取出status_code、service、instance 三个标签，可用于前端匹配与展示
            value 1 //作为一个值，出现1次计数1次
        }
        metrics {
            name "http_response_time" //指标名称
            timestamp parsed.timestamp //增加一个时间戳
            labels status_code: parsed.statusCode, service: parsed.service, instance: parsed.instance //抽取出status_code、service、instance 三个标签，可用于前端匹配与展示
            value parsed.duration //作为一个值，对于时间后续可以累加
        }
    }
    // ...
}
```

随后，在 `log-mal-rules` 目录中增加指标 MAL 的声明，此外还需要在 `application.yaml` 中声明启用这个配置文件：`log-analyzer:default:malFiles`

```yaml
metrics:
  - name: log_count_debug
    exp: log_count.tagEqual('level', 'DEBUG').sum(['service', 'instance']).increase('PT1M')
  - name: log_count_error
    exp: log_count.tagEqual('level', 'ERROR').sum(['service', 'instance']).increase('PT1M')
  - name: response_time_percentile
    exp: http_response_time.sum(['le', 'service', 'instance']).increase('PT5M').histogram().histogram_percentile([50,70,90,99])
    
```

以上就是MAL中的语法，实现过滤、预聚合，给到最终进行存储
通过上述的MAL的定义，在页面看板中，就可以在service面板中，获取到日志文件中按照时间维护DEBUG/ERROR类型的日志有多少条

- slowSql：通过"LOG_KIND" = "SLOW_SQL"的标记，可以让OAP服务识别普通日志和慢查询日志
  
OAP服务也会对于每个服务，默认每10分钟存储TOP50的SQL，topNReportPeriod: ${SW_CORE_TOPN_REPORT_PERIOD:10})

- statement

除了展示具体的慢SQL日志,还可以提取SQL statement

- latency

提取数据库慢SQL的延时信息

- id

id字段指示数据库慢SQL

- sampledTrace

对大量的细节数据进行采样，降低下游存储等压力

#### Filter:Sink模块

接收器持久化模块，通过上述的读取、解析、打标处理，最终日志指标数据需要存储起来

- Sampler：针对数据进行采样，减少真实数据量，如果命中多个取样器，最后一个取样器决定取样结果
  - rateLimit: 1分钟采样多少条日志. rateLimit("SamplerID") 需要一个取样ID，声明相同的ID可以使用相同的取样策略
  - possibility: 通过Java随机数产生一个可能性，与百分比参数进行比对

```js
filter {
    // ... parser

    sink {
        sampler {
            if (parsed.service == "ImportantApp") {
                rateLimit("ImportantAppSampler") {
                    rpm 1800  // samples 1800 pieces of logs every minute for service "ImportantApp"
                }
            } else {
                rateLimit("OtherSampler") {
                    rpm 180   // samples 180 pieces of logs every minute for other services than "ImportantApp"
                }
            }
        }
    }
}

filter {
    // ... parser

    sink {
        sampler {
            if (parsed.service == "ImportantApp") {
                possibility(80) { // samples 80% of the logs for service "ImportantApp"
                }
            } else {
                possibility(30) { // samples 30% of the logs for other services than "ImportantApp"
                }
            }
        }
    }
}

```

- Dropper: 是一个特殊的接收器，根据某些标签把一类日志全部丢弃，比如DEBUG日志

```js
filter {
    // ... parser

    sink {
        if (parsed.level == "DEBUG") {
            dropper {}
        } else {
            sampler {
                // ... configs
            }
        }
    }
}
```

- Enforcer：是一种特殊的接收器，强制采样日志，主要场景：对于已经配置采样器，但是对于一些重要的ERROR日志希望百分之百展现的，可以使用这个

```js
filter {
    // ... parser

    sink {
        sampler {
            // ... sampler configs
        }
        if (parsed.level == "ERROR" || parsed.userId == "TestingUserId") { // sample error logs or testing users' logs (userId == "TestingUserId") even if the sampling strategy is configured
            enforcer {
            }
        }
    }
}
```

----
如果喜欢我的文章的话，可以去[GitHub上给一个免费的关注](https://github.com/CzyerChen/)吗？
