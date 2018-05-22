## 第三章：Job和JobDetail的更多细节
# 内容
正如你在[第二章：Quartz API、调度任务以及触发器](lesson-2.md)看到的，org.quartz.Job很容易实现，在接口中只有一个`execute`方法。本节主要关注：Job的特点、Job接口的execute方法以及JobDetail。

你定义了一个实现Job接口的类，这个类仅仅表明该Job需要完成什么类型的任务，除此之外，Quartz还需要知道该Job实例所包含的属性；这将由JobDetail类来完成。

JobDetail实例是通过JobBuilder类创建的，导入该类下的所有静态方法，会让你编码时有DSL的感觉：
```jav
 import static org.quartz.JobBuilder.*;
```
让我们先看看Job的特征（nature）以及Job实例的生命期。不妨先回头看看[第一章：使用Quartz](lesson-1.md)中的代码片段：
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
现在这样定义下面这样的一个调度任务类“HelloJob”：
```java
 public class HelloJob implements Job {

    public HelloJob() {
    }

    public void execute(JobExecutionContext context)
      throws JobExecutionException{
      System.err.println("Hello!  HelloJob is executing.");
    }
  }
```
可以看到，我们传给Scheduler一个JobDetail实例，因为我们在创建JobDetail时，将要执行的Job的类名(类型)传给了JobDetail，所以Scheduler就知道了要执行何种类型的Job；每次当Scheduler执行Job时，在调用其`execute(…)`方法之前会**创建该类的一个新的实例**；执行完毕，对该实例的引用就被丢弃了，实例会被垃圾回收；这种执行策略带来的一个后果是，**Job必须有一个无参的构造函数**（当使用默认的JobFactory时）；另一个后果是，在Job类中，不应该定义有状态的数据属性，因为在Job的多次执行中，这些属性的值不会保留。

译者注：这里又一次强调了Job是无状态的。另外，可以实现自定义的JobFactory来改变获取Job实例的方式，例如从Spring的IOC容器中获取，这样就不用每次新建一个Job实例再执行。

那么如何给Job实例增加属性或配置呢？如何在Job的多次执行中，跟踪Job的状态呢？答案就是:JobDataMap，JobDetail对象的一部分。

## JobDataMap
JobDataMap中可以包含不限量的（序列化的）数据对象，在Job实例执行的时候，可以使用其中的数据；JobDataMap是java.util.Map接口的一个实现，额外增加了一些便于存取基本类型的数据的方法。

将Job加入到Scheduler之前，在构建JobDetail时，可以将数据放入JobDataMap，如下示例：
```java
 // define the job and tie it to our DumbJob class
  JobDetail job = newJob(DumbJob.class)
      .withIdentity("myJob", "group1") // name "myJob", group "group1"
      .usingJobData("jobSays", "Hello World!")
      .usingJobData("myFloatValue", 3.141f)
      .build();
```
在Job的执行过程中，可以从JobDataMap中取出数据，如下示例：
```java
public class DumbJob implements Job {

    public DumbJob() {
    }

    public void execute(JobExecutionContext context)
      throws JobExecutionException
    {
      JobKey key = context.getJobDetail().getKey();

      JobDataMap dataMap = context.getJobDetail().getJobDataMap();

      String jobSays = dataMap.getString("jobSays");
      float myFloatValue = dataMap.getFloat("myFloatValue");

      System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
    }
  }
```
**如果您使用具备持久化特性的JobStore（在本教程的JobStore部分中进行了讨论），您应该谨慎决定放置在JobDataMap中的内容，因为其中的对象将被序列化，因此它们容易出现类版本问题。**显然，标准的Java类型应该是非常安全的，但除此之外，任何时候有人改变你已经序列化了实例的类的定义时，都必须注意不要打破兼容性。另外，你可以强制要求JDBC-JobStore和JobDataMap只允许在JobDataMap中存储基本类型和字符串类型，这样可以避免后续的序列化问题。

如果你在Job的实现类中，为JobDataMap中存储的数据的key增加set方法（如在上面示例中，增加setJobSays(String val)方法），那么Quartz的默认JobFactory实现在Job被实例化的时候会自动调用这些set方法，这样你就不需要在`execute()`方法中显式地从JobDataMap中取数据了。

如果你有一个存储在调度器中的调度任务，供多个触发器定期/重复使用，但每次独立触发，则希望为作业提供不同的数据输入，可以为触发器定义与之关联的JobDataMap。

在Job执行时，JobExecutionContext中的JobDataMap为我们提供了很多的便利。它是JobDetail中的JobDataMap和Trigger中的JobDataMap的数据并集，但是如果存在相同的数据，则后添加者会覆盖之前添加的Key相同的值（毕竟就是一个java.util.Map的实例）。

