
---
  title: netty框架及原理解析
  date: {{ date }}   
  categories: ['网络通信']
  tags: ['Java','网络通信','netty']       
  comments: true    
  img:             
---

本篇首发于[橙寂博客](http://www.luckyhe.com/post/38.html)转载请加上此标示。
正式开始了netty的学习，netty是基于nio上的一个框架。期间翻阅了很多文档以及资料。先推荐给大家。

## 相关文档
- Netty源码在线阅读：
[Netty-4.1.x地址](http://docs.52im.net/extend/docs/src/netty4_1/)
[Netty-4.0.x地址](http://docs.52im.net/extend/docs/src/netty4/)
[Netty-3.x地址](http://docs.52im.net/extend/docs/src/netty3/)
- Netty在线API文档：
[Netty-4.1.x API文档](http://docs.52im.net/extend/docs/api/netty4_1/)
[Netty-4.0.x API文档](http://docs.52im.net/extend/docs/api/netty4/)
[Netty-3.x API文档](http://docs.52im.net/extend/docs/api/netty3/)

- 优秀博客
[新手入门：目前为止最透彻的的Netty高性能原理和框架架构解析](https://www.cnblogs.com/imstudy/p/9908791.html)

## netty概述

因为nio编写起来很困难。如果不熟悉很容易就会出错。Netty 对 JDK 自带的 NIO 的 API 进行了封装。完美的解决的nio的问题。

- **Netty的主要特点有**：
- 1）设计优雅：适用于各种传输类型的统一 API 阻塞和非阻塞 Socket；基于灵活且可扩展的事件模型，可以清晰地分离关注点；高度可定制的线程模型 - 单线程，一个或多个线程池；真正的无连接数据报套接字支持（自 3.1 起）。
- 2）使用方便：详细记录的 Javadoc，用户指南和示例；没有其他依赖项，JDK 5（Netty 3.x）或 6（Netty 4.x）就足够了。
- 3）高性能、吞吐量更高：延迟更低；减少资源消耗；最小化不必要的内存复制。
- 4）安全：完整的 SSL/TLS 和 StartTLS 支持。5）社区活跃、不断更新：社区活跃，版本迭代周期短，发现的 Bug 可以被及时修复，同时，更多的新功能会被加入。


## Netty 常见使用场景
- 1）互联网行业：在分布式系统中，各个节点之间需要远程服务调用，高性能的 RPC 框架必不可少，Netty 作为异步高性能的通信框架，往往作为基础通信组件被这些 RPC 框架使用。典型的应用有：阿里分布式服务框架 Dubbo 的 RPC 框架使用 Dubbo 协议进行节点间通信，Dubbo 协议默认使用 Netty 作为基础通信组件，用于实现各进程节点之间的内部通信。
- 2）游戏行业：无论是手游服务端还是大型的网络游戏，Java 语言得到了越来越广泛的应用。Netty 作为高性能的基础通信组件，它本身提供了 TCP/UDP 和 HTTP 协议栈。非常方便定制和开发私有协议栈，账号登录服务器，地图服务器之间可以方便的通过 Netty 进行高性能的通信
- 3）大数据领域：经典的 Hadoop 的高性能通信和序列化组件 Avro 的 RPC 框架，默认采用 Netty 进行跨界点通信，它的 Netty Service 基于 Netty 框架二次封装实现。

## Netty 架构

Netty 作为异步事件驱动的网络，高性能之处主要来自于其 I/O 模型和线程处理模型，前者决定如何收发数据，后者决定如何处理数据。

-  I/O 复用模型
netty的io模型是 I/O 复用模型。也就是nio中selector那一套。我在这篇文章对io与nio已经写过了。

#### 线程模型
netty的线程模型是**事件处理模型**
通常，我们设计一个**事件处理模型**的程序有两种思路：
-  1）轮询方式：线程不断轮询访问相关事件发生源有没有发生事件，有发生事件就调用事件处理逻辑；（nio）
- 2）事件驱动方式：发生事件，主线程把事件放入事件队列，在另外线程不断循环消费事件列表中的事件，调用事件对应的处理逻辑处理事件。事件驱动方式也被称为消息通知方式，其实是设计模式中观察者模式的思路。

- 事件驱动模型（图）
![932eee29f89e076c26c8e0c5c47b74a7.png](en-resource://database/619:1)
主要包括 4 个基本组件：
1）事件队列（event queue）：接收事件的入口，存储待处理事件；
2）分发器（event mediator）：将不同的事件分发到不同的业务逻辑单元；
3）事件通道（event channel）：分发器与处理器之间的联系渠道；
4）事件处理器（event processor）：实现业务逻辑，处理完成后会发出事件，触发下一步操作。
可以看出，相对传统轮询模式，事件驱动有如下优点：
1）可扩展性好：分布式的异步架构，事件处理器之间高度解耦，可以方便扩展事件处理逻辑；
2）高性能：基于队列暂存事件，能方便并行异步处理事件。

-  Reactor模型
Reactor 是反应堆的意思，Reactor 模型是指通过一个或多个输入同时传递给服务处理器的服务请求的事件驱动处理模式。
服务端程序处理传入多路请求，并将它们同步分派给请求对应的处理线程，Reactor 模式也叫 Dispatcher 模式，即 I/O 多了复用统一监听事件，收到事件后分发(Dispatch 给某进程)，是编写高性能网络服务器的必备技术之一。
Reactor 模型中有 2 个关键组成：
1）Reactor：Reactor 在一个单独的线程中运行，负责监听和分发事件，分发给适当的处理程序来对 IO 事件做出反应。它就像公司的电话接线员，它接听来自客户的电话并将线路转移到适当的联系人；
2）Handlers：处理程序执行 I/O 事件要完成的实际事件，类似于客户想要与之交谈的公司中的实际官员。Reactor 通过调度适当的处理程序来响应 I/O 事件，处理程序执行非阻塞操作。
![b61f90a34a93c942ff54bdfc296e722c.png](en-resource://database/617:1)

- netty模型
Netty 主要基于主从 Reactors 多线程模型（如下图）做了一定的修改，其中主从 Reactor 多线程模型有多个 Reactor：
1）MainReactor 负责客户端的连接请求，并将请求转交给 SubReactor；
2）SubReactor 负责相应通道的 IO 读写请求；
3）非 IO 请求（具体逻辑处理）的任务则会直接写入队列，等待 worker threads 进行处理。
这里引用 Doug Lee 大神的 Reactor 介绍——Scalable IO in Java 里面关于主从 Reactor 多线程模型的图：
![7ef03d8345242cad7ff2d476597ea17a.png](en-resource://database/621:1)

#### netty主要组件
- 【Bootstrap、ServerBootstrap】：
Bootstrap 意思是引导，一个 Netty 应用通常由一个 Bootstrap 开始，主要作用是配置整个 Netty 程序，串联各个组件，Netty 中 Bootstrap 类是客户端程序的启动引导类，ServerBootstrap 是服务端启动引导类。

- 【Future、ChannelFuture】：
正如前面介绍，在 Netty 中所有的 IO 操作都是异步的，不能立刻得知消息是否被正确处理。
但是可以过一会等它执行完成或者直接注册一个监听，具体的实现就是通过 Future 和 ChannelFutures，他们可以注册一个监听，当操作执行成功或失败时监听会自动触发注册的监听事件。

- Channel】：
Netty 网络通信的组件，能够用于执行网络 I/O 操作。Channel 为用户提供：
1）当前网络连接的通道的状态（例如是否打开？是否已连接？）
2）网络连接的配置参数 （例如接收缓冲区大小）
3）提供异步的网络 I/O 操作(如建立连接，读写，绑定端口)，异步调用意味着任何 I/O 调用都将立即返回，并且不保证在调用结束时所请求的 I/O 操作已完成。
4）调用立即返回一个 ChannelFuture 实例，通过注册监听器到 ChannelFuture 上，可以 I/O 操作成功、失败或取消时回调通知调用方。
5）支持关联 I/O 操作与对应的处理程序。

不同协议、不同的阻塞类型的连接都有不同的 Channel 类型与之对应。下面是一些常用的 Channel 类型：
>NioSocketChannel，异步的客户端 TCP Socket 连接。NioServerSocketChannel，异步的服务器端 TCP Socket 连接。NioDatagramChannel，异步的 UDP 连接。
NioSctpChannel，异步的客户端 Sctp 连接。
NioSctpServerChannel，异步的 Sctp 服务器端连接，这些通道涵盖了 UDP 和 TCP 网络 IO 以及文件 IO。

- 【Selector】：
Netty 基于 Selector 对象实现 I/O 多路复用，通过 Selector 一个线程可以监听多个连接的 Channel 事件。当向一个 Selector 中注册 Channel 后，Selector 内部的机制就可以自动不断地查询(Select) 这些注册的 Channel 是否有已就绪的 I/O 事件（例如可读，可写，网络连接完成等），这样程序就可以很简单地使用一个线程高效地管理多个 Channel

- 【NioEventLoop】：
NioEventLoop 中维护了一个线程和任务队列，支持异步提交执行任务，线程启动时会调用 NioEventLoop 的 run 方法，执行 I/O 任务和非 I/O 任务：I/O 任务，即 selectionKey 中 ready 的事件，如 accept、connect、read、write 等，由 processSelectedKeys 方法触发。非 IO 任务，添加到 taskQueue 中的任务，如 register0、bind0 等任务，由 runAllTasks 方法触发。两种任务的执行时间比由变量 ioRatio 控制，默认为 50，则表示允许非 IO 任务执行的时间与 IO 任务的执行时间相等。

- 【NioEventLoopGroup】：
NioEventLoopGroup，主要管理 eventLoop 的生命周期，可以理解为一个线程池，内部维护了一组线程，每个线程(NioEventLoop)负责处理多个 Channel 上的事件，而一个 Channel 只对应于一个线程。
- 【ChannelHandler】：
ChannelHandler 是一个接口，处理 I/O 事件或拦截 I/O 操作，并将其转发到其 ChannelPipeline(业务处理链)中的下一个处理程序。ChannelHandler 本身并没有提供很多方法，因为这个接口有许多的方法需要实现，方便使用期间，可以继承它的子类：
>ChannelInboundHandler 用于处理入站 I/O 事件。ChannelOutboundHandler 用于处理出站 I/O 操作。

- 【ChannelHandlerContext】：
保存 Channel 相关的所有上下文信息，同时关联一个 ChannelHandler 对象


- ChannelPipline】：
保存 ChannelHandler 的 List，用于处理或拦截 Channel 的入站事件和出站操作。。


#### netty服务端demo
>   //服务端的引导  也就是服务器的开头
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        //这个负责创建连接
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        //负责处理客户端
        NioEventLoopGroup workerGroup = new NioEventLoopGroup();
        //指定group  group相等于一个线程池 bossGroup 负责处理连接   workerGroup 负责处理客户端
        serverBootstrap.group(bossGroup, workerGroup)
                //设置服务端的channel
                .channel(NioServerSocketChannel.class)
                //服务端的channel的处理连接的handler
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline().
                                addLast(new StringDecoder())
                                // //客户端的channel的处理连接的handler
                                .addLast(new SimpleChannelInboundHandler<String>() {
                                    @Override
                                    protected void channelRead0(ChannelHandlerContext channelHandlerContext, String str) throws Exception {
                                        System.out.println(str);
                                    }
                                });
                    }
                })
                //设置一些连接参数
                .option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .bind(8000).sync().addListener(future -> {
            if(future.isSuccess()) {
                System.out.println(new Date() + ": 端口["+ port + "]绑定成功!");
            } else{
                System.err.println("端口["+ port + "]绑定失败!");
            }
        });;

- 服务端流程
1. 创建引导ServerBootstrap
2. 创建并指定线程组NioEventLoopGroup （bossGroup，workerGroup）
3. 指定服务端的chanel（NioServerSocketChannel）在这一步netty已经去注册了监听事件
4. 指定服务端连接事件的handler处理器
5. 指定客户端读事件的的处理器
6. 设置连接参数
7. 绑定端ip以及端口以及处理方式（sync）

***
## 总结
在netty的学习中，我们对其整个运行过程以及组件进行了了解。netty与你哦比使用起来简便了很多。降低了门槛。并且解决了nio留下的一些问题。但是netty这个框架的本质还是nio。只是netty的那几大组件把他封装起来了。但是要写出一个很好的服务器还是如果需要去深人源码。
