---
  title: 史上最详细lunix上安装mysql(centos7)
  date: {{ date }}   
  categories: ['lunix'] 
  tags: ['lunix','centsos7','mysql']       
  comments: true    
  img:             
---
## 绪言
centos7跟其他版本有些区别，所以自己写了这么一篇。网上很多都是没用的，步骤不全。我这篇超级全。下面开始安装
 首先一个全新的系统 yum update 一下 升级一下依赖包。
 ## 1.创建目录
  mkdir /usr/local/mysql5.6
   ##    2.上传mysql镜像文件
  这边我使用的是winscp连接工具直接上传。不同工具上传方式不同的。我上传的是rpm包所以不需要解压。如果是tar.gz结尾。需要解压 
解压mysql
tar -xvf MySQL-5.6.44-1.el7.x86_64.rpm-bundle.tar //注意，是-xvf不是-zxvf

 ## 3.卸载centos自带的mariadb
  rpm -qa | grep mariadb
  
  rpm -e --nodeps  mariadb-libs-5.5.60-1.el7_5.x86_64

 ## 4.卸载mysql
  rpm -qa | grep mysql
  
  rpm -e --nodeps 文件名
检查服务
chkconfig --list | grep -i mysql //查看服务

chkconfig --del mysql

 ##  5.安装依赖
yum install perl

yum install net-tools

yum -y install autoconf //此包安装时会安装Data:Dumper模块

 ##  6.增加mysq用户组（非必须）
groupadd mysql
useradd -r -g mysql mysql //创建用户并把该用户加入到组mysql，这里的 -r是指该用户是内部用户，不允许外部登录
passwd mysql //给用户mysql设置密码，需要输入2次 这一项非必须。、
 ##  7.开始安装（官网下载）
 [下载mysql](https://www.mysql.com/downloads/)
 
rpm -ivh MySQL-client-5.6.44-1.el7.x86_64.rpm

rpm -ivh MySQL-devel-5.6.44-1.el7.x86_64.rpm

rpm -ivh MySQL-server-5.6.44-1.el7.x86_64.rpm

## 8.启动mysql
检查状态：service mysql status

启动服务：service mysql start

找到默认密码  cat /root/.mysql_secret

绕过密码登录 mysqld_safe --user=mysql --skip-grant-tables --skip-networking & //绕过密码登录

登录 mysql -uroot -p   如果执行了绕过密码登录这一步不需要输入密码

修改密码：  SET PASSWORD = PASSWORD('root');

赋予所有用户远程访问权限 grant all privileges on *.* to 'root'@'%' identified by 'root' with grant option;

刷新权限 flush privileges;
## 9.开放3306端口 

firewall-cmd --state

firewall-cmd --permanent --zone=public --add-port=3306/tcp //添加3306端口

firewall-cmd --reload
## 11.把mysql加入开机自启

chkconfig --list mysql //查看mysql服务

chkconfig mysqld on //开启MySQL服务自动开启命令

chkconfig mysql on //开启MySQL服务自动开启命令

## 11.mysql集合重要目录
/var/lib/mysql 数据库文件

/usr/share/mysql 命令及配置文件

/usr/bin mysqladmin、mysqldump等命令

/usr/my.cnf 核心配置文件







