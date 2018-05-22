# Quartz核心API
# org.quartz.Scheduler
## 接口方法
`String getSchedulerName() throws SchedulerException;`
- 返回调度器的名称。

`String getSchedulerInstanceId() throws SchedulerException;`
- 返回调度器的实例ID。

`SchedulerContext getContext() throws SchedulerException;`
- 返回调度器的上下文对象，类型是org.quartz.chedulerContext。

`void start() throws SchedulerException;`
- 启动调度器的工作线程，然后工作线程才能触发触发器。当调度器被创建之后，它处于"stand-by(准备中，也就是暂时不启动)"模式，stand-by模式下的调度器无法触发触发器。另外可以直接通过standby()方法使调度器变成stand-by模式。注意，如果该调度器实例是首次调用此start()方法，错失触发(misfire)/恢复(recovery)程序将会启动。

## 调度器状态管理方法

`void startDelayed(int seconds) throws SchedulerException;`
- 延迟指定的seconds秒后调用start()方法(此方法调用不会阻塞)。此方法在使用自定义初始化器的应用程序中非常有用，这些应用程序可以在执行作业所需的资源完全初始化之前立即创建调度器。

`boolean isStarted() throws SchedulerException;`
- 检查调度器是否已经启动。注意，这个方法只是反映了调度器实例是否调用了start()方法，所以此方法会返回true即使调度器实例处于stand-by模式或者已经关闭(shutdown)。

`void standby() throws SchedulerException;`
- 暂时停止调度器触发触发器的操作(**这个方法的作用可以理解为暂停调度器实例的调度动作**)。注意，当start()方法被再次调用后(调度器跳出stand-by模式，其实这里的意思是调度器本来已经首次调用了start()方法，然后调用standby()使之处于stand-by模式，现在再次调用start()方法，即调度器暂停->调度器恢复)，触发器的错失触发指令不会再执行 - 任何的错失触发将会(通过JobStore的正常程序逻辑)立即检测出来。

`boolean isInStandbyMode() throws SchedulerException;`
- 检查调度器实例是否处于stand-by模式。

`void shutdown() throws SchedulerException;`
- 终止调度器触发触发器的动作(其实也就是终止调度器实例)，清理所有和调度器实例相关的资源，此方法和shutdown(false)的作用一致。注意，调度器实例一旦调用此方法之后不能重新启动。此方法调用之后，调度器不会等待当前正在执行的作业完成后再终止调度器实例。

`void shutdown(boolean waitForJobsToComplete) throws SchedulerException;`
- 终止调度器触发触发器的动作(**其实也就是终止调度器实例**)，清理所有和调度器实例相关的资源，此方法和shutdown(false)的作用一致。注意，调度器实例一旦调用此方法之后不能重新启动。waitForJobsToComplete此布尔值参数可以控制调度器是否等待当前正在执行的作业完成后再终止调度器实例。

`boolean isShutdown() throws SchedulerException;`
- 检查调度器实例是否已经终止。

`SchedulerMetaData getMetaData() throws SchedulerException;`
- 返回调度器实例的元数据对象，类型是org.quartz.SchedulerMetaData，元数据主要包括配置描述和调度器实例的功能。注意，此返回值只是一个瞬时的快照版本，每次获取到的结果可能是不相同的。

`List<JobExecutionContext> getCurrentlyExecutingJobs() throws SchedulerException;`
- 返回此调度器实例中的所有正在执行的作业执行上下文列表。注意点一：此方法对于集群是无感知的，也就是它的作用范畴仅仅限于当前的调度器实例。注意点二：返回的作业执行上下文列表是瞬时快照版本，每次获取到的结果可能是不相同的。请查阅接口文档来获取和JobExecutionContext相关的资料 - 特别是如果你使用了RMI。

`void setJobFactory(JobFactory factory) throws SchedulerException;`
- 为调度器实例设置JobFactory实例，JobFactory用于创建Job实例。有些使用者希望在他们的应用程序使用JobFactory创建Job实例的时候添加特殊的处理机制 - 例如提供依赖注入的机会。

`ListenerManager getListenerManager()  throws SchedulerException;`
- 返回调度器实例的ListenerManager引用。所有的监听器(JobListener、TriggerListener和SchedulerListener)都是通过ListenerManager进行注册的。

## 调度相关的方法
`Date scheduleJob(JobDetail jobDetail, Trigger trigger) throws SchedulerException;`
- 添加给定的org.quartz.JobDetail实例到调度器实例中，此org.quartz.JobDetail实例和给定的触发器实例关联。如果给定的触发器无法引用任何的作业，那么它会设置为引用此方法所传递的作业(这里的意思是，如果Trigger中不指定JobKey，那么它就会被设置为当前传入的JobDetail中的JobKey，也就是强制关联)。当Job或者Trigger无法添加到调度器或者调度器内部出现异常，会抛出SchedulerException。

