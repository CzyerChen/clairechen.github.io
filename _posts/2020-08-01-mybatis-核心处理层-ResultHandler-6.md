---
layout:     post
title:      Mybatis核心处理层-ResultHandler参数映射核心逻辑-6
subtitle:   装饰器/策略
date:       2020-08-01
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
    - 装饰器模式
    - 策略模式 
---

## ResultHandler 参数映射核心逻辑-5

### 一、StatementHandler

```text
                    StatementHandler
                          |
          |-------------------------------|
   BaseStatementHandler        RoutingStatementHandler(装饰器/策略：针对创建什么类型StatementHandler，做了类型判断)
         |
        子类
 PrepareStatementHandler
 SimpleStatementHandler
 CallableStatementHandler
```

#### 1. RoutingStatementHandler

- 此处的RoutingStatementHandler 并没有特殊的实现，只是根据StatementType字段，创建不同的StatementHandler，主要功能实现只要在于SimpleStatementHandler、PreparedStatementHandler、CallableStatementHandler 三个对象

```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

  }
```

#### 2. BaseStatementHandler

- 主要实现prepare方法

```java
 @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      //初始化 Statement 对象
      statement = instantiateStatement(connection);
      //设置超时时间
      setStatementTimeout(statement, transactionTimeout);
      //设置FetchSize
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }
```

#### 3.ParameterHandler

- 实现类：DefaultParameterHandler
- 主要针对SQL中占位符?，替换实际参数内容

```java
 @Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      //遍历mapping
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          //获取参数名
          String propertyName = parameterMapping.getProperty();
          //获取参数值
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            //设置参数值
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException | SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
```

#### 4.SimpleStatementHandler

- 继承自BaseStatementHandler，主要用于实现query / update等方法

```java

  //初始化，创建创建用于操作的Statement对象
  @Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
      return connection.createStatement();
    } else {
      return connection.createStatement(mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    }
  }

  // query queryCursor batch方法均直接调用Statement实现，故不介绍

   //执行insert update delete 方法
  @Override
  public int update(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    Object parameterObject = boundSql.getParameterObject();
    //获取主键类型
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    int rows;
    if (keyGenerator instanceof Jdbc3KeyGenerator) {
      //  Jdbc3KeyGenerator 类型
      //  执行SQL
      statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
      //获取影响行数
      rows = statement.getUpdateCount();
      //将从数据获取的主键数据放回参数列表中
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else if (keyGenerator instanceof SelectKeyGenerator) {
      statement.execute(sql);
      rows = statement.getUpdateCount();
      //根据<selectKey>中配置的SQL语句获取主键ID，添加到参数列表中
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else {
      statement.execute(sql);
      rows = statement.getUpdateCount();
    }
    return rows;
  }

```

#### 5. PrepareStatementHandler

```java
//初始化略有不同
@Override
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    //主键生成策略是Jdbc3KeyGenerator
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        //返回数据库生成的主键
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        //insert结束后，返回指定主键列的内容
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
      //普通类型查询
      return connection.prepareStatement(sql);
    } else {
      //cursor
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    }
  }
```

#### 6.CallableStatementHandler

- 内部使用CallableStatement，调用指定存储过程， query、update方法与上一致

