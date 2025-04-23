---
title: 《spring》容器启动 -step3- 配置BeanFactory
date: 2021-01-08
categories:
   - [spring, 容器]
tags:
   - [容器]
---

    这是spring系列的第三篇文章，主要介绍的是容器启动中的第3个步骤，prepareBeanFactory。

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
   color: #6260ff;
}
</style>

# 一、Spring
`spring框架`是Java生态中最主流的轻量级开源应用框架，其核心目标是简化企业级应用开发，通过`IOC（控制反转）`和`AOP（面向切面编程）`两大核心机制实现解耦、模块化和可维护性。

<!-- more -->

# 二、prepareBeanFactory(beanFactory);
主要职责是为BeanFactory设置必要的上下文环境，包括类加载器、表达式解析器、后置处理器等，确保后续 Bean 的正确实例化和初始化。
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置类加载器（ClassLoader）
    beanFactory.setBeanClassLoader(getClassLoader());
    //配置表达式解析器（BeanExpressionResolver）
    //支持SpELl表达式解析，用于@Value注解、XML配置中的#{...}表达式    
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    //注册属性编辑器（PropertyEditorRegistrar)
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 添加ApplicationContextAwareProcessor，
    //作用：处理实现了Aware接口（如ApplicationContextAware、EnvironmentAware）的Bean
    //流程：在postProcessBeforeInitialization阶段调用 invokeAwareInterfaces()，注入 ApplicationContext等对象
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 注册ApplicationListenerDetector
    // 作用：检测实现了 ApplicationListener 接口的Bean，并将其注册到上下文的事件广播器中。
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 在工厂中提前注册一些单例Bean
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

## 2.1、核心步骤解析
1. **设置类加载器（ClassLoader）**
   ```java
   beanFactory.setBeanClassLoader(getClassLoader());
   ```
  - **作用**：指定BeanFactory加载Bean类时使用的类加载器，通常为当前应用上下文的类加载器。
  - **必要性**：确保Bean类能够正确加载，特别是在模块化环境（如OSGi）或自定义类加载场景中。

2. **配置表达式解析器（BeanExpressionResolver）**
   ```java
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   ```
  - **作用**：支持SpEL（Spring Expression Language）表达式的解析，用于`@Value`注解、XML配置中的`#{...}`表达式等。
  - **示例**：在Bean属性中注入动态值：`<property name="timeout" value="#{systemProperties['default.timeout']}"/>`

3. **注册属性编辑器（PropertyEditorRegistrar）**
   ```java
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
   ```
  - **作用**：注册自定义`PropertyEditor`，将字符串值转换为复杂类型（如`Resource`、`Environment`）。
  - **示例**：XML中的`classpath:config.xml`字符串被转换为`ClassPathResource`对象。

4. **添加ApplicationContextAwareProcessor**
   ```java
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   ```
  - **作用**：处理实现了`Aware`接口（如`ApplicationContextAware`、`EnvironmentAware`）的Bean，在初始化阶段注入依赖。
  - **流程**：在`postProcessBeforeInitialization`阶段调用`invokeAwareInterfaces()`，注入`ApplicationContext`等对象。

5. **忽略自动装配的接口**
   ```java
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   // ... 其他如ResourceLoaderAware, ApplicationEventPublisherAware等
   ```
  - **原因**：避免这些接口被自动装配机制处理，因为它们已由`ApplicationContextAwareProcessor`手动注入。

6. **注册可解析依赖（Resolvable Dependencies）**
   ```java
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);
   ```
  - **作用**：当Bean需要注入`BeanFactory`、`ApplicationContext`等依赖时，直接返回预设实例，无需查找。

7. **注册ApplicationListenerDetector**
   ```java
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
   ```
  - **职责**：检测实现了`ApplicationListener`接口的Bean，并将其注册到上下文的事件广播器中。
  - **注意**：确保监听器在上下文事件发布时能够接收通知。









