---
layout: post
title: 大数据批量处理系统总结
description: 调研和总结了Hadoop和Spark的基本概念
category: 大数据
tags: [分布式系统,批量处理,Hadoop,Spark]
---

## 批量数据处理技术

批量数据处理技术实现了大规模静态数据的高吞吐处理，主要针对静态数据集，以吞吐量大为显著特征，其典型的技术是分布式编程框架MapReduce，被广泛应用于数据抽取、转换、查询、统计等数据处理中。在MapReduce 技术之上，还产生了Pig, SCOPE, Hive和Tenzing等类SQL 的高级查询语言和系统，用于编写数据查询应用。

<!--more-->

## Hadoop（MapReduce）

Hadoop是一个分布式系统架构，由Apache基金会所开发，其核心主要包括两个组件：HDFS和MapReduce，前者为海量存储提供了存储，而后者为海量的数据提供了计算。这里我们主要关注MapReduce。以下资料来源于Hadoop的官方说明文档和论文。

Hadoop Map/Reduce是一个使用简易的软件框架，基于它写出来的应用程序能够运行在由上千个商用机器组成的大型集群上，并以一种可靠容错的方式并行处理上T级别的数据集。

一个Map/Reduce 作业（job） 通常会把输入的数据集切分为若干独立的数据块，由 map任务（task）以完全并行的方式处理它们。框架会对map的输出先进行排序， 然后把结果输入给reduce任务。通常作业的输入和输出都会被存储在文件系统中。 整个框架负责任务的调度和监控，以及重新执行已经失败的任务。

通常，Map/Reduce框架和分布式文件系统是运行在一组相同的节点上的，也就是说，计算节点和存储节点通常在一起。这种配置允许框架在那些已经存好数据的节点上高效地调度任务，这可以使整个集群的网络带宽被非常高效地利用。

Map/Reduce框架由一个单独的master JobTracker 和每个集群节点一个slave TaskTracker共同组成。master负责调度构成一个作业的所有任务，这些任务分布在不同的slave上，master监控它们的执行，重新执行已经失败的任务。而slave仅负责执行由master指派的任务。

应用程序至少应该指明输入/输出的位置（路径），并通过实现合适的接口或抽象类提供map和reduce函数。再加上其他作业的参数，就构成了作业配置（job configuration）。然后，Hadoop的 job client提交作业（jar包/可执行程序等）和配置信息给JobTracker，后者负责分发这些软件和配置信息给slave、调度任务并监控它们的执行，同时提供状态和诊断信息给job-client。

Map/Reduce框架运转在<key, value> 键值对上，也就是说， 框架把作业的输入看为是一组<key, value> 键值对，同样也产出一组 <key, value> 键值对做为作业的输出，这两组键值对的类型可能不同。

应用程序通常会通过提供map和reduce来实现 Mapper和Reducer接口，它们组成作业的核心。map函数接受一个键值对（key-value pair），产生一组中间键值对。MapReduce框架会将map函数产生的中间键值对里键相同的值传递给一个reduce函数。reduce函数接受一个键，以及相关的一组值，将这组值进行合并产生一组规模更小的值（通常只有一个或零个值）。如图1所示，MapReduce的工作流程中，一切都是从最上方的user program开始的，user program链接了MapReduce库，实现了最基本的Map函数和Reduce函数。图中执行的顺序都用数字标记了。

![hadoop](/images/batchComputing/hadoop.jpg)

<center>图1 MapReduce执行流程</center>

1. MapReduce库先把user program的输入文件划分为M份（M为用户定义），每一份通常有16MB到64MB，如图左方所示分成了split0~4；然后使用fork将用户进程拷贝到集群内其它机器上。
2. user program的副本中有一个称为master，其余称为worker，master是负责调度的，为空闲worker分配作业（Map作业或者Reduce作业），worker的数量也是可以由用户指定的。
3. 被分配了Map作业的worker，开始读取对应分片的输入数据，Map作业数量是由M决定的，和split一一对应；Map作业从输入数据中抽取出键值对，每一个键值对都作为参数传递给map函数，map函数产生的中间键值对被缓存在内存中。
4. 缓存的中间键值对会被定期写入本地磁盘，而且被分为R个区，R的大小是由用户定义的，将来每个区会对应一个Reduce作业；这些中间键值对的位置会被通报给master，master负责将信息转发给Reduce worker。
5. master通知分配了Reduce作业的worker它负责的分区在什么位置（肯定不止一个地方，每个Map作业产生的中间键值对都可能映射到所有R个不同分区），当Reduce worker把所有它负责的中间键值对都读过来后，先对它们进行排序，使得相同键的键值对聚集在一起。因为不同的键可能会映射到同一个分区也就是同一个Reduce作业（谁让分区少呢），所以排序是必须的。
6. reduce worker遍历排序后的中间键值对，对于每个唯一的键，都将键与关联的值传递给reduce函数，reduce函数产生的输出会添加到这个分区的输出文件中。
7. 当所有的Map和Reduce作业都完成了，master唤醒正版的user program，MapReduce函数调用返回user program的代码。

