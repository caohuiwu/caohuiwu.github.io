---
title: tomcat-netty对比
date: 2020-01-01 11:54:20
categories:
  - [tomcat]
---
# 一、Acceptor线程数量对比
## Tomcat：默认是1，该线程监听客户端连接请求
```
    protected class Acceptor extends AbstractEndpoint.Acceptor {
        ........
        SocketChannel socket = null;
        try {
            // Accept the next incoming connection from the server
            // socket
            //接收客户端连接请求，使用acceptor，会阻塞，直到有连接请求
            socket = serverSock.accept();
        }
        ........
    }
```

* Acceptor：单线程阻塞监听连接事件，统一处理连接
* Poller：轮询器（最少2个），内部封装了selector，轮询该selector
* PollerEvent：事件处理线程队列，将事件（socket）注册进selector内
* 最后使用线程池去处理请求。

## Netty：
```
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```
SelectionKey.OP_ACCEPT标志就是监听套接字所感兴趣的事件了

* 使用selector非阻塞模式

[参考博客](https://github.com/fengjiachun/doc/blob/master/netty/Netty%E6%BA%90%E7%A0%81%E7%BB%86%E8%8A%822--bind.md)




