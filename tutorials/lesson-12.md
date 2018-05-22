# 第十二章：其他特性
# 内容
## 插件
Quartz提供了一个用于插入附加功能的接口（org.quartz.spi.SchedulerPlugin）。

你可以从org.quartz.plugins 包中找到提供各种实用功能的Quartz插件。它们提供诸如在调度器启动时自动调度Job的功能，记录Job和Trigger相关事件的历史，并确保当JVM退出时调度器将彻底关闭。

## JobFactory
当Trigger触发时，通过Scheduler上配置的JobFactory实例化与之关联的Job。默认的JobFactory只是在Job类上(反射)调用newInstance()。你可能需要创建自己的JobFactory实现，以完成诸如让应用程序的IoC或DI容器生成/初始化Job实例等等的操作。

请参阅org.quartz.spi.JobFactory接口以及Scheduler.setJobFactory（fact)等相关方法。

## 'Factory-Shipped' Jobs
Quartz还提供了许多实用Job类型，你可以在应用程序中用于执行诸如发送电子邮件和调用EJB等工作。这些开箱即用的Job类型可以在org.quartz.jobs包中找到(要引入依赖quartz-jobs)。


原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-12.html