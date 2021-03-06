﻿---
  title: 消息队列的概述(MQ)
  date: {{ date }}   
  categories: ['后端'] 
  tags: ['mq','Java']       
  comments: true    
  img:             
---
# 绪言
>目前消息对列的使用还是很广泛的，很多公司对这一块技术都会有要求。比如kafka，activeMQ，RabbitMQ是目前使用较多消息中间件。博主目前使用过activeMQ跟RabbitMQ用起来差别也不大。所以这边看公司需要掌握其中一种就好了。
# 什么是消息队列

>消息（Message）是指在应用之间传送的数据，消息可以非常简单，比如只包含文本字符串，也可以更复杂，可能包含嵌入对象。 
>消息队列（Message Queue）是一种应用间的通信方式，消息发送后可以立即返回，有消息系统来确保信息的可靠专递，消息发布者只管把消息发布到MQ中而不管谁来取，消息使用者只管从MQ中取消息而不管谁发布的，这样发布者和使用者都不用知道对方的存在。

#  消息队列的主要形式

 1. 队列(quene) 点对点式
 >消息发送者发送消息，消息代理将其放入一个队列中，消息接收者从队列中获取消息内容，消息读取后被移出队列
消息只有唯一的发送者和接受者，但并不是说只能有一个接收者通俗来说就是一对一的发消息
举例子:qq私聊，微信私聊
 2. 主题（topic）发布订阅式 
 >发送者（发布者）发送消息到主题，多个接收者（订阅者）监听（订阅）这个主题，那么就会在消息到达时同时收到消息
举例子 ：公众号发了个信息然后你关注了的人都能收到，微信群发。

# 消息队列协议
**1.JMS（Java Message Service）JAVA消息服务**
>基于JVM消息代理的规范。ActiveMQ、HornetMQ是JMS实现

**2.AMQP（Advanced Message Queuing Protocol）**
>高级消息队列协议，也是一个消息代理的规范，兼容JMS
>RabbitMQ是AMQP的实现

**  对比 **
这两种规范在spring中都是支持的，所以随便使用哪个都是可以的
我这边来说下他们的不同
1.JMS是基于java api定义的所以他跨平台性较差
2.AMQP它是一种网络线级协议所以他是跨平台的，跨语言特性较好
3.AMQP中引用了交换机跟路由件的概念。提供的消息模型更多详情在下篇对rabbit的博客中讲解
3.AMQP本质是基于byte传输的，支持序列化跟发序列化，在spring中集成的很好。

![2.png](http://image.luckyhe.com/mblog/0d145912c4cb1a06630fd888973af132.png)
# 消息队列能干什么
####  异步处理
  场景:有这么一个场景就是注册了账号，需要发短信，又需要发送邮件。要以最快的速度去完成大家  会怎么去做。
  解决方案:
>第一种方法就是顺序先发短信，在发邮件 
>第二种 使用多线程
>第三种 使用消息队列
>
#### 应用解耦
场景:在一个分布式应用中新增了一个商品 在搜索模块中要把这个商品静态化然后还要把这个商品加入到solr的索引库中。注意这是两个模块的功能。
解决方案:
>新增了商品发送一个消息，搜索模块订阅这个消息然后做功能。当然这里可能你别的模块也需要用到这个消息，这时只需要让别的模块也订阅这个消息就好了。
>
#### 处理高并发
场景:秒杀场景
解决方案:
>用户下单了发送一个消息到消息队列中，这时相应的服务在去消息队列中调用做相应的功能就好了。

我的下一篇文章会针对RabbitMQ做下介绍。这是我的个人站点希望大佬指导批评[橙寂博客](http://www.luckyhe.com)
