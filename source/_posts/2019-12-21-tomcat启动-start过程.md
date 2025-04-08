---
title: 《tomcat》启动-start过程
date: 2019-12-21 20:19:03
categories:
   - [tomcat]
---

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

# 一、tomcat的启动过程
```text
exec "$PRGDIR/$EXECUTABLE" start "$@"
```
- `exec命令`：替换当前shell进程为`catalina.sh`，并传递参数（如start）
- `start参数`：指示`catalina.sh`以后台模式启动Tomcat

<!--more-->

## catalina.sh的核心流程

启动Tomcat
```text
if [ "$1" = "start" ]; then
  # 以后台模式启动
  nohup "$PRGDIR"/../bin/bootstrap.jar ...
elif [ "$1" = "run" ]; then
  # 以调试模式启动（前台）
  exec "$_RUNJAVA" ...
fi
```
- `nohup命令`：使Tomcat在后台运行，关闭终端后仍保持运行。
- `bootstrap.jar`：Java启动类 `org.apache.catalina.startup.Bootstrap`的入口。

## Java启动过程
初始化类加载器
```java
public void init() throws Exception {
    initClassLoaders(); // 初始化 Catalina 类加载器
    Thread.currentThread().setContextClassLoader(catalinaLoader);
}
```
启动catalina容器
```java
public static void main(String args[]) {
    synchronized (daemonLock) {
        Bootstrap daemon = new Bootstrap();
        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            daemon.load(args);
            daemon.start();
        }
    }
}
```
执行`Boostrap`的main方法，此时tomcat就在JVM作为一个线程启动了。启动过程主要有2个步骤
1. 资源初始化，`load()`，绑定`serverSocket`
2. 资源启动，`start()`，connector创建acceptor连接线程池。


# 二、start启动过程
执行start方法
```java
/**
 * start daemon.
 */
private void start(String[] arguments) throws Exception {
    // Call the load() method
    String methodName = "load";
    Method method = catalinaDaemon.getClass().getMethod(methodName, paramTypes);
```
通过反射，执行`Catalina.java`的start方法
```java
public void start() {
   if (getServer() == null) {
      load();
   }
   // Start the new server
   try {
      getServer().start();
   } catch (LifecycleException e) {
   }
}
```

## 2.1、StandardServer.start()
实际的server实现是`StandardServer`
```java
protected void startInternal() throws LifecycleException {
    // Start our defined Services
    synchronized (servicesLock) {
        for (int i = 0; i < services.length; i++) {
            services[i].start();
        }
    }
}
```
从server的启动上看，一个server有多个service

## 2.2、StandardService.start()
实际的service实现是`StandardService`
```java
protected void startInternal() throws LifecycleException {
    // Start our defined Container first
    if (engine != null) {
        synchronized (engine) {
            engine.start();
        }
    }
    synchronized (executors) {
        for (Executor executor: executors) {
            executor.start();
        }
    }
    mapperListener.start();
    // Start our defined Connectors second
    synchronized (connectorsLock) {
        for (Connector connector: connectors) {
            try {
                // If it has already failed, don't try and start it
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            } catch (Exception e) {
            }
        }
    }
}
```
主要有3个执行步骤：
1. service执行`engine`的启动。(engine继承Container，所以实际是进行container的初始化)
2. 然后线程池启动。`executor`是Tomcat自己实现的线程池。
3. 然后进行连接器(`connector`)的启动（一组连接器，server.xml中配置的多个connector，对应了端口号和协议）


## 2.3、engine.start()
执行`engine.start()`方法，最终只会执行`ContainerBase.startInternal()`方法，主要做了以下事情：
- 找到子容器，然后添加到线程中执行，`StartChild`内部的run方法就是调用`child.start()`方法。我们知道容器级别为：`Engine->Host->Context->Wrapper`，所以会产生`递归调用`，遍历着把子容器都给`start`一下。然后调用`Future.get`方法，等待子容器处理完start方法。
- 调用Pipeline（实现类为`StandardPipeline`）的start方法。
- 启动后台线程。后台线程会定时扫描，实现热加载和热部署，是一个单线程。虽然方法是在`ContainerBase`里，但是启动后台线程是要有条件的，必须`backgroundProcessorDelay`大于0，该参数默认为-1，只有`StandardEngine`中设置成了10

