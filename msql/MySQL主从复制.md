
《 MySQL主从复制》首发[橙寂博客](http://www.luckyhe.com/post/71.html)转发请加此提示


# MySQL主从复制

mysql服务器的主从配置，这样可以实现读写分离，也可以在主库挂掉后从备用库中恢复。
需要两台机器，安装mysql，两台机器要在相通的局域网内，可以分布在不同的服务器上，也可以在一台服务器上启动多个服务。

 

主机A: 192.168.1.100
从机B:192.168.1.101
可以有多台从机

### 1、先登录主机 A，在主服务器上，设置一个从数据库的账户，使用**REPLICATION SLAVE（从复制）**赋予权限，如：
```mysql

mysql>GRANT REPLICATION SLAVE ON *.* TO 'slave'@'192.168.144.131' IDENTIFIED BY '123456';
FLUSH PRIVILEGES；
```
赋予从机权限，有多台从机，就执行多次。

### 2、 打开主机`A`的`my.cnf`，输入如下：（修改主数据库的配置文件my.cnf，开启BINLOG，并设置server-id的值，修改之后必须重启Mysql服务）

**server-id = 1    #主机标示，整数**

**log_bin = /var/log/mysql/mysql-bin.log   #确保此文件可写，开启bin-log**

**read-only =0  #主机，读写都可以**<br>
**binlog-do-db  =test   #需要备份数据，多个写多行**<br>
**binlog-ignore-db =mysql #不需要备份的数据库，多个写多行**<br>

可以通过`mysql>show variables like 'log_%';` 验证二进制日志是否已经启动。

 


### 3、现在可以停止主数据的的更新操作，并生成主数据库的备份，

我们可以通过`mysqldump`到处数据到从数据库，当然了，你也可以直接用`cp`命令将数据文件复制到从数据库去，注意在导出数据之前先对主数据库进行READ LOCK，以保证数据的一致性
```mysql
mysql> flush tables with read lock;

Query OK, 0 rows affected (0.19 sec)
```
然后mysqldump导出数据：
```mysql
mysqldump -h127.0.0.1 -p3306 -uroot -p test > /home/chenyz/test.sql
```


### 4、得到主服务器当前二进制日志名和偏移量，这个操作的目的是为了在从数据库启动后，从这个点开始进行数据的恢复。
```mysql
mysql> show master status\G;

*************************** 1. row ***************************

File: mysql-bin.000003

Position: 243

Binlog_Do_DB:

Binlog_Ignore_DB:

1 row in set (0.00 sec)
```


最好在主数据库备份完毕，恢复写操作。
```mysql
mysql> unlock tables;

Query OK, 0 rows affected (0.28 sec)
```


### 5、将刚才主数据备份的test.sql复制到从数据库，进行导入。

### 6、修改从数据库的my.cnf，增加server-id参数，指定复制使用的用户，主数据库服务器的ip，端口以及开始执行复制日志的文件和位置。



#### 6.1配置的方式

我的5.6中用不了，启动报错。（看了好几篇文章都说不能用了）

打开从机`B`的`my.cnf`，输入
```mysql


server-id=2
log_bin=/var/log/mysql/mysql-bin.log
master-host = 192.168.1.182
master-user = slave
master-pass = 123456
master-port =3306
master-connect-retry = 60
replicate-do-db = test 

```






#### 6.2命令的方式

登录mysql后执行
```mysql
CHANGE MASTER TO
MASTER_HOST='192.168.1.182',
MASTER_USER='slave',
MASTER_PASSWORD='123456',
## 不写就是默认
master_log_file='mysql-bin.000002',
## 不写默认
master_log_pos=120;

```

### 7、在从服务器上,启动slave进程
```mysql
mysql> start slave;
```


### 8、在从服务器进行show salve status验证
```mysql
mysql> SHOW SLAVE STATUS\G

*************************** 1. row ***************************

Slave_IO_State: Waiting for master to send event

Master_Host: localhost

Master_User: root

Master_Port: 3306

Connect_Retry: 3

Master_Log_File: mysql-bin.003

Read_Master_Log_Pos: 79

Relay_Log_File: gbichot-relay-bin.003

Relay_Log_Pos: 548

Relay_Master_Log_File: mysql-bin .003

Slave_IO_Running: Yes

Slave_SQL_Running: Yes
```
### 9、验证

在主机A中，`mysql>show master status\G;`
在从机B中，`mysql>show slave status\G;`
能看到大致这些内容
```mysql
File: mysql-bin.000001
Position: 1374
Binlog_Do_DB: test
Binlog_Ignore_DB: mysql
```
可以在主机A中，做一些INSERT, UPDATE, DELETE 操作，看看主机B中，是否已经被修改。


### 10.总结

mysql主从复制的关键点是binlog,从服务器负责同步主服务的binlog实现了主从复制。需要了解binlog的请看我上一篇文章[MySQL数据恢复－－binlog](http://www.luckyhe.com/post/70.html)。下一篇我会教大家怎么去实现读写分离。