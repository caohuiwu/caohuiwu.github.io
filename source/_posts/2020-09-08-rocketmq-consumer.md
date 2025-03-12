---
title: 《rocketmq》consumer的原理
date: 2020-09-08
categories:
  - [ rocketmq, consumer, 消息消费 ]
  - [ rocketmq, rebalance ]
---

    这是rocketmq系列的第三篇文章，主要介绍的是rocketmq的消费者。

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
</style>

# 一、核心组件--Consumer
消息消费者，负责消费消息

简单示例如下：
```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("GROUP_NAME");
consumer.setNamesrvAddr("127.0.0.1:9876"); // 设置 NameServer 地址
consumer.subscribe("TOPIC_NAME", "*");      // 订阅 Topic 和 Tag（* 表示所有 Tag）
consumer.setConsumeThreadMin(10);           // 最小消费线程数
consumer.setConsumeThreadMax(20);           // 最大消费线程数
consumer.start();                           // 启动消费者
```
最重要的是<code class="my-code">consumer.start()</code>消费者启动。

<!-- more -->


# 二、consumer启动
入口：<code class="my-code">consumer.start()</code>
```java
public class DefaultMQPushConsumer extends ClientConfig implements MQPushConsumer {
    public void start() throws MQClientException {
        setConsumerGroup(NamespaceUtil.wrapNamespace(this.getNamespace(), this.consumerGroup));
        this.defaultMQPushConsumerImpl.start();
        if (null != traceDispatcher) {
            try {
                traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
            } catch (MQClientException e) {
                log.warn("trace dispatcher start failed ", e);
            }
        }
    }
}
```
**备注**： DefaultMQProducer的构造器，send和start等相关的方法，其实都是围绕<code class="my-code">DefaultMQProducerImpl</code>来转，defaultMQProducerImpl：默认生产者的实现类，其start方法作为生产者启动的核心方法,接下来将核心分析其start方法的实现.

## 2.1、DefaultMQPushConsumerImpl#start
代码简化后如下：
```java
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        // 初始化流程
        case CREATE_JUST:
            log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                    this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
            this.serviceState = ServiceState.START_FAILED;
            // 校验设置consumerGroup的配置
            this.checkConfig();
            // 解析订阅的topic和tag等数据，构建订阅关系模型放到rebalanceImpl里
            this.copySubscription();

            if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                this.defaultMQPushConsumer.changeInstanceNameToPID();
            }

            // 初始化MQClientInstance、rebalaceImpl等
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
            this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
            this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

            if (this.pullAPIWrapper == null) {
                this.pullAPIWrapper = new PullAPIWrapper(
                        mQClientFactory,
                        this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
            }
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

            // 初始化消费进度
            // 集群模式的话，消费进度保存在服务端broker；广播模式，保存在客户端consumer上
            if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
            } else {
                switch (this.defaultMQPushConsumer.getMessageModel()) {
                    case BROADCASTING:
                        this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    case CLUSTERING:
                        this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    default:
                        break;
                }
                this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
            }
            this.offsetStore.load();

            // 根据是否顺序or并发消费，来创建不同的消费线程服务
            // 本次主要关注并发消费，
            if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                this.consumeOrderly = true;
                this.consumeMessageService =
                        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
                //POPTODO reuse Executor ?
                this.consumeMessagePopService = new ConsumeMessagePopOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
            } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                this.consumeOrderly = false;
                this.consumeMessageService =
                        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
                //POPTODO reuse Executor ?
                this.consumeMessagePopService =
                        new ConsumeMessagePopConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
            }
            this.consumeMessageService.start();
            // POPTODO
            this.consumeMessagePopService.start();

            // 向mQClientFactory注册当前DefaultMQPushConsumerImpl
            // mQClientFactory里维护了一个consumerGroup和DefaultMQPushConsumerImpl的映射表
            boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                this.consumeMessageService.shutdown(defaultMQPushConsumer.getAwaitTerminationMillisWhenShutdown());
                throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
            }

            // 核心启动方法
            // 这块是重点的启动流程，稍后展开说明
            mQClientFactory.start();
            log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
        default:
            break;
    }

    this.updateTopicSubscribeInfoWhenSubscriptionChanged();
    this.mQClientFactory.checkClientInBroker();
    //向每个Broker发送心跳信息
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    //立即触发一次rebalance
    this.mQClientFactory.rebalanceImmediately();//这个方法内部实际上，是通过唤醒一个RebalanceService，来触发Rebalance：
}
```
可以看到Consumer启动主要分为5个步骤：
- **步骤1**：启动准备工作，这里使用{...}表示省略，以更清楚看清整个流程
- **步骤2**：从nameserver更新topic路由信息，收集到了Rebalance所需的队列信息
- **步骤3**：检查consumer配置(主要是为了功能兼容，例如consumer要使用SQL92过滤，但是broker并没有开启，则broker会返回错误)
- **步骤4**：向每个broker发送心跳信息，将自己加入消费者组
- **步骤5**：立即触发一次rebalance，在步骤2和4的基础上立即触发一次Rebalance


