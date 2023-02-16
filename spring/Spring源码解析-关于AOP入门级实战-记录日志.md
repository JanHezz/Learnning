《Spring源码解析-SpringAOP入门级实战》首发[牧码人博客](http://www.luckyhe.com/post/89.html)转发请加此提示

## Spring源码解析-SpringAOP入门级实战

####  题外话

大家好我是鸽子王-**牧码人**，鸽了大半年，一直沉迷于加班的快乐之中。近半年其实挺迷茫，一直在加班，写一堆业务代码。感觉自己技术栈越走越窄了。没有很大的进步。当然写业务代码期间也碰到了不少棘手问题，也优化了不少问题。最近腾了点时间还是决定把它分享给大家。

#### 1.引入

Spring AOP一直是面试必考点之一，很多人尽管没有使用过，但是也被面试官吊打过不少了，所以对于`AOP`的一些概念的东西也就倒背如流了。这一点我面试人的过程中我也会经常去问，很多人对于`切点`,`通知`等专业词汇张口就来。一问`AOP`能用来做啥，大家跟约定好一样，就是记日志。我今天就给大家讲讲怎么使用`AOP`来实现记录日志的功能。


#### 2.应用场景

最近接了一个需求，大致就是上传文件到FTP服务器上。需要记录文件名，上传时间，上传状态、数量等。需求不是很难。我这边大概有10个定时任务是做这种类似的功能，因为上传的文件内容都是各不相同的。

#### 3.需求分析

对于这个需求无非就是一个记录日志的功能，但是量很多有10几个这种类似的功能。但是记录日志这个方法都是通用的。虽然每个上传文件处理逻辑是不一样的，但是记录日志这个日志是通用的。上传前要记录一下上传中，上传之后，把数据量，上传状态改为成功。

#### 4.实现方式

##### 4.1 传统方式

```java
 try {
     	           InterfaceLog interfaceLog = new InterfaceLog();
					interfaceLog.setTableCode("ANLYSIS_BUSINESS_DISCOUNT_DATA");
					interfaceLog.setTableName("分产品折扣月表");
					interfaceLog.setNum(0);
					interfaceLog.setIsSuccess("上传中");
					interfaceLog.setCreateTime(new Date());
                 	interfaceLogService.save(interfaceLog);
                    //保存文件数据
					int num =upload();
					interfaceLog.setNum(num);
					interfaceLog.setIsSuccess("成功");
					interfaceLogService.update(interfaceLog);
                } catch (Exception e) {
					interfaceLog.setIsSuccess("失败");
					interfaceLog.setFailReason(e.getMessage());
					interfaceLogService.update(interfaceLog);
                    e.printStackTrace();
                }
```

这是前同事的代码，这是另一个功能，解析文件记录日志的功能。这种代码，我看到了大概50个类左右。

##### 4.2AOP方式实现（注解方式实现）

定义注解

```java
/**
 * @CLASSNAME FileIssueLogEnum
 * @Description 文件下发日志注解
 * @Auther JanHezz
 * @BLOG www.luckyhe.com
 * @DATE 2021/9/6 19:02
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FileIssueLogAnnotation {

	/**
	 * 描述
	 *
	 * @return {String}
	 */
	String value();
}


```

创建`Aspect`容器

```java
/**
 * @CLASSNAME FileIssueLogEnum
 * @Description 文件下发日志切面
 * @Auther JanHezz
 * @BLOG www.luckyhe.com
 * @DATE 2021/9/6 19:02
 */
@Slf4j  
@Aspect //核心注解，定义切面容器
@EnableAsync//核心注解，支持异步
@AllArgsConstructor
@Component
public class SysFileIssueLogAspect {

    @Autowired
    SysFileIssueLogService logService;

	@SneakyThrows
	@Around("@annotation(sysFileIssueLog)")
	public Object around(ProceedingJoinPoint point,
                         FileIssueLogAnnotation sysFileIssueLog) {
        String strClassName = point.getTarget().getClass().getName();
        String strMethodName = point.getSignature().getName();
        log.debug("[类名]:{},[方法]:{}", strClassName, strMethodName);

        Object[] args = point.getArgs();

        SysFileIssueLog log = new SysFileIssueLog();

        for (Object arg : args) {
            if (arg instanceof AbstractFileIssueStParam) {
                AbstractFileIssueStParam queryStParam = (FileIssueStParam) arg;
                log.setFileName(queryStParam.getFileName());
                log.setPeriodDate(queryStParam.getPeriodLocalDate());
                log.setDecision(
                        queryStParam.getDecision()
                );
                log.setType(sysFileIssueLog.value());
            }
        }
        logService.saveFileLog(log);
        Object obj = null;
        try {
            obj = point.proceed();

            if (obj instanceof R) {
                R<Boolean> r = (R<Boolean>) obj;

                if (r.getData()) {
                    log.setIssueState(FileIssueLogEnum.SUCCESS.getType());
                } else {
                    log.setIssueState(FileIssueLogEnum.FAIL.getType());
                    log.setErrorLog(r.getMsg());
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
            log.setErrorLog(e.getMessage());
            log.setIssueState(FileIssueLogEnum.FAIL
                    .getType());
        }
        log.setUpdatedTime(DateUtil.toLocalDateTime(new Date()));
        logService.updateById(log);
        return obj;
    }

}

```


##### 4.3在需要使用的地方加上注解

```java
public interface RemoteFileService {

    /**
     * 生成xx文件
     */
    @FileIssueLogAnnotation(value = "xx文件")
    @PostMapping("/sthidata/genTravelCheckDataFile")
    R<Boolean> genTravelCheckDataFile(@RequestBody FileIssueStParam param,
                                      @RequestHeader(SecurityConstants.FROM) String from);


}

```

以上就是通过注解实现`AOP`记录日志功能，只要是加上了`FileIssueLogAnnotation`这注解的方法，便会被切面容器管理到。


#### AOP机制浅谈

`AOP`也称面向切面，面向切面编程最大的方便就是减少代码冗余。其实面向切面的原理就是`JDK`中的动态代理。代理一个方法，处理这个方法前做一些事，处理完后在做一些事。只不过`Spring`团队利用这个机制，把代码进行了封装。更加方便大家使用。

- **jdk动态代理**

  定义处理逻辑

```
  /**
 * @CLASSNAME IHello
 * @Description 代理类
 * @Auther JanHezz
 * @BLOG www.luckyhe.com
 * @DATE 2021/9/6 19:02
 */
public interface IHello{
    void sayHello();
}

public class HelloImpl implements IHello {
    @Override
    public void sayHello() {
        System.out.println("Hello JAVA");
    }
}
```

    定义代理类

```
 
  /**
 * @CLASSNAME MyInvocationHandler
 * @Description 代理类
 * @Auther JanHezz
 * @BLOG www.luckyhe.com
 * @DATE 2021/9/6 19:02
 */
public class MyInvocationHandler implements InvocationHandler {
 
    /** 目标对象 */
    private Object target;
 
    public MyInvocationHandler(Object target){
        this.target = target;
    }
 
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("------代理前-------------");
        // 执行相应的目标方法
        Object rs = method.invoke(target,args);
        System.out.println("------代理后-------------");
        return rs;
    }
}
```

    实现测试

```
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Proxy;
 
/**
 * 使用JDK动态代理的五大步骤:
 * 1.通过实现InvocationHandler接口来自定义自己的InvocationHandler;
 * 2.通过Proxy.getProxyClass获得动态代理类
 * 3.通过反射机制获得代理类的构造方法，方法签名为getConstructor(InvocationHandler.class)
 * 4.通过构造函数获得代理对象并将自定义的InvocationHandler实例对象传为参数传入
 * 5.通过代理对象调用目标方法
 */
 
 /**
 * @CLASSNAME MyProxyTest
 * @Description 代理测试
 * @Auther JanHezz
 * @BLOG www.luckyhe.com
 * @DATE 2021/9/6 19:02
 */
public class MyProxyTest {
    public static void main(String[] args)
            throws NoSuchMethodException, IllegalAccessException, InstantiationException, InvocationTargetException {
        // =========================第一种==========================
        // 1、生成$Proxy0的class文件
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        // 2、获取动态代理类
        Class proxyClazz = Proxy.getProxyClass(IHello.class.getClassLoader(),IHello.class);
        // 3、获得代理类的构造函数，并传入参数类型InvocationHandler.class
        Constructor constructor = proxyClazz.getConstructor(InvocationHandler.class);
        // 4、通过构造函数来创建动态代理对象，将自定义的InvocationHandler实例传入
        IHello iHello1 = (IHello) constructor.newInstance(new MyInvocationHandler(new HelloImpl()));
        // 5、通过代理对象调用目标方法
        iHello1.sayHello();
 
        // ==========================第二种=============================
        /**
         * Proxy类中还有个将2~4步骤封装好的简便方法来创建动态代理对象，
         *其方法签名为：newProxyInstance(ClassLoader loader,Class<?>[] instance, InvocationHandler h)
         */
        IHello  iHello2 = (IHello) Proxy.newProxyInstance(IHello.class.getClassLoader(), // 加载接口的类加载器
                new Class[]{IHello.class}, // 一组接口
                new MyInvocationHandler(new HelloImpl())); // 自定义的InvocationHandler
        iHello2.sayHello();
    }
}

//执行结果
------代理前-------------
Hello JAVA
------代理后-------------
```

所以我认为的`AOP`是一个处理思想。最关键的点还是了解他的思想。了解思想后，你便可以使用它来解决一些问题。

- **`SpringAop`的一些专业术语**

  - 通知（Advice）: AOP 框架中的增强处理。通知描述了切面何时执行以及如何执行增强处理。
  - 连接点（join point）: 连接点表示应用执行过程中能够插入切面的一个点，这个点可以是方法的调用、异常的抛出。在 Spring AOP 中，连接点总是方法的调用。
  - 切点（PointCut）: 可以插入增强处理的连接点。
  - 切面（Aspect）: 切面是通知和切点的结合。
  - 引入（Introduction）：引入允许我们向现有的类添加新的方法或者属性。
  - 织入（Weaving）: 将增强处理添加到目标对象中，并创建一个被增强的对象，这个过程就是织入。

  概念性的东西了解一下就行，不同的时候有不同的理解。

- **`SpringAop`的一些常用注解**

  - @Aspect : 定义切面类。（类级）
  - @Pointcut(): 定义一个切点供通知使用。（方法级）
  - @Before: 前置通知注解。（方法级）
  - @After: 后置通知注解。（方法级）
  - @AfterReturning:切点类处理完毕后，返回值时调用。（方法级）
  - @Around：环绕通知。这个是Before跟After结合。（方法级）

## 总结

`SpringAOP`是学习`Spring`的核心。`AOP`这个概念，对于我们日常开发也是非常重要的。会写一个`AOP`的例子并不难，重要的往往是理解这种代码思想。并且在日常的业务代码中灵活使用它。接下来，有时间的话，我会跟着大家一起看一下`Spring`是怎么使用`JDk`的动态代理，实现`AOP`的。
