---
layout:     post
title:      SpringBoot+Druid+Sqlite最佳实践

subtitle:   SpringBoot+Druid+Sqlite
date:       2024-05-11
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SpringBoot
    - Druid
    - SQLite
---

一次以外的机会接触到了SQLite这样一个轻量型的嵌入式数据库组件，对于日常使用的可能都是传统RDBMS，或者当下更为流行NoSQL和大数据存储

对于这样一个自包含的，无服务器的，零配置的，事务性的 SQL 数据库引擎，主要是面向嵌入式或者终端类的使用场景，或者是说开箱即用、小型项目的场景

- [关于SQLite - 世界上装机量最多的数据库？](#关于sqlite---世界上装机量最多的数据库)
  - [关于Druid](#关于druid)
  - [关于SpringBoot](#关于springboot)
  - [整合并查询](#整合并查询)
    - [创建DB文件](#创建db文件)
    - [初始化一个SpringBoot程序](#初始化一个springboot程序)
    - [导入Maven依赖](#导入maven依赖)
    - [配置服务连接参数](#配置服务连接参数)
    - [书写一个JPA程序完成一次查询](#书写一个jpa程序完成一次查询)

## 关于SQLite - 世界上装机量最多的数据库？

- 使用C语言开发，使得它小巧精致而高效，直接采用偏底层的语言，使用文件的逻辑，实现SQL数据库的逻辑
- 使用方：包括但不限于 Python、Java、C# 等
- 无服务器的，零配置的，真的轻量
- 遵守ACID的关系型数据库管理系统，让了解Mysql等数据库的人极易上手
- SQLite 是一个自包含的程序如果你使用的是 Linux 或者 MacOS，那么 SQLite 很可能已经预装了，真的开箱即用

```bash
:~$ sqlite3
SQLite version 3.28.0 2019-04-15 14:49:49
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> 
```

它虽然支持了增删改查以及事务等特性，但是它为什么是在嵌入式方向使用，而不是大范围使用呢，主要还是不面向复杂的处理逻辑、大数量的存储与查询，所以还是要将对的东西用在对的方面

### 关于Druid

Alibaba/Druid 想必大家肯定不陌生，一个国内使用率比较高的一个数据库连接池，其他的还有：HikariCP, tomcat-jdbc, c3p0, dbcp，用的人多也就迭代和丰富了很多功能。
定位为：为监控而生的数据库连接池，接入了一个高性能的数据库连接池，还送了一套数据库操作的监控，肯定会让人心动

它也是非常自信的与其他数据库连接池做了比对，具体内容参看：[Druid与其他数据库连接池的比对](./2024-05-11-DataSource-数据库连接池Druid.md)

支持的数据库种类繁多，可以适配国内数据库

### 关于SpringBoot

这个不用多说，Java开发者应该无人不知无人不晓

### 整合并查询

#### 创建DB文件

首先登录查看操作指南，需要记住命令都是"."开头，`.help`展示帮助指令，`.open filename`创建一个数据库， `.quit` 退出命令行

```bash
:~$ sqlite3
SQLite version 3.28.0 2019-04-15 14:49:49
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .help
.auth ON|OFF             Show authorizer callbacks
.backup ?DB? FILE        Backup DB (default "main") to FILE
.bail on|off             Stop after hitting an error.  Default OFF
.binary on|off           Turn binary output on or off.  Default OFF
.cd DIRECTORY            Change the working directory to DIRECTORY
.changes on|off          Show number of rows changed by SQL
.check GLOB              Fail if output since .testcase does not match
.clone NEWDB             Clone data into NEWDB from the existing database
.databases               List names and files of attached databases
.dbconfig ?op? ?val?     List or change sqlite3_db_config() options
.dbinfo ?DB?             Show status information about the database
.dump ?TABLE? ...        Render all database content as SQL
.echo on|off             Turn command echo on or off
.eqp on|off|full|...     Enable or disable automatic EXPLAIN QUERY PLAN
.excel                   Display the output of next command in a spreadsheet
.exit ?CODE?             Exit this program with return-code CODE
.expert                  EXPERIMENTAL. Suggest indexes for specified queries
.fullschema ?--indent?   Show schema and the content of sqlite_stat tables
.headers on|off          Turn display of headers on or off
.help ?-all? ?PATTERN?   Show help text for PATTERN
.import FILE TABLE       Import data from FILE into TABLE
.imposter INDEX TABLE    Create imposter table TABLE on index INDEX
.indexes ?TABLE?         Show names of indexes
.limit ?LIMIT? ?VAL?     Display or change the value of an SQLITE_LIMIT
.lint OPTIONS            Report potential schema issues.
.log FILE|off            Turn logging on or off.  FILE can be stderr/stdout
.mode MODE ?TABLE?       Set output mode
.nullvalue STRING        Use STRING in place of NULL values
.once (-e|-x|FILE)       Output for the next SQL command only to FILE
.open ?OPTIONS? ?FILE?   Close existing database and reopen FILE
.output ?FILE?           Send output to FILE or stdout if FILE is omitted
.parameter CMD ...       Manage SQL parameter bindings
.print STRING...         Print literal STRING
.progress N              Invoke progress handler after every N opcodes
.prompt MAIN CONTINUE    Replace the standard prompts
.quit                    Exit this program
.read FILE               Read input from FILE
.restore ?DB? FILE       Restore content of DB (default "main") from FILE
.save FILE               Write in-memory database into FILE
.scanstats on|off        Turn sqlite3_stmt_scanstatus() metrics on or off
.schema ?PATTERN?        Show the CREATE statements matching PATTERN
.selftest ?OPTIONS?      Run tests defined in the SELFTEST table
.separator COL ?ROW?     Change the column and row separators
.session ?NAME? CMD ...  Create or control sessions
.sha3sum ...             Compute a SHA3 hash of database content
.shell CMD ARGS...       Run CMD ARGS... in a system shell
.show                    Show the current values for various settings
.stats ?on|off?          Show stats or turn stats on or off
.system CMD ARGS...      Run CMD ARGS... in a system shell
.tables ?TABLE?          List names of tables matching LIKE pattern TABLE
.testcase NAME           Begin redirecting output to 'testcase-out.txt'
.timeout MS              Try opening locked tables for MS milliseconds
.timer on|off            Turn SQL timer on or off
.trace ?OPTIONS?         Output each SQL statement as it is run
.vfsinfo ?AUX?           Information about the top-level VFS
.vfslist                 List all available VFSes
.vfsname ?AUX?           Print the name of the VFS stack
.width NUM1 NUM2 ...     Set column widths for "column" mode
sqlite> 
```

创建一个mydb

```bash
sqlite> .open mydb
sqlite> .databases
main: /Users/xxx/mydb
```

非常简单地创建了一个DB文件

#### 初始化一个SpringBoot程序

使用`start.spring.io` 或者其他的初始化方式，建立一个简单的SpringBoot项目。

#### 导入Maven依赖

maven里面主要需要，springboot-jpa、druid、sqlite-jdbc、sqlite-dialect(RDBMS 方言)

- springboot-jpa: 这个如果使用mybatis/mybatis-plus等ORM框架，都是可以的
- druid: 这边使用druid-spring-boot-starter，主要为了便捷，单用druid自己手动去配置也是可以的
- sqlite-jdbc：sqlite对于连接的实现，必须
- sqlite-dialect：是结合JPA所需的，对于Hibernate的方言，这边采用maven库中2023年更新的依赖包

以上的配置，最关键的就是版本的依赖关系，Hibernete5?Hibernete6 + SpringBoot

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>sqlitedemo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>sqlitedemo</name>
    <description>sqlitedemo</description>
    <properties>
        <java.version>17</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>2.6.13</spring-boot.version>
    </properties>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-log4j2</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.20</version>
        </dependency>

        <dependency>
            <groupId>org.xerial</groupId>
            <artifactId>sqlite-jdbc</artifactId>
            <version>3.42.0.0</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.2.19</version>
        </dependency>

        <dependency>
            <groupId>com.github.gwenn</groupId>
            <artifactId>sqlite-dialect</artifactId>
            <version>0.1.4</version>
        </dependency>
<!--        <dependency>-->
<!--            <groupId>com.zsoltfabok</groupId>-->
<!--            <artifactId>sqlite-dialect</artifactId>-->
<!--            <version>1.0</version>-->
<!--        </dependency>-->
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${spring-boot.version}</version>
                <configuration>
                    <mainClass>com.example.sqlitedemo.SqlitedemoApplication</mainClass>
                    <skip>true</skip>
                </configuration>
                <executions>
                    <execution>
                        <id>repackage</id>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

```

#### 配置服务连接参数

application.properties

```yaml
server.port=8080
spring.application.name=sqlite-test

spring.datasource.druid.url=jdbc:sqlite::resource:db/sqlite.db
spring.datasource.druid.username=
spring.datasource.druid.password=
spring.datasource.druid.driver-class-name=org.sqlite.JDBC
spring.datasource.druid.initial-size=5
spring.datasource.druid.min-idle=5
spring.datasource.druid.max-active=20
spring.datasource.druid.max-wait=60000
spring.datasource.druid.pool-prepared-statements=false
spring.datasource.druid.max-pool-prepared-statement-per-connection-size=-1
spring.datasource.druid.validation-query=SELECT '1' from sqlite_master
spring.datasource.druid.validation-query-timeout=3
spring.datasource.druid.test-on-borrow=true
spring.datasource.druid.test-on-return=false
spring.datasource.druid.test-while-idle=true
spring.datasource.druid.time-between-eviction-runs-millis=60000
spring.datasource.druid.min-evictable-idle-time-millis=300000
spring.datasource.druid.filters= stat

# WebStatFilter
spring.datasource.druid.web-stat-filter.enabled=false
spring.datasource.druid.web-stat-filter.url-pattern=/*
spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*
spring.datasource.druid.web-stat-filter.session-stat-enable=true
spring.datasource.druid.web-stat-filter.session-stat-max-count=1000
spring.datasource.druid.web-stat-filter.principal-session-name=
spring.datasource.druid.web-stat-filter.principal-cookie-name=
spring.datasource.druid.web-stat-filter.profile-enable=

# StatViewServlet
spring.datasource.druid.stat-view-servlet.enabled=false
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
spring.datasource.druid.stat-view-servlet.reset-enable=false
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=123456
spring.datasource.druid.stat-view-servlet.allow=
spring.datasource.druid.stat-view-servlet.deny=

# Spring
spring.datasource.druid.aop-patterns= com.*.service.*

# Dialect(com.github.gwenn.sqlite-dialect)
spring.jpa.database-platform=org.sqlite.hibernate.dialect.SQLiteDialect
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=none
spring.jpa.hibernate.naming.strategy=org.hibernate.cfg.ImprovedNamingStrategy
spring.jpa.properties.hibernate.dialect=org.sqlite.hibernate.dialect.SQLiteDialect
spring.jpa.properties.hibernate.event.merge.entity_copy_observer=allow

spring.aop.proxy-target-class=true
```

注意这里配置的DB的地址，与你刚才新建的DB地址要一致

#### 书写一个JPA程序完成一次查询

- 通过命令行创建一个table

```bash
sqlite> CREATE TABLE user
   ...> (
   ...>   id    VARCHAR(8),
   ...>   name    VARCHAR(30)
   ...> );
sqlite> 
```

- 定义domain

```java
@Entity
@Table(name = "user", schema = "main", catalog = "")
public class User {
    private String id;
    private String name;
    private String age;

    @Id
    @Column(name = "id", nullable = true, length = 8)
    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    @Basic
    @Column(name = "name", nullable = true, length = 30)
    public String getName() {
        return stationId;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

- 定义service

```java
public interface UserService {
    List<User> findList();
}

@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private UserRepo userRepo;


    @Override
    public List<User> findList() {
        return userRepo.findAllByNameNotNull();
    }
}
```

- 定义dao

```java
public interface UserRepo extends JpaRepository<User,String> {

    List<User> findAllByNameNotNull();
}
```

- 测试

启动服务，书写接口测试

```log
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::               (v2.6.13)

2024-05-10 10:29:41.354  INFO 49697 --- [           main] c.e.sqlitedemo.SqlitedemoApplication     : Starting SqlitedemoApplication using Java 17.0.8 
2024-05-10 10:29:41.357  INFO 49697 --- [           main] c.e.sqlitedemo.SqlitedemoApplication     : No active profile set, falling back to 1 default profile: "default"
2024-05-10 10:29:42.630  INFO 49697 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFAULT mode.
2024-05-10 10:29:42.736  INFO 49697 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 81 ms. Found 1 JPA repository interfaces.
2024-05-10 10:29:43.141  INFO 49697 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'com.alibaba.druid.spring.boot.autoconfigure.stat.DruidSpringAopConfiguration' of type [com.alibaba.druid.spring.boot.autoconfigure.stat.DruidSpringAopConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2024-05-10 10:29:43.176  INFO 49697 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'spring.datasource.druid-com.alibaba.druid.spring.boot.autoconfigure.properties.DruidStatProperties' of type [com.alibaba.druid.spring.boot.autoconfigure.properties.DruidStatProperties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2024-05-10 10:29:43.186  INFO 49697 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'advisor' of type [org.springframework.aop.support.RegexpMethodPointcutAdvisor] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2024-05-10 10:29:43.570  INFO 49697 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
2024-05-10 10:29:43.582  INFO 49697 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2024-05-10 10:29:43.582  INFO 49697 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.68]
2024-05-10 10:29:43.752  INFO 49697 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2024-05-10 10:29:43.752  INFO 49697 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2317 ms
2024-05-10 10:29:43.843  INFO 49697 --- [           main] c.a.d.s.b.a.DruidDataSourceAutoConfigure : Init DruidDataSource
2024-05-10 10:29:44.533  INFO 49697 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
2024-05-10 10:29:44.750  INFO 49697 --- [           main] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2024-05-10 10:29:44.873  INFO 49697 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 5.6.12.Final
2024-05-10 10:29:45.276  INFO 49697 --- [           main] o.hibernate.annotations.common.Version   : HCANN000001: Hibernate Commons Annotations {5.1.2.Final}
2024-05-10 10:29:45.563  INFO 49697 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.sqlite.hibernate.dialect.SQLiteDialect
2024-05-10 10:29:45.667  INFO 49697 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.sqlite.hibernate.dialect.SQLiteDialect
2024-05-10 10:29:46.753  INFO 49697 --- [           main] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
2024-05-10 10:29:46.771  INFO 49697 --- [           main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2024-05-10 10:29:47.403  WARN 49697 --- [           main] JpaBaseConfiguration$JpaWebConfiguration : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
2024-05-10 10:29:47.746  INFO 49697 --- [           main] o.s.b.a.w.s.WelcomePageHandlerMapping    : Adding welcome page: class path resource [static/index.html]
2024-05-10 10:29:48.028  INFO 49697 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2024-05-10 10:29:48.040  INFO 49697 --- [           main] c.e.sqlitedemo.SqlitedemoApplication     : Started SqlitedemoApplication in 7.664 seconds (JVM running for 11.35)
2024-05-10 10:30:12.802  INFO 49697 --- [nio-8081-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2024-05-10 10:30:12.802  INFO 49697 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2024-05-10 10:30:12.804  INFO 49697 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 2 ms
Hibernate: select user0_.id as u_1_0_, user0_.name as name2_0_ from user user0_ where user0_.name is not null 
```

能够正常访问

以上SQLite初体验到此结束~~

----
如果喜欢我的文章的话，可以去[GitHub上给一个免费的关注](https://github.com/CzyerChen/)吗？
