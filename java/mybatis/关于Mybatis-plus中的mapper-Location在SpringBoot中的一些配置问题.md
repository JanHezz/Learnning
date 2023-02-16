---


  title: 关于Mybatis-plus中的mapper-Location在SpringBoot中的一些配置问题
  date: {{ date }}   
  categories: ['Mybatis']
  tags: ['Mybatis','SpringBoot']       
  comments: true    
  img:             

---

本篇文章主要介绍关于我在`SpringBoot`中使用`MyBatis-Plus`是如何解决`Invalid bound statement (not found)`这个异常的。我先抛一些我在这个途中遇到的一些问题，看看各位了解不了解。

1. 当`Mybatis`的`xml`文件不在`resouce`下时该如何配置。
2. 如何去指定`mapper-Location`的配置。
3. `classpath*`跟`classpath`的区别是啥
4. Invalid bound statement (not found)出现的原因是什么

以上就是我遇到这个问题之后总结的三个问题。

## 缘由

作者来了一下新公司，这边的框架看的我很闷，特别是关于mybatis的一些用法。这边的sql都是用注解写在Mapper文件上。例如

```java
 @Select("SELECT id,status, " +
            " actual_usage_id usage_id," +
            " location_id ," +
            " group_id ," +
            " breakdown_Level_id, " +
            " receive_persion_id " +
            "FROM " +
            " t_repair_workorder  " +
            " ${ew.customSqlSegment} ")
    List<IndexDutyPageVo> dutyFaultPage(@Param(Constants.WRAPPER) Wrapper<?> wrappser);
```

整个项目全是这种写法，我一开始以为是规范。后面问了一个老员工才知道。说以前这个项目是写在xml的，但是后面改了一下架构之后xml的配置就扫描不到了。嗯嗯嗯...

这边的项目结构，xml文件不是放在`resouce`下，并且具体的业务包是跟`maven`引入进去的（这个就是我前文提到的架构改变了）。注意这两个是重点。我猜测他们不会配置的点应该就是这个原因了吧。

## 解决问题

竟然知道了问题就开始解决问题。

###### 当`Mybatis`的`xml`文件不在`resouce`下时该如何配置。

`Mybatis`中如果xml文件不在`resource`目录下的话，默认打包是会被忽略的，所以需要在pom文件中加一段配置。

```xml
 <build>
        <resources>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.xml</include>
            </includes>
            <filtering>false</filtering>
        </resource>
         </resources>	
    </build>
```

改完这个后重新`build`一下，注意去查看下target文件夹下是否`xml`文件。

######  如何去指定`mapper-Location`的配置

```
mybatis-plus:
  mapper-locations: classpath*:top/**/*.xml
## top是我具体文件夹可以不要,  **的意思代表一个或者多个目录
```

###### `classpath*`跟`classpath`的区别是啥

这个问题是重点要考的记一下，带`*`的话会扫描jar包下面的文件，不带`*`只会扫描当前项目。

######  Invalid bound statement (not found)出现的原因是什么?如何排查这个问题

这个报错的出现，就是代表你的`mapper`文件跟`xml`映射不到。如果你确保你的框架没有问题下，其它的代码都能映射得到的情况，那么你就要注意了，首先你的`xml`文件的名字跟`Mapper`文件是不是一致的，方法名跟`xml`的`id`是不是一致的。如果你这两个都对了，再去查你的`mapper-locations`的配置，这里没问题，再去查编译包。看看`xml`编译到了不。如果这些都没问题。那人跟代码只要一个能跑就行。

## 废话时间

其实使用`xml`跟使用注解的形式都能完成需求，没多大的区别。但是使用`xml`的可读性，以及易维护性。个人觉得比注解方式强太多了。此次问题的出现，关键在于架构的改变，架构者一想把`xml`从`resource`移除，二又想把业务模块热插拔。这个想法是好的。但是做事做一半真的不太可取。



本篇首发于[牧码人博客](http://www.luckyhe.com/post/95.html)转载请加上此标示。

