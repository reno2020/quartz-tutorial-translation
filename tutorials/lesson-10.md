# 第十章：配置、资源的使用以及SchedulerFactory
# 内容
Quartz的架构设计是模块化的，因此要运行它需要把几个组件组合在一起使用。幸运的是，有一些工具就是为了完成这个目标。

Quartz在能够正常运作之前，下面的几个核心组件必须配置好：
- ThreadPool
- JobStore
- DataSources(如果需要的话)
- Scheduler(本身)

线程池提供了一组Quartz在执行Job时使用的线程。线程池中的线程越多，并发运行的Job数越多。但是，太多的线程可能会破坏你的系统。大多数Quartz的用户发现，5个左右的线程是充足的 - 因为在任何给定时间，他们的Job数量少于100个，通常不会同时运行这些Job，而且这些Job是短暂的（快速完成）。其他用户认为他们需要10个，15个，50个甚至100个线程，因为他们具有数万个具有各种调度计划的Trigger，最终在任何给定时刻平均执行10到100个Job。为调度器的线程池找到正确的线程池大小完全取决于你需要使用调度器来完成什么工作(这里意思应该是需要按照工作量去定义线程池大小)。没有真正的规则，除了保持线程数量尽可能小（为了你昂贵的服务器资源） - 但需要确保线程数量已足够让你的Job按时启动。请注意，如果Trigger的触发时间到达，并且线程池中没有可用的线程，Quartz将阻塞（暂停）直到线程可用，然后Job将执行 - 也就是在它本该执行的时间点延后一个时间段执行。这甚至可能导致线程的错过触发 - 如果在调度器中配置的“错过触发阈值”的持续时间内没有可用的线程。

ThreadPool接口在org.quartz.spi包中定义，你可以按照你喜欢的方式创建ThreadPool实现。Quartz带有一个名为org.quartz.simpl.SimpleThreadPool（虽然简单，但非常令人满意）的线程池。这个ThreadPool只是在它的线程池中维护一个固定的线程集 - 永远不会增长，永远不会缩小。但是它是非常强大的，并经过很好的测试 - 几乎所有使用Quartz的人都使用这个池。

JobStores和DataSource在本教程的[第九章：JobStores](tutorials/lesson-9.md)讨论过。值得注意的是，所有JobStore都实现了org.quartz.spi.JobStore接口 - 任何一个JobStore实现都不符合你的需求，那么你可以自己创建并且自行实现org.quartz.spi.JobStore接口。

最后，你需要创建你的Scheduler实例。Scheduler本身需要被赋予一个名字，告诉其RMI设置，并且设置JobStore和ThreadPool的实例。RMI设置包括调度程序是否应将自身创建为RMI的服务器对象（使其可用于远程连接），要使用的主机和端口等。StdSchedulerFactory（下面将讨论）还可以创建Scheduler实例，这些Scheduler实例可以是创建在远程程序中的Scheduler的代理(RMI存根)。

## StdSchedulerFactory
StdSchedulerFactory是org.quartz.SchedulerFactory接口的一个实现。它使用一组属性（java.util.Properties）来创建和初始化Quartz的Scheduler实例。属性通常存储在文件中并从文件中加载，但也可以由程序创建并直接传递到工厂类实例。简单地调用工厂中的getScheduler()将生成Scheduler实例，并初始化它（和它的ThreadPool，JobStore和DataSource）并返回一个公共接口org.quartz.Scheduler的句柄。

在Quartz发行版的"docs/config"目录中有一些示例配置（包括属性的描述）。你可以在Quartz文档的“参考”部分的“配置”手册中找到完整的文档。

## DirectSchedulerFactory
DirectSchedulerFactory是另一个SchedulerFactory实现。对于希望编程式创建其Scheduler实例的用户是有用的。通常不鼓励使用它，原因如下：
- （1）DirectSchedulerFactory要求用户更好地了解他们正在做什么。
- （2）它不允许声明性配置(不能使用配置文件) - 换句话说，你需要硬编码Scheduler实例的所有属性项。

## 日志
Quartz使用SLF4J框架来满足所有的日志记录需求。为了“调整”日志记录设置（例如输出量以及输出位置），你需要了解SLF4J框架，这超出了本文档的范围。

如果需要捕获Trigger启动和Job执行的额外信息，可以启用org.quartz.plugins.history.LoggingJobHistoryPlugin或org.quartz.plugins.history.LoggingTriggerHistoryPlugin。

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-10.html