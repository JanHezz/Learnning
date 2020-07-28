《Spring源码解析-入门Spring的IOC思想》首发[橙寂博客](http://www.luckyhe.com/post/79.html)转发请加此提示

## Spring源码解析-入门Spring的IOC思想

#### 1.引入

大家在面试的时候，应该都会碰到这么一个问题。请浅谈一下`Spring IOC`(控制反转)思想?或者是解释下什么是`DI`(依赖注入)?。本篇会从`Spring官方文档`的角度,结合自己的工作经验
，给大家讲一下自己对`Spring`的一些理解。


#### 2.概述
## Spring源码解析-元数据的读取（XML）

`Spring Core`是`Spring`的核心。而`Spring`的核心讲的就是一个`IOC`思想。`Spring`的`IOC`容器管理着是一个或者多个`Bean`。在一个`Spring`应用启动,`Spring`会根据我们提供的元数据。
做一个`自省`的过程。当我们真正用一些`Bean`时。这时我们可以通过注解（@Autowired）的方式或者是主动调用`ApplicationContext`的`getBean()`方法根据标识符（Id或者带包名的类名）从容器中取得我们想要的
`Bean`对象。这就是`IOC`也叫做`DI`。

当然在容器中每个`Bean`都是有个唯一的`Id`的，如果我们不显示的提供,`Spring`会根据Java的规范默认按照驼峰命名法命名（首字母小写）。(例如:postService,postDao)

#### 3.配置元数据

如果想`Spring`去管理`Bean`那么你就得告诉`Spring`你需要管理的`Bean`。这个过程叫做配置元数据。配置元数据有两中方式。
每个`Bean`在`Spring`中被定义成了`BeanDefinition`这个对象里面保存我们这个`Bean`的一些基本属性：

1. 带包名的类名
2. 其他类的引用
3. 管理`bean`的连接池的的大小,或者连接数。
4. 其他一些行为。比如`Bean`的作用域。

##### 3.1配置`Bean`的两种方式

- 注解

在我们的日常工作中,我们会使用

```java
@Commpont

@Service

@Configuration
//代表这是一个配置类
public class QuartzConfig {

    /**
     * 往容器中初始注入scheduler
     * @return
     * @throws SchedulerException
     */
    @Bean
    public Scheduler scheduler() throws SchedulerException{
        SchedulerFactory schedulerFactoryBean = new StdSchedulerFactory();
        return schedulerFactoryBean.getScheduler();
    }
}

```
以上注解都是告诉`Spring`我需要将带这些注解的`Bean`教给`Spring`容器管理。除了这个具体每个注解带的含义是不一样的。有想要了解的自己去了解下。


- xml配置

dao.xml
```java

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="accountDao"
        class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for data access objects go here -->

</beans>

```
以上是一个`Dao`的配置文件。接下来程序去加载这个文件就好了。

```java

//这里可以加载多个文件,分隔。
ApplicationContext context = new ClassPathXmlApplicationContext("daos.xml");

// 通过Id获得容器中的对象
JpaAccountDao dao = context.getBean("accountDao");

// 使用对象中的方法
List<String> userList = dao.getUsernameList();

```


#### 依赖注入（DI）

元数据配置好了，但是通常在我们日常的使用中我们的`Bean`肯定是依赖了其他`Bean`的。
比如一个`Controller`中需要注入`Service`等等。
所以在配置的时候我们就需要把需要依赖的类(或属性)注入进去。

- xml配置

构造方式的形式注入`Bean`
```java
package x.y;

//构造方式的形式注入
public class ThingOne {

    public ThingOne(ThingTwo thingTwo, ThingThree thingThree) {
        // ...
    }
}

<beans>
    <bean id="beanOne" class="x.y.ThingOne">
        <constructor-arg ref="beanTwo"/>
        <constructor-arg ref="beanThree"/>
    </bean>

    <bean id="beanTwo" class="x.y.ThingTwo"/>

    <bean id="beanThree" class="x.y.ThingThree"/>
</beans>

```
构造方式的形式注入属性

```java
package examples;

public class ExampleBean {

    // Number of years to calculate the Ultimate Answer
    private int years;

    // The Answer to Life, the Universe, and Everything
    private String ultimateAnswer;

    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}

//第一种方式
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg type="java.lang.String" value="42"/>
</bean>

//第二种方式

<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg index="0" value="7500000"/>
    <constructor-arg index="1" value="42"/>
</bean>


```

`Setter`方式注入

```java

public class ExampleBean {

    private AnotherBean beanOne;

    private YetAnotherBean beanTwo;

    private int i;

    public void setBeanOne(AnotherBean beanOne) {
        this.beanOne = beanOne;
    }

    public void setBeanTwo(YetAnotherBean beanTwo) {
        this.beanTwo = beanTwo;
    }

    public void setIntegerProperty(int i) {
        this.i = i;
    }
}

<bean id="exampleBean" class="examples.ExampleBean">
    <!-- setter injection using the nested ref element -->
    <property name="beanOne">
        <ref bean="anotherExampleBean"/>
    </property>

    <!-- setter injection using the neater ref attribute -->
    <property name="beanTwo" ref="yetAnotherBean"/>
    <property name="integerProperty" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>


```

- 注解方式注入


这种方式应该是目前用的最多的一种形式。特别是在`SpringBoot`项目中,抛弃`xml`配置的方式,全局使用了注解的方式。
（题外话:SpringBoot中的自动装配机制其实跟`Spring`的机制是相通的。Springboot内置了一些自动装配的配置类。所以只需要改改配置文件。就能自动注入我们需要类）

当你想要在你的`Bean`中使用`@Autoried`  `@Resource(name = "bean的名字")`注入您需要的`Bean`。
你一定要确定`Bean`存在于容器中。（也就是说一定要先提供的元数据）。

下面讲几个在日常开发中经常会用到的例子

```java

@Controller
public class BaseController
{
	//系统用户
	@Autowired
	public SysUserService sysUserService;


  //这个name一定是你配置的名字，如果你没配那么默认是首字母小写
  @Resource(name = "sysPermissionService")
  public SysPermissionService sysPermissionService;
  }

```

上面两个例子是在开发经常使用的。


## 补充

- `自动装配机制`

上面讲了怎么往`Bean`对象中注入属性以及`对象`。并且演示了怎么往`Bean`中注入另一个`Bean`。那么`SpringIOC`容器是怎么做到两个`Bean`之间互相协作的呢？这完全取决于`IOC`的自动装配机制。自动装配可以通过`name`,'type','构造函数'等模式来实现这个功能。
默认是通过`Type`来实现的。也是说比如你想要个`String`类型的对象它就会给你注入一个`String`类型的。

- `Bean`的作用域

每个`Bean`在容器中是有自己的作用域的。一共存在`singleton`(单例),`prototype`(原型),`request`，'session','application','websocket'。
这里主要讲一下'singleton'和'prototype'。我在面试中被问到过这两者的区别,我是这么回答的:singleton是无状态的,prototype是有状态的。然后我举了个例子,比如有个`orderService`购物车服务,一个客户是不是有一个购物车,而且购物车是不能被共享的。也就是每个人都要创建一个购物车。这就是`prototype`原型模式。每次获取`Bean`都会有一个全新的`Bean`。所以说`prototype`有状态.
而`singleton`容器中只有一个实例。使用`singleton`定义的实例在同一个线程中A对象跟B对象都依赖的是同一个实例。`IOC`中的对象默认是`singleto`作用域。
有想要了解其他的参考[spring官方文档Bean的作用域](https://docs.spring.io/spring/docs/5.2.2.RELEASE/spring-framework-reference/core.html#beans-factory-scopes)

- 容器的扩展

`IOC`内置了一些接口。方便我们在`SpringIOC`外部操作`Bean`对容器进行扩展。比如`ApplicationContextAware`接口。`BeanPostProcessor`接口,`BeanFactoryPostProcessor`接口都内置了回调方法。
这里给大家讲一个实用的工具类就是,获取`ApplicationContext`这个容器对象这样我们就能操作`getBean方法`。
下面我们会通过继承`ApplicationContextAware `来实现。

```java
public class SpringContextHolder implements ApplicationContextAware, DisposableBean {

    private static ApplicationContext applicationContext = null;


    /**
     * 取得存储在静态变量中的ApplicationContext.
     */
    public static ApplicationContext getApplicationContext() {
        assertContextInjected();
        return applicationContext;
    }

    /**
     * 从静态变量applicationContext中取得Bean, 自动转型为所赋值对象的类型.
     */
    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) {
        assertContextInjected();
        return (T) applicationContext.getBean(name);
    }

    /**
     * 从静态变量applicationContext中取得Bean, 自动转型为所赋值对象的类型.
     */
    public static <T> T getBean(Class<T> requiredType) {
        assertContextInjected();
        return applicationContext.getBean(requiredType);
    }

    /**
     * 清除SpringContextHolder中的ApplicationContext为Null.
     */
    public static void clearHolder() {
        applicationContext = null;
    }

    /**
     * 实现ApplicationContextAware接口, 注入Context到静态变量中.
     */
    @Override
    public void setApplicationContext(ApplicationContext appContext) {
        applicationContext = appContext;
    }

    /**
     * 实现DisposableBean接口, 在Context关闭时清理静态变量.
     */
    @Override
    public void destroy() throws Exception {
        SpringContextHolder.clearHolder();
    }

    /**
     * 检查ApplicationContext不为空.
     */
    private static void assertContextInjected() {
        Validate.validState(applicationContext != null, "applicaitonContext属性未注入, 请在applicationContext.xml中定义SpringContextHolder.");
    }
}

```
想要了解其它两个的使用参考[spring官方文档容器扩展](https://docs.spring.io/spring/docs/5.2.2.RELEASE/spring-framework-reference/core.html#beans-factory-extension)

## 总结

`SpringIOC`是学习`Spring`的核心。下面笔者会带着大家一起去学习`Spring Core`的源码。敬请期待。