`Date scheduleJob(Trigger trigger) throws SchedulerException;`
- 对传入的org.quartz.Trigger实例进行调度，org.quartz.Trigger实例中通过其自身的设置绑定了对应的Job(因为Trigger中具有JobKey属性，由此属性去绑定对应的Job)。当无法绑定到对应的已存在的Job、Trigger实例无法添加到调度器或者调度器内部出现异常，会抛出SchedulerException。

`void scheduleJobs(Map<JobDetail, Set<? extends Trigger>> triggersAndJobs, boolean replace) throws SchedulerException;`
- 对传入的所有给定的Job和Job关联的Trigger进行调度。如果给定的Job和Job关联的Trigger有任何的其中之一已经存在于调度器实例中，并且replace参数没有设置为true，那么就会抛出异常SchedulerException(ObjectAlreadyExistsException)。

`void scheduleJob(JobDetail jobDetail, Set<? extends Trigger> triggersForJob, boolean replace) throws SchedulerException;`
- 功能和上一个方法一样，不过是针对单个Job。

`boolean unscheduleJob(TriggerKey triggerKey) throws SchedulerException;`
- 从调度器中移除指定的触发器。注意，如果和此触发器关联的Job没有其他关联的触发器实例(在Quartz中，一个Job可以关联多个Trigger)，并且Job没有设置为持久化(durable)，那么Job也会被删除(一般来说，我们使用的时候会设置Job的持久化特性，因此此方法只是移除触发器，Job的记录依然是存在的)。

`boolean unscheduleJobs(List<TriggerKey> triggerKeys) throws SchedulerException;`
- 功能和上一个方法一样，不过是针对一个TriggerKey列表。

`Date rescheduleJob(TriggerKey triggerKey, Trigger newTrigger) throws SchedulerException;`
- 移除(删除)通过TriggerKey指定的一个org.quartz.Trigger实例，重新存储一个新的org.quartz.Trigger实例关联到被移除的org.quartz.Trigger实例的Job(新的Trigger实例必须指定被移除的Trigger中关联的Job中相同的name和group)，新Trigger的名称必须和旧的(被移除)Trigger的名称不同。此方法将会返回null如果给定的triggerKey在调度器实例中找不到对应的记录(此时新的Trigger也不会成功添加)；如果设置成功则返回新的Trigger首次触发的时间。

`void addJob(JobDetail jobDetail, boolean replace) throws SchedulerException;`
- 添加指定的Job到调度器中，此Job没有关联任何的Trigger。此Job会处于“休眠状态”直到它被关联到一个触发器实例或者调用Scheduler.triggerJob()进行手动触发。JobDetail中必须定义持久化特性(durability属性设置为true)，否则会抛出SchedulerException异常。replace参数用于控制是否允许覆盖已经存在的同JobKey的Job。

`void addJob(JobDetail jobDetail, boolean replace, boolean storeNonDurableWhileAwaitingScheduling) throws SchedulerException;`
- 功能和上一个方法基本一样，新增了storeNonDurableWhileAwaitingScheduling参数。storeNonDurableWhileAwaitingScheduling参数设置为true，那么一个非持久化的Job就可以添加到调度器中，但是如果它被调度，它将会恢复正常的非持久化的行为(一旦没有Triiger和它关联，也就是它的触发器被移除，它将会被删除)。

`boolean deleteJob(JobKey jobKey) throws SchedulerException;`
- 从调度器中删除通过JobKey指定的Job - 以及所有的和Job关联的Trigger实例。删除成功返回true，失败返回false。

`boolean deleteJobs(List<JobKey> jobKeys) throws SchedulerException;`
- 功能和上一个方法一样，不过是针对JobKey列表，也就是批量删除。

`void triggerJob(JobKey jobKey) throws SchedulerException;`
- 立即触发通过给定JobKey关联的org.quartz.JobDetail实例(客观上表现为org.quartz.JobDetail对应的org.quartz.Job的实例马上被回调)。

`void triggerJob(JobKey jobKey, JobDataMap data) throws SchedulerException;`
- 功能和上一个方法一样，不过多添加了一个JobDataMap参数，此参数会最终传入到org.quartz.Job实例的execute方法中。

`void pauseJob(JobKey jobKey) throws SchedulerException;`
- 暂停通过给定JobKey关联的org.quartz.JobDetail实例 - 实际上就是暂停org.quartz.JobDetail实例的所有关联的Trigger实例。pauseJob和resumeJob相对。

