---
layout:     post
title:      Mybatis核心处理层-SqlSource/SqlNode
subtitle:   组合模式/OGNL
date:       2020-07-14
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
    - 组合模式
    - OGNL
---

## SqlSource/SqlNode

### 一、SqlSource

#### 1.组合模式

- 通过组合的模式，用户可以像调用简单组件那样调用复杂组件，并且不需要知道复杂组件内部的执行逻辑
- 优点：操作简单，屏蔽复杂性，符合“开放-封闭”原则
- 组成部分

```text
抽象组件 -- Component

树叶  --- Leaf

树枝  --- Composite

调用者 --- Client
```

#### 2.OGNL 表达式的使用

- 表达式
- root 对象
- OgnlContext 上下文对象
- root对象和数组的对象如何访问

#### 3.Dynamic Context

- 主要负责记录解析动态SQL之后的SQL语句片段，是记录结果的容器

- ContextMap DynamicContext内部类，继承自HashMap，修改了get方法

```java
//继承自HashMap
  static class ContextMap extends HashMap<String, Object> {
    private static final long serialVersionUID = 2977601501966151582L;
    private final MetaObject parameterMetaObject;
    private final boolean fallbackParameterObject;

    //创建是传入运行参数列表
    public ContextMap(MetaObject parameterMetaObject, boolean fallbackParameterObject) {
      this.parameterMetaObject = parameterMetaObject;
      this.fallbackParameterObject = fallbackParameterObject;
    }

    @Override
    public Object get(Object key) {
      String strKey = (String) key;
      //存在key
      if (super.containsKey(strKey)) {
        //直接返回值
        return super.get(strKey);
      }

      if (parameterMetaObject == null) {
        return null;
      }

      if (fallbackParameterObject && !parameterMetaObject.hasGetter(strKey)) {
        return parameterMetaObject.getOriginalObject();
      } else {
        // issue #61 do not modify the context when reading
        //在运行的参数中查找对应属性
        return parameterMetaObject.getValue(strKey);
      }
    }
  }
```

- 最为常用的是appendSql  和 getSql

### 二、SqlNode

#### 1.SqlNode: TextSqlNode /StaticTextSqlNode / MixedSqlNode / IfSqlNode

#### 2.StaticTextSqlNode

- 非动态SQL 

#### 3.MixedSqlNode

- List SqlNode 会循环解析内部所有SqlNode

#### 4.TextSqlNode

- 用于解析${}

#### 5.IfSqlNode

- test 的 apply方法，用于判断是否执行IF 片段

#### 6.TrimSqlNode

- 用于添加或删除前缀或后缀

#### 7.WhereSqlNode

- 如果以AND OR开头，就将其删除，替换为where

#### 8.SetSqlNode

- 如果以“，”结尾，就将其删除，在开头添加SET

#### 9.ForeachSqlNode

- 内部类 PrefixedContext

```java
 private class PrefixedContext extends DynamicContext {
    //context
    private final DynamicContext delegate;
    //前缀
    private final String prefix;
    //是否已经处理过前缀
    private boolean prefixApplied;
    ...


    @Override
    public void appendSql(String sql) {
      if (!prefixApplied && sql != null && sql.trim().length() > 0) {
        //没有处理过prefix 并且SQL含有内容
        delegate.appendSql(prefix);
        prefixApplied = true;
      }
      delegate.appendSql(sql);
    }
 }
```

#### 10.ChooseSqlNode

- choose when otherwise --> when 替换为IfSqlNode  otherwise 替换为MixedSqlNode

#### 11.VarDeclSqlNode

- 动态SQL中的<bind>节点，可以从OGNL表达式中创建一个变量并将其记录到上下文中

#### 12.SqlSourceBuilder

- 进行SQL #{}占位符替换

```java
 public SqlSource parse(
    String originalSql, //sql语句 
    Class<?> parameterType, //用户传入的实参
    Map<String, Object> additionalParameters // 形参和实参的对应关系
  ) {
    // ParameterMappingTokenHandler 是解析#{}的核心
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    //使用 GenericTokenParser 与 ParameterMappingTokenHandler 共同解析#{}占位符
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    String sql;
    if (configuration.isShrinkWhitespacesInSql()) {
      sql = parser.parse(removeExtraWhitespaces(originalSql));
    } else {
      sql = parser.parse(originalSql);
    }
    //创建 StaticSqlSource 内含SQL 包含?和parameterMapping 
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }
```

