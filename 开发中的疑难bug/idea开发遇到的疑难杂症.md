#### 1.idea中明明引用了jar包编译时一直提示文件找不到。

> mvn idea:module
>
> 使用这个命令会重新引用maven下面的子包。idea的iml文件会重建

2.证书生成

> ```
> ## 根据cer文件生成访问key
> keytool -import -alias serverkey -file jxgsgl.cer  -keystore D:/tclient.ks
> ```

3.启动类过长

> ```
> 报错：Command line is too long. Shorten command line for MaxKeyApplication or also for Spring Boot default configuration?
> 解决方案
> .idea下的workspace.xml文件的PropertiesComponent标签下加一行
> <property name="dynamic.classpathvalue" value="true"/>
> ```
