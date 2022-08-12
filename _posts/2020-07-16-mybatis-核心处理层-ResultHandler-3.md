---
layout:     post
title:      Mybatis核心处理层-ResultHandler参数映射核心逻辑-3
subtitle:   结果集映射/嵌套映射
date:       2020-07-16
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
    - 结果集映射
    - 嵌套映射
---

## ResultSetHandler 参数映射核心逻辑 -3

### 一、嵌套映射

- 以上处理了普通映射类型，还有另一个处理分支是处理嵌套映射类型的

#### 1.handleRowValuesForNestedResultMap 里面很多方法是与上面处理简单类型是相同的，不再介绍

```java
 private void handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    final DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    //1.找到指定offset的行
    skipRows(resultSet, rowBounds);
    Object rowValue = previousRowValue;
    //2.检测是否还有需要映射的记录
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      //3.确定映射使用的ResultMap对象
      final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      //4.CacheKey可以作为缓存的key,也可以唯一确认一个结果对象
      final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
      // 5.根据CacheKey查找nestedResultObjects集合
      // nestedResultObjects 存储所有结果对象
      Object partialObject = nestedResultObjects.get(rowKey);
      // issue #577 && #542
      //6.检测  resultOrdered
      //resultOrdered 字段,仅针对嵌套的情况适用。
      // 用于判断映射完一个结果是是否清除nestResultObjects,能够尽快清理内存，减小内存使用
      if (mappedStatement.isResultOrdered()) {
        if (partialObject == null && rowValue != null) {
          //清除原有数据
          nestedResultObjects.clear();
          //存储新的Cache
          storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
        }
        //7.对ResultSet中的一行进行记录
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
      } else {
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
        if (partialObject == null) {
          //8. 存储结果对象
          storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
        }
      }
    }
    if (rowValue != null && mappedStatement.isResultOrdered() && shouldProcessMoreRows(resultContext, rowBounds)) {
      //8.存储结果对象
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
      previousRowValue = null;
    } else if (rowValue != null) {
      previousRowValue = rowValue;
    }
  }
```

- 进行嵌套处理的时候，会将内部的对象进行缓存，CacheKey可以作为缓存中的key值，也可以唯一标识一个结果对象

```java
 private CacheKey createRowKey(ResultMap resultMap, ResultSetWrapper rsw, String columnPrefix) throws SQLException {
    final CacheKey cacheKey = new CacheKey();
    cacheKey.update(resultMap.getId());
    List<ResultMapping> resultMappings = getResultMappingsForRowKey(resultMap);
    if (resultMappings.isEmpty()) {
      //没有找到result mapping
      if (Map.class.isAssignableFrom(resultMap.getType())) {
        //resultMap的类型是map
        //由结果集中所有类名以及当前记录行的所有列值一起构成CacheKey对象
        createRowKeyForMap(rsw, cacheKey);
      } else {
        //由结果集中未映射的列名以及他们在当前记录行中对应的列值一起构成CacheKey对象
        createRowKeyForUnmappedProperties(resultMap, rsw, cacheKey, columnPrefix);
      }
    } else {
      //由resultMapping中的列名和当前记录行中响应的列值一起构成CacheKey对象
      createRowKeyForMappedProperties(resultMap, rsw, cacheKey, resultMappings, columnPrefix);
    }
    if (cacheKey.getUpdateCount() < 2) {
      //没有找到构成CacheKey对象的内容，返回NullCachekey
      return CacheKey.NULL_CACHE_KEY;
    }
    return cacheKey;
  }
```

#### 2.createRowKeyForMappedProperties 对明确的列创建rowKey

```java
private void createRowKeyForMappedProperties(ResultMap resultMap, ResultSetWrapper rsw, CacheKey cacheKey, List<ResultMapping> resultMappings, String columnPrefix) throws SQLException {
    for (ResultMapping resultMapping : resultMappings) {
      if (resultMapping.isSimple()) {
        //获取列的名称
        final String column = prependPrefix(resultMapping.getColumn(), columnPrefix);
        //获取typeHandler对象
        final TypeHandler<?> th = resultMapping.getTypeHandler();
        // 获取映射的列名
        List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
        // Issue #114
        if (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH))) {
          //获取列的值
          final Object value = th.getResult(rsw.getResultSet(), column);
          if (value != null || configuration.isReturnInstanceForEmptyRow()) {
            //将列名列值添加到CacheKey中
            cacheKey.update(column);
            cacheKey.update(value);
          }
        }
      }
    }
  }
```

#### 3.getRowValue 获取一行的嵌套的内容，主要涉及嵌套处理逻辑

```java
//对一行数据进行映射
  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, CacheKey combinedKey, String columnPrefix, Object partialObject) throws SQLException {
    final String resultMapId = resultMap.getId();
    Object rowValue = partialObject;
    if (rowValue != null) {
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      putAncestor(rowValue, resultMapId);
      applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, false);
      ancestorObjects.remove(resultMapId);
    } else {
      final ResultLoaderMap lazyLoader = new ResultLoaderMap();
      //1. 创建映射后的结果对象
      rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
      if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        boolean foundValues = this.useConstructorMappings;
        //2.是否开启自动映射
        if (shouldApplyAutomaticMappings(resultMap, true)) {   //2.2 自动映射
          //2.1自动映射不明确的列
          foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
        }
        //2.3.映射ResultMap中明确的列
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
        //2.4 放入ancestorObjects集合
        putAncestor(rowValue, resultMapId);
        //2.5 处理嵌套映射
        foundValues = applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, true) || foundValues;
        //2.6 移除ancestorObjects结合
        ancestorObjects.remove(resultMapId);
        foundValues = lazyLoader.size() > 0 || foundValues;
        rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
      }
      if (combinedKey != CacheKey.NULL_CACHE_KEY) {
        nestedResultObjects.put(combinedKey, rowValue);
      }
    }
    return rowValue;
  }

```

