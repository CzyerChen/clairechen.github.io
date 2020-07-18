---
layout:     post
title:      Mybatis核心处理层-ResultHandler参数映射核心逻辑-3
subtitle:   结果集映射/延迟加载/动态代理
date:       2020-07-16
author:     Claire
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java
    - Mybatis
    - 结果集映射
    - 延迟加载
    - 动态代理
---

# ResultSetHandler 参数映射核心逻辑 -4

## 一、嵌套查询&延迟加载

- 延迟加载的含义：对象会在真正时候的时候才会被加载到内存，并且调用。初始化时只绑定对应属性的代理对象，真正使用时，调用代理对象最终实现延迟加载属性

### 1.CGLIB 动态代理

- 在java中提到比较多的就是动态代理就包括：JDK动态代理、CGLIB动态代理
- 两者的区别就在，JDK动态代理依靠实现接口实现方法的代理，CGLIB利用字节码技术无需实现接口，继承的方式也能够实现类方法的代理

#### jdk动态代理

```java
public interface MethodInterface {
  String saySomething();
}

public class MethodImpl  implements MethodInterface{
  @Override
  public String saySomething() {
    return "Hello Impl";
  }
}
public class ProxyInstance implements InvocationHandler {
  private Object target;

  public Object getTarget() {
    return target;
  }

  public void setTarget(Object target) {
    this.target = target;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
     System.out.println("===调用中===");
    return method.invoke(target,args);
  }

  public Object createProxy(){
    return Proxy.newProxyInstance(target.getClass().getClassLoader(),target.getClass().getInterfaces(),this);
  }
}
public class DynamicInvokeTest {

  public static void main(String[] args) {
    //jdk动态代理
    ProxyInstance proxyInstance = new ProxyInstance();
    MethodInterface method = new MethodImpl();
    proxyInstance.setTarget(method);
    MethodInterface instance = (MethodInterface)proxyInstance.createProxy();
    String something = instance.saySomething();
    System.out.println(something);
  }
}
```

#### cglib动态代理

```java
public abstract class AbstractMethod {
  public abstract String saySomething();

  public String say(){
    System.out.println("Hello");
    return "Hello";
  }
}
public class MethodClass extends AbstractMethod{
  @Override
  public String saySomething() {
    return "Hello subClass";
  }
}
public class EnhanceProxy implements MethodInterceptor {

  public  Object createEnhancerPorxy(Class<?> superClass){
    Enhancer enhancer = new Enhancer();
    enhancer.setCallback(this);
    enhancer.setSuperclass(superClass);
    return enhancer.create();
  }

  @Override
  public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
    System.out.println("===调用中===");
    return methodProxy.invokeSuper(o,objects);
  }
}

public class DynamicInvokeTest {

  public static void main(String[] args) {
    //cglib 动态代理
    EnhanceProxy enhanceProxy = new EnhanceProxy();
    AbstractMethod enhancerPorxy = (AbstractMethod) enhanceProxy.createEnhancerPorxy(MethodClass.class);
    String something1 = enhancerPorxy.saySomething();
    System.out.println(something1);
  }
}
```

- 对比JDK动态代理/cglib动态代理：jdk对象创建时间比cglib对象创建时间短很多，cglib动态代理的执行时间是比jdk动态代理稍短
- 因而CGLIB动态代理更适合单例模式，由于cglib动态代理需要动态生成字节码，存储在JVM永久堆，较多使用可能造成OOM
- 因而Spring都是默认JDK动态代理，少许不实现接口的代理类使用CGLIB动态代理实现
- JDK动态代理，代理类实现InvokeHandler,cglib动态代理，代理类实现MethodInterceptor

### 2.javassist 代理

- javassist是一个开源的生成java字节码的类库，能够简单、快速使用API动态生成类
- 与cglib不同的是，代理类实现的是MethodHandler接口

