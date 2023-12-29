---
layout:     post
title:      JSONPlaceholder 使用手册
subtitle:   JSONPlaceholder
date:       2023-12-26
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - JSONPlaceholder
    - mock
---

[官方使用指南快速索引>>点这里](https://jsonplaceholder.typicode.com/guide/)
快速导览：

- [什么是JSONPlaceholder?有啥用?](#什么是jsonplaceholder有啥用)
- [如何使用JSONPlaceholder?](#如何使用jsonplaceholder)
  - [关于“增”](#关于增)
  - [关于“改”](#关于改)
  - [关于“查”](#关于查)
  - [关于“删”](#关于删)
  - [关于“分页查”](#关于分页查)
  - [关于“根据ID查多个”](#关于根据id查多个)
- [尝试自己搭一个？](#尝试自己搭一个)
- [扩展的可能？](#扩展的可能)


## 什么是JSONPlaceholder?有啥用?

这是一个开放可调用的API接口，增、改、查、分页查一个简单资源

什么时候需要？

- 当你需要测试一种API调用的方式
- 当你想模拟一些API json数据
- 当你是一个妥妥的新手

> 你可能需要它

## 如何使用JSONPlaceholder?

可以使用：

- 浏览器直接访问Get请求，比如
  
```bash
/posts/1/comments
/albums/1/photos
/users/1/albums
/users/1/todos
/users/1/posts
```

- 可以使用编程语言请求接口

通用接口：

|接口|对象类型|结构|公共默认数据量|
|--|--|--|--|
|/posts|帖子|```{"userId":1,"id":1,"title":"string","body":"string"}```|100个
|/comments|评论|```{"postId":1,"id":1,"name":"string","email":"string","body":"string"}```|500个|
|/albums|专辑|```{"userId":1,"id":1,"title":"string"}```|100个|
|/photos|照片|```{"albumId":1,"id":1,"title":"string","url":"string","thumbnailUrl":"string"}```|5000个|
|/todos|待办|```{"userId":1,"id":1,"title":"string","completed":false}```|200个|
|/users|用户|```{"id":1,"name":"string","username":"string","email":"string","address":{"street":"string","suite":"string","city":"string","zipcode":"string","geo":{"lat":"string","lng":"string"}},"phone":"string","website":"string","company":{"name":"string","catchPhrase":"string","bs":"string"}}```|10个|

自己测试的话，以上数据结构各取所需即可

以上几种数据类型也有关联关系，比如：

```bash
/comments?postId=1 == /posts/1/comments
/albums/1/photos
/users/1/albums
/users/1/todos
/users/1/posts
```

以下使用`/todos` 为例演示增删改查

```bash
GET	/posts
GET	/posts/1
GET	/posts/1/comments
GET	/comments?postId=1
POST /posts
PUT	/posts/1
PATCH	/posts/1
DELETE	/posts/1
```

### 关于“增”

`POST /posts`

```bash
POST /posts HTTP/1.1
Host: jsonplaceholder.typicode.com
Content-Type: application/json
Content-Length: 75

{
    "userId": 2,
    "id": 1,
    "title": "帖子101",
    "body": "帖子的内容"
}
----
response:
{
    "userId": 2,
    "id": 101,
    "title": "帖子101",
    "body": "帖子的内容"
}
```

ID会自动生成，类似数据库里面的自增，并且会返回增加的对象

### 关于“改”

`PATCH	/posts/1`

```bash
PATCH /posts/101 HTTP/1.1
Host: jsonplaceholder.typicode.com
Content-Type: application/json
Content-Length: 76

{
    "userId": 2,
    "id": 1,
    "title": "帖子102",
    "body": "帖子的内容2"
}
----
response:
{
    "userId": 2,
    "id": 1,
    "title": "帖子102",
    "body": "帖子的内容2"
}
```

根据ID修改内容，会返回修改后的内容

### 关于“查”

`GET /posts` 查全部，默认是100条
`GET /posts/1` 根据ID查询对象

```bash
GET /posts/1 HTTP/1.1
Host: jsonplaceholder.typicode.com
X-Total-Count: 10
```

### 关于“删”

`DELETE	/posts/1` 根据ID删除对象

```bash
DELETE /posts/100 HTTP/1.1
Host: jsonplaceholder.typicode.com
```

http response code 200，无返回值

### 关于“分页查”

`GET /posts?_start=10&_limit=10` 使用query param分页查

```bash
GET /posts?_start=10&_limit=10 HTTP/1.1
Host: jsonplaceholder.typicode.com
```

There's also a custom header X-Total-Count in the response.

### 关于“根据ID查多个”

`/posts?id=2&id=5&id=10&id=12` 根据多个ID查询相应的posts
`/posts?&userId=2&userId=5&userId=10&userId=12` 根据多个userId查询相应的posts

```bash
GET /posts?userId=2&userId=5&userId=10&userId=12 HTTP/1.1
Host: jsonplaceholder.typicode.com
X-Total-Count: 10
```

## 尝试自己搭一个？

[fork me here, start quickly](https://github.com/CzyerChen/jsonplaceholderdemo)

rename the request url `https://my-json-server.typicode.com/[your name]/[your project name]`, like `https://my-json-server.typicode.com/CzyerChen/jsonplaceholderdemo`

like default `posts`, you can call `GET https://my-json-server.typicode.com/CzyerChen/jsonplaceholderdemo/posts`, and you will get results.

Awesome! you can mock data and write into `db.json`.

like the demo in `https://github.com/CzyerChen/jsonplaceholderdemo`, call `https://my-json-server.typicode.com/CzyerChen/jsonplaceholderdemo/userinfo` and return the data

```json
[
  {
    "id": 1,
    "userName": "string",
    "age": 11,
    "mail": "string@demo.com",
    "sex": 0,
    "active": 1
  },
  {
    "id": 2,
    "userName": "string",
    "age": 12,
    "mail": "string@demo.com",
    "sex": 1,
    "active": 0
  },
  {
    "id": 3,
    "userName": "string",
    "age": 13,
    "mail": "string@demo.com",
    "sex": 2,
    "active": 1
  }
]
```

## 扩展的可能？

增加简单的认证接口？比如jwt/oauth2?

`issues` 中也有提到，是否能够提供初级使用者所需的认证类接口，或许可以自己根据源码来开发哦