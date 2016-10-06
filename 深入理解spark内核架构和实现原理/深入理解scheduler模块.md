# 深入理解 Scheduler 模块
## Scheduler 模块主要包括三大部分，1.DAGScheduler，2.TaskScheduler，3.SchedulerBackend
### Scheduler 模块概述
   * DAGScheduler 是高层调度器，根据用户提交应用程序的 rdd 的依赖关系生成 DAG，并根据依赖关系将 DAG 划分为不同的 Stage，划分的标准是以宽依赖为节点的。DAGScheduler 会为每一个 stage 生成一批 task（taskSet）。task 生成结束后会被 DAGScheduler 提交到 TaskScheduler。DAGScheduler 在不同的资源管理框架下实现是相同的。
   * SchedulerBackend 是一个 trait，作用是为 task 分配资源，并启动 task。它使用 reviveOffers 完成上述调度过程。CoarseGrainedSchedulerBackend 是 SchedulerBackend 的一个重要实现。
   * TaskScheduler 是底层调度器，也是一个 trait，作用是接受不同 stage 的任务，并向集群提交这些任务。并为执行特别慢的任务启动备份任务。当前 TaskSchedulerImpl 是其唯一的实现。 TaskSchedulerImpl 通过调用 SchedulerBackend 的 reviveOffers 完成任务调度。每个 TaskScheduler 都唯一对应一个 SchedulerBackend。
   
### Scheduler 之 DAGScheduler 实现详解
   * DAGscheduler 的创建。 DAGscheduler 和 TaskScheduler 都是在 SparkContext 创建的时候创建的。其中，TaskScheduler 是通过 SparkContext 的 createTaskScheduler 方法创建的。而 DAGScheduler 是通过直接调用自己的构造函数创建的 。但是创建 DAGScheduler 的时候需要 TaskScheduler 作为参数，因此需要在 TaskScheduler 创建之后才能创建 DAGScheduler。这个构造函数除了需要 SparkContext 和 TaskScheduler 作为构造参数之外还需要 MapOutputTrackerMaster 和 BlockManagerMaster 作为参数。同时 DAGScheduler 还会创建一个 eventProcessActor，这个 Actor 主要负责调用 DAGScheduler 的方法来处理 DAGScheduler 发送给它的各种消息。eventProcessActor 是由 DAGSchedulerActorSupervisor 负责创建的，这样 DAGSchedulerActorSupervisor 能够监控 eventProcessActor 运行状态。如果 eventProcessActor 出现错误，DAGSchedulerActorSupervisor 会取消所有 job，停止 SparkContext，退出程序。
   * Job 的提交。用户提交的应用程序中的每一个 action 都会触发一个 job，并调用 spark contex 中的 runJob 方法，这些 runJob 方法只是参数不同，最终都会调用 DAGScheduler.runJob 方法。 以 count 为例，调用过程如下：count -> SparkContext.runJob -> DAGScheduler.runJob -> DAGScheduler.submitJob(向 eventProcessActor 发送 jobSubmit 消息) -> DAGSchedulerEventProcessActor.receive(JobSubmit) -> DAGScheduler.handleJobSubmit。在这个过程中，submitJob 方法首先会为这个 job 生成一个 jobId，并生成一个 JobWaiter 实例来监听这个job 的执行情况。当这个 job 的所有 stage 的所有 task 都成功完成，这个 job 才被标记为成功。
   * stage 的划分和提交。
       - stage 的划分。上述调用栈的最后一步 handleJobSubmit 会开始 stage 的划分。handleJobSubmit 通过调用 DAGScheduler.newStage 方法来创建 finalStage，在创建 finalStage 时会递归的创建当前 Stage 的 parentStage，然后创建当前 Stage。parentStage 的创建是通过 getParentStage 方法来完成的，getParentStage 是整个 Stage 划分的核心。
       - stage的提交。当 finalStage 创建后，handleJobSubmit 会调用 submitStage 来提交 finalStage，如果它的某些 parentStage 还没有提交，那么它会递归的提交那些还未提交的 parentStage，只有所有的 parentStage 都计算完成后，才提交 finalStage。可以看出 stage 的划分是从后向前逆向划分，stage 的提交是从前向后正向提交。最后通过 DAGscheduler 的 submitMissingTasks 向 TaskScheduler 提交 Task，这个过程首先会判断出哪些 partition 需要计算，然后为这些 partition 生成 Task 并将这些 Task 封装到 TaskSet 中提交给 TaskScheduler，到此为止 DAGscheduler 的工作就结束了。  
                
### Scheduler 之 TaskScheduler 实现详解。 
   * TaskScheduler 的创建和 task 的提交。
       - TaskScheduler 的创建。TaskScheduler 是在 SparkContext 创建的时候创建的。其中，TaskScheduler 是通过 SparkContext 的 createTaskScheduler 方法创建的。每个 TaskScheduler 都唯一对应一个 SchedulerBackend。其中 TaskScheduler 是整个 Application 不同 job 之间的调度（在 Task 执行失败时启动重试机制，为慢任务启动备份任务）。SchedulerBackend 负责与 ClusterManager 进行交互取得 Application 所需要的资源，并将这些资源传递给 TaskScheduler，由 TaskScheduler 最终为 Task 分配计算资源。
       - task 的提交：task 的提交是由 TaskSchedulerImpl 的 submitTasks 开始的。调用栈如下：TaskSchedulerImpl.submitTasks -> SchedulerBuilder.addTaskSetManager -> CoarseGrainedSchedulerBackend.reviveOffers -> DriverActor.makeOffers -> TaskSchedulerImpl.resourceOffers -> DriverActor.launchTasks -> receiveWithLogging.launchTask -> Executor.launchTasks。上述调用栈中有几个关键调用，调用栈一：TaskSchedulerImpl.submitTasks 主要是将 TaskSet 加入到 TaskSetManager 中，让 TaskSetManager 管理 TaskSet。TaskSetManager 会根据数据的就近原则为 task 分配计算资源并监控 task 的执行状态（失败任务重试，慢任务推测）。调用栈二：SchedulerBuilder.addTaskSetManager 是 Application 级别的调度器，目前支持两种调度策略，FIFO，FAIR，调度策略可以通过 spark.scheduler.mode 设置。默认是 FIFO。调用栈五：TaskSchedulerImpl.resourceOffers 是为每个 task 具体分配资源的，为了避免将 task 集中分配到某些机器上，会随机打散这些 task。调用栈六：DriverActor.launchTasks 是将分配好计算资源的 task 发送到 executor 中去，executor 在收到这个消息后启动 task 并执行 task。其中 1-6 是在 driver 端，7-8 是在 executor 端。
   * 两种调度模式详解
       - FIFO
       - FAIR
   * Task 运算结果处理
       - task 在 executor 端运行结束后，executor 端会向 driver 端发送一条 StateUpdate 的消息来通知 driver 当前 task 的执行状态更新为 FINISHED，driver 在收到这条消息后会通知 taskScheduler，taskScheduler 会在这个 executor 上重新分配计算任务。一个 task 的状态只有是 TaskState.FINISHED 才能被标记为执行成功，其余状态（KILLED，FAILED，LOST）都是执行失败。接下来，Executor 会将结果回传给 Driver ，根据结果大小使用不同的回传策略，如果结果大于 1GB 直接丢弃，但是可以通过 spark.driver.maxResultSize 进行设置。如果不大直接回传给 driver，这里的回传是通过 akka 的消息传递机制完成的。akka 消息传递机制的最大值默认是 10MB，但是可以通过 spark.akka.frameSize 进行设置。
       - 处理任务成功执行的机制
       - 处理任务失败的容错机制