## 2.2、MQClientInstance#start()
```java
public void start() throws MQClientException {
    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                // 这里拿到nameServer地址list
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // Start request-response channel
                // 开启客户端和mq服务端的连接通道
                this.mQClientAPIImpl.start();
                // Start various schedule tasks
                // 启动定时任务
                // 主要是定时2分钟调用上面fetchNameServerAddr方法，拉取最新的nameServerAddr
                this.startScheduledTask();
                // Start pull service
                // 启动拉取消息的服务
                this.pullMessageService.start();
                // Start rebalance service
                // 启动负载均衡服务
                this.rebalanceService.start();
                // Start push service
                // 启动消息producer的推送服务，这块不展开了
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                // 以上操作都成功的话，将服务置为running
                this.serviceState = ServiceState.RUNNING;
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```
主要流程分了如下几步，下面会将下面的几步逐个展开说明：
1. 拿到nameServer地址list
2. 开启客户端和mq服务端的连接通道
3. 启动定时任务（定时调用拿nameServer地址list）
4. 启动拉取消息的服务
5. 启动负载均衡服务
6. 启动消息producer的推送服务


### 2.2.1、拿到nameServer地址list
首先进入到<code class="my-code">org.apache.rocketmq.client.impl.MQClientAPIImpl#fetchNameServerAddr</code>可以看下面方法，可以看到主要就是调用<code class="my-code">topAddressing.fetchNSAddr</code>方法
```java
public String fetchNameServerAddr() {
    try {
        // 主要是执行该方法
        String addrs = this.topAddressing.fetchNSAddr();
        if (!UtilAll.isBlank(addrs)) {
            if (!addrs.equals(this.nameSrvAddr)) {
                log.info("name server address changed, old=" + this.nameSrvAddr + ", new=" + addrs);
                this.updateNameServerAddressList(addrs);
                this.nameSrvAddr = addrs;
                return nameSrvAddr;
            }
        }
    } catch (Exception e) {
        log.error("fetchNameServerAddr Exception", e);
    }
    return nameSrvAddr;
}
```

### 2.2.2、开启客户端和mq服务端的连接通道
<code class="my-code">org.apache.rocketmq.remoting.netty.NettyRemotingClient#start</code>
这块主要分为三个部分：
1. 启动netty客户端
2. 延迟3s扫描连接响应表
3. 扫描<code class="my-code">availableNamesrvAddrMap</code>

```java
public void start() {
  // 这里标准的编写netty客户端代码
  // 主要是添加了NettyClientHandler
  // 但是此时bootstarp还没有绑定远程服务器host和端口
    if (this.defaultEventExecutorGroup == null) {
        this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
            nettyClientConfig.getClientWorkerThreads(),
            new ThreadFactoryImpl("NettyClientWorkerThread_"));
    }
    Bootstrap handler = this.bootstrap.group(this.eventLoopGroupWorker).channel(NioSocketChannel.class)
        .option(ChannelOption.TCP_NODELAY, true)
        .option(ChannelOption.SO_KEEPALIVE, false)
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, nettyClientConfig.getConnectTimeoutMillis())
        .handler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline pipeline = ch.pipeline();
                if (nettyClientConfig.isUseTLS()) {
                    if (null != sslContext) {
                        pipeline.addFirst(defaultEventExecutorGroup, "sslHandler", sslContext.newHandler(ch.alloc()));
                        LOGGER.info("Prepend SSL handler");
                    } else {
                        LOGGER.warn("Connections are insecure as SSLContext is null!");
                    }
                }
                ch.pipeline().addLast(
                    nettyClientConfig.isDisableNettyWorkerGroup() ? null : defaultEventExecutorGroup,
                    new NettyEncoder(),
                    new NettyDecoder(),
                    new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()),
                    new NettyConnectManageHandler(),
                    new NettyClientHandler());
            }
        });
    if (nettyClientConfig.getClientSocketSndBufSize() > 0) {
        LOGGER.info("client set SO_SNDBUF to {}", nettyClientConfig.getClientSocketSndBufSize());
        handler.option(ChannelOption.SO_SNDBUF, nettyClientConfig.getClientSocketSndBufSize());
    }
    if (nettyClientConfig.getClientSocketRcvBufSize() > 0) {
        LOGGER.info("client set SO_RCVBUF to {}", nettyClientConfig.getClientSocketRcvBufSize());
        handler.option(ChannelOption.SO_RCVBUF, nettyClientConfig.getClientSocketRcvBufSize());
    }
    if (nettyClientConfig.getWriteBufferLowWaterMark() > 0 && nettyClientConfig.getWriteBufferHighWaterMark() > 0) {
        LOGGER.info("client set netty WRITE_BUFFER_WATER_MARK to {},{}",
            nettyClientConfig.getWriteBufferLowWaterMark(), nettyClientConfig.getWriteBufferHighWaterMark());
        handler.option(ChannelOption.WRITE_BUFFER_WATER_MARK, new WriteBufferWaterMark(
            nettyClientConfig.getWriteBufferLowWaterMark(), nettyClientConfig.getWriteBufferHighWaterMark()));
    }
    if (nettyClientConfig.isClientPooledByteBufAllocatorEnable()) {
        handler.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
    }
 
    TimerTask timerTaskScanResponseTable = new TimerTask() {
        @Override
        public void run(Timeout timeout) {
            try {
                NettyRemotingClient.this.scanResponseTable();
            } catch (Throwable e) {
                LOGGER.error("scanResponseTable exception", e);
            } finally {
                timer.newTimeout(this, 1000, TimeUnit.MILLISECONDS);
            }
        }
    };
    this.timer.newTimeout(timerTaskScanResponseTable, 1000 * 3, TimeUnit.MILLISECONDS);
 
    int connectTimeoutMillis = this.nettyClientConfig.getConnectTimeoutMillis();
    TimerTask timerTaskScanAvailableNameSrv = new TimerTask() {
        @Override
        public void run(Timeout timeout) {
            try {
                NettyRemotingClient.this.scanAvailableNameSrv();
            } catch (Exception e) {
                LOGGER.error("scanAvailableNameSrv exception", e);
            } finally {
                timer.newTimeout(this, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }
        }
    };
    this.timer.newTimeout(timerTaskScanAvailableNameSrv, 0, TimeUnit.MILLISECONDS);
}
```



