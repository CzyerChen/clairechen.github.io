---
layout:     post
title:      Mybatis核心处理层-Executor
subtitle:   装饰器/模板方法
date:       2020-08-08
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
    - 装饰器模式
    - 模板方法
---

## Mybatis核心处理层-Executor

- 前面介绍了一些StatementHandler,用于最终执行query/insert/update/delete/batch/queryCursor操作
- 本篇介绍它的上层Executor

```text
                 Executor
                    |
      |--------------------------------|
  BaseExecutor （模板方法）      CachingExecutor（装饰器，二级缓存）
      |
    子类 
  SimpleExecutor
  ReuseExecutor
  CloseExecutor
  BatchExecutor  
```

### 1. 模板方法

- 一连串操作的顺序由“模板方法”维护，多个实现可能内容并不相同，那些真正实现的方法称为“基础方法”
- 模板方法，可以使不变的逻辑维护在父类中，具体实现维护在子类中，他们的方法调用逻辑相同
- 实现了定义与实现的解耦，很好的体现了“开放-封闭”原则，充分利用了面向对象的“多态性”

### 2. BaseExecutor

- 维护一级缓存的实现/事务的实现
- 给出doUpdate、doQuery、doQueryCursor、doFlushStatements四个抽象方法给子类实现
- 本地缓存使用PerpetualCache实现，使用的本地HashMap
- 延迟加载一级缓存使用ConcurrentLinkedQueue 实现

#### (1)一级缓存

- 会话级别的缓存，能够针对短时间内重复查询的SQL能够起到缓存的作用，直接获取缓存数据，而不请求数据库。从而减小数据库压力
- 一级缓存生命周期与session一致，因而session关闭一级缓存就会被清除
- 一级缓存是默认开启的
- 一个查询，会根据生成的CacheKey是否在缓存中命中，如果命中直接返回缓存，如果不命中则会查询数据，获取结果后存入缓存中

- CacheKey标识一个查询

```java
 @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    //offset
    cacheKey.update(rowBounds.getOffset());
    //limit
    cacheKey.update(rowBounds.getLimit());
    //sql
    cacheKey.update(boundSql.getSql());
    //获取具体参数
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        //把查询内容存储
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      //环境id
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```

- 查询方法进行一级缓存的使用

```java
 public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      //不嵌套，并且需要清空缓存
      clearLocalCache();
    }
    List<E> list;
    try {
      //如有嵌套，层级递增
      queryStack++;
      //获取缓存
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        //存在缓存
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //不存在缓存，查询结果赋值给list
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      //已无嵌套查询
      //遍历延迟加载对象
      for (DeferredLoad deferredLoad : deferredLoads) {
        //触发加载一级缓存中对应的值
        deferredLoad.load();
      }
      // issue #601
      //清除队列
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        //根据LocalCacheScope配置，看是否需要清空缓存
        clearLocalCache();
      }
    }
    return list;
  }

  //handleLocallyCachedOutputParameters
   private void handleLocallyCachedOutputParameters(MappedStatement ms, CacheKey key, Object parameter, BoundSql boundSql) {
    if (ms.getStatementType() == StatementType.CALLABLE) {
      final Object cachedParameter = localOutputParameterCache.getObject(key);
      if (cachedParameter != null && parameter != null) {
        final MetaObject metaCachedParameter = configuration.newMetaObject(cachedParameter);
        final MetaObject metaParameter = configuration.newMetaObject(parameter);
        for (ParameterMapping parameterMapping : boundSql.getParameterMappings()) {
          if (parameterMapping.getMode() != ParameterMode.IN) {
            final String parameterName = parameterMapping.getProperty();
            //获取缓存的值
            final Object cachedValue = metaCachedParameter.getValue(parameterName);
            metaParameter.setValue(parameterName, cachedValue);
          }
        }
      }
    }
  }

  //queryFromDatabase
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    //现在缓存中放入占位符
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      //查询结果
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      //移除占位符
      localCache.removeObject(key);
    }
    //放入确切值
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }

```

- DeferredLoad

```java

    //判断是否在一级缓存中
    public boolean canLoad() {
      //检测缓存中是否存在，或者是否正在加载阶段（即占位符占位着）
      return localCache.getObject(key) != null && localCache.getObject(key) != EXECUTION_PLACEHOLDER;
    }

    //加载参数
    public void load() {
      @SuppressWarnings("unchecked")
      // we suppose we get back a List
        //缓存中获取对象值
      List<Object> list = (List<Object>) localCache.getObject(key);
      Object value = resultExtractor.extractObjectFromList(list, targetType);
      resultObject.setValue(property, value);
    }

```

- 以上发现flushCache 参数，LocalCacheScope配置 都可以影响是否删除缓存，除此以外，在insert update delete 对应的doUpdate之前都需要清除缓存

#### （2）事务操作

- commit & rollback

