---
title: Mybatis线上事故解析(一、二级缓存)
date: 2018-12-11 23:00:42
categories: 
  - [mybatis]
tags:
  - [线上事故]
---

# 一、问题场景：统一登录平台出现无法登录和菜单异常问题。

# 二、问题原因：
   
    1、mybatis二级缓存组件使用不当：在高并发情况下，mybatis二级缓存组件实现方式会在数据更新，修改和删除时，
        频繁调用redis中的keysByScan去获取当前库中全局匹配到的key，当key数量足够多时，执行时间会十分漫长，
        导致整个redis服务并夯住，出于不可用状态
    2、redis超时时间设置过长，redis超时后，不能将请求漏给mysql数据库做兜底支撑
    3、线索系统，每5分钟，全量刷新一次所有用户权限数据，一天调用量达到2000w以上，占整个权限服务的调用量的75%以上 

反思总结：
    
    1、并发场景下掉所有的mybatis二级缓存组件，在服务层做精准缓存，而不是dao层做数据缓存
    2、负责同学对系统中引入的组件具体实现和参数配置不了解
    3、负责同学对系统日志配置不了解，日志配置不当导致引入的组件的日志未能打印出来，影响问题的排期方向和时间
    4、配置文件不要打入jar包，而是以文件方式统一放在服务的config目录中，方便修改 

<!--more-->    
![redis-cache](2018-12-11-Mybatis-online-accident/redis-cache.jpg)    
![redis-connect](2018-12-11-Mybatis-online-accident/redis-connect.jpg)
![redis-io](2018-12-11-Mybatis-online-accident/redis-io.jpg)
   
    
# 三、Mybatis一、二级缓存原理解析

mybaits 一共有两级缓存：一级缓存的配置 key 是 localCacheScope，而二级缓存的配置 key 是 cacheEnabled，从名字上可以得出以下信息：
一级缓存是本地或者说局部缓存，它不能被关闭，只能配置缓存范围。SESSION 或者 STATEMENT。
二级缓存才是 mybatis 的正统，功能会更强大些。

#### 3.1、一级缓存
配置：
开发者只需在MyBatis的配置文件中，添加如下语句，就可以使用一级缓存。共有两个选项，SESSION或者STATEMENT，默认是SESSION级别，即在一个MyBatis会话中执行的所有语句，都会共享这一个缓存。一种是STATEMENT级别，可以理解为缓存只对当前执行的这一个Statement有效。

    <setting name="localCacheScope" value="SESSION"/>

#### 3.2、二级缓存
配置：

1.在MyBatis的配置文件中开启二级缓存。
    
    <setting name="cacheEnabled" value="true"/>
    
2.在MyBatis的映射XML中配置cache或者 cache-ref 。
cache标签用于声明这个namespace使用二级缓存，并且可以自定义配置。

    <mapper namespace="com..mapper.RoleMapper">
        <cache type="com.mybatis.RedisCache">
            <property name="expireSeconds" value="14400"/>
        </cache>
    </mapper>   
    
    type：cache使用的类型，默认是PerpetualCache，这在一级缓存中提到过。
    eviction： 定义回收的策略，常见的有FIFO，LRU。
    flushInterval： 配置一定时间自动刷新缓存，单位是毫秒。
    size： 最多缓存对象的个数。
    readOnly： 是否只读，若配置可读写，则需要对应的实体类能够序列化。
    blocking： 若缓存中找不到对应的key，是否会一直blocking，直到有对应的数据进入缓存。
    cache-ref代表引用别的命名空间的Cache配置，两个命名空间的操作使用的是同一个Cache。

    <cache-ref namespace="mapper.StudentMapper"/>

spring+mybatis的执行过程：

    <!--mybatis设置-->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="configLocation" value="classpath:mybatis-config.xml" />
        <property name="dataSource" ref="mysqlDataSource" /> <!--必须有-->
        <property name="mapperLocations" value="classpath:mapper/**/*.xml" />
        <property name="typeAliasesPackage" value="contract.model" />
        <property name="resultTypesPackage" value="contract" />
    </bean>

Spring对SqlSessionFactroyBean的初始化过程

    首先SqlSessionFactoryBean 实现了了Spring 的FacatoryBean，顾名思义就是Spring的对象创建工厂，Spring在创建对象的时候会调用对象工厂的getObject()方法。

    public SqlSessionFactory getObject() throws Exception {
        if (this.sqlSessionFactory == null) {
            this.afterPropertiesSet();
        }
        return this.sqlSessionFactory;
    }
    
    public void afterPropertiesSet() throws Exception {
        Assert.notNull(this.dataSource, "Property 'dataSource' is required");
        Assert.notNull(this.sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
        Assert.state(this.configuration == null && this.configLocation == null || this.configuration == null || this.configLocation == null, "Property 'configuration' and 'configLocation' can not specified with together");
        this.sqlSessionFactory = this.buildSqlSessionFactory();
    }
    
    protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
        //配置文件读取
        ....
        return this.sqlSessionFactoryBuilder.build(configuration);
    }
    
    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
    
在初始化SqlSesion时，会使用Configuration类创建一个全新的Executor，作为DefaultSqlSession构造函数的参数，创建Executor代码如下所示：
    
    public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
        executorType = executorType == null ? defaultExecutorType : executorType;
        executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
        Executor executor;
        if (ExecutorType.BATCH == executorType) {
          executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
          executor = new ReuseExecutor(this, transaction);
        } else {
          executor = new SimpleExecutor(this, transaction);
        }
        // 尤其可以注意这里，如果二级缓存开关开启的话，是使用CahingExecutor装饰BaseExecutor的子类
        if (cacheEnabled) {
          executor = new CachingExecutor(executor);                      
        }
        executor = (Executor) interceptorChain.pluginAll(executor);
        return executor;
    }
    
# 四、总结

    1.MyBatis一级缓存的生命周期和SqlSession一致。
    2.MyBatis一级缓存内部设计简单，只是一个没有容量限定的HashMap，在缓存的功能性上有所欠缺。
    3.MyBatis的一级缓存最大范围是SqlSession内部，有多个SqlSession或者分布式的环境下，数据库写操作会引起脏数据，建议设定缓存级别为Statement。    
    
#### 4.1、类关系图：
![sqlsession-class-relation](2018-12-11-Mybatis-online-accident/sqlsession-class-relation.jpg)    
    
每个SqlSession中持有了Executor，每个Executor中有一个LocalCache。当用户发起查询时，MyBatis根据当前执行的语句生成MappedStatement，在Local Cache进行查询，如果缓存命中的话，直接返回结果给用户，如果缓存没有命中的话，查询数据库，结果写入Local Cache，最后返回结果给用户。    

参考博客：
[SpringBoot 下 Mybatis 的缓存](https://juejin.im/post/5cacaf2df265da03a97acd25)
[聊聊MyBatis缓存机制](https://tech.meituan.com/2018/01/19/mybatis-cache.html)
[深入浅出mybatis之与spring集成](https://www.cnblogs.com/nuccch/p/7693801.html)
    