### 2.2.3、启动定时任务（定时调用拿nameServer地址list）
<code class="my-code">org.apache.rocketmq.client.impl.factory.MQClientInstance#startScheduledTask</code>

主要做启动一些客户端的定时操作，这里不展开说明
```java
private void startScheduledTask() {
  // 还是调用上面的获取nameServer的方法，定时取nameServer地址列表
    if (null == this.clientConfig.getNamesrvAddr()) {
        this.scheduledExecutorService.scheduleAtFixedRate(() -> {
            try {
                MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
            } catch (Exception e) {
                log.error("ScheduledTask fetchNameServerAddr exception", e);
            }
        }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
    }
 
  // 定时更新topic路由信息
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        try {
            MQClientInstance.this.updateTopicRouteInfoFromNameServer();
        } catch (Exception e) {
            log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
        }
    }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);
 
    // 定时发送信息
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        try {
            MQClientInstance.this.cleanOfflineBroker();
            MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
        } catch (Exception e) {
            log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
        }
    }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);
    // 定时持久化消费位点
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        try {
            MQClientInstance.this.persistAllConsumerOffset();
        } catch (Exception e) {
            log.error("ScheduledTask persistAllConsumerOffset exception", e);
        }
    }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
    // 定时调整线程池
    this.scheduledExecutorService.scheduleAtFixedRate(() -> {
        try {
            MQClientInstance.this.adjustThreadPool();
        } catch (Exception e) {
            log.error("ScheduledTask adjustThreadPool exception", e);
        }
    }, 1, 1, TimeUnit.MINUTES);
}
```

### 2.2.4、启动拉取消息的服务
这一部分主要是mq客户端向broker拉取消息，然后解码、过滤后分批交给消费服务进行消费

<code class="my-code">this.pullMessageService.start()</code>会走到如下方法,<code class="my-code">org.apache.rocketmq.common.ServiceThread#start</code>
```java
public void start() {
    log.info("Try to start service thread:{} started:{} lastThread:{}", getServiceName(), started.get(), thread);
    if (!started.compareAndSet(false, true)) {
        return;
    }
    // 设置stop标记我false，同时新起一个线程执行当前类的run方法
    stopped = false;
    this.thread = new Thread(this, getServiceName());
    this.thread.setDaemon(isDaemon);
    this.thread.start();
    log.info("Start service thread:{} started:{} lastThread:{}", getServiceName(), started.get(), thread);
}
```
<code class="my-code">pullMessageService</code>实现了Runnable接口，会单独起一个线程并调用如下run方法

