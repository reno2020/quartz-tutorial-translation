# 第十一章：高级(企业级)特性
# 内容
## 集群
Quartz集群目前与JDBC-Jobstore（JobStoreTX或JobStoreCMT）和(<--译者注：其实我觉得这里应该是"或")TerracottaJobStore一起使用。功能包括负载均衡和Job故障转移（如果JobDetail的“请求恢复(request recovery)”标志设置为true）。

使用JobStoreTX或JobStoreCMT通过将"org.quartz.jobStore.isClustered"属性设置为“true”来启用集群模式。**集群中的每个实例都应该使用相同的quartz.properties文件。**这些相同的配置文件中，也允许下面几项属性配置是可以不相同的：不同的线程池大小，以及"org.quartz.scheduler.instanceId"设置不同的属性值。集群中的每个节点必须具有唯一的instanceId，通过将"AUTO"作为此属性的值，可以轻松完成此定义（这样就不需要使用不同的配置文件）。下面是注意事项：
```
不要在各自独立的机器上各自开启集群模式，除非它们的时钟使用某种形式的时间同步服务（守护进程）进行同步，而这些时间同步服务（守护进程）运行得非常有规律（各个机器的时钟差距必须在一秒内）。 如果你不熟悉如何执行此操作， 请参阅http://www.boulder.nist.gov/timefreq/service/its.htm。
```

```
非集群模式下，两个不同的实例千万不要使用同一套Quartz的表。否则，你一定会遇到不正常的(调度)行为，有可能会遭遇严重的数据损坏，。
```

集群中触发机制是：Job每次只能被一个节点触发(也就是虽然集群中每台机器都跑着Quartz的调度器，但是Job需要被触发的时刻只有一台机器会进行触发)。我的意思是，如果Job有一个重复的Trigger，告诉它每10秒钟触发一次，那么在12:00:00，正好一个节点将执行这个Job，在12:00:10，也是由一个节点将执行此Job等。并不一定是每次由相同的节点执行Job - 由哪个节点执行它是随机的。负载均衡机制对于繁忙的调度程序（大量的Trigger）来说是近乎随机的。但是对于非繁忙(只有一两个Trigger)的调度器集群来说，有可能偏向于由同一个节点执行。

使用TerracottaJobStore建立Quartz集群只需要将把Scheduler的JobStore配置为TerracottaJobStore（[第九章：JobStores](lesson-9.md)中介绍过）即可，然后你的调度器就会被设置为集群模式。

您可能还需要考虑如何设置Terracotta服务器，特别是开启持久化特性等功能的配置选项，以及搭建一些列高可用的Terracotta服务器。

TerracottaJobStore的企业版提供了高级的"Quartz Where"功能，允许将作业的智能定位到适当的Quartz集群节点。

有关JobStore和Terracotta的更多信息，请访问http://www.terracotta.org/quartz

## JTA事务
正如[第九章：JobStores](lesson-9.md)所述，JobStoreCMT允许在较大的JTA事务中执行Quartz调度操作。

通过将"org.quartz.scheduler.wrapJobExecutionInUserTransaction"属性设置为"true"，Job也可以在JTA事务（UserTransaction）内执行。使用此选项集，一个JTA事务将在Job的execute方法被调用之前调用其begin()方法，并且在Job执行完成调用之后调用其commit()。这适用于所有的Job。

如果你希望指定每个Jobs是否包裹在JTA事务内执行，那么你应该在Job类上使用@ExecuteInJTATransaction注解。

除了Quartz自动将Job执行包装到JTA事务中，使用JobStoreCMT之后在Scheduler接口方法也可以使用事务处理。你可以明确一点：在使用Scheduler接口方法之前已经开启了一个事务。你可以直接通过使用UserTransaction或将使用调度程序的代码放在使用容器管理事务的SessionBean中来执行此操作。

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-11.html 