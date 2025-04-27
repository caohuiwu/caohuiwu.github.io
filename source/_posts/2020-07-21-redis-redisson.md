---
title: 《Redis》redisson
date: 2020-07-21
categories:
  - [redis]
tags:
  - [redisson]
---

<style>
.my-code {
   color: red;
}
.orange {
   color: orange
}
.red {
   color: red
}
code {
   color: #0ABF5B;
}
</style>

    这是“Redis”系列的第十篇文章，主要介绍的是redisson的分布式锁实现。

# 一、redisson
`Redisson`是基于`Redis`构建的Java内存数据网格（`in-memory data grid`），提供了一系列分布式服务和数据结构的实现，尤其在分布式锁、分布式集合、分布式服务等方面表现突出。它通过封装Redis的原生操作，为Java开发者提供了更易用的API。
<!-- more -->


# 二、使用示例

## 2.1、单节点配置与简单加锁
```java
import org.redisson.Redisson;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
public class RedissonLockExample {
    public static void main(String[] args) {
        // 1. 配置Redisson客户端（单节点模式）
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        // 创建客户端
        RedissonClient redisson = Redisson.create(config);
        // 2. 获取锁对象（默认非公平锁）
        RLock lock = redisson.getLock("myLock");
        try {
            // 3. 尝试加锁（无超时，自动启用看门狗机制）
            lock.lock();
            System.out.println("获取到锁，开始执行业务逻辑...");
            // 模拟业务执行时间（假设需要5秒）
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 4. 释放锁（无论成功与否必须调用）
            lock.unlock();
            System.out.println("释放锁，业务逻辑执行完毕。");
        }
        // 5. 关闭Redisson客户端
        redisson.shutdown();
    }
}
```

## 2.2、带超时控制的加锁示例
```java
import java.util.concurrent.TimeUnit;
public class TimedLockExample {
    public static void main(String[] args) {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        RedissonClient redisson = Redisson.create(config);
        RLock lock = redisson.getLock("timedLock");
        try {
            // 尝试加锁：最多等待10秒（waitTime），锁自动释放时间为30秒（leaseTime）
            boolean isLocked = lock.tryLock(10, 30, TimeUnit.SECONDS);
            if (isLocked) {
                System.out.println("成功获取锁，开始执行业务...");
                // 业务逻辑
                Thread.sleep(5000);
            } else {
                System.out.println("获取锁超时，放弃执行");
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 安全释放锁（即使未获取到锁，调用unlock也不会报错）
            lock.unlock();
        }
        redisson.shutdown();
    }
}
```

| 方法/参数                        | 说明                                        |
|------------------------------|-------------------------------------------|
| lock()                       | 无超时加锁，自动启用看门狗机制（默认锁有效期30s，续期间隔10秒）        |
| lock(leaseTime, unit)        | 指定锁的过期时间，禁用看门狗（锁在leaseTime后自动过期）          |
| tryLock()                    | 尝试立即加锁，失败直接返回false，默认启用看门狗                |
| tryLock(waitTime, leaseTime) | 最多等待waitTime时间获取锁，成功后锁有效期为leaseTime，禁用看门狗 |


# 三、看门狗解析
加锁方法逻辑如下：
```java
public class RedissonLock extends RedissonBaseLock {
    @Override
    public void lock() {
        try {
            lock(-1, null, false);
        } catch (InterruptedException e) {
            throw new IllegalStateException();
        }
    }
    private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
        long threadId = Thread.currentThread().getId();
        //尝试获取锁
        Long ttl = tryAcquire(-1, leaseTime, unit, threadId);
        // lock acquired，成功获取锁
        if (ttl == null) {
            return;
        }
        //未获取到锁，订阅锁释放通知
        CompletableFuture<RedissonLockEntry> future = subscribe(threadId);
        pubSub.timeout(future);//为订阅操作设置超时时间，防止无限期等待
        RedissonLockEntry entry;
        //同步订阅（处理中断）
        if (interruptibly) {
            //线程在等待订阅结果时，若被中断，抛出InterruptedException。适用于需要响应外部中断的场景（如超时控制或紧急退出）
            entry = commandExecutor.getInterrupted(future);
        } else {
            //禁止中断，线程会一直阻塞直到订阅完成或超时。忽略中断信号
            entry = commandExecutor.get(future);
        }
        try {
            //循环重试获取锁
            while (true) {
                ttl = tryAcquire(-1, leaseTime, unit, threadId);
                // 获取成功
                if (ttl == null) {
                    break;
                }
                // 根据锁的剩余时间阻塞等待
                if (ttl >= 0) {
                    //得到一个Semaphore信号量对象，这是jdk的内置对象，Semaphore#tryAcquire表示阻塞并等待唤醒
                    try {
                        entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    } catch (InterruptedException e) {
                        if (interruptibly) {
                            throw e;
                        }
                        entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    }
                } else {
                    if (interruptibly) {
                        entry.getLatch().acquire();
                    } else {
                        entry.getLatch().acquireUninterruptibly();
                    }
                }
            }
        } finally {
            unsubscribe(entry, threadId);//取消订阅
        }
//        get(lockAsync(leaseTime, unit));
    }
}
```

