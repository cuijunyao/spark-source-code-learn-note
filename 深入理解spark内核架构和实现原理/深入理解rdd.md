# 深入解析 RDD
## rdd 的五大特征 
 * Internally, each RDD is characterized by five main properties:
   - A list of partitions
   - A function for computing each split
   - A list of dependencies on other RDDs
   - Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
   - Optionally, a list of preferred locations to compute each split on (e.g. block locations for
     an HDFS file)
     
 * 对以上源码做如下分析
    - 每个 rdd 由多个 partition 组成，每个 partition 会被 blockManager 映射成一个 block，block 会被一个 task 负责计算。partition 的个数决定了计算任务的并行度，用户在创建 rdd 时可以指定分片的个数，如果不指定就会采用默认值，默认值就是程序所分配到的 cpu 的个数。因此，用户可以通过控制 partition 的个数和程序分配的 cpu 的个数（一般 partition 的个数是 cpu 个数的三倍左右能达到较好的并行度）来提高并行度以做到性能调优。可以通过配置参数 `--conf spark.executor.instances` 和 `--conf spark.executor.cores` 控制程序分配的 cpu 个数，通过 `--conf spark.default.parallelism` 和 repartition 控制程序的并行度
   - 一个操作函数会作用于 rdd的 所有 partition。一个stage中的多个算子（map，filter，map，filter）可以看成一个算子，以 pipline 形式执行，每次计算一个算子时会对前面的算子重新计算一遍，不会存储中间结果。这种做法大量节省了内存，但降低了计算速度，可以通过 cacch 方法，将某一算子的结果缓存，这样在后续计算的过程中不会对该算子及其以前的算子做重复运算，而是直接从内存中读取结果，以达到性能调优的目的。
   - RDD 之间存在依赖关系。每个子 rdd 都包含如何从对应的父 rdd 转换而来的信息。因此，某个 rdd 的部分分区丢失后可以通过这种依赖关系重新计算得到，而不需要重新计算所有分区。rdd 的这种 lineage 关系是 spark 容错机制的一大保证。
   - 一个 partitioner，即 rdd 的分片函数。当前 spark 中实现了两种类型的分片函数，一个是基于哈希的 HashPartitioner，另外一个是基于范围的 RangePartitioner。只有对 key-value 的 rdd，才会有 partitioner，非 key-value 的 rdd 的partitioner 的值为 None。partitioner 函数不但决定了 rdd 本身的分片数量，也决定了 parent rdd shuffle 输出的分片数量。
   - 一个位置列表，用来存取每个 partition 的优先位置（preferred location）。举例来说，对于一个 HDFS 文件，这个列表保存的就是 partition 所在的块的位置。按着"数据不动代码的理念"，spark 在进行任务调度的时候，尽可能的将 task 分配到其所要处理的块的存储位置。比如:某个 partition 的块的位置在集群中的第十台机器上，那么 spark 在分配计算任务时，会将处理该分区的 task 分配到第十台机器上。
   
## rdd 的创建和转换
* rdd 的创建有两种方式
    - 通过已有的数据集创建，例如 HDFS 上的数据。
    - 通过其他 rdd 创建新的 rdd。
* rdd 中所有转换都是惰性的，他们只是记住这些转换动作并不会真正的开始计算，只有当触发一个 action 时，这些转换动作才被真正执行。
 
## rdd 的缓存和检查点
* 每次计算一个新的 rdd 时都会重新计算一遍之前的 rdd。例如一个 rdd 在两个 job（job 是由 action触发）中同时被使用，那么可以将该 rdd 缓存在内存中，第二个 job 在计算时不会重新计算该 rdd，而是直接从内存中读取结果。缓存的策略可以通过调用 persist() 或 cache() 标记一个要被持久化的 rdd。cache 实际上是调用 persist 的快捷方法，persist 方法中的参数可以指定缓存的级别。
* rdd 的缓存虽然极大的加快了计算速度，但是如果缓存丢失，则需要重新计算。如果是窄依赖只需要计算丢失的分区即可，如果是宽依赖，则需要计算所有的分区，这种情况是比较耗时的。为了避免这种耗时的计算，spark 又引入了检查点机制。缓存机制是在计算完成后直接将结果保存在内存或者磁盘中，检查点机制则不同，它是在计算完成后重新建立一个 job 来计算，并清空之前的 rdd 之间的依赖关系。为了避免重复计算，建议在使用检查点机制时先将 rdd 缓存。

