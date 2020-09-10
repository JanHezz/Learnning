《微服务系列-Sentinel初体验》首发[牧马人博客](http://www.luckyhe.com/post/83.html)转发请加此提示

微服务系列断更很久了，时间久的笔者公司都换了一家了,目前这家公司用的是springalibaba全家桶。本次的主人公式`Sen`

## 前因

简单的介绍了一下我的项目结构，我们是一个微服务项目，配置中心使用的是阿里的[nacos 1.2.1](https://github.com/alibaba/nacos/tree/1.2.1)做的服务的灰度发布记住要考的。网关使用的是`gateway`做动态路由。前台使用的是`Vue`。本人负责的一个`Activiti`模块,在正式上线之前。我的项目都是能正常跑的。但是代码部署到正式机一直报一个bug。见代码
```
Caused by: java.io.FileNotFoundException: class path resource [org/springframework/security/config/annotation/authentication/configurers/GlobalAuthenticationConfigurerAdapter.class] cannot be opened because it does not exist
	at org.springframework.core.io.ClassPathResource.getInputStream(ClassPathResource.java:180)
	at org.springframework.core.type.classreading.SimpleMetadataReader.getClassReader(SimpleMetadataReader.java:56)
	at org.springframework.core.type.classreading.SimpleMetadataReader.<init>(SimpleMetadataReader.java:50)
	at org.springframework.core.type.classreading.SimpleMetadataReaderFactory.getMetadataReader(SimpleMetadataReaderFactory.java:103)
	at org.springframework.boot.type.classreading.ConcurrentReferenceCachingMetadataReaderFactory.createMetadataReader(ConcurrentReferenceCachingMetadataReaderFactory.java:86)
	at org.springframework.boot.type.classreading.ConcurrentReferenceCachingMetadataReaderFactory.getMetadataReader(ConcurrentReferenceCachingMetadataReaderFactory.java:73)
	at org.springframework.core.type.classreading.SimpleMetadataReaderFactory.getMetadataReader(SimpleMetadataReaderFactory.java:81)
	at org.springframework.context.annotation.ConfigurationClassParser.asSourceClass(ConfigurationClassParser.java:695)
	at org.springframework.context.annotation.ConfigurationClassParser$SourceClass.getSuperClass(ConfigurationClassParser.java:1009)
	at org.springframework.context.annotation.ConfigurationClassParser.doProcessConfigurationClass(ConfigurationClassParser.java:340)
	at org.springframework.context.annotation.ConfigurationClassParser.processConfigurationClass(ConfigurationClassParser.java:249)
	at org.springframework.context.annotation.ConfigurationClassParser.processMemberClasses(ConfigurationClassParser.java:371)
	at org.springframework.context.annotation.ConfigurationClassParser.doProcessConfigurationClass(ConfigurationClassParser.java:271)
	at org.springframework.context.annotation.ConfigurationClassParser.processConfigurationClass(ConfigurationClassParser.java:249)
	at org.springframework.context.annotation.ConfigurationClassParser.processImports(ConfigurationClassParser.java:599)
	... 28 common frames omitted```

```
但是测试机发版不影响。这个bug爆肝一天解决了。这个bug不玄学，有正常报错。所以我们对比了配置文件。很快定位到了问题。是`Activityi`正式环境配置文件没有排除
>org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration.class
这个问题解决完后，我并不知道有人把测试机的配置文件改了，也就是nacos配置中心的文件。


## 后果

解决完这个问题后。以为尘埃落定了。这时候玄学开始了。测试环境网关路由转发到我本机的时候一直报资源不存在的错误。也就是`404`测试机上所有服务都报这个错。后端代码没有任何报错。并且`nacos`的服务中心是能检测到该服务的。

#### 第一步
看到这个问题我第一时间把`gateway`,`nacos`,`auth`都重启一遍。让路由重新加载。这时候测试机之间访问是正常的。唯独之前出现问题的`Activiti`模块的那个服务。转发不过来。也没有任何报错。因为`nacos`中检查到了我本机的服务,这时候我并没有去怀疑我本机的服务其实没有真实启动起来.然后我爆肝一天一夜检查。检查路由，检查数据库，重启服务。重复这样的步骤。搞了一天。毫无进展。

#### 第二步

一天过去了心态小崩，因为正式环境急需发版，但是测试环境起不来。代码都写好了，没法测试。所以第二天我求助于同组的开发叫他们起一下我那个服务。最后的结果2个人可以正常访问,3个人跟我一样的问题。这时候我心里闪过一句`caonima`。啥玩意啊？？？。
然后我认认真对比了一下，同事跟我的本机代码。毫无头绪。我俩除了电脑不同，啥都是一致的。检查了一下网络，我们都是连的公司内网。如果公司人电脑都不行，那么肯定不是我的问题。但是是一半一半。崩溃边缘

#### 第三步

收拾好心情我认真对比了下，成功者跟我自己本机代码的日志。启动大部分都是一样的。但是我本机一直循环打印

```
2020-07-28 18:14:50.407 DEBUG 13064 --- [      Thread-50] o.activiti.engine.impl.db.DbSqlSession   : flush summary: 0 insert, 0 update, 0 delete.
2020-07-28 18:14:50.407 DEBUG 13064 --- [      Thread-50] o.activiti.engine.impl.db.DbSqlSession   : now executing flush...
Closing JDBC Connection [Transaction-aware proxy for target Connection [com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl@5529fd4e]]
```
就是这段鬼代码。我本机一直打印，但是成功启动的人是不会打印。但是我们代码都一样啊，怎么会出现这个问题呢。此处省略一万个caonima...

## 解决方案

Two years later我再去检查了一遍`nacos`配置中心的配置文件。然后我发现竟然有人改了配置文件。然后我仔细对比了一下改动内容。唯一不同的就是加了中文注释。这时候我报着试一试的办法。把中文注释去了。然后项目竟然起来了？？？
沃特么。`nacos`配置文件存在数据库中。我做梦也特么想不到一个中文注释会引起这个问题。这个bug其实应该是编码问题，springboot项目是会存在这个问题。但是普通项目中启动会报错。但是我启动时并无报错，至今查阅了一些资料，也没能解释这个玄学。所以这个锅就给`nacos`背吧。

## 后话

阿里的小伙伴如果看到这个文章，求求你代码多打点日志，然后把这个bug给解决下吧。这个bug告诉我一个道理,如果在英文水平允许的情况下竟然使用英文注释吧。如果你手中有玄学bug可以把文章发给我哦。