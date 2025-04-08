---
title: 《springmvc》启动过程
date: 2021-05-10
categories:
  - [spring, springmvc]
---

    这是springmvc系列的第1篇文章，主要介绍的是springmvc的启动过程。

<style>
.my-code {
   color: orange;
}
.orange {
   color: rgb(255, 53, 2)
}
.red {
   color: red
}
code {
   color: #0ABF5B;
}
</style>

# 一、什么是springMVC？
springmvc是spring框架中用于构建web应用程序的核心模块，它基于`MVC（model-view-controller）`设计模式，提供了一套灵活、高效的请求处理机制。
> MVC是一种设计模式，这种设计模式建议将一个请求由`M（Module）、V（View）、C（controller）`三个部分进行处理。请求先经过`controller`，controller调用其他服务层得到`Module`，最后将Module数据渲染成试图（View）返回客户端。Spring MVC是Spring生态圈的一个组件，一个遵守MVC设计模式的WEB MVC框架。这个框架可以和Spring无缝整合，上手简单，易于扩展。

<!--more-->

# 二、配置

## 2.1、添加依赖
在项目的pom.xml文件中添加springMVC及相关依赖
```java
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.3.22</version>
    </dependency>
    <!-- 其他依赖，如Spring Core、日志框架等 -->
</dependencies>
```

## 2.2、配置web.xml
在`web.xml`中配置前端控制器`DispatcherServlet`
```java
<servlet>
    <servlet-name>springMVC</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring-mvc.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>springMVC</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

## 2.3、创建springMVC配置文件
创建`spring-mvc.xml`文件，配置组件扫描、视图解析器等。
```java
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <context:component-scan base-package="com.example.controller" />

    <mvc:annotation-driven />

    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/" />
        <property name="suffix" value=".jsp" />
    </bean>
</beans>
```


# 三、启动过程

## 3.1、启动Tomcat
Tomcat启动，解析`web.xml`文件



