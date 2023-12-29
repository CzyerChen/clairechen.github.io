---
layout:     post
title:      Skywalking 之本地Plugin-test测试
subtitle:   Skywalking Plugin-test 
date:       2023-12-20
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - 云原生
    - Skywalking
---

- [1、关于 Plugin-test 官方文档解读](#1关于-plugin-test-官方文档解读)
- [2、基础镜像依赖](#2基础镜像依赖)
- [3、测试用例项目介绍](#3测试用例项目介绍)
  - [startup.sh](#startupsh)
  - [expectedData.yaml](#expecteddatayaml)
  - [configuration.yml](#configurationyml)
  - [support-version.list](#support-versionlist)
- [4、实际测试](#4实际测试)

### 1、关于 Plugin-test 官方文档解读

[官方文档Plugin-test说明链接](https://github.com/CzyerChen/skywalking-java/blob/main/docs/en/setup/service-agent/java-agent/Plugin-test.md)

环境需求：

1. MacOS/Linux
2. Jdk8 及以上
3. docker环境

### 2、基础镜像依赖

主要涵盖JDK和Container两块:

- 仅JVM环境，无需tomcat的情况：面向内嵌的Web应用服务器，比如SpringBoot项目，默认`eclipse-temurin:8-jdk`，如果JDK调整可以自行去搜索其他版本，比如`eclipse-temurin:17-jdk`
- 需要额外Tomcat环境：面向无Web应用服务器的项目，比如Spring项目，默认`tomcat:8.5-jdk8-openjdk`，如果JDK调整可以自行去搜索其他版本，比如`tomcat:10.1-jdk17-temurin`

### 3、测试用例项目介绍

- 测试用例项目什么时候写？
  有新增插件类型，现有的测试用例通过增加版本已经不满足流程测试需求，此时需要增加额外的一个测试用例项目，便于他们快速熟悉插件的使用逻辑
- 测试用例项目怎么写？
  可以参照下面的流程来组织项目
- 测试项目提交前需要满足什么？
  按照下面的流程完成本地的自测，跑通后可以提交
- 如果项目跑起来测试能够正常，测试工具无法跑通，怎么办？
  可能是测试工具中的一次不适配，可以自己研究测试工具的源码

项目主体架构

**JVM类型基础项目结构**

```xml
[plugin-scenario]
    |- [bin]
        |- startup.sh
    |- [config]
        |- expectedData.yaml
    |- [src]
        |- [main]
            |- ...
        |- [resource]
            |- log4j2.xml
    |- pom.xml
    |- configuration.yml
    |- support-version.list

[] = directory
```

**Tomcat类型基础项目结构**

```xml
[plugin-scenario]
    |- [config]
        |- expectedData.yaml
    |- [src]
        |- [main]
            |- ...
        |- [resource]
            |- log4j2.xml
        |- [webapp]
            |- [WEB-INF]
                |- web.xml
    |- pom.xml
    |- configuration.yml
    |- support-version.list

[] = directory
```

#### startup.sh

仅 JVM 类型有这个脚本，比如：

```bash
home="$(cd "$(dirname $0)"; pwd)"
java -jar ${agent_opts} -Dskywalking.plugin.feign.collect_request_body=true \
-Dskywalking.plugin.feign.supported_content_types_prefix='application/json,text/' ${home}/../libs/feign-scenario.jar &
```

#### expectedData.yaml

预期数据的情况，做验证

运算符：

- nq: 不等于
- eq: 等于
- ge: 大于等于
- gt: 大于

默认字符串：

- not null: 不为空
- not blank: 不为空，不为空字符串
- null: null或者空字符串
- eq: 等于
- start with: 测试字符串以什么开头
- end with: 测试字符串以什么结尾

综合样例：

```yaml
segmentItems:
- serviceName: gateway-projectB-scenario
  segmentSize: nq 0
  segments:
  - segmentId: not null
    spans:
    - operationName: /provider/timeout/error
      parentSpanId: 0
      spanId: 1
      isError: true
      spanType: Exit
      tags:
      - { key: url, value: not null }
      - { key: http.method, value: GET }
      - { key: http.status_code, value: '500' }
      logs:
      - logEvent:
        - { key: event, value: error }
        - { key: error.kind, value: not null }
        - { key: message, value: not null }
        - {key: stack, value: not null}
      - logEvent:
        - { key: event, value: error }
        - { key: error.kind, value: not null }
        - { key: message, value: not null }
        - { key: stack, value: not null }
    - operationName: GET:/provider/b/testcase
      parentSpanId: -1
      spanId: 0
      spanLayer: Http
      startTime: nq 0
      endTime: nq 0
      componentId: 14
      isError: false
      spanType: Entry
      peer: ''
      tags:
      - {key: url, value: not null}
      - {key: http.method, value: GET}
      - {key: http.status_code, value: '200'}
      refs:
      - {parentEndpoint: SpringCloudGateway/RoutingFilter, networkAddress: 'localhost:18070',
        refType: CrossProcess, parentSpanId: 1, parentTraceSegmentId: not null, parentServiceInstance: not
          null, parentService: not null, traceId: not null}
      skipAnalysis: 'false'
- serviceName: gateway-projectA-scenario
  segmentSize: nq 0
  segments:
  - segmentId: not null
    spans:
    - operationName: /provider/timeout/error
      parentSpanId: -1
      spanId: 0
      spanLayer: Http
      isError: true
      spanType: Entry
      tags:
      - {key: url, value: 'http://localhost:8080/provider/timeout/error' }
      - {key: http.method, value: GET}
      - {key: http.status_code, value: '500'}
  - segmentId: not null
    spans:
    - operationName: SpringCloudGateway/sendRequest
      parentSpanId: 0
      spanId: 1
      spanLayer: Http
      startTime: nq 0
      endTime: nq 0
      componentId: 61
      isError: true
      spanType: Exit
      peer: 1.2.3.4:18070
      skipAnalysis: false
      tags:
      - { key: url, value: not null }
      logs:
      - logEvent:
        - { key: event, value: error }
        - { key: error.kind, value: not null }
        - { key: message, value: not null }
        - { key: stack, value: not null}
    - operationName: SpringCloudGateway/RoutingFilter
      parentSpanId: -1
      spanId: 0
      startTime: nq 0
      endTime: nq 0
      componentId: 61
      spanType: Local
      refs:
      - { parentEndpoint: '/provider/timeout/error', networkAddress: '', refType: CrossThread,
        parentSpanId: 0, parentTraceSegmentId: not null, parentServiceInstance: not
          null, parentService: not null, traceId: not null }
      skipAnalysis: 'false'
  - segmentId: not null
    spans:
    - operationName: SpringCloudGateway/sendRequest
      parentSpanId: 0
      spanId: 1
      spanLayer: Http
      startTime: nq 0
      endTime: nq 0
      componentId: 61
      isError: false
      spanType: Exit
      peer: 'localhost:18070'
      tags:
      - {key: url, value: not null}
      - {key: http.status_code, value: '200'}
      skipAnalysis: 'false'
    - operationName: SpringCloudGateway/RoutingFilter
      parentSpanId: -1
      spanId: 0
      startTime: nq 0
      endTime: nq 0
      componentId: 61
      isError: false
      spanType: Local
      peer: ''
      refs:
      - {parentEndpoint: '/provider/b/testcase', networkAddress: '', refType: CrossThread,
        parentSpanId: 0, parentTraceSegmentId: not null, parentServiceInstance: not
          null, parentService: not null, traceId: not null}
      skipAnalysis: 'false'
  - segmentId: not null
    spans:
    - operationName: /provider/b/testcase
      parentSpanId: -1
      spanId: 0
      spanLayer: Http
      startTime: nq 0
      endTime: nq 0
      componentId: 67
      isError: false
      spanType: Entry
      peer: not null
      tags:
      - { key: url, value: 'http://localhost:8080/provider/b/testcase' }
      - { key: http.method, value: GET }
      - { key: http.status_code, value: '200' }
      skipAnalysis: 'false'

```

#### configuration.yml

```yml
type: jvm
entryService: http://localhost:8080/feign-scenario/case/feign-scenario
healthCheck: http://localhost:8080/feign-scenario/case/healthCheck
startScript: ./bin/startup.sh
environment:
dependencies:
```

- type: 描述当前服务的类型: jvm 或 tomcat
- entryService: 验证的接口
- healthCheck: 服务正常启动的验证接口
- startScript: 服务正常启动后，需要执行的脚本
- runningMode：default(default), with_optional, or with_bootstrap，三种形式，默认是default
- withPlugins: 在 with_optional或with_bootstrap生效的情况下存在，引入额外的依赖jar，比如apm-spring-annotation-plugin-*.jar
- environment：同docker-compose#environment
- depends_on：同docker-compose#depends_on
- dependencies：同docker-compose#services, image, links, hostname, command, environment

#### support-version.list

描述当前测试覆盖的版本情况，通常挑选代表性变更的版本即可，保证插件的功能都支持正常

### 4、实际测试

```bash
#进入项目根目录
>> cd ~/skywalking

#执行脚本 --debug可以输出真实执行过程中的数据
>> bash ./test/plugin/run.sh  -f spring-scenario

#脚本中会根据JDK版本进行基础的编译打包
>> mvn -q --batch-mode -f ~/skywalking-java/test/plugin/pom.xml -DskipTests -Dbase_image_java=eclipse-temurin:17-jdk-ubi9-minimal -Dcontainer_image_version=1.0.0 clean package
模板：
${mvnw} -q --batch-mode \
  -f ${home}/pom.xml \
  -DskipTests \
  -Dbase_image_java=${base_image_java} \
  -Dbase_image_tomcat=${base_image_tomcat} \
  -Dcontainer_image_version=${container_image_version} \
  clean package

#将镜像打包到本地仓库
docker run -d \
        --memory=1024m \
        --name spring-scenario-4.0.0-1.0.0 \
        --env SCENARIO_NAME=spring-scenario \
        --env SCENARIO_VERSION=4.0.0  \
        --env SCENARIO_SUPPORT_FRAMEWORK=gateway-4.x-scenario \
        --env SCENARIO_START_SCRIPT=./bin/startup.sh \
        --env SCENARIO_ENTRY_SERVICE=http://localhost:8080/provider/b/testcase \
        --env SCENARIO_HEALTH_CHECK_URL=http://localhost:8080/provider/b/healthCheck \
        --env SCENARIO_EXTEND_ENTRY_HEADER= \
        --env CATALINA_OPTS= \
        --env DEBUG_MODE= \
        -v ~/skywalking-java/test/plugin/workspace/spring-scenario/agent_with_optional:/usr/local/skywalking/scenario/agent \
        -v ~/skywalking-java/test/plugin/workspace/spring-scenario/4.0.0:/usr/local/skywalking/scenario \
        -v ~/skywalking-java/test/plugin/../jacoco:/jacoco \
        skywalking/agent-test-jvm:1.0.0

#脚本中会获取容器id，请求healthCheck接口、entryService接口，满足healthCheck条件后，请求service接口，与expectData来核对，完成后清除容器，完成验证
```

真实案例 scenario.sh：

```bash
#!/usr/bin/env bash

[[ -n $1 ]] && set -ex

PRG="$0"
PRGDIR=`dirname "$PRG"`
[ -z "$SCENARIO_HOME" ] && SCENARIO_HOME=`cd "$PRGDIR" >/dev/null; pwd`

testcase_name=gateway-3.x-scenario-3.0.0

status=1


docker run -d \
        --memory=1024m \
        --name gateway-3.x-scenario-3.0.0-1.0.0 \
        --env SCENARIO_NAME=gateway-3.x-scenario \
        --env SCENARIO_VERSION=3.0.0  \
        --env SCENARIO_SUPPORT_FRAMEWORK=gateway-3.x-scenario \
        --env SCENARIO_START_SCRIPT=./bin/startup.sh \
        --env SCENARIO_ENTRY_SERVICE=http://localhost:8080/provider/b/testcase \
        --env SCENARIO_HEALTH_CHECK_URL=http://localhost:8080/provider/b/healthCheck \
        --env SCENARIO_EXTEND_ENTRY_HEADER= \
        --env CATALINA_OPTS= \
        --env DEBUG_MODE=on \
        -v /Users/chenzy/files/gitFile/skywalking-own5/skywalking-java/test/plugin/workspace/gateway-3.x-scenario/agent_with_optional:/usr/local/skywalking/scenario/agent \
        -v /Users/chenzy/files/gitFile/skywalking-own5/skywalking-java/test/plugin/workspace/gateway-3.x-scenario/3.0.0:/usr/local/skywalking/scenario \
        -v /Users/chenzy/files/gitFile/skywalking-own5/skywalking-java/test/plugin/../jacoco:/jacoco \
        skywalking/agent-test-jvm:1.0.0

sleep 3

container_name=`docker ps -aqf "name=gateway-3.x-scenario-3.0.0-1.0.0"`
status=$(docker wait ${container_name})

if [[ -z ${container_name} ]]; then
    echo "docker startup failure!" >&2
    status=1
else
    [[ $status -ne 0 ]] && docker logs ${container_name} >&2
    docker logs ${container_name} >${SCENARIO_HOME}/logs/container.log
    docker container rm -f $container_name
fi


exit $status
```

实际打包好执行的目录可以参考：

```bash
gateway-3.x-scenario
│   ├── 3.0.0
│   │   ├── agent
│   │   ├── data
│   │   │   ├── actualData.yaml
│   │   │   └── expectedData.yaml
│   │   ├── gateway-3.x-scenario.zip
│   │   ├── logs
│   │   │   ├── collector.out
│   │   │   ├── container.log
│   │   │   ├── gateway-3.x-scenario-3.0.0.log
│   │   │   ├── helper.log
│   │   │   ├── scenario.out
│   │   │   ├── skywalking-api.log
│   │   │   └── validator.out
│   │   └── scenario.sh
│   └── agent_with_optional
│       ├── LICENSE
│       ├── NOTICE
│       ├── activations
│       │   └── ...
│       ├── bootstrap-plugins
│       │   └── ...
│       ├── config
│       │   └── agent.config
│       ├── licenses
│       │   └── LICENSE-asm.txt
│       ├── logs
│       │   └── skywalking-api.log
│       ├── optional-plugins
│       │   └── ...
│       ├── optional-reporter-plugins
│       │   └──...
│       ├── plugins
│       │   └── ...
```