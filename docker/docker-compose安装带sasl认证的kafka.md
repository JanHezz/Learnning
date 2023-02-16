---

  title: docker-compose安装带SASL认证的kafka
  date: {{ date }}   
  categories: ['Docker']
  tags: ['Docker','Docker-compose']       
  comments: true    
  img:             
---

本篇文章会介绍使用使用`docker-compose`安装带SASL认证的Kafka消息队列。为啥会有这篇文章主要是网上一些文章太折磨人了，都是互抄的而且都是错的。我配合`SpringBoot`调试搞了我一天。为啥要搞SASL认证也主要是为了安全考虑。如果不加认证，`Kafka`应用就是裸露在外面的，会有安全风险。阅读此文前，需要对`docker-compose`有一个基础认知，本文并不属于小白文。

本篇首发于[牧码人博客](http://www.luckyhe.com/post/95.html)转载请加上此标示。

## 准备工作

1. **docker-compose.yml**

   ```dockerfile
   # 版本根据你的docker版本来的，目前主流应该都是3.几的版本
   version: '3.8'
   services:
     zookeeper:
       image: wurstmeister/zookeeper
       volumes:
          - /data/zookeeper/data:/data
          - /home/docker-compose/kafka/config:/opt/zookeeper-3.4.13/conf/
          - /home/docker-compose/kafka/config:/opt/zookeeper-3.4.13/secrets/ 
       container_name: zookeeper
       environment:
         ZOOKEEPER_CLIENT_PORT: 2181
         ZOOKEEPER_TICK_TIME: 2000
         SERVER_JVMFLAGS: -Djava.security.auth.login.config=/opt/zookeeper-3.4.13/secrets/server_jaas.conf
       ports:
         - 12181:2181
       restart: always
     kafka_node1:
       image: wurstmeister/kafka
       container_name: kafka_node1
       depends_on:
         - zookeeper
       ports: 
         - 9092:9092
       volumes:
         - /home/docker-compose/kafka/data:/kafka
         - /home/docker-compose/kafka/config:/opt/kafka/secrets/
       environment:
         KAFKA_BROKER_ID: 0
         KAFKA_ADVERTISED_LISTENERS: SASL_PLAINTEXT://127.0.0.1:9092
         KAFKA_ADVERTISED_PORT: 9092 
         KAFKA_LISTENERS: SASL_PLAINTEXT://0.0.0.0:9092
         KAFKA_SECURITY_INTER_BROKER_PROTOCOL: SASL_PLAINTEXT
         KAFKA_PORT: 9092 
         KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL: PLAIN
         KAFKA_SASL_ENABLED_MECHANISMS: PLAIN
         KAFKA_AUTHORIZER_CLASS_NAME: kafka.security.auth.SimpleAclAuthorizer
         KAFKA_SUPER_USERS: User:admin
         KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND: "true" #设置为true，ACL机制为黑名单机制，只有黑名单中的用户无法访问，默认为false，ACL机制为白名单机制，只有白名单中的用户可以访问
         KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
         KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
         KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
         KAFKA_HEAP_OPTS: "-Xmx512M -Xms16M"
         KAFKA_OPTS: -Djava.security.auth.login.config=/opt/kafka/secrets/server_jaas.conf
       restart: always
    ##  kafdrop 监控kafka的Ui工具 
     kafdrop:
       image: obsidiandynamics/kafdrop
       restart: always
       ports:
          - "19001:9000"
       environment:
          KAFKA_BROKERCONNECT: "kafka_node1:9092"
       ## 如kafka开启了sasl认证后以下 sasl认证链接是必要的，下面的事经过base64加密后的结果
          KAFKA_PROPERTIES: c2FzbC5tZWNoYW5pc206IFBMQUlOCiAgICAgIHNlY3VyaXR5LnByb3RvY29sOiBTQVNMX1BMQUlOVEVYVAogICAgICBzYXNsLmphYXMuY29uZmlnOiBvcmcuYXBhY2hlLmthZmthLmNvbW1vbi5zZWN1cml0eS5zY3JhbS5TY3JhbUxvZ2luTW9kdWxlIHJlcXVpcmVkIHVzZXJuYW1lPSdhZG1pbicgcGFzc3dvcmQ9J2pkeXgjcXdlMTInOw==
       depends_on:
         - zookeeper
         - kafka_node1
       cpus: '1'
       mem_limit: 1024m
       container_name: kafdrop
       restart: always
   ```

2. `server_jaas.conf`

   ```
   Client {
       org.apache.zookeeper.server.auth.DigestLoginModule required
       username="admin"
       password="123456";
   };
   
   
   Server {
       org.apache.zookeeper.server.auth.DigestLoginModule required
       username="admin"
       password="123456"
       ## user_用户名="密码" 这种格式是用来配置账号跟密码的
       user_super="123456"
       user_admin="123456";
   };
   
   KafkaServer {
       org.apache.kafka.common.security.plain.PlainLoginModule required
       username="admin"
       password="123456"  
     ## user_用户名="密码"  这种格式是用来配置账号跟密码的
       user_admin="123456";
   };
   
   KafkaClient {
       org.apache.kafka.common.security.plain.PlainLoginModule required
       username="admin"
       password="123456";
   };
   ```

    以下三个文件为`zookeeper`的配置文件，除`zoo.cfg`（核心）文件做了改变，其余都是使用docker cp 命令 从zookeeper容器copy出来的。

3. `zoo.cfg`

      ```
      # The number of milliseconds of each tick
      tickTime=2000
      # The number of ticks that the initial 
      # synchronization phase can take
      initLimit=10
      # The number of ticks that can pass between 
      # sending a request and getting an acknowledgement
      syncLimit=5
      # the directory where the snapshot is stored.
      # do not use /tmp for storage, /tmp here is just 
      # example sakes.
      dataDir=/opt/zookeeper-3.4.13/data
      # the port at which the clients will connect
      clientPort=2181
      # the maximum number of client connections.
      # increase this if you need to handle more clients
      #maxClientCnxns=60
      #
      # Be sure to read the maintenance section of the 
      # administrator guide before turning on autopurge.
      #
      # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
      #
      # The number of snapshots to retain in dataDir
      autopurge.snapRetainCount=3
      # Purge task interval in hours
      # Set to "0" to disable auto purge feature
      autopurge.purgeInterval=1
      
      ## 开启SASl关键配置
      authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
      
      requireClientAuthScheme=sasl
      
      jaasLoginRenew=3600000
      zookeeper.sasl.client=true
      ```

4. `configuration.xsl`

         ```
         <?xml version="1.0"?>
         <xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform" version="1.0">
         <xsl:output method="html"/>
         <xsl:template match="configuration">
         <html>
         <body>
         <table border="1">
         <tr>
          <td>name</td>
          <td>value</td>
          <td>description</td>
         </tr>
         <xsl:for-each select="property">
         <tr>
           <td><a name="{name}"><xsl:value-of select="name"/></a></td>
           <td><xsl:value-of select="value"/></td>
           <td><xsl:value-of select="description"/></td>
         </tr>
         </xsl:for-each>
         </table>
         </body>
         </html>
         </xsl:template>
         </xsl:stylesheet>
         
         ```

5. `log4j.properties`

            ```
            # Define some default values that can be overridden by system properties
            zookeeper.root.logger=INFO, CONSOLE
            zookeeper.console.threshold=INFO
            zookeeper.log.dir=.
            zookeeper.log.file=zookeeper.log
            zookeeper.log.threshold=DEBUG
            zookeeper.tracelog.dir=.
            zookeeper.tracelog.file=zookeeper_trace.log
            
            #
            # ZooKeeper Logging Configuration
            #
            
            # Format is "<default threshold> (, <appender>)+
            
            # DEFAULT: console appender only
            log4j.rootLogger=${zookeeper.root.logger}
            
            # Example with rolling log file
            #log4j.rootLogger=DEBUG, CONSOLE, ROLLINGFILE
            
            # Example with rolling log file and tracing
            #log4j.rootLogger=TRACE, CONSOLE, ROLLINGFILE, TRACEFILE
            
            #
            # Log INFO level and above messages to the console
            #
            log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender
            log4j.appender.CONSOLE.Threshold=${zookeeper.console.threshold}
            log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout
            log4j.appender.CONSOLE.layout.ConversionPattern=%d{ISO8601} [myid:%X{myid}] - %-5p [%t:%C{1}@%L] - %m%n
            
            #
            # Add ROLLINGFILE to rootLogger to get log file output
            #    Log DEBUG level and above messages to a log file
            log4j.appender.ROLLINGFILE=org.apache.log4j.RollingFileAppender
            log4j.appender.ROLLINGFILE.Threshold=${zookeeper.log.threshold}
            log4j.appender.ROLLINGFILE.File=${zookeeper.log.dir}/${zookeeper.log.file}
            
            # Max log file size of 10MB
            log4j.appender.ROLLINGFILE.MaxFileSize=10MB
            # uncomment the next line to limit number of backup files
            log4j.appender.ROLLINGFILE.MaxBackupIndex=10
            
            log4j.appender.ROLLINGFILE.layout=org.apache.log4j.PatternLayout
            log4j.appender.ROLLINGFILE.layout.ConversionPattern=%d{ISO8601} [myid:%X{myid}] - %-5p [%t:%C{1}@%L] - %m%n
        
            #
            # Add TRACEFILE to rootLogger to get log file output
            #    Log DEBUG level and above messages to a log file
            log4j.appender.TRACEFILE=org.apache.log4j.FileAppender
            log4j.appender.TRACEFILE.Threshold=TRACE
            log4j.appender.TRACEFILE.File=${zookeeper.tracelog.dir}/${zookeeper.tracelog.file}
            
            log4j.appender.TRACEFILE.layout=org.apache.log4j.PatternLayout
            ### Notice we are including log4j's NDC here (%x)
            log4j.appender.TRACEFILE.layout.ConversionPattern=%d{ISO8601} [myid:%X{myid}] - %-5p [%t:%C{1}@%L][%x] - %m%n

## 开始搭建
1. 上传docker-compose文件
> mkdir  /home/docker-compose/kafka
> cd   /home/docker-compose/kafka 
2. 上传剩余配置文件到config文件夹下
> mkdir  /home/docker-compose/kafka/config
3. 构建
> docker-compose  up  -d
> docker ps 查询进程
> 所有进程正常启动的话   访问 http://ip:19001/  能正常显示管理界面就代码安装成功了