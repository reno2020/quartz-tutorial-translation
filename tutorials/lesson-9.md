# 第九章：JobStores
JobStore负责记录你提供到调度器的所有“工作数据”：所有的Job、所有的Trigger、所有的Calendar(org.quartz.Calendar)等等。为你的Quartz调度器选择一个合适的JobStore是一个重要的步骤。幸运的是，一旦你明白不同的JobStore之间的差异，那么作出合适的选择是十分简单的。

你声明你提供给用于生成调度器实例对应的SchedulerFactory实例时候用到的属性文件（或对象）中，应该指定你的调度器应使用哪个类型的JobStore（以及它的相关配置）。注意一点：
```
切勿在代码中直接使用JobStore实例。由于某些原因，许多使用者试图这样做。JobStore应该只用于Quartz的后台。你必须告诉Quartz（通过配置）使用哪个类型的JobStore，你在代码中应该只能使用Scheduler接口。
```

## RAMJobStore
RAMJobStore是使用最简单的JobStore，它也是性能最高的（在CPU时间方面）。RAMJobStore的功能显然和它的名字相关：它将其所有数据保存在RAM(内存)中，这就是为什么它是闪电般的快，也是为什么它的的配置这么简单。缺点是当你的应用程序结束（或崩溃）时，所有调度信息都将丢失 - 这意味着RAMJobStore无法履行作业和Trigger上的“非易失性”设置。对于某些应用程序，这是可以接受的 - 甚至是所需的行为，但对于其他应用程序，这可能是灾难性的。

要使用RAMJobStore（并假设您使用的是StdSchedulerFactory），只需将Quartz的JobStore类属性配置指定为org.quartz.simpl.RAMJobStore：
```java
org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore
```
除此之外没有其他需要担心的配置项。

## JDBCJobStore
JDBCJobStore的命名也相当恰当 - 它通过JDBC将其所有数据保存在数据库中。因此，它配置比RAMJobStore要复杂一点，而且也不是那么快。但是，它性能下降并不是很糟糕，特别是如果你使用的数据库表在相应的主键或者外键加上索引。在相对主流的并且有一个像样的局域网（在调度器和数据库之间）的机器上，检索和更新一个触发中的Trigger的时间通常将小于10毫秒。

JDBCJobStore几乎可以在任何类型的任何数据库中使用，已被广泛应用于Oracle，PostgreSQL，MySQL，MS SQLServer，HSQLDB和DB2。要使用JDBCJobStore，必须首先创建一组数据库表以供Quartz使用。你可以在Quartz发行版的"docs/dbTables"目录中找到表创建SQL脚本。如果你的数据库类型尚未有脚本，请查看其中一个脚本，然后以数据库所需的任何方式进行修改。需要注意的一点是，在这些脚本中，所有的表都以前缀"QRTZ_"开始（如表"QRTZ_TRIGGERS"和"QRTZ_JOB_DETAIL"）。你可以通知JDBCJobStore表的前缀是什么（在你的Quartz属性配置中），也就是你也可以修改这个表前缀的值。对于多个调度程序实例，使用不同的前缀可能有助于同一个数据库中的多个调度器实例创建多组表。

创建表后，在配置和启动JDBCJobStore之前，你还有一个重要的决定。你需要确定应用程序需要哪种类型的事务。如果你不需要将调度命令（例如添加和删除Trigger）绑定到其他代码逻辑的事务中，那么可以通过使用JobStoreTX对JobStore进行事务管理（这是最常见的选择）。

如果你需要Quartz与其他事务（即J2EE应用程序服务器）一起工作，那么你应该使用JobStoreCMT - 在这种情况下，Quartz将让应用程序服务器容器管理事务。

最后一个难题是设置一个DataSource，使得JDBCJobStore可以从中获取数据库的连接。定义Quartz的DataSource有下面的几种方式。一种方法是让Quartz创建和管理DataSource本身 - 通过提供数据库的所有连接信息。另一种方法是通过提供JDBCJobStore的DataSource的JNDI名称，让Quartz使用由Quartz正在运行的应用程序服务器来管理DataSource。有关属性的详细信息，请参阅发行版本中"docs/config"文件夹中的示例配置文件。

