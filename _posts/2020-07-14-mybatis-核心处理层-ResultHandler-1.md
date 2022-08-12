---
layout:     post
title:      Mybatis核心处理层-ResultHandler参数映射核心逻辑-1
subtitle:   结果集映射
date:       2020-07-16
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
---

## ResultSetHandler 参数映射核心逻辑 -1

### 一、ResultSetHandler

```java
public interface ResultSetHandler {

  //处理ResultSet
  <E> List<E> handleResultSets(Statement stmt) throws SQLException;

  //处理游标ResultSet
  <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;

  //处理输出参数
  void handleOutputParameters(CallableStatement cs) throws SQLException;

}
```

### 二、实现类 DefaultResultSetHandler

- 包含的属性：

```java
  private final Executor executor;
  //mybatis配置类
  private final Configuration configuration;
  //
  private final MappedStatement mappedStatement;
  //limit offset
  private final RowBounds rowBounds;
  //参数处理器
  private final ParameterHandler parameterHandler;
  //结果处理器
  private final ResultHandler<?> resultHandler;
  //sql 对象
  private final BoundSql boundSql;
  //类型处理器注册中心
  private final TypeHandlerRegistry typeHandlerRegistry;
  //对象工厂
  private final ObjectFactory objectFactory;
  //反射工厂
  private final ReflectorFactory reflectorFactory;

  // nested resultmaps
  private final Map<CacheKey, Object> nestedResultObjects = new HashMap<>();
  private final Map<String, Object> ancestorObjects = new HashMap<>();
  private Object previousRowValue;

  // multiple resultsets
  private final Map<String, ResultMapping> nextResultMaps = new HashMap<>();
  private final Map<CacheKey, List<PendingRelation>> pendingRelations = new HashMap<>();

  // Cached Automappings
  private final Map<String, List<UnMappedColumnAutoMapping>> autoMappingsCache = new HashMap<>();

  // temporary marking flag that indicate using constructor mapping (use field to reduce memory usage)
  private boolean useConstructorMappings;
```

#### 1.处理结果 handleResultSets

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    //
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    //结果列表
    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    //获取第一个结果集对象
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    //MappedStatement 在初始化的时候 resultMap对象被解析为 ResultMap对象，保存在resultMaps的集合中
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    //验证结果集不为空，否则抛出异常
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      //获取ResultMap对象
      ResultMap resultMap = resultMaps.get(resultSetCount);
      //解析result内容
      handleResultSet(rsw, resultMap, multipleResults, null);
      //获取下一个集合
      rsw = getNextResultSet(stmt);
      //清空nestedResultObjects
      cleanUpAfterHandlingResultSet();
      //递增resultSetCount
      resultSetCount++;
    }
    //MappedStatement resultSets 针对于多结果集的情况
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        //根据resultSet的名称，获取未处理的ResultMapping
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          //根据ResultMap对象映射结果集
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        //获取下一个结果集
        rsw = getNextResultSet(stmt);
        //清空nestedResultObjects集合
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }

  private ResultSetWrapper getFirstResultSet(Statement stmt) throws SQLException {
    ResultSet rs = stmt.getResultSet();
    while (rs == null) {
      // move forward to get the first resultset in case the driver
      // doesn't return the resultset as the first result (HSQLDB 2.1)
      if (stmt.getMoreResults()) {
        rs = stmt.getResultSet();
      } else {
        if (stmt.getUpdateCount() == -1) {
          // no more results. Must be no resultset
          break;
        }
      }
    }
    return rs != null ? new ResultSetWrapper(rs, configuration) : null;
  }

//主要应对SQL片段内部还有嵌套的未解析的SQL片段的场景
  private ResultSetWrapper getNextResultSet(Statement stmt) {
    // Making this method tolerant of bad JDBC drivers
    try {
      if (stmt.getConnection().getMetaData().supportsMultipleResultSets()) {
        // Crazy Standard JDBC way of determining if there are more results
        if (!(!stmt.getMoreResults() && stmt.getUpdateCount() == -1)) {
          ResultSet rs = stmt.getResultSet();
          if (rs == null) {
            return getNextResultSet(stmt);
          } else {
            return new ResultSetWrapper(rs, configuration);
          }
        }
      }
    } catch (Exception e) {
      // Intentionally ignored.
    }
    return null;
  }
```

#### ResultSetWrapper 封装ResultSet的一些属性

```java
//resultSet对象
  private final ResultSet resultSet;
  //处理器注册中心
  private final TypeHandlerRegistry typeHandlerRegistry;
  //记录ResultSet 中每列列名
  private final List<String> columnNames = new ArrayList<>();
  //记录ResultSet 中每列对应的Java类型
  private final List<String> classNames = new ArrayList<>();
  //记录ResultSet 中每列对应的JdbcType类型
  private final List<JdbcType> jdbcTypes = new ArrayList<>();
  // 记录每列TypeHandler对象，key是列名，value是TypeHandler集合
  private final Map<String, Map<Class<?>, TypeHandler<?>>> typeHandlerMap = new HashMap<>();
  //记录被映射的列名，其中key是ResultMap对象的id，value是该ResultMap对象映射的类名集合
  private final Map<String, List<String>> mappedColumnNamesMap = new HashMap<>();
  //记录了未映射的列名，其中key是ResultMap对象的id，value是ResultMap对象未映射的列名集合
  private final Map<String, List<String>> unMappedColumnNamesMap = new HashMap<>();

```

- 初始化java类型 jdbc类型

```java
 final ResultSetMetaData metaData = rs.getMetaData();
    final int columnCount = metaData.getColumnCount();
    for (int i = 1; i <= columnCount; i++) {
      columnNames.add(configuration.isUseColumnLabel() ? metaData.getColumnLabel(i) : metaData.getColumnName(i));
      jdbcTypes.add(JdbcType.forCode(metaData.getColumnType(i)));
      classNames.add(metaData.getColumnClassName(i));
    }
```

- 获取映射与未映射的列名

```java
 public List<String> getMappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
    List<String> mappedColumnNames = mappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
    if (mappedColumnNames == null) {
      loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);
      mappedColumnNames = mappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
    }
    return mappedColumnNames;
  }

   private void loadMappedAndUnmappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
    List<String> mappedColumnNames = new ArrayList<>();
    List<String> unmappedColumnNames = new ArrayList<>();
    final String upperColumnPrefix = columnPrefix == null ? null : columnPrefix.toUpperCase(Locale.ENGLISH);
    final Set<String> mappedColumns = prependPrefixes(resultMap.getMappedColumns(), upperColumnPrefix);
    for (String columnName : columnNames) {
      final String upperColumnName = columnName.toUpperCase(Locale.ENGLISH);
      if (mappedColumns.contains(upperColumnName)) {
        mappedColumnNames.add(upperColumnName);
      } else {
        unmappedColumnNames.add(columnName);
      }
    }
    mappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), mappedColumnNames);
    unMappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), unmappedColumnNames);
  }
```