```java
//ContainerBase#startInternal
protected synchronized void startInternal() throws LifecycleException {
    //1. 找到子容器，然后调用start方法
    Container children[] = findChildren();
    List<Future<Void>> results = new ArrayList<>();
    for (Container child : children) {
        results.add(startStopExecutor.submit(new StartChild(child)));
    }
    for (Future<Void> result : results) {
        result.get();
    }
    //2. 
    if (pipeline instanceof Lifecycle) {
        ((Lifecycle) pipeline).start();
    }
    //3. 启动容器后台线程，做热加载和热更新
    threadStart();
}

private static class StartChild implements Callable<Void> {
   private Container child;
   public StartChild(Container child) {
      this.child = child;
   }
   @Override
   public Void call() throws LifecycleException {
      child.start();
      return null;
   }
}
```

### 2.3.1、host.start()
Host（实现类为StandardHost）中的start没有特殊的，会调用父类ContainerBase的startInternal方法，会继续找子容器调用start方法

### 2.3.2、Context.start()
在`Context`容器中，start方法的执行逻辑会比较复杂：会去解析`web.xml`文件，
`StandardContext.startInternal()`启动过程中，触发`Lifecycle.CONFIGURE_START_EVENT` 事件，简化后代码如下：
```java
protected synchronized void startInternal() throws LifecycleException {
   //1、通知容器启动事件，去解析web.xml
   fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);
   //Start our child containers, if not already started
   for (Container child : findChildren()) {
      if (!child.getState().isAvailable()) {
         child.start();
      }
   }
   //2、进行listener的创建和启动
   if (ok) {
     if (!listenerStart()) {
         log.error(sm.getString("standardContext.listenerFail"));
         ok = false;
     }
   }
   // 3、filter的创建和启动
   if (ok) {
     if (!filterStart()) {
         log.error(sm.getString("standardContext.filterFail"));
         ok = false;
     }
   }
   //加载和实例化servlet
   // Load and initialize all "load on startup" servlets
   if (ok) {
     if (!loadOnStartup(findChildren())){
         log.error(sm.getString("standardContext.servletFail"));
         ok = false;
     }
   }
}
```

创建一个Loader，内部会包含一个类加载器，并且调用start方法。
```java
//StandardContext#startInternal
WebappLoader webappLoader = new WebappLoader();
webappLoader.setDelegate(getDelegate());
setLoader(webappLoader);

Loader loader = getLoader();
if (loader instanceof Lifecycle) {
    ((Lifecycle) loader).start();
}
```
找子容器（Wrapper），然后调用start方法。
```java
for (Container child : findChildren()) {
    if (!child.getState().isAvailable()) {
        child.start();
    }
}
```
调用`ServletContainerInitializer`接口的`onStartup`方法
```java
for (Map.Entry<ServletContainerInitializer, Set<Class<?>>> entry :initializers.entrySet()) {
    entry.getKey().onStartup(entry.getValue(),getServletContext());
}
```
找子容器需要在启动的时候load的，子容器Wrapper内部加载Servlet，并调用`Servlet#init`方法。这种就是在启动的时候加载，默认情况下，是延迟加载。
```java
loadOnStartup(findChildren());
```
`Wrapper`容器中，并没有特殊的地方。

