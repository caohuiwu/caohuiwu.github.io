---
title: 《zookeeper》运行机制
date: 2020-09-05
categories:
  - [ zookeeper, ZAB ]
  - [ zookeeper, watch ]
  - [ zookeeper, 脑裂 ]
---

    这是“zookeeper”系列的第二篇文章，主要介绍的是相关协议及内部原理。

<style>
.my-code {
   color: green;
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

# 一、zookeeper
<code>zookeeper</code>是一个分布式协调服务框架，它是一个为分布式应用提供一致性服务的软件。
- 主要用于实现分布式系统中master选举、分布式协调、集群管理、负载均衡、分布式锁等功能。
- 可以用来统一配置管理、统一命名服务、分布式锁、集群管理

<!-- more -->

# 二、ZAB协议
ZAB（`Zookeeper Atomic Broadcast`，zk原子广播协议）协议是zk实现分布式协调服务的关键，确保集群中各个节点的数据一致性。

## 2.1、概念介绍
在介绍zab协议之前首先要知道zookeeper相关的几个概念，才能更好的了解zab协议。

### 2.1.1、集群角色
- `Leader`：同一时间集群总只允许有一个Leader，提供对客户端的读写功能，负责将数据同步至各个节点；
- `Follower`：提供对客户端读功能，写请求则转发给`Leader`处理，当`Leader`崩溃失联之后参与`Leader`选举；
- `Observer`：与`Follower`不同的是但不参与`Leader`选举。

### 2.1.2、服务状态
- `LOOKING`：当节点认为群集中没有`Leader`，服务器会进入LOOKING状态，目的是为了查找或者选举Leader；
- `FOLLOWING`：follower角色；
- `LEADING`：leader角色；
- `OBSERVING`：observer角色；

### 2.1.3、ZXID
`Zxid（Zookeeper transaction ID）`是极为重要的概念，用于唯一标识事务的全局有序ID，它是一个long型（64位）整数，分为两部分：
- 纪元（epoch）部分
- 计数器（counter）部分，是一个全局有序的数字。

epoch代表当前集群所属的哪个leader，leader的选举就类似一个朝代的更替，你前朝的剑不能斩本朝的官，用epoch代表当前命令的有效性，counter是一个递增的数字。

zxid的生成和解析逻辑主要集中在以下类中：

#### 2.1.3.1、ZxidUtils
```java
public static long makeZxid(int epoch, int counter) {
    return ((long)epoch << 32) | (counter & 0xffffffffL);
}

public static int getEpochFromZxid(long zxid) {
    return (int)(zxid >> 32);
}

public static int getCounterFromZxid(long zxid) {
    return (int)zxid;
}
```

#### 2.1.3.2、leader节点生成zxid
Leader在广播事务前生成Zxid
```java
// Leader 初始化时设置 epoch
private long epoch = 1; // 初始 epoch 由选举结果决定

// 生成新 Zxid
public long getNextZxid() {
    // 递增 counter，高 32 位为 epoch
    return ZxidUtils.makeZxid(epoch, counter.incrementAndGet());
}
```

## 2.2、ZAB的状态
Zookeeper还给ZAB定义的4中状态，反应Zookeeper从选举到对外提供服务的过程中的四个步骤。状态枚举定义：
```java
public enum ZabState {
    ELECTION,
    DISCOVERY,
    SYNCHRONIZATION,
    BROADCAST
}
```
- `ELECTION`: 集群进入选举状态，此过程会选出一个节点作为leader角色；
- `DISCOVERY`：连接上leader，响应leader心跳，并且检测leader的角色是否更改，通过此步骤之后选举出的leader才能执行真正职务；
- `SYNCHRONIZATION`：整个集群都确认leader之后，将会把leader的数据同步到各个节点，保证整个集群的数据一致性；
- `BROADCAST`：过渡到广播状态，集群开始对外提供服务。

## 2.2、ZAB协议的两种模式
ZAB协议主要包括两个模式。
- 消息广播模式
- 崩溃恢复模式

### 2.2.1、崩溃恢复模式（recovery phase）
- 触发条件：集群启动或Leader崩溃。
- 目标：选举新Leader，同步数据到最新状态。

#### 2.2.1.1、崩溃恢复过程
当Leader节点登记或出现其他故障时，集群进入崩溃恢复模式，过程如下：
- 选举阶段
- 发现阶段
- 同步阶段

##### 1、选举阶段
- 每个节点初始化时都处于选举状态（LOOKING）。
- 节点向其他节点发送投票信息，包含（节点ID，ZXID）
- 节点收到其他节点的投票后，根据（ZXID，节点ID）的优先级更新自己的投票。
- 当某个节点的投票获得过半节点的认可时，该节点成为Leader。

##### 2、发现阶段
- Leader收集所有的Follower的ZXID，确定最新的ZXID。
- Leader生成新的epoch，并将epoch广播给所有Follower

##### 3、同步阶段
- Leader根据每个Follower的数据状态，发送相应的同步指令
- Follower根据指令进行数据同步，确保与Leader的数据一致。

### 2.2.2、消息广播模式（broadcast phase）
- 触发条件：完成Leader选举后
- 目标：将客户端请求（事务）有序广播到所有节点。

#### 2.2.2.1、消息广播过程
集群正常运行时，采用消息广播模式（两阶段提交 2PC）保证数据一致性，过程如下：

##### 1、Proposal阶段
1. 请求接收与分配ZXID：Leader节点接收客户端的写请求，并为每个请求分配一个全局唯一且递增的ZXID（事务ID）。
2. 广播事务Proposal：Leader将事务封装成Proposal消息，广播给所有的Follower节点。

##### 2、ACK阶段
3. Follower响应：Follower节点接收到Proposal后，将其写入本地事务日志，并向Leader发送ACK响应。

##### 3、Commit阶段
4. 事务提交：当Leader收到超过半数Follower的ACK响应后，认为事务可以提交，向所有Follower发送Commit消息。
5. Follower提交事务：Follower接收到Commit消息后，完成事务的提交。



# 三、watch机制
Zookeeper的watch机制是其实现分布式协调功能的核心特性之一，允许客户端监听指定节点（ZNode）的变化，并在事件发生时接收异步通知。以下是Watch机制的详细解析：

## 3.1、Watch的工作流程
### 3.1.1、客户端注册Watch
```java
// Java 示例：监听节点数据变化
Stat stat = new Stat();
byte[] data = zk.getData("/myNode", watchedEvent -> {
    System.out.println("Watch 触发: " + watchedEvent.getType());
}, stat);
```

### 3.1.2、服务端记录Watch
服务端将Watch信息（路径、事件类型、客户端会话）存储在内存中。
```java
// 添加watcher的示例代码
public Stat setData(String path, byte data[], int version, long zxid, long time) throws KeeperException.NoNodeException {
    // ...
    watchManager.triggerWatch(path, EventType.NodeDataChanged);
    return s;
}
```

### 3.1.3、事件触发与通知
当节点发生符合条件的变化时，服务端生成WatchedEvent，通过客户端的Watcher回调通知。
```java
// 触发watcher的示例代码
public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
        watchers = watchTable.remove(path);
        if (watchers == null || watchers.isEmpty()) {
            return null;
        }
        for (Watcher w : watchers) {
            HashSet<String> paths = watch2Paths.get(w);
            if (paths != null) {
                paths.remove(path);
            }
        }
    }
    for (Watcher w : watchers) {
        if (supress != null && supress.contains(w)) {
            continue;
        }
        w.process(e);
    }
    return watchers;
}
```

### 3.1.4、客户端处理事件
客户端收到事件后，需根据业务逻辑重新注册Watch。

## 3.2、Watch的服务端实现
在ZooKeeper服务端，watcher的记录和管理主要通过`WatchManager`类来实现。以下是相关源码的简要说明：

### 3.2.1、 **数据结构**
- `WatchManager`内部维护了多个哈希表，用于记录不同事件类型（如`NodeDataChanged`、`NodeDeleted`等）的watcher。
- 主要数据结构包括`watchTable`、`dataWatches`、`existWatches`和`childWatches`，分别用于存储节点数据变化、节点存在性变化以及子节点变化的watcher。

```java
// 服务端记录watch的主要数据结构
private final WatchManager watchManager;
private final Map<String, HashSet<Watcher>> watchTable = new HashMap<String, HashSet<Watcher>>();
private final Map<String, Set<Watcher>> dataWatches = new HashMap<String, Set<Watcher>>();
private final Map<String, Set<Watcher>> existWatches = new HashMap<String, Set<Watcher>>();
private final Map<String, Set<Watcher>> childWatches = new HashMap<String, Set<Watcher>>();
```

### 3.2.2、 **添加watcher**
- 当客户端注册watcher时，服务端会将watcher添加到相应的哈希表中。
- 例如，在`DataTree`类的`setData`方法中，会调用`watchManager.triggerWatch`来添加watcher。

```java
// 添加watcher的示例代码
public Stat setData(String path, byte data[], int version, long zxid, long time) throws KeeperException.NoNodeException {
    // ...
    watchManager.triggerWatch(path, EventType.NodeDataChanged);
    return s;
}
```

### 3.2.3、**触发watcher**
- 当节点状态发生变化时（如数据更新、节点删除等），服务端会调用`WatchManager`的`triggerWatch`方法来触发相应的watcher。
- `triggerWatch`方法会从哈希表中获取所有相关的watcher，并调用它们的`process`方法。

```java
// 触发watcher的示例代码
public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
        watchers = watchTable.remove(path);
        if (watchers == null || watchers.isEmpty()) {
            return null;
        }
        for (Watcher w : watchers) {
            HashSet<String> paths = watch2Paths.get(w);
            if (paths != null) {
                paths.remove(path);
            }
        }
    }
    for (Watcher w : watchers) {
        if (supress != null && supress.contains(w)) {
            continue;
        }
        w.process(e);
    }
    return watchers;
}
```
客户端使用 SendThread.readResponse() 方法来统一处理服务端的相应：

#### 1、process示例
在ZooKeeper中，`process`方法是`Watcher`接口的一部分，用于处理收到的事件通知。以下是一个简化的`process`方法示例，展示了如何处理不同类型的事件：

```java
public class MyWatcher implements Watcher {
    @Override
    public void process(WatchedEvent event) {
        System.out.println("收到事件通知：" + event);
        
        // 检查事件的状态
        switch (event.getState()) {
            case SyncConnected:
                // 客户端与服务端成功连接
                System.out.println("客户端已连接到ZooKeeper服务器");
                break;
            case Disconnected:
                // 客户端与服务端断开连接
                System.out.println("客户端与ZooKeeper服务器断开连接");
                break;
            case AuthFailed:
                // 身份验证失败
                System.out.println("身份验证失败，无法连接到ZooKeeper服务器");
                break;
            case Expired:
                // 会话过期
                System.out.println("会话已过期，需要重新连接");
                break;
            // 其他状态...
        }
        
        // 检查事件的类型
        switch (event.getType()) {
            case None:
                // 无事件类型
                break;
            case NodeCreated:
                // 节点被创建
                System.out.println("节点创建：" + event.getPath());
                // 处理节点创建的逻辑
                break;
            case NodeDeleted:
                // 节点被删除
                System.out.println("节点删除：" + event.getPath());
                // 处理节点删除的逻辑
                break;
            case NodeDataChanged:
                // 节点数据发生变化
                System.out.println("节点数据变化：" + event.getPath());
                // 处理节点数据变化的逻辑
                break;
            case NodeChildrenChanged:
                // 子节点列表发生变化
                System.out.println("子节点列表变化：" + event.getPath());
                // 处理子节点列表变化的逻辑
                break;
            // 其他事件类型...
        }
    }
}
```

在这个示例中，`MyWatcher`类实现了`Watcher`接口，并重写了`process`方法。当收到事件通知时，`process`方法会根据事件的状态和类型执行相应的逻辑。例如，如果收到`NodeDataChanged`事件，可能会重新获取节点数据并更新本地状态。

需要注意的是，这个示例是一个简化版本，实际使用中可能需要根据具体业务需求进行更复杂的处理。此外，由于ZooKeeper的watcher是一次性的，触发后会自动移除，因此如果需要在`process`方法中重新注册watcher，需要确保在适当的时机进行注册。


### 3.2.4、 **移除watcher**
- 当watcher被触发后，服务端会从哈希表中移除该watcher，确保每个watcher只被触发一次。

通过上述机制，服务端能够高效地记录、触发和移除watcher，实现客户端对节点状态变化的实时感知。




## 3.3、Watch的核心特性

### 3.3.1、一次性触发
- Watch被触发后自动失效，需要重新注册才能继续监听后续变化。
- 设计原因：减少服务端资源占用，避免因客户端未处理事件导致的堆积。

### 3.3.2、事件类型
- 节点事件：NodeCreated, NodeDeleted, NodeDataChanged, NodeChildrenChanged

### 3.3.3、有效性保证
- 客户端先看到数据变化，再收到Watch事件通知。
- 事件通知的顺序和数据变更顺序一致。


# 四、惊群
## 4.1、ZK 中惊群问题的根源
1. 事件通知机制
ZooKeeper 使用了一种基于事件的通知机制，当节点数据发生变化、子节点列表发生变化或者其他特定事件发生时，会向注册了监听该节点的客户端发送通知。
2. 客户端行为模式

## 4.2、应对策略的具体实现示例
- 合理设计应用逻辑
以一个分布式缓存系统为例，多个缓存节点（客户端）监听 ZooKeeper 上的缓存配置节点变化。当配置发生变化时，每个客户端在收到通知后，首先检查自己的本地缓存状态和当前系统负载情况。如果本地缓存已经是最新的或者系统负载过高，就不进行立即的数据更新操作，而是等待一段时间后再次评估是否需要更新，这样可以避免不必要的处理
- 使用排队机制
在一个分布式锁的实现中，客户端在等待获取锁时，可以将自己加入一个等待队列中，并按照顺序依次尝试获取锁。当锁释放时，只有队列头部的客户端进行获取锁的操作，如果获取失败，它会重新进入队列尾部等待。其他客户端则继续等待，而不是同时进行竞争。这样可以有效地减少竞争和资源浪费。例如，使用 Java 语言可以通过自定义的数据结构和同步机制来实现这样的排队逻辑
- 优化客户端数量和监听策略
对于一个大型的分布式监控系统，根据监控的层次和功能将客户端进行分组。比如，将负责监控服务器硬件状态的客户端分为一组，监听应用程序性能指标的客户端分为另一组，不同组的客户端监听不同的 ZooKeeper 节点或者节点层次。这样可以减少同时被通知的客户端数量，降低惊群问题的影响。同时，可以根据实际的业务需求和系统负载动态调整客户端的分组和监听策略。


# 五、脑裂
`脑裂`指分布式系统中，由于网络分区（network partition）导致集群被分割成多个独立子集群，每个子集群误认为自己是唯一存活的部分，并继续对外提供服务，导致数据不一致和冲突。

例如5台机器，部署在2个机房，此时网络出现故障，可能导致2个leader，从而脑裂

## 5.1、ZK如何解决脑裂？
Zookeeper通过ZAB协议和集群配置规则避免脑裂，核心机制如下：

### 5.1.1、quorum（多数派原则）
- 选举要求：只有获得半数以上（N/2+1）节点投票的候选者才能成为Leader。
- 写操作要求：事务提交需要半数以上节点确认。
- 结果：若集群被分割为两部分，仅包含多数节点的子集群能继续工作，另外一部分因无法达到多数而停止服务。

### 5.1.2、Epoch机制
- 任期编号：每个Leader分配唯一的epoch值（递增整数）。
- 拒绝旧Leader：新Leader广播更高的epoch，旧Leader的请求会被其他节点拒绝。

### 5.1.3、节点配置为奇数
推荐节点数：3、5、7等奇数，确保任何网络分区下，至多一个子集群能形成多数。

