# 第二章：Quartz API、调度任务以及触发器
# 内容
## Quartz API：

下面是Quartz API中的关键接口：
- Scheduler：与调度器交互的主要API(实际上这个就是调度器)。
- Job：org.quartz.Job，希望由调度器执行的组件，是一个接口，也就是我们使用的时候被调度的任务需要实现此接口。
- JobDetail：org.quartz.JobDetail，调度任务详情，用于定义调度任务。
- Trigger：org.quartz.Trigger，也就是触发器，它是一个定义了给定调度任务将被执行的时间表的组件。
- JobBuilder：org.quartz.JobBuilder，用于定义或者构建JobDetail实例。
- TriggerBuilder：org.quartz.TriggerBuilder，用于定义或者构建Trigger实例。

其实Job就是使用者需要实现的调度任务接口，它以JobDetail的形式存放在Quartz管理的内存或者表里面。

Scheduler的生命期，从SchedulerFactory创建它的实例时开始，到Scheduler的实例调用shutdown()方法时结束。Scheduler被创建后，可以添加、删除和查询JobDetail和Trigger，以及执行其它们与调度相关的操作（如暂停Trigger）。但是，Scheduler只有在调用start()方法后，才会真正地触发Trigger（Trigger运作之后才能fire具体的Job），见[第一章：使用Quartz](lesson-1.md)。

Quartz提供的“builder”类，可以认为是一种领域特定语言（DSL，Domain Specific Language)。其实这些Builder就是流式的API。上一章你已经看过相应的例子，这里再贴出来一次：
```java
 // define the job and tie it to our HelloJob class
  JobDetail job = newJob(HelloJob.class)
      .withIdentity("myJob", "group1") // name "myJob", group "group1"
      .build();

  // Trigger the job to run now, and then every 40 seconds
  Trigger trigger = newTrigger()
      .withIdentity("myTrigger", "group1")
      .startNow()
      .withSchedule(simpleSchedule()
          .withIntervalInSeconds(40)
          .repeatForever())            
      .build();

  // Tell quartz to schedule the job using our trigger
  sched.scheduleJob(job, trigger);
```
定义JobDetail的代码使用的是从JobBuilder静态导入的方法。同样，定义Trigger的代码使用的是从TriggerBuilder静态导入的方法 。 另外，也导入了SimpleSchedulerBuilder类的静态方法。DSL的静态导入可以通过以下导入语句来实现：
```java
import static org.quartz.JobBuilder.*;
import static org.quartz.SimpleScheduleBuilder.*;
import static org.quartz.CronScheduleBuilder.*;
import static org.quartz.CalendarIntervalScheduleBuilder.*;
import static org.quartz.TriggerBuilder.*;
import static org.quartz.DateBuilder.*;
```
SchedulerBuilder接口的各种实现类，可以定义不同类型的调度计划（schedule）；DateBuilder类包含很多方法，可以很方便地构造表示不同时间点的java.util.Date实例（如定义下一个小时为偶数的时间点，如果当前时间为9:43:27，则定义的时间为10:00:00）。

其实就是SchedulerBuilder是策略接口，它的子类提供了多种不同类型的调度计划的实现，DateBuilder内部的多数方法依赖于Calendar，它主要功能是用来快速定义一个具体的时刻，因为有时候Cron表达式可能不满足特定的场景，这个时候DateBuilder可以派上用场。

## Jobs and Triggers：
一个调度任务就是一个实现了org.quartz.Job接口(只有一个简单的接口方法execute)的类：

**The Job Interface**
```java
 package org.quartz;

  public interface Job {

    public void execute(JobExecutionContext context)
      throws JobExecutionException;
  }
```
当Job的一个Trigger被触发（稍后会讲到）时，`execute()`方法由Scheduler的一个工作线程调用。传递给`execute()`方法的JobExecutionContext对象中保存着该Job运行时的一些信息 ，执行Job的Scheduler的引用，触发Job的Trigger的引用，JobDetail对象引用，以及一些其它信息（如果使用了Spring的话，可以传入Spring的上下文对象ApplicationContext）。

JobDetail对象是在将Job加入Scheduler时，由客户端程序（你的程序）创建的。它包含Job的各种属性设置，以及用于存储Job实例状态信息的JobDataMap。本节是对Job实例的简单介绍，更多的细节将在下一节讲到。

Trigger用于触发Job的执行。当你准备调度一个Job时，你创建一个Trigger的实例，然后设置调度相关的属性。Trigger也有一个相关联的JobDataMap，用于给Job传递一些触发相关的参数。Quartz自带了各种不同类型的Trigger，最常用的主要是SimpleTrigger(间隔一定时间(重复)执行)和CronTrigger(基于Cron表达式构建调度计划)。

SimpleTrigger主要用于一次性执行的Job（只在某个特定的时间点执行一次），或者Job在特定的时间点执行，重复执行N次，每次执行间隔T个时间单位。CronTrigger在基于日历的调度上非常有用，如“每个星期五的正午”，或者“每月的第十天的上午10:15”等。

为什么既有Job，又有Trigger呢？很多任务调度器并不区分Job和Trigger。有些调度器只是简单地通过一个执行时间和一些Job标识符来定义一个Job；其它的一些调度器将Quartz中描述的Job和Trigger对象合二为一。在开发Quartz的时候，我们认为将触发器和要调度的任务分离是合理的。在我们看来，这可以带来很多好处。

例如，**Job被创建后，可以保存在Scheduler中，与Trigger是独立的，同一个Job可以有多个Trigger；这种松耦合的另一个好处是，当与Scheduler中的Job关联的Trigger都过期时，可以配置Job稍后被重新调度，而不用重新定义Job；还有，可以修改或者替换Trigger，而不用重新定义与之关联的Job。**

译者注：上面这段内容十分重要，在Quartz中，调度任务和触发器是独立分离的，并且可以总结出一点：Quartz中Job是无状态的，有状态的是Trigger。因此我们在做一个调度任务查询列表展示的时候应该展示的是**触发器的状态**，而不应该是调度任务的状态；至于调度任务是否执行成功，只能通过添加监听器或者查看日志去判断或者说调度任务的运行状态应该交由开发者去监控和管理。

## Identities
Identities其实就是调度任务和触发器的身份标识。当Job和Trigger注册到Quartz的调度器中的时候需要定义相应的识别标记(其实就是JobKey和TriggerKey)。调度任务和触发器（JobKey和TriggerKey）的识别标记中允许使用“分组(group)”，这对于组织你的工作和触发诸如“报告工作”和“维护工作”等类别是有用的。作业或触发器的键的名称部分必须在组内是惟一的---换句话说，作业或触发器的完整键（或标识符）是名称（name）和组别（group）的复合。这里可以先这样理解，JobKey(name和group)是JobDetail的联合主键，TriggerKey(name和group)是Trigger的联合主键。

你现在对调度任务和触发器有了大致的了解，你可以在[第三章]()和[第四章]()了解到更多关于它们的使用方式。

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-02.html