#### 4.applyNestedResultMappings 处理嵌套的resultMapping

```java
private boolean applyNestedResultMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String parentPrefix, CacheKey parentRowKey, boolean newObject) {
    boolean foundValues = false;
    for (ResultMapping resultMapping : resultMap.getPropertyResultMappings()) {
      //获取nestedResultMapId
      final String nestedResultMapId = resultMapping.getNestedResultMapId();
      //1.检测   nestedResultMapId 和resultSet
      if (nestedResultMapId != null && resultMapping.getResultSet() == null) {
        //存在嵌套
        try {
          final String columnPrefix = getColumnPrefix(parentPrefix, resultMapping);
          //2.获取嵌套使用的resultmap
          final ResultMap nestedResultMap = getNestedResultMap(rsw.getResultSet(), nestedResultMapId, columnPrefix);
          if (resultMapping.getColumnPrefix() == null) {
            // try to fill circular reference only when columnPrefix
            // is not specified for the nested result map (issue #215)
            Object ancestorObject = ancestorObjects.get(nestedResultMapId);
            if (ancestorObject != null) {
              //3.循环引用问题
              if (newObject) {
                linkObjects(metaObject, resultMapping, ancestorObject); // issue #385
              }
              continue;
            }
          }
          //4.为嵌套对象创建CacheKey
          final CacheKey rowKey = createRowKey(nestedResultMap, rsw, columnPrefix);
          //与外层对象的CacheKey合并，得到全局唯一的CacheKey对象
          final CacheKey combinedKey = combineKeys(rowKey, parentRowKey);
          //查找处理过的相同key的嵌套对象
          Object rowValue = nestedResultObjects.get(combinedKey);
          boolean knownValue = rowValue != null;
          //5.如果嵌套类型为Collection类型，且未初始化，会通过下面的方法进行集合对象初始化
          instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject); // mandatory
          if (anyNotNullColumnHasValue(resultMapping, columnPrefix, rsw)) {
            //6.根据NotNullColumn的属性，检测相应列是否为空
            //7.通过 getRowValue 完成嵌套映射
            rowValue = getRowValue(rsw, nestedResultMap, combinedKey, columnPrefix, rowValue);
            if (rowValue != null && !knownValue) {
              //8.将嵌套对象保存到外层对象中
              linkObjects(metaObject, resultMapping, rowValue);
              foundValues = true;
            }
          }
        } catch (SQLException e) {
          throw new ExecutorException("Error getting nested result map values for '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
        }
      }
    }
    return foundValues;
  }
```

#### 5.linkObjects 处理循环引用 （A 内部属性 associate B ，B 内部属性 associate A）

```java
 private void linkObjects(MetaObject metaObject, ResultMapping resultMapping, Object rowValue) {
    //集合参数初始化
    final Object collectionProperty = instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject);
    if (collectionProperty != null) {
      final MetaObject targetMetaObject = configuration.newMetaObject(collectionProperty);
      //是集合就调用add
      targetMetaObject.add(rowValue);
    } else {
      //不是集合就直接设置参数即可
      metaObject.setValue(resultMapping.getProperty(), rowValue);
    }
  }
```

#### 6.instantiateCollectionPropertyIfAppropriate 如果集合属性没有初始化则初始化，已经初始化则返回值

```java
private Object instantiateCollectionPropertyIfAppropriate(ResultMapping resultMapping, MetaObject metaObject) {
    final String propertyName = resultMapping.getProperty();
    Object propertyValue = metaObject.getValue(propertyName);
    if (propertyValue == null) {
      //没有值
      Class<?> type = resultMapping.getJavaType();
      if (type == null) {
        type = metaObject.getSetterType(propertyName);
      }
      try {
        if (objectFactory.isCollection(type)) {
          //是集合，初始化一个空集合
          propertyValue = objectFactory.create(type);
          //设置value
          metaObject.setValue(propertyName, propertyValue);
          return propertyValue;
        }
      } catch (Exception e) {
        throw new ExecutorException("Error instantiating collection property for result '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
      }
    } else if (objectFactory.isCollection(propertyValue.getClass())) {
      //已经初始化，返回属性值即可
      return propertyValue;
    }
    return null;
  }
```

#### 7.combineKeys 相同cachekey 如果parent不同，需要生成不同的key

```java
  private CacheKey combineKeys(CacheKey rowKey, CacheKey parentRowKey) {
    //检查边界
    if (rowKey.getUpdateCount() > 1 && parentRowKey.getUpdateCount() > 1) {
      CacheKey combinedKey;
      try {
        //rowKey 克隆
        combinedKey = rowKey.clone();
      } catch (CloneNotSupportedException e) {
        throw new ExecutorException("Error cloning cache key.  Cause: " + e, e);
      }
      //与parentKey合并，由于外界的cachekey不同，内部的组合后最终的cachekey也就不相同了
      combinedKey.update(parentRowKey);
      return combinedKey;
    }
    return CacheKey.NULL_CACHE_KEY;
  }
```

