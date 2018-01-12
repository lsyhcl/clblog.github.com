---
layout: post
title:  "前端的一些资料和工具"
date:   2015-05-18 14:06:05
categories: Front-end
excerpt: 记录一些好用的前端工具和框架。
---

* content
{:toc}


### 3. 主从模式Reactor

#### MainReactor

```java
package com.tr.reactor.mainsub;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * MainReactor模块，包含dispatch
 */
public class MainReactor implements Runnable
{
    final Selector selector;
    
    final ServerSocketChannel serverSocketChannel;
    
    /**
     * 初始化Reactor
     * 
     * @throws IOException
     */
    public MainReactor()
        throws IOException
    {
        selector = Selector.open();
        serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.bind(new InetSocketAddress(8091));
        SelectionKey selectionKey = serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        // 将给定的对象附加到此键。之后可通过 attachment 方法获取已附加的对象。这里附加Acceptor用于连接建立
        selectionKey.attach(new Acceptor(serverSocketChannel, selector));
    }
    
    /**
     * 循环查看Selector。是否有新的请求或者响应
     */
    @Override
    public void run()
    {
        try
        {
            while (!Thread.interrupted())
            {
                // 请求来了通过Selector来管理
                selector.select();
                Set selectorKeys = selector.selectedKeys();
                Iterator iterator = selectorKeys.iterator();
                while (iterator.hasNext())
                {
                    SelectionKey key = (SelectionKey)iterator.next();
                    dispatch(key);
                    iterator.remove();
                }
            }
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }
    
    /**
     * 分发每一个有激活的选择键，将之前的对象（如：Acceptor、Handler）取出，并调用其run方法
     * 
     * @param key 选择键
     */
    void dispatch(SelectionKey key)
    {
        Runnable r = (Runnable)key.attachment();
        if (r != null)
        {
            r.run();
        }
    }
    
    /**
     * 程序入口
     * 
     * @param args
     * @throws IOException
     */
    public static void main(String[] args)
        throws IOException
    {
        ExecutorService executorService = Executors.newSingleThreadExecutor();
        executorService.submit(new MainReactor());
    }
}

```



#### Acceptor

```java
package com.tr.reactor.mainsub;

import java.io.IOException;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;

/**
 * Acceptor，用于建立连接。建立连接后获取与客户端的SocketChannel，然后创建Handler去处理读写请求
 */
public class Acceptor implements Runnable
{
    private ServerSocketChannel serverSocketChannel;
    
    private Selector selector;
    
    // 获取CPU核心数
    private final int cores = Runtime.getRuntime().availableProcessors();
    
    // 创建于核心数相同的subSelector
    private final Selector[] selectors = new Selector[cores];
    
    // 当前可使用的subReactor索引
    private int selIdx = 0;
    
    private SubReactor[] subReactors = new SubReactor[cores]; // subReactor線程
    
    private Thread[] threadsSub = new Thread[cores]; // subReactor線程
    
    public Acceptor(ServerSocketChannel serverSocketChannel, Selector selector)
        throws IOException
    {
        this.serverSocketChannel = serverSocketChannel;
        this.selector = selector;
        
        // 创建多个SubReactor
        for (int i = 0; i < cores; i++)
        {
            selectors[i] = Selector.open();
            subReactors[i] = new SubReactor(selectors[i], serverSocketChannel, i);
            threadsSub[i] = new Thread(subReactors[i]);
            threadsSub[i].start();
        }
    }
    
    @Override
    public void run()
    {
        
        try
        {
            System.out.println("accept");
            // 建立连接后获取与客户端的SocketChannel
            SocketChannel socketChannel = serverSocketChannel.accept();
            if (null != socketChannel)
            {
                subReactors[selIdx].restart(true);
                // 建立Handler去专门处理读写请求
                new Handler(selectors[selIdx], socketChannel);
                subReactors[selIdx].restart(false);
                selIdx++;
                if (selIdx == cores)
                {
                    selIdx = 0;
                }
            }
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
        
    }
}

```



#### SubReactor

> Sub Reactor在实作上有个重点要注意，
>
> 当一个监听中而阻塞住的selector由于Acceptor需要注册新的IO事件到该selector上时，
>
> Acceptor会调用selector的wakeup()函数唤醒阻塞住的selector，以注册新IO事件后再继续监听。
>
> 但Sub Reactor中循环调用selector.select()的线程回圈可能会因为循环太快，导致selector被唤醒后再度**于IO事件成功注册前**被调用selector.select()而阻塞住，
>
> 因此我们需要给Sub Reactor线程循环设置一个flag来控制，这里是**restart**
>
> 让selector被唤醒后不会马上进入下回合调用selector.select()的Sub Reactor线程循环，
>
> 等待我们将新的IO事件注册完之后才能让Sub Reactor线程继续运行。

