---
title: bio
date: 2019-05-18 22:11:10
tags: bio
categories: io
---

从代码层次了解BIO

    //服务端
    public class BioServer {
    
        public static void main(String[] args) {
            byte[] bs = new byte[1024];
            try {
                //服务端的监听socket，只负责监听连接，监听的端口是：9878
                ServerSocket serverSocket = new ServerSocket();
                serverSocket.bind(new InetSocketAddress(9878));
    
                while (true) {//可以进行下一次的通信
                    System.out.println("等待连接");
                    Socket accept = serverSocket.accept();//服务端进程将阻塞(将释放CPU资源)，直至连接请求过来，然后会生成一个socket
                    //accept，这个socket是负责和客户端数据交换的
                    System.out.println("连接成功");
    
                    System.out.println("等待数据");
                    int readCount = accept.getInputStream().read(bs);//read也将阻塞
                    System.out.println("数据获取成功");
                    System.out.println("读取的数量=" + readCount);
                    String content = bs.toString();
                    System.out.println("读取的内容为：" + content);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    //客户端
    public class BioClient {
    
        public static void main(String[] args) {
            Socket socket = new Socket();
            socket.connect(new InetSocketAddress("127.0.0.1", 9878));
        }
    }
    
<!--more-->

以下是图形解释

![bio-1](bio/bio.png)

bio：accept(),read()两个阻塞，如果采用单线程时，无法处理并发。当上一个连接未处理完成，server端则不会处理其他客户端请求。

bio的优化方式：
1、多线程方式，主线程负责连接，子线程数据交换

    while (true) {//可以进行下一次的通信
        System.out.println("等待连接");
        Socket accept = serverSocket.accept();//服务端进程将阻塞(将释放CPU资源)，直至连接请求过来，然后会生成一个socket
        //accept，这个socket是负责和客户端数据交换的
        System.out.println("连接成功");

        Threa thread = new Thread();
        thread.start();
        //int readCount = accept.getInputStream().read(bs);//read也将阻塞
        //System.out.println("数据获取成功");
        //System.out.println("读取的数量=" + readCount);
        //String content = bs.toString();
        //System.out.println("读取的内容为：" + content);
    }
缺点：线程利用率太低，大部分线程都是无效线程（不会进行数据传输）
使用多线程方式，需要考虑线程利用率，线程都是用来数据传输。    
    