<code class="my-code">org.apache.rocketmq.client.impl.consumer.PullMessageService#run</code>
```java
public class PullMessageService extends ServiceThread {
    @Override
    public void run() {
        logger.info(this.getServiceName() + " service started");
        // 上文设置了stopped为false，会到while里
        while (!this.isStopped()) {
            try {
                // 从messageRequestQueue里取一个MessageRequest执行。若队列为空则堵塞
                MessageRequest messageRequest = this.messageRequestQueue.take();
                if (messageRequest.getMessageRequestMode() == MessageRequestMode.POP) {
                    this.popMessage((PopRequest) messageRequest);
                } else {
                    this.pullMessage((PullRequest) messageRequest);
                }
            } catch (InterruptedException ignored) {
            } catch (Exception e) {
                logger.error("Pull Message Service Run Method exception", e);
            }
        }

        logger.info(this.getServiceName() + " service end");
    }
}
```
以pullMessage方式为例，进入如下方法

<code class="my-code">org.apache.rocketmq.client.impl.consumer.PullMessageService#pullMessage</code>
```java
public class PullMessageService extends ServiceThread {
    private void pullMessage(final PullRequest pullRequest) {
        // 根据PullRequest的消费组名从map中获取一个MQConsumerInner
        final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
        if (consumer != null) {
            // consumer不为空执行拉消息方法
            DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
            impl.pullMessage(pullRequest);
        } else {
            logger.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
        }
    }
}
```
下面展开说明<code class="my-code">DefaultMQPushConsumerImpl#pullMessage</code>方法