## 3.1、关键步骤解析

### 3.1.1、尝试获取锁：`tryAcquire()`
尝试获取锁的核心方法：
```java
private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    return get(tryAcquireAsync(waitTime, leaseTime, unit, threadId));
}
 
private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    RFuture<Long> ttlRemainingFuture;
    if (leaseTime > 0) {
        // 调用lua脚本，尝试加锁
        ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
    } else {
        // 这里的if、else的区别就在于，如果没有设置leaseTime，就使用默认的internalLockLeaseTime（默认30秒）
        ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
                TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
    }
    CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {
        // ttlRemaining：剩余过期时间
        // 如果ttlRemaining为空，也就是tryLockInnerAsync方法中的lua执行结果返回空，证明获取锁成功
        if (ttlRemaining == null) {
            if (leaseTime > 0) {
                internalLockLeaseTime = unit.toMillis(leaseTime);
            } else {
                // 如果没有设置锁的持有时间（leaseTime），则启动看门狗，定时给锁续期，防止业务逻辑未执行完成锁就过期了
                scheduleExpirationRenewal(threadId);
            }
        }
        //未获取到锁，返回锁的剩余时间
        return ttlRemaining;
    });
    return new CompletableFutureWrapper<>(f);
}
```
Lua脚本如下：
```lua
-- Lua脚本（关键部分）
local lockKey = KEYS[1] -- 锁的键名
local threadId = ARGV[1] -- 当前线程ID（UUID + 线程ID）
local lockValue = ARGV[2] -- 锁的值（通常为1）
local leaseTime = tonumber(ARGV[3]) -- 锁的租约时间（毫秒）

-- 尝试设置锁（HSET）
if redis.call("hset", lockKey, threadId, lockValue) == 1 then
    -- 设置过期时间（防止业务超时未释放）
    redis.call("pexpire", lockKey, leaseTime)
    return nil -- 获取成功
else
    -- 返回锁的剩余时间（让客户端等待）
    return redis.call("pttl", lockKey)
end
```

### 3.1.2、未获取到锁，`订阅`锁释放事件
- **目的**：当线程未立即获取到锁时，通过订阅锁的释放通知，避免空轮询，提高效率。

执行逻辑如下：
```java
private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
    long threadId = Thread.currentThread().getId();
    //尝试获取锁
    Long ttl = tryAcquire(-1, leaseTime, unit, threadId);
    // lock acquired，成功获取锁
    if (ttl == null) {
        return;
    }
    //未获取到锁，订阅锁释放通知
    CompletableFuture<RedissonLockEntry> future = subscribe(threadId);
    pubSub.timeout(future);//为订阅操作设置超时时间，防止无限期等待
    RedissonLockEntry entry;
    //同步订阅（处理中断）
    if (interruptibly) {
        //线程在等待订阅结果时，若被中断，抛出InterruptedException。适用于需要响应外部中断的场景（如超时控制或紧急退出）
        entry = commandExecutor.getInterrupted(future);
    } else {
        //禁止中断，线程会一直阻塞直到订阅完成或超时。忽略中断信号
        entry = commandExecutor.get(future);
    }
    try {
        //循环重试获取锁
        while (true) {
            ttl = tryAcquire(-1, leaseTime, unit, threadId);
            // 获取成功
            if (ttl == null) {
                break;
            }
            // 根据锁的剩余时间阻塞等待
            if (ttl >= 0) {
                //得到一个Semaphore信号量对象，这是jdk的内置对象，Semaphore#tryAcquire表示阻塞并等待唤醒
                try {
                    entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } catch (InterruptedException e) {
                    if (interruptibly) {
                        throw e;
                    }
                    entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                }
            } else {
                if (interruptibly) {
                    entry.getLatch().acquire();
                } else {
                    entry.getLatch().acquireUninterruptibly();
                }
            }
        }
    } finally {
        unsubscribe(entry, threadId);//取消订阅
    }
}
```
- **方式**：通过`subscribe()`方法订阅锁的释放事件。
- 根据`interruptibly`参数决定是否允许线程在等待订阅结果时响应中断。