```java
//添加javassit依赖包，可以去maven 仓库挑选版本，当前使用3.27.0-GA
public class JavassistTest {

  public static void main(String[] args) throws NotFoundException, CannotCompileException, IOException {
    ClassPool classPool = ClassPool.getDefault();
    CtClass ctClass = classPool.makeClass("org.apache.ibatis.executor.resultset.JavassitPerson");
    CtField ctField = new CtField(classPool.get("java.lang.String"), "name", ctClass);
    ctField.setModifiers(Modifier.PRIVATE);
    ctClass.addField(ctField, CtField.Initializer.constant("joe"));
    ctClass.addMethod(CtNewMethod.setter("setName",ctField));
    ctClass.addMethod(CtNewMethod.getter("getName",ctField));

    CtConstructor ctConstructor = new CtConstructor(new CtClass[]{}, ctClass);
    ctConstructor.setBody("{name=\"lily\";}");
    ctClass.addConstructor(ctConstructor);


    CtConstructor ctConstructor1 = new CtConstructor(new CtClass[]{classPool.get("java.lang.String")},ctClass);
    ctConstructor1.setBody("{$0.name=$1;}");
    ctClass.addConstructor(ctConstructor1);

    CtMethod ctMethod = new CtMethod(CtClass.voidType, "log", new CtClass[]{}, ctClass);
    ctMethod.setModifiers(Modifier.PUBLIC);
    ctMethod.setBody("{System.out.println(name);}");
    ctClass.addMethod(ctMethod);


    ctClass.writeFile("/Users/chenzy/files/gitFile/mybatis-3/src/test/java/org/apache/ibatis/executor/resultset");
  }
}

//最终生成类
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package org.apache.ibatis.executor.resultset;

public class JavassitPerson {
    private String name = "joe";

    public void setName(String var1) {
        this.name = var1;
    }

    public String getName() {
        return this.name;
    }

    public JavassitPerson() {
        this.name = "lily";
    }

    public JavassitPerson(String var1) {
        this.name = var1;
    }

    public void log() {
        System.out.println(this.name);
    }
}

```

- mybatis中使用动态代理就用到了Cglib和Javassit的方式

### 3.ResultLoader & ResultLoaderMap & proxyFactory

- ResultLoader 一次延迟加载所需的全部字段
- ResultLoaderMap 主要继续需要延迟加载的内容，一起加载
- PorxyFactory代理工厂，就包含CglibProxyFactory、JavassitProxyFactory两种

#### (1)ResultLoader

- 属性

```java
 //mybatis全局配置
  protected final Configuration configuration;
  //执行器
  protected final Executor executor;
  protected final MappedStatement mappedStatement;
  //实参
  protected final Object parameterObject;
  //返回值类型
  protected final Class<?> targetType;
  //对象工厂，用于实例化返回值对象
  protected final ObjectFactory objectFactory;
  //缓存
  protected final CacheKey cacheKey;
  //SQL对象
  protected final BoundSql boundSql;
  //结果处理器
  protected final ResultExtractor resultExtractor;
  //resultLoader 的id
  protected final long creatorThreadId;

  protected boolean loaded;
  //结果对象
  protected Object resultObject;
```

- 下面看延迟加载所有信息的流程，loadResult是入口方法

