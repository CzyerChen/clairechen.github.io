---
layout:     post
title:      Spring javaconfig化改造
subtitle:   去xml、注解化
date:       2022-01-29
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - spring
    - xml
    - 注解
---

> 背景说明：<br/>
> 我们的代码都会基于某些组件从某一个版本开始开发，然后面临不断迭代更新的过程。<br/>
>今天我们这边就是面临一个从Spring4到Spring5，从配置文件到注解注入的改造。

> 依赖版本的变化主要有：
> 
> 1. Spring 4.3.18.RELEASE -> 5.3.6
> 2. Spring Data 1.11.14.RELEASE  ->	2.5.1
> 3. Hibernate	4.2.4.Final -> 5.4.29.Final
> 4. jedis	 2.10.0 -> 3.6.0
> 5. 本地缓存 guava 18.0 -> caffeine 2.9.2


> 遇到的问题大约有：
> 
> 1. 如何去除web.xml
> 2. 如何去除applicationContext.xml
> 3. 如何去除springmvc.xml
> 4. 如何扫描注入配置文件
> 5. 测试方法如何启动

-----------
**目录**

- [一、准备内容](#一准备内容)
  - [一个正常能跑的springmvc项目](#一个正常能跑的springmvc项目)
  - [applicationContext.xml样例](#applicationcontextxml样例)
  - [springmvc.xml样例](#springmvcxml样例)
  - [redis-context.xml样例](#redis-contextxml样例)
  - [web.xml样例](#webxml样例)
- [二、去除web.xml](#二去除webxml)
- [三、去除applicationContext.xml](#三去除applicationcontextxml)
- [四、去除springmvc.xml](#四去除springmvcxml)
- [五、去除数据库xml配置](#五去除数据库xml配置)
- [六、去除redis-context.xml配置](#六去除redis-contextxml配置)
- [七、测试方法配置](#七测试方法配置)
- [Refer](#refer)

------------

## 一、准备内容

### 一个正常能跑的springmvc项目

```
   .//java
   .//resources
   .//resources/applicationContext.xml
   .//resources/springmvc.xml
   .//resources/config.properties
   .//resources/db.properties
   .//resources/redis-context.xml
   .//resources/redis.properties
   .//resources/logback.xml
   .//webapp
   .//webapp/res
   .//webapp/css
   .//webapp/images
   .//webapp/js
   .//webapp/common
   .//webapp/WEB-INF
   .//webapp/WEB-INF/web.xml
   .//webapp/third-party
   .//webapp/data
```

### applicationContext.xml样例

```xml  
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:jpa="http://www.springframework.org/schema/data/jpa"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:jaxws="http://cxf.apache.org/jaxws"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
		http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/tx 
		http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
		http://www.springframework.org/schema/context 
		http://www.springframework.org/schema/context/spring-context-4.0.xsd
		http://www.springframework.org/schema/data/jpa 
		http://www.springframework.org/schema/data/jpa/spring-jpa-1.3.xsd
		http://cxf.apache.org/jaxws
        http://cxf.apache.org/schemas/jaxws.xsd">


    <import resource="redis-context.xml"/>
    <context:property-placeholder location="classpath:db.properties,classpath:config.properties"
                                  ignore-unresolvable="true"/>
    <context:component-scan base-package="com.learning">
        <context:exclude-filter type="annotation"
                                expression="org.springframework.stereotype.Controller"/>
        <context:exclude-filter type="annotation"
                                expression="org.springframework.web.bind.annotation.ControllerAdvice"/>
    </context:component-scan>
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
          init-method="init" destroy-method="close">
        <property name="driverClassName" value="${jdbc.driverClass}"/>
        <property name="url" value="${jdbc.jdbcUrl}"/>
        <property name="username" value="${jdbc.user}"/>
        <property name="password" value="${jdbc.password}"/>
        <property name="initialSize" value="${minPoolSize}"/>
        <property name="minIdle" value="${minPoolSize}"/>
        <property name="maxActive" value="${maxPoolSize}"/>
        <property name="maxWait" value="60000"/>
        <property name="removeAbandoned" value="${removeAbandoned}" />
        <property name="removeAbandonedTimeout" value="${removeAbandonedTimeout}" />
        <property name="logAbandoned" value="${logAbandoned}"/>
        <property name="timeBetweenEvictionRunsMillis" value="60000"/>
        <property name="minEvictableIdleTimeMillis" value="300000"/>
        <property name="validationQuery" value="SELECT 'x'"/>
        <property name="testWhileIdle" value="true"/>
        <property name="testOnBorrow" value="false"/>
        <property name="testOnReturn" value="false"/>
        <property name="poolPreparedStatements" value="true"/>
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20"/>
        <property name="proxyFilters">
            <list>
                <ref bean="stat-fliter"/>
                <ref bean="log-filter"/>
            </list>
        </property>
    </bean>

    <bean id="stat-fliter" class="com.alibaba.druid.filter.stat.StatFilter"/>

    <bean id="log-filter" class="com.alibaba.druid.filter.logging.Slf4jLogFilter">
        <property name="connectionLogEnabled" value="true"/>
        <property name="statementLogEnabled" value="true"/>
        <property name="resultSetLogEnabled" value="true"/>
    </bean>

    <bean id="appCtxUtil" class="com.learning.AppCtxUtil"></bean>

    <bean id="entityManagerFactory"
          class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
        <property name="dataSource" ref="dataSource"></property>
        <property name="jpaVendorAdapter">
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter"></bean>
        </property>
        <property name="packagesToScan" value="com.learning.model"></property>
        <property name="jpaProperties">
            <props>
                <prop key="hibernate.hbm2ddl.auto">${hibernate.hbm2ddl.auto}</prop>
                <prop key="hibernate.dialect">${hibernate.dialect}</prop>
                <prop key="hibernate.show_sql">${hibernate.show_sql}</prop>
                <prop key="hibernate.format_sql">${hibernate.format_sql}</prop>
                <prop key="hibernate.max_fetch_depth">${hibernate.max_fetch_depth}</prop>
                <prop key="hibernate.jdbc.fetch_size">${hibernate.jdbc.fetch_size}</prop>
                <prop key="hibernate.jdbc.batch_size">${hibernate.jdbc.batch_size}</prop>
                <prop key="hibernate.temp.use_jdbc_metadata_defaults">
                    ${hibernate.temp.use_jdbc_metadata_defaults}
                </prop>
                <prop key="hibernate.enable_lazy_load_no_trans">${hibernate.enable_lazy_load_no_trans}</prop>
            </props>
        </property>
    </bean>

    <bean id="transactionManager"
          class="org.springframework.orm.jpa.JpaTransactionManager">
        <property name="entityManagerFactory" ref="entityManagerFactory"></property>
    </bean>

    <jpa:repositories base-package="com.learning.repository"
                      entity-manager-factory-ref="entityManagerFactory"
                      transaction-manager-ref="transactionManager">
    </jpa:repositories>

    <tx:annotation-driven transaction-manager="transactionManager"/>

</beans>

```

### springmvc.xml样例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:aop="http://www.springframework.org/schema/aop" xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
		http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
        http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd">

    <aop:aspectj-autoproxy proxy-target-class="true"/>
    <context:property-placeholder location="classpath:config.properties" signore-unresolvable="true"/>
	
	<context:component-scan base-package="com.learning.admin,com.learning.monitor" use-default-filters="false">
		<context:include-filter type="annotation" 
			expression="org.springframework.stereotype.Controller"/>
		<context:include-filter type="annotation" 
			expression="org.springframework.web.bind.annotation.ControllerAdvice"/>
	</context:component-scan>
    <context:annotation-config/>


    <mvc:view-controller path="/" view-name="redirect:login.do?method=toLogin"/>
    <mvc:resources location="/images/" mapping="/images/**"/>
    <mvc:resources location="/js/" mapping="/js/**" />
    <mvc:resources location="/css/" mapping="/css/**" />
   	<mvc:default-servlet-handler/>
	
	<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView" />
        <property name="prefix" value="/WEB-INF/pages/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

	<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">  
        <property name="defaultEncoding" value="utf-8"></property>
        <property name="maxUploadSize" value="20971520"></property>
        <property name="resolveLazily" value="true"/>
   </bean>
	
	<mvc:annotation-driven/>
    <mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/**"/>
            <bean class="com.learning.filter.ReferrerInterceptor"/>
        </mvc:interceptor>
    </mvc:interceptors>
	
	<bean class="com.learning.exception.ExceptionHandle">
</bean> 
	
</beans>


```

### redis-context.xml样例

```xml  
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd">

    <!-- scanner redis properties -->
    <context:property-placeholder location="classpath:redis.properties"
                                  ignore-unresolvable="true"/>

    <bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxIdle" value="${redis.maxIdle}"/>
        <property name="minIdle" value="${redis.minIdle}"/>
        <property name="maxTotal" value="${redis.maxTotal}"/>
        <property name="maxWaitMillis" value="${redis.maxWaitMillis}"/>
        <property name="testOnBorrow" value="${redis.testOnBorrow}"/>
        <property name="testOnReturn" value="${redis.testOnReturn}"/>
        <property name="testWhileIdle" value="${redis.testWhileIdle}"/>
        <property name="timeBetweenEvictionRunsMillis" value="${redis.timeBetweenEvictionRunsMillis}"/>
        <property name="minEvictableIdleTimeMillis" value="${redis.minEvictableIdleTimeMillis}"/>
        <property name="jmxEnabled" value="${redis.jmxEnabled}"/>
    </bean>

    <bean id="cachepoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxIdle" value="${cache.maxIdle}"/>
        <property name="minIdle" value="${cache.minIdle}"/>
        <property name="maxTotal" value="${cache.maxTotal}"/>
        <property name="maxWaitMillis" value="${redis.maxWaitMillis}"/>
        <property name="testOnBorrow" value="${redis.testOnBorrow}"/>
        <property name="testOnReturn" value="${redis.testOnReturn}"/>
        <property name="testWhileIdle" value="${redis.testWhileIdle}"/>
        <property name="timeBetweenEvictionRunsMillis" value="${redis.timeBetweenEvictionRunsMillis}"/>
        <property name="minEvictableIdleTimeMillis" value="${redis.minEvictableIdleTimeMillis}"/>
        <property name="jmxEnabled" value="${redis.jmxEnabled}"/>
    </bean>

    <bean id="statispoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxIdle" value="${statis.maxIdle}"/>
        <property name="minIdle" value="${statis.minIdle}"/>
        <property name="maxTotal" value="${statis.maxTotal}"/>
        <property name="maxWaitMillis" value="${redis.maxWaitMillis}"/>
        <property name="testOnBorrow" value="${redis.testOnBorrow}"/>
        <property name="testOnReturn" value="${redis.testOnReturn}"/>
        <property name="testWhileIdle" value="${redis.testWhileIdle}"/>
        <property name="timeBetweenEvictionRunsMillis" value="${redis.timeBetweenEvictionRunsMillis}"/>
        <property name="minEvictableIdleTimeMillis" value="${redis.minEvictableIdleTimeMillis}"/>
        <property name="jmxEnabled" value="${redis.jmxEnabled}"/>
    </bean>
    <bean id="submitpoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxIdle" value="${submit.maxIdle}"/>
        <property name="minIdle" value="${submit.minIdle}"/>
        <property name="maxTotal" value="${submit.maxTotal}"/>
        <property name="maxWaitMillis" value="${redis.maxWaitMillis}"/>
        <property name="testOnBorrow" value="${redis.testOnBorrow}"/>
        <property name="testOnReturn" value="${redis.testOnReturn}"/>
        <property name="testWhileIdle" value="${redis.testWhileIdle}"/>
        <property name="timeBetweenEvictionRunsMillis" value="${redis.timeBetweenEvictionRunsMillis}"/>
        <property name="minEvictableIdleTimeMillis" value="${redis.minEvictableIdleTimeMillis}"/>
        <property name="jmxEnabled" value="${redis.jmxEnabled}"/>
    </bean>

    <bean id="connectionFactory"
          class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
          p:host-name="${redis.host}" p:port="${redis.port}" p:password="${redis.pass}"
          p:database="${redis.db}" p:pool-config-ref="poolConfig"/>

    <!-- org.springframework.data.redis.core.StringRedisTemplate -->
    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="connectionFactory"/>
        <property name="keySerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="valueSerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="hashKeySerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="hashValueSerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
    </bean>

    <bean id="cacheconnectionFactory"
          class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
          p:host-name="${cacheredis.host}" p:port="${cacheredis.port}" p:password="${cacheredis.pass}"
          p:database="${cacheredis.db}" p:pool-config-ref="cachepoolConfig"/>

    <bean id="cacheTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="cacheconnectionFactory"/>
        <property name="keySerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="valueSerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="hashKeySerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="hashValueSerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
    </bean>

    <bean id="statisConnectionFactory"
          class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
          p:host-name="${statisredis.host}" p:port="${statisredis.port}"
          p:password="${statisredis.pass}"
          p:database="${statisredis.db}" p:pool-config-ref="statispoolConfig">
    </bean>

    <bean id="statisRedisTemplate"
          class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="statisConnectionFactory"/>
        <property name="keySerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="valueSerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="hashKeySerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="hashValueSerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
    </bean>

    <bean id="submitStatusReplyConnectionFactory"
          class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory"
          p:host-name="${submitstatusreplyredis.host}" p:port="${submitstatusreplyredis.port}"
          p:password="${submitstatusreplyredis.pass}"
          p:database="${submitstatusreplyredis.db}" p:pool-config-ref="submitpoolConfig">
    </bean>

    <bean id="submitStatusReplyRedisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="submitStatusReplyConnectionFactory"/>
        <property name="keySerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="valueSerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="hashKeySerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
        <property name="hashValueSerializer">
            <ref bean="stringRedisSerializer"/>
        </property>
    </bean>

    <bean id="stringRedisSerializer"
          class="org.springframework.data.redis.serializer.StringRedisSerializer"/>

</beans>


```

### web.xml样例

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
		http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         id="WebApp_ID" version="3.0">

    <display-name>learning</display-name>
    <absolute-ordering/>
    <context-param>
        <param-name>webAppRootKey</param-name>
        <param-value>learning.root</param-value>
    </context-param>

    <context-param>
        <param-name>logbackConfigLocation</param-name>
        <param-value>classpath:logback.xml</param-value>
    </context-param>

    <listener>
        <listener-class>ch.qos.logback.ext.spring.web.LogbackConfigListener</listener-class>
    </listener>

        <listener-class>org.springframework.web.util.IntrospectorCleanupListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            classpath:applicationContext.xml
        </param-value>
    </context-param>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <listener>
        <listener-class>
            org.springframework.web.context.request.RequestContextListener
        </listener-class>
    </listener>

    <servlet>
        <servlet-name>springDispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>springDispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

    <filter>
        <filter-name>DruidWebStatFilter</filter-name>
        <filter-class>com.alibaba.druid.support.http.WebStatFilter</filter-class>
        <init-param>
            <param-name>exclusions</param-name>
            <param-value>*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>DruidWebStatFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <servlet>
        <servlet-name>DruidStatView</servlet-name>
        <servlet-class>com.alibaba.druid.support.http.StatViewServlet</servlet-class>
        <init-param>
            <param-name>resetEnable</param-name>
            <param-value>true</param-value>
        </init-param>
        <init-param>
            <param-name>allow</param-name>
            <param-value>127.0.0.1</param-value>
        </init-param>
        <init-param>
            <param-name>loginUsername</param-name>
            <param-value>druid</param-value>
        </init-param>
        <init-param>
            <param-name>loginPassword</param-name>
            <param-value>druid</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>DruidStatView</servlet-name>
        <url-pattern>/druid/*</url-pattern>
    </servlet-mapping>

    <filter>
        <filter-name>characterEncodingFilter</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
        <init-param>
            <param-name>forceEncoding</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
    <filter-mapping>
        <filter-name>characterEncodingFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>

    <filter>
        <filter-name>permission</filter-name>
        <filter-class>com.learning.filter.AccessControlFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>permission</filter-name>
        <url-pattern>*.do</url-pattern>
    </filter-mapping>

    <session-config>
        <session-timeout>60</session-timeout>
    </session-config>

    <error-page>
        <error-code>400</error-code>
        <location>/WEB-INF/pages/error/400.jsp</location>
    </error-page>
    <error-page>
        <error-code>403</error-code>
        <location>/WEB-INF/pages/error/403.jsp</location>
    </error-page>
    <error-page>
        <error-code>404</error-code>
        <location>/WEB-INF/pages/error/404.jsp</location>
    </error-page>
    <error-page>
        <error-code>500</error-code>
        <location>/WEB-INF/pages/error/error.jsp</location>
    </error-page>
    <error-page>
        <error-code>502</error-code>
        <location>/WEB-INF/pages/error/error.jsp</location>
    </error-page>
    <error-page>
        <error-code>503</error-code>
        <location>/WEB-INF/pages/error/error.jsp</location>
    </error-page>
    <error-page>
        <error-code>504</error-code>
        <location>/WEB-INF/pages/error/error.jsp</location>
    </error-page>
</web-app>
```

## 二、去除web.xml

- web.xml是系统启动资源加载的依据
- 加载的顺序是：web.xml->context-param->listener -> filter -> servlet ，其中context-param用于向 ServletContext 提供键值对，即应用程序上下文信息，listener, filter 等在初始化时会用到这些上下文中的信息，然而对于某些配置而言，它们出现的顺序是有先后关联的。
- 首先加载Spring上下文环境配置文件，然后加载SpringMVC配置文件，并且如果配置了相同的内容，SpringMVC配置文件会被优先使用。为了避免这种情况，我们Spring上下文加载的时候扫描全部文件，mvc加载的时候仅扫描controller目录下文件，避免覆盖混乱。

- 从Servlet 3.0开始，除了在web.xml文件中进行声明式配置外，还可以通过实现或扩展Spring提供的这三个支持类之一来以编程方式配置(WebAppInitializer、AbstractDispatcherServletInitializer、AbstractAnnotationConfigDispatcherServletInitializer其一均可)

- WebApplicationInitializer确保SpringServletContainerInitializer （本身会自动引导）检测到ApplicationInitializer类，并将其用于初始化任何Servlet 3容器

```java
public class ApplicationInitializer implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext container) throws ServletException {
        AnnotationConfigWebApplicationContext rootContext = new AnnotationConfigWebApplicationContext();
        rootContext.register(ContextConfig.class);
        rootContext.setDisplayName("learning");
        container.setAttribute("webAppRootKey", "learning.root");
        container.setAttribute("logbackConfigLocation", "classpath:logback.xml");

        container.addListener(new LogbackConfigInitListener());
        container.addListener(new IntrospectorCleanupListener());
        container.addListener(new ContextLoaderListener(rootContext));
        container.addListener(new RequestContextListener());

        AnnotationConfigWebApplicationContext dispatcherContext = new AnnotationConfigWebApplicationContext();
        dispatcherContext.register(WebMvcConfig.class);

        ServletRegistration.Dynamic springDispatcherServlet = container.addServlet("springDispatcherServlet", new DispatcherServlet(dispatcherContext));
        springDispatcherServlet.setLoadOnStartup(1);
        springDispatcherServlet.addMapping("/");

        ServletRegistration.Dynamic statViewServlet = container.addServlet("DruidStatView", new StatViewServlet());
        statViewServlet.setInitParameter("resetEnable", "true");
        statViewServlet.setInitParameter("allow", "127.0.0.1");
        statViewServlet.setInitParameter("loginUsername", "druid");
        statViewServlet.setInitParameter("loginPassword", "druid");
        statViewServlet.addMapping("/druid/*");

        FilterRegistration filterRegistrationDruid = container.addFilter("DruidWebStatFilter", new WebStatFilter());
        filterRegistrationDruid.setInitParameter("exclusions", "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        filterRegistrationDruid.addMappingForUrlPatterns(null, true, "/*");

        FilterRegistration.Dynamic filterRegistrationCharacter = container.addFilter("characterEncodingFilter", new CharacterEncodingFilter());
        filterRegistrationCharacter.setInitParameter("encoding", "UTF-8");
        filterRegistrationCharacter.setInitParameter("forceEncoding", "true");
        filterRegistrationCharacter.addMappingForUrlPatterns(null, false, "/*");

        FilterRegistration.Dynamic filterRegistrationXss = container.addFilter("xssRequestFilter", new XssFilter());
        filterRegistrationXss.addMappingForUrlPatterns(null, false, "/*");

        FilterRegistration.Dynamic filterRegistrationPermission = container.addFilter("permission", new AccessControlFilter());
        filterRegistrationPermission.addMappingForUrlPatterns(null, false, "*.do");
    }
}

```

- 参考web.xml中的配置，将context-param、listener、filter、servlet这几项进行定义，其中ContextConfig、WebMvcConfig是Spring托管的对象，用于加载Spring上下文和mvc上下文

- AbstractDispatcherServletInitializer是WebApplicationInitializer的基类，实现方式类似
- AbstractAnnotationConfigDispatcherServletInitializer 十分简洁，不必手动配置DispatcherServlet和/或ContextLoaderListener，十分方便

```
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
 
   @Override
   protected Class<?>[] getRootConfigClasses() {
      return new Class[] { RootConfig.class };
   }
 
   @Override
   protected Class<?>[] getServletConfigClasses() {
      return new Class[] { WebMvcConfig.class };
   }
 
   @Override
   protected String[] getServletMappings() {
      return new String[] { "/" };
   }
}
```

## 三、去除applicationContext.xml

- 配置包扫描、开启切面，注入Spring上下对象

```java
@Configuration
@ComponentScan(basePackages = {"com.learning"},
        excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Controller.class),
                @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = ControllerAdvice.class)
        })
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class ContextConfig {

    @Bean
    public AppCtxUtil getAppCtxUtil() {
        return new AppCtxUtil(new ConcurrentHashMap<>());
    }
}
```

## 四、去除springmvc.xml

- 开启包扫描，开启注解驱动

```java
@Configuration
@ComponentScan(basePackages = {"com.learning.controller"},
        useDefaultFilters = false,
        includeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION,classes = Controller.class),
        @ComponentScan.Filter(type = FilterType.ANNOTATION,classes = ControllerAdvice.class)})
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private StorageProperties storageProperties;
    
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("redirect:login.do?method=toLogin");
    }

    @Bean
    public InternalResourceViewResolver getViewResolver(){
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setViewClass(org.springframework.web.servlet.view.JstlView.class);
        viewResolver.setPrefix("/WEB-INF/pages/");
        viewResolver.setSuffix(".jsp");
        return  viewResolver;
    }

    @Bean("multipartResolver")
    public CommonsMultipartResolver getMultipartResolver(){
        CommonsMultipartResolver multipartResolver = new CommonsMultipartResolver();
        multipartResolver.setDefaultEncoding("utf-8");
        //20M
        multipartResolver.setMaxUploadSize(storageProperties.getMaxUploadSize());
        multipartResolver.setResolveLazily(true);
        multipartResolver.setMaxInMemorySize(storageProperties.getMaxInMemorySize());
        return multipartResolver;
    }
    
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {

        registry.addResourceHandler("/images/**")
                .addResourceLocations("/images/");

        registry.addResourceHandler("/js/**")
                .addResourceLocations("/js/");

        registry.addResourceHandler("/css/**")
                .addResourceLocations("/css/");

    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        ReferrerInterceptor interceptor = new ReferrerInterceptor();
        registry.addInterceptor(interceptor).addPathPatterns("/**");
    }

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}

```

## 五、去除数据库xml配置

- 开启JPA扫描、开启事务管理 

```java
@Component
@EnableJpaRepositories(basePackages = "com.learning.repository",entityManagerFactoryRef = "entityManagerFactory",transactionManagerRef = "transactionManager")
@EnableTransactionManagement
public class DatabaseConfiguration {
    @Autowired
    private DatabaseProperties databaseProperties;
    @Autowired
    private HibernateProperties hibernateProperties;

    @Bean("statFliter")
    public StatFilter getStatFilter(){
        return new StatFilter();
    }

    @Bean("logFilter")
    public Slf4jLogFilter getLogFilter(){
        Slf4jLogFilter logFilter = new Slf4jLogFilter();
        logFilter.setConnectionLogEnabled(Boolean.TRUE);
        logFilter.setStatementLogEnabled(Boolean.TRUE);
        logFilter.setResultSetLogEnabled(Boolean.TRUE);
        return logFilter;
    }

    @Bean(name = "dataSource",initMethod = "init",destroyMethod = "close")
    public DruidDataSource getDataSource(@Qualifier("statFliter")StatFilter statFilter,@Qualifier("logFilter")Slf4jLogFilter logFilter){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(databaseProperties.getDriverClass());
        dataSource.setUrl(databaseProperties.getJdbcUrl());
        dataSource.setUsername(databaseProperties.getUser());
        dataSource.setPassword(databaseProperties.getPassword());
        dataSource.setInitialSize(databaseProperties.getMinPoolSize());
        dataSource.setMinIdle(databaseProperties.getMinPoolSize());
        dataSource.setMaxActive(databaseProperties.getMaxPoolSize());
        dataSource.setMaxWait(databaseProperties.getMaxWait());
        dataSource.setRemoveAbandoned(databaseProperties.getRemoveAbandoned());
        dataSource.setRemoveAbandonedTimeout(databaseProperties.getRemoveAbandonedTimeout());
        dataSource.setLogAbandoned(databaseProperties.getLogAbandoned());
        dataSource.setTimeBetweenEvictionRunsMillis(databaseProperties.getTimeBetweenEvictionRunsMillis());
        dataSource.setMinEvictableIdleTimeMillis(databaseProperties.getMinEvictableIdleTimeMillis());
        dataSource.setValidationQuery(databaseProperties.getValidationQuery());
        dataSource.setTestWhileIdle(databaseProperties.getTestWhileIdle());
        dataSource.setTestOnBorrow(databaseProperties.getTestOnBorrow());
        dataSource.setTestOnReturn(databaseProperties.getTestOnReturn());
        dataSource.setPoolPreparedStatements(databaseProperties.getPoolPreparedStatements());
        dataSource.setMaxPoolPreparedStatementPerConnectionSize(databaseProperties.getMaxPoolPreparedStatementPerConnectionSize());
        dataSource.setProxyFilters(Arrays.asList(statFilter,logFilter));
        return dataSource;
    }

    @Bean(name = "jpaVendorAdapter")
    public HibernateJpaVendorAdapter getHibernateJpaVendorAdapter(){
        return  new HibernateJpaVendorAdapter();
    }
    @Bean(name = "entityManagerFactory")
    public LocalContainerEntityManagerFactoryBean getEntityManagerFactoryBean(@Qualifier("dataSource")DruidDataSource dataSource,
                                                                              @Qualifier("jpaVendorAdapter")HibernateJpaVendorAdapter jpaVendorAdapter){
        LocalContainerEntityManagerFactoryBean entityManagerFactoryBean = new LocalContainerEntityManagerFactoryBean();
        entityManagerFactoryBean.setPackagesToScan("com.learning.model");
        entityManagerFactoryBean.setDataSource(dataSource);
        entityManagerFactoryBean.setJpaVendorAdapter(jpaVendorAdapter);
        entityManagerFactoryBean.setJpaProperties(hibernateProperties.getProperties());
          return entityManagerFactoryBean;
    }

    @Bean(name = "transactionManager")
    public JpaTransactionManager getJpaTransactionManager(@Qualifier("entityManagerFactory")LocalContainerEntityManagerFactoryBean entityManagerFactoryBean){
        JpaTransactionManager transactionManager = new JpaTransactionManager();
        transactionManager.setEntityManagerFactory(entityManagerFactoryBean.getObject());
        return transactionManager;
    }

    @Bean(name = "jdbcTemplate")
    public JdbcTemplate getJdbcTemplate(@Qualifier("dataSource")DruidDataSource dataSource){
        return new JdbcTemplate(dataSource);
    }
}
```

## 六、去除redis-context.xml配置

- 配置缓存

```java
@Component
public class CacheConfiguration {
    @Autowired
    private RedisProperties redisProperties;

    @Bean(name = "stringRedisSerializer")
    public StringRedisSerializer getStringRedisSerializer() {
        return new StringRedisSerializer();
    }

    @Bean(name = "poolConfig")
    public JedisPoolConfig getPoolConfig() {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxIdle(redisProperties.getBaseMaxIdle());
        poolConfig.setMinIdle(redisProperties.getBaseMinIdle());
        poolConfig.setMaxTotal(redisProperties.getBaseMaxTotal());
        poolConfig.setMaxWaitMillis(redisProperties.getMaxWaitMillis());
        poolConfig.setTestOnBorrow(redisProperties.getTestOnBorrow());
        poolConfig.setTestOnReturn(redisProperties.getTestOnReturn());
        poolConfig.setTestWhileIdle(redisProperties.getTestWhileIdle());
        poolConfig.setTimeBetweenEvictionRunsMillis(redisProperties.getTimeBetweenEvictionRunsMillis());
        poolConfig.setMinEvictableIdleTimeMillis(redisProperties.getMinEvictableIdleTimeMillis());
        poolConfig.setJmxEnabled(redisProperties.getJmxEnabled());
        return poolConfig;
    }

    @Bean(name = "connectionFactory")
    public JedisConnectionFactory getConnectionFactory(@Qualifier("poolConfig") JedisPoolConfig poolConfig) {
        RedisStandaloneConfiguration configuration = new RedisStandaloneConfiguration(redisProperties.getBaseHost(), redisProperties.getBasePort());
        configuration.setPassword(redisProperties.getBasePass());
        configuration.setDatabase(redisProperties.getBaseDb());
        JedisClientConfiguration clientConfiguration = JedisClientConfiguration.builder().usePooling().poolConfig(poolConfig).build();
        return new JedisConnectionFactory(configuration, clientConfiguration);
    }


    @Bean(name = "redisTemplate")
    public RedisTemplate<String, Object> getRedsiTemplate(@Qualifier("connectionFactory") JedisConnectionFactory connectionFactory, @Qualifier("stringRedisSerializer") StringRedisSerializer stringRedisSerializer) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(connectionFactory);
        redisTemplate.setKeySerializer(stringRedisSerializer);
        redisTemplate.setValueSerializer(stringRedisSerializer);
        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        redisTemplate.setHashValueSerializer(stringRedisSerializer);
        return redisTemplate;
    }
}
```

## 七、测试方法配置

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {ContextConfig.class,WebMvcConfig.class})
@WebAppConfiguration("src/main/webapp")
public class BaseSpringJunitTest {

}
```

## Refer

1.[spring-dispatcherservlet-tutorial](https://rumenz.com/java-topic/spring5/webmvc/spring-dispatcherservlet-tutorial/index.html)

2.[spring5-mvc-hibernate5-example](https://rumenz.com/java-topic/spring5/webmvc/spring5-mvc-hibernate5-example/index.html)