### 2.3.3、wrapper.start()
```java
public class StandardWrapper extends ContainerBase
    implements ServletConfig, Wrapper, NotificationEmitter {
   @Override
   protected synchronized void startInternal() throws LifecycleException {
      // Send j2ee.state.starting notification
      if (this.getObjectName() != null) {
         Notification notification = new Notification("j2ee.state.starting",
                 this.getObjectName(),
                 sequenceNumber++);
         broadcaster.sendNotification(notification);
      }
      // Start up this component
      super.startInternal();

      setAvailable(0L);
      // Send j2ee.state.running notification
      if (this.getObjectName() != null) {
         Notification notification =
                 new Notification("j2ee.state.running", this.getObjectName(),
                         sequenceNumber++);
         broadcaster.sendNotification(notification);
      }
   }
 }
```

## 2.4、Executor.startInternal()
```java
protected void startInternal() throws LifecycleException {
    taskqueue = new TaskQueue(maxQueueSize);
    TaskThreadFactory tf = new TaskThreadFactory(namePrefix,daemon,getThreadPriority());
    executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), maxIdleTime, TimeUnit.MILLISECONDS,taskqueue, tf);
    executor.setThreadRenewalDelay(threadRenewalDelay);
    if (prestartminSpareThreads) {
        executor.prestartAllCoreThreads();
    }
    taskqueue.setParent(executor);
    setState(LifecycleState.STARTING);
}
```
1. 创建任务队列
2. 创建工作线程

## 2.5、Connector.startInternal()
连接器的启动
```java
protected void startInternal() throws LifecycleException {
    try {
        protocolHandler.start();//启动协议处理器
    } catch (Exception e) {
   }
}
```
   
## 2.6、protocolHandler.start()
协议处理器的启动，`org.apache.coyote.http11.Http11NioProtocol`的`start()`方法，最终执行`AbstractHProtocal.start()`方法。
```java
@Override
public void start() throws Exception {
   endpoint.start();
}
```

## 2.7、endpoint.start()
终端的启动，最终执行`NIOEndpoint`的`startInternal()`方法。
```java
@Override
public void startInternal() throws Exception {
    if (!running) {
        processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE, socketProperties.getProcessorCache());
        eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE, socketProperties.getEventCache());
        nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE, socketProperties.getBufferPool());
        // Create worker collection
        if ( getExecutor() == null ) {
            createExecutor();
        }
        initializeConnectionLatch();
        // Start poller threads【轮询线程】
        pollers = new Poller[getPollerThreadCount()];
        for (int i=0; i<pollers.length; i++) {
            pollers[i] = new Poller();
            Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();
        }
        //开启接收线程
        startAcceptorThreads();
    }
}
//创建接收线程
protected final void startAcceptorThreads() {
   int count = getAcceptorThreadCount();
   acceptors = new Acceptor[count];
   for (int i = 0; i < count; i++) {
      acceptors[i] = createAcceptor();
      String threadName = getName() + "-Acceptor-" + i;
      acceptors[i].setThreadName(threadName);
      Thread t = new Thread(acceptors[i], threadName);
      t.setPriority(getAcceptorThreadPriority());
      t.setDaemon(getDaemon());
      t.start();
   }
}
```
1. 终端的启动，首先创建轮询线程(一般为1个，轮询selector多路复用器)
    * poller是轮训selector的线程
2. 然后启动接收线程【连接池线程数量】

<code class="red">Acceptor线程启动了</code>，此时就已经可以监听连接消息了。

### 2.7.1、NIOEndpoint.Acceptor
接收线程：`NIOEndpoint.Acceptor`
```java
protected class Acceptor extends AbstractEndpoint.Acceptor {
    @Override
    public void run() {
        int errorDelay = 0;
        // Loop until we receive a shutdown command
        while (running) {
            state = AcceptorState.RUNNING;
            try {
                //if we have reached max connections, wait
                countUpOrAwaitConnection();
                AsynchronousSocketChannel socket = null;
                try {
                   //阻塞，直到有新连接到达
                    socket = serverSock.accept().get();
                } catch (Exception e) {
                }
                // Configure the socket
                if (running && !paused) {
                   //处理连接，生成PollEvent，添加到Poller线程的events同步队列里
                    if (!setSocketOptions(socket)) {
                        closeSocket(socket);
                   }
                } else {
                    closeSocket(socket);
                }
            } catch (Throwable t) {
            }
        }
        state = AcceptorState.ENDED;
    }
}
```
接收socket连接请求