要使用JDBCJobStore（假设你使用的是StdSchedulerFactory），首先需要将Quartz配置文件中的的JobStore类属性设置为org.quartz.impl.jdbcjobstore.JobStoreTX或org.quartz.impl.jdbcjobstore.JobStoreCMT - 具体可以参考上一段文字的描述，不过决定权还是在你手中。

**配置Quartz以使用JobStoreTx**
```
org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
```

接下来，你需要为JobStore选择一个DriverDelegate。DriverDelegate负责执行特定数据库可能需要的任何JDBC相关的工作。StdJDBCDelegate是一个使用“vanilla(原意识香草味的，这里大概的意思是原生的)”JDBC代码（和SQL语句）来工作的。如果Quartz没有为你的数据库类型提供特定的DriverDelegate，请尝试使用此DriverDelegate - 我们仅仅为那些使用StdJDBCDelegate(几乎是最常使用)出现了问题的数据库类型提供了特殊的订造的DriverDelegate。其他DriverDelegate可以在"org.quartz.impl.jdbcjobstore"包或其子包中找到。其他DriverDelegate包括DB2v6Delegate（用于DB2版本6及更早版本），HSQLDBDelegate（用于HSQLDB），MSSQLDelegate（用于Microsoft SQLServer），PostgreSQLDelegate（用于PostgreSQL）），WeblogicDelegate（用于使用Weblogic创建的JDBC驱动程序）。

一旦你选择了DriverDelegate后，将其类名设置为JDBCJobStore的对应属性以便JDBCJobStore能够使用它。

为JDBCJobStore配置DriverDelegate
```
org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate
```
接下来，你需要通知JobStore你正在使用的表前缀（如上所述）。

**配置JDBCJobStore对应的表前缀**
```
org.quartz.jobStore.tablePrefix = QRTZ_
```
最后，你需要设置JobStore应该使用哪个DataSource。DataSource的命名也必须在Quartz配置文件中的属性中定义。在这种情况下，我们指定Quartz应该使用DataSource名称"myDS"（在配置属性中的其他位置也是用这个名称去定义）。

**配置JDBCJobStore对应的DataSource**
```
org.quartz.jobStore.dataSource = myDS
```
注意事项一：
```
如果你的调度器一直处于忙碌的状态(满负载)（正在执行的Job数量几乎与线程池大小相同），那么你应该将DataSource中的连接数设置为线程池容量+2。
```
注意事项二：
```
可以将"org.quartz.jobStore.useProperties"配置参数设置为"true"（默认为false），以指示JDBCJobStore将JobDataMap中的所有值都作为字符串，从而JobDataMap中的属性可以作为键值对存储，而不是使用BLOB类型，因为BLOB类型是为了以其序列化形式存储更多复杂的对象。从长远来看，这是更安全的，因为你避免了将非String类序列化为BLOB类型的版本问题。
```

## TerracottaJobStore
TerracottaJobStore提供了一种伸缩性和鲁棒性的手段，它并不使用数据库。这意味着你的数据库可以免受Quartz的负载，可以将数据库所有资源分配给应用程序的其余部分。

TerracottaJobStore可以运行在群集或非群集环境在集群或者非集群环境下，TerracottaJobStore都可以为应用程序的作业数据提供存储介质，即便是应用程序重启的间隙，因为数据是存储在Terracotta服务器中。它的性能比基于使用数据库的JDBCJobStore要好得多（约一个数量级），但比RAMJobStore要慢。

要使用TerracottaJobStore（假设你使用的是StdSchedulerFactory），只需将配置文件中的类名称org.quartz.jobStore.class = org.terracotta.quartz.TerracottaJobStore指定为Quartz的JobStore类属性，并添加一个额外的配置项来指定Terracotta服务器的位置即可：

**使用TerracottaJobStore配置Quartz**
```
org.quartz.jobStore.class = org.terracotta.quartz.TerracottaJobStore
org.quartz.jobStore.tcConfigUrl = localhost:9510
```
更多关于JobStore和Terracotta的内容可以参阅http://www.terracotta.org/quartz。

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-09.html

