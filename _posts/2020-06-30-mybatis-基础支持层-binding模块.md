---
layout:     post
title:      Mybatis基础支持层-Binding模块
subtitle:   Binding/代理/工厂
date:       2020-06-30
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
    - Binding
    - 代理/工厂
---

# mybatis基础支持层-Binding模块

## 一、MapperRegistry

- Mapper的注册中心，主要用于addMapper和getMapper,并对已经加载的Mapper进行缓存
- addMapper

```java
 public <T> void addMapper(Class<T> type) {
    //是否为接口
    if (type.isInterface()) {
      //已知mapper列表是否有Mapper映射
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        //解析注解
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```

- getMapper

```java
  @SuppressWarnings("unchecked")
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      //实例化，完成创建接口代理对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

## 二、MapperProxyFactory

- Mapper代理工厂，主要通newInstance 获取MapperProxy对象
- 对每一个Mapper类，也会缓存其包含的方法及其调用的映射

```java
  //Mapper接口对应的Class对象
  private final Class<T> mapperInterface;
  //缓存Mapper中的方法，以及其实现的调用
  private final Map<Method, MapperMethodInvoker> methodCache = new ConcurrentHashMap<>();

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
    //创建s实现Mapper的接口代理对象
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
   //创建MapperProxy对象
    //每次调用都会创建一个新的 MapperProxy对象
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

## 三、 MapperProxy

- 属性

```java
//允许的方法修饰符
  private static final int ALLOWED_MODES = MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
      | MethodHandles.Lookup.PACKAGE | MethodHandles.Lookup.PUBLIC;
  // Lookup对象的构造方法
  private static final Constructor<Lookup> lookupConstructor;
  //Lookup对象的私有方法
  private static final Method privateLookupInMethod;
  //执行SQL的会话
  private final SqlSession sqlSession;
  //Mapper接口
  private final Class<T> mapperInterface;
  //方法以及方法的调用映射   MapperMethodInvoker用以完成参数转换以及SQL语句的执行
  private final Map<Method, MapperMethodInvoker> methodCache;
```

- 静态代码块初始化Lookup方法的构造器，以便后续初始化Lookup，最终使用MethodHandler,调用方法

```java
static {
    Method privateLookupIn;
    try {
      //以下创建Lookup,是为了使用MethodHandler 为了能够动态修改方法参数的类型和他们的顺序，和反射不同，是在创建的时候就完成了访问检查，和反射在运行时再检查不同，可以节省一些消耗
      //调用 MethodHandles   Lookup 的私有构造方法
      //MethodHandles.publicLookup() 共有方法的调用就比较简单，但是不能够控制  allowedModes
      privateLookupIn = MethodHandles.class.getMethod("privateLookupIn", Class.class, MethodHandles.Lookup.class);
    } catch (NoSuchMethodException e) {
      privateLookupIn = null;
    }
    privateLookupInMethod = privateLookupIn;

    Constructor<Lookup> lookup = null;
    if (privateLookupInMethod == null) {
      // JDK 1.8
      try {
        lookup = MethodHandles.Lookup.class.getDeclaredConstructor(Class.class, int.class);
        lookup.setAccessible(true);
      } catch (NoSuchMethodException e) {
        throw new IllegalStateException(
            "There is neither 'privateLookupIn(Class, Lookup)' nor 'Lookup(Class, int)' method in java.lang.invoke.MethodHandles.",
            e);
      } catch (Exception e) {
        lookup = null;
      }
    }
    lookupConstructor = lookup;
  }
```

- 了解一下最核心的invoke方法，如果缓存中存在方法代理的映射，则直接返回使用。如果不存在，就要新建Invoker并将其缓存起来

```java
 public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //如果目标是继承自Object，就直接调用
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else {
        //调用方法并缓存Method和Proxy的映射
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }

  private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
    try {
      // A workaround for https://bugs.openjdk.java.net/browse/JDK-8161372
      // It should be removed once the fix is backported to Java 8 or
      // MyBatis drops Java 8 support. See gh-1929
      //在Mapper映射中获取invoker
      MapperMethodInvoker invoker = methodCache.get(method);
      if (invoker != null) {
        //存在invoker直接返回
        return invoker;
      }

      //无invoker 添加进入缓存
      return methodCache.computeIfAbsent(method, m -> {
        if (m.isDefault()) {  //是默认方法
          try {
            if (privateLookupInMethod == null) {
              //方法不存在jdk8
              return new DefaultMethodInvoker(getMethodHandleJava8(method));
            } else {
              return new DefaultMethodInvoker(getMethodHandleJava9(method));
            }
          } catch (IllegalAccessException | InstantiationException | InvocationTargetException
              | NoSuchMethodException e) {
            throw new RuntimeException(e);
          }
        } else {
          //不是默认方法
          return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
        }
      });
    } catch (RuntimeException re) {
      Throwable cause = re.getCause();
      throw cause == null ? re : cause;
    }
  }

  private static class PlainMethodInvoker implements MapperMethodInvoker {
    //内含MapperMethod对象
    private final MapperMethod mapperMethod;

    public PlainMethodInvoker(MapperMethod mapperMethod) {
      super();
      this.mapperMethod = mapperMethod;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
      //直接拿到指定方法执行
      return mapperMethod.execute(sqlSession, args);
    }
  }

  private static class DefaultMethodInvoker implements MapperMethodInvoker {
    private final MethodHandle methodHandle;

    public DefaultMethodInvoker(MethodHandle methodHandle) {
      super();
      this.methodHandle = methodHandle;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
      //bindTo 摸一个instance 再调用并带有参数
      return methodHandle.bindTo(proxy).invokeWithArguments(args);
    }
  }
```

