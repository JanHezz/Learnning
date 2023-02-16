---
  title: 关于我把SpringBoot应用做成docker应用这件事
  date: {{ date }}   
  categories: ['Docker']
  tags: ['Docker','Docker-compose']       
  comments: true    
  img:             
---

本篇文章会介绍使用`dockefile`方式制作镜像并启动，以及`docker-compose`配合`dockerfile`启动`SpringBoot`项目。本篇首发于[牧码人博客](http://www.luckyhe.com/post/94.html)转载请加上此标示。

## 准备工作

1. **jar包** 
2. **Dockerfile** （跟jar包放一个位置）

   ```dockerfile
   # 环境变量 容器启动后仍保留
   #ENV JAVA_OPTS="-Xms248M  -Xmx512M  -XX:PermSize=248M -XX:MaxPermSize=512M"
   
   # 指定容器运行时的参数，容器启动后就失效了，必须在from之前定义
   # ARG JAVA_OPTS="-Xms248M  -Xmx512M  -XX:PermSize=248M -XX:MaxPermSize=512M"
   
   # 以什么镜像为前提 由于是springboot应用所以需要现有java环境
   FROM java:8
   
   # 镜像构建时就会执行命令
   #RUN ["/bin/bash", "-c", "echo hello"]
   
   #容器启动时运行的命令
   # 指定这个容器启动的时候要运行的命令，不可以追加命令
   #CMD
   
   #作者
   MAINTAINER jan
   
   ## 容器的工作地址 相当于cd 
   WORKDIR /app/mblog
   
   ## 把本地的包添加到工作目录下并改名  还有一个copy原理是一样的
   ADD blog-latest.jar app.jar
   
   ##  挂载数据卷到宿主机中，容器不存储数据 
   ## 关于VOLUME如果想了解更深看这个 https://cloud.tencent.com/developer/article/1896358
   VOLUME /tmp
   
   LABEL  version="1.0" description="牧码人博客" by="牧码人" 
   
   ## 指定容器运行时的user
   USER root
   
   ## 镜像触发器
   ## 当所构建的镜像被用做其它镜像的基础镜像，该镜像中的触发器将会被钥触发
   ## ONBUILD RUN /usr/local/bin/python-build --dir /app/src
   
   ## 暴露的端口
   EXPOSE 8080
   
   # 指定运行容器启动过程执行命令，覆盖CMD参数
   # 指定这个容器启动的时候要运行的命令，可以追加命令(与CMD的区别)
   ENTRYPOINT ["java","-jar","/app/mblog/app.jar"]
   
   
   ```

3.**docker-compose.yml**(非必须，由于本人喜欢用docker-compose所以才使用了)
   简述下啥是`docker-compose`

> Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration. 
> Compose是一个用于定义和运行多容器Docker应用程序的工具。使用Compose，您可以使用一个YAML文件来配置应用程序的服务。然后，使用一个命令创建并启动配置中的所有服务
> 
   人话：大概就是使用docker-compose可以管理多容器的docker程序，使用一个yaml文件配置下就可以。举个例子：一个简单的java程序需要jdk环境，数据库环境，redis环境。如果不使用docker-compose你就需要多次运行。如果使用docker-compose你只需要在yaml配置好，然后docker-compose  up 一下。多个应用便会同时启动。 
   本篇示例只讲了docker-compose管理单个容器的情况，主要是了解一下yaml的配置。使用yaml配置个人觉得比较优雅，如果不使用它那么你的docker启动命令是这样的 docker run --net=host -p 8269:8080 -v /app/mblog/storage/:/app/mblog/storage/ -d mblog:latest  参数一多这个docker命令便非常长了。

   ```dockerfile
   # 版本根据你的docker版本来的，目前主流应该都是3.几的版本
   version: '3.8
   services:
     mblog:
       ## 启动前，先执行build构建镜像
       ## 等同于 docker build -t mblog:latest .
       build:
         context: .
         dockerfile: ./Dockerfile
       # 镜像名
       image: mblog:latest
       # 容器名
       container_name: mblog
       # 自动重启
       restart: always
       #端口映射
       ports:
         - 8269:8080
       # 环境变量  
       environment:
         - spring.profiles.active=mysql
         - JVM_OPTS=-server -Xmx218M -Xms1024M -XX:+DisableExplicitGC -XX:SurvivorRatio=1 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSParallelRemarkEnabled -XX:+CMSClassUnloadingEnabled -XX:LargePageSizeInBytes=128M -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+PrintClassHistogram -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC -XX:+HeapDumpOnOutOfMemoryError -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m -Xloggc:/logs/gc.log
       # 目录映射  
       volumes:
         - /app/mblog/storage/:/app/mblog/storage/
   ```

## 启动命令

**原生docker方式制作镜像并启动** 

构建镜像命令：

> docker build -t mblog:latest . 

启动命令

> docker run --net=host -p 8269:8080 -v /app/mblog/storage/:/app/mblog/storage/ -d mblog:latest

**docker-compose方式启动项目**

> docker-compose up -d --build

随后docker ps 查看容器是否启动成功即可