以下是在作业执行期间从JobExecutionContext的合并JobDataMap获取数据的简单示例：
```java
public class DumbJob implements Job {

    public DumbJob() {
    }

    public void execute(JobExecutionContext context)
      throws JobExecutionException
    {
      JobKey key = context.getJobDetail().getKey();

      JobDataMap dataMap = context.getMergedJobDataMap();  // Note the difference from the previous example

      String jobSays = dataMap.getString("jobSays");
      float myFloatValue = dataMap.getFloat("myFloatValue");
      ArrayList state = (ArrayList)dataMap.get("myStateData");
      state.add(new Date());

      System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
    }
  }
```
如果你希望使用JobFactory实现数据的自动“注入”功能，则示例代码为：
```java
  public class DumbJob implements Job {


    String jobSays;
    float myFloatValue;
    ArrayList state;

    public DumbJob() {
    }

    public void execute(JobExecutionContext context)
      throws JobExecutionException
    {
      JobKey key = context.getJobDetail().getKey();

      JobDataMap dataMap = context.getMergedJobDataMap();  // Note the difference from the previous example

      state.add(new Date());

      System.err.println("Instance " + key + " of DumbJob says: " + jobSays + ", and val is: " + myFloatValue);
    }

    public void setJobSays(String jobSays) {
      this.jobSays = jobSays;
    }

    public void setMyFloatValue(float myFloatValue) {
      myFloatValue = myFloatValue;
    }

    public void setState(ArrayList state) {
      state = state;
    }

  }
```
你会注意到该类的整体代码更长，但`execute()`方法中的代码更简洁清晰。而且，虽然代码更多了，但如果你的IDE可以自动生成setter方法，你就不需要写代码调用相应的方法从JobDataMap中获取数据了，所以你实际需要编写的代码更少了。当前，如何选择，由你决定。

## Job实例
很多使用者对于Job实例到底由什么构成感到很迷惑。我们将尝试在这里解释一下，并且在下面的部分中解释关于调度任务状态和并发性相关内容。

您可以创建一个单独的Job类，并通过创建JobDetails的多个实例（每个实例具有自己的属性集和JobDataMap），最后，将所有的Job实例都添加加到Scheduler中。

例如，你可以创建一个实现Job接口的类，名为“SalesReportJob”。该Job需要一个参数（通过JobDataMap传入）指定销售报告应该基于的销售人员的名称。然后，你可以创建多个Job实例（JobDetails），比如“SalesReportForJoe”、“SalesReportForMike”，将“joe”和“mike”作为JobDataMap的数据传给对应的Job实例。

当触发器触发时，它所关联的JobDetail（实例定义）将被加载，并且它引用的作业类将通过Scheduler上配置的JobFactory实例化。默认的JobFactory只是在作业类上调用`newInstance()`，然后尝试调用该类的匹配JobDataMap中的键的名称的setter方法。你可能希望创建自己的JobFactory实现来完成诸如让应用程序的IoC或DI容器生成/初始化Job实例等事情。

在Quartz的描述语言中，我们将每个存储的JobDetail称为“Job定义”或“JobDetail实例”，并将每个正在执行的作业称为“Job实例”或“Job定义的实例”。通常，如果我们只使用“Job”这个词，我们就是指一个命名定义，或者JobDetail。当我们指的是实现工作界面的类时，我们通常使用术语“Job类”。

## Job的状态和并发性
关于Job状态数据（也就是JobDataMap）和并发性还有一些内容需要补充。有几个注释可以添加到你的Job类中，这些注解会影响Job的状态和并发性。

@DisallowConcurrentExecution是一个注解，可以添加到Job类中，告诉Quartz不要同时执行给定Job定义（指给定Job类）的多个实例。
注意这里的措词。拿前一小节的例子来说，如果“SalesReportJob”类上有该注解，则同一时刻仅允许执行一个“SalesReportForJoe”实例，但可以并发地执行“SalesReportForMike”类的一个实例。所以该限制是针对JobDetail的，而不是Job类。但是我们认为（在设计Quartz的时候）应该将该注解放在Job类上，因为Job类的改变经常会导致其行为发生变化。

@PersistJobDataAfterExecution是一个注解，可以添加到Job类中，告诉Quartz在`execute()`方法成功完成后（不抛出异常）更新JobDetail的JobDataMap的存储副本数据，使得该Job（即JobDetail）在下一次执行的时候，JobDataMap中是更新后的数据，而不是更新前的旧数据。像 @DisallowConcurrentExecution注解一样，尽管注解是加在Job类上的，但其限制作用是针对Job实例的，而不是Job类的。由Job类来承载该注解，是因为此注解的功能会影响类的编码方式（例如，'有状态'需要被执行方法内的代码明确'理解'--> 这里原文是e.g. the ‘statefulness’ will need to be explicitly ‘understood’ by the code within the execute method）。

如果你使用了@PersistJobDataAfterExecution注解，我们强烈建议你同时使用@DisallowConcurrentExecution注解，因为当同一个Job（JobDetail）的两个实例被并发执行时，由于竞争，JobDataMap中存储的数据很可能是不确定的。

## Job的其他属性
通过JobDetail对象，可以给Job实例配置的其它属性有：
- Durability：持久化特性，布尔值。如果一个Job是非持久的，当没有活跃的Trigger与之关联的时候，会被自动地从Scheduler中删除。也就是说，非持久的Job的生命期是由Trigger的存在与否决定的。
- RequestsRecovery - 请求恢复特性，布尔值。如果一个Job是可恢复的，并且在其执行的时候，Scheduler发强制关闭（hard shutdown)（比如运行的进程崩溃了或者服务器宕机了），则当Scheduler重新启动的时候，该Job会被重新执行。此时，该Job的`JobExecutionContext.isRecovering()`返回true。

这两个属性在JobBuilder中都有相应的设置方法。

## JobExecutionException
最后，我们需要通知你关于该`Job.execute(..)`方法的一些细节。唯一可以从execute方法抛出的异常（包括RuntimeExceptions）是JobExecutionException。因此，通常应该用“try-catch”块来包装`execute`方法的全部内容。你还应该花一些时间查看JobExecutionException的文档，因为你的Job可以使用该异常告诉Scheduler，你希望如何来处理发生的异常。

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-03.html 
