---
  title: io与nio的那些事
  date: {{ date }}   
  categories: ['网络通信']
  tags: ['Java','网络通信']       
  comments: true    
  img:             
---

本篇首发于[橙寂博客](http://www.luckyhe.com/post/38.html)转载请加上此标示。
最近要学习netty，netty是基于nio上的一个框架。所以了解什么是netty之前先了解下io与nio。

## io模式
 [io五种模式详解](https://blog.csdn.net/ZWE7616175/article/details/80591587)
（1）阻塞I/O模型：最好理解的I/O模型就是阻塞I/O模型，所有文件操作都是阻塞的。其系统调用直到数据包到达且被复制到应用进程的缓冲区中或者发生错误时才返回，在此期间一直会等待。因为进程在从调用到它返回的整段时间内都是被阻塞的，因此被称为阻塞I/O模型。

（2）非阻塞I/O模型：应用进程调用到内核的时候，如果该缓冲区没有数据的话，就直接返回一个错误，一般都对非阻塞I/O模型进行轮询检查这个状态，看内核是不是有数据到来。

（3）I/O复用模型：Linux提供select/poll，进程通过将一个或多个文件描述符(fd)传递给select或poll系统调用，阻塞在select操作上，这样select/poll可以帮我们侦测多个fd是否处于就绪状态。select/poll是顺序扫描fd是否就绪，而且支持的fd数量有限，因此它的使用受到了一些制约。Linux还提供了一个epoll系统调用，epoll使用基于事件驱动方式代替顺序扫描，因此性能更高。当有fd就绪时，立即回调函数rollback。

（4）信号驱动I/O模型：开启套接口信号驱动I/O功能。当数据准备就绪时，就为该进程生成一个信号，通过信号回调通知应用程序来读取数据。

（5）异步I/O：告知内核启动某个操作，并让内核在整个操作完成后（包括将数据从内核复制到用户自己的缓冲区）通知我们。这种模型与信号驱动模型的主要区别是：信号驱动I/O由内核通知我们何时可以开始一个I/O操作；异步I/O模型由内核通知我们I/O操作何时已经完成。

## 最原始io编程
传统的io网络编程是基于**socket**跟**ServerSocket**实现的。他是**阻塞式**的一种实现。这种模式所有文件读写都是阻塞式的。说白了就是我在读文件或者写文件。只有等它干完我才能做别的事。
![d0e5a611093507fe7426af5674499e45.png](http://www.luckyhe.com/storage/thumbnails/_signature/2L75TPOT7AHKSCL027O6ETMEU7.png)

- 服务端demo
```
package com.janhe.netty_learnning.io;

import java.io.IOException;
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * @CLASSNAME Server
 * @Description
 * @Auther Jan  橙寂
 * @DATE 2019/8/27 0027 9:53
 */

public class Server {

    /**
     * 通过io模式去创建一个服务端，基于二进制流来进行数据传输
     *
     * @param args
     */
    public static void main(String[] args) throws IOException {


        //创建服务端监听8000端口
        ServerSocket serverSocket = new ServerSocket(8000);

        new Thread(() -> {


            try {
                //阻塞连接等待客户端加入
                Socket client = serverSocket.accept();

                //开启个新的线程去处理连接
                new Thread(() -> {

                    byte[] data = new byte[1024];
                    InputStream inputStream = null;
                    try {
                        inputStream = client.getInputStream();
                        while (true) {
                            int len = 0;
                            while ((len = inputStream.read(data)) != -1) {
                                System.out.printf(new String(data, 0, len));
                            }
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }

                }).start();

            } catch (IOException e) {
                e.printStackTrace();
            }

        }).start();
    }
}

```

- 客户端demo
```
package com.janhe.netty_learnning.io;

import java.io.IOException;
import java.net.Socket;
import java.util.Date;

/**
 * @CLASSNAME Client
 * @Description
 * @Auther Jan  橙寂
 * @DATE 2019/8/27 0027 14:12
 */

public class Client {

    /**
     * 客户端向服务端发送数据
     *
     * @param args
     */
    public static void main(String[] args) {

        new Thread(() -> {
            try {
                //创建个客户端
                Socket socket = new Socket("127.0.0.1", 8000);
                while (true) {
                    //每隔两秒像服务端发送一条信息
                    socket.getOutputStream().write((new Date() + "你好").getBytes());
                    socket.getOutputStream().flush();
                    Thread.sleep(2000);
                }
            } catch (IOException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}

```

- 传统io总结
这里可以看出。这种模型有很多的问题。
1. 线程资源受限：线程不是无限大。同一时间有大量线程系统就会崩。
2. cpu切换效率低下：cpu的核数也是固定的。
3. 数据读写都是以流的方式：读完就没了，需要自己缓存。

接着jdk1.4后nio就出现了。

## nio编程
https://blog.csdn.net/canot/article/details/51372651
nio在传统的io做了巨大改变。当然这些改变还是为了改变传统io遗留下来的问题。
NIO的核心类库多路复用器**selector**就是基于epoll的多路复用技术实现。
在I/O编程过程中，当需要同时处理多个客户端接入请求时，可以利用多线程或者I/O多路复用技术进行处理。I/O多路复用技术通过把多个I/O的阻塞复用到同一个**selector**的阻塞上，从而使得系统在单线程的情况下可以同时处理多个客户端请求。与传统的多线程/多进程模型比，I/O多路复用的最大优势是系统开销小，系统不需要创建新的额外进程或者线程，也不需要维护这些进程和线程的运行，降低了系统的维护工作量，节省了系统资源。

#### nio服务端新增的api
>1.进行异步I/O操作的缓冲区ByteBuffer等；
2.进行异步I/O操作的管道Pipe；
3.进行各种I/O操作（异步或者同步）的Channel，包括4.ServerSocketChannel和SocketChannel；
5.多种字符集的编码能力和解码能力；
6.实现非阻塞I/O操作的多路复用器selector；
7.基于流行的Perl实现的正则表达式类库；
8.文件通道FileChannel。

#### 使用nio的好处
1. 线程资源：减少了频繁的创建线程，线程切换效率提高了。
2. 数据读写都是以**ByteBuffer**的形式。nio维护了一个缓存区，每次都是从通道（**channel**）去缓存区(**ByteBuffer**)中取数据。不在以字节为单位，而是以字节块为单位。

#### 使用nio编程

-  服务端
```
package com.janhe.netty_learnning.nio;

import org.junit.Test;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.nio.charset.Charset;
import java.util.Iterator;
import java.util.Set;

/**
 * @CLASSNAME NioServer
 * @Description
 * @Auther Jan  橙寂
 * @DATE 2019/8/27 0027 15:00
 */

public class NioServer {


    public static void init() throws IOException {
        Charset charset = Charset.forName("UTF-8");
        // 创建一个选择器，可用close()关闭，isOpen()表示是否处于打开状态，他不隶属于当前线程
        Selector selector = Selector.open();
        // 创建ServerSocketChannel，并把它绑定到指定端口上
        ServerSocketChannel server = ServerSocketChannel.open();
        server.socket().bind(new InetSocketAddress(7777), 1024);
        // 设置为非阻塞模式, 这个非常重要
        server.configureBlocking(false);
        // 在选择器里面注册关注这个服务器套接字通道的accept事件
        // ServerSocketChannel只有OP_ACCEPT可用，OP_CONNECT,OP_READ,OP_WRITE用于SocketChannel
        server.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
            //休眠时间为1s，无论是否有读写等事件发生，selector每隔1s都被唤醒一次
            selector.select(1000);
            //如果有我们事件进来key就会有值。
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> it = keys.iterator();
            SelectionKey key = null;
            while (it.hasNext()) {
                //如果key对应的Channel包含客户端的链接请求
                // OP_ACCEPT 这个只有ServerSocketChannel才有可能触发
                key = it.next();
                // 由于select操作只管对selectedKeys进行添加，所以key处理后我们需要从里面把key去掉
                it.remove();
                if (key.isAcceptable()) {
                    System.out.println("有人关注了");
                    // 得到与客户端的套接字通道
                    ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
                    //ServerSocketChannel的accept接收客户端的连接请求并创建SocketChannel实例，完成上述操作后，相当于完成了TCP的三次握手，TCP物理链路正式建立。
                    //我们需要将新创建的SocketChannel设置为异步非阻塞，同时也可以对其TCP参数进行设置，例如TCP接收和发送缓冲区的大小等。此处省掉
                    SocketChannel channel = ssc.accept();
                    channel.configureBlocking(false);
                    channel.register(selector, SelectionKey.OP_READ);
                    //将key对应Channel设置为准备接受其他请求
                    key.interestOps(SelectionKey.OP_ACCEPT);
                }
                if (key.isReadable()) {
                    System.out.println("客户端发了数据");
                    SocketChannel channel = (SocketChannel) key.channel();
                    ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                    String content = "";
                    try {
                        int readBytes = channel.read(byteBuffer);
                        if (readBytes > 0) {
                            //切换为读的状态，内部的实现比较复杂
                            //内部维护了三个指针 position  capacity  limit  也就是调用这个方法后postion归0 读的是 0 -limit 的数据
                            //实在还不同看源码 这一步是必要的
                            byteBuffer.flip();
                            byte[] bytes = new byte[byteBuffer.remaining()];
                            byteBuffer.get(bytes);
                            content += new String(bytes);
                            System.out.println(content);
                            //回应客户端
                            doWrite(channel);
                        }
                        // 写完就把状态关注去掉，否则会一直触发写事件(改变自身关注事件)
                        key.interestOps(SelectionKey.OP_READ);
                    } catch (IOException i) {
                        //如果捕获到该SelectionKey对应的Channel时出现了异常,即表明该Channel对于的Client出现了问题
                        //所以从Selector中取消该SelectionKey的注册
                        key.cancel();
                        if (key.channel() != null) {
                            key.channel().close();
                        }
                    }
                }
            }
        }
    }

    private static void doWrite(SocketChannel sc) throws IOException {
        byte[] req = "服务器已接受".getBytes();
        ByteBuffer byteBuffer = ByteBuffer.allocate(req.length);
        byteBuffer.put(req);
        byteBuffer.flip();
        sc.write(byteBuffer);

    }


    public static void main(String[] args) throws IOException {
        new Thread(() -> {
            try {
                init();
            } catch (IOException e) {
                e.printStackTrace();
                System.exit(1);
            }
        }).start();

    }


}

```
-  客户端
```
package com.janhe.netty_learnning.nio;


import org.junit.Test;

import java.io.IOException;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.nio.charset.Charset;
import java.util.Iterator;
import java.util.Set;
import java.util.concurrent.ArrayBlockingQueue;

/**
 * @CLASSNAME NioClient
 * @Description
 * @Auther Jan  橙寂
 * @DATE 2019/8/27 0027 16:49
 */

public class NioClient {




    static Charset charset = Charset.forName("UTF-8");
    private static volatile boolean stop = false;
    public ArrayBlockingQueue<String> arrayQueue = new ArrayBlockingQueue<String>(8);
    public  static void  init() throws IOException{
         Selector selector = null;
        selector = Selector.open();
        SocketChannel channel = SocketChannel.open();
        // 设置为非阻塞模式，这个方法必须在实际连接之前调用(所以open的时候不能提供服务器地址，否则会自动连接)
        channel.configureBlocking(false);
        if(channel.connect(new InetSocketAddress("127.0.0.1",7777))){
            channel.register(selector, SelectionKey.OP_READ);
            //发送消息
            doWrite(channel, "66666666");
        }else {
            //注册连接事件
            channel.register(selector, SelectionKey.OP_CONNECT);
        }

        //启动一个接受服务器反馈的线程
        //  new Thread(new ReceiverInfo()).start();

        while (!stop){
            //如果有我们注册的事情发生了，它的传回值就会大于0
            selector.select(1000);
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> it = keys.iterator();
            SelectionKey key = null;
            while (it.hasNext()){
                key = it.next();
                it.remove();
                SocketChannel sc = (SocketChannel) key.channel();
                // OP_CONNECT 两种情况，链接成功或失败这个方法都会返回true
                if (key.isConnectable()){
                    // 由于非阻塞模式，connect只管发起连接请求，finishConnect()方法会阻塞到链接结束并返回是否成功
                    // 另外还有一个isConnectionPending()返回的是是否处于正在连接状态(还在三次握手中)
                    if (channel.finishConnect()) {
                      System.out.println("准备发送数据");
                        // 链接成功了可以做一些自己的处理
                        channel.write(charset.encode("I am Coming"));
                        // 处理完后必须吧OP_CONNECT关注去掉，改为关注OP_READ
                        key.interestOps(SelectionKey.OP_READ);
                        sc.register(selector,SelectionKey.OP_READ);
                        //    new Thread(new DoWrite(channel)).start();
                        doWrite(channel, "66666666");
                    }else {
                        //链接失败，进程推出或直接抛出IOException
                        System.exit(1);
                    }
                } if(key.isReadable()){
                    //读取服务端的响应
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    int readBytes = sc.read(buffer);
                    String content = "";
                    if (readBytes>0){
                        buffer.flip();
                        byte[] bytes = new byte[buffer.remaining()];
                        buffer.get(bytes);
                        content+=new String(bytes);
                        stop=true;
                    }else if(readBytes<0) {
                        //对端链路关闭
                        key.channel();
                        sc.close();
                    }
                    System.out.println(content);
                    key.interestOps(SelectionKey.OP_READ);
                }
            }
        }
    }
    private static   void doWrite(SocketChannel sc,String data) throws IOException{
        byte[] req =data.getBytes();
        ByteBuffer byteBuffer = ByteBuffer.allocate(req.length);
        byteBuffer.put(req);
        byteBuffer.flip();
        sc.write(byteBuffer);

    }


    public static void main(String[] args) throws IOException {

        init();
    }

}

```

##  nio总结
NIO 有一个主要的类Selector,这个类似一个观察者，只要我们把需要探知的socketchannel告诉Selector(注册相关事件),我们接着做别的事情，当有事件发生时，他会通知我们，传回一组SelectionKey,我们读取这些Key,就会获得我们刚刚注册过的socketchannel,然后，我们从这个Channel中读取数据，放心，包准能够读到，接着我们可以处理这些数据。

Selector内部原理实际是在做一个对所注册的channel的轮询访问，不断的轮询(目前就这一个算法)，一旦轮询到一个channel有所注册的事情发生，比如数据来了，他就会站起来报告，交出一把钥匙，让我们通过这把钥匙来读取这个channel的内容。

## aio编程（nio2.0）
在jdk1.7中提供了全新的nio异步通道，nio2.0也称aio。这次的更新引入了异步通道的概念。并提供了异步文件通道和异步套接字通道的实现。异步通道提供两种方式获取获取操作结果。分别是：
>1.java.util.concurrent.Future类来表示异步操作的结果；2.mpletionHandler接口的实现类作为操作完成的回调。

NIO2.0的异步套接字通道是真正的异步非阻塞I/O，它对应UNIX网络编程中的事件驱动I/O（AIO），它不需要通过多路复用器（Selector）对注册的通道进行轮询操作即可实现异步读写，从而简化了NIO的编程模型。

在nio2.0是基于**Reactor**（事件处理模型）接下来代码会继续讲到这一点。

- aio服务端demo
```
package com.janhe.netty_learnning.aio;

import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.CountDownLatch;

/**
 * @CLASSNAME AioServer
 * @Description aio也就是nio2.0 jdk 1.7后提供的新的异步通道概念
 * @Auther Jan  橙寂
 * @DATE 2019/8/28 0028 9:50
 */

public class AioServer {

    AsynchronousServerSocketChannel asynchronousServerSocketChannel;
    //使用对列缓存数据，因为当前一个异步写调用还没完成之前，调用异步写会抛WritePendingException
    //这里比较坑
    private final Queue<ByteBuffer> queue = new LinkedList<ByteBuffer>();
    //是否可以写
    private boolean writing = false;
    CountDownLatch latch;

    public void start() {
        try {

            asynchronousServerSocketChannel = AsynchronousServerSocketChannel.open();
            asynchronousServerSocketChannel.bind(new InetSocketAddress(17777));
        } catch (IOException e) {
            e.printStackTrace();
        }

        latch = new CountDownLatch(1);

        doAccept();

        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

    private void doAccept() {
        asynchronousServerSocketChannel.accept(this, new AccessCompleteHandler());
    }


    class AccessCompleteHandler implements CompletionHandler<AsynchronousSocketChannel, AioServer> {

        @Override
        public void completed(AsynchronousSocketChannel channel, AioServer attachment) {
            // 当我们调用AsynchronousServerSocketChannel的accept方法后，如果有新的客户端连接接入，
            // 系统将回调我们传入的CompletionHandler实例的completed方法，表示新的客户端已经接入成功，
            // 因为一个AsynchronousServerSocket Channel可以接收成千上万个客户端，
            // 所以我们需要继续调用它的accept方法，接收其他的客户端连接，
            // 最终形成一个循环。每当接收一个客户读连接成功之后，再异步接收新的客户端连接。
            attachment.asynchronousServerSocketChannel.accept(attachment, this);
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

            //处理这个链接
            handle(channel);
        }

        @Override
        public void failed(Throwable throwable, AioServer aioServer) {

        }
    }


    class ReadCompleteHandler implements CompletionHandler<Integer, ByteBuffer> {

        private AsynchronousSocketChannel channel;

        public ReadCompleteHandler(AsynchronousSocketChannel result) {
            this.channel = result;
        }

        @Override
        public void completed(Integer result, ByteBuffer buffer) {
            //flip操作，为后续从缓冲区读取数据做准备
            if (result > 0) {
                buffer.flip();
                System.out.print("数据大小为：" + buffer.remaining());
                byte[] bytes = new byte[buffer.remaining()];
                buffer.get(bytes);
                try {
                    System.out.println("服务器接收：" + new String(bytes, "UTF-8"));
                } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                }

                buffer.clear();
                channel.read(buffer, buffer, this);
            } else {
                latch.countDown();
            }
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment) {
            try {
                this.channel.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }

    private void handle(final AsynchronousSocketChannel channel) {
        System.out.println(Thread.currentThread().getName() + ": run in handle method");
        //每个AsynchronousSocketChannel，分配一个缓冲区
        final ByteBuffer readBuffer = ByteBuffer.allocateDirect(1024);
        readBuffer.clear();
        channel.read(readBuffer, null, new CompletionHandler<Integer, Object>() {
            @Override
            public void completed(Integer count, Object attachment) {
                System.out.println(Thread.currentThread().getName() + ": run in read completed method");
                if (count > 0) {
                    try {
                        readBuffer.flip();
                        System.out.print("数据大小为：" + readBuffer.remaining());
                        byte[] bytes = new byte[readBuffer.remaining()];
                        readBuffer.get(bytes);
                        try {
                            System.out.println("服务器接收：" + new String(bytes, "UTF-8"));
                        } catch (UnsupportedEncodingException e) {
                            e.printStackTrace();
                        }
                        writeStringMessage(channel, "nihaokehuduan");
                        readBuffer.clear();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                } else {
                    try {
                        //如果客户端关闭socket，那么服务器也需要关闭，否则浪费CPU
                        channel.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
                //异步调用OS处理下个读取请求
                //这里传入this是个地雷，小心多线程
                channel.read(readBuffer, null, this);
            }

            /**
             * 服务器读失败处理
             * @param exc
             * @param attachment
             */
            @Override
            public void failed(Throwable exc, Object attachment) {
                System.out.println("server read failed: " + exc);
                if (channel != null) {
                    try {
                        channel.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }

        });
    }


    private void writeMessage(final AsynchronousSocketChannel channel, final ByteBuffer buffer) {
        boolean threadShouldWrite = false;

        synchronized (queue) {
            queue.add(buffer);
            // Currently no thread writing, make this thread dispatch a write
            if (!writing) {
                writing = true;
                threadShouldWrite = true;
            }
        }

        if (threadShouldWrite) {
            writeFromQueue(channel);
        }
    }

    private void writeFromQueue(final AsynchronousSocketChannel channel) {
        ByteBuffer buffer;

        synchronized (queue) {
            buffer = queue.poll();
            if (buffer == null) {
                writing = false;
            }
        }

        // No new data in buffer to write
        if (writing) {
            writeBuffer(channel, buffer);
        }
    }

    private void writeBuffer(final AsynchronousSocketChannel channel, ByteBuffer buffer) {
        channel.write(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer result, ByteBuffer buffer) {
                if (buffer.hasRemaining()) {
                    channel.write(buffer, buffer, this);
                } else {
                    // Go back and check if there is new data to write
                    writeFromQueue(channel);
                }
            }

            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                System.out.println("server write failed: " + exc);
            }
        });
    }

    /**
     * Sends a message
     *
     * @param msg
     */
    private void writeStringMessage(final AsynchronousSocketChannel channel, String msg) {
        writeMessage(channel, ByteBuffer.wrap(msg.getBytes()));
    }


    public static void main(String[] args) {
        new Thread(() -> {
            AioServer server = new AioServer();
            server.start();
        }).start();

    }
}



```

- aio客户端demo
```
package com.janhe.netty_learnning.aio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.CountDownLatch;

/**
 * @CLASSNAME AioClient
 * @Description aio 客户端代码
 * @Auther Jan  橙寂
 * @DATE 2019/8/28 0028 15:51
 */

public class AioClient {

    final AsynchronousSocketChannel channel = AsynchronousSocketChannel.open();

    CountDownLatch latch = new CountDownLatch(1);

    public AioClient() throws IOException {
    }

    public void start() {
        channel.connect(new InetSocketAddress("127.0.0.1", 17777), null, new CompletionHandler() {

            @Override
            public void completed(Object result, Object attachment) {
                System.out.print("链接成功");
                //final ByteBuffer byteBuffer = ByteBuffer.wrap("12312414".getBytes());
                byte[] bytes = "1231231231".getBytes();
                ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                byteBuffer.put(bytes);

                byteBuffer.flip();

                channel.write(byteBuffer, byteBuffer, new CompletionHandler<Integer, ByteBuffer>() {

                    @Override
                    public void completed(Integer result, ByteBuffer attachment) {
                        if (attachment.hasRemaining()) {
                            channel.write(attachment, attachment, this);
                        } else {
                            System.out.print("发送成功");
                            ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                            channel.read(readBuffer, readBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                                @Override
                                public void completed(Integer result, ByteBuffer attachment) {

                                    //把缓存区从写转换到读模式
                                    attachment.flip();
                                    byte[] bytes = new byte[attachment.remaining()];
                                    attachment.get(bytes);

                                    System.out.print("接受信息：" + new String(bytes));

                                    //给服务端发消息
                                    doWrite("hahah");
                                    // latch.countDown();
                                }

                                @Override
                                public void failed(Throwable exc, ByteBuffer attachment) {
                                    exc.printStackTrace();

                                }
                            });
                        }
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer attachment) {
                        exc.printStackTrace();

                    }
                });
            }

            @Override
            public void failed(Throwable exc, Object attachment) {
                exc.printStackTrace();
            }
        });

        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private void doWrite(String data) {
        ByteBuffer byteBuffer = ByteBuffer.wrap(data.getBytes());
        channel.write(byteBuffer, byteBuffer, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer result, ByteBuffer attachment) {
                attachment.flip();
                if (attachment.hasRemaining()) {
                    channel.write(attachment, attachment, this);
                } else {
                    latch.countDown();
                }
            }

            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                try {
                    System.out.println("发送失败");
                    channel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
    }


    public static void main(String[] args) throws IOException {
        new Thread(() -> {
            AioClient client = null;
            try {
                client = new AioClient();
                client.start();
            } catch (IOException e) {
                e.printStackTrace();
            }

        }).start();

    }

}

```

## aio总结
在aio中基于**reactor**（反应堆）事件驱动模型来实现。但是编程难度还是有点大，要对多线程，跟nio有足够的了解才能封装个不错的框架。但是还是能查询出优点那就是基于**reactor**的设计还是很好用的。
- 事件驱动模式:
发生事件，主线程把事件放入事件队列，在另外线程不断循环消费事件列表中的事件，调用事件对应的处理逻辑处理事件。事件驱动方式也被称为消息通知方式，其实是设计模式中观察者模式的思路。

接下来我会带大家一起去了解netty，netty的线程模型就是事件驱动模型。在netty我们更多的关注的是netty的组件的使用。让我们网络通信的编程变得更简单。