所有执行完毕后，MapReduce输出放在了R个分区的输出文件中（分别对应一个Reduce作业）。用户通常并不需要合并这R个文件，而是将其作为输入交给另一个MapReduce程序处理。

## Spark

Spark是UC Berkeley AMP lab所开源的类Hadoop MapReduce的通用的并行计算框架，Spark基于map reduce算法实现的分布式计算，拥有Hadoop MapReduce所具有的优点；但不同于MapReduce的是Job中间输出结果可以保存在内存中，从而不再需要读写HDFS，因此Spark能更好地适用于数据挖掘与机器学习等需要迭代的map reduce的算法。

Spark中一个主要的结构是RDD（Resilient Distributed Datasets），这是一种只读的数据划分，并且可以在丢失之后重建。它利用了一种血统（Lineage）的概念实现容错，如果一个RDD丢失了那么有足够的信息支持如何从其他RDD重建它。RDD可以被认为是提供了一种高度限制（只读、只能由别的RDD变换而来）的共享内存，但是这些限制可以使得自动容错的开支变得很低。RDD使用 “血统”的容错机制，即每一个RDD都包含关于它是如何从其他RDD变换过来的以及如何重建某一块数据的信息。RDD仅支持粗颗粒度变换(coarse-grained transformation)，即仅记录在单个块上执行的单个操作，然后创建某个RDD的变换序列存储下来，当数据丢失时，我们可以用变换序列（血统）来重新计算，恢复丢失的数据，以达到容错的目的。在RDD的变换序列变得很长的时候，建立一些数据检查点以加快容错的速度。RDD不适合大块的写，但是更高效和容错，且没有检查的开销，因为可以借助血缘恢复，且丢失的分区在重新计算的时候，也是在不同节点上并行的，比回滚整个程序快很多。对于大块数据的操作，RDD会调度到本地数据执行，提高效率。数据太大就存硬盘。RDD适合批处理，具体是对数据集里所有元素进行同一个操作，这样他执行转换和恢复就很快，原因是不记录大量log。但是RDD不适合异步细粒度更新，比如web引用的存储和增量wen爬虫。

Spark 中的应用程序称为驱动程序，这些驱动程序可实现在单一节点上执行的操作或在一组节点上并行执行的操作。驱动程序可以在数据集上执行两种类型的操作：动作和转换。动作 会在数据集上执行一个计算，并向驱动程序返回一个值；而转换 会从现有数据集中创建一个新的数据集。动作的示例包括执行一个 Reduce 操作（使用函数）以及在数据集上进行迭代（在每个元素上运行一个函数，类似于 Map 操作）。转换示例包括 Map 操作和 Cache 操作（它请求新的数据集存储在内存中）。

与 Hadoop 类似，Spark 支持单节点集群或多节点集群。对于多节点操作，Spark 依赖于 Mesos 集群管理器。Mesos 为分布式应用程序的资源共享和隔离提供了一个有效平台（参见 图2）。该设置充许 Spark 与 Hadoop 共存于节点的一个共享池中。

![spark](/images/batchComputing/spark.jpg)

<center>图2 Spark 依赖于 Mesos 集群管理器实现资源共享和隔离</center>

### Spark Streaming

Spark Streaming（D-Streaming）: 构建在Spark上处理Stream数据的框架，基本的原理是将Stream数据分成小的时间片断（几秒），以类似batch批量处理的方式来处理这小部分数据。Spark Streaming构建在Spark上，一方面是因为Spark的低延迟执行引擎（100ms+）可以用于实时计算，另一方面相比基于Record的其它处理框架（如Storm），RDD数据集更容易做高效的容错处理。此外小批量处理的方式使得它可以同时兼容批量和实时数据处理的逻辑和算法。方便了一些需要历史数据和实时数据联合分析的特定应用场合。

### Spark处理体系的优劣评价

优势：