```java
//入口
  public Object loadResult() throws SQLException {
    //执行延迟加载，将结果化为一个list
    List<Object> list = selectList();
    //将list结果转化为target对象
    resultObject = resultExtractor.extractObjectFromList(list, targetType);
    return resultObject;
  }

   private <E> List<E> selectList() throws SQLException {
    //延迟加载的executor对象
    Executor localExecutor = executor;
    if (Thread.currentThread().getId() != this.creatorThreadId || localExecutor.isClosed()) {
      //如果id不相同，或者已关闭，就开启一个新的executor
      localExecutor = newExecutor();
    }
    try {
      //执行查询
      return localExecutor.query(mappedStatement, parameterObject, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER, cacheKey, boundSql);
    } finally {
      if (localExecutor != executor) {
        localExecutor.close(false);
      }
    }
  }

  public Object extractObjectFromList(List<Object> list, Class<?> targetType) {
    Object value = null;
    if (targetType != null && targetType.isAssignableFrom(list.getClass())) {
      //结果为list类型
      value = list;
    } else if (targetType != null && objectFactory.isCollection(targetType)) {
      //结果为集合类型
      value = objectFactory.create(targetType);
      MetaObject metaObject = configuration.newMetaObject(value);
      //创建完对象后直接使用addAll添加结果
      metaObject.addAll(list);
    } else if (targetType != null && targetType.isArray()) {
      //数组
      Class<?> arrayComponentType = targetType.getComponentType();
      Object array = Array.newInstance(arrayComponentType, list.size());
      //创建一个list长度的数组
      if (arrayComponentType.isPrimitive()) {
        //数组类型是原始类型，直接设置参数
        for (int i = 0; i < list.size(); i++) {
          Array.set(array, i, list.get(i));
        }
        value = array;
      } else {
        //不是原始类型，就转成Object数组
        value = list.toArray((Object[])array);
      }
    } else {
      //不是集合或数组
      if (list != null && list.size() > 1) {
        //对象数量大于1，出现异常，无法接收
        throw new ExecutorException("Statement returned more than one row, where no more than one was expected.");
      } else if (list != null && list.size() == 1) {
        //只有一个对象就返回这个对象即可
        value = list.get(0);
      }
    }
    return value;
  }
```

#### (2)ResultLoaderMap 

- 属性

```java
//保存属性及其ResultLoader对象的映射，其中key是大些的属性名 ，LoadPair是内部类，封装了ResultLoader
  private final Map<String, LoadPair> loaderMap = new HashMap<>();
```

- LoadPair 属性

```java
/**
     * Name of factory method which returns database connection.
     */
    private static final String FACTORY_METHOD = "getConfiguration";
    /**
     * Object to check whether we went through serialization..
     */
    private final transient Object serializationCheck = new Object();
    /**
     * Meta object which sets loaded properties.
     * 外层对象
     */
    private transient MetaObject metaResultObject;
    /**
     * Result loader which loads unread properties.
     * 负责延迟加载 核心方法loadResult
     */
    private transient ResultLoader resultLoader;
    /**
     * Wow, logger.
     */
    private transient Log log;
    /**
     * Factory class through which we get database connection.
     */
    private Class<?> configurationFactory;
    /**
     * Name of the unread property.
     * 属性名
     */
    private String property;
    /**
     * ID of SQL statement which loads the property.
     */
    private String mappedStatement;
    /**
     * Parameter of the sql statement.
     */
    private Serializable mappedParameter;
```

- ResultLoaderMap 两个入口方法load loadAll

```java

  //入口
  public boolean load(String property) throws SQLException {
    //移除指定属性
    LoadPair pair = loaderMap.remove(property.toUpperCase(Locale.ENGLISH));
    if (pair != null) {
      pair.load();
      return true;
    }
    return false;
  }

  //入口
  public void loadAll() throws SQLException {
    final Set<String> methodNameSet = loaderMap.keySet();
    String[] methodNames = methodNameSet.toArray(new String[methodNameSet.size()]);
    for (String methodName : methodNames) {
      load(methodName);
    }
  }

   //入库
    public void load() throws SQLException {
      /* These field should not be null unless the loadpair was serialized.
       * Yet in that case this method should not be called. */
      if (this.metaResultObject == null) {
        throw new IllegalArgumentException("metaResultObject is null");
      }
      if (this.resultLoader == null) {
        throw new IllegalArgumentException("resultLoader is null");
      }

      this.load(null);
    }

    //最终方法
    public void load(final Object userObject) throws SQLException {
      if (this.metaResultObject == null || this.resultLoader == null) {
        if (this.mappedParameter == null) {
          throw new ExecutorException("Property [" + this.property + "] cannot be loaded because "
                  + "required parameter of mapped statement ["
                  + this.mappedStatement + "] is not serializable.");
        }

        final Configuration config = this.getConfiguration();
        final MappedStatement ms = config.getMappedStatement(this.mappedStatement);
        if (ms == null) {
          throw new ExecutorException("Cannot lazy load property [" + this.property
                  + "] of deserialized object [" + userObject.getClass()
                  + "] because configuration does not contain statement ["
                  + this.mappedStatement + "]");
        }

        this.metaResultObject = config.newMetaObject(userObject);
        //初始化ResultLoader
        this.resultLoader = new ResultLoader(config, new ClosedExecutor(), ms, this.mappedParameter,
                metaResultObject.getSetterType(this.property), null, null);
      }

      /* We are using a new executor because we may be (and likely are) on a new thread
       * and executors aren't thread safe. (Is this sufficient?)
       *
       * A better approach would be making executors thread safe. */
      if (this.serializationCheck == null) {
        final ResultLoader old = this.resultLoader;
        this.resultLoader = new ResultLoader(old.configuration, new ClosedExecutor(), old.mappedStatement,
                old.parameterObject, old.targetType, old.cacheKey, old.boundSql);
      }

      //最终就是调用ResultLoader的 this.resultLoader.loadResult() 进行结果的延迟加载
      this.metaResultObject.setValue(property, this.resultLoader.loadResult());
    }
```

