# 第五章：SimpleTrigger
# 内容
SimpleTrigger可以满足的调度需求是：在具体的时间点执行一次，或者在具体的时间点执行并且以指定的间隔重复执行若干次（其实永远重复也可以）。比如，你有一个Trigger，你可以设置它在2015年1月13日的上午11:23:54准时触发，或者在当前这个时间点触发，并且每隔2秒触发一次，一共重复5次。

根据描述，你可能已经发现了，SimpleTrigger的属性包括：开始时间、结束时间、重复次数以及重复的间隔。这些属性的含义与你所期望的是一致的，只是关于结束时间有一些地方需要注意。

重复次数，可以是0、正整数，以及常量SimpleTrigger.REPEAT_INDEFINITELY(实际值为-1，即无限重复执行次数)。重复的间隔，必须是0，或者long型的正数，单位是毫秒。注意，如果重复间隔为0，Trigger将会以重复次数并发执行(或者以Scheduler可以处理的近似并发数并发执行)。

如果你还不熟悉Quartz的DateBuilder类，了解后你会发现使用它可以非常方便地构造基于startTime(或endTime)的调度策略。

endTime属性的值会覆盖设置重复次数的属性值；比如，你可以创建一个Trigger，在终止时间之前每隔10秒执行一次，你不需要去计算在开始时间和终止时间之间的重复次数，只需要设置终止时间并将重复次数设置为REPEAT_INDEFINITELY(当然，你也可以将重复次数设置为一个很大的值，并保证该值比Trigger在终止时间之前实际触发的次数要大即可)。

SimpleTrigger实例通过TriggerBuilder设置主要的属性，通过SimpleScheduleBuilder设置与SimpleTrigger相关的属性。要使用这些builder的静态方法，需要静态导入：
```java
import static org.quartz.TriggerBuilder.*;
import static org.quartz.SimpleScheduleBuilder.*;
import static org.quartz.DateBuilder.*:
```
下面的例子，是基于简单调度(simple schedule)创建的Trigger。建议都看一下，因为每个例子都包含一个不同的实现：

**指定时间开始触发，不重复触发：**
```java
 SimpleTrigger trigger = (SimpleTrigger) newTrigger()
    .withIdentity("trigger1", "group1")
    .startAt(myStartTime) // some Date
    .forJob("job1", "group1") // identify job with name, group strings
    .build();
```
**指定时间触发，每隔10秒触发一次，重复10次：**
```java
 trigger = newTrigger()
        .withIdentity("trigger3", "group1")
        .startAt(myTimeToStartFiring)  // if a start time is not given (if this line were omitted), "now" is implied
        .withSchedule(simpleSchedule()
            .withIntervalInSeconds(10)
            .withRepeatCount(10)) // note that 10 repeats will give a total of 11 firings
        .forJob(myJob) // identify job with handle to its JobDetail itself                   
        .build();
```
**5分钟以后开始触发，仅触发一次：**
```java
    trigger = (SimpleTrigger) newTrigger() 
        .withIdentity("trigger5", "group1")
        .startAt(futureDate(5, IntervalUnit.MINUTE)) // use DateBuilder to create a date in the future
        .forJob(myJobKey) // identify job with its JobKey
        .build();
```
**立即触发，每个5分钟触发一次，直到22:00：**
```java
    trigger = newTrigger()
        .withIdentity("trigger7", "group1")
        .withSchedule(simpleSchedule()
            .withIntervalInMinutes(5)
            .repeatForever())
        .endAt(dateOf(22, 0, 0))
        .build();
```
**建立一个触发器，将在下一个小时的整点触发，然后每2小时重复触发一次：**
```java
  trigger = newTrigger()
        .withIdentity("trigger8") // because group is not specified, "trigger8" will be in the default group
        .startAt(evenHourDate(null)) // get the next even-hour (minutes and seconds zero ("00:00"))
        .withSchedule(simpleSchedule()
            .withIntervalInHours(2)
            .repeatForever())
        // note that in this example, 'forJob(..)' is not called which is valid 
        // if the trigger is passed to the scheduler along with the job  
        .build();

    scheduler.scheduleJob(trigger, job);
```
请查阅TriggerBuilder和SimpleScheduleBuilder提供的方法，以便了解对上述示例中未提到的选项。
```
TriggerBuilder(以及Quartz的其它builder)会为那些没有被显式设置的属性选择合理的默认值。比如：如果你没有调用withIdentity(..)方法，TriggerBuilder会为Trigger生成一个随机的名称；如果没有调用startAt(..)方法，则默认使用当前时间，即Trigger立即生效。
```
## SimpleTrigger Misfire策略
SimpleTrigger有几个misfire相关的策略，告诉quartz当misfire发生的时候应该如何处理。(Misfire策略的介绍可以参考[第四章：关于Trigger的更多细节](tutorials/lesson-4.md))。这些策略以常量的形式在SimpleTrigger中定义(JavaDoc中介绍了它们的功能)。这些策略包括：

**SimpleTrigger的Misfire策略常量：**
```
MISFIRE_INSTRUCTION_SMART_POLICY 
MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
MISFIRE_INSTRUCTION_FIRE_NOW
MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT
```
译者补充：其实这些错过触发策略基本都可以从名称得出它们的实际操作，总结如下：
- MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY：这个不是忽略已经错失的触发的意思，而是说忽略MisFire策略。它会在资源合适的时候，重新触发所有的MisFire的错过触发，并且不会影响现有的调度时间。比如，SimpleTrigger每15秒执行一次，而中间有5分钟时间它都MisFire了，一共错失了20次触发，5分钟后，假设资源充足了，并且任务允许并发，它会被一次性并发触发20次。
- MISFIRE_INSTRUCTION_FIRE_NOW：忽略已经MisFire的触发，并且立即触发一次。这通常只适用于只执行一次的任务。
- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT：将startTime设置当前时间，立即重新触发，包括MisFire的触发。
- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT：将startTime设置当前时间，立即重新触发，不包括MisFire的触发。
- MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT：在下一次调度时间点触发，包括MisFire的的触发。
- MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT：在下一次调度时间点触发，不包括MisFire的的触发。
- MISFIRE_INSTRUCTION_SMART_POLICY：默认策略，大致意思是“把处理逻辑交给聪明的Quartz去决定”。
  - 如果是只执行一次的调度，使用MISFIRE_INSTRUCTION_FIRE_NOW。
  - 如果是无限次的调度(repeatCount是无限的)，使用MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT。
  - 否则，使用MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT。

其实，错过触发的策略相对复杂，可以参考[Quartz Scheduler Misfire Instructions Explained](https://dzone.com/articles/quartz-scheduler-misfire)。

回顾一下，所有的Trigger都有一个Trigger.MISFIRE_INSTRUCTION_SMART_POLICY策略可以使用，该策略也是所有Trigger的默认策略。

如果使用smart policy，SimpleTrigger会根据实例的配置及状态，在所有MISFIRE策略中动态选择一种Misfire策略。可以从SimpleTrigger.updateAfterMisfire()的JavaDoc中解释了该动态行为的具体细节。

在使用SimpleTrigger构造Trigger时，misfire策略作为简单调度(simple schedule)的一部分进行配置(通过SimpleSchedulerBuilder设置)，例子如下：
```java
 trigger = newTrigger()
        .withIdentity("trigger7", "group1")
        .withSchedule(simpleSchedule()
            .withIntervalInMinutes(5)
            .repeatForever()
            .withMisfireHandlingInstructionNextWithExistingCount())
        .build();
```

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-05.html 
