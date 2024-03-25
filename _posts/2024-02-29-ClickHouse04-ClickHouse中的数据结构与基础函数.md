---
layout:     post
title:      ClickHouse04-ClickHouse基础数据类型与函数
subtitle:   ClickHouse
date:       2024-03-18
author:     Claire
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - ClickHouse
---

在认识一个数据库时，必须需要了解它所包含的数据结构有哪些，这样才能辅助进行表结构的设计。

相比传统的数据库，它有什么类似的数据结构，它又有什么特殊的数据结构呢？

- [数据类型](#数据类型)
  - [整型](#整型)
  - [浮点型](#浮点型)
  - [布尔型](#布尔型)
  - [字符型](#字符型)
  - [时间型](#时间型)
  - [枚举型](#枚举型)
  - [空间数据](#空间数据)
  - [JSON](#json)
  - [UUID](#uuid)
  - [Arrays/Maps/元组(Tuple)/Set](#arraysmaps元组tupleset)
  - [IP地址](#ip地址)
  - [嵌套数据型](#嵌套数据型)
  - [聚合数据型](#聚合数据型)
  - [其他特殊的数据结构](#其他特殊的数据结构)
- [基础函数](#基础函数)
  - [算术函数](#算术函数)
  - [时间日期函数](#时间日期函数)
  - [数组函数](#数组函数)
  - [比较函数](#比较函数)
  - [JSON函数](#json函数)
  - [字符串操作类函数](#字符串操作类函数)
    - [基础操作](#基础操作)
    - [替换](#替换)
    - [查询定位](#查询定位)
    - [分解](#分解)
  - [UDF 用户自定义函数](#udf-用户自定义函数)
  - [更多函数](#更多函数)

## 数据类型

以下通过与 MySQL 和 PostgreSQL 横向对比一下常见的数据结构

|类型|ClickHouse|MySQL|PostgreSQL|
|整型|UInt8, UInt16, UInt32, UInt64, UInt128, UInt256, Int8, Int16, Int32, Int64, Int128, Int256|TINYINT\SMALLINT\MEDIUMINT\INT\INTEGER\BIGINT|smallint\integer\bigint|
|浮点型|Float32\Float64\Decimal|FLOAT\DOUBLE\DECIMAL|decimal\numeric|
|布尔型| Bool|boolean|boolean|
|字符型|String\FixedString|CHAR\VARCHAR\TINYBLOB\TINYTEXT\BLOB\TEXT...|varchar\char\text|
|时间类型|Date\Date32\DateTime\DateTime64|DATE\TIME\DATETIME\TIMESTAMP|timestamp\date\interval|
|枚举|Enum|ENUM\SET|ENUM|
|空间数据|geo_point|GEOMETRY, POINT, LINESTRING, POLYGON, MULTIPOINT, MULTILINESTRING, MULTIPOLYGON, GEOMETRYCOLLECTION|point\line\lseg\box\path\polygon\circle|

### 整型

signed int 数值范围：

- Int8 — [-128 : 127]
- Int16 — [-32768 : 32767]
- Int32 — [-2147483648 : 2147483647]
- Int64 — [-9223372036854775808 : 9223372036854775807]
- Int128 — [-170141183460469231731687303715884105728 : 170141183460469231731687303715884105727]
- Int256 — [-57896044618658097711785492504343953926634992332820282019728792003956564819968 : 57896044618658097711785492504343953926634992332820282019728792003956564819967]

别名:

- Int8 — TINYINT, INT1, BYTE, TINYINT SIGNED, INT1 SIGNED.
- Int16 — SMALLINT, SMALLINT SIGNED.
- Int32 — INT, INTEGER, MEDIUMINT, MEDIUMINT SIGNED, INT SIGNED, INTEGER SIGNED.
- Int64 — BIGINT, SIGNED, BIGINT SIGNED, TIME.
- UInt Ranges
- UInt8 — [0 : 255]
- UInt16 — [0 : 65535]
- UInt32 — [0 : 4294967295]
- UInt64 — [0 : 18446744073709551615]
- UInt128 — [0 : 340282366920938463463374607431768211455]
- UInt256 — [0 : 115792089237316195423570985008687907853269984665640564039457584007913129639935]

Unsigned int 数值范围：

- UInt8 — [0 : 255]
- UInt16 — [0 : 65535]
- UInt32 — [0 : 4294967295]
- UInt64 — [0 : 18446744073709551615]
- UInt128 — [0 : 340282366920938463463374607431768211455]
- UInt256 — [0 : 115792089237316195423570985008687907853269984665640564039457584007913129639935]

别名:

- UInt8 — TINYINT UNSIGNED, INT1 UNSIGNED.
- UInt16 — SMALLINT UNSIGNED.
- UInt32 — MEDIUMINT UNSIGNED, INT UNSIGNED, INTEGER UNSIGNED
- UInt64 — UNSIGNED, BIGINT UNSIGNED, BIT, SET

### 浮点型

Float 对应关系：

- Float32 — float.
- Float64 — double.

别名:

- Float32 — FLOAT, REAL, SINGLE.
- Float64 — DOUBLE, DOUBLE PRECISION.

```sql
CREATE TABLE IF NOT EXISTS float_vs_decimal
(
   my_float Float64,
   my_decimal Decimal64(3)
)Engine=MergeTree ORDER BY tuple()

INSERT INTO float_vs_decimal SELECT round(randCanonical(), 3) AS res, res FROM system.numbers LIMIT 1000000; # Generate 1 000 000 random number with 2 decimal places and store them as a float and as a decimal

SELECT sum(my_float), sum(my_decimal) FROM float_vs_decimal;
> 500279.56300000014    500279.563
```

Decimal(P, S)， P - 精，有效范围： [ 1 ： 76 ]，S - 刻度，有效范围：[ 0 ： P ]:

- P from [ 1 : 9 ] - for Decimal32(S)
- P from [ 10 : 18 ] - for Decimal64(S)
- P from [ 19 : 38 ] - for Decimal128(S)
- P from [ 39 : 76 ] - for Decimal256(S)

Decimal 代表的数值范围：

- Decimal32(S) - ( -1 * 10^(9 - S), 1 * 10^(9 - S) )
- Decimal64(S) - ( -1 * 10^(18 - S), 1 * 10^(18 - S) )
- Decimal128(S) - ( -1 * 10^(38 - S), 1 * 10^(38 - S) )
- Decimal256(S) - ( -1 * 10^(76 - S), 1 * 10^(76 - S) )

### 布尔型

Bool值：true (1), false (0).

```sql
CREATE TABLE test_bool
(
    `A` Int64,
    `B` Bool
)
ENGINE = Memory;

INSERT INTO test_bool VALUES (1, true),(2,0);

SELECT * FROM test_bool;
┌─A─┬─B─────┐
│ 1 │ true  │
│ 2 │ false │
└───┴───────┘
```

### 字符型

String 类型有： LONGTEXT, MEDIUMTEXT, TINYTEXT, TEXT, LONGBLOB, MEDIUMBLOB, TINYBLOB, BLOB, VARCHAR, CHAR

定长String，例如：FixedString(16) for MD5, FixedString(32) for SHA256

### 时间型

常规使用：

```sql
CREATE TABLE dt
(
    `timestamp` Date,
    `event_id` UInt8
)
ENGINE = TinyLog;
```

### 枚举型

```sql
CREATE TABLE t_enum
(
    x Enum('hello' = 1, 'world' = 2)
)
ENGINE = TinyLog
```

### 空间数据

```sql
CREATE TABLE geo_point (p Point) ENGINE = Memory();
INSERT INTO geo_point VALUES((10, 10));
SELECT p, toTypeName(p) FROM geo_point;
```

### JSON

类似PostgreSQL:

```sql
CREATE TABLE json
(
    o JSON
)
ENGINE = Memory

INSERT INTO json VALUES ('{"a": 1, "b": { "c": 2, "d": [1, 2, 3] }}')
```

### UUID

类似于PostgreSQL，对分布式系统而言，这种标识符比序列能更好的提供唯一性保证

```sql
CREATE TABLE t_uuid (x UUID, y String) ENGINE=TinyLog

INSERT INTO t_uuid SELECT generateUUIDv4(), 'Example 1'

SELECT * FROM t_uuid
```

### Arrays/Maps/元组(Tuple)/Set

数组类型(类似PostgreSQL)：

```sql
SELECT array(1, 2, NULL) AS x, toTypeName(x)
```

Map类型：

```sql
CREATE TABLE table_map (a Map(String, UInt64)) ENGINE=Memory;
INSERT INTO table_map VALUES ({'key1':1, 'key2':10}), ({'key1':2,'key2':20}), ({'key1':3,'key2':30});
```

Tuple:

```sql
SELECT tuple(1, 'a') AS x, toTypeName(x)
```

Set:

```sql
SELECT UserID IN (123, 456) FROM ...
SELECT (CounterID, UserID) IN ((34, 123), (101500, 456)) FROM ...
```

### IP地址

类似PostgreSQL:

```sql
CREATE TABLE hits (url String, from IPv6) ENGINE = MergeTree() ORDER BY url;

┌─url────────────────────────────────┬─from──────────────────────────┐
│ https://clickhouse.com          │ 2001:44c8:129:2632:33:0:252:2 │
│ https://clickhouse.com/docs/en/ │ 2a02:e980:1e::1               │
│ https://wikipedia.org              │ 2a02:aa08:e000:3100::2        │
└────────────────────────────────────┴───────────────────────────────┘
```

### 嵌套数据型

类似于PostgreSQL 的复合类型

```sql
CREATE TABLE test.visits
(
    CounterID UInt32,
    StartDate Date,
    Sign Int8,
    IsNew UInt8,
    VisitID UInt64,
    UserID UInt64,
    ...
    Goals Nested
    (
        ID UInt32,
        Serial UInt32,
        EventTime DateTime,
        Price Int64,
        OrderID String,
        CurrencyID UInt32
    ),
    ...
) ENGINE = CollapsingMergeTree(StartDate, intHash32(UserID), (CounterID, StartDate, intHash32(UserID), VisitID), 8192, Sign)
```

### 聚合数据型

```sql
CREATE TABLE simple (id UInt64, val SimpleAggregateFunction(sum, Double)) ENGINE=AggregatingMergeTree ORDER BY id;

CREATE TABLE t
(
    column1 AggregateFunction(uniq, UInt64),
    column2 AggregateFunction(anyIf, String, UInt8),
    column3 AggregateFunction(quantiles(0.5, 0.9), UInt64)
) ENGINE = ...
```

### 其他特殊的数据结构

Expression: lambda函数

Nothing: `SELECT toTypeName(array())`

Interval：`SELECT toTypeName(INTERVAL 4 DAY)`

[完整官方示例见](https://clickhouse.com/docs/en/sql-reference/data-types)

## 基础函数

### 算术函数

算术运算主要日常进行一些数值统计，通过SQL的方式就得出结果

- a+b / plus(a,b)
- a-b / minus(a,b)
- a*b / multiply(a,b)
- a/b / divide(a,b)
- intDiv(a, b)
- intDivOrZero(a, b)
- a % b / modulo(a, b)
- moduloOrZero(a, b)
- positiveModulo(a, b) / pmod(a, b) / positive_modulo(a, b)
- -a / negate(a)
- abs(a)
- gcd(a, b)
- lcm(a, b)
  
[更多运算内容见](https://clickhouse.com/docs/en/sql-reference/functions/arithmetic-functions)

### 时间日期函数

时间日期函数在统计和报表应用中会被频繁使用，常见的就是转日期、修改样式、计算间隔、计算星、计算第几天等

- makeDate(year, month, day);makeDate(year, day_of_year); 产生一个Date
- makeDateTime(year, month, day, hour, minute, second[, timezone])，产生一个DateTime
- timestamp(expr[, expr_time]), string转为timestamp， `SELECT timestamp('2023-12-31') as ts;`
- toYear(value), toQuarter(value), toMonth(value), toDayOfYear(value), toDayOfMonth(value), toDayOfWeek(t[, mode[, timezone]]) ,toHour(value), toMinute(value),toSecond(value), toMillisecond(value) 简单获悉一个Date/DateTime的年、月、日、时、分、秒、毫秒、星期等
- toStartOfYear(value)，toStartOfQuarter(value)，toStartOfMonth(value)，toLastDayOfMonth(value)，toMonday(value)，toStartOfWeek(t[, mode[, timezone]])，toLastDayOfWeek(t[, mode[, timezone]])，toStartOfDay(value)，toStartOfHour(value)，toStartOfMinute(value)，toStartOfSecond(value, [timezone])，toStartOfFiveMinutes(value)，toStartOfTenMinutes(value)，toStartOfFifteenMinutes(value)，toStartOfInterval(date_or_date_with_time, INTERVAL x unit [, time_zone])，简单获取基于某一个时间的开始或结束时间，本月月初、本星期初、本年初、本月末、10分钟前、15分钟前、基于某一时间的任意前后一段间隔时间

时间函数部分还是比较常用且重要的，可以学习并理解[所有函数](https://clickhouse.com/docs/en/sql-reference/functions/date-time-functions)

### 数组函数

数组函数主要是日常对一个数组类型的数据进行初始化、取值等

- range([start, ] end [, step]), 根据范围截取： `SELECT range(5), range(1, 5), range(1, 5, 2), range(-1, 5, 2);`
- arrayConcat(arrays), 将数组连接：`SELECT arrayConcat([1, 2], [3, 4], [5, 6]) AS res`
- arrayElement(arr, n), operator arr[n]
- has(arr, elem)，确认某个数组是否有某个元素
- hasSubstr，确认某个数组是否有子串
- indexOf(arr, x)，确认某个元素的index: `SELECT indexOf([1, 3, NULL, NULL], NULL)`
- arrayJoin: [不寻常的函数，参看官方使用方法](https://clickhouse.com/docs/en/sql-reference/functions/array-join)

数组上的运算方式有很多、很强大，[建议查阅官方文档](https://clickhouse.com/docs/en/sql-reference/functions/array-functions#indexofarr-x)

### 比较函数

- a = b (operator)，a == b (operator)，equals(a, b)
- a != b (operator)，a <> b (operator)，notEquals(a, b)
- a < b (operator)，less(a, b)
- a > b (operator)，greater(a, b)
- a <= b (operator)，lessOrEquals(a, b)
- a >= b (operator)，greaterOrEquals(a, b)

### JSON函数

JSON函数通常用于存储非固定结构的数据后，需要获取部分字段进行查询、统计等

- simpleJSONHas(json, field_name) 判断是否存在
- simpleJSONExtractUInt(json, field_name)，simpleJSONExtractInt(json, field_name)，simpleJSONExtractFloat(json, field_name)，simpleJSONExtractBool(json, field_name)，simpleJSONExtractRaw(json, field_name)，simpleJSONExtractString(json, field_name)，将json中字段提出出来到指定的类型

### 字符串操作类函数

字符串的操作也是我们日常使用中经常碰到的

#### 基础操作

- empty(x)
- notEmpty(x)
- length
- leftPad(string, length[, pad_string]),rightPad(string, length[, pad_string]) 填充 `SELECT leftPad('abc', 7, '*'), leftPad('def', 7);`
- lower,upper,lowerUTF8,upperUTF8
- isValidUTF8,toValidUTF8
- repeat(s, n)
- space(n)
- reverse
- concat(s1, s2, ...),concatWithSeparator(sep, expr1, expr2, expr3...)
- substring(s, offset[, length]),substr,mid,byteSlice
- substringIndex(s, delim, count): `SELECT substringIndex('www.clickhouse.com', '.', 2)` 拆分后取子串
- base58Encode(plaintext)，base58Encode(plaintext)，base64Encode，base64Decode
- endsWith，startsWith
- trim，trimLeft，trimRight，trimBoth
- ascii(s)

#### 替换

- replaceOne(haystack, pattern, replacement)
- replaceAll(haystack, pattern, replacement)
- replaceRegexpOne(haystack, pattern, replacement)
- translate(s, from, to) 转换

#### 查询定位

- position(haystack, needle[, start_pos])
- positionCaseInsensitive
- multiSearchAllPositions(haystack, [needle1, needle2, ..., needleN])
- match(haystack, pattern)
- extract(haystack, pattern)
- like(haystack, pattern)
- notLike
- ilike : insensitive like
- notILike
- countMatches(haystack, pattern)
- regexpExtract(haystack, pattern[, index])

#### 分解

- splitByChar(separator, s[, max_substrings]))
- splitByString(separator, s[, max_substrings]))
- splitByRegexp(regexp, s[, max_substrings]))
- splitByWhitespace(s[, max_substrings]))
- arrayStringConcat(arr\[, separator\])
- extractAllGroups(text, regexp)
- ngrams(string, ngramsize)

### UDF 用户自定义函数

使用XML声明一个函数，放置在 `/etc/clickhouse-server`下，比如 `/etc/clickhouse-server/test_function.xml`

```xml
<functions>
    <function>
        <type>executable</type>
        <name>test_function_python</name>
        <return_type>String</return_type>
        <argument>
            <type>UInt64</type>
            <name>value</name>
        </argument>
        <format>TabSeparated</format>
        <command>test_function.py</command>
    </function>
</functions>
```

书写自定义脚本，放置在 `/var/lib/clickhouse/user_script`下，比如：`/var/lib/clickhouse/user_scripts/test_function.py`

```py
#!/usr/bin/python3

import sys

if __name__ == '__main__':
    for line in sys.stdin:
        print("Value " + line, end='')
        sys.stdout.flush()
```

在查询中使用：

```sql
SELECT test_function_python(toUInt64(2));
```

[具体自定义函数的字段，参看官方](https://clickhouse.com/docs/en/sql-reference/functions/udf)

### 更多函数

- Bit
- BitMap
- Conditional
- Dictionaries
- Distance
- Geo
- Encoding
- Encryption
- Files
- Hash
- Ip Address
- Logical
- Machine Learning
- Maps
- Mathmatical
- NLP
- Random Numbers
- Rounding
- Time Series
- Time Window
- Tuples
- ULID
- URLs
- UUIDs
- Aggregate Functions
- Table Functions
- Window Functions

[具体参看官方使用指南](https://clickhouse.com/docs/en/sql-reference/functions/)

----
如果喜欢我的文章的话，可以去[GitHub上给一个免费的关注](https://github.com/CzyerChen/)吗？