#### 3.1.2.1、`subscribe()`订阅
通过订阅Redis的`Pub/Sub`频道，监听锁释放事件。
```java
protected CompletableFuture<RedissonLockEntry> subscribe(long threadId) {
  return pubSub.subscribe(getEntryName() + ":" + threadId,
            getChannelName() + ":" + getLockName(threadId));
}
```
- 通过`subscribe(threadId)`向Redis注册一个订阅请求，生成一个唯一的频道（如`redisson_lock__channel:{lockName}`）。
- 返回`CompletableFuture<RedissonLockEntry>`，表示异步订阅操作的结果
- `RedissonLockEntry`：包含一个`信号量（Semaphore）`或计数器（`CountDownLatch`），用于阻塞线程直至锁释放事件触发。

`pubSub.subscribe()`具体逻辑：主要就是构建一个`CompletableFuture`。
```java
public CompletableFuture<E> subscribe(String entryName, String channelName) {
  AsyncSemaphore semaphore = service.getSemaphore(new ChannelName(channelName));
  CompletableFuture<E> newPromise = new CompletableFuture<>();
  semaphore.acquire().thenAccept(c -> {
    if (newPromise.isDone()) {
      semaphore.release();
      return;
    }
    E entry = entries.get(entryName);
    if (entry != null) {
      entry.acquire();
      semaphore.release();
      entry.getPromise().whenComplete((r, e) -> {
        if (e != null) {
          newPromise.completeExceptionally(e);
          return;
        }
        newPromise.complete(r);
      });
      return;
    }
    //内容太多，省略
  });
  return newPromise;
}
```

#### 3.1.2.2、信号量唤醒
```java
public class SemaphorePubSub extends PublishSubscribe<RedissonLockEntry> {
    public SemaphorePubSub(PublishSubscribeService service) {
        super(service);
    }
    @Override
    protected RedissonLockEntry createEntry(CompletableFuture<RedissonLockEntry> newPromise) {
        return new RedissonLockEntry(newPromise);
    }
    @Override
    protected void onMessage(RedissonLockEntry value, Long message) {
        value.tryRunListener();
        //这里就是唤醒信号量的地方
        value.getLatch().release(Math.min(value.acquired(), message.intValue()));
    }
}
```

### 3.1.3、启动看门狗：`scheduleExpirationRenewal()`
回到“尝试获取锁：`tryAcquire()`”逻辑，内部会执行`scheduleExpirationRenewal()`方法
```java
// 启动看门狗线程（在tryAcquire成功后）
private void scheduleExpirationRenewal() {
    watchdog = new WatchdogTimerTask();
    watchdog.schedule(0, internalLockLeaseTime / 3); // 默认每10秒续期（30秒/3）
}

// 续期任务（Lua脚本）
private class WatchdogTimerTask extends TimerTask {
    public void run() {
        renewExpiration(); // 通过Lua脚本延长锁的TTL
    }
}
```
- **定时任务间隔**：默认为 i`nternalLockLeaseTime / 3`，如30秒/3=10秒
- **递归续期**：每次续期成功后，递归调用 `renewExpiration()`，重新启动定时任务，形成循环续期
- **Lua续期脚本**：
```lua
//检查当前线程是否持有锁
if redis.call("hexists", KEYS[1], ARGV[1]) == 1 then
    // 如果持有锁，则延长锁的过期时间
    redis.call("pexpire", KEYS[1], ARGV[2])
    return 1
end
return 0
```
检查当前线程是否持有锁，若持有则延迟过期时间。

**看门狗不启动的条件**：
- 显示指定了 `leaseTime & leaseTime > 0`：锁在指定时间后自动过期，无需续期。


### 3.1.4、锁的可重入
线程标识与计数器
- 每个线程的锁对应一个**Hash结构**，存储以下信息
  - **线程ID**：标识当前锁的持有者
  - **计数器**（counter）：记录同一线程获取锁的次数
- **示例存储结构**：`HSET myLock:threadId "thread-123" 3`
  - `thread-123`是线程标识。
  - `3` 标识该线程已持有锁3次。

LUA脚本
```lua
-- 加锁 Lua 脚本片段
if redis.call("HEXISTS", KEYS[1], ARGV[1]) == 0 then
    -- 锁不存在，创建锁
    redis.call("HSET", KEYS[1], ARGV[1], 1)
    redis.call("PEXPIRE", KEYS[1], ARGV[2])
    return nil
else
    -- 检查是否为当前线程持有
    local count = redis.call("HGET", KEYS[1], ARGV[1])
    if count then
        redis.call("HINCRBY", KEYS[1], ARGV[1], 1)  -- 递增计数器
        return nil
    else
        -- 其他线程持有，返回剩余时间
        return redis.call("PTTL", KEYS[1])
    end
end
```


# 四、主从同步延迟引发的分布式锁失效问题

Redisson本身不能直接解决Redis主从同步的底层问题（如主从数据延迟、同步失败等），但它通过特定的设计方案（如multiLock）规避了由主从同步延迟引发的分布式锁失效问题。