```java

  @Override
  public void commit(boolean required) throws SQLException {
    if (closed) {
      //连接关闭了
      throw new ExecutorException("Cannot commit, transaction is already closed");
    }
    //清除一级缓存
    clearLocalCache();
    //批处理多条SQL，不回滚
    flushStatements();
    if (required) {
      //如果需要提交，则提交
      transaction.commit();
    }
  }

  @Override
  public void rollback(boolean required) throws SQLException {
    if (!closed) {
      //连接未关闭
      try {
        //清除一级缓存
        clearLocalCache();
        //批处理SQL，需要混滚
        flushStatements(true);
      } finally {
        if (required) {
          //如果需要回滚
          transaction.rollback();
        }
      }
    }
  }
```

### 3. SimpleExecutor

- 主要实现4个方法：doQuery doQueryCursor doFlushStatements doUpdate
- 都是依靠Configuration  获取Connection 获取PrepareStatement  生成StatementHandler 最终依靠handler来实现以上4个方法

```java
 @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```

### 4.ReuseExecutor

- private final Map<String, Statement> statementMap = new HashMap<>();
- 依靠Map实现Statement的重用
- 方法的实现与SimpleExecutor相同
- prepareStatement阶段，会判断缓存是否有建立的相同模式下的SQL语句对应的Statement对象，实现重用
- 会在doFlushStatement的时候，将Map清空

```java
  @Override
  public List<BatchResult> doFlushStatements(boolean isRollback) {
    for (Statement stmt : statementMap.values()) {
      closeStatement(stmt);
    }
    statementMap.clear();
    return Collections.emptyList();
  }

```

### 5.BatchExecutor

- 对于频繁单条SQL发往数据库，使得数据库负载过高的情况，客户端通常会将批量的SQL积累后，在适当的时机，发往数据库同时执行（有上限），以减轻数据库与网络的压力

- JDBC中批处理仅支持insert\update\delete 不支持select
- 因而核心代码都聚集在doUpdate的方法中

```java
  @Override
  public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
      //SQL与最近一次的查询情况相同，直接添加到同一个对象中进行等待
      int last = statementList.size() - 1;
      stmt = statementList.get(last);
      applyTransactionTimeout(stmt);
      handler.parameterize(stmt);// fix Issues 322 处理?占位符
      BatchResult batchResult = batchResultList.get(last);
      batchResult.addParameterObject(parameterObject);
    } else {
      //如果不相同，则打开一个新的连接，创建新的Statement,并添加到集合中进行等待
      Connection connection = getConnection(ms.getStatementLog());
      stmt = handler.prepare(connection, transaction.getTimeout());
      handler.parameterize(stmt);    // fix Issues 322  处理?占位符
      currentSql = sql;
      currentStatement = ms;
      statementList.add(stmt);
      batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    handler.batch(stmt);
    return BATCH_UPDATE_RETURN_VALUE;
  }
```

- 执行批处理的关键方法：doFlushStatements

```java
@Override
  public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
      List<BatchResult> results = new ArrayList<>();
      if (isRollback) {
        //需要回滚
        return Collections.emptyList();
      }
      for (int i = 0, n = statementList.size(); i < n; i++) {
        //依次取出每一个需要执行的Statement
        Statement stmt = statementList.get(i);
        applyTransactionTimeout(stmt);
        BatchResult batchResult = batchResultList.get(i);
        try {
          //stmt.executeBatch()核心批处理方法
          //updateCount 是拿stmt.executeBatch() 返回值--影响的行数，进行设置
          batchResult.setUpdateCounts(stmt.executeBatch());
          MappedStatement ms = batchResult.getMappedStatement();
          List<Object> parameterObjects = batchResult.getParameterObjects();
          //处理主键
          KeyGenerator keyGenerator = ms.getKeyGenerator();
          if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
            Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
            jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
          } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { //issue #141
            for (Object parameter : parameterObjects) {
              keyGenerator.processAfter(this, ms, stmt, parameter);
            }
          }
          // Close statement to close cursor #1109
          closeStatement(stmt);
        } catch (BatchUpdateException e) {
          StringBuilder message = new StringBuilder();
          message.append(batchResult.getMappedStatement().getId())
              .append(" (batch index #")
              .append(i + 1)
              .append(")")
              .append(" failed.");
          if (i > 0) {
            message.append(" ")
                .append(i)
                .append(" prior sub executor(s) completed successfully, but will be rolled back.");
          }
          throw new BatchExecutorException(message.toString(), e, results, batchResult);
        }
        results.add(batchResult);
      }
      return results;
    } finally {
      for (Statement stmt : statementList) {
        closeStatement(stmt);
      }
      currentSql = null;
      statementList.clear();
      batchResultList.clear();
    }
  }
```

### 6. CachingExecutor

- 不同于上面的模板方法，CachingExecutor是采用了装饰的方法，为mybatis提供二级缓存

#### 什么是二级缓存？

1. mybatis-config中 cahceEnabled是开始二级缓存的总开关

- 为true时生效，默认为true

2. 配置cache / cacheRef 节点就表明开启二级缓存

- 默认实现是PerpetualCache
- 通过cache / cacheRef的配置，用户可以实现在命名空间粒度上管理二级缓存

3.<select>节点中 配置useCache属性

- 标识该操作的结果是否需要保存到二级缓存中