<code class="my-code">org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage</code>
```java
public void pullMessage(final PullRequest pullRequest) {
    // 获取处理队列
    final ProcessQueue processQueue = pullRequest.getProcessQueue();
    // droped为true直接返回
    if (processQueue.isDropped()) {
        log.info("the pull request[{}] is dropped.", pullRequest.toString());
        return;
    }
 
    // 设置最近的拉取时间戳
    pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());
 
    // 如果状态不是running的话，则将pullRequest延迟3秒放进messageRequestQueue，并返回
    // executePullRequestLater方法操作即是延迟pullTimeDelayMillsWhenException时间将pullRequest放进messageRequestQueue尾部
    // executePullRequestLater方法通过上述操作操作，来打到随后执行pullRequest目的
    try {
        this.makeSureStateOK();
    } catch (MQClientException e) {
        log.warn("pullMessage exception, consumer state not ok", e);
        this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
        return;
    }
 
    // pause为true的话，延迟1s将pullRequest放进messageRequestQueue，并返回
    if (this.isPause()) {
        log.warn("consumer was paused, execute pull request later. instanceName={}, group={}", this.defaultMQPushConsumer.getInstanceName(), this.defaultMQPushConsumer.getConsumerGroup());
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);
        return;
    }
 
    // 下面这部分为消息拉取的流控代码
    long cachedMessageCount = processQueue.getMsgCount().get();
    long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);
 
    // 当前处理队列消息超过1000时，延迟50ms执行该pullRequest；该操作超过1000次，打印日志
    if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL);
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(
                "the cached message count exceeds the threshold {}, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                this.defaultMQPushConsumer.getPullThresholdForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
        }
        return;
    }
 
    // 当前处理队列消息占用空间超过100mb时，延迟50ms后执行该pullRequest
    if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL);
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(
                "the cached message size exceeds the threshold {} MiB, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                this.defaultMQPushConsumer.getPullThresholdSizeForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
        }
        return;
    }
 
    // 非顺序消息的话
    if (!this.consumeOrderly) {
        // 如果处理队列里的的最大消息和最小消息的queueSize差值大于2000的话，流控延后处理
        if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL);
            if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
                log.warn(
                    "the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",
                    processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),
                    pullRequest, queueMaxSpanFlowControlTimes);
            }
            return;
        }
    // 下面为顺序消息，本次先不分析
    } else {
        if (processQueue.isLocked()) {
            if (!pullRequest.isPreviouslyLocked()) {
                long offset = -1L;
                try {
                    offset = this.rebalanceImpl.computePullFromWhereWithException(pullRequest.getMessageQueue());
                    if (offset < 0) {
                        throw new MQClientException(ResponseCode.SYSTEM_ERROR, "Unexpected offset " + offset);
                    }
                } catch (Exception e) {
                    this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
                    log.error("Failed to compute pull offset, pullResult: {}", pullRequest, e);
                    return;
                }
                boolean brokerBusy = offset < pullRequest.getNextOffset();
                log.info("the first time to pull message, so fix offset from broker. pullRequest: {} NewOffset: {} brokerBusy: {}",
                    pullRequest, offset, brokerBusy);
                if (brokerBusy) {
                    log.info("[NOTIFYME]the first time to pull message, but pull request offset larger than broker consume offset. pullRequest: {} NewOffset: {}",
                        pullRequest, offset);
                }
 
                pullRequest.setPreviouslyLocked(true);
                pullRequest.setNextOffset(offset);
            }
        } else {
            this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
            log.info("pull message later because not locked in broker, {}", pullRequest);
            return;
        }
    }
 
    // 拿到该消息topic的订阅信息，topic  tag 表达式等数据
    final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    if (null == subscriptionData) {
        this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
        log.warn("find the consumer's subscription failed, {}", pullRequest);
        return;
    }
 
    final long beginTimestamp = System.currentTimeMillis();
 
    // 下面是拉取消息的回调函数，等下展开说明
    PullCallback pullCallback = new PullCallback() {
        @Override
        public void onSuccess(PullResult pullResult) {
            if (pullResult != null) {
                pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                    subscriptionData);
 
                switch (pullResult.getPullStatus()) {
                    case FOUND:
                        long prevRequestOffset = pullRequest.getNextOffset();
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                        long pullRT = System.currentTimeMillis() - beginTimestamp;
                        DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                            pullRequest.getMessageQueue().getTopic(), pullRT);
 
                        long firstMsgOffset = Long.MAX_VALUE;
                        if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        } else {
                            firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();
 
                            DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());
 
                            boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                            DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                pullResult.getMsgFoundList(),
                                processQueue,
                                pullRequest.getMessageQueue(),
                                dispatchToConsume);
 
                            if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                                DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                    DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                            } else {
                                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            }
                        }
 
                        if (pullResult.getNextBeginOffset() < prevRequestOffset
                            || firstMsgOffset < prevRequestOffset) {
                            log.warn(
                                "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                                pullResult.getNextBeginOffset(),
                                firstMsgOffset,
                                prevRequestOffset);
                        }
 
                        break;
                    case NO_NEW_MSG:
                    case NO_MATCHED_MSG:
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());
 
                        DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
 
                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        break;
                    case OFFSET_ILLEGAL:
                        log.warn("the pull request offset illegal, {} {}",
                            pullRequest.toString(), pullResult.toString());
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());
 
                        pullRequest.getProcessQueue().setDropped(true);
                        DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {
 
                            @Override
                            public void run() {
                                try {
                                    DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                        pullRequest.getNextOffset(), false);
 
                                    DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());
 
                                    DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());
 
                                    log.warn("fix the pull request offset, {}", pullRequest);
                                } catch (Throwable e) {
                                    log.error("executeTaskLater Exception", e);
                                }
                            }
                        }, 10000);
                        break;
                    default:
                        break;
                }
            }
        }
 
        @Override
        public void onException(Throwable e) {
            if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                log.warn("execute the pull request exception", e);
            }
 
            if (e instanceof MQBrokerException && ((MQBrokerException) e).getResponseCode() == ResponseCode.FLOW_CONTROL) {
                DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_BROKER_FLOW_CONTROL);
            } else {
                DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
            }
        }
    };
 
    boolean commitOffsetEnable = false;
    long commitOffsetValue = 0L;
    if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) {
        commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
        if (commitOffsetValue > 0) {
            commitOffsetEnable = true;
        }
    }
 
    // 拿到订阅表达式 类过滤标识等
    String subExpression = null;
    boolean classFilter = false;
    SubscriptionData sd = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    if (sd != null) {
        if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
            subExpression = sd.getSubString();
        }
 
        classFilter = sd.isClassFilterMode();
    }
 
    // 构建标记
    int sysFlag = PullSysFlag.buildSysFlag(
        commitOffsetEnable, // commitOffset
        true, // suspend
        subExpression != null, // subscription
        classFilter // class filter
    );
    try {
        // 执行真正拉取消息的方法，并传输回调函数pullCallBack
        this.pullAPIWrapper.pullKernelImpl(
            pullRequest.getMessageQueue(),
            subExpression,
            subscriptionData.getExpressionType(),
            subscriptionData.getSubVersion(),
            pullRequest.getNextOffset(),
            this.defaultMQPushConsumer.getPullBatchSize(),
            this.defaultMQPushConsumer.getPullBatchSizeInBytes(),
            sysFlag,
            commitOffsetValue,
            BROKER_SUSPEND_MAX_TIME_MILLIS,
            CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
            CommunicationMode.ASYNC,
            pullCallback
        );
    } catch (Exception e) {
        log.error("pullKernelImpl exception", e);
        this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
    }
}
```
#### 2.2.4.1、真正的拉消息
这一部分主要是通过netty客户端向broker拉取消息


<code class="my-code">org.apache.rocketmq.client.impl.consumer.PullAPIWrapper#pullKernelImpl</code>
- <code class="my-code">PullApiWrapper</code>：消息拉取核心对象

