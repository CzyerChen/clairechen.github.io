---
layout:     post
title:      OpenSIPS 邂逅 Kafka：构建高效 VoIP 消息处理架构

subtitle:   OpenSIPS
date:       2025-06-10
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - OpenSIPS
    - Kafka
    - VOIP
---

- [使用场景](#使用场景)
- [使用步骤](#使用步骤)
  - [引入模块](#引入模块)
  - [组装\&发送数据](#组装发送数据)
  - [消费数据](#消费数据)
  - [故障转移](#故障转移)

## 使用场景

1. 异步日志处理：将 OpenSIPS 中的 SIP 信令日志、通话记录（CDR）等数据发送到 Kafka 队列中。
   
2. 事件通知与监控：利用 OpenSIPS 的 event_interface 模块将 SIP 事件（如呼叫建立、断开、注册等）推送到 Kafka

OpenSIPS中事件接口有以下类型：

- EVENT_DATAGRAM - Publish JSON-RPC notifications using UDP, stable
- EVENT_FLATSTORE - Text/File backend for events, stable
- EVENT_KAFKA - Publish JSON-RPC notifications/generic messages to Apache Kafka , stable
- EVENT_STREAM - Publish JSON-RPC notifications using TCP, stable
- EVENT_ROUTE - Route triggering based on events, stable
- EVENT_ROUTING - Event-based routing, stable
- EVENT_RABBITMQ - Publish JSON-RPC notifications using AMQP over TCP , stable
- EVENT_VIRTUAL - Aggregator of event backends (failover & balancing), stable
- EVENT_XMLRPC - Event XMLRPC client module , stable

3. 分布式消息队列集成：在复杂的 VoIP 架构中，OpenSIPS 可以通过 Kafka 与其他服务（如计费系统、CRM 系统）解耦
   
4. 计费与数据分析：OpenSIPS 生成的 CDR（Call Detail Records）可以通过 Kafka 推送至后端计费系统
   
5. 故障隔离与重试机制：在 OpenSIPS 调用外部服务时（如鉴权、计费接口），如果目标服务不可用，可以将请求暂存到 Kafka
   
6. 微服务架构下的通信桥梁：在基于微服务的 VoIP 架构中，OpenSIPS 作为 SIP 边界网关，可通过 Kafka 与其他微服务（如认证服务、媒体控制服务）进行异步通信
   
7. 消息广播与事件驱动架构：OpenSIPS 可将特定的 SIP 事件广播到 Kafka 的多个主题，供不同的下游服务消费
   
8. 性能优化与流量削峰：在高并发场景下，Kafka 可以作为缓冲层，缓解 OpenSIPS 与后端系统之间的流量压力
   
9.  自定义业务逻辑扩展：通过 Kafka 与外部业务逻辑模块解耦，可以在不影响 OpenSIPS 核心逻辑的前提下，灵活扩展新的业务功能

## 使用步骤

### 引入模块

```text
loadmodule "event_kafka.so"
modparam("event_kafka", "broker_id", "[k1]127.0.0.1:9092/opensips?g.linger.ms=100&t.acks=all")
```

链接语法：'kafka:' brokers '/' topic ['?' properties]
properties语法：'g.'|'t.' property '=' value ['&' 'g.'|'t.' property '=' value] ...
可以设置的proroperty[参考官方说明](https://github.com/confluentinc/librdkafka/blob/master/CONFIGURATION.md)

### 组装&发送数据

```text
        $json(sql_obj) := "{}";
        $json(sql_obj/table) = 'acc';
        $json(sql_obj/method) = $param(method);
        $json(sql_obj/fromTag) = $param(from_tag);
        $json(sql_obj/toTag) = $param(to_tag);
        $json(sql_obj/callid) = $param(callid);
        $json(sql_obj/sipCode) = $param(sip_code);
        $json(sql_obj/sipReason) = $param(sip_reason);
        $json(sql_obj/time) = $param(time);
        $json(sql_obj/duration) = $param(duration);
        $json(sql_obj/msDuration) = $param(ms_duration);
        $json(sql_obj/setuptime) = $param(setuptime);
        $json(sql_obj/created) = $param(created);

        kafka_publish("k1", $json(sql_obj), $ci, "kafka_report");
```

可异步监听回执

```text
route[kafka_report] {
    xlog("[$avp(kafka_id)] status=$avp(kafka_status) key=$avp(kafka_key) msg=$avp(kafka_msg)\n");
    ...
}
```

### 消费数据

```bash
$ bin/kafka-console-consumer.sh   --bootstrap-server 127.0.0.1:9092   --topic opensips --from-beginning


{ "status": 1, "dlg_id": "17091151056627", "callid": "1-14809@127.0.0.1", "from_uri": "sip:20250610@127.0.0.1:5060", "from_tag": "1", "to_uri": "sip:tt061013776167200@127.0.0.1:5900", "caller_sock": "127.0.0.1:5900", "caller_contact": "sip:20250610@127.0.0.1:5060", "start_time": 0, "timeout": 0, "caller_in": "20250610", "callee_in": "tt061013776167200", "caller_gateway": "46", "callee_gateway": "9", "src_ip": "127.0.0.1", "dst_ip": "127.0.0.1" }
{ "status": 5, "dlg_id": "17091151056627", "to_uri": "sip:tt061013776167200@127.0.0.1:5900", "to_tag": "1", "callee_contact": "sip:127.0.0.1:5080;transport=UDP", "start_time": "1749459233", "timeout": "1749462833", "caller_in": "20250610", "callee_in": "tt061013776167200", "caller_gateway": "46", "callee_gateway": "9", "src_ip": "127.0.0.1", "dst_ip": "127.0.0.1" }
{ "table": "acc", "method": "INVITE", "fromTag": "1", "toTag": "1", "callid": "1-14809@127.0.0.1", "sipCode": "200", "sipReason": "OK", "time": 1749459233, "duration": 4, "msDuration": 3411, "setuptime": 8, "created": 1749459225, "srcIp": "127.0.0.1", "dstIp": "127.0.0.1", "caller": "20250610", "callee": "331213776167200", "callStartTime": "1749459233.433263", "callEndTime": "1749459236.844957", "callerIn": "20250610", "calleeIn": "tt061013776167200", "callerOut": "20241213ob16701", "calleeOut": "calleeout330613776167200", "callergateway": "46", "calleegateway": "9", "calllevel": "0", "routinglevel": "0", "calleraccount": "1", "calleeaccount": "2", "callerCallid": "1-14809@127.0.0.1", "calleeCallid": "", "area": "", "endSide": "1", "endCode": "9201", "endReason": "caller hang up", "realDuration": "3411", "through_jt": "1", "callerproductid": "", "calleeproductid": "", "routing_path": "9-200-0;", "node_addr": "127.0.0.1:5900", "multi_gw": "", "s_timeout": "", "event_time": "1749459236.847516" }
{ "status": 6, "dlg_id": "17091151056627", "callid": "1-14809@127.0.0.1" }
```

### 故障转移

引入event_virtual/event_flatstore，将事件消息通过队列传递，并且支持故障转移

`异常信息 ->> EVENT ->> KAFKA(故障链路 ->> EVENT_VIRTUAL ->> EVENT_FLATSTORE)`

```text
loadmodule "event_flatstore.so"
loadmodule "event_kafka.so"
loadmodule "event_virtual.so"

startup_route {
	subscribe_event("E_MY_EVENT", "virtual:FAILOVER kafka:127.0.0.1:9092/opensipsfailover flatstore:/var/log/myevents");
}
```