## 四、MapperMethod

- 内含两个属性，同时也是内部类:SqlCommand(记录SQL详情)、MethodSignature（记录方法的参数、参数值、返回值等）

### 1.SqlCommand

- 记录指令 :SQL语句名称 类型 与type字段

```java
//静态内部类
  public static class SqlCommand {
    //方法名称
    private final String name;
    //方法类型   UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH
    private final SqlCommandType type;

    //初始化name 和 SqlCommandType
    public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
      //获取方法名称
      final String methodName = method.getName();
      final Class<?> declaringClass = method.getDeclaringClass();
      MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
          configuration);
      if (ms == null) {
        //最终没有找到MappedStatement
        if (method.getAnnotation(Flush.class) != null) {
          //flush 指令
          name = null;
          type = SqlCommandType.FLUSH;
        } else {
          //不存在
          throw new BindingException("Invalid bound statement (not found): "
              + mapperInterface.getName() + "." + methodName);
        }
      } else {
        name = ms.getId();
        type = ms.getSqlCommandType();
        if (type == SqlCommandType.UNKNOWN) {
          throw new BindingException("Unknown execution method for: " + name);
        }
      }
    }

     //获取MappedStatement
    private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName, Class<?> declaringClass, Configuration configuration) {
      //类名加上方法名是唯一标识
      String statementId = mapperInterface.getName() + "." + methodName;
      if (configuration.hasStatement(statementId)) {
        //mapperStatementMap中存在，直接返回对象
        return configuration.getMappedStatement(statementId);
      } else if (mapperInterface.equals(declaringClass)) {
        return null;
      }
      for (Class<?> superInterface : mapperInterface.getInterfaces()) {
        //  superInterface 是 declaringClass 的子类
        if (declaringClass.isAssignableFrom(superInterface)) {
          MappedStatement ms = resolveMappedStatement(superInterface, methodName, declaringClass, configuration);
          if (ms != null) {
            return ms;
          }
        }
      }
      return null;
    }
  } 
```

### 2.MethodSignature

> 内含方法的一些详细信息,例如返回值\参数等名称\值\类型的记录

#### paramNameResolver

- 主要负责记录参数的名称与值,初始化的时候能够解析参数的index和参数名的映射关系,内部通过一个SortedMap<Integer, String> names,记录参数名的顺序

```java
public ParamNameResolver(Configuration config, Method method) {
    this.useActualParamName = config.isUseActualParamName();
    //获取参数的类型
    final Class<?>[] paramTypes = method.getParameterTypes();
    //获取参数的注解
    final Annotation[][] paramAnnotations = method.getParameterAnnotations();
    //有序的Map 记录index和content
    final SortedMap<Integer, String> map = new TreeMap<>();
    int paramCount = paramAnnotations.length;
    // get names from @Param annotations
    //获取@Param修饰的参数，遍历所有方法参数
    for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
      //是RowBounds子类，或者RestHandler子类 跳过分析
      if (isSpecialParameter(paramTypes[paramIndex])) {
        // skip special parameters
        continue;
      }
      String name = null;
      for (Annotation annotation : paramAnnotations[paramIndex]) {
        //只获取 注解是@Param的value值
        if (annotation instanceof Param) {
          hasParamAnnotation = true;
          //获取value值
          name = ((Param) annotation).value();
          break;
        }
      }
      if (name == null) {
        // 没有@Param注解，@Param was not specified.
        //根据参数  useActualParamName - 是否使用参数名作为value的配置
        if (useActualParamName) {
          name = getActualParamName(method, paramIndex);
        }
        if (name == null) {
          // use the parameter index as the name ("0", "1", ...)
          // gcode issue #71
          //使用参数的索引作为名称
          name = String.valueOf(map.size());
        }
      }
      //记录到map
      map.put(paramIndex, name);
    }
    names = Collections.unmodifiableSortedMap(map);
  }
```

- 通过以上方法进行参数的加载,通过getNamedParams 进行参数值的映射

