# 第七章：Trigger监听器和Job监听器
监听器（listener）是你创建的对象，主要作用是接收和处理调度器回调的事件(event)。TriggerListener接收到与触发器（Trigger）相关的事件，JobListener接收与调度任务(Job)相关的事件。

与触发器相关的事件包括：触发器正要触发，触发器错失触发，触发器触发完成（调度任务已被触发开始执行，触发器完成当次触发）。

**org.quartz.TriggerListener接口**
```java
public interface TriggerListener {

    public String getName();

    public void triggerFired(Trigger trigger, JobExecutionContext context);

    public boolean vetoJobExecution(Trigger trigger, JobExecutionContext context);

    public void triggerMisfired(Trigger trigger);

    public void triggerComplete(Trigger trigger, JobExecutionContext context,
            int triggerInstructionCode);
}
```
与调度任务相关的事件包括：Job即将执行时的通知以及Job执行完成时的通知。
**org.quartz.JobListener接口**
```java
public interface JobListener {

    public String getName();

    public void jobToBeExecuted(JobExecutionContext context);

    public void jobExecutionVetoed(JobExecutionContext context);

    public void jobWasExecuted(JobExecutionContext context,
            JobExecutionException jobException);

}
```

## 使用自定义的监听器
要创建一个Listener，只需创建一个实现org.quartz.TriggerListener或org.quartz.JobListener接口实现的实例。然后，Listener需要在运行时注册到调度器中。Listener必须指定一个名称（通过实现JobListener接口的`getName()`方法来传入监听器的名字）。

为了方便起见，自定义监听器也可以继承自JobListenerSupport类或TriggerListenerSupport类，并且只需覆盖你感兴趣的方法。

Listener通过调度器的ListenerManager进行注册，可以通过一个匹配器(Matcher)实例去匹配监听你感兴趣的Job或者Trigger。注意下面的事项：
```
Listener在运行时才注册到调度器中，并且不与Job或者触发器一起存储在JobStore中。这是因为通常监听器是直接集成到应用程序之中(这里的意思大概是监听器中会有应用程序里面相关的逻辑)。因此每当应用程序启动的时候，所有的监听器需要重新注册到调度器中。
```

**添加匹配的特定Job的JobListener：**
```java
scheduler.getListenerManager().addJobListener(myJobListener, KeyMatcher.jobKeyEquals(new JobKey("myJobName", "myJobGroup")));
```
你可能需要为匹配器和关键类使用静态导入，这将使你定义匹配器的逻辑更简洁：
```java
import static org.quartz.JobKey.*;
import static org.quartz.impl.matchers.KeyMatcher.*;
import static org.quartz.impl.matchers.GroupMatcher.*;
import static org.quartz.impl.matchers.AndMatcher.*;
import static org.quartz.impl.matchers.OrMatcher.*;
import static org.quartz.impl.matchers.EverythingMatcher.*;
...etc.
```
使用静态导入后上面的例子变成这样：
```java
scheduler.getListenerManager().addJobListener(myJobListener, jobGroupEquals("myJobGroup"));
```
添加对两个特定Group的所有Job感兴趣的JobListener：
```java
scheduler.getListenerManager().addJobListener(myJobListener, or(jobGroupEquals("myJobGroup"), jobGroupEquals("yourGroup")));
```
添加对所有Job感兴趣的JobListener：
```java
scheduler.getListenerManager().addJobListener(myJobListener, allJobs());
```
注册TriggerListener的工作原理和JobListener基本相同。

大多数Quartz的使用者并不使用Listener。但是当应用程序有获取任务调度相关的事件通知的需求时，如果使用了Listener可以避免在Job的逻辑里面加入额外的通知逻辑，这一点对于使用者而言是很方便的。

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-07.html