《关于oauth2一些浅谈》首发[橙寂博客](http://www.luckyhe.com/post/79.html)转发请加此提示

## 关于oauth2一些浅谈

#### 1.引入

关于`Oauth2`首先概念上的一些东西我想纠正下:就是`Oauth2`是目前比较完善的一种权限认证规范(思想),他并不是一门技术。只是当前`Spring`基于这个规范,基于`Spring Security`写了一套方便广大开发者，比较完善的一个类库。
本文会从规范的角度去讲一个完整`Outh2`规范是咋样的。


#### 2.概述

`OAuth`（开放授权）是一个开放标准，允许用户授权第三方移动应用访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方移动应用或分享他们数据的所有内容，`OAuth2.0`是OAuth协议的延续版本，但不向后兼容OAuth 1.0即完全废止了OAuth1.


简单说，OAuth 就是一种授权机制。数据的所有者告诉系统，同意授权第三方应用进入系统，获取这些数据。系统从而产生一个短期的进入令牌（token），用来代替密码，供第三方应用使用。


#### 3.应用场景

###### 生活场景

我们首先来说一个现实中的问题一个小区。在小区里面涉及到了这几个对象，业主，保安，门禁系统，临时访问者(比如外卖员，快递员)。

![小区门禁图](G:\Learnning\pic\Security\小区门禁图.jpg)

在目前的社会中，业主通过刷卡，刷脸等方式是可以自由出去小区的。因为小区保安已经对他们进行了授权。但是外卖员以及快递员普遍是进不去小区的。这时候我们需要开始出一套可以对外进行授权，但是必须严格控制
外卖员的权限（只能进入大门进入具体单元)。

这时候我们就可以使用`Outh2`思想来对门禁系统进行改革。首先保安就得对外卖员进行授权。权限的大小是由保安来决定的。授权完毕后给他们发一个令牌。然后外卖员拿着授权令牌他就可以进去小区了。但是他进入不了具体的楼栋。我们对他授权仅限于进入小区大门。

但是其中有一些外卖员为了方便把令牌随便借给别人。这时候我们对于这个机制进行了进一步优化我们给授权加了期限，如果期限到期你就得拿着你的令牌过来。我对于进行新的认证。然后重新颁发令牌。

###### 编码场景

举个例子，qq的认证。现在很多系统都需要支持第三方认证。

![腾讯qq认证](G:\Learnning\pic\Security\腾讯qq认证.jpg)


在我们系统中我们点击qq授权链接后，qq便会把授权码发给我们的系统。我们通过这个授权码便可以去拿到qq的令牌。拿到令牌我们便可以拿到他的一些基本信息。最终实现qq登录业务系统的效果。
在这个环节中使用到的便是`Outh2`中的授权码模式。


#### Outh2中的一些规范


- **OAuth 2.0的运行流程如下图，摘自RFC 6749。**

![运行流程](G:\Learnning\pic\Security\认证流程.jpg)
>
- （A）用户打开客户端以后，客户端要求用户给予授权。
- （B）用户同意给予客户端授权。
- （C）客户端使用上一步获得的授权，向认证服务器申请令牌。
- （D）认证服务器对客户端进行认证以后，确认无误，同意发放令牌。
- （E）客户端使用令牌，向资源服务器申请获取资源。
- （F）资源服务器确认令牌无误，同意向客户端开放资
源。


- **授权模式**

`OAuth2`有四种模式分别是

1. 授权码模式（authorization code）
2.  简化模式（implicit）
3. 密码模式（resource owner password credentials）
4. 客户端模式（client credentials）

注意，不管哪一种授权方式，第三方应用申请令牌之前，都必须先到系统备案，说明自己的身份，然后会拿到两个身份识别码：客户端 ID（`client_ID`）和客户端密钥（`client_secret`）。这是为了防止令牌被滥用，没有备案过的第三方应用，是不会拿到令牌的。我们最常用的两种是`授权码模式`和`密码模式`

**1. 授权码模式**