#### (3)ProxyFactory

- ProxyFactory的两个子类，一个使用cglib 一个使用Javassit , 挑选比较熟悉的CGLIB了解一下执行过程
- CGLIB动态代理上面的流程也了解到了，会有createProxy的方法创建代理，其次就是实现MethodInterceptor的方法调用intercept方法

```java
  @Override
  public Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    return EnhancedResultObjectProxyImpl.createProxy(target, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
  }

   //EnhancedResultObjectProxyImpl 是内部类，实现了MethodInterceptor
   private static class EnhancedResultObjectProxyImpl implements MethodInterceptor{
         //代理的目标类
    private final Class<?> type;
    //延迟加载的Map， 最后依靠ResultLoader进行延迟加载
    private final ResultLoaderMap lazyLoader;
    //mybatis-config中的参数配置，aggressiveLazyLoading 配置项的值
    private final boolean aggressive;
    //延迟加载的方法名列表
    private final Set<String> lazyLoadTriggerMethods;
    //对象工厂
    private final ObjectFactory objectFactory;
    //构造参数类型列表
    private final List<Class<?>> constructorArgTypes;
    //构造参数实参列表
    private final List<Object> constructorArgs;

    //静态方法调用，创建代理方法
    public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      final Class<?> type = target.getClass();
      //创建callback对象
      EnhancedResultObjectProxyImpl callback = new EnhancedResultObjectProxyImpl(type, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
      //创建代理对象
      Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
      //参数复制
      PropertyCopier.copyBeanProperties(type, target, enhanced);
      return enhanced;
    }

    //当callback方法被调用时，就会调用
    @Override
    public Object intercept(Object enhanced, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
      //方法名
      final String methodName = method.getName();
      try {
        //同步resultLoaderMap对象
        synchronized (lazyLoader) {
          if (WRITE_REPLACE_METHOD.equals(methodName)) {
            Object original;
            if (constructorArgTypes.isEmpty()) {
              //创建实际对象
              original = objectFactory.create(type);
            } else {
              //创建实际对象
              original = objectFactory.create(type, constructorArgTypes, constructorArgs);
            }
            PropertyCopier.copyBeanProperties(type, enhanced, original);
            if (lazyLoader.size() > 0) {
              //需要延迟加载
              return new CglibSerialStateHolder(original, lazyLoader.getProperties(), objectFactory, constructorArgTypes, constructorArgs);
            } else {
              return original;
            }
          } else {
            //不是writeReplace方法
            if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
              //存在延迟加载，并且不是finalize方法
              if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                //配置了延迟加载
                //全部属性加载完成
                lazyLoader.loadAll();
              } else if (PropertyNamer.isSetter(methodName)) {
                //setter方法
                final String property = PropertyNamer.methodToProperty(methodName);
                lazyLoader.remove(property);
              } else if (PropertyNamer.isGetter(methodName)) {
                //getter方法
                final String property = PropertyNamer.methodToProperty(methodName);
                if (lazyLoader.hasLoader(property)) {
                  //检测是否为延迟加载属性
                  //如果延迟加载，则加载属性
                  lazyLoader.load(property);
                }
              }
            }
          }
        }
        //调用目标对象方法
        return methodProxy.invokeSuper(enhanced, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
  }
  }

//创建cglib用到的enhancer对象，这个步骤就比较熟悉了
  static Object crateProxy(Class<?> type, Callback callback, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    //通过CGLIB动态代理，代理type这个类,绑定callback方法
    Enhancer enhancer = new Enhancer();
    enhancer.setCallback(callback);
    enhancer.setSuperclass(type);
    try {
      type.getDeclaredMethod(WRITE_REPLACE_METHOD);
      // ObjectOutputStream will call writeReplace of objects returned by writeReplace
      if (LogHolder.log.isDebugEnabled()) {
        LogHolder.log.debug(WRITE_REPLACE_METHOD + " method was found on bean " + type + ", make sure it returns this");
      }
    } catch (NoSuchMethodException e) {
      enhancer.setInterfaces(new Class[] { WriteReplaceInterface.class });
    } catch (SecurityException e) {
      // nothing to do here
    }
    Object enhanced;
    if (constructorArgTypes.isEmpty()) {
      //无参构造
      enhanced = enhancer.create();
    } else {
      //有参构造
      Class<?>[] typesArray = constructorArgTypes.toArray(new Class[constructorArgTypes.size()]);
      Object[] valuesArray = constructorArgs.toArray(new Object[constructorArgs.size()]);
      enhanced = enhancer.create(typesArray, valuesArray);
    }
    return enhanced;
  }

```

