---
layout:     post
title:      Nacos|SpringBoot项目示例
subtitle:   Nacos
date:       2023-08-17
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringBoot
    - Nacos
    - Nacos-Spring
---

## 1.maven pom依赖

```xml
     <dependency>
            <groupId>com.alibaba.nacos</groupId>
            <artifactId>nacos-spring-context</artifactId>
            <version>1.1.1</version>
        </dependency>
```

## 2.配置nacos初始化配置

```java
@Configuration
@EnableNacosConfig(globalProperties = @NacosProperties(serverAddr = "http://127.0.0.1:8848", username="nacos",password = "nacos"))
@NacosPropertySources({
        @NacosPropertySource(dataId = "example", autoRefreshed = true,properties =@NacosProperties(namespace = "") ),
        @NacosPropertySource(dataId = "arthas",groupId = "sms-admin",autoRefreshed = true,properties =@NacosProperties(namespace = "015ae736-f567-4633-b6ce-62d2adf0970a") ),
        @NacosPropertySource(dataId = "arthas",groupId = "sms-channel",autoRefreshed = true,properties =@NacosProperties(namespace = "f114336d-d549-44bc-bd57-4f41ba5a8347") )
})
public class NacosConfig {
}
```

## 3.@NacosValue读取

```java
    @NacosValue(value = "${useLocalCache:false}", autoRefreshed = true)
    private boolean useLocalCache;

    @NacosValue(value = "${testnumber:-1}", autoRefreshed = true)
    private Integer testNumber;
```

## 4.@NacosConfigurationProperties

```java
@Component
@NacosConfigurationProperties(prefix = "arthas",dataId = "arthas",groupId = "sms-admin",autoRefreshed = true,properties =@NacosProperties(namespace = "015ae736-f567-4633-b6ce-62d2adf0970a"),type = ConfigType.PROPERTIES)
public class DemoProperties {
    private String home;
    private String agentid;
    private String configMap;
}
```

## 5.参数发布，动态刷新

```java
    @NacosInjected
    private ConfigService configService;

    @PostMapping                        
    @ResponseBody
    public ResponseEntity<String> publish(@RequestParam String dataId,
                                          @RequestParam(defaultValue = "DEFAULT_GROUP") String group,
                                          @RequestParam String content) throws NacosException {
        boolean result = configService.publishConfig(dataId, group, content);
        if (result) {
            return new ResponseEntity<String>("Success", HttpStatus.OK);
        }
        return new ResponseEntity<String>("Fail", HttpStatus.INTERNAL_SERVER_ERROR);
    }

```

## 6.本地冷启动

1)在本地缓存目录，手动根据namespace group dateid 的要求，写入指定数据，在程序中能够读取

2)用于一些特殊网络限制冷启动的情况

3)默认{home}/nacos/config/fixed-{host}_{port}-{namespace}_nacos/snapshot-tenant/{namespace}/{group}

 ```text
/Users/xxx/nacos/config/fixed-127.0.0.1_8848-015ae736-f567-4633-b6ce-62d2adf0970a_nacos/snapshot-tenant/015ae736-f567-4633-b6ce-62d2adf0970a/sms-channel
```