`void pauseJobs(GroupMatcher<JobKey> matcher) throws SchedulerException;`
- 功能和上一个方法基本一致，不过可以通过GroupMatcher实例去构造JobKey匹配规则从而暂停的所有满足规则org.quartz.JobDetail实例，这里不详细展开GroupMatcher的使用方式。

`void resumeJob(JobKey jobKey) throws SchedulerException;`
- 恢复通过给定JobKey关联的org.quartz.JobDetail实例- 实际上就是恢复org.quartz.JobDetail实例的所有关联的Trigger实例。pauseJob和resumeJob相对。注意：如果org.quartz.JobDetail实例关联的任一个Trigger出现一次或者多次错过触发，那么此方法调用后，这些错过触发的Trigger将会应用错过触发策略。

`void resumeJobs(GroupMatcher<JobKey> matcher) throws SchedulerException;`
- 功能和上一个方法基本一致，不过可以通过GroupMatcher实例去构造JobKey匹配规则从而恢复的所有满足规则org.quartz.JobDetail实例，这里不详细展开GroupMatcher的使用方式。

`void pauseTrigger(TriggerKey triggerKey) throws SchedulerException;`
- 暂停通过给定TriggerKey关联的org.quartz.Trigger实例。

`void pauseTriggers(GroupMatcher<TriggerKey> matcher) throws SchedulerException;`
- 功能和上一个方法基本一样，不过可以通过GroupMatcher实例去构造TriggerKey匹配规则从而暂停的所有满足规则org.quartz.Trigger实例，这里不详细展开GroupMatcher的使用方式。

`void resumeTrigger(TriggerKey triggerKey) throws SchedulerException;`
- 恢复通过给定TriggerKey关联的org.quartz.Trigger实例。如果TriggerKey关联的org.quartz.Trigger实例出现一次或者多次错过触发，那么此方法调用后，Trigger实例将会应用错过触发策略。

`void resumeTriggers(GroupMatcher<TriggerKey> matcher) throws SchedulerException;`
- 功能和上一个方法基本一样，不过可以通过GroupMatcher实例去构造TriggerKey匹配规则从而恢复的所有满足规则org.quartz.Trigger实例，这里不详细展开GroupMatcher的使用方式。

`void pauseAll() throws SchedulerException;`
- 暂停所有的Trigger - 和通过Trigger的分组(group)去暂停所有分组的Trigger类似。此方法调用之后，一定要调用resumeAll()方法，否则所有的新添加的Trigger都会处于暂停的状态。也就是调用resumeAll()方法是为了清除调度器中“所有新增的Trigger都设置为暂停”的记忆状态。

`void resumeAll() throws SchedulerException;`
- 恢复所有的Trigger - 和通过Trigger的分组(group)去恢复所有分组的Trigger类似。任一个被恢复的Trigger出现一次或者多次错过触发，那么此方法调用后，这些错过触发的Trigger将会应用错过触发策略。

`List<String> getJobGroupNames() throws SchedulerException;`
- 获取所有的org.quartz.JobDetail实例的分组。

`Set<JobKey> getJobKeys(GroupMatcher<JobKey> matcher) throws SchedulerException;`
- 通过GroupMatcher获取满足匹配规则的JobKey集合。

`List<? extends Trigger> getTriggersOfJob(JobKey jobKey) throws SchedulerException;`
- 通过指定的JobKey获取一个Job关联的所有触发器列表。

`List<String> getTriggerGroupNames() throws SchedulerException;`
- 获取所有的触发器的分组列表。

`Set<TriggerKey> getTriggerKeys(GroupMatcher<TriggerKey> matcher) throws SchedulerException;`
- 通过GroupMatcher获取满足匹配规则的TriggerKey集合。

`Set<String> getPausedTriggerGroups() throws SchedulerException;`
- 获取所有的暂停中的触发器的分组集合。

`JobDetail getJobDetail(JobKey jobKey) throws SchedulerException;`
- 通过指定的JobKey获取对应的org.quartz.JobDetail实例。注意：返回的JobDetail实例只是存储的JobDetail的一个快照版本。如果希望修改JobDetail，你需要重新存储JobDetail(例如通过addJob(JobDetail, boolean)方法，这里可以理解为这个返回值是不可变的，如果想要修改JobDetail则需要重新添加并且覆盖旧值)。

`Trigger getTrigger(TriggerKey triggerKey) throws SchedulerException;`
- 通过指定的TriggerKey获取对应的org.quartz.Trigger实例。注意：返回的Trigger实例只是存储的Trigger的一个快照版本。如果希望修改Trigger，你需要重新存储JTrigger(例如通过rescheduleJob(TriggerKey, Trigger)方法，这里可以理解为这个返回值是不可变的，如果想要修改Trigger则需要重新添加并且移除旧的Trigger)。

`TriggerState getTriggerState(TriggerKey triggerKey) throws SchedulerException;`
- 返回指定的TriggerKey对应的org.quartz.Trigger实例的状态。详细的状态值见org.quartz.Trigger.TriggerState。这个方法比较重要，因为从这个方法可以明确对应的触发器处于什么状态，可以依据此作为监控依据或者健康检查。

`void resetTriggerFromErrorState(TriggerKey triggerKey) throws SchedulerException;`
- 重置通过指定的TriggerKey获取对应的org.quartz.Trigger实例的状态，重置操作为按照适当的情况重置状态值为TriggerState#ERROR->TriggerState#NORMAL或者TriggerState#ERROR->TriggerState#PAUSED。这个方法只会对真正处于TriggerState#ERROR状态的Trigger生效。如果Trigger所在的分组处于暂停模式，那么调用此方法的状态重置为TriggerState#ERROR->TriggerState#PAUSED，否则为TriggerState#ERROR->TriggerState#NORMAL。

`void addCalendar(String calName, Calendar calendar, boolean replace, boolean updateTriggers) throws SchedulerException;`
- 添加(注册)给定的org.quartz.Calendar实例到调度器中。replace参数设置为true的时候，此时添加的Calendar实例会覆盖原有的值(如果存在的话)。updateTriggers参数为true的时候，调度器中已经引用了名称为calName的所有Trigger都会以当前传入的Calendar实例进行一次更新。同名的Calendar在调度器中已存在同时不指定replace为true会抛出SchedulerException异常。

`boolean deleteCalendar(String calName) throws SchedulerException;`
- 通过指定的名称calName从调度器中删除对应的org.quartz.Calendar实例。注意：如果因为org.quartz.Calendar实例的删除导致当前存在的Trigger指向了不存在的Calendar，将会抛出SchedulerException异常。删除成功将会返回true，否则返回false。

`Calendar getCalendar(String calName) throws SchedulerException;`
- 通过指定的名称calName从调度器获取对应的org.quartz.Calendar实例。

`List<String> getCalendarNames() throws SchedulerException;`
- 获取所有已经注册到调度器中的Calendar的名称列表。

`boolean interrupt(JobKey jobKey) throws UnableToInterruptJobException;`
- 在此调度器实例中，通过指定的JobKey对所有正在执行中实现了InterruptableJob接口的Job进行中断(这里为什么说所有?因为在允许作业并发的情况下，同一个Job的实现类也就是同一组JobKey对应的Job实例有可能有多个在并发执行)。如果超过一个匹配到的InterruptableJob实例正在执行，那么每一个InterruptableJob实例都会回调interrupt()方法。注意：如果其中的一个InterruptableJob实例回调interrupt()方法时候直接抛出异常，那么剩下的其他匹配到的InterruptableJob实例将不会回调interrupt()方法。此方法对集群无感知，也就是作用范围仅仅限于当前的调度器实例。

`boolean interrupt(String fireInstanceId) throws UnableToInterruptJobException;`
- 功能和上一个方法大致相同，但是fireInstanceId是通过JobExecutionContext#getFireInstanceId()传递的，这个fireInstanceId属性在org.quartz.spi.OperableTrigger中建立了setter和getter方法，初步来看是和Trigger相关的。

`boolean checkExists(JobKey jobKey) throws SchedulerException;`
- 通过指定的JobKey检查调度器中是否存在对应的Job实例。

`boolean checkExists(TriggerKey triggerKey) throws SchedulerException;`
- 通过指定的TriggerKey检查调度器中是否存在对应的Trigger实例。

`void clear() throws SchedulerException;`
- 清除(删除)调度器实例中的所有调度数据，包括所有的Job实例、Trigger实例和Calendar实例。注意：这个方法一定要谨慎使用，有点“删库跑路”的意味。

# Quartz中的注解
## ExecuteInJTATransaction
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ExecuteInJTATransaction {

    int timeout() default -1;
}
```
应用在org.quartz.Job的实现类，指定作业执行的时候，使用JTA事务。一般建议事务要委托到外部的容器(例如Spring)管理，不建议使用此注解。

## DisallowConcurrentExecution
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface DisallowConcurrentExecution {

}
```
应用在org.quartz.Job的实现类，指定作业执行的时候不能并发执行。也就是说，同一个Job的实现类对应的调度任务实例在同一个调度器实例中只能存在一个。
## PersistJobDataAfterExecution
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface PersistJobDataAfterExecution {

}
```
应用在org.quartz.Job的实现类，每次作业执行完成之后(org.quartz.Job实例的execute方法执行完毕)，调度器会重新存储此作业实例的org.quartz.JobDataMap实例数据，确保下一次调用Job实例的execute方法的时候传入的org.quartz.JobDataMap实例数据是最新的。