#### (4)如何通过以上的动态代理实现嵌套结果的获取

```java
private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    //嵌套查询id
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    final String property = propertyMapping.getProperty();
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    //参数类型
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    //参数值
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
      //获取嵌套查询的sql 和 cachekey
      final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
      final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
      //目标类型
      final Class<?> targetType = propertyMapping.getJavaType();
      //检测缓存中是否存在嵌套结果对象
      if (executor.isCached(nestedQuery, key)) {
        //创建 deferLoad 对象，从缓存中加载结果对象
        executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
        value = DEFERRED;
      } else {
        //延迟加载
        final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
        if (propertyMapping.isLazy()) {
          //配置了延迟加载
          //加入resultLoaderMap中，需要使用的时候再加载
          lazyLoader.addLoader(property, metaResultObject, resultLoader);
          value = DEFERRED;
        } else {
          //没有配置延迟加载
          value = resultLoader.loadResult();
        }
      }
    }
    return value;
  }

  private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    this.useConstructorMappings = false; // reset previous mapping result
    final List<Class<?>> constructorArgTypes = new ArrayList<>();
    final List<Object> constructorArgs = new ArrayList<>();
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
      for (ResultMapping propertyMapping : propertyMappings) {
        // issue gcode #109 && issue #149
        if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
          //存在嵌套对象 ，并且是延迟加载属性
          resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
          break;
        }
      }
    }
    this.useConstructorMappings = resultObject != null && !constructorArgTypes.isEmpty(); // set current mapping result
    return resultObject;
  }
```

#### (5)多结果集映射

- applyPropertyMappings 中的多结果集场景

```java
  if (propertyMapping.isCompositeResult()  //与嵌套查询配合使用，将参数值传递给内层作为参数
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH))) // 基本类型的属性映射
          || propertyMapping.getResultSet() != null) { //多结果集的场景处理
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
       ...
          }

  //明确属性值映射
  private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    if (propertyMapping.getNestedQueryId() != null) {
      //存在嵌套
      return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
      //存在多结果集
      addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
      return DEFERRED;
    } else {
      //直接处理
      final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
      //处理列前缀
      final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      return typeHandler.getResult(rs, column);
    }
  }        
```

- addPendingChildRelation 

