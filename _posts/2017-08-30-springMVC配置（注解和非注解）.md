---
layout: post
title: "spring配置文件详解"
date: 2017-08-01
excerpt: "各种乱七八糟的配置是我觉得spring入门最难的地方"
tags: [Java, spring]
comments: true
---

 &nbsp;  &nbsp; 一般springMVC项目会引入两个配置文件，分别是：xxx-servlet.xml和ApplicationContext.xml，其中xxx-servlet.xml是springMVC的配置文件，一般在其中定义controller，用于拦截url转发views，ApplicationContext.xml是spring全局的配置文件，用来控制spring特性，比如aop等，如果只有一个servlet，则不需要配置，如果要使用，需要在web.xml中声明。而用Java配置类取代xml配置可以使项目变得更精简，非常适合小型应用。

## web.xml

```xml
<!-- 非注解 -->
<!-- Spring MVC配置 -->
<!-- ====================================== -->
<servlet>
    <servlet-name>spring</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!-- 可以自定义servlet.xml配置文件的位置和名称，默认为WEB-INF目录下，名称为[<servlet-name>]-servlet.xml，如spring-servlet.xml -->
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>spring</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>


<!-- Spring配置，非必要 -->
<!-- ====================================== -->
<listener>
   <listenerclass>
     org.springframework.web.context.ContextLoaderListener
   </listener-class>
</listener>
```



## xxx-servlet.xml

 &nbsp;  &nbsp; xxx-servlet.xml是controller级别的上下文，不涉及除了转发之外的任何实体，所以它的作用范围仅仅限制正在servlet级别。

 &nbsp;  &nbsp; servlet配置文件里应该只有视图的解析方式、静态资源文件的存放位置、controller的初始化方式。它只负责请求的转发。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"     
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"     
        xmlns:context="http://www.springframework.org/schema/context"     
   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd   
       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd   
       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-3.0.xsd   
       http://www.springframework.org/schema/context <a href="http://www.springframework.org/schema/context/spring-context-3.0.xsd">http://www.springframework.org/schema/context/spring-context-3.0.xsd</a>">

    <!-- 启用spring mvc 注解 -->
    <context:annotation-config />

    <!-- 设置使用注解的类所在的jar包 -->
    <context:component-scan base-package="controller"></context:component-scan>

    <!-- 完成请求和注解POJO的映射 -->
    <bean class="org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter" />
　　
    <!-- 对转向页面的路径解析。prefix：前缀， suffix：后缀 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" p:prefix="/jsp/" p:suffix=".jsp" />
</beans>
```

## AbstractAnnotationConfigDispatcherServletInitializer的实现取代web.xml

```java
package config;

import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;

public class SpittrWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected String[] getServletMappings()
    {
        return new String[] {"/"};
    }

    @Override
    protected Class<?>[] getRootConfigClasses()
    {
        return new Class<?>[] {RootConfig.class};
    }

    @Override
    protected Class<?>[] getServletConfigClasses()
    {
        return new Class<?>[] {WebConfig.class};
    }
}
```



## Java配置类webconfig取代xxx-servlet.xml

```java
package config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.ViewResolver;
import org.springframework.web.servlet.config.annotation.DefaultServletHandlerConfigurer;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;
import org.springframework.web.servlet.view.InternalResourceViewResolver;

@Configuration
@EnableWebMvc
@ComponentScan("要扫描的包")
public class WebConfig extends WebMvcConfigurerAdapter{