- ParameterMappingTokenHandler

```java
 private static class ParameterMappingTokenHandler extends BaseBuilder implements TokenHandler {

    //参数映射列表
    private List<ParameterMapping> parameterMappings = new ArrayList<>();
    //参数类型
    private Class<?> parameterType;
    //DynamicContext.bindings 对应的MetaObject对象
    private MetaObject metaParameters;

    ...
     //SQL处理入口
    @Override
    public String handleToken(String content) {
      parameterMappings.add(buildParameterMapping(content));
      return "?";  //返回问号占位符
    }

    private ParameterMapping buildParameterMapping(String content) {
      Map<String, String> propertiesMap = parseParameterMapping(content);
      String property = propertiesMap.get("property");
      Class<?> propertyType;
      if (metaParameters.hasGetter(property)) { // issue #448 get type from additional params
        propertyType = metaParameters.getGetterType(property);
      } else if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
        propertyType = parameterType;
      } else if (JdbcType.CURSOR.name().equals(propertiesMap.get("jdbcType"))) {
        propertyType = java.sql.ResultSet.class;
      } else if (property == null || Map.class.isAssignableFrom(parameterType)) {
        propertyType = Object.class;
      } else {
        MetaClass metaClass = MetaClass.forClass(parameterType, configuration.getReflectorFactory());
        if (metaClass.hasGetter(property)) {
          propertyType = metaClass.getGetterType(property);
        } else {
          propertyType = Object.class;
        }
      }
      ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
      Class<?> javaType = propertyType;
      String typeHandlerAlias = null;
      for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        if ("javaType".equals(name)) {
          javaType = resolveClass(value);
          builder.javaType(javaType);
        } else if ("jdbcType".equals(name)) {
          builder.jdbcType(resolveJdbcType(value));
        } else if ("mode".equals(name)) {
          builder.mode(resolveParameterMode(value));
        } else if ("numericScale".equals(name)) {
          builder.numericScale(Integer.valueOf(value));
        } else if ("resultMap".equals(name)) {
          builder.resultMapId(value);
        } else if ("typeHandler".equals(name)) {
          typeHandlerAlias = value;
        } else if ("jdbcTypeName".equals(name)) {
          builder.jdbcTypeName(value);
        } else if ("property".equals(name)) {
          // Do Nothing
        } else if ("expression".equals(name)) {
          throw new BuilderException("Expression based parameters are not supported yet");
        } else {
          throw new BuilderException("An invalid property '" + name + "' was found in mapping #{" + content + "}.  Valid properties are " + PARAMETER_PROPERTIES);
        }
      }
      if (typeHandlerAlias != null) {
        builder.typeHandler(resolveTypeHandler(javaType, typeHandlerAlias));
      }
      return builder.build();
    }
 }
```

- DynamicSqlSource 获取BoundSql

```java
  @Override
  public BoundSql getBoundSql(Object parameterObject) {
    //创建承载结果的上下文
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    //组合模式，调用整个树形结构中全部的SqlNode.apply，每个SqlNode解析的结果都添加到context中
    rootSqlNode.apply(context);
    //创建SQlSourceBuilder, 将#{} 替换为？占位符
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);

    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    //创建BoundSql对象
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    context.getBindings().forEach(boundSql::setAdditionalParameter);
    return boundSql;
  }

```

- RawSqlSource
  当 XMLScriptBuilder.parseDynamicTags() 如果只含有#{} ,不含有动态节点或者${} 这不是动态SQL，则创建响应的StaticTextSqlNode的对象；
  当 XMLScriptBuilder.parseScriptNode() 如果判断不是动态SQL，则创建相应的RawSqlSource对象


- 综上，StaticSqlSource,DynamicSqlSource（处理动态SQL，在真正执行之前进行解析和替换），RawSqlSource（处理静态SQL，在mybatis完成初始化之后就进行解析替换） 最终都会获取到BoundSql对象，包含一个带有?占位符的SQL， parameter参数映射关系集合，用户传入的参数集合

