# 第六章：CronTrigger
# 内容
CronTrigger通常比SimpleTrigger更有用，如果你需要一个基于类似日历的概念重复出现的工作调度计划，而不是SimpleTrigger的精确指定时间间隔。使用CronTrigger，你可以指定任务触发的时间表，例如“每周五中午”或“每个工作日和上午9:30”，甚至“每周一至周五上午9:00至10点之间每5分钟”和1月份的星期五”。即使如此，和SimpleTrigger一样，CronTrigger有一个startTime，它指定何时生效，以及一个（可选的）endTime，用于指定何时停止任务调度。

## Cron表达式
Cron-Expressions用于配置CronTrigger的实例。Cron表达式是由七个子表达式组成的字符串，用于描述调度计划的各个细节。这些子表达式用空格分隔，各个部分表示如下：
- 1.Seconds
- 2.Minutes
- 3.Hours
- 4.Day-of-Month
- 5.Month
- 6.Day-of-Week
- 7.Year (optional field -> 可选的)

一个完整的Cron-Expressions的例子是字符串"0 0 12 ? * WED" - 这意味着“每个星期三下午12:00”。单个子表达式可以包含范围值、"/"或列表。例如，可以用"MON-FRI"、"MON,WED,FRI"或甚至"MON-WED,SAT"代替前一个（例如"WED"）示例中的星期几字段。

通配符（'*'，在官方文档中是''，估计是官方文档有误，cron不支持空字符串）可用于说明该字段可以取任何值。因此，前一个例子的Month字段中的'*'字符仅仅是“每个月”。因此，Day-of-Month字段中的'*'显然意味着“每周的每一天”。

所有字段都有一组可以指定的有效值。这些值应该是相当明显的 - 例如Seconds和Minutes只允许数字0到59，Hours只允许数字0到23。Day-of-Month可以是1-31的任何值，但是你需要注意在给定的月份中有多少天！Month可以指定为0到11之间的值，或者使用字符串JAN，FEB，MAR，APR，MAY，JUN，JUL，AUG，SEP，OCT，NOV和DEC。Day-of-Week可以指定为1到7（1 = 星期日）之间的值，或者使用字符串SUN，MON，TUE，WED，THU，FRI和SAT。

'/'字符可用于指定值的增量。例如，如果在Minutes字段中输入"0/15"，则表示“从第0分钟开始开始，每隔15分钟”。如果你在Minutes字段中使用"3/20"，则意味着“从第3分钟开始，每隔20分钟” - 换句话说，它与Minutes字段中定义的"3,243,43"意义相同。请注意"/35"的细微之处并不代表“每35分钟” - 这意味着“从第0分钟开始，每隔35分钟” - 或者换句话说，与指定"0,35"相同。

'?'字符只允许使用在Day-of-Month和Day-of-Week字段中。用于表示“没有特定的值”。当您需要在Day-of-Month和Day-of-Week两个字段之一定义其中一个字段的确切值，那么另一个字段可以用'?'，这点十分有用。请参阅下面的示例（和CronTrigger JavaDoc）以进行说明。

'L'字符只允许用于Day-of-Month和Day-of-Week字段中。这个字符其实就是'last'的缩写，但是在Day-of-Month和Day-of-Week各自的字段中有不同的含义。例如，Day-of-Month字段中的"L"表示“月的最后一天” - 例如1月31日，非闰年2月28日。如果在Day-of-Week字段中使用它，它只是意味着"7"或"SAT"。但是在使用了Day-of-Month前提下，在Day-of-Week中使用'L'，就意味着“xxx月的最后一个星期xxx”，例如"6L"或"FRIL"都意味着“月的最后一个星期五”。你还可以指定月份最后一天的偏移量，例如"L-3"，这意味着一个月份的倒数第三天。当使用'L'选项时，切记不要指定列表或范围值，因为你会得到混乱或者意外的结果。

'W'用于指定给定日期最相近的工作日（星期一至星期五）。例如，如果将"15W"指定为Day-of-Month字段的值，则意思是：“距离本月15日最近的工作日”。

'#'用于指定月份的“第n个”星期XXX，格式是'n#p'，表示月的第p个星期n。例如，Day-of-Week字段中的"6＃3"或"FRI＃3"的值表示“月的第三个星期五”。

以下是一些表达式及其含义的示例 - 你可以在JavaDoc的org.quartz.CronExpression中找到更多的资料。

译者注：这里有疑惑的就是'*'和'?'的区别，总结如下：
- 问号(?)的作用是指明该字段“没有特定的值”，星号(*)指明该字段“代表所有可能值”。
- 星号(*)和其它值，比如数字，都是给该字段指明特定的值，只不过用星号(*)代表所有可能值。
- Cron-Expression对日期和星期字段的处理规则是它们必须互斥，即只能且必须有一个字段有特定的值，另一个字段必须是“没有特定的值”。
- 问号(?)就是用来对日期和星期字段做互斥的。