```java
public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
      //无参数
      return null;
    } else if (!hasParamAnnotation && paramCount == 1) {
      //未使用@Param, 仅有一个参数
      Object value = args[names.firstKey()];
      return wrapToMapIfCollection(value, useActualParamName ? names.get(0) : null);
    } else {
      //多个参数场景
      final Map<String, Object> param = new ParamMap<>();
      int i = 0;
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
        //获取index对应的字段名，并从args的数组中找出对应index的value
        param.put(entry.getValue(), args[entry.getKey()]);
        // add generic param names (param1, param2, ...)
        //参数名就是命名成param+index的形式则不需要再添加
        final String genericParamName = GENERIC_NAME_PREFIX + (i + 1);
        // ensure not to overwrite parameter named with @Param
        if (!names.containsValue(genericParamName)) {
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }

```

#### MethodSignature

- 所含参数较多，主要描述返回值类型，以及参数列表解析的对象

```java
  //返回多个
    private final boolean returnsMany;
    //返回Map
    private final boolean returnsMap;
    //返回为空
    private final boolean returnsVoid;
    //返回 Cursor 类型
    private final boolean returnsCursor;
    private final boolean returnsOptional;
    //返回值类型
    private final Class<?> returnType;
    //如果返回值是Map,则记录了作为key的列名
    private final String mapKey;
    //标记结果处理器的位置
    private final Integer resultHandlerIndex;
    //标记rowBounds类型参数的位置
    private final Integer rowBoundsIndex;
    //参数名解析
    private final ParamNameResolver paramNameResolver;

      public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      //解析返回值类型
      Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
      //返回是类
      if (resolvedReturnType instanceof Class<?>) {
        this.returnType = (Class<?>) resolvedReturnType;
      } else if (resolvedReturnType instanceof ParameterizedType) {
        //返回是 ParameterizedType
        this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
      } else {
        this.returnType = method.getReturnType();
      }
      //初始化字段
      this.returnsVoid = void.class.equals(this.returnType);
      this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
      this.returnsCursor = Cursor.class.equals(this.returnType);
      this.returnsOptional = Optional.class.equals(this.returnType);
      //getMapKey 主要用于处理如果返回值为@MapKey注解修饰
      this.mapKey = getMapKey(method);
      this.returnsMap = this.mapKey != null;
      this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
      this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
      this.paramNameResolver = new ParamNameResolver(configuration, method);
    }
```

- 通过convertArgsToSqlCommandParam方法，调用上面介绍的paramNameResolver.getNamedParams方法，能够实现获取参数值的映射Map
- 通过getUniqueParamIndex 查看参数列表中是否有与传入的paramType一致或其子类的参数，来判断是否含有paramType
- 通过getMapKey，搜索MapKey的注解的值

### 3.executeXXX核心方法

- 类型根据返回值的需求分为：executeForMany（sqlSession.selectList），executeForMap(sqlSession.selectMap)，executeForCursor(sqlSession.selectCursor),sqlSession.selectOne等，下面介绍一下最为常用的executeForMany，executeForMap

- executeForMany

```java
 //处理返回值为集合或数组类型
  private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
    List<E> result;
    //获取参数值映射
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      //sqlSession.selectList进行查询
      result = sqlSession.selectList(command.getName(), param, rowBounds);
    } else {
      result = sqlSession.selectList(command.getName(), param);
    }
    // issue #510 Collections & arrays support
    //转换结果
    if (!method.getReturnType().isAssignableFrom(result.getClass())) {
      if (method.getReturnType().isArray()) {
        //转换为数组
        return convertToArray(result);
      } else {
        //转换为集合对象
        return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
      }
    }
    return result;
  }
  private <E> Object convertToDeclaredCollection(Configuration config, List<E> list) {
    Object collection = config.getObjectFactory().create(method.getReturnType());
    //生成MetaObject
    MetaObject metaObject = config.newMetaObject(collection);
    //添加结果，实际上就是调用Collection.addAll
    metaObject.addAll(list);
    return collection;
  }

  @SuppressWarnings("unchecked")
  private <E> Object convertToArray(List<E> list) {
    //获取Array的类型
    Class<?> arrayComponentType = method.getReturnType().getComponentType();
    //创建一个固定大小的array
    Object array = Array.newInstance(arrayComponentType, list.size());
    if (arrayComponentType.isPrimitive()) {
      //类型为原始类型，（boolean、char、byte、short、int、long、float、double）
      //对每一位设置参数值
      for (int i = 0; i < list.size(); i++) {
        Array.set(array, i, list.get(i));
      }
      return array;
    } else {
      //类型为自定义类型
      return list.toArray((E[]) array);
    }
  }

```

- executeForMap

```java
 //处理返回值为Map的情况
  private <K, V> Map<K, V> executeForMap(SqlSession sqlSession, Object[] args) {
    Map<K, V> result;
    Object param = method.convertArgsToSqlCommandParam(args);
    if (method.hasRowBounds()) {
      RowBounds rowBounds = method.extractRowBounds(args);
      //调用 sqlSession.selectMap
      result = sqlSession.selectMap(command.getName(), param, method.getMapKey(), rowBounds);
    } else {
      result = sqlSession.selectMap(command.getName(), param, method.getMapKey());
    }
    return result;
  }
```
