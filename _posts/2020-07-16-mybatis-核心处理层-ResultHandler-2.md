---
layout:     post
title:      Mybatis核心处理层-ResultHandler参数映射核心逻辑-2
subtitle:   结果集映射/简单映射/自动映射
date:       2020-07-16
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
    - 结果集映射
---

## ResultSetHandler 参数映射核心逻辑 -2

### 一、简单映射

```java
  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        //多结果集映射，外层的结果集
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
          //未指定结果集处理器，则使用 DefaultResultHandler
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          //处理映射，将结果对象添加到defaultResultHandler
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          //将defaultResultHandler中结果添加到multiResults
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          //使用用户指定的结果集处理器进行处理
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rsw.getResultSet());
    }
  }
```

#### 1.无论是多结果集、未指定resulthandler 或 指定resultHandler 都会调用handleRowValues 作为入口

```java
 public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
      ensureNoRowBounds();
      checkResultHandler();
      //嵌套内容
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
        //简单内容
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }

```

#### 2.handleRowValuesForSimpleResultMap 处理简单映射

```java
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    //1 找到指定Offset行
    skipRows(resultSet, rowBounds);
    //判断是否处理行数达到上线
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      //解析ResultMap
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      //获取ResultSet中的记录
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      //存储结果对象，放入ResultHanlder的resultlIST中
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
  }

```

- skipRows

```java
  //skipRows
    private void skipRows(ResultSet rs, RowBounds rowBounds) throws SQLException {
    //判断resultMap类型
    if (rs.getType() != ResultSet.TYPE_FORWARD_ONLY) {
      if (rowBounds.getOffset() != RowBounds.NO_ROW_OFFSET) {
        //直接定位到offset未知
        rs.absolute(rowBounds.getOffset());
      }
    } else {
      //多次调用到offset的位置
      for (int i = 0; i < rowBounds.getOffset(); i++) {
        if (!rs.next()) {
          break;
        }
      }
    }
  }

```

- shouldProcessMoreRows

```java
//shouldProcessMoreRows
  private boolean shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds) {
    //检测DefaultResultContext的stopped字段，结果集数量小于RowBounds.limit
    return !context.isStopped() && context.getResultCount() < rowBounds.getLimit();
  }
```

- resolveDiscriminatedResultMap

```java
  //resolveDiscriminatedResultMap
   public ResultMap resolveDiscriminatedResultMap(ResultSet rs, ResultMap resultMap, String columnPrefix) throws SQLException {
    //记录处理过得ResultMap id
    Set<String> pastDiscriminators = new HashSet<>();
    //获取 Discriminator 对象
    Discriminator discriminator = resultMap.getDiscriminator();
    while (discriminator != null) {
      //获取Discriminator 的值
      final Object value = getDiscriminatorValue(rs, discriminator, columnPrefix);
      //查找mapId
      final String discriminatedMapId = discriminator.getMapIdFor(String.valueOf(value));
      if (configuration.hasResultMap(discriminatedMapId)) {
        //存在resultMap
        resultMap = configuration.getResultMap(discriminatedMapId);
        //记录当前Discriminator 对象
        Discriminator lastDiscriminator = discriminator;
        discriminator = resultMap.getDiscriminator();
        //判断不是环形引用
        if (discriminator == lastDiscriminator || !pastDiscriminators.add(discriminatedMapId)) {
          break;
        }
      } else {
        break;
      }
    }
    return resultMap;
  }
```

- getRowValue

```java
  //getRowValue
    private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    //延迟加载
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    //1. 创建该行记录映射后得到的结果对象，该对象的类型由resultMap的type决定
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      //创建相应的MetaObject对象
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      // foundValues--true 成功映射任意属性；foundValues--false 未成功映射所有属性
      boolean foundValues = this.useConstructorMappings;
      //检测是否进行自动映射
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        //2.自动映射未明确指定的列
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
      }
      //3.映射明确的列
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      //如果没有成功映射任务属性，则根据mybatis-config.xml中的 rreturnInstanceForEmptyRow 决定是否返回空对象或者null
      rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
  }
```

- createResultObject

```java
 // createResultObject 的最终方法
  private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
      throws SQLException {
    //获取ResultMap中记录的type属性，也就是该行记录最终映射成的结果对象类型
    final Class<?> resultType = resultMap.getType();
    //
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    //获取constrcutor节点
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
      //1.存在类型处理器
      return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    } else if (!constructorMappings.isEmpty()) {
      //2.存在constructor节点
      return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
      //3.是接口，或者有默认无参构造函数
      return objectFactory.create(resultType);
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
      //4.不是嵌套的，开启自动映射 ，查找何时的构造函数，并创建结果对象
      return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs);
    }
    //抛出异常
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
  }
```

- createParameterizedResultObject

