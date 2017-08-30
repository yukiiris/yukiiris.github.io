---
layout: post
title: "springMVC配置（注解和非注解）"
date: 2017-08-01
excerpt: "这是我觉得spring入门最难的地方"
tags: [Java, spring]
comments: true
---

## web.xml

&nbsp; &nbsp;  &nbsp; DispatcherServlet是springMVC应用的入口，每一个请求会先经过它，再由它转发给相应的controller。

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
        <param-value><!-- servlet.xml文件位置，默认/WEB-INF/remoting-servlet.xml --></param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>spring</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>
  

<!-- Spring配置 -->
<!-- ====================================== -->
<listener>
   <listenerclass>
     org.springframework.web.context.ContextLoaderListener
   </listener-class>
</listener>
```



## spring-servlet.xml

 &nbsp;  &nbsp; spring-servlet.xml是controller级别的上下文，不涉及除了转发之外的任何实体，所以它的作用范围仅仅限制正在servlet级别。

 &nbsp;  &nbsp; servlet配置文件里应该只有视图的解析方式、静态资源文件的存放位置、controller的初始化方式。它只负责请求的转发。

