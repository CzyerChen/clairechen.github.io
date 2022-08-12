---
layout:     post
title:      MySQL-为什么不用UUID做主键
subtitle:   MySQL
date:       2020-11-06
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - SQL
---

## 为什么不用UUID做主键

- 分布式数据库一个偷懒的ID的实现方式，就是使用UUID
- 除此以外，没有什么必然理由使用UUID做主键

- 占用存储，会比自增有更多填充因子，无序的基本占用空间基本扩大一半
- 无序，每次插入需要造成脏页，需要消耗CPU去刷新页的数据
- 查询的时候也不能很好的定位主键


https://segmentfault.com/a/1190000025122226

https://v2ex.com/t/633171