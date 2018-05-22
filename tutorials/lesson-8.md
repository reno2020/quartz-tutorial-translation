# 第八章：Scheduler监听器
# 内容
SchedulerListener和TriggerListener或JobListener十分相似，它接收调度器的相关事件，调度器的相关事件不一定和特定的Trigger或者Job相关。

与Scheduler相关的事件包括：添加Job/Trigger、删除Job/Trigger、调度器中的严重错误以及关闭调度器的通知等等。

**org.quartz.SchedulerListener接口**
```java
public interface SchedulerListener {

    public void jobScheduled(Trigger trigger);

    public void jobUnscheduled(String triggerName, String triggerGroup);

    public void triggerFinalized(Trigger trigger);

    public void triggersPaused(String triggerName, String triggerGroup);

    public void triggersResumed(String triggerName, String triggerGroup);

    public void jobsPaused(String jobName, String jobGroup);

    public void jobsResumed(String jobName, String jobGroup);

    public void schedulerError(String msg, SchedulerException cause);

    public void schedulerStarted();

    public void schedulerInStandbyMode();

    public void schedulerShutdown();

    public void schedulingDataCleared();
}
```
SchedulerListener也是通过ListenerManager注册到调度器。SchedulerListener几乎可以是任何实现了org.quartz.SchedulerListener接口的对象(<--译者吐槽：这句话怎么看都觉得有点多余)。

**添加SchedulerListener：**
```java
scheduler.getListenerManager().addSchedulerListener(mySchedListener);
```
**删除SchedulerListener：**
```java
scheduler.getListenerManager().removeSchedulerListener(mySchedListener);
```

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-08.html
