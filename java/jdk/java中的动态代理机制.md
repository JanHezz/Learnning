《java中的动态代理机制》首发[橙寂博客](http://www.luckyhe.com/post/75.html)转发请加此提示


这几天在看`Spring的源码解析`,在看到`aop`的时候。我才知道`aop`思想关键就是jdk中的动态代理机制。而关于动态代理机制,关键就在于`java.lang.reflect`报下的`proxy`类跟`InvocationHandler`接口。

# java中的动态代理机制

`Proxy `类

`Proxy`是用来创建一个代理对象的类。里面有很多内置方法。但我们常用的一个`api`是`newProxyInstance`方法.

`newProxyInstance`是用来创建一个代理对象的

- loader：一个classloader对象，定义了由哪个classloader对象对生成的代理类进行加载
- interfaces：一个interface对象数组，表示我们将要给我们的代理对象提供一组什么样的接口，如果我们提供了这样一个接口对象数组，那么也就是声明了代理类实现了这些接口，代理类就可以调用接口中声明的所有方法。(通常是被代理对象的接口)
- h：一个InvocationHandler对象，表示的是当动态代理对象调用方法的时候会关联到哪一个InvocationHandler对象上，并最终由其调用。

`InvocationHandler`接口

`InvocationHandler`接口是proxy代理实例的调用处理程序实现的一个接口，每一个proxy代理实例都有一个关联的调用处理程序；在代理实例调用方法时，方法调用被编码分派到调用处理程序的invoke方法。

也就是说`InvocationHandler`定义了一个`invoke`接口。当代理对象执行每一个`api`时都会调用`invoke`方法。

invoke方法中有三个参数
Object proxy, Method method, Object[] args

- proxy：代理对象实例
- method：被代理对象调用的api。
- args：被代理对象的参数。


## 实战

对上面的`api`简单进行了介绍后。现在我们做个例子。

定义一个ProxyTest接口。
```java
/**
 * @CLASSNAME ProxyTest
 * @Description
 * @Auther Jan  橙寂
 * @DATE 2020/2/19 0019 10:08
 */
public interface ProxyTest {


    void test();
}


```

具体实现类

```java
/**
 * @CLASSNAME ProxyTestImpl
 * @Description
 * @Auther Jan  橙寂
 * @DATE 2020/2/19 0019 10:09
 */

public class ProxyTestImpl implements ProxyTest {
    @Override
    public void test() {
        System.out.printf("test");
    }
}


```

实现一个`InvocationHandler`接口。并重写`invoke`方法

```java

package main.jdk;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/**
 * @CLASSNAME ProxyInvocationHandler
 * @Description
 * @Auther Jan  橙寂
 * @DATE 2020/2/19 0019 10:14
 */

public class ProxyInvocationHandler implements InvocationHandler {

    Object object;

    public ProxyInvocationHandler(Object object) {
        this.object = object;
    }

    @Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable {

        System.out.printf("方法执行前");

        Object invoke = method.invoke(object, objects);

        System.out.printf("方法执行后");
         return o;
    }
}


···

测试类

```java

public static void main(String[] args) {

       //被代理对象
       ProxyTestImpl  proxyTest=new ProxyTestImpl();


       //代理对象执行者
       ProxyInvocationHandler proxyInvocationHandler = new ProxyInvocationHandler(proxyTest);

       //创建一个代理对象
       ProxyTest o = (ProxyTest) Proxy.newProxyInstance(proxyTest.getClass().getClassLoader(), proxyTest.getClass().getInterfaces(), proxyInvocationHandler);

       //调用具体api。验证proxyTest对象已经被代理了
       o.test();


   }

```
输出结果:方法执行前test方法执行后。

上面的例子,也就是java的动态代理机制,也就是`aop`思想原理所在。
