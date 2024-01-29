﻿《浅谈Cookie跨域获取》首发[橙寂博客](http://www.luckyhe.com/post/14.html)转发请加此提示

## <center>浅谈Cookie跨域获取</center>
#### 背景
>最近在接入一个第三方的单点登录平台，使用的`Oauth2`对接的，本来是没啥问题，奈何退出环节他是使用Cookie进行退出的，这时就涉及到了一个跨域问题。看下图
>
>![image-20240129101020846](G:\Learnning\pic\前端\cookie退出流程.png)
>
>由此引出本文主要讨论到的一个问题，**两个域名完全不一样的的系统，甲方能拿到乙方的cookie嘛？**
#### 何为同源策略？
>**同源策略**是一个重要的安全策略，它用于限制一个[源](https://developer.mozilla.org/zh-CN/docs/Glossary/Origin)的文档或者它加载的脚本如何能与另一个源的资源进行交互。
>
>它能帮助阻隔恶意文档，减少可能被攻击的媒介。例如，它可以防止互联网上的恶意网站在浏览器中运行 JS 脚本，从第三方网络邮件服务（用户已登录）或公司内网（因没有公共 IP 地址而受到保护，不会被攻击者直接访问）读取数据，并将这些数据转发给攻击者。

###### 源的定义

如果两个 URL 的协议、端口（如果有指定的话）和主机都相同的话，则这两个 URL 是同源的。这个方案也被称为“协议/主机/端口元组”，或者直接是“元组”。（“元组”是指一组项目构成的整体，具有双重/三重/四重/五重等通用形式。）

下表给出了与 URL http://store.company.com/dir/page.html 的源进行对比的示例：

| URL                                             | 结果 | 原因                              |
| ----------------------------------------------- | ---- | --------------------------------- |
| http://store.company.com/dir2/other.html        | 同源 | 只有路径不同                      |
| http://store.company.com/dir/inner/another.html | 同源 | 只有路径不同                      |
| https://store.company.com/secure.html           | 失败 | 协议不同                          |
| http://store.company.com:81/dir/etc.html        | 失败 | 端口不同（http:// 默认端口是 80） |
| http://news.company.com/dir/other.html          | 失败 | 主机不同                          |



#### 跨域读取cookie
基于同源策略的定义，A.com跟B.com是不同源的，理论上来说A.com是无法获取到B.com的Cookie的。那么是否有方法能够让B.com的cookie被A.com读取到呢？

**cookie**是存储在客户端上的，如果没有同源的限制，那么cookie是可以被随意获取到的。下面我简单介绍一下cookie在浏览器中的一些参数设置。以google浏览器为例子。打开F12

![image-20240129105603399](G:\Learnning\pic\前端\Cookie参数.png)



| 名字     | 意义                                                         |
| -------- | ------------------------------------------------------------ |
| name     | cookies的名字，也是cookies的唯一标识                         |
| value    | 也就是cookies的内容，这是cookies有用的内容                   |
| Domain   | cookies所属的域                                              |
| Path     | cookies所属的路径，他是属于某个路径的，/代表根路径           |
| expires  | cookies的到期时间，如果为0则永不过期                         |
| http     | 如果打钩则通过脚本无法获取，可以防止xss攻击                  |
| secure   | https连接才会传送该cookies，增强了安全性                     |
| sameSite | 可选值Strict 和 Lax，None，Strict 严格模式，不能被第三方网站获取，Lax：宽松模式，可以被第三方获取，None是最为宽松的一种设定，通常用于开放我们的服务给不同的第三方接入，同时又需要追踪用户的场景，比如广告，设置为 `None` 时需要考虑开放的安全性。 |

看到这里其实我的问题已经有了答案，大家也明白了关键点就是cookie的一些属性设置特别是**sameSite**这个属性了，也就是说把sameSite设置为**None**的话。第三方跨域请求我的退出地址就会携带上我的cookie。

#### 前端跨域的请求方式
1. **原生js**

```js
 var script = document.createElement('script');
    script.type = 'text/javascript';

    
    script.src = 'http://www.B.com:8080/loginOut?t=123&callback=handleCallback';
    document.head.appendChild(script);

    // 回调执行函数
    function handleCallback(res) {
        alert(JSON.stringify(res));
    }
```

2. **jquery Ajax实现**

```js
 $.ajax({
    url: 'http://www.B.com:8080/loginOut',
    type: 'get',
    dataType: 'jsonp',  // 请求方式为jsonp
    jsonpCallback: "handleCallback",  // 自定义回调函数名
    data: {}
});
```
3. **Vue axios实现**

```js
this.$http = axios;
this.$http.jsonp('http://www.B.com:8080/loginOut', {
    params: {},
    jsonp: 'handleCallback'
}).then((res) => {
    console.log(res); 
})
```
这里提醒一下后端提供的接口必须为get请求才可以。



#### 关于统一平台为何使Cookie用来退出？

> 这里其实我问过统一平台的开发，我问他作为统一平台为何退出的时候需要去获取别的平台作为**Cookie**，为何不使用Uid的方式，或者别的方式。
>
> 统一平台：我不清楚第三方系统的系统唯一标识是否统一是uid。有些可能是手机号有些可能是用户名。所以唯一标识采用第三方自己定义的cookie来确定。
>
> 我听完后直接回了一句：好有道理！！！(心里想不就是懒嘛！提供一个随机code，然后我再用随机码去调用认证中心获取用户信息(uid,uname)等，这样不一样可以解决这个问题嘛。）
>
> 目前用cookie的方式有个很烦的弊端：
>
> 1. 接入方必须提供https 。
>
> 2.  第三方应用登录后需要维护一个唯一的cookie。不管你系统是不是用cookie为登录标识，你都需要维护一个。

#### 总结
>在某些特定情况下不同源的网站之间Cookie是可以进行传递的。我看了很多文章都只写了如何实现跨域请求，但是没有一个人能说清**不同源的网站之间Cookie是否可以进行传递**。有很多人给的方案都是告诉我如何做单点，比如说建议我注册一个子域名的，比如说告诉我cookie不行，要使用token去做单点。但是感觉问了一圈感觉没有人理解我这个问题。希望我这篇文章能给大家一点帮助。
