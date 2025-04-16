---
title: 《dubbo》provider启动
date: 2021-06-05
categories:
  - [dubbo]
---

    这是dubbo系列的第2篇文章，主要介绍的是dubbo的provider启动流程。

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

# 一、dubbo
Dubbo 是一款微服务开发框架，它提供了 RPC通信 与 微服务治理 两大关键能力。这意味着，使用 Dubbo 开发的微服务，将具备相互之间的远程发现与通信能力， 同时利用 Dubbo 提供的丰富服务治理能力，可以实现诸如服务发现、负载均衡、流量调度等服务治理诉求。同时 Dubbo 是高度可扩展的，用户几乎可以在任意功能点去定制自己的实现，以改变框架的默认行为来满足自己的业务需求。

<!--more-->

Dubbo主要提供了`3大核心功能`：面向接口的远程方法调用，智能容错和负载均衡，以及服务自动注册和发现。
1. **远程方法调用**
网络通信框架，提供对多种NIO框架抽象封装，包括“同步转异步”和“请求-响应”模式的信息交换方式。

2. **智能容错和负载均衡**
提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持。

3. **服务注册和发现**
服务注册，基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器。


# 二、示例代码
**启动类：**
```java
package com.alibaba.dubbo.demo.provider;
import org.springframework.context.support.ClassPathXmlApplicationContext;
public class Provider {
    public static void main(String[] args) throws Exception {
        //Prevent to get IPV6 address,this way only work in debug mode
        //But you can pass use -Djava.net.preferIPv4Stack=true,then it work well whether in debug mode or not
        System.setProperty("java.net.preferIPv4Stack", "true");
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-provider.xml"});
        context.start();
        System.in.read(); // press any key to exit
    }
}
```
**自定义service：**
```java
package com.alibaba.dubbo.demo;
//接口定义：
public interface DemoService {
    String sayHello(String name);
}

//服务实现
public class DemoServiceImpl implements DemoService {
    @Override
    public String sayHello(String name) {
        System.out.println("[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "] Hello " + name + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return "Hello " + name + ", response from provider: " + RpcContext.getContext().getLocalAddress();
    }
}
```
**配置文件：**
```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <!-- provider's application name, used for tracing dependency relationship -->
    <dubbo:application name="demo-provider"/>
    <!-- use multicast registry center to export service -->
    <dubbo:registry address="multicast://224.5.6.7:1234"/>
    <!-- use dubbo protocol to export service on port 20880 -->
    <dubbo:protocol name="dubbo" port="20880"/>
    <!-- service implementation, as same as regular local bean -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>
    <!-- declare the service interface to be exported -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
</beans>
```

# 三、provider启动流程
`provider`启动流程包括
- **配置解析**
- **服务注册**
- **协议暴露**
- **网络监听**


## 3.1、配置解析与初始化
示例代码中，通过XML配置引入Dubbo。Dubbo的命名空间，相关的handler如下：
```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```
在`init()`方法中，注册了多个`BeanDefinitionParser`，每个解析器负责处理特定的`Dubbo XML`标签。

### 3.1.1、标签解析器
| XML 标签          | 解析器类                         | 对应配置类/Bean 类          | 关键参数说明          |
|-------------------|----------------------------------|----------------------------|----------------------|
| `application`     | `DubboBeanDefinitionParser`      | `ApplicationConfig`        | 应用全局配置（如应用名） |
| `module`          | `DubboBeanDefinitionParser`      | `ModuleConfig`             | 模块配置（已逐渐废弃）   |
| `registry`        | `DubboBeanDefinitionParser`      | `RegistryConfig`           | 注册中心配置（如 Zookeeper 地址） |
| `monitor`         | `DubboBeanDefinitionParser`      | `MonitorConfig`            | 监控中心配置          |
| `provider`        | `DubboBeanDefinitionParser`      | `ProviderConfig`           | 服务提供者默认配置    |
| `consumer`        | `DubboBeanDefinitionParser`      | `ConsumerConfig`           | 服务消费者默认配置    |
| `protocol`        | `DubboBeanDefinitionParser`      | `ProtocolConfig`           | 协议配置（如 Dubbo、HTTP） |
| `service`         | `DubboBeanDefinitionParser`      | `ServiceBean`              | 服务暴露配置          |
| `reference`       | `DubboBeanDefinitionParser`      | `ReferenceBean`            | 服务引用配置          |
| `annotation`      | `AnnotationBeanDefinitionParser` | -                          | 注解驱动配置（如包扫描） |

