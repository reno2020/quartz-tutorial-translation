# 第一章：使用Quartz
# 前提
术语：
- Scheduler：调度器。
- SchedulerFactory：调度器工厂。
- Trigger：触发器。
- Job：(调度)任务或者作业，在Quartz中体现为JobDetail。

在后面的翻译中，因为个人习惯，可能会中英互用，映射关系为：
- Scheduler === 调度器
- SchedulerFactory === 调度器工厂
- Trigger === 触发器
- Job、JobDetail === (调度)任务
- fire === 触发

# 内容
在使用此调度器(Scheduler)之前，它需要被实例化(谁猜到这一点了? <-- 这里估计是官方的调皮)。为了实例化调度器，你需要用到SchedulerFactory。一些Quartz的使用者可能会在JNDI存储中保留SchedulerFactory的实例，其他使用者可能会觉得直接初始化会更加简单（例如下面的示例）。

一旦调度器完成了实例化，就可以启动(start)、暂停(stand-by)、停止(shutdown)。注意：一旦调度器被停止，它就不能够重新启动，除非重新实例化另一个调度器实例。所有的触发器(Trigger)不会触发任务(也就是任务不会执行)，除非调度器已经启动。但是如果调度器(虽然已经启动)处于暂停状态，所有的触发器也不会触发任务。

下面是一个代码片段，实例化并启动一个调度器，调度执行一个任务：
```java
 SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();

  Scheduler sched = schedFact.getScheduler();

  sched.start();

  // define the job and tie it to our HelloJob class
  JobDetail job = newJob(HelloJob.class)
      .withIdentity("myJob", "group1")
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
正如你所见，使用Quartz是十分简单的。在下一节[第二章：Quartz API、调度任务以及触发器](lesson-2.md)中我们将会概述一下调度任务、触发器以及Quartz的API，以便你可以更全面地了解这个例子。

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-01.html 