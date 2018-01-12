---
layout: post
title:  reactor and netty
date:   2018-01-12 00:00:00
categories: netty
excerpt: 
---

* content
{:toc}

# 从Reactor模式到Netty再到AIO

## BIO和NIO

### BIO

 在JDK1.4出来之前，我们建立网络连接的时候采用BIO模式，需要先在服务端启动一个ServerSocket，然后在客户端启动Socket来对服务端进行通信，默认情况下服务端需要对每个请求建立一堆线程等待请求，而客户端发送请求后，先咨询服务端是否有线程相应，如果没有则会一直等待或者遭到拒绝请求，如果有的话，客户端会线程会等待请求结束后才继续执行。

### NIO

NIO基于Reactor，当socket有流可读或可写入socket时，操作系统会相应的通知引用程序进行处理，应用再将流读取到缓冲区或写入操作系统。  也就是说，这个时候，已经不是一个连接就要对应一个处理线程了，而是有效的请求，对应一个线程，当连接没有数据时，是没有工作线程来处理的。

BIO与NIO一个比较重要的不同，是我们使用BIO的时候往往会引入多线程，每个连接一个单独的线程；而NIO则是使用单线程或者只使用少量的多线程，每个连接共用一个线程。

NIO的最重要的地方是当一个连接创建后，不需要对应一个线程，这个连接会被注册到多路复用器上面，所以所有的连接只需要一个线程就可以搞定，当这个线程中的多路复用器进行轮询的时候，发现连接上有请求的话，才开启一个线程进行处理，也就是一个请求一个线程模式。

**在NIO的处理方式中，当一个请求来的话，开启线程进行处理，可能会等待后端应用的资源(JDBC连接等)，其实这个线程就被阻塞了，当并发上来的话，还是会有BIO一样的问题。**

HTTP/1.1出现后，有了Http长连接，这样除了超时和指明特定关闭的http header外，这个链接是一直打开的状态的，这样在NIO处理中可以进一步的进化，在后端资源中可以实现资源池或者队列，当请求来的话，开启的线程把请求和请求数据传送给后端资源池或者队列里面就返回，并且在全局的地方保持住这个现场(哪个连接的哪个请求等)，这样前面的线程还是可以去接受其他的请求，而后端的应用的处理只需要执行队列里面的就可以了，这样请求处理和后端应用是异步的。当后端处理完，到全局地方得到现场，产生响应，这个就实现了异步处理。

## java.nio

### Selector

`Selector`是java提供的一个选择器，每个channel都会使用`channel.register`注册到selector上。当有请求来到的时候，`selector.open`会返回（selector.open是阻塞的）有响应的selectionKey，通过`selectionKey.channel`就能获取到对应的Channel

使用`Selector`的前提是channel设置了`channel.configureBlocking(false)`

channel可以监听4种状态分别是

* `SelectionKey.OP_CONNECT`  一个Channel成功连接到一个服务器端
* `electionKey.OP_ACCEPT`   服务器端接受一个连接请求
* `SelectionKey.OP_READ`  从一个Channel读数据
* `SelectionKey.OP_WRITE`  向一个Channel写数据

同时还可以调用`selector.wakeup`使得`selector.open`快速返回

### 客户端

首先我们使用java.nio包提供的`SocketChannel`、`Selector`、`SelectionKey`来实现一个客户端

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;

public class NioClient
{
    public static void main(String[] args)
        throws IOException
    {
        // 获取socket通道
        SocketChannel channel = SocketChannel.open();
        channel.configureBlocking(false);
        // 获得通道管理器
        Selector selector = Selector.open();
        channel.connect(new InetSocketAddress(8091));
        // 为该通道注册SelectionKey.OP_CONNECT事件
        channel.register(selector, SelectionKey.OP_CONNECT);
        while (true)
        {
            // 选择注册过的io操作的事件(第一次为SelectionKey.OP_CONNECT)
            selector.select();
            for (SelectionKey key : selector.selectedKeys())
            {
                if (key.isConnectable())
                {
                    System.out.println("isConnectable");
                    SocketChannel channel1 = (SocketChannel)key.channel();
                    if (channel1.isConnectionPending())
                    {
                        channel1.finishConnect();// 如果正在连接，则完成连接
                    }
                    // 设置监听事件为读事件
                    channel1.register(selector, SelectionKey.OP_READ);
                }
                else if (key.isReadable())
                { // 有可读数据事件。
                    SocketChannel channel1 = (SocketChannel)key.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(10);
                    channel1.read(buffer);
                    byte[] data = buffer.array();
                    String message = new String(data);
                    System.out.println("recevie message from server:, size:" + buffer.position() + " msg: " + message);
                    channel1.write(ByteBuffer.wrap("hello".getBytes()));
                }
            }
        }
    }
}

```

### 服务端

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;

public class NioServer
{
    public static void main(String[] args)
        throws IOException
    {
        ServerSocketChannel socketChannel = ServerSocketChannel.open();
        socketChannel.configureBlocking(false);
        
        Selector selector = Selector.open();
        socketChannel.socket().bind(new InetSocketAddress(8091));
        
        socketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true)
        {
            // 当有注册的事件到达时，方法返回，否则阻塞。
            selector.select();
            System.out.println(selector.selectedKeys().size());
            Iterator it = selector.selectedKeys().iterator();
            while (it.hasNext()){
                SelectionKey key = (SelectionKey)it.next();
                if (key.isAcceptable())
                {// 有新的连接建立
                    System.out.println("is acceptable");
                    ServerSocketChannel socketChannel1 = (ServerSocketChannel)key.channel();
                    // 通过 ServerSocketChannel.accept() 方法监听新进来的连接。当 accept()方法返回的时候,它返回一个包含新进来的连接的 SocketChannel
                    SocketChannel channel = socketChannel1.accept();
                    if (channel == null)
                    {
                        continue;
                    }
                    channel.configureBlocking(false);
                    channel.write(ByteBuffer.wrap("send message to client".getBytes()));
                    // 在与客户端连接成功后，为客户端通道注册SelectionKey.OP_READ事件。
                    channel.register(selector, SelectionKey.OP_READ);
                }
                else if (key.isReadable())
                {// 有可读数据事件
                    SocketChannel channel = (SocketChannel)key.channel();
                    System.out.println(channel.hashCode());
                    ByteBuffer buffer = ByteBuffer.allocate(10);
                    int read = channel.read(buffer);
                    byte[] data = buffer.array();
                    String message = new String(data);
                    System.out.println("receive message from client, size:" + buffer.position() + " msg: " + message);
                }
                // 在Selector中有响应的key不会自动消除，需要手动去除
                it.remove();
            }
        }
    }
}

```