    @Bean
    public ViewResolver viewResolver()
    {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setExposeContextBeansAsAttributes(true);
        return  resolver;
    }

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer)
    {
        configurer.enable();
    }
}
```

@EnableWebMvc 等同于 mvc:annotation-driven 在XML中. 它能够为使用@RequestMapping向特定的方法传入的请求映射@Controller-annotated 类。

@ComponentScan 等同于 context:component-scan base-package= 提供 spring 在哪里寻找 管理 beans/classes.

## ApplicationContext.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:context="http://www.springframework.org/schema/context" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"  
    xmlns:tx="http://www.springframework.org/schema/tx" xmlns:p="http://www.springframework.org/schema/p" xmlns:util="http://www.springframework.org/schema/util" xmlns:jdbc="http://www.springframework.org/schema/jdbc"  
    xmlns:cache="http://www.springframework.org/schema/cache"  
    xsi:schemaLocation="  
    http://www.springframework.org/schema/context  
    http://www.springframework.org/schema/context/spring-context.xsd  
    http://www.springframework.org/schema/beans  
    http://www.springframework.org/schema/beans/spring-beans.xsd  
    http://www.springframework.org/schema/tx  
    http://www.springframework.org/schema/tx/spring-tx.xsd  
    http://www.springframework.org/schema/jdbc  
    http://www.springframework.org/schema/jdbc/spring-jdbc-3.1.xsd  
    http://www.springframework.org/schema/cache  
    http://www.springframework.org/schema/cache/spring-cache-3.1.xsd  
    http://www.springframework.org/schema/aop  
    http://www.springframework.org/schema/aop/spring-aop.xsd  
    http://www.springframework.org/schema/util  
    http://www.springframework.org/schema/util/spring-util.xsd">  

    <!-- 自动扫描web包 ,将带有注解的类 纳入spring容器管理 -->  
    <context:component-scan base-package="com.eduoinfo.finances.bank.web"></context:component-scan>  

    <!-- 引入jdbc配置文件 -->  
    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">  
        <property name="locations">  
            <list>  
                <value>classpath*:jdbc.properties</value>  
            </list>  
        </property>  
    </bean>  

    <!-- dataSource 配置 -->  
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">  
        <!-- 基本属性 url、user、password -->  
        <property name="url" value="${jdbc.url}" />  
        <property name="username" value="${jdbc.username}" />  
        <property name="password" value="${jdbc.password}" />  

        <!-- 配置初始化大小、最小、最大 -->  
        <property name="initialSize" value="1" />  
        <property name="minIdle" value="1" />  
        <property name="maxActive" value="20" />  

        <!-- 配置获取连接等待超时的时间 -->  
        <property name="maxWait" value="60000" />  

        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->  
        <property name="timeBetweenEvictionRunsMillis" value="60000" />  

        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->  
        <property name="minEvictableIdleTimeMillis" value="300000" />  

        <property name="validationQuery" value="SELECT 'x'" />  
        <property name="testWhileIdle" value="true" />  
        <property name="testOnBorrow" value="false" />  
        <property name="testOnReturn" value="false" />  

        <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->  
        <property name="poolPreparedStatements" value="false" />  
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20" />  

        <!-- 配置监控统计拦截的filters -->  
        <property name="filters" value="stat" />  
    </bean>  

    <!-- mybatis文件配置，扫描所有mapper文件 -->  
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean" p:dataSource-ref="dataSource" p:configLocation="classpath:mybatis-config.xml" p:mapperLocations="classpath:com/eduoinfo/finances/bank/web/dao/*.xml" />  

    <!-- spring与mybatis整合配置，扫描所有dao -->  
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer" p:basePackage="com.eduoinfo.finances.bank.web.dao" p:sqlSessionFactoryBeanName="sqlSessionFactory" />  

    <!-- 对dataSource 数据源进行事务管理 -->  
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager" p:dataSource-ref="dataSource" />  

    <!-- 配置使Spring采用CGLIB代理 -->  
    <aop:aspectj-autoproxy proxy-target-class="true" />  

    <!-- 启用对事务注解的支持 -->  
    <tx:annotation-driven transaction-manager="transactionManager" />  

    <!-- Cache配置 -->  
    <cache:annotation-driven cache-manager="cacheManager" />  
    <bean id="ehCacheManagerFactory" class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean" p:configLocation="classpath:ehcache.xml" />  
    <bean id="cacheManager" class="org.springframework.cache.ehcache.EhCacheCacheManager" p:cacheManager-ref="ehCacheManagerFactory" />  

</beans>  
```



## Java配置类Rootconfig取代ApplicationContext.xml

```java
package config;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.FilterType;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;

@Configuration
@ComponentScan(basePackages = {"spittr"}, excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, value = EnableWebMvc.class)})
public class RootConfig {

}
```
