---
layout:     post
title:      Mybatis核心处理层-ResultHandler参数映射核心逻辑-5
subtitle:   主键生成
date:       2020-07-16
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
---

## ResultHandler 参数映射核心逻辑-5

### 一、KeyGenerator

- 处理insert语句的时候，并不会自动返回生成的主键内容，而只会返回插入记录的条数。如果插入记录是自增主键希望返回实际内容的时候，可以使用KeyGenerator接口

```java
public interface KeyGenerator {

  //insert SQL执行前执行
  void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter);
  //insert SQL执行后执行
  void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

}
```

- KeyGenerator接口有三个实现类：NoKeyGenerator、Jdbc3KeyGenerator/SelectKeyGenerator

- NoKeyGenerator:processBefore/processAfter两个方法不带任何实现

- Jdbc3KeyGenerator： 用于取回数据库生成的自增id，对应于mybatis-config.xml中的全局配置，SQL节点的useGeneratedKeys 属性
- processBefore无实现，processAfter实现自增主键id的获取

#### 1.Jdbc3KeyGenerator

- 用于取回数据库生成的自增id，对应mybatis-config.xml配置文件中的useGeneratedKeys 全局配置，以及映射映射配置文件中SQL节点的useGeneratedKeys属性
- processBefore方法为空，processAfter方法为实现主体,直接调用processBatch批量处理结果集

```java
public void processBatch(MappedStatement ms, Statement stmt, Object parameter) {
    // keyProperties 中存储主键名称和属性名映射
    final String[] keyProperties = ms.getKeyProperties();
    if (keyProperties == null || keyProperties.length == 0) {
      return;
    }
    //获取数据库生成主键
    try (ResultSet rs = stmt.getGeneratedKeys()) {
      //获取result元信息
      final ResultSetMetaData rsmd = rs.getMetaData();
      final Configuration configuration = ms.getConfiguration();
      //判断数据库主键列数与 keyProperties 中的是否一致
      if (rsmd.getColumnCount() < keyProperties.length) {
        // Error?
      } else {
        //将生成主键设置到用户传入参数的位置
        assignKeys(configuration, rs, rsmd, keyProperties, parameter);
      }
    } catch (Exception e) {
      throw new ExecutorException("Error getting generated key or setting result to parameter object. Cause: " + e, e);
    }
  }

  private void assignKeys(Configuration configuration, ResultSet rs, ResultSetMetaData rsmd, String[] keyProperties,
      Object parameter) throws SQLException {
    if (parameter instanceof ParamMap || parameter instanceof StrictMap) {
      // Multi-param or single param with @Param Map属性 参数
      assignKeysToParamMap(configuration, rs, rsmd, keyProperties, (Map<String, ?>) parameter);
    } else if (parameter instanceof ArrayList && !((ArrayList<?>) parameter).isEmpty()
        && ((ArrayList<?>) parameter).get(0) instanceof ParamMap) {
      // Multi-param or single param with @Param in batch operation list属性参数
      assignKeysToParamMapList(configuration, rs, rsmd, keyProperties, (ArrayList<ParamMap<?>>) parameter);
    } else {
      // Single param without @Param 普通对象参数
      assignKeysToParam(configuration, rs, rsmd, keyProperties, parameter);
    }
  }

```

#### 2.SelectKeyGenerator

- 对于不支持主键自动自增的，可以使用<selectKey>标签生成主键，需要设置主键生成的SQL，需要表明SQL执行前执行还是执行后执行，都是调用processGeneratedKeys方法实现

```java
 private void processGeneratedKeys(Executor executor, MappedStatement ms, Object parameter) {
    try {
      if (parameter != null && keyStatement != null && keyStatement.getKeyProperties() != null) {
        String[] keyProperties = keyStatement.getKeyProperties();
        final Configuration configuration = ms.getConfiguration();
        final MetaObject metaParam = configuration.newMetaObject(parameter);
        // Do not close keyExecutor.
        // The transaction will be closed by parent executor.
        Executor keyExecutor = configuration.newExecutor(executor.getTransaction(), ExecutorType.SIMPLE);
        //主键查询SQL
        List<Object> values = keyExecutor.query(keyStatement, parameter, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER);
        if (values.size() == 0) {
          //主键SQL没有返回值
          throw new ExecutorException("SelectKey returned no data.");
        } else if (values.size() > 1) {
          //主键SQL返回值有多个
          throw new ExecutorException("SelectKey returned more than one value.");
        } else {
          //主键SQL返回值有一个
          MetaObject metaResult = configuration.newMetaObject(values.get(0));
          if (keyProperties.length == 1) {
            //主键参数有一个
            if (metaResult.hasGetter(keyProperties[0])) {
              //有getter ,映射值
              setValue(metaParam, keyProperties[0], metaResult.getValue(keyProperties[0]));
            } else {
              // no getter for the property - maybe just a single value object
              // so try that
              setValue(metaParam, keyProperties[0], values.get(0));
            }
          } else {
            //多个值的主键
            handleMultipleProperties(keyProperties, metaParam, metaResult);
          }
        }
      }
    } catch (ExecutorException e) {
      throw e;
    } catch (Exception e) {
      throw new ExecutorException("Error selecting key or setting result to parameter object. Cause: " + e, e);
    }
  }

    private void handleMultipleProperties(String[] keyProperties,
      MetaObject metaParam, MetaObject metaResult) {
    String[] keyColumns = keyStatement.getKeyColumns();

    if (keyColumns == null || keyColumns.length == 0) {
      // no key columns specified, just use the property names
      //没有指定主键，则通过字段名称来设置值
      for (String keyProperty : keyProperties) {
        setValue(metaParam, keyProperty, metaResult.getValue(keyProperty));
      }
    } else {
      if (keyColumns.length != keyProperties.length) {
        throw new ExecutorException("If SelectKey has key columns, the number must match the number of key properties.");
      }
      for (int i = 0; i < keyProperties.length; i++) {
        //有多个列值，通过列名给对应的列设置值
        setValue(metaParam, keyProperties[i], metaResult.getValue(keyColumns[i]));
      }
    }
  }
```

```sql
<insert id=”insert” >
<selectKey keyProperty=”id” resultType=”int” order=”BEFORE”> 
-- 插入前填充ID值
SELECT FLOOR(RAND() * 10000) ;
</selectKey>
insert into T_USER (ID, username,pwd) values ( #{id} , #{username} , #{pwd) ) 
</insert>
```