### 2.7.2、处理请求：setSocketOptions
```java
protected boolean setSocketOptions(SocketChannel socket) {
    // Process the connection
    try {
        //disable blocking, APR style, we are gonna be polling it
        socket.configureBlocking(false);
        Socket sock = socket.socket();
        socketProperties.setProperties(sock);
        NioChannel channel = nioChannels.pop();
        if (channel == null) {
            SocketBufferHandler bufhandler = new SocketBufferHandler(
                    socketProperties.getAppReadBufSize(),
                    socketProperties.getAppWriteBufSize(),
                    socketProperties.getDirectBuffer());
            if (isSSLEnabled()) {
                channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
            } else {
                channel = new NioChannel(socket, bufhandler);
            }
        } else {
            channel.setIOChannel(socket);
            channel.reset();
        }
        //将channer注册进Poller，poller相当于selector
        getPoller0().register(channel);
    } catch (Throwable t) {
    }
    return true;
}
//注册过程，创建PollerEvent，将事件注册进Poller
public void register(final NioChannel socket) {
    socket.setPoller(this);
    NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
    socket.setSocketWrapper(ka);
    ka.setPoller(this);
    ka.setReadTimeout(getSocketProperties().getSoTimeout());
    ka.setWriteTimeout(getSocketProperties().getSoTimeout());
    ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
    ka.setSecure(isSSLEnabled());
    ka.setReadTimeout(getConnectionTimeout());
    ka.setWriteTimeout(getConnectionTimeout());
    PollerEvent r = eventCache.pop();
    ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
    if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
    else r.reset(socket,ka,OP_REGISTER);
    addEvent(r);
}
```

### 2.7.3、Poller轮询：
```java
public void run() {
    // Loop until destroy() is called
    while (true) {
        boolean hasEvents = false;
        Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        // Walk through the collection of ready keys and dispatch
        // any active event.
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
            // Attachment may be null if another thread has called
            // cancelledKey()
            if (attachment == null) {
                iterator.remove();
            } else {
                iterator.remove();
                processKey(sk, attachment);
            }
        }//while
    }//while
    getStopLatch().countDown();
}
```

### 2.7.3、processKey
```java
protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
    try {
        if ( close ) {
            cancelledKey(sk);
        } else if ( sk.isValid() && attachment != null ) {
            if (sk.isReadable() || sk.isWritable() ) {
                if ( attachment.getSendfileData() != null ) {
                    processSendfile(sk,attachment, false);
                } else {
                    unreg(sk, attachment, sk.readyOps());
                    boolean closeSocket = false;
                    // Read goes before write
                    if (sk.isReadable()) {
                        if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                            closeSocket = true;
                        }
                    }
                    if (!closeSocket && sk.isWritable()) {
                        if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                            closeSocket = true;
                        }
                    }
                    if (closeSocket) {
                        cancelledKey(sk);
                    }
                }
            }
        } else {
            //invalid key
            cancelledKey(sk);
        }
    } catch ( CancelledKeyException ckx ) {
        cancelledKey(sk);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error("",t);
    }
}
```
processSocket
```java
public boolean processSocket(SocketWrapperBase<S> socketWrapper,
        SocketEvent event, boolean dispatch) {
    try {
        if (socketWrapper == null) {
            return false;
        }
        SocketProcessorBase<S> sc = processorCache.pop();
        if (sc == null) {
            sc = createSocketProcessor(socketWrapper, event);
        } else {
            sc.reset(socketWrapper, event);
        }
        Executor executor = getExecutor();
        if (dispatch && executor != null) {
            executor.execute(sc);
        } else {
            sc.run();
        }
    } catch (RejectedExecutionException ree) {
        getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
        return false;
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        // This means we got an OOM or similar creating a thread, or that
        // the pool and its queue are full
        getLog().error(sm.getString("endpoint.process.fail"), t);
        return false;
    }
    return true;
}
```


具体执行过程
```java
protected class SocketProcessor extends SocketProcessorBase<NioChannel> {
    @Override
    protected void doRun() {
        NioChannel socket = socketWrapper.getSocket();
        SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
        try {
            int handshake = -1;
            try {
                if (key != null) {
                    if (socket.isHandshakeComplete()) {
                        // No TLS handshaking required. Let the handler
                        // process this socket / event combination.
                        handshake = 0;
                    } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                            event == SocketEvent.ERROR) {
                        // Unable to complete the TLS handshake. Treat it as
                        // if the handshake failed.
                        handshake = -1;
                    } else {
                        handshake = socket.handshake(key.isReadable(), key.isWritable());
                        // The handshake process reads/writes from/to the
                        // socket. status may therefore be OPEN_WRITE once
                        // the handshake completes. However, the handshake
                        // happens when the socket is opened so the status
                        // must always be OPEN_READ after it completes. It
                        // is OK to always set this as it is only used if
                        // the handshake completes.
                        event = SocketEvent.OPEN_READ;
                    }
                }
            } catch (IOException x) {
                handshake = -1;
                if (log.isDebugEnabled()) log.debug("Error during SSL handshake",x);
            } catch (CancelledKeyException ckx) {
                handshake = -1;
            }
            if (handshake == 0) {
                SocketState state = SocketState.OPEN;
                // Process the request from this socket
                if (event == null) {
                    state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                } else {
                    state = getHandler().process(socketWrapper, event);
                }
                if (state == SocketState.CLOSED) {
                    close(socket, key);
                }
            } else if (handshake == -1 ) {
                getHandler().process(socketWrapper, SocketEvent.CONNECT_FAIL);
                close(socket, key);
            } else if (handshake == SelectionKey.OP_READ){
                socketWrapper.registerReadInterest();
            } else if (handshake == SelectionKey.OP_WRITE){
                socketWrapper.registerWriteInterest();
            }
        } catch (CancelledKeyException cx) {
            socket.getPoller().cancelledKey(key);
        } catch (VirtualMachineError vme) {
            ExceptionUtils.handleThrowable(vme);
        } catch (Throwable t) {
            log.error("", t);
            socket.getPoller().cancelledKey(key);
        } finally {
            socketWrapper = null;
            event = null;
            //return to cache
            if (running && !paused) {
                processorCache.push(this);
            }
        }
    }
}
```

getHandler().process
```java
AbstractProtocol
state = processor.process(wrapper, status);
AbstractProtocal类中，获取processor后去执行任务
```

processor.process----AbstractProcessorLight
```java
public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
        throws IOException {
    SocketState state = SocketState.CLOSED;
    Iterator<DispatchType> dispatches = null;
    do {
        //判断状态
        } else if (status == SocketEvent.OPEN_READ) {
            state = service(socketWrapper);
        }
    } while (state == SocketState.ASYNC_END ||
            dispatches != null && state != SocketState.CLOSED);

    return state;
}
```
service ----Http11Processor
```java
getAdapter().service(request, response);
```


adapter.service ---- CoyoteAdapter
```java
// Parse and set Catalina and configuration specific
// request parameters
postParseSuccess = postParseRequest(req, request, res, response);
// Calling the container
connector.getService().getContainer().getPipeline().getFirst().invoke(
        request, response);
```
1. 根据请求信息解析, 将对应的Host, Context, Wrapper容器对象封装到request实例中(重点)
2. 调用StandardEngine的pipeline对Request,Response进行处理, StandardHostValve保存在pipeline的first属性中


postParseRequest
```java
connector.getService().getMapper().map(serverName, decodedURI,
        version, request.getMappingData());
```


# 启动过程总结：
1. Load阶段已经绑定了端口号
2. start启动阶段，需要启动轮询线程、创建工作线程、连接池线程




参考文章：
https://blog.csdn.net/qq_38975553/article/details/103443321