```java
public PullResult pullKernelImpl(
        .....
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        // 根据messageQueue拿到broker信息，包括broker地址、是否slave、版本等
        FindBrokerResult findBrokerResult =
            this.mQClientFactory.findBrokerAddressInSubscribe(this.mQClientFactory.getBrokerNameFromMessageQueue(mq),
                this.recalculatePullFromWhichNode(mq), false);
        // 拿不到的话，从nameServer更新topic信息后，重新拿
        if (null == findBrokerResult) {
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(mq.getTopic());
            findBrokerResult =
                this.mQClientFactory.findBrokerAddressInSubscribe(this.mQClientFactory.getBrokerNameFromMessageQueue(mq),
                    this.recalculatePullFromWhichNode(mq), false);
        }
        if (findBrokerResult != null) {
            {
                // check version
                if (!ExpressionType.isTagType(expressionType)
                    && findBrokerResult.getBrokerVersion() < MQVersion.Version.V4_1_0_SNAPSHOT.ordinal()) {
                    throw new MQClientException("The broker[" + mq.getBrokerName() + ", "
                        + findBrokerResult.getBrokerVersion() + "] does not upgrade to support for filter message by " + expressionType, null);
                }
            }
            int sysFlagInner = sysFlag;
 
            if (findBrokerResult.isSlave()) {
                sysFlagInner = PullSysFlag.clearCommitOffsetFlag(sysFlagInner);
            }
 
            // 组装请求头
            PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
            requestHeader.setConsumerGroup(this.consumerGroup);
            requestHeader.setTopic(mq.getTopic());
            requestHeader.setQueueId(mq.getQueueId());
            requestHeader.setQueueOffset(offset);
            requestHeader.setMaxMsgNums(maxNums);
            requestHeader.setSysFlag(sysFlagInner);
            requestHeader.setCommitOffset(commitOffset);
            requestHeader.setSuspendTimeoutMillis(brokerSuspendMaxTimeMillis);
            requestHeader.setSubscription(subExpression);
            requestHeader.setSubVersion(subVersion);
            requestHeader.setMaxMsgBytes(maxSizeInBytes);
            requestHeader.setExpressionType(expressionType);
            requestHeader.setBname(mq.getBrokerName());
 
            // 拿到broker地址
            String brokerAddr = findBrokerResult.getBrokerAddr();
            if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) {
                brokerAddr = computePullFromWhichFilterServer(mq.getTopic(), brokerAddr);
            }
 
            // 通过mq客户端拉取消息
            PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(
                brokerAddr,
                requestHeader,
                timeoutMillis,
                communicationMode,
                pullCallback);
 
            return pullResult;
        }
 
        throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
    }
```
进入<code class="my-code">org.apache.rocketmq.client.impl.MQClientAPIImpl#pullMessage</code>
```java
public PullResult pullMessage(
    final String addr,
    final PullMessageRequestHeader requestHeader,
    final long timeoutMillis,
    final CommunicationMode communicationMode,
    final PullCallback pullCallback
) throws RemotingException, MQBrokerException, InterruptedException {
    RemotingCommand request;
    // 这个先不纠结，根据不同的code组装不同的RemotingCommand
    if (PullSysFlag.hasLitePullFlag(requestHeader.getSysFlag())) {
        request = RemotingCommand.createRequestCommand(RequestCode.LITE_PULL_MESSAGE, requestHeader);
    } else {
        request = RemotingCommand.createRequestCommand(RequestCode.PULL_MESSAGE, requestHeader);
    }
 
    // 上文的communicationMode字段传入的是ASYNC，所以这里走的ASYNC模式
    switch (communicationMode) {
        case ONEWAY:
            assert false;
            return null;
        case ASYNC:
            this.pullMessageAsync(addr, request, timeoutMillis, pullCallback);
            return null;
        case SYNC:
            return this.pullMessageSync(addr, request, timeoutMillis);
        default:
            assert false;
            break;
    }
 
    return null;
}
```



# 三、consumer消息消费
消息消费有几个概念：
1. 消息消费模型（<code class="my-code">MessageModel</code>）：有广播、集群。广播模式所有consumer消费topic下所有信息；集群模式下所有consumer分别消费topic一部分MessageQueue。


## 3.1、consumer拉取消息
通过PullMessageService


# 四、consumer rebalance

## 4.1、Consumer Rebalance触发时机
Broker在Rebalance过程中起的是协调通知的作用，可以帮忙我们从整体对<code class="my-code">Rebalance</code>有个初步的认知。但是Rebalance的细节，却是在Consumer端完成的。
- **在启动时**，消费者会立即向所有Broker发送一次发送心跳(HEART_BEAT)请求，Broker则会将消费者添加由ConsumerManager维护的某个消费者组中。然后这个Consumer自己会立即触发一次Rebalance。
- **在运行时**，消费者接收到Broker通知会立即触发Rebalance，同时为了避免通知丢失，会周期性触发Rebalance；
- **当停止时**，消费者向所有Broker发送取消注册客户端(UNREGISTER_CLIENT)命令，Broker将消费者从ConsumerManager中移除，并通知其他Consumer进行Rebalance。

