---
layout:     post
title:      SpringBoot 数据库连接池与 ManticoreSearch 兼容性测试

subtitle:   Manticore Search
date:       2025-06-25
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Manticore Search
    - MySQL
    - Spring Boot
    - 数据库连接池
---

本文将详细介绍 Spring Boot 如何使用不同的数据库连接池与 Manticoresearch 组件进行交互，帮助初学者快速上手。我们会先进行数据库连接池与 Manticore 的兼容性测试，接着分析各个连接池的特点，最后给出选择建议，并附上详细的操作步骤和示例代码。

- [数据库连接池 Manticore兼容性测试](#数据库连接池-manticore兼容性测试)
  - [**1. HikariCP**](#1-hikaricp)
  - [**2. Druid**](#2-druid)
  - [**3. Tomcat JDBC**](#3-tomcat-jdbc)
  - [**4. Apache DBCP2**](#4-apache-dbcp2)
  - [**5. C3P0**](#5-c3p0)
- [各个数据库连接池的特点](#各个数据库连接池的特点)
  - [**1. HikariCP**](#1-hikaricp-1)
  - [**2. Apache DBCP2**](#2-apache-dbcp2)
  - [**3. C3P0**](#3-c3p0)
  - [**4. Druid**](#4-druid)
  - [**5. Tomcat JDBC Pool**](#5-tomcat-jdbc-pool)
  - [**核心区别总结**](#核心区别总结)
  - [**选择建议**](#选择建议)
- [总结](#总结)


## 数据库连接池 Manticore兼容性测试

在开始使用 Spring Boot 与 Manticoresearch 交互之前，我们需要测试不同的数据库连接池与 Manticore 的兼容性。下面是测试结果，这些结果可以帮助我们选择合适的连接池。

先说结论：

| 兼容性 | 连接库 | 实现类 | 特点 |
| --- | --- | --- | --- |
| NO | HikariCP | HikariDataSource | 高性能，Spring Boot 默认 |
| NO | Druid | DruidDataSource | 阿里开源，内置监控/防火墙 |
| YES | Tomcat | JDBC | org.apache.tomcat.jdbc.pool.DataSource |
| YES | DBCP2 | BasicDataSource | Apache 老牌连接池 |
| YES | C3P0 | ComboPooledDataSource | 历史久远，兼容性强 |

### **1. HikariCP**

HikariCP 是 Spring Boot 默认的数据库连接池，以高性能和轻量级著称。但在与 Manticoresearch 交互时，可能会遇到兼容性问题。下面为初学者详细介绍其使用和遇到的问题。

spring-boot-starter-jdbc 3.2.5 默认引入连接池为 HikariCP

在Spring Boot 1.x版本中默认使用tomcat-jdbc连接池，而在2.x版本中则默认使用hikaricp连接池。考虑到用户询问的是3.2.5版本，这属于2.x之后的版本，应该延续使用HikariCP作为默认连接池。

报错如下：

```
   com.zaxxer.hikari.pool.HikariPool        : HikariPool-1 - Exception during pool initialization.

java.sql.SQLException: Could not map transaction isolation '0' to a valid JDBC level.
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:130) ~[mysql-connector-j-8.2.0.jar:8.2.0]
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:98) ~[mysql-connector-j-8.2.0.jar:8.2.0]
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:90) ~[mysql-connector-j-8.2.0.jar:8.2.0]
at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:64) ~[mysql-connector-j-8.2.0.jar:8.2.0]
at com.mysql.cj.jdbc.ConnectionImpl.getTransactionIsolation(ConnectionImpl.java:1183) ~[mysql-connector-j-8.2.0.jar:8.2.0]
at com.zaxxer.hikari.pool.PoolBase.checkDefaultIsolation(PoolBase.java:480) ~[HikariCP-5.0.1.jar:na]
at com.zaxxer.hikari.pool.PoolBase.checkDriverSupport(PoolBase.java:443) ~[HikariCP-5.0.1.jar:na]
at com.zaxxer.hikari.pool.PoolBase.setupConnection(PoolBase.java:411) ~[HikariCP-5.0.1.jar:na]
at com.zaxxer.hikari.pool.PoolBase.newConnection(PoolBase.java:364) ~[HikariCP-5.0.1.jar:na]

```  

默认引入，无需额外配置，即可使用

```xml
       <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
```

```yml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:9306
    username: root
    password:
    driver-class-name: com.mysql.cj.jdbc.Driver
```   

```java
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                                .url(dbUrl)
                                .username(username)
                                .password(password)
                                .driverClassName(driverClassName)
                                .build();
    }

@Bean
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
    return new JdbcTemplate(dataSource);
    }
```



### **2. Druid**

Druid 是阿里开源的数据库连接池，功能丰富，内置监控和防火墙功能。同样，在与 Manticoresearch 交互时也可能出现兼容性问题。以下是详细介绍。

报错如下：

```
         com.alibaba.druid.pool.DruidDataSource   : create connection holder error

java.sql.SQLException: Could not map transaction isolation '0' to a valid JDBC level.
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:130) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:98) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:90) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:64) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mysql.cj.jdbc.ConnectionImpl.getTransactionIsolation(ConnectionImpl.java:1183) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.alibaba.druid.pool.DruidConnectionHolder.<init>(DruidConnectionHolder.java:204) ~[druid-1.2.19.jar:na]
	at com.alibaba.druid.pool.DruidConnectionHolder.<init>(DruidConnectionHolder.java:93) ~[druid-1.2.19.jar:na]
	at com.alibaba.druid.pool.DruidDataSource.put(DruidDataSource.java:2631) ~[druid-1.2.19.jar:na]
	at com.alibaba.druid.pool.DruidDataSource$CreateConnectionThread.run(DruidDataSource.java:2989) ~[druid-1.2.19.jar:na]
```
   
```xml
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
        <exclusions>
                <exclusion>
                        <groupId>com.zaxxer</groupId>
                        <artifactId>HikariCP</artifactId>
                </exclusion>
        </exclusions>
</dependency>
    <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.19</version>
        </dependency>
```   

```yml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:9306
    username: root
    password:
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
```     

```java
            @Bean
    public DataSource dataSource(){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl(dbUrl);
        dataSource.setUsername(username);
        dataSource.setPassword(password);
        dataSource.setMaxActive(20);
        return dataSource;
    }

@Bean
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
        }
```

       
### **3. Tomcat JDBC**

Tomcat JDBC 是 Tomcat 内置的连接池，独立于容器也能使用，与 Manticoresearch 兼容性良好。下面为初学者展示如何在 Spring Boot 中使用它与 Manticore 交互。
  
正常查询

```xml
   <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.zaxxer</groupId>
                    <artifactId>HikariCP</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.apache.tomcat</groupId>
            <artifactId>tomcat-jdbc</artifactId>
            <version>11.0.0</version>
        </dependency>
```
       
```yml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:9306
    username: root
    password:
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: org.apache.tomcat.jdbc.pool.DataSource
```

```java
    @Bean
    public DataSource dataSource(){
        org.apache.tomcat.jdbc.pool.DataSource dataSource = new org.apache.tomcat.jdbc.pool.DataSource();
        dataSource.setUrl(dbUrl);
        dataSource.setUsername(username);
        dataSource.setPassword(password);
        dataSource.setMaxActive(20);
        return dataSource;
    }

@Bean
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
        }
```

### **4. Apache DBCP2**

Apache DBCP2 是 Apache 老牌连接池，稳定性高，与 Manticoresearch 兼容性不错。接下来介绍在 Spring Boot 中使用 DBCP2 连接 Manticore 的方法。

正常查询

```xml

    <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.zaxxer</groupId>
                    <artifactId>HikariCP</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-dbcp2</artifactId>
            <version>2.9.0</version>
        </dependency>
```


```yml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:9306
    username: root
    password:
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: org.apache.commons.dbcp2.BasicDataSource
```

```java
    @Bean
    public DataSource dataSource(){
        BasicDataSource dataSource = new BasicDataSource();
        dataSource.setUrl(dbUrl);
        dataSource.setUsername(username);
        dataSource.setPassword(password);
        dataSource.setMaxTotal(20); // 最大连接数
        return dataSource;
    }

@Bean
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
        }
```

### **5. C3P0**

C3P0 是历史悠久的数据库连接池，兼容性强，但性能相对较差。在与 Manticoresearch 交互时也存在兼容性问题。下面是详细情况。

报错如下：

```
 c.m.v2.resourcepool.BasicResourcePool    : com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@71b27e9d -- Acquisition Attempt Failed!!! Clearing pending acquires. While trying to acquire a needed new resource, we failed to succeed more than the maximum number of allowed acquisition attempts (30). Last acquisition attempt exception: 

java.sql.SQLException: Could not map transaction isolation '0' to a valid JDBC level.
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:130) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:98) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:90) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:64) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mysql.cj.jdbc.ConnectionImpl.getTransactionIsolation(ConnectionImpl.java:1183) ~[mysql-connector-j-8.2.0.jar:8.2.0]
	at com.mchange.v2.c3p0.impl.NewPooledConnection.<init>(NewPooledConnection.java:88) ~[c3p0-0.10.0.jar:na]
	at com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:166) ~[c3p0-0.10.0.jar:na]
	at com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:143) ~[c3p0-0.10.0.jar:na]
	at com.mchange.v2.c3p0.impl.C3P0PooledConnectionPool$1PooledConnectionResourcePoolManager.acquireResource(C3P0PooledConnectionPool.java:343) ~[c3p0-0.10.0.jar:na]
	at com.mchange.v2.resourcepool.BasicResourcePool.doAcquire(BasicResourcePool.java:1112) ~[c3p0-0.10.0.jar:na]
	at com.mchange.v2.resourcepool.BasicResourcePool.doAcquireAndDecrementPendingAcquiresWithinLockOnSuccess(BasicResourcePool.java:1099) ~[c3p0-0.10.0.jar:na]
	at com.mchange.v2.resourcepool.BasicResourcePool.access$700(BasicResourcePool.java:9) ~[c3p0-0.10.0.jar:na]
	at com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask.run(BasicResourcePool.java:1846) ~[c3p0-0.10.0.jar:na]
	at com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:696) ~[mchange-commons-java-0.3.0.jar:0.3.0]

```


```xml
     <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.zaxxer</groupId>
                    <artifactId>HikariCP</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.mchange</groupId>
            <artifactId>c3p0</artifactId>
            <version>0.10.0</version>
        </dependency>
```

```yml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:9306
    username: root
    password:
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.mchange.v2.c3p0.ComboPooledDataSource
```

```java
       @Bean
    public DataSource dataSource() throws PropertyVetoException {
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        dataSource.setDriverClass(driverClassName);
        dataSource.setJdbcUrl(dbUrl);
        dataSource.setUser(username);
        dataSource.setPassword(password);
        dataSource.setMaxPoolSize(20);
        return dataSource;
    }

@Bean 
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
        }

```

在 Java 中，`javax.sql.DataSource` 是 JDBC 规范定义的数据库连接池接口，有多种实现类型。对于初学者来说，了解这些主流连接池的核心区别，有助于选择合适的连接池与 Manticoresearch 交互。以下是详细对比：

---

## 各个数据库连接池的特点

### **1. HikariCP**
- **特点**：**性能最优**（轻量级、无锁设计），Spring Boot 默认连接池。
- **优势**：
        - 极简设计，代码量少（130KB）
        - 并发性能高（如 `ConcurrentBag` 优化）
        - 自动规避连接泄漏
- **适用场景**：高并发、云原生应用。
- **配置示例**：
  ```java
  HikariConfig config = new HikariConfig();
  config.setJdbcUrl("jdbc:mysql://localhost:3306/db");
  config.setUsername("user");
  config.setPassword("pass");
  config.setMaximumPoolSize(20);
  DataSource dataSource = new HikariDataSource(config);
  ```

---

### **2. Apache DBCP2**
- **特点**：Apache 老牌连接池（Tomcat 内置实现）。
- **优势**：
        - 成熟稳定，功能完备
        - 支持 JMX 监控
- **缺点**：性能低于 HikariCP。
- **适用场景**：传统企业级应用。
- **配置示例**：
  ```java
  BasicDataSource dataSource = new BasicDataSource();
  dataSource.setUrl("jdbc:mysql://localhost:3306/db");
  dataSource.setUsername("user");
  dataSource.setPassword("pass");
  dataSource.setMaxTotal(20); // 最大连接数
  ```

---

### **3. C3P0**
- **特点**：历史悠久的连接池（早期 Hibernate 默认）。
- **优势**：
        - 兼容性极佳（支持旧 JDK）
        - 自动重连机制完善
- **缺点**：性能较差，配置复杂。
- **适用场景**：遗留系统维护。
- **配置示例**：
  ```java
  ComboPooledDataSource dataSource = new ComboPooledDataSource();
  dataSource.setDriverClass("com.mysql.cj.jdbc.Driver");
  dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/db");
  dataSource.setUser("user");
  dataSource.setPassword("pass");
  dataSource.setMaxPoolSize(20);
  ```

---

### **4. Druid**
- **特点**：阿里开源，**功能最丰富**（监控、安全）。
- **优势**：
        - 内置 SQL 监控/防火墙/慢查询日志
        - 支持扩展（如加密、JSON 操作）
        - 性能接近 HikariCP
- **适用场景**：需要监控和安全保障的系统。
- **配置示例**：
  ```java
  DruidDataSource dataSource = new DruidDataSource();
  dataSource.setUrl("jdbc:mysql://localhost:3306/db");
  dataSource.setUsername("user");
  dataSource.setPassword("pass");
  dataSource.setMaxActive(20);
  // 开启监控
  dataSource.setFilters("stat,wall");
  ```

---

### **5. Tomcat JDBC Pool**
- **特点**：Tomcat 内置连接池（独立于容器使用）。
- **优势**：
        - 轻量高效（性能优于 DBCP）
        - 支持异步获取连接
- **适用场景**：Tomcat 项目或需要轻量级替代方案。
- **配置示例**：
  ```java
  org.apache.tomcat.jdbc.pool.DataSource dataSource = new org.apache.tomcat.jdbc.pool.DataSource();
  dataSource.setUrl("jdbc:mysql://localhost:3306/db");
  dataSource.setUsername("user");
  dataSource.setPassword("pass");
  dataSource.setMaxActive(20);
  ```

---

### **核心区别总结**
| **特性**         | HikariCP       | Druid          | DBCP2          | C3P0           | Tomcat Pool    |
|------------------|----------------|----------------|----------------|----------------|----------------|
| **性能**         | ⭐⭐⭐⭐⭐ (最高)   | ⭐⭐⭐⭐          | ⭐⭐⭐           | ⭐⭐             | ⭐⭐⭐⭐          |
| **监控功能**     | 基础           | ⭐⭐⭐⭐⭐ (强大)  | 支持 JMX       | 有限           | 基础           |
| **连接泄漏检测** | 自动回收       | 支持           | 支持           | 支持           | 支持           |
| **代码体积**     | 极小 (130KB)   | 较大           | 中等           | 较大           | 中等           |
| **流行度**       | Spring Boot 默认 | 国内广泛       | 老牌稳定       | 逐渐淘汰       | Tomcat 项目    |
| **适用场景**     | 高并发云原生   | 全功能+监控    | 传统企业应用   | 旧系统维护     | Tomcat 环境    |

---

### **选择建议**

对于初学者来说，选择合适的数据库连接池能让 Spring Boot 与 Manticoresearch 的交互更加顺畅。根据上述特点和兼容性测试结果，我们给出以下选择建议：
1. **性能优先** → 选 `HikariCP`
2. **需要监控/安全** → 选 `Druid`
3. **Tomcat 环境** → 考虑 `Tomcat JDBC Pool`
4. **维护旧系统** → `DBCP2` 或 `C3P0`

> 💡 **Spring Boot 默认顺序**：
> 1. HikariCP → 2. Tomcat JDBC Pool → 3. DBCP2

----

## 总结

综上所述，要Springboot结合数据库连接池调用Manticoresearch进行数据的查询的话，目前能够适配的是`Tomcat JDBC Pool`和`DBCP2`，`HikariCP`和`Druid`这种主流的话，由于存在事务的校验，都无法很好的适配。对于可用的两款，推荐`Tomcat JDBC Pool`。