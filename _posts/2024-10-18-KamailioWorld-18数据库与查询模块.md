---
layout:     post
title:      Kamailio-与数据库的交互

subtitle:   Kamailio MySQL PostgreSQL JDBC
date:       2024-10-21
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kamailio
    - MySQL
    - PostgreSQL
    - JDBC
---

对于Kamailio连接数据库，今天介绍3种可用的工具，来解决一些问题：

- db_mysql：依托MySQL广大的拥护者，KM也不例外的有它的支持，而且是入门演示默认的选项
- db_postgres：大规模数据的交互和分析，PosgreSQL 会比 MySQL更加得心应手，需要大数据量级支持的，可以考虑
- sqlops：jdbcclient，帮你利用JDBC查询额外的数据

- [db\_mysql](#db_mysql)
- [db\_postgres](#db_postgres)
- [数据库使用的样例](#数据库使用的样例)
- [sqlops](#sqlops)
  - [数据连接](#数据连接)
  - [获取数据](#获取数据)
  - [其他属性](#其他属性)
  - [如何查询？](#如何查询)
  - [一些伪变量](#一些伪变量)

## db_mysql

使用逻辑很简单，引入模块、配置连接信息、调用函数进行增删改查

加载这个模块需要额外的开发库版本支持，一些Linux服务器上是`"libmysqlclient-dev"`，如果是MariaDB的话，叫`"libmariadbclient-dev"`.

- ping_interval (integer) 存活检测，单位是秒，默认5 min
- server_timezone 时区，如果Kamailio和数据库不在同一个地区，那么这就是必要的
- timeout_interval 超时时间，包括连接超时、读写超时，系统会重试3次，默认2秒/次，也就是默认超时是6秒
- auto_reconnect 是否自动重连，默认是1，自动重连
- insert_delayed 是否延迟插入，默认是0，不延迟，如果设置为1，所有去MySQL的插入都是延迟的
- update_affected_found (integer) update操作是否返回影响的行数，默认不返回，那么你只能得到一个updated表示执行过了的结果，而不是准确的行数
- opt_ssl_mode 连接ssl模式，1开启，0关闭
- opt_ssl_ca ssl模式的ca证书

MySQL作为一个基础的存储截止，在其余模块中可以引用，使用模块内的逻辑读写对应的表
后续举例

## db_postgres

posgresql是类似的，也是需要额外的开发库来支持连接

- ostgreSQL library - e.g., libpq5.
- PostgreSQL devel library - to compile the module (e.g., libpq-dev).

具体看你使用的Linux系统进行安装即可

同样也是包含了重试、超时等基础参数

- retries (integer) 重试次数，默认是2
- timeout (integer) 超时时间
- tcp_keepalive (integer) 主要针对于“TCP_KEEPIDLE” socket 选项
- lockset (integer) 锁的数量，默认是2的次方，默认值4，也就是16
- bytea_output_escape (integer) 是否应请求转义 bytea 字段的输出
- con_param (str) url连接额外参数 比如：connect_timeout=15;tcp_user_timeout=5000

使用也是同MySQL，后续举例

## 数据库使用的样例

话单模块，直接引入DBURL，即可使用数据库模块进行库表数据的读写

`modparam("acc", "db_url", DBURL)`

其他模块也是直接引入，做数据存储：

`modparam("usrloc", "db_url", DBURL)`
`modparam("auth_db", "db_url", DBURL)`
`modparam("permissions", "db_url", DBURL)`

## sqlops

这个是用于原始jdbc sql查询的组件，可以支持：

- 多数据库连接
- 多个查询结果
- 通过伪变量获取数据
- 通过数组获取结果
- 在同一个工作进程中，一个结果可以被多次使用
- 查询结果可以转存xavps里，虽事务上下文被清理，减少手动处理

不需要额外的开发库，引入后即可使用

### 数据连接

```bash
modparam("sqlops","sqlcon","cb=>mysql://kamailio:kamailiorw@localhost/kamailio")
modparam("sqlops","sqlcon","ca=>dbdriver://username:password@dbhost/dbname")
```

- cb,ca是连接的名称，用于后续使用，命名要浅显易懂直观
- 紧接着就是数据库的常规连接信息

### 获取数据

`sqlres (str)`，设置SQL返回数据集的结果ID

`results_maxsize (int)` 结果数量的上限，默认32

### 其他属性 

`tr_buf_size (int)`，SQL操作转换大小，默认2048

`log_buf_size (int)` 记录原始 SQL 操作时的缓冲区大小 （字符），默认128

`connect_mode (int)` 0-启动时连接失败则启动失败，1-启动时连接失败依旧启动

### 如何查询？

`sql_query(), sql_xquery() and sql_pvquery()` 三种查询方式，参数主要就是`connection, query[, result]`，提供连接信息，书写查询条件，获取查询结果。三个主要是在最终获取查询结果的环节有差异。

```bash
modparam("sqlops","sqlcon","ca=>dbdriver://username:password@dbhost/dbname")

#获取一次，需要释放
sql_query("ca", "select * from domain", "ra");
xlog("number of rows in table domain: $dbr(ra=>rows)\n");
sql_result_free("ra");

#可获取多次，与事务绑定，无需手动释放
sql_xquery("ca", "select * from domain", "ra");
xlog("first domain: $xavp(ra=>domain) with id: $xavp(ra=>domain_id)\n");
...
if (sql_xquery("ca", "select * from domain", "ra") == 1) {
    xlog("domain: $xavp(ra=>domain) with id: $xavp(ra=>domain_id)\n");
}

#可一次性获取多个参数值，承载结果的对象必须是可写入的
sql_pvquery("ca", "select 'col1', 2, NULL, 'sip:test@example.com'",
	"$var(a), $avp(col2), $xavp(item[0]=>s), $ru");
```

`sql_result_free(result)` 用于释放查询结果

`sql_query_async` 异步查询
 
### 一些伪变量

在查询的时候，也有看到，会出现 `$dbr(ra=>rows)` 这样的取值方式，那么有哪些其他的参数值可选呢？

- rows - 返回查询的数量
- cols - 返回查询结果的列数
- [row,col] - 返回指定【行，列】的数据，类似二维数组
- colname[N] - 返回指定某一列的数据集

`$sqlrows(con)` 通常用于返回在insert\update\delete操作下影响的数据库行数
