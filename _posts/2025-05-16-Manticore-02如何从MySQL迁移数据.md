---
layout:     post
title:      将关系型数据从MySQL迁移到Manticore Search

subtitle:   Manticore Search
date:       2025-05-16
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Manticore Search
    - MySQL
---

将MySQL中的历史数据同步到Manticore Search可以通过多种方法实现，具体选择取决于你的需求和环境。以下是常见的几种方法及其步骤和注意事项：

- [方法一：使用SQL导入(主流推荐)](#方法一使用sql导入主流推荐)
- [方法二：使用ETL工具](#方法二使用etl工具)
- [方法三：使用脚本编写自定义同步](#方法三使用脚本编写自定义同步)

## 方法一：使用SQL导入(主流推荐)

这个方法是最常用的一种方法，也是最简单的一种方法。流程可控，数据可验证。

步骤：

1. 导出MySQL数据：

使用 mysqldump 或其他工具将MySQL表导出为SQL文件。

这里需要注意，如果字段类型能够一一对应，是常规的数据形式，就可以快速导入。
> 但是Manticore中，对于时间类型的字段，和MySQL中有较大不同，需要转换。datetime 如果是常规到秒级别的，可以导出转为时间戳类型，则可以快速导入Manticore，如果是到毫秒级别的，则需要转换成13位Long类型数值，并且Manticore中必须是bigint类型才能接收，额外还需要注意不能是float类型(xxxxxxxxxxxxx.000)，否则精度会异常。

示例命令：

```bash
mysqldump -h IP  -u user -D dbname -p --complete-insert --no-create-info --skip-triggers --skip-add-drop-table tmptablename > tmptablename.sql  
```

此指令将数据导入为INSERT 语句，便于使用Manticore的mysql客户端导入。

2. 转换SQL文件：

如果需要，调整SQL文件以适应Manticore的语法要求。
上面导出SQL语句的方法，基本是就是最原始的类手动操作，兼容性是比较高的，但是需要注意，尽量在导出文件之前，就做好所有字段的转换处理，变成文件之后再修改，数据量大是不推荐的。

3. 导入到Manticore：

在 Manticore 准备好表结构，因为Manticore有自动建表的功能，所以不提前建立表结构，会导致表字段类型自动化识别错误，导致不必要的问题。  

使用Manticore的 mysql 客户端连接到Manticore并执行SQL文件。

```bash
mysql -h IP -P port -u user < tmptablename.sql 
```

Manticore 没有密码和数据库形式的概念(目前，已知权限功能已在开发中)，指定对应的数据就可以导入了。

注意：

1.注意秒类型时间数据的同步
2.注意毫秒类型时间数据的同步，确定导出是整数类型，不要带浮点数，否则精度可能缺失
3.注意时间类型数据的时区问题
4.注意特殊MVA类型的数据无法通过这种方式很好地同步

## 方法二：使用ETL工具

官方提供了Logstash的数据同步方式，支持常规的读取、转换和写入流程，此外如果简单的转换不能够满足，只能给结合Kafka和应用来实现传输、转换与写入的流程了。

## 方法三：使用脚本编写自定义同步

这个方法是最复杂的一种方法，需要你手动编写代码，并且需要你熟悉编程语言。

对于大表MySQL是很痛苦的，因为数据的深分页读取可能会拖垮整个数据库，但是对于使用Manticore MVA 特殊类型的，无法通过SQL将数据进行转换的，只能通过程序来清洗。