1. Spark对小数据集能达到亚秒级的延迟，这对于Hadoop MapReduce（以下简称MapReduce）是无法想象的（由于“心跳”间隔机制，仅任务启动就有数秒的延迟）。
2. 就大数据集而言，对典型的迭代机器 学习、即席查询（ad-hoc query）、图计算等应用，Spark版本比基于MapReduce、Hive和Pregel的实现快上十倍到百倍。
3. Spark的突破在于，在保证容错的前提下，用内存来承载工作集。Spark的容错（传统上有两种方法：日志和检查点）采用日志数据更新，Spark记录的是粗粒度的RDD更新，这样开销可以忽略不计。鉴于Spark的函数式语义和幂等特性，通过重放日志更新来容错，也不会有副作用。
4. Spark框架的高效和低延迟保证了Spark Streaming操作的准实时性。
5. 利用Spark框架提供的丰富API和高灵活性，可以精简地写出较为复杂的算法。编程模型的高度一致使得上手Spark Streaming相当容易，同时也可以保证业务逻辑在实时处理和批处理上的复用。 
6. Spark提供的数据集操作类型有很多种，不像Hadoop只提供了Map和Reduce两种操作。比如map, filter, flatMap,sample, groupByKey, reduceByKey, union, join, cogroup, mapValues, sort,partionBy等多种操作类型，他们把这些操作称为Transformations。同时还提供Count, collect, reduce, lookup, save等多种actions。这些多种多样的数据集操作类型，给上层应用者提供了方便。各个处理节点之间的通信模型不再像Hadoop那样就是唯一的Data Shuffle一种模式。用户可以命名，物化，控制中间结果的分区等。可以说编程模型比Hadoop更灵活。
7. Graphx。早在0.5版本，Spark就带了一个小型的Bagel模块，提供了类似Pregel的功能。到0.8版本时，鉴于业界对分布式图计算的需求日益见涨，Spark开始独立一个分支Graphx-Branch，作为独立的图计算模块，借鉴GraphLab，开始设计开发GraphX。1.0版本，GraphX正式投入生产使用。
图存储模式：点分割；
图计算模型：BSP，GAS。
特点：离线，批量，同步
8. spark streaming 增量式计算

劣势（不适宜的场景）：

1. Spark首先是一种粗粒度数据并行（数据并行跟任务并行）的计算范式。决定了 Spark无法完美支持细粒度、异步更新的操作。图计算就有此类操作，所以此时Spark不如GraphLab（一个大规模图计算框架）。还有一些应用， 需要细粒度的日志更新和数据检查点，它也不如RAMCloud（斯坦福的内存存储和计算研究项目）和Percolator（Google增量计算技术）。例如web服务的存储或者是增量的web爬虫和索引。就是对于那种增量修改的应用模型，当然不适合把大量数据拿到内存中了。增量改动完了，也就不用了，不需要迭代了。
2. 对于目前版本的Spark Streaming而言，其最小的Batch Size的选取在0.5~2秒钟之间（Storm目前最小的延迟是100ms左右），所以Spark Streaming能够满足除对实时性要求非常高（如高频实时交易）之外的所有流式准实时计算场景。
3.  对于内存的过度消耗
4. 如何根据不同应用确定一个更好的时间间隔？Spark Streaming会把batch窗口内接收到的所有数据存放在Spark内部的可用内存区域中，因此必须确保当前节点Spark的可用内存至少能够容纳这个batch窗口内所有的数据，否则必须增加新的资源以提高集群的处理能力。
5. 设置合理的batch窗口。在Spark Streaming中，Job之间有可能存在着依赖关系，后面的Job必须确保前面的Job执行结束后才能提交。若前面的Job执行时间超出了设置的batch窗口，那么后面的Job就无法按时提交，这样就会进一步拖延接下来的Job，造成后续Job的阻塞。因此，设置一个合理的batch窗口确保Job能够在这个batch窗口中结束是必须的。 
6. 由于Spark目前只是在UC Berkeley的一个研究项目，目前看到的最大规模也就200台机器，没有像Hadoop那样的部署规模，所以，在大规模使用的时候还是要慎重考虑的。
7. 容错做checkpoint的两种方式，一个是checkpoint data，一个是logging the updates。貌似Spark采用了后者。虽然后者看似节省存储空间。但是由于数据处理模型是类似DAG的操作过程，由于图中的某个节点出错，由于lineage chains的依赖复杂性，可能会引起全部计算节点的重新计算，这样成本也不低。是存数据，还是存更新日志，做checkpoint还是由用户说了算，把这个皮球踢给了用户。所以是由用户根据业务类型，衡量是存储数据IO和磁盘空间的代价和重新计算的代价，选择代价较小的一种策略。