特殊解析器：`AnnotationBeanDefinitionParser`
- **功能**：处理 `<dubbo:annotation>` 标签，启用 `Dubbo` 注解扫描。
- **作用**：扫描 `@DubboService`（服务提供者）和 `@DubboReference`（服务消费者）注解，自动注册 Bean。
- **配置示例**：
  ```xml
  <dubbo:annotation package="com.example.service" />
  ```


## 3.2、服务注册

### 3.2.1、核心类解析`ServiceBean`
- **作用**：服务提供者的配置入口，继承 `ServiceConfig`。
- **生命周期**：在 Spring 容器初始化完成后，调用 `export()` 方法暴露服务。
- **关键属性**：
    - `interface`：服务接口类。
    - `ref`：服务实现 Bean 的引用。
    - `registry`：关联的注册中心配置。

`ServiceBean`实现了`InitializingBean`接口，在Bean的初始化时，执行其`afterPropertiesSet()`方法：
```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, ApplicationContextAware, BeanNameAware {
  // ...
  @Override
  public void afterPropertiesSet() throws Exception {
    // 1. 检查必要配置是否缺失
    if (getProvider() == null) {
      // 尝试从 Spring 容器中获取全局 ProviderConfig
      setProvider(ApplicationModel.getConfigManager().getDefaultProvider().orElse(null));
    }
    if (getApplication() == null) {
      // 获取全局 ApplicationConfig
      setApplication(ApplicationModel.getConfigManager().getDefaultApplication().orElse(null));
    }

    // 2. 填充未显式配置的默认值（如协议、注册中心）
    if (StringUtils.isEmpty(getProtocolIds())) {
      // 使用默认协议配置（如 DubboProtocol）
      List<ProtocolConfig> protocols = ApplicationModel.getConfigManager().getDefaultProtocols();
      if (!protocols.isEmpty()) {
        setProtocols(protocols);
      }
    }

    // 3. 触发服务暴露（根据 delay 配置决定是否延迟）
    if (shouldExport()) {
      export();
    }
  }

  private boolean shouldExport() {
    Boolean export = getExport();
    // 若未配置 export 属性，默认暴露服务
    return export == null || export;
  }
}
```
核心操作详解
- **配置校验与补全**
  - **检查必要属性**：确保`interface`（服务接口）、`ref`（服务实现Bean）已正确配置
  - **填充全局配置**：
    - 若未显示配置`provider、application、Protocol`等属性，尝试从spring容器或Dubbo全局配置管理器（`ConfigManager`）中获取默认配置。
    - 例如：未指定`<dubbo:protocol>`时，自动使用默认的Dubbo协议（端口20880）。
- **服务暴露触发**
  - **条件判断**：根据`export`属性（默认为`true`）决定是否暴露服务。
  - **延迟暴露处理**：
    - 若配置了`delay`参数（如`<dubbo:service delay="5000">`），则通过定时任务延迟调用`export()`。
    - 若未配置`delay`或`delay=-1`，则立即触发`export()`

解析`export()`方法，其继承自`ServiceConfig`：是服务暴露核心入口，负责将服务实现注册到注册中心并启动网络监听：
```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean{
    @Override
    public void export() {
        super.export();
        // Publish ServiceBeanExportedEvent
        publishExportEvent();
    }
}
```
分析`ServiceConfig`的`export()`逻辑。
```java
public class ServiceConfig<T> extends AbstractServiceConfig {
  public synchronized void export() {
    // 1.1 检查是否已暴露，避免重复暴露
    if (exported) {
      return;
    }
    exported = true;

    // 1.2 校验必要配置（接口名、实现类引用等）
    checkAndUpdateSubConfigs();

    // 1.3 延迟暴露处理（若配置了 delay）
    if (shouldDelay()) {
      delayExportExecutor.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
    } else {
      doExport();
    }
  }
}
```
核心暴露逻辑`doExport()`，以下是总结提炼后的代码：
```java
protected synchronized void doExport() {
    // 2.1 再次检查是否已暴露
    if (exported) {
        return;
    }
    exported = true;

    // 2.2 生成服务 URL（包含协议、IP、端口、接口等信息）
    List<URL> registryURLs = loadRegistries(true);
    for (ProtocolConfig protocolConfig : protocols) {
        String pathKey = URL.buildKey(getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version);
        // 2.3 构建服务 URL
        URL url = new URL(protocolConfig.getName(), host, port, pathKey, getServiceParameters(protocolConfig));

        // 2.4 多协议暴露：为每个协议生成独立的 Exporter
        Exporter<?> exporter = protocol.export(createInvoker(url));
        exporters.add(exporter);
    }

    // 2.5 触发注册中心服务注册
    registerService();
}
```
### 步骤1、生成注册中心URL
- **方法**：`loadRegistries()`
- **作用**：加载所有注册中心配置（如多个`ZK/Nacos`节点），生成对应的注册中心URL
- 示例输出
```text
zookeeper://localhost:2181/org.apache.dubbo.registry.RegistryService?application=user-service
```

### 步骤2、构建服务URL
- **方法**：`URL url = new URL(protocolConfig.getName(), host, port, pathKey, getServiceParameters(protocolConfig));`
- **参数解析**：
  - **协议**：如dubbo、HTTP
  - **主机与端口**：自动获取或手动指定
  - **路径**：服务接口全限定名
  - **参数**：超时时间、版本version、分组group等
- 示例URL
```text
dubbo://192.168.1.100:20880/com.example.UserService?timeout=3000&version=1.0.0
```

### 步骤3、协议暴露 protocol.export()
- **协议选择**：根据**protocolConfig**获取对应的**Protocol**实现（如**DubboProtocol**）。
- **生成invoker**：将服务实现封装为`Invoker`对象，用于处理远程调用。
```java
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
```
> Invoker 创建过程
> `proxyFactory.getInvoker(ref, (Class) interfaceClass, url)`
> 通过ProxyFactory 创建而来，Dubbo 默认的 `ProxyFactory` 实现类是 `JavassistProxyFactory`
> 为目标类创建 Wrapper，在创建 Wrapper 子类的过程中，子类代码生成逻辑会对 getWrapper 方法传入的 Class 对象进行解析，拿到诸如类方法，类成员变量等信息

- **启动网络服务**：调用协议实现的`export()`方法，启动`Netty`等网络服务器。
```java
// DubboProtocol.export()
public <T> Exporter<T> export(Invoker<T> invoker) {
    // 创建 Exporter，管理服务生命周期
    DubboExporter<T> exporter = new DubboExporter<>(serviceKey, invoker, protocolServer);
    // 启动 Netty 服务端
    protocolServer.start();
    return exporter;
}
```


### 步骤4、注册服务到注册中心
**流程**：将服务URL发布到所有注册中心。
```java
for (Registry registry : registries) {
    //RegistryProtocol#register
    registry.register(serviceUrl);
}
```
实现细节
- 注册中心客户端（如`ZookeeperRegistry`)创建持久化节点。
- 服务元数据写入注册中心（如`Zookeeper`的`/dubbo/com.example.UserService/providers`节点）。


## 3.3、完整流程图示
```text
1. export() 触发
   │
2. 校验配置，生成注册中心 URL
   │
3. 遍历协议配置，生成服务 URL
   │
4. 为每个协议生成 Invoker 并暴露服务
   │   ├─ 启动 Netty 服务端（DubboProtocol）
   │   └─ 生成 Exporter 管理生命周期
   │
5. 注册服务 URL 到所有注册中心
   │
6. 服务就绪，等待 Consumer 请求
```


参考文章：
[Dubbo源码跟踪](https://nnkwrik.github.io/2019/04/16/20190416/)