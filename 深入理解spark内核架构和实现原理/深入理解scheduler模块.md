## 深入理解 Scheduler 模块
#### Scheduler 模块主要包括三大部分，1.DAGScheduler，2.TaskScheduler，3.SchedulerBackend
* Scheduler 模块概述
   - DAGScheduler 是高层调度器，根据用户提交应用程序的 rdd 的依赖关系生成 DAG，并根据依赖关系将 DAG 划分为不同的 Stage，划分的标准是以宽依赖为节点的。DAGScheduler 会为每一个 stage 生成一批 task（taskSet）。task 生成结束后会被 DAGScheduler 提交到 TaskScheduler。DAGScheduler 在不同的资源管理框架下实现是相同的。
   - SchedulerBackend 是一个 trait，作用是为 task 分配资源，并启动 task。它使用 reviveOffers 完成上述调度过程。CoarseGrainedSchedulerBackend 是 SchedulerBackend 的一个重要实现。
   - TaskScheduler 是底层调度器，也是一个 trait，作用是接受不同 stage 的任务，并向集群提交这些任务。并为执行特别慢的任务启动备份任务。当前 TaskSchedulerImpl 是其唯一的实现。 TaskSchedulerImpl 通过调用 SchedulerBackend 的 reviveOffers 完成任务调度。每个 SchedulerBackend 都会唯一对应一个 TaskScheduler。
* DAGScheduler 实现详解