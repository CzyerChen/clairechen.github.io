---
layout:     post
title:      RESTful API设计规范-RESTful
subtitle:   RESTful
date:       2020-08-10
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - RESTful API
    - API 设计
---

# RESTful API 规范

## 一、什么是RESTful

- REST 取自Roy Thomas Fielding 2000年的博士论文
- Roy Thomas Fielding 是HTTP1.0/1.1主要设计者，Apache服务器软件的作者之一，Apache基金会第一任主席
- 符合架构原理的前提下，理解和评估以网络为基础的应用软件的架构设计，得到一个功能强、性能好、适宜通信的架构
- REST -- Representational Stete Transfer，如果一个架构符合REST 原则，则称其为RESTful架构
- 网站即软件
- 理查德森模型将REST架构的成熟度分为三个等级

```text
level 1 Resources
level 2 Http verbs
level 3 Hybermedia Controls
```

### 1.资源 Resources

- 网络上的资源：文本、图片、服务等

### 2.表现层 Representation

- "资源"具体展现出来的形式，叫做“表现层”

### 3.状态转换 State Transfer

- 一个请求，肯定会涉及到数据和状态的变化
- HTTP通信是无状态的，那么所有状态就保留在服务端
- 客户端如果想操作服务端，就必须通过渠道，让服务端进行“状态转换”
- 四个动词：GET /POST /PUT /DELETE ,能够让服务端进行"状态转换"

### 4.什么是RESTful架构

- 每一个URI代表一种资源
- 客户端与服务端质检，传递这种资源的某种表现层
- 客户端通过四个HTTP动词，对服务端资源进行操作，实现“表现层状态转化”

### 5.误区

- URI包含动词，如果及时有表达不了的动词，也只能用名词来替代，描述一种服务，而不是使用动词

### 其他总结

```text
REST四个基本原则：
1.使用HTTP动词：GET POST PUT DELETE；
2.无状态连接，服务器端不应保存过多上下文状态，即每个请求都是独立的；
3.为每个资源设置URI；
4.通过XML JSON进行数据传递；
实现上述原则的架构即可称为RESTFul架构。
1.互联网环境下，任何应用的架构和API可以被快速理解；
2.分布式环境下，任何请求都可以被发送到任意服务器；
3.异构环境下，任何资源的访问和使用方式都统一；
```

## 二、RESTful API指南

### 1.使用HTTPs协议

### 2.将API部署在专用域名之下，使用统一的url前缀，例如/api

### 3.将API版本放入URL，例如/api/v1

### 4.EndPoint 路径

标识API具体网址，网址中不能有动词，只有名词，并且名词最好能够与数据库表的名称对应，API中名词应该是复数

### 5.HTTP动词

```text
GET: select 操作，从服务器获取资源
POST：create操作，在服务器新建一个资源
PUT：update操作，在服务器更新资源，客户端提供改变后的完整资源
PATCH：update 操作，在服务器更新资源，客户端提供改变的数据
DELETE：delete操作，从服务器删除资源

HEAD：获取资源的元信息
OPTIONS：获取信息
```

### 6.过滤信息

参数的设计允许冗余，允许API路径和URL参数偶尔重复

### 7.状态码

```text
200 OK - [GET]：服务器成功返回用户请求的数据，该操作是幂等的（Idempotent）。
201 CREATED - [POST/PUT/PATCH]：用户新建或修改数据成功。
202 Accepted - [*]：表示一个请求已经进入后台排队（异步任务）
204 NO CONTENT - [DELETE]：用户删除数据成功。
400 INVALID REQUEST - [POST/PUT/PATCH]：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的。
401 Unauthorized - [*]：表示用户没有权限（令牌、用户名、密码错误）。
403 Forbidden - [*] 表示用户得到授权（与401错误相对），但是访问是被禁止的。
404 NOT FOUND - [*]：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的。
406 Not Acceptable - [GET]：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
410 Gone -[GET]：用户请求的资源被永久删除，且不会再得到的。
422 Unprocesable entity - [POST/PUT/PATCH] 当创建一个对象时，发生一个验证错误。
500 INTERNAL SERVER ERROR - [*]：服务器发生错误，用户将无法判断发出的请求是否成功
```

[全量HTTP状态码](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)

### 8.错误处理

- 如果是4XX的错误，应该向用户返回出错信息

### 9.返回结果

```text
GET /collection：返回资源对象的列表（数组）
GET /collection/resource：返回单个资源对象
POST /collection：返回新生成的资源对象
PUT /collection/resource：返回完整的资源对象
PATCH /collection/resource：返回完整的资源对象
DELETE /collection/resource：返回一个空文档
```

### 10.Hypermedia API

返回结果中，提供连接，连向其他的API方法

### 11.返回数据格式

最好使用JSON/XML,增加可读性


## 三、RESTFUL 最佳实践

### URL设计

1. 动词+宾语： GET /api/files ; POST /api/files ; DELETE /api/files/{id}
2. 宾语是名词复数形式 
3. 避免多级URL ： GET /api/libraries/{lid}/books/{bid} ---> GET /api/libararies/books/{bid}?lid=
4. HTTP状态码

```text
1xx：相关信息
2xx：操作成功
3xx：重定向
4xx：客户端错误
5xx：服务器错误

GET: 200 OK
POST: 201 Created
PUT: 200 OK
PATCH: 200 OK
DELETE: 204 No Content


4xx状态码表示客户端错误，主要有下面几种。
400 Bad Request：服务器不理解客户端的请求，未做任何处理。
401 Unauthorized：用户未提供身份验证凭据，或者没有通过身份验证。
403 Forbidden：用户通过了身份验证，但是不具有访问资源所需的权限。
404 Not Found：所请求的资源不存在，或不可用。
405 Method Not Allowed：用户已经通过身份验证，但是所用的 HTTP 方法不在他的权限之内。
410 Gone：所请求的资源已从这个地址转移，不再可用。
415 Unsupported Media Type：客户端要求的返回格式不支持。比如，API 只能返回 JSON 格式，但是客户端要求返回 XML 格式。
422 Unprocessable Entity ：客户端上传的附件无法处理，导致请求失败。
429 Too Many Requests：客户端的请求次数超过限额

5xx状态码表示服务端错误。一般来说，API 不会向用户透露服务器的详细信息，所以只要两个状态码就够了。
500 Internal Server Error：客户端请求有效，服务器处理时发生了意外。
503 Service Unavailable：服务器无法处理请求，一般用于网站维护状态。

```

5. 服务器响应JSON,而不是纯文本
6. 发生错误时，不要返回 200 状态码，给出错误信息

## 四、总结

- 通常在实操当中我们需要注意

1. API是否添加版本标识？version?
2. API是否统一前缀?api prefix？
3. API URL 设计是否符合动词+宾语 形式 （通常最直观的不符合RESTful的就是这一点）
4. API是否给出合理的响应？通用的JSON形式，规范化的response code?