---
  title: Lunix上安装docker
  date: {{ date }}   
  categories: ['lunix'] 
  tags: ['lunix','centsos7','docker']       
  comments: true    
  img:             
---
参考官网教程
https://docs.docker.com/install/linux/docker-ce/centos/
## 	1. 准备工作


 >1、Linux7以上或者cent OS6及以上版本
 
 >2、内核3.1.0以上
 
 >3、64位操作系统
uname - r 可以查看lunix版本内核

>4.卸载旧版本
```
sudo yum remove docker \
docker-client \ 
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
\ docker-engine
 ```
>5.安装所需的包。yum-utils提供了yum-config-manager 效用，并device-mapper-persistent-data和lvm2由需要 devicemapper存储驱动程序。

$ sudo yum install -y yum-utils \

  device-mapper-persistent-data \

  lvm2
>6.使用以下命令设置稳定存储库。

$ sudo yum-config-manager \

    --add-repo \

    https://download.docker.com/linux/centos/docker-ce.repo
	 
## 2安装docker


>1.yum安装 


yum install  docker-ce
>2. 查看版本

   docker -v    docker version 都可以
>3.启动docker
	
systemctl start docker
	 
测试docker运行成功没

       sudo docker run hello-world
        
## 安装成功

> 1. 设置开机启动


    systemctl enable docker
	 
>2.停止docker

   systemctl stop docker







