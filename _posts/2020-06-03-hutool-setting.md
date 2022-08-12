---
layout:     post
title:      Hutool的使用
subtitle:   Hutool工具类
date:       2017-12-19
author:     BY
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - hutool
    - 工具类
---

> 看到hutool有很多好用的工具类，希望从中学习到更多对于框架和基础的理解，决定对每一个模块进行了解，并与自己日常百度到的版本进行比对

## 目录

- Copier
- SimpleCache

### 一、Copier

使用函数式接口的实现，不同实现类可以实现各自的拷贝方法

#### 1. BeanCopier

#### 2.FileCopier

### 二、SimpleCache

使用WeakHashMap实现简单缓存，比较反射带来的性能损耗

#### 1.StampedLock vs ReentrantLock vs ReentrantReadWriteLock vs synchronized

BeanUtil
BeanInfoCache

ReflectUtil