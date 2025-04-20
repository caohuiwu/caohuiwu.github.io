---
title: 《dubbo》Invoker
date: 2021-06-11
categories:
  - [dubbo]
---

    这是dubbo系列的第5篇文章，主要介绍的是dubbo的Invoker。

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

在Consumer启动过程中，我们大概了解了Invoker，但是具体的结构没有深入讲解。

# 二、Invoker
Dubbo内部定义的类，封装调用逻辑，动态代理对象调用方法时最终委托给`Invoker`执行。
```java
public interface Invoker<T> extends Node {
    Class<T> getInterface();
    Result invoke(Invocation invocation) throws RpcException;
}
```

## 2.1、Invoker相关实现
`dubbo`中常用的`Invoker`类，如下：
![常见的invoker](2021-06-11-dubbo-invoker/常见的invoker.png)
- `AbstractProxyInvoker` 本地执行类的`Invoker`，实际通过Java反射的方式执行原始对象的方法。为`proxyFactory.getInvoker`生成的被`wrapper`包装过的`Invoker`-------`InvokerWrapper`。
- `AbstractInvoker`: 远程通信类的`Invoker`，实际通过通信协议发起远程调用请求，并接收响应。
- `AbstraceClusterInvoker` 多个远程通信类的`Invoker`聚合成的集群`Invoker`，加入了集群容错和负载均衡策略。`cluster`集群下的`Invoker`实现，采用模板方法设计模式，`abstract`定义方法框架，具体方法实现由子类实现。


## 2.2、Invoker和URL
通过`protocol.refer`创建`invoker`时，需要将`URL`传入。
```java
invoker = refprotocol.refer(interfaceClass, urls.get(0));
```

### 2.2.1、URL
**URL的作用**：统一资源定位符，在Dubbo中传递配置信息，格式如下：
```text
protocol://host:port/path?key1=value1&key2=value2
```

**URL参数传递示例**
```xml
<!--消费者配置-->
<dubbo:reference interface="com.example.UserService" timeout="5000" loadbalance="leastactive"/>
```


**关键参数**

| 参数          | 说明                 | 示例值               |
|---------------|----------------------|----------------------|
| `protocol`    | 协议类型             | `dubbo`, `http`      |
| `host:port`   | 服务提供者地址       | `192.168.1.100:20880`|
| `path`        | 服务接口全限定名     | `com.example.UserService` |
| `version`     | 服务版本             | `1.0.0`              |
| `timeout`     | 调用超时时间（ms）   | `3000`               |
| `loadbalance` | 负载均衡策略         | `random`, `roundrobin` |

### 2.2.2、Invoker与URL的关系

#### 2.2.2.1、创建Invoker时依赖URL
- **配置来源**：URL包含协议、地址、接口、参数等信息，决定如何创建Invoker。
- **示例代码**：
  ```java
  // 创建Dubbo协议的Invoker
  URL url = new URL("dubbo", "192.168.1.100", 20880, "com.example.UserService", params);
  Invoker<UserService> invoker = protocol.refer(UserService.class, url);
  ```

#### 2.2.2.2、Invoker行为由URL参数控制
- **超时时间**：`timeout=3000`。
- **重试策略**：`retries=2`。
- **负载均衡**：`loadbalance=random`。

**运行时动态调整**：通过Dubbo动态配置中心修改URL参数，实时生效。

#### 2.2.2.3、不同协议对应不同Invoker实现
| 协议    | Invoker实现类       | 说明                      |
|---------|---------------------|-------------------------|
| Dubbo   | `DubboInvoker`      | 基于Netty的二进制RPC调用，默认协议   |
| HTTP    | `HttpInvoker`       | 基于HTTP/JSON的RESTful调用   |
| gRPC    | `GrpcInvoker`       | 基于gRPC协议的调用             |



## 2.3、Invoker的创建
在创建代理对象的过程中去创建`Invoker`，简化后的执行逻辑如下：
```java
public class ReferenceConfig<T> extends AbstractReferenceConfig {
    private T createProxy(Map<String, String> map) {
        if (isJvmRefer) {
            //本地调用
            invoker = refprotocol.refer(interfaceClass, url);
        } else {
            //远程调用
            if (url != null && url.length() > 0) {
                //消费者通过配置直接指定了提供者的地址。
                urls.add(ClusterUtils.mergeUrl(url, map));
            } else {
                //消费者通过注册中心获取提供者的地址。
                List<URL> us = loadRegistries(false);
                if (us != null && !us.isEmpty()) {
                    for (URL u : us) {
                      URL monitorUrl = loadMonitor(u);
                      urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    }
                }
            }
            if (urls.size() == 1) {
                //单URL场景的远程调用
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                //多URL场景的远程调用
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                  invokers.add(refprotocol.refer(interfaceClass, url));
                }
                // 创建 StaticDirectory 实例，并由 Cluster 对多个 Invoker 进行合并
                invoker = cluster.join(new StaticDirectory(invokers));
            }
        }
    }
}
```
从代码中，我们可以分成以下流程
- 本地调用
- 远程调用
  - 第一步：获取URL，添加到`urls`集合
    - 消费者通过**配置**直接指定了提供者的地址
    - 消费者通过**注册中心**获取提供者的地址。
  - 第二步：根据URL构建`Invoker`
    - URL只有一个，直接创建：这里的单URL 场景指的是 单一注册中心或 单一直连URL(服务提供者)时 远程调用服务`Invoker` 的创建，会直接调用 `refprotocol.refer`
    - URL多个，通过 `Cluster` 对多个 `Invoker` 进行合并

从代码中可以看到，invoker都是通过`refprotocol.refer()`方法创建的。

更多invoker的创建流程，查看下一篇文章。


参考文章：
[消费者启动流程](https://blog.csdn.net/qq_36882793/article/details/115726433)
[消费者启动流程 - RegistryProtocol#refer](https://blog.csdn.net/qq_36882793/article/details/116661370)