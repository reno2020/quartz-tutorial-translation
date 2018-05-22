# 第四章：关于Trigger的更多细节
# 内容
与Job一样，Trigger也很容易使用，但是还有一些扩展选项需要理解，以便更好地使用Qartz。Trigger也有很多类型，我们可以根据实际需要来选择。

最常用的两种Trigger会分别在[第五章：SimpleTrigger](lesson-5.md)和[第六章：CronTrigger](lesson-6.md)中讲到。

## Trigger的公共属性
所有类型的Trigger都有TriggerKey这个属性，表示Trigger的身份（唯一标识）；除此之外，Trigger还有很多其它的公共属性。这些属性，在构建Trigger的时候可以通过TriggerBuilder设置。Trigger的公共属性有：
- jobKey属性：当Trigger触发时被执行的Job的身份（唯一标识）；
- startTime属性：设置trigger第一次触发的时间点，该属性的值是java.util.Date类型，表示某个指定的时间点；有些类型的Trigger，会在设置的startTime时立即触发，有些类型的Trigger，表示其触发是在startTime之后开始生效。比如，现在是1月份，你设置了一个Trigger--“在每个月的第5天执行”，然后你将startTime属性设置为4月1号，则该Trigger第一次触发会是在几个月以后了(即4月5号)。
- endTime属性：表示Trigger失效的时间点，该属性的值是java.util.Date类型。比如，”每月第5天执行”的trigger，如果其endTime是7月1号，则其最后一次执行时间是6月5号。

其它的属性，会在下文中解释。

## Priority(优先级)
有时候，当你有很多Trigger实例（或者你的Quartz线程池中只有有少量工作线程，不足以触发所有的触发器）时，Quartz可能没有足够的资源来立即触发所有计划同时触发的触发器。在这种情况下，你可能想要控制哪些Trigger可以优先使用（在当前可用的）Quartz工作线程。为此，你可以在触发器上设置优先级属性。如果N个触发器同时触发，但当前只有Z个工作线程可用，则首先执行具有最高优先级的Z个触发器。如果您没有在触发器上设置优先级，那么它将使用默认优先级5。priority属性的值可以是任意整数，正数或者负数都是允许的。
- 注意：只有同时触发的Trigger之间才会比较优先级。10:59触发的Trigger总是在11:00触发的Trigger之前执行。
- 注意：如果Trigger是可恢复的，在恢复后再调度时，优先级与原Trigger是一样的。

## Misfire Instructions(错过触发策略)
Trigger还有一个重要的属性misfire（这里应该称为“错过触发策略”更合理，本质就是一次处理触发器错失触发的策略）；如果Scheduler关闭了，或者Quartz线程池中没有可用的线程来执行jJb，此时持久性的Trigger就会错过(miss)其触发时间，即错过触发(misfire)。不同类型的Trigger，有不同的misfire机制。它们默认都使用“智能机制(smart policy)”，即根据Trigger的类型和配置动态调整行为。当Scheduler启动的时候，查询所有错过触发(misfire)的持久化的Trigger。然后根据它们各自的misfire机制更新Trigger的信息。当你在项目中使用Quartz时，你应该对各种类型的Trigger的misfire机制都比较熟悉，这些misfire机制在JavaDoc中有说明。关于misfire机制的细节，会在讲到具体的Trigger时再作介绍。

## Calendar(日历)
Quartz的org.quartz.Calendar对象(不是java.util.Calendar对象)可以在定义和存储Trigger的时候与Trigger进行关联。Calendar用于从Trigger的调度计划中排除时间段。比如，可以创建一个Trigger，每个工作日的上午9:30执行，然后增加一个Calendar，排除掉所有的业务假期(也就是工作日中的法定假期)。

任何实现了Calendar接口的可序列化对象都可以作为Calendar对象，Calendar接口如下：
```java
package org.quartz;

public interface Calendar {

  public boolean isTimeIncluded(long timeStamp);

  public long getNextIncludedTime(long timeStamp);

}
``
译者吐槽：这里贴出来的Calendar接口的源码估计是老版本的org.quartz.Calendar接口，对比了下**2.2.x**版本的同一个接口并不是长这样的。

注意到这些方法的参数类型为long。你也许猜到了，他们就是毫秒单位的时间戳。即Calendar排除时间段的单位可以精确到毫秒。你也许对“排除一整天”的Calendar比较感兴趣。Quartz提供的org.quartz.impl.HolidayCalendar类可以很方便地实现。

Calendar必须先实例化，然后通过`addCalendar()`方法注册到Scheduler。如果使用HolidayCalendar，实例化后，需要调用`addExcludedDate(Date date)`方法从调度计划中排除时间段。以下示例是将同一个Calendar实例用于多个Trigger：
```java
HolidayCalendar cal = new HolidayCalendar();
cal.addExcludedDate( someDate );
cal.addExcludedDate( someOtherDate );

sched.addCalendar("myHolidays", cal, false);


Trigger t = newTrigger()
    .withIdentity("myTrigger")
    .forJob("myJob")
    .withSchedule(dailyAtHourAndMinute(9, 30)) // execute job daily at 9:30
    .modifiedByCalendar("myHolidays") // but not on holidays
    .build();

// .. schedule job with trigger

Trigger t2 = newTrigger()
    .withIdentity("myTrigger2")
    .forJob("myJob2")
    .withSchedule(dailyAtHourAndMinute(11, 30)) // execute job daily at 11:30
    .modifiedByCalendar("myHolidays") // but not on holidays
    .build();

// .. schedule job with trigger2
```
接下来的几个章节将介绍触发器的构建/构建细节。现在，只要明确上面的代码会创建两个触发器，每个触发器都计划每天触发一次。但是，在日历（Calendar）排除的期间内发生的任何触发都将被跳过。

请参阅org.quartz.impl.calendar包，了解适合你的Calendar实现。

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-04.html