```java
 private void addPendingChildRelation(ResultSet rs, MetaObject metaResultObject, ResultMapping parentMapping) throws SQLException {
    //1.创建缓存cacheKey:parentMapping + parentMapping.column + 值
    CacheKey cacheKey = createKeyForMultipleResults(rs, parentMapping, parentMapping.getColumn(), parentMapping.getColumn());
    // 2.PendingRelation 记录了当前结果对象对应的MetaObject 和 parentMapping对象
    PendingRelation deferLoad = new PendingRelation();
    deferLoad.metaObject = metaResultObject;
    deferLoad.propertyMapping = parentMapping;
    List<PendingRelation> relations = pendingRelations.computeIfAbsent(cacheKey, k -> new ArrayList<>());
    // issue #255
    //3.添加panding relation
    relations.add(deferLoad);
    ResultMapping previous = nextResultMaps.get(parentMapping.getResultSet());
    if (previous == null) {
      //4.内部嵌套对象，放入  nextResultMaps 中
      nextResultMaps.put(parentMapping.getResultSet(), parentMapping);
    } else {
      if (!previous.equals(parentMapping)) {
        throw new ExecutorException("Two different properties are mapped to the same resultSet");
      }
    }
  }

 private CacheKey createKeyForMultipleResults(ResultSet rs, ResultMapping resultMapping, String names, String columns) throws SQLException {
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(resultMapping);
    if (columns != null && names != null) {
      //按照逗号切割列名
      String[] columnsArray = columns.split(",");
      String[] namesArray = names.split(",");
      for (int i = 0; i < columnsArray.length; i++) {
        //查看值
        Object value = rs.getString(columnsArray[i]);
        if (value != null) {
          //添加列名和值
          cacheKey.update(namesArray[i]);
          cacheKey.update(value);
        }
      }
    }
    return cacheKey;
  }
```

- handleResultSets 中处理多结果集的情况

```java
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

```

- storeObject 存储结果中处理多结果集情况

```java
  private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    if (parentMapping != null) {
      //多结果集
      linkToParents(rs, parentMapping, rowValue);
    } else {
      //普通映射
      callResultHandler(resultHandler, resultContext, rowValue);
    }
  }


  // MULTIPLE RESULT SETS
  private void linkToParents(ResultSet rs, ResultMapping parentMapping, Object rowValue) throws SQLException {
    //多结果集
    CacheKey parentKey = createKeyForMultipleResults(rs, parentMapping, parentMapping.getColumn(), parentMapping.getForeignColumn());
    List<PendingRelation> parents = pendingRelations.get(parentKey);
    if (parents != null) {
      for (PendingRelation parent : parents) {
        if (parent != null && rowValue != null) {
          linkObjects(parent.metaObject, parent.propertyMapping, rowValue);
        }
      }
    }
  }

   //将已经存在的嵌套对象设置到外层对象的相应属性中
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

#### (6)游标

- Cursor 接口 实现类DefaultCursor,继承了Iterable 接口
- DefaultResultSetHandler.handleCursorResultSets

```java
  public <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling cursor results").object(mappedStatement.getId());
     //获取结果封装成 ResultSetWrapper
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    //获取映射所需的resultMap列表
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();

    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    //只能映射一个结果集，因而只能存在一个ResultMap对象
    if (resultMapCount != 1) {
      throw new ExecutorException("Cursor results cannot be mapped to multiple resultMaps");
    }
    //获取第一个ResultMap对象
    ResultMap resultMap = resultMaps.get(0);
    //构建游标对象
    return new DefaultCursor<>(this, resultMap, rsw, rowBounds);
  }
```

- 内部属性

```java
  // ResultSetHandler stuff
  //结果处理
  private final DefaultResultSetHandler resultSetHandler;
  //映射使用的resultMap
  private final ResultMap resultMap;
  private final ResultSetWrapper rsw;
  //映射的起止位置
  private final RowBounds rowBounds;
  protected final ObjectWrapperResultHandler<T> objectWrapperResultHandler = new ObjectWrapperResultHandler<>();

  //通过迭代器获取映射得到的结果对象
  private final CursorIterator cursorIterator = new CursorIterator();
  //是否正在迭代结果集
  private boolean iteratorRetrieved;

  private CursorStatus status = CursorStatus.CREATED;
  //记录已经完成的行数
  private int indexWithRowBound = -1;