## 4.2、启动时触发
<code class="my-code">DefaultMQPushConsumerImpl</code>的start方法显示了一个消费者的启动流程，如下图所示：
代码简化后如下：
```java
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {...}
    this.updateTopicSubscribeInfoWhenSubscriptionChanged();
    this.mQClientFactory.checkClientInBroker();
    //向每个Broker发送心跳信息
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    //立即触发一次rebalance
    this.mQClientFactory.rebalanceImmediately();//这个方法内部实际上，是通过唤醒一个RebalanceService，来触发Rebalance：
}
```
可以看到Consumer启动主要分为5个步骤：
- **步骤1**：启动准备工作，这里使用{...}表示省略，以更清楚看清整个流程
- **步骤2**：从nameserver更新topic路由信息，收集到了Rebalance所需的队列信息
- **步骤3**：检查consumer配置(主要是为了功能兼容，例如consumer要使用SQL92过滤，但是broker并没有开启，则broker会返回错误)
- **步骤4**：向每个broker发送心跳信息，将自己加入消费者组
- **步骤5**：立即触发一次rebalance，在步骤2和4的基础上立即触发一次Rebalance

### 4.2.1、步骤2 ：更新订阅的topic路由信息
上述代码步骤2，调用<code class="my-code">updateTopicSubscribeInfoWhenSubscriptionChanged()</code>方法，从NameServer更新topic路由信息，由于一个消费者可以订阅多个topic，因此这个Topic都需要更新，如下：
```java
private void updateTopicSubscribeInfoWhenSubscriptionChanged() {
    Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
    if (subTable != null) {
        for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
            final String topic = entry.getKey();
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        }
    }
}
```
通过这一步，当前Consumer就拿到了Topic下所有队列信息，具备了Rebalance的第一个条件。

### 4.2.2、步骤4 向broker发送心跳信息
在上述启动流程中的第4步，调用<code class="my-code">sendHeartbeatToAllBrokerWithLock</code>方法，给每个Broker都发送一个心跳请求。
```java
this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
```
当Broker收到心跳请求后，将这个消费者注册到ConsumerManager中，前文提到，当Consumer数量变化时，Broker会主动通知其他消费者进行Rebalance。

而心跳的数据，这些数据是在MQClientInstance类的<code class="my-code">prepareHeartbeatData</code>方法来准备的。我们在前文通过mqadmin命令行工具的consumerConnection 自命令查看到的消费者订阅信息，在这里都出现了，如下图红色框所示
```java
private HeartbeatData prepareHeartbeatData() {
        HeartbeatData heartbeatData = new HeartbeatData();
        // clientID
        heartbeatData.setClientID(this.clientId);
        // Consumer
        for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
            MQConsumerInner impl = entry.getValue();
            if (impl != null) {
                ConsumerData consumerData = new ConsumerData();
                consumerData.setGroupName(impl.groupName());
                consumerData.setConsumeType(impl.consumeType());
                consumerData.setMessageModel(impl.messageModel());
                consumerData.setConsumeFromWhere(impl.consumeFromWhere());
                consumerData.getSubscriptionDataSet().addAll(impl.subscriptions());
                consumerData.setUnitMode(impl.isUnitMode());

                heartbeatData.getConsumerDataSet().add(consumerData);
            }
        }

        // Producer的心跳信息
        for (Map.Entry<String/* group */, MQProducerInner> entry : this.producerTable.entrySet()) {
            //略
        }
        return heartbeatData;
    }
```
提示：可以看到心跳数据<code class="my-code">HeartbeatData</code>中，既包含Consumer信息，也包含Producer信息(这里进行了省略)。


### 4.2.3、步骤5：立即触发一次Rebalance
消费者启动流程的最后一步是调用以下方法立即触发一次rebalance：
```java
this.mQClientFactory.rebalanceImmediately();
```
这个方法内部实际上，是通过唤醒一个<code class="my-code">RebalanceService</code>，来触发Rebalance：
```java
public void rebalanceImmediately() {
    this.rebalanceService.wakeup();
}
```
这里我们并不着急分析RebalanceService的内部具体实现，因为所有的Rebalance触发都是以这个类为入口，我们将在讲解完运行时/停止时的Rebalance触发时机后，统一进行说明。


## 4.3、运行时触发
消费者在运行时，通过两种机制来触发Rebalance：
- 监听broker 消费者数量变化通知，触发rebalance
- 周期性触发rebalance，避免Broker的Rebalance通知丢失。
  - 为了避免Broker的Rebalance通知丢失问题，客户端还会通过RebalanceService定时的触发Rebalance，默认间隔是20秒