```java
Object createParameterizedResultObject(ResultSetWrapper rsw, Class<?> resultType, List<ResultMapping> constructorMappings,
                                         List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix) {
    boolean foundValues = false;
    //遍历构造函数参数
    for (ResultMapping constructorMapping : constructorMappings) {
      //构造参数类型
      final Class<?> parameterType = constructorMapping.getJavaType();
      final String column = constructorMapping.getColumn();
      final Object value;
      try {
        if (constructorMapping.getNestedQueryId() != null) {
          value = getNestedQueryConstructorValue(rsw.getResultSet(), constructorMapping, columnPrefix);
        } else if (constructorMapping.getNestedResultMapId() != null) {
          final ResultMap resultMap = configuration.getResultMap(constructorMapping.getNestedResultMapId());
          value = getRowValue(rsw, resultMap, getColumnPrefix(columnPrefix, constructorMapping));
        } else {
          //获取 TypeHandler 直接获取结果
          final TypeHandler<?> typeHandler = constructorMapping.getTypeHandler();
          value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(column, columnPrefix));
        }
      } catch (ResultMapException | SQLException e) {
        throw new ExecutorException("Could not process result for mapping: " + constructorMapping, e);
      }
      //记录参数类型
      constructorArgTypes.add(parameterType);
      //记录参数实际值
      constructorArgs.add(value);
      foundValues = value != null || foundValues;
    }
    // objectFactory.create 创建对象
    return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
  }
```

- createByConstructorSignature

```java
  private Object createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) throws SQLException {
    //查找构造函数
    final Constructor<?>[] constructors = resultType.getDeclaredConstructors();
    //查找默认无参构造函数
    final Constructor<?> defaultConstructor = findDefaultConstructor(constructors);
    if (defaultConstructor != null) {
      //存在无参构造
      return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, defaultConstructor);
    } else {
      //不存在无参构造
      for (Constructor<?> constructor : constructors) {
        if (allowedConstructorUsingTypeHandlers(constructor, rsw.getJdbcTypes())) {
          //存在对应类型的类型处理器
          return createUsingConstructor(rsw, resultType, constructorArgTypes, constructorArgs, constructor);
        }
      }
    }
    throw new ExecutorException("No constructor found in " + resultType.getName() + " matching " + rsw.getClassNames());
  }
```

- createUsingConstructor 通过constructor创建结果对象

```java
  private Object createUsingConstructor(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, Constructor<?> constructor) throws SQLException {
    boolean foundValues = false;
    for (int i = 0; i < constructor.getParameterTypes().length; i++) {
      //获取参数类型
      Class<?> parameterType = constructor.getParameterTypes()[i];
      //参数名称
      String columnName = rsw.getColumnNames().get(i);
      //获取类型处理器
      TypeHandler<?> typeHandler = rsw.getTypeHandler(parameterType, columnName);
      //获取实际值
      Object value = typeHandler.getResult(rsw.getResultSet(), columnName);
      //添加处理过得类型
      constructorArgTypes.add(parameterType);
      //添加处理过的值
      constructorArgs.add(value);
      foundValues = value != null || foundValues;
    }
    //创建结果对象  objectFactory.create
    return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
  }
```

- shouldApplyAutomaticMappings 是否使用自动映射

```java
  private boolean shouldApplyAutomaticMappings(ResultMap resultMap, boolean isNested) {
    //resultMap 配置autoMapping
    if (resultMap.getAutoMapping() != null) {
      return resultMap.getAutoMapping();
    } else {
      if (isNested) {
        //是嵌套的情况，settings --AutoMappingBehavior 配置FULL
        return AutoMappingBehavior.FULL == configuration.getAutoMappingBehavior();
      } else {
        //不是嵌套情况，settings -- AutoMappingBehavior 配置不是NONE即可
        return AutoMappingBehavior.NONE != configuration.getAutoMappingBehavior();
      }
    }
  }

```

- applyAutomaticMappings 自动映射逻辑

```java
private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (!autoMapping.isEmpty()) {
      //自动映射
      //遍历没有明确映射配置的列
      for (UnMappedColumnAutoMapping mapping : autoMapping) {
        //获取实际值
        final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          metaObject.setValue(mapping.property, value);
        }
      }
    }
    return foundValues;
  }
```

- applyPropertyMappings 处理明确参数映射的

```java
private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    //
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
      //处理列前缀
      String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);

      if (propertyMapping.getNestedResultMapId() != null) {
        // the user added a column attribute to a nested result map, ignore it
        //该属性需要使用一个嵌套ResultMap进行映射，忽略column
        column = null;
      }
      if (propertyMapping.isCompositeResult()  //与嵌套查询配合使用，将参数值传递给内层作为参数
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH))) // 基本类型的属性映射
          || propertyMapping.getResultSet() != null) { //多结果集的场景处理
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
        // issue #541 make property optional
        //获取属性名称
        final String property = propertyMapping.getProperty();
        if (property == null) {
          continue;
        } else if (value == DEFERRED) {
          // DEFERRED 标识占位符对象
          foundValues = true;
          continue;
        }
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          //设置属性值
          metaObject.setValue(property, value);
        }
      }
    }
    return foundValues;
  }

```

- storeObject

```java
  //storeObject
    //存储结果对象，如果是嵌套子类，就将结果存储到父类的属性中，如果是普通映射，就存储到ResultHandler中
  private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    if (parentMapping != null) {
      //嵌套对象
      linkToParents(rs, parentMapping, rowValue);
    } else {
      //普通映射
      callResultHandler(resultHandler, resultContext, rowValue);
    }
  }
```