## Cron表达式示例
CronTrigger示例1 - 创建一个触发器的表达式，每5分钟就会触发一次

"0 0/5 * * * ?"

CronTrigger示例2 - 创建一个触发器的表达式，每5分钟触发一次，分钟后10秒（即上午10时10分，上午10:05:10等）。

"10 0/5 * * * ?"

CronTrigger示例3 -  创建一个触发器的表达式，在每个星期三和星期五的10:30，11:30，12:30和13:30创建触发器的表达式。

"0 30 10-13 ? * WED,FRI"

CronTrigger示例4 -  创建一个触发器的表达式，每个月5日和20日上午8点至10点之间每半小时触发一次。请注意，触发器将不会在上午10点开始，仅在8:00，8:30，9:00和9:30

"0 0/30 8-9 5,20 * ?"

请注意，一些调度要求太复杂，无法用单一触发表示 - 例如“每上午9:00至10:00之间每5分钟，下午1:00至晚上10点之间每20分钟”一次。在这种情况下的解决方案是简单地创建两个触发器，并用它们来触发相同的Job。

## 构建CronTrigger
CronTrigger实例使用TriggerBuilder（用于设置触发器的主要属性）和CronScheduleBuilder（对于CronTrigger特定的属性）构建。要以DSL风格使用这些构建器，请使用静态导入：
```java
import static org.quartz.TriggerBuilder.*;
import static org.quartz.CronScheduleBuilder.*;
import static org.quartz.DateBuilder.*:
```
**建立一个触发器，每天上午8点至下午5点之间每隔一分钟触发一次：**
```java
  trigger = newTrigger()
    .withIdentity("trigger3", "group1")
    .withSchedule(cronSchedule("0 0/2 8-17 * * ?"))
    .forJob("myJob", "group1")
    .build();
```
**建立一个触发器，将在上午10:42每天触发一次：**
```java
 trigger = newTrigger()
    .withIdentity("trigger3", "group1")
    .withSchedule(dailyAtHourAndMinute(10, 42))
    .forJob(myJobKey)
    .build();
```
或者
```java
  trigger = newTrigger()
    .withIdentity("trigger3", "group1")
    .withSchedule(cronSchedule("0 42 10 * * ?"))
    .forJob(myJobKey)
    .build();
```
**建立一个触发器，将在星期三上午10:42在TimeZone（系统默认值）之外触发：**
```java
  trigger = newTrigger()
    .withIdentity("trigger3", "group1")
    .withSchedule(weeklyOnDayAndHourAndMinute(DateBuilder.WEDNESDAY, 10, 42))
    .forJob(myJobKey)
    .inTimeZone(TimeZone.getTimeZone("America/Los_Angeles"))
    .build();
```
或者
```java
  trigger = newTrigger()
    .withIdentity("trigger3", "group1")
    .withSchedule(cronSchedule("0 42 10 ? * WED"))
    .inTimeZone(TimeZone.getTimeZone("America/Los_Angeles"))
    .forJob(myJobKey)
    .build();
```

## CronTrigger Misfire说明
以下错过触发配置可以用于通知Quartz当CronTrigger发生错失触发时应该做什么。（Misfire策略的介绍可以参考[第四章：关于Trigger的更多细节](tutorials/lesson-4.md)）。这些策略以常量的形式在CronTrigger接口中定义（常量上包括描述其行为的JavaDoc）。这些常量包括：
```java
MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
MISFIRE_INSTRUCTION_DO_NOTHING
MISFIRE_INSTRUCTION_FIRE_NOW
```
所有触发器都可以使用Trigger.MISFIRE_INSTRUCTION_SMART_POLICY错过触发策略，它就是所有触发器默认的错失触发策略。“SMART POLICY”策略在使用了CronTrigger的情况下解释为MISFIRE_INSTRUCTION_FIRE_NOW。`CronTrigger.updateAfterMisfire()`方法的JavaDoc解释了此行为的确切细节。

在构建CronTriggers时，你可以将misfire指令指定为cron schedule(cron 调度)的一部分（通过CronSchedulerBuilder）：
```java
trigger = newTrigger()
    .withIdentity("trigger3", "group1")
    .withSchedule(cronSchedule("0 0/2 8-17 * * ?")
        ..withMisfireHandlingInstructionFireAndProceed())
    .forJob("myJob", "group1")
    .build();
```

原文链接：http://www.quartz-scheduler.org/documentation/quartz-2.2.x/tutorials/tutorial-lesson-06.html 