## 4.4、停止时触发
最后，消费者在正常停止时，需要调用shutdown方法，这个方法的工作逻辑如下所示：
```java
public synchronized void shutdown(long awaitTerminateMillis) {
        switch (this.serviceState) {
            case CREATE_JUST:
                break;
            case RUNNING:
                this.consumeMessageService.shutdown(awaitTerminateMillis);
                this.persistConsumerOffset();
                this.mQClientFactory.unregisterConsumer(this.defaultMQPushConsumer.getConsumerGroup());
                this.mQClientFactory.shutdown();
                log.info("the consumer [{}] shutdown OK", this.defaultMQPushConsumer.getConsumerGroup());
                this.rebalanceImpl.destroy();
                this.serviceState = ServiceState.SHUTDOWN_ALREADY;
                break;
            case SHUTDOWN_ALREADY:
                break;
            default:
                break;
        }
    }
```
可以看到停止也分为5步，我们重点关注第2、3步：

在停止时，会首先通过第2步持久化offset，前文提到过默认情况下，offset是异步提交的，为了避免重复消费，因此在关闭时，必须要对尚未提交的offset进行持久化。其实就是发送更新offset请求(UPDATE_CONSUMER_OFFSET)给Broker，Broker对应更新ConsumerOffsetManager中的记录。这样当队列分配给其他消费者时，就可以从这个位置继续开始消费。

接着第3步调用unregisterConsumer方法，向所有broker发送UNREGISTER_CLIENT命令，取消注册Consumer。broker接收到这个命令后，将consumer从ConsumerManager中移除，然后通知这个消费者下的其他Consumer进行Rebalance。

至此，我们已经讲解完了Consumer启动时/运行时/停止时，所有可能的Rebalance触发时机，在下一小节，将介绍消费者Rebalance具体步骤。


## 4.5、Consumer Rebalance流程
前面花了大量的篇幅，讲解了Rebalance元数据维护，Broker通知机制，以及Consumer的Rebalance触发时机，目的是让读者有一个更高层面的认知，而不是直接分析单个Consumer Rebalance的具体步骤，避免一叶障目不见泰山。


### 4.5.1、Rebalance流程整体介绍
RocketMQ消息队列重新分布是由<code class="my-code">RebalanceService</code>线程实现的，一个MQClientInstance持有一个RebalanceService实现，并随着MQClientInstance的启动而启动。接下来我们带着上面的问题，了解一下RebalanceService的run()方法。
- RebalanceService 线程默认每隔 20s 执行一次 doRebalance() 方法。可以使用 -Drocketmq.client.rebalance.waitInterval=interval 改变默认配置。

```java
public class RebalanceService extends ServiceThread {
    private final MQClientInstance mqClientFactory;
    public RebalanceService(MQClientInstance mqClientFactory) {
        this.mqClientFactory = mqClientFactory;
    }

    @Override
    public void run() {
        log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            this.waitForRunning(waitInterval);
            this.mqClientFactory.doRebalance();
        }

        log.info(this.getServiceName() + " service end");
    }
```
不同的触发机制最终底层都调用了MQClientInstance的doRebalance方法，而在这个方法的源码中，并没有区分哪个消费者组需要进行Rebalance，只要任意一个消费者组需要Rebalance，这台机器上启动的所有其他消费者，也都要进行Rebalance。相关源码如下所示：
```java
// org.apache.rocketmq.client.impl.factory.MQClientInstance#doRebalance
    public void doRebalance() {
        // 遍历所有消费者
        for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
            MQConsumerInner impl = entry.getValue();
            if (impl != null) {
                try {
                    // 对消费者执行 doRebalance
                    impl.doRebalance();
                } catch (Throwable e) {
                    log.error("doRebalance exception", e);
                }
            }
        }
    }
```
doRebalance()方法中，会循环遍历所有注册的消费者信息，然后执行对应消费组的 doRebalance() 方法：
```java
// org.apache.rocketmq.client.impl.consumer.RebalanceImpl#doRebalance
    public void doRebalance(final boolean isOrder) {
        // 一个consumerGroup可以同时消费多个topic，所以此处返回的是一个Map，key 是 topic 名称
        Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
        if (subTable != null) {
            for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
                final String topic = entry.getKey();
                try {
                    this.rebalanceByTopic(topic, isOrder);
                } catch (Throwable e) {
                    if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                        log.warn("rebalanceByTopic Exception", e);
                    }
                }
            }
        }
        this.truncateMessageQueueNotMyTopic();
    }
```
RocketMQ按照Topic维度进行Rebalance，会导致一个很严重的结果：如果一个消费者组订阅多个Topic，可能会出现分配不均，部分消费者处于空闲状态。




# 五、小结
1. rebalance是消费者独有的机制，用于动态分配队列资源。

参考文章：
https://juejin.cn/post/7250374485568503867
https://juejin.cn/post/7031701457086709796
https://cloud.tencent.com/developer/article/1554950