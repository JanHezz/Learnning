《vue环境的搭建（一）》首发[橙寂博客](http://www.luckyhe.com/post/14.html)转发请加此提示

## <center>vue环境的搭建</center>
#### 前言
>vue是现在前端框架非常火的一个东西,所以我也想学习下看看这究竟是个什么东西。刚开始很闷进去vue官网看了下文档。对于刚入门者是不太友好的（需要一定的node.js的开发经验不然大家会跟我一样蒙）。后面经过自己的查阅总算是找到了门道。
#### 安装nodejs环境
>1.下载[http://nodejs.cn/download/](https://)先进去下载我下的是10.15版本![1.png](http://image.luckyhe.com/mblog/82e33edfae5835cea9b71282db374ebb.png)
>2.安装 打开window的是命令行工具（作为程序员这个就不需要交了吧）

>输入node -v查看版本号显示版本号为安装成功了
![2.png](http://image.luckyhe.com/mblog/c211fc18ef32b5d0b1007f4e32cb8d43.png)
#### 更改镜像
>国内使用npm是非常慢的，所以我们这边使用淘宝的npm镜像

>npm install -g cnpm --registry=https://registry.npm.taobao.org
#### 安装vue_cli脚手架
>cnpm install --global vue-cli

>验证输入vue验证是否安装成功
#### 开始helloword
>环境装好了现在开始熟悉的helloword环节。先切换到您的项目目录切换到D盘直接输入d：然后回车。

>vue init webpack firstvue fistvue是项目名称 输入后一直回车就行了直到出现这个界面![3.png](http://image.luckyhe.com/mblog/da91b3e09b6399d6e52deb1efccf28ba.png)
>
>第一个是路由所以yes 后面几个全为no为yes也不会报错。
#### 测试
>项目文件建好了进去你的项目根文件夹
cd 路径
进去后

>cnpm install安装依赖

>cnpm run dev 运行项目项目默认是在8080端口
![4.png](http://image.luckyhe.com/mblog/2d3e964fce8be545a593e9eb302ea762.png)