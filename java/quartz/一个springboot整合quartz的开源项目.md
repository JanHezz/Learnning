---
  title: 一个springboot整合quartz的开源项目
  date: {{ date }}   
  categories: ['后端']
  tags: ['quartz','spring','springboot']       
  comments: true    
  img:             
---


## 什么是quartz
Quartz 是一个完全由 Java 编写的开源作业调度框架，为在 Java 应用程序中进行作业调度提供了简单却强大的机制。


#### quartz初体验
1. 首先使用quartz需要使用他jar包。
```
<!--quartz依赖-->
       <dependency>
           <groupId>org.quartz-scheduler</groupId>
           <artifactId>quartz</artifactId>
           <version>2.2.1</version>
       </dependency>
```
2. 创建Scheduler 对象

这个对象由**SchedulerFactory**去创建
```
SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();

  Scheduler sched = schedFact.getScheduler();
```
3. 创建Job对象

**execute**是任务调度具体执行的方法

```
public class HelloJob implements Job {

   public HelloJob() {
   }

   public void execute(JobExecutionContext context)
     throws JobExecutionException
   {
     System.err.println("Hello!  HelloJob is executing.");
   }
 }
```
4. 创建触发器

```
//40s执行一次
rigger trigger = newTrigger()
      .withIdentity("myTrigger", "group1")
      .startNow()
      .withSchedule(simpleSchedule()
          .withIntervalInSeconds(40)
          .repeatForever())            
      .build();

```

5. 测试

```
SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();

  Scheduler sched = schedFact.getScheduler();

  sched.start();

  // 指定工作的类
  JobDetail job = newJob(HelloJob.class)
      .withIdentity("myJob", "group1")
      .build();

  // 指定触发器40s执行一次
  Trigger trigger = newTrigger()
      .withIdentity("myTrigger", "group1")
      .startNow()
      .withSchedule(simpleSchedule()
          .withIntervalInSeconds(40)
          .repeatForever())
      .build();

  // 告诉定时器任务是什么触发器是什么
  sched.scheduleJob(job, trigger);

  //开始
  sched.start();
```

## quratz的主要组件
我们需要明白 Quartz 的几个核心概念，这样理解起 Quartz 的原理就会变得简单了。

1. Job 表示一个工作，要执行的具体内容。此接口中只有一个方法，如下：
> void execute(JobExecutionContext context)
2. JobDetail 表示一个具体的可执行的调度程序，Job 是这个可执行程调度程序所要执行的内容，另外 JobDetail 还包含了这个任务调度的方案和策略。

3. Trigger 代表一个调度参数的配置，什么时候去调。

4. Scheduler 代表一个调度容器，一个调度容器中可以注册多个 JobDetail 和 Trigger。当 Trigger 与 JobDetail 组合，就可以被 Scheduler 容器调度了。

## springquartz做了什么？
quartz的使用很简单，但是在我们平时工作做，我们对于定时调度我们需要的是能动态的配置他。所以我把它整合了一下使它拥有了下面的功能。（在这个demo中我仅仅是整合了quartz非常的简洁明白）
1. 动态的指定任务（能具体到使用的类和方法）
>不用一个任务创建一个job类了。我可以指定到任何一个类和方法。
2. 支持并发与不并发
>通过配置的方式指定并发执行以及不并发执行。

3. 日志记录
>把定时调度的日志记录了下来。
