---
layout:     post
title:      PostgreSQL处理JSON数据
subtitle:   PostgreSQL
date:       2023-05-31
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - PostgreSQL
    - JSON
---


由于项目内使用的Postgresql 且存储了一些非结构化的json数据，里面含有统计与记录，并且有嵌套关系，所以需要了解如何查询和处理Postgresql中的JSON数据

- Postgresql：9.6
- 官方文档：http://postgres.cn/docs/9.6/functions-json.html#FUNCTIONS-JSON-CREATION-TABLE
- 一张基础订单表结构：

```sql
-- order表
"id" bigserial primary key,
"order_id" varchar(55) COLLATE "default",
"product_id" int8,
"order_json" text COLLATE "default",
"create_time" timestamp(6),
```

## json和jsonb 操作符

|操作符	|右操作数类型	|描述|	例子	|例子结果
|--|--|--|--|--|
|->	|int	|获得 JSON 数组元素（索引从 0 开始，负整数结束）|	'[{"a":"foo"},{"b":"bar"},{"c":"baz"}]'::json->2	|{"c":"baz"}|
|->	|text	|通过键获得 JSON 对象域|	'{"a": {"b":"foo"}}'::json->'a'|{"b":"foo"}
|->>|	int	|以文本形式获得 JSON 数组元素|	'[1,2,3]'::json->>2	|3
|->>|	text|	以文本形式获得 JSON 对象域|	'{"a":1,"b":2}'::json->>'b'	|2
|#>	|text[]	|获取在指定路径的 JSON 对象	|'{"a": {"b":{"c": "foo"}}}'::json#>'{a,b}'	|{"c": "foo"}
|#>>|	text[]	|以文本形式获取在指定路径的 JSON 对象|	'{"a":[1,2,3],"b":[4,5,6]}'::json#>>'{a,2}'	|3

## 如何查看JSON指定的key内容？

通过::json的语法

```sql
select order_json::json->'orderBody' from order -- 对象域
select order_json::json->>'orderBody' from order -- 文本
select order_json::json#>'{orderBody}' from order -- 对象域
select order_json::json#>>'{orderBody}' from order -- 文本
```

还有更多的jsonb操作符和json操作函数见官方文档

## 怎么处理多层嵌套的JSON？

就是采用基本的JSON语法，注意结果是对象域还是文本，对象域可以继续取用字段，文本就不能继续查看JSON咯

```sql
select '{"sites":{"site":{"id":"1","name":"菜鸟教程","url":"www.runoob.com"}}}'::json->'sites'->'site' -- 对象域
select '{"sites":{"site":{"id":"1","name":"菜鸟教程","url":"www.runoob.com"}}}'::json->'sites'->>'site' -- 文本
```

## 怎么处理JSON数组呢?

也是通过JSON的基本操作先定位到数组对象所在的Key,通过key取到对应的value后直接->(0),就可以取用到对应的对象域，注意对象域和文本，转化为文本就不能够在取key和具体数据数据咯

还有很多关于json相关的方法，可以详见官方文档

```sql
select '{"sites":{"site":[{"id":"1","name":"菜鸟教程","url":"www.runoob.com"},{"id":"2","name":"菜鸟工具","url":"c.runoob.com"},{"id":"3","name":"Google","url":"www.google.com"}]}}'::json->'sites'->'site'->(0)

select '{"sites":{"site":[{"id":"1","name":"菜鸟教程","url":"www.runoob.com"},{"id":"2","name":"菜鸟工具","url":"c.runoob.com"},{"id":"3","name":"Google","url":"www.google.com"}]}}'::json->'sites'->'site'->(0)->>'id'
```

延伸：如何取用JSON数组的最后一个对象数据？

```sql
select json_array_length('{"sites":{"site":[{"id":"1","name":"菜鸟教程","url":"www.runoob.com"},{"id":"2","name":"菜鸟工具","url":"c.runoob.com"},{"id":"3","name":"Google","url":"www.google.com"}]}}'::json->'sites'->'site') -- 查询json数据的长度

select '{"sites":{"site":[{"id":"1","name":"菜鸟教程","url":"www.runoob.com"},{"id":"2","name":"菜鸟工具","url":"c.runoob.com"},{"id":"3","name":"Google","url":"www.google.com"}]}}'::json->'sites'->'site'->(json_array_length('{"sites":{"site":[{"id":"1","name":"菜鸟教程","url":"www.runoob.com"},{"id":"2","name":"菜鸟工具","url":"c.runoob.com"},{"id":"3","name":"Google","url":"www.google.com"}]}}'::json->'sites'->'site') -1)
```

## 怎么替换JSON字符串中的内容？

通过select语句先看一下官方语法

|函数	|返回值	|描述	|例子	|例子结果|
|--|--|--|--|--|
|jsonb_set(target jsonb, path text[], new_value jsonb[, create_missing boolean])|jsonb|如果create_missing是真的 （缺省是true）并且通过path 指定部分不存在，那么返回target， 它具有path指定部分， new_value替换部分， 或者new_value添加部分。 正如路径导向的操作符，负整数出现在JSON数组结尾的path>计数中。	|(1)jsonb_set('[{"f1":1,"f2":null},2,null,3]', '{0,f1}','[2,3,4]', false)<br/>(2)jsonb_set('[{"f1":1,"f2":null},2]', '{0,f3}','[2,3,4]')|[{"f1":[2,3,4],"f2":null},2,null,3] <br/>[{"f1": 1, "f2": null, "f3": [2, 3, 4]}, 2]|

官方描述的挺明确：jsonb_set的方法

- 第一位参数，需要是jsonb的对象域
- 第二位参数，是访问对应value的path(注意这个path的语法可以是{a,b},表名key-a中的key-b,数据的话参看表格中的(2))
- 第三位参数，就是一个新的值，来替换第一个参数中的第二个参数key的value
- 第四个参数，如果create_missing是真的 （缺省是true）并且通过path 指定部分不存在，那么返回target， 它具有path指定部分， new_value替换部分， 或者new_value添加部分。 正如路径导向的操作符，负整数出现在JSON数组结尾的path>计数中

参照官方文档，简单的一次内容替换

```sql
select jsonb_set(order_json::jsonb,'{premsg}','test'::jsonb) from order 
```

那么如果是嵌套多层的JSON value可以替换吗？--可以的，语法是一样的，就是需要定位到指定的字段就可以

```sql
select jsonb_set((order_json::json->>'rspDesc')::jsonb, '{preOrder}', '"11111"'::jsonb) from order
```

上面是select语句，那具体的update语句怎么写呢？

```sql
语法：UPDATE 表明 set 列名 = (jsonb_set(列名::jsonb,'{key}','"value"'::jsonb)) where 条件 

update order set order_json = jsonb_set(order_json::jsonb,'{rspDesc}',(jsonb_set((event_json::json->>'rspDesc')::jsonb, '{preNumber}', '"999999999"'::jsonb)::jsonb)) -- 需要先把需要改的内容替换好，然后在整体更新替换，此时这个rspDesc是对象域格式
```

网上基本没有这样的例子，或者在我这执行是错误的，在此记录下。