## rdd 的依赖关系
* rdd 的依赖关系包含两个纬度，一个是子 rdd 由哪些父 rdd 转换而来，另一个是子 rdd 的partition 由哪些父 rdd 的 partition 转换而来。spark 将 rdd 的依赖关系分为两种，一种是窄依赖，一种是宽依赖。对于窄依赖而言父 rdd 与子 rdd 是一一对应的，父 rdd 的 partition 与子 rdd 的 partition 也是一一对应的。对于宽依赖并非如此。
* 窄依赖是指父 rdd 的一个 partition 只能被子 rdd 的一个 parttion 使用。
   - map，filter，union 都是窄依赖，某些 join 也是窄依赖（子 rdd 的 partition 只是和已知的，特定的父 rdd 的 partition 进行 join）。对于窄依赖有两种具体实现，一种是一对一的依赖，即 OneToOneDependency，这种依赖子 rdd 的 partition 只依赖父 rdd 中相同 ID 的 partition。另一种是范围的依赖，即RangeDependency，它仅仅在 union 操作中使用。
* 宽依赖是指父 rdd 的一个 partition 被子 rdd 的多个 partition 使用。
   - reduceByKey，groupByKey，某些 join（子 rdd 的 partition 和父 rdd 的所有 partition 进行 join）宽依赖只有一种实现，即 ShuffleDependency。宽依赖支持两种 ShuffleManager，基于 Hash 的 Shuffle 机制（HashShuffleManager）和基于 Sort 的 Shuffle 机制（SortShuffleManager）。
   
## rdd 的 DAG 生成和 Stage 划分
* rdd 的 DAG 生成和 Stage 的划分是由 DAGScheduler 来完成的。DAGScheduler 根据用户提交的应用程序的 rdd 的依赖关系生成 DAG，并根据依赖关系将 DAG 划分为不同的 Stage，划分 Stage 的标准是以宽依赖为节点的。因为，对于窄依赖而言，partition 的转换操作可以在同一个线程里完成，所有窄依赖操作都会被划分到同一个阶段中；对于宽依赖而言，必须等待父 rdd shuffle 完成后才能进行接下来的计算，因此宽依赖就是 spark 划分 stage 的依据。DAGScheduler 会为每一个 stage 生成一批 task（taskSet），这些 task 的计算逻辑完成相同只是在处理不同 partition 的数据。

## rdd 的计算
* rdd 的计算最终是由 task 来完成的。
  - task 简介：spark 的 task 分为两种，一种是 ShuffleMapTask，另外一种是 ResultTask。除了 DAG 的最后一个阶段是 ResultTask，其余所有阶段都是 ShuffleMapTask。这些 task 会被 TaskScheduler 通过 cluster Manager 发送到 executor 中去执行。
  - task 执行的起点：org.apache.spark.scheduler.Task 的 run 方法会调用 ShuffleMapTask 或者 ResultTask 的 runTask 方法，runTask 方法又会调用 org.apache.spark.rdd.RDD 的 iterator 方法，计算由此开始。iterator 方法会首先检查存储级别，如果存储级别不是 None，就检查是否有缓存，如果有则直接读取缓存结果，如果没有则进行计算。如果存储级别是 None，检查是否有 checkPoint，如果有 checkPoint 则直接读取结果，如果没有则进行计算。
  
## spark env 中包含的重要信息