```java
package com.tr.reactor.mainsub;

import java.io.IOException;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.util.Iterator;
import java.util.Set;

/**
 * SubReactor模块，包含dispatch
 */
public class SubReactor implements Runnable
{
    final ServerSocketChannel serverSocketChannel;
    
    final int index;
    
    final Selector selector;
    
    // 由于当reactor在selector.select()阻塞的时候，channel无法向selector注册，所以要用一个flag来控制，当唤醒用wakeup唤醒selector后，不至于回环太快又进入selector.select()阻塞
    private boolean restart = false;
    
    // 每个subReactor都有MainReactor的serverSocketChannel。属于自己的selector
    public SubReactor(Selector selector, ServerSocketChannel serverSocketChannel, int index)
    {
        this.serverSocketChannel = serverSocketChannel;
        this.index = index;
        this.selector = selector;
    }
    
    @Override
    public void run()
    {
        try
        {
            while (!Thread.interrupted())
            {
                while (!restart)
                {
                    // 请求来了通过Selector来管理
                    selector.select();
                    Set selectorKeys = selector.selectedKeys();
                    System.out.println(selectorKeys.size());
                    Iterator iterator = selectorKeys.iterator();
                    while (iterator.hasNext())
                    {
                        SelectionKey key = (SelectionKey)iterator.next();
                        dispatch(key);
                        iterator.remove();
                    }
                }
                
            }
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }
    
    /**
     * 分发每一个有激活的选择键，将之前的对象（如：Acceptor、Handler）取出，并调用其run方法
     *
     * @param key 选择键
     */
    void dispatch(SelectionKey key)
    {
        Runnable r = (Runnable)key.attachment();
        if (r != null)
        {
            r.run();
        }
    }
    
    public void restart(boolean b)
    {
        restart = b;
    }
}

```



#### Handler

```java
package com.tr.reactor.mainsub;

import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Arrays;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * Handler专门处理读写请求
 */
public class Handler implements Runnable
{
    final SocketChannel socket;

    final SelectionKey selectionKey;

    // 专门处理读字节的缓存池
    private ByteBuffer input = ByteBuffer.allocate(10);

    // 专门处理写字节的缓存池
    private ByteBuffer output = ByteBuffer.allocate(10);

    private static final int READING = 0, SENDING = 1;

    int state = READING;

    // 线程池处理process业务
    private ExecutorService executorService = Executors.newFixedThreadPool(10);

    public Handler(Selector selector, SocketChannel socketChannel)
            throws IOException
    {

        socket = socketChannel;
        socketChannel.configureBlocking(false);
        // 唤醒selector。
        selector.wakeup();
        // 把socketChannel注册到Selector，标记感兴趣为读事件
        selectionKey = socketChannel.register(selector, SelectionKey.OP_READ);
        // 将给定的对象附加到此键。之后可通过 attachment 方法获取已附加的对象。这里附加Handler用于处理读写
        selectionKey.attach(this);
        // 唤醒selector。
        selector.wakeup();
        socketChannel.write(ByteBuffer.wrap("send message to client".getBytes()));
    }

    /**
     * 具体的请求处理，可能是读事件、写事件等
     */
    @Override
    public void run()
    {
        try
        {
            if (state == READING)
                read();
            else if (state == SENDING)
                send();
        }
        catch (IOException ex)
        {
            ex.printStackTrace();
        }
    }

    /**
     * 处理读事件
     *
     * @throws IOException
     */
    void read()
    {
        try
        {
            System.out.println("read");
            input.clear();
            socket.read(input);
            byte[] data = input.array();
            if (inputIsComplete())
            {
                executorService.submit(new Processer(Arrays.copyOf(data, data.length)));
                // process(Arrays.copyOf(data, data.length));
            }
        }
        catch (IOException e)
        {
            e.printStackTrace();
            closeChannel();
        }

    }

    /**
     * 检查input是否已经完成
     *
     * @return
     */
    private boolean inputIsComplete()
    {
        return true;
    }

    private void process(byte[] data)
    {
        String message = new String(data);
        System.out.println("receive message from client, size:" + data.length + " msg: " + message);
    }

    /**
     * 处理写事件
     *
     * @throws IOException
     */
    void send()
            throws IOException
    {
        System.out.println("write");
        socket.write(output);
        if (outputIsComplete())
            selectionKey.cancel();
    }

    /**
     * 检查output是否已经完成
     *
     * @return
     */
    private boolean outputIsComplete()
    {
        return true;
    }

    class Processer implements Runnable
    {
        byte[] data;

        public Processer(byte[] data)
        {
            this.data = data;
        }

        @Override
        public void run()
        {
            process(data);
            // state = SENDING;
            // Normally also do first write now
            // selectionKey.interestOps(SelectionKey.OP_WRITE);

        }
    }


    /**
     * 关闭Channel。selectionKey取消注册
     */
    public void closeChannel()
    {
        try
        {
            selectionKey.cancel();
            socket.close();
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }

}

```

## 参考

[BIO与NIO、AIO的区别(这个容易理解)](http://blog.csdn.net/skiof007/article/details/52873421)

[Reactor（反应器）模式初探](http://blog.csdn.net/pistolove/article/details/53152708)

[网络编程基础(5) : IO多路复用(多Reactor)(主从式Reactor)](http://blog.csdn.net/yehjordan/article/details/51026045)