```

- 获取DefaultCursor对象后，可以调用iterator获取迭代器对结果集进行迭代，迭代器是CursorIterator
- 查看迭代的next方法

```java
 @Override
    public T next() {
      // Fill next with object fetched from hasNext()
      T next = object;

      //尚未获取结果
      if (!objectWrapperResultHandler.fetched) {
        next = fetchNextUsingRowBound();
      }

      //已经获取结果
      if (objectWrapperResultHandler.fetched) {
        //修改标识符
        objectWrapperResultHandler.fetched = false;
        //置空结果对象
        object = null;
        //递增处理行数
        iteratorIndex++;
        //返回对象
        return next;
      }
      throw new NoSuchElementException();
    }

    protected T fetchNextUsingRowBound() {
    //获取一行数据对象
    T result = fetchNextObjectFromDatabase();
    //从结果集中一条条映射，忽略offset之前的映射结果
    while (objectWrapperResultHandler.fetched && indexWithRowBound < rowBounds.getOffset()) {
      result = fetchNextObjectFromDatabase();
    }
    return result;
  }

  protected T fetchNextObjectFromDatabase() {
    //游标是否关闭
    if (isClosed()) {
      return null;
    }

    try {
      //尚未获取标识
      objectWrapperResultHandler.fetched = false;
      //游标处于打开状态
      status = CursorStatus.OPEN;
      //resultset 未关闭
      if (!rsw.getResultSet().isClosed()) {
        //处理一行内容，使用 resultSetHandler 进行处理
        resultSetHandler.handleRowValues(rsw, resultMap, objectWrapperResultHandler, RowBounds.DEFAULT, null);
      }
    } catch (SQLException e) {
      throw new RuntimeException(e);
    }

    //获取最终处理结果
    T next = objectWrapperResultHandler.result;
    if (objectWrapperResultHandler.fetched) {
      //递增处理行数
      indexWithRowBound++;
    }
    // No more object or limit reached
    if (!objectWrapperResultHandler.fetched || getReadItemsCount() == rowBounds.getOffset() + rowBounds.getLimit()) {
      //无内容可消费，关闭结果集，游标修改状态
      close();
      status = CursorStatus.CONSUMED;
    }
    //置空结果，为下一次解析做准备
    objectWrapperResultHandler.result = null;

    return next;
  }
```

#### (7)输出类型的参数

- DefaultResultSetHandler.handleOutputParameters

```java
 @Override
  public void handleOutputParameters(CallableStatement cs) throws SQLException {
    //获取用户传入的实际参数，并为其创建响应的MetaObject对象
    final Object parameterObject = parameterHandler.getParameterObject();
    final MetaObject metaParam = configuration.newMetaObject(parameterObject);
    //获取参数相关信息
    final List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    for (int i = 0; i < parameterMappings.size(); i++) {
      final ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() == ParameterMode.OUT || parameterMapping.getMode() == ParameterMode.INOUT) {
        if (ResultSet.class.equals(parameterMapping.getJavaType())) {
          //如果是resultSet类型则需要映射
          handleRefCursorOutputParameter((ResultSet) cs.getObject(i + 1), parameterMapping, metaParam);
        } else {
          //直接获取typeHandler获取结果，并设置到parameterObject中
          final TypeHandler<?> typeHandler = parameterMapping.getTypeHandler();
          metaParam.setValue(parameterMapping.getProperty(), typeHandler.getResult(cs, i + 1));
        }
      }
    }
  }

  private void handleRefCursorOutputParameter(ResultSet rs, ParameterMapping parameterMapping, MetaObject metaParam) throws SQLException {
    if (rs == null) {
      return;
    }
    try {
      final String resultMapId = parameterMapping.getResultMapId();
      final ResultMap resultMap = configuration.getResultMap(resultMapId);
      final ResultSetWrapper rsw = new ResultSetWrapper(rs, configuration);
      if (this.resultHandler == null) {
        final DefaultResultHandler resultHandler = new DefaultResultHandler(objectFactory);
        handleRowValues(rsw, resultMap, resultHandler, new RowBounds(), null);
        metaParam.setValue(parameterMapping.getProperty(), resultHandler.getResultList());
      } else {
        handleRowValues(rsw, resultMap, resultHandler, new RowBounds(), null);
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rs);
    }
  }
```