# 深入理解 deploy 模块
## deploy 模块是 cluster 层面的调度
* yarn 架构中的基本术语
   - ResourceManager 是整个集群的资源管理器，管理集群所有的 cpu，磁盘，内存，网络等
   - NodeManager 是节点上的资源管理器，管理节点上所有的 cpu，磁盘，内存，网络等，并向 ResourceManager 汇报每个节点上的资源状态
   - ApplicationMaster 每个应用都有一个 ApplicationMaster，它的职责是向 ResourceManager 申请资源容器，运行任务并监控任务的运行状态
   - Container 是资源容器，一个 container 运行一个进程，默认一个进程拥有一个线程
* yarn 两种模式概述
  - yarn cluster 模式，用户提交的 Application 都会通过 yarn-client 提交到 ResourceManager 上，ResourceManager 会在集群中的某个节点上启动 ApplicationMaster，当 ApplicationMaster 被启动后才算完成整个任务的提交。接下来 ApplicationMaster 会将自己注册成为一个 yarn ApplicationMaster，此时才开始执行用户提交的 Application
  - yarn client 模式，该模式与 yarn cluster 模式的区别在于用户提交的 Application 的 SparkContext 是在本机上运行，这种模式便于查看日志调试 bug，而 yarn cluster 模式的 SparkContext 是在集群中的某个节点上运行，是看不到具体的日志信息的
* deploy 模块概述，deploy 是经典的 master/slaver 架构， 模块主要包含三个子模块：Master、Worker、Client。
  - 各子模块的消息传递
     - Master 和 Worker 通信，work 向 master 发送的消息主要包括三类：1.注册， work 向 master 注册自己 2.状态汇报，汇报 Executor 和 Driver 的运行状态 3.报活心跳，worker 会定期向 master 发送报活的心跳。master 向 worker 发送的消息包括两类：1.响应 worker 的注册 2.给 worker 发送指令，包括让 worker 重新注册，让 worker 重启 executor、driver，kill executor，driver，3.master 的更新
     - Master 和 driver client 通信，driver client 向 master 发送的消息有三类，1.注册，driver client 向 master 注册自己，2.获取当前 driver 的运行状态 3.向 master 发送 kill driver 的请求。master 向 driver client 发送的消息包括两类：1. 回复 driver 是否注册成功，2.回复 driver 当前的运行状态，3. 通知 worker kill driver
     - master 和 app client 通信，app client 向 master 发送的消息包括两类：1.注册 application，2.在 master 故障恢复后，回复 master 已经保存了最新的 master 信息。master 向 app client 发送的消息包括五类：1.回复 app client application 注册成功，2.executor 增加，3.executor 状态更新，4.删除 application，5.master 的更新
     - driver client 和 Executor 通信，executor 向 driver client 发送消息的包括两类：1.向 driver client 注册executor，2.向 driver client 汇报 executor 中 task 的运行状态。driver client向 executor 发送的消息包括两类，1.响应 executor 的注册请求，2.启动 task，kill task
  
* Master 的启动，Worker 的启动，Executor 的启动
     - 有两种情况下 Master 会启动，一种是集群第一次启动，另一种是 master 遇到故障恢复时启动。Master 的启动支持几种选举机制和元数据持久化方式，生产中一般采用 zookeeper 的选举机制。当 master 遇到故障时，zookeeper 会从备选 master 中选出一个 master 作为 leader，新选取的 master 会从 zookeeper 中恢复元数据信息。如果 zookeeper 中没有元数据信息那么 master 的状态会立即变为 ALIVE，如果有元数据信息 master 的状态会变为 RECOVERING，并通知 worker，app client，driver client，master 已经改变，只有在 master 的状态为 ALIVE 时才能对外提供服务(主要是各个模块的注册和监控)。对于已经运行的 application 来说，master 会将其状态更改为 UNKNOWN 并通知 app client，对于 worker，master 也会将其状态更改为 UNKNOWN。在 master 接到 appclient 和 worker 的响应后会将他们的状态重新设置为正常状态，首先 master 会查看所有的 application 和 worker 的状态是否都是正常状态，如果都是正常状态则代表恢复已经结束，如果有部分 appclient 和 worker 长时间都没有回应，master 还是会认为恢复已经结束(调用 completeRecovery)。
     - Worker 的启动只做一件事就是向 master 注册自己，且会向所有的 master 发送注册请求(tryRegisterAllMaster)，如果 master 回复注册失败，worker 会判断是否真的注册失败，如果注册失败了，则直接退出；如果已经注册成功了，则会忽略这个消息。注册的重试机制只有 6 次，为了避免同一时刻所有的 worker 都向 master 发起注册请求，重试是有时间间隔的。
     - Execuor 的启动，参看 executor 模块。
     - superviser 是基于 python 开发的一套进程管理工具，如果服务进程挂掉不是由于宕机造成的，superviser 会重新拉起该进程。
     - executor 连续十次异常退出后，master 会将这个 application 标记为失败，然后退出应用程序。