## 4.1、Redis主从同步的问题根源
Redis的主从同步是其高可用的基础，但存在以下问题：
- **同步延迟**：主节点写入的数据需要一定时间同步到从节点。若主节点在同步完成前宕机，从节点可能未获取到最小数据，导致数据不一致。
  - **分布式锁失效**：若直接使用Redis主从架构实现分布式锁，主节点宕机后，新主节点可能未同步锁信息，导致锁失效或竞争条件。


## 4.2、Resisson如何解决主从一致性问题？

### 4.2.1、multiLock方案
原理
- **多节点独立部署**：将多个Redis节点（非主从关系）作为独立节点，所有节点并列存在。
- **分布式锁的多节点获取**：获取锁时需同时在所有节点上成功获取锁才算成功。
- **高可用与一致性**：
  - 即使某个节点宕机，其他节点扔保留锁信息，确保锁的持久性。
  - 若原主节点宕机，新主节点必须通过所有节点的锁验证，避免锁失效。

示例代码：
```java
// 配置多个独立的 Redis 节点
Config config = new Config();
config.useSingleServer().setAddress("redis://192.168.244.130:6379");
config.useSingleServer().setAddress("redis://192.168.244.131:6379");
config.useSingleServer().setAddress("redis://192.168.244.132:6379");

RedissonClient redisson = Redisson.create(config);

// 使用 multiLock 获取锁
RLock lock1 = redisson.getLock("lock1");
RLock lock2 = redisson.getLock("lock2");
RLock lock3 = redisson.getLock("lock3");
RLock multiLock = redisson.getMultiLock(lock1, lock2, lock3);

try {
    multiLock.lock(); // 必须所有节点都成功获取锁
    // 业务逻辑
} finally {
    multiLock.unlock(); // 必须所有节点都释放锁
}
```

源码如下：
```java
public class RedissonMultiLock implements RLock {
    @Override
    public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        long newLeaseTime = -1;
        if (leaseTime > 0) {
            if (waitTime > 0) {
                newLeaseTime = unit.toMillis(waitTime)*2;
            } else {
                newLeaseTime = unit.toMillis(leaseTime);
            }
        }

        long time = System.currentTimeMillis();
        long remainTime = -1;
        if (waitTime > 0) {
            remainTime = unit.toMillis(waitTime);
        }
        long lockWaitTime = calcLockWaitTime(remainTime);

        int failedLocksLimit = failedLocksLimit();
        List<RLock> acquiredLocks = new ArrayList<>(locks.size());
        //循环遍历，导致网络延迟叠加
        for (ListIterator<RLock> iterator = locks.listIterator(); iterator.hasNext();) {
            RLock lock = iterator.next();
            boolean lockAcquired;
            try {
                if (waitTime <= 0 && leaseTime <= 0) {
                    lockAcquired = lock.tryLock();
                } else {
                    long awaitTime = Math.min(lockWaitTime, remainTime);
                    lockAcquired = lock.tryLock(awaitTime, newLeaseTime, TimeUnit.MILLISECONDS);
                }
            } catch (RedisResponseTimeoutException e) {
                unlockInner(Arrays.asList(lock));
                lockAcquired = false;
            } catch (Exception e) {
                lockAcquired = false;
            }

            if (lockAcquired) {
                acquiredLocks.add(lock);
            } else {
                if (locks.size() - acquiredLocks.size() == failedLocksLimit()) {
                    break;
                }

                if (failedLocksLimit == 0) {
                    unlockInner(acquiredLocks);
                    if (waitTime <= 0) {
                        return false;
                    }
                    failedLocksLimit = failedLocksLimit();
                    acquiredLocks.clear();
                    // reset iterator
                    while (iterator.hasPrevious()) {
                        iterator.previous();
                    }
                } else {
                    failedLocksLimit--;
                }
            }

            if (remainTime > 0) {
                remainTime -= System.currentTimeMillis() - time;
                time = System.currentTimeMillis();
                if (remainTime <= 0) {
                    unlockInner(acquiredLocks);
                    return false;
                }
            }
        }
        if (leaseTime > 0) {
            acquiredLocks.stream()
                    .map(l -> (RedissonBaseLock) l)
                    .map(l -> l.expireAsync(unit.toMillis(leaseTime), TimeUnit.MILLISECONDS))
                    .forEach(f -> f.toCompletableFuture().join());
        }
        return true;
    }
}
```


影响：
- **多节点操作的延迟**： multiLock 需要同时操作多个Redis节点，导致网络延迟叠加。
- **资源消耗**：每个锁操作都需要在多个节点上执行原子命令，增加Redis服务器的计算和内存开销。

优化建议：
- 减少节点数量
- **集群模式**：使用Redis集群模式替代多主从架构，减少跨节点通信的复杂度。