1.第三方应用首先带上已注册的`client_ID`客户端id以及`client_secret`密钥我们这边拿`github`举例。
```java
https://github.com/login/oauth/authorize?
  client_id=7e015d8ce32370079895&client_secret=xxxxx
  redirect_uri=http://localhost:8080/oauth/redirect
```
其中`redirect_uri`是你本机项目接受code的url。在很多系统中`client_id`与`client_secret`都不是明文传输的,我们公司就是加密的。但加密的前提必须双方沟通好了的加密方式。

2. 请求认证地址后，`gitHub`会请求`redirect_uri`并携带一个code。这个code是会过期的。普遍时间为10分钟。
```
http://localhost:8080/oauth/redirect?
  code=cxadasd1e1
```

3. 拿到这个code后，去请求`gitHub`拿取token

```java
https://github.com/login/oauth/access_token? +
    client_id=${clientID}& +
    client_secret=${clientSecret}&+
    code=${requestToken}
  ```

4. 拿到token后把token存起来。然后去拿取想要的信息。这一步做完后,所有的业务就是你自己得事了。`token`给你了，你想要咋搞就咋搞。

**2.密码模式**

这种模式一般适用于内部系统。

1.内部应用首先带上已注册的`client_ID`客户端id以及`client_secret`密钥。`userName`用户名,`password`密码、去请求认证服务器。默认地址一般为`/oauth/token`（后续文章聊`Security`我会告诉你为啥默认地址是这个）

2.请求成功后，认证服务器会返回一个`token`给你。

3.拿取`token`去访问你想要的api接口。

**3. 简化模式**

这种模式一般适用于没有后端的系统。

因为有些项目没有后端，所以认证服务器会直接把`token`发给第三方系统。

1. 带上已注册的`client_ID`客户端id以及`client_secret`密钥。去请求认证服务器。
···java
https://www.luckyhe.com/oauth/authorize?
  response_type=token&
  client_id=CLIENT_ID&
  redirect_uri=CALLBACK_URL&
  scope=read
···
2. 这是认证服务器会直接带上`token`来访问你提供的`redirect_uri`。


**4. 客户端模式**

这种模式一般适用于没有后端的系统。

最后一种方式是凭证式（client credentials），适用于没有前端的命令行应用，即在命令行下请求令牌。

第一步，A 应用在命令行向 B 发出请求。
```
https://www.luckyhe.com/token?
  grant_type=client_credentials&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET
```
上面 URL 中，`grant_type`参数等于`client_credentials`表示采用凭证式，`client_id`和`client_secret`用来让 B 确认 A 的身份。

第二步，B 网站验证通过以后，直接返回令牌。

这种方式给出的令牌，是针对第三方应用的，而不是针对用户的，即有可能多个用户共享同一个令牌。也就是说这个令牌拿到后，A网站可以把这个`token`给`C`,`D`网站用。

5. **更新令牌**

以上四种模式令牌都是有期限的，如果让用户重新走一遍上面的流程，再申请一个新的令牌，很可能体验不好，而且也没有必要。*OAuth 2.0* 允许用户自动更新令牌。

具体方法是，B 网站颁发令牌的时候，一次性颁发两个令牌，一个用于获取数据，另一个用于获取新的令牌（refresh token 字段）。令牌到期前，用户使用 refresh token 发一个请求，去更新令牌。

```java
https://www.luckyhe.com/oauth/token?
  grant_type=refresh_token&
  client_id=CLIENT_ID&
  client_secret=CLIENT_SECRET&
  refresh_token=REFRESH_TOKEN
```
上面 URL 中，`grant_type`参数为`refresh_token`表示要求更新令牌，`client_id`参数和`client_secret`参数用于确认身份，`refresh_token`参数就是用于更新令牌的令牌。

B 网站验证通过以后，就会颁发新的令牌。

#### 总结

本篇文章主要介绍了，什么是`Oauth2`,以及`Oauth2`的一些规范。在我了解到的很多公司中都搭建了`Oauth2`认证中心,我司也不例外。但是每个公司都扩展了自己的一些业务在里面。很多公司甚至在获取`token`这个步骤里。把用户信息直接返回出来了。下篇文章讲一下我自己对`Spring Security`的一些理解，以及怎么样扩展出属于自己的个性的`Oauth2`认证中心。
