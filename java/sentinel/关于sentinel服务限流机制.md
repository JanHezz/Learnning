《关于Sentinel 服务限流机制》首发[橙寂博客](http://www.luckyhe.com/post/79.html)转发请加此提示

## 关于Sentinel 服务限流机制

#### 1.介绍

 [Sentinel](https://github.com/alibaba/Sentinel) 是阿里中间件团队研发的面向分布式服务架构的轻量级高可用流量控制组件，开源的。Sentinel 主要以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度来帮助用户保护服务的稳定性。


#### 2.与Hystrix 对比

服务熔断，一直是分布式应用的热点问题，目前面向分布式应用的流量控制组件中用户量比较多的有两个，这两者都能比较容易整合到目前主流的一些分布式框架中。

1. `Hystrix` 是`Netflix`公司开源的一个老牌框架,`Netflix`还开源了`eureka`，`Zuul`等一些著名的开源框架，在分布式领域，`Netflix`是很牛逼的一个存在。
2.  `Sentinel`则为阿里开源的一个中间件。它属于是站在前人的肩膀上，出现的一款产品，所以它的功能更强大一些。在使用上也更符合国人的一些习惯和思想。

下面来看下这两个框架的对比

|                | Sentinel                                       | Hystrix                       |
| :------------- | :--------------------------------------------- | ----------------------------- |
| 隔离策略       | 基于并发数                                     | 线程池隔离/信号量隔离         |
| 熔断降级策略   | 基于响应时间或失败比率                         | 基于失败比率                  |
| 实时指标实现   | 滑动窗口                                       | 滑动窗口（基于 RxJava）       |
| 规则配置       | 支持多种数据源                                 | 支持多种数据源                |
| 扩展性         | 多个扩展点                                     | 插件的形式                    |
| 基于注解的支持 | 即将发布                                       | 支持                          |
| 调用链路信息   | 支持同步调用                                   | 不支持                        |
| 限流           | 基于 QPS / 并发数，支持基于调用关系的限流      | 不支持                        |
| 流量整形       | 支持慢启动、匀速器模式                         | 不支持                        |
| 系统负载保护   | 支持                                           | 不支持                        |
| 实时监控 API   | 各式各样                                       | 较为简单                      |
| 控制台         | 开箱即用，可配置规则、查看秒级监控、机器发现等 | 不完善                        |
| 常见框架的适配 | Servlet、Spring Cloud、Dubbo、gRPC 等          | Servlet、Spring Cloud Netflix |


#### 3.应用场景

在分布式环境中，不可避免会造成一些服务的失败。所以诞生了一些优秀的限流框架库，它们的共同特点都旨在控制分布式服务中提供更大容限和服务失败之间的相互关系。使复杂的分布式系统更具弹性。

这里我来说几个在日常开发中会遇到的一些点。

1. 服务调用时间过长
2. 高并发请求