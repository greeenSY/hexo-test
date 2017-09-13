---
layout: post
title: 大数据流式处理系统总结之三
description: 对于自己调研过或者了解过的大数据流式处理系统进行特点的总结和对比
category: 大数据
tags: [分布式系统,流式处理]
---

整理了之前调研过的大数据流式处理系统，包括S4,Storm,MapRedcue Online,Facebook Data Freeway and Puma,Kafka,TimeStream,Naiad等。

这一篇文章整理了MapRedcue Online,Facebook Data Freeway and Puma,Kafka,TimeStream,Naiad等。

## MapReduce Online

MapReduce Online由加州大学伯克利分校提出，对于mapreduce进行改进以适应流式计算，提出了pipeline的思想：Map产生的中间结果，应该尽快的传输到Reduce端。用户可以在提交job的时候，指定一个步进比例，比如10%。 当Reduce检测到自己收到的Map处理是这个步进比例的倍数时，比如30%，就在收到的这些数据的基础上，进行一次Reduce计算，得到一个中间结果。HOP把这个过程叫做“snapshot”。这种方式没有去考虑Map如何对数据进行取样，也没有考虑数据skew的问题，但是无论如何，”snapshot”提供了一种可能，让用户有可能得到某种形式的反馈。

MapReduce Online重写定义和修改MapReduce框架，使得中间数据可以在操作之间传输，实现流水化作业，从而实现对于实时事件和流处理的需求。同时满足原本的Mapreduce的接口和容错性。

MapReduce Online提供了两种级别的pipeline：

1. Job内pipeline——Map阶段和Reduce阶段之间的pipeline。每一个reduce task都与map task有链接，一旦map输出则发送TCP socket到合适的reduce task，reduce task将接受所有发送来的pipeline数据并存入内存缓冲区，并按照溢出把缓冲区中的内容排序写入磁盘，当所有map task完成后再执行reduce函数。当一个reduce task没有被安排时则相应的map tasks把记录写入磁盘，直到reduce task分配位置后再从中拉取记录，每一个redecer都链接不超过规定数目的mapper，为了防止网络IO对于map操作的影响，mapper的处理和发送放在两个线程中进行，前者写入buffer后者将其发送到制定reducer。如果mapper中的输出产生时就立即发送到reducer中，因此后者需要承担更多的排序任务，同时combiner的操作也无法进行，对于这个问题，mapper的发送线程需要等待buffer中的数据到达一个阈值之后再进行combiner并排序，作为溢出文件写入磁盘，在溢出文件产生时首先要查看TaskTracker中的队列，如果数目超过阈值则不再注册而是积累多个溢出文件，一旦队列中数目减少到阈值一下则合并文件并注册，这个机制在redecer中也是类似的。和原本机制不同之处在于原来map完写入磁盘统一处理，现在是分别发送到各个reducer，reducer不需要空等完整的输出写入磁盘。
2. Job间pipeline，可以把reduce的结果直接传输到下一个阶段的map上以减少HDFS中临时存储的开销，但是两者的计算工作不能重叠，在线聚合和连续查询流水可以产生快照输出，这样实现了jobs间的pipeline。

MapReduce Online的容错，reducer只会合并同一个map task 的结果和所有完成的map task结果，所以出错的map会被发现和重新执行。reduce task失效则运行一个相同的拷贝。考虑一个checkpoint机制，可以在出错时重新执行。

MapReduce Online单作业的在线聚合，通过快照的方式实现交互式操作。快照的取得时刻取决与作业的进程进度，这个进度是根据已经被reduce task流水接受的map task在所有map task中的比例。多作业的在线聚合，在前一个reduce产生快照后，直接传入下一个阶段map开始工作。对于连续处理任务（流处理作业）。Map每产生一个结果则尽快传送到reduce task上，增加flush api来强制当前的map输出的结果发送到reduce tasks。reduce 函数需要被定期调用，其他的参数如调用周期，输入行数等需要被设定。

## Facebook Data Freeway and Puma

Data Freeway and Puma是Facebook支持开发的一款基于Hive/Hadoop的、分布式的、高效率的、数据传输通道和大数据流式计算系统。

Data Freeway系统: Data Freeway是Facebook支持开发的一款可扩展数据流架构(scalable data stream framework)，可以有效地支持4种数据间的传输，即，文件到文件、文件到消息、消息到消息和消息到文件。其系统结构如图1所示，Data Freeway数据流架构由4个组件构成，即，Scribe，Calligraphus，Continuous Copier和PTail。Scribe组件位于用户端，其功能是将用户的数据通过RPC发送到服务器端；Calligraphus组件实现了对日志类型的维护与管理，其功能是通过Zookeeper系统，将位于缓冲区中的数据并发写到HDFS中；Continuous Copier组件的功能是实现在各个HDFS系统间进行文件的迁移；PTail组件实现了并行地将文件输出。

![Data Freeway系](/images/streamComputing/DataFreeway.jpg)

<center>图1 Data Freeway系统架构</center>

Puma系统: Puma是Facebook的可靠数据流聚合引擎(reliable stream aggregation engine)系统，如图2所示，我们分析的版本为Puma3系统。

![Puma3](/images/streamComputing/Puma3.jpg)

<center>图2 Puma3系统架构</center>

Puma3在本地内存中实现了数据聚合功能，极大地提高了数据的计算能力，有效地降低了系统延迟。Puma3系统实现时，在Calligraphus阶段通过聚合主键完成对数据的分片，其中，每个分片都是内存中的哈希表，每个表项对应一个Key及用户定义的聚合方法，如统计、求和、平均值等操作。HBase子系统会定期地从Puma3中将内存中的数据备份到HBase中，进行数据的持久化存储。只有当Puma3发生故障时，才从HBase中读取副本，进行数据的重放，实现对因故障丢失数据的恢复；在无故障的情况下，HBase子系统不参与数据的计算，因此提高了数据的计算能力。

存在不足: Data Freeway and Puma系统存在的不足主要包括:数据延迟在秒级，无法满足大数据流式计算所需要的毫秒级应用需求；将哈希表完全放入内存的加速机制，导致内存需求量大；资源调度策略不够简单、高效，不能灵活适应连续的工作负载。


## Linkedin Kafka

Kafka是Linkedin所支持的一款开源的、分布式的、高吞吐量的发布订阅消息系统，可以有效地处理互联网中活跃的流式数据，如网站的页面浏览量、用户访问频率、访问统计、好友动态等，开发语言是Scala，可以使用Java进行编写。

Kafka系统在设计过程中主要考虑到了以下需求特征:消息持久化是一种常态需求；吞吐量是系统需要满足的首要目标；消息的状态作为订阅者(consumer)存储信息的一部分，在订阅者服务器中进行存储；将发布者(producer)、代理(broker)和订阅者(consumer)显式地分布在多台机器上，构成显式的分布式系统。形成了以下关键特性:在磁盘中实现消息持久化的时间复杂度为O(1)，数据规模可以达到TB级别；实现了数据的高吞吐量，可以满足每秒数十万条消息的处理需求；实现了在服务器集群中进行消息的分片和序列管理；实现了对Hadoop系统的兼容，可以将数据并行地加载到 Hadoop集群中。

系统架构：Kafka消息系统的架构是由发布者(producer)、代理(broker)和订阅者(consumer)共同构成的显式分布式架构，即，分别位于不同的节点上，如图3所示。各部分构成一个完整的逻辑组，并对外界提供服务，各部分间通过消息(message)进行数据传输。其中，发布者可以向一个主题(topic)推送相关消息，订阅者以组为单位，可以关注并拉取自己感兴趣的消息，通过Zookeeper实现对订阅者和代理的全局状态信息的管理，及其负载均衡的实现。

![Kafka](/images/streamComputing/Kafka.jpg)

<center>图3 Kafka系统架构</center>

数据存储：Kafka消息系统通过仅仅进行数据追加的方式实现对磁盘数据的持久化保存，实现了对大数据的稳定存储，并有效地提高了系统的计算能力。通过采用Sendfile系统调用方式优化了网络传输，减少了1次内存拷贝，提高了系统的吞吐量，即使对于普通的硬件，Kafka消息系统也可以支持每秒数十万的消息处理能力。此外，在Kafka消息系统中，通过仅保存订阅者已经计算数据的偏量信息，一方面可以有效地节省数据的存储空间，另一方面，也简化了系统的计算方式，方便了系统的故障恢复。

消息传输：Kafka消息系统采用了推送、拉取相结合的方式进行消息的传输，其中，当发布者需要传输消息时，会主动地推送该消息到相关的代理节点；当订阅者需要访问数据时，其会从代理节点中进行拉取。通常情况下，订阅者可以从代理节点中拉取自己感兴趣的主题消息。

负载均衡：在Kafka消息系统中，发布者和代理节点之间没有负载均衡机制，但可以通过专用的第4层负载均衡器在Kafka代理之上实现基于TCP连接的负载均衡的调整。订阅者和代理节点之间通过Zookeeper实现了负载均衡机制，在Zookeeper中管理全部活动的订阅者和代理节点信息，当有订阅者和代理节点的状态发生变化时，才实时进行系统的负载均衡的调整，保障整个系统处于一个良好的均衡状态。

存在不足：Kafka系统存在的不足主要包括:只支持部分容错，即，节点失效转移时会丢失原节点内存中的状态信息；代理节点没有副本机制保护，一旦代理节点出现故障，该代理节点中的数据将不再可用；代理节点不保存订阅者的状态，删除消息时无法判断该消息是否已被阅读。

## Microsoft TimeStream

TimeStream是Microsoft在StreamInsight的基础上开发的一款分布式的、低延迟的、实时连续的大数据流式计算系统，通过弹性替代机制，可以自适应因故障恢复和动态配置所导致的系统负载均衡的变化，使用C#.NET来编写。

TimeStream的开发是基于大数据流式计算以下两点来考虑的:(a) 连续到达的流式大数据已经远远超出了单台物理机器的计算能力，分布式的计算架构成为必然的选择；(b) 新产生的流式大数据必须在极短的时间延迟内，经过相关任务拓扑进行计算后，产生出能够反映该输入数据特征的计算结果。

任务拓扑结构：TimeStream中的数据计算逻辑是基于数据流DAG实现的，如图4所示，在数据流DAG中的每个顶点v，在获取输入数据流i后，触发相关操作fv，产生新数据流o，并更新顶点v的状态从t到t’，即，(t’，o)=fv(t，i)。

![TimeStream](/images/streamComputing/TimeStream.jpg)

<center>图4 TimeStream数据流任务拓扑顶点</center>

在TimeStream中，一个数据流子图sub-DAG是指在数据流DAG中，两顶点及该两顶点间的全部顶点和有向边的集合，即，满足:对于数据流子图sub-DAG中任意两顶点v1和v2，以及数据流DAG中任意一顶点v，若顶点v位于顶点v1和v2的有向边上，那么顶点v一定是数据流子图sub-DA G的一个顶点。数据流子图sub-DAG在逻辑上可以简化为一个与其功能相同的顶点，如图 22所示，在一个由7个顶点所组成的数据流DAG中，由顶点v2， v3，v4和v5及其有向边所构成的数据流子图sub-DAG，可以简化为一个输入数据流为i、输出数据流为o的逻辑顶点。

弹性等价替代：在TimeStream中，当出现服务器故障或系统负载剧烈持续变化的情况时，可以通过数据流子图sub-DAG间、数据流子图sub-DAG与顶点间以及各顶点间的弹性等价替代，动态、实时地适应系统的负载变化需求。具体而言，弹性等价替代可以进一步细分为3种情况:

1. 顶点间的弹性等价替代。当数据流DAG中的任意一顶点v出现故障不能正常工作时，系统会启动一个具有相同功能的顶点v¢，并接管顶点v的工作；
2. 数据流子图sub-DAG与顶点间的弹性等价替代。如图5所示，当整个系统的负载过轻时，为了节省系统的资源，可以通过一个新的顶点v代替由顶点v2，v3，v4和v5所组成的数据流子图sub-DAG，该新顶点v将实现数据流子图sub-DAG的全部功能；反之，当系统的负载过重时，也可以用一个数据流子图sub-DAG代替任意一个顶点v，实现功能的分解和任务的分担；
3. 数据流子图sub-DAG间的弹性等价替代。如图6所示，右侧由顶点v2，v3，v4和v5所组成的数据流子图sub-DAG实现了HashPartition，Computation和Union等功能，但当系统的Computation功能的计算量突然持续增大后，用左侧由顶点v8，v9，v10，v11，v12和v13所组成的数据流子图sub-DAG弹性等价替代右侧的 子图，实现了将Computation计算节点由2个增加到4个，提高了Computation的计算能力。

![TimeStream sub-DAG](/images/streamComputing/TimeStreamsubDAG.jpg)

<center>图5 TimeStream 数据流子图sub-DAG</center>

![TimeStream replace](/images/streamComputing/TimeStreamreplace.jpg)

<center>图6 TimeStream 数据流子图sub-DAG弹性等价替代</center>

通过弹性等价替代机制可以有效地适应系统因故障和负载的变化对系统性能产生的影响，保证系统性能的稳定性；但在弹性等价替代的过程中，一定要实现替代子图或顶点间的等价，并尽可能地进行状态的恢复。所谓的等价，即对于相同的输入，子图或顶点可以在功能上产生相同的输出，唯一存在的区别在于其性能的不同。状态的恢复是通过对数据流DAG中的依赖关系跟踪机制来实现，并尽可能全面地进行系统状态的恢复。

系统架构:在TimeStream的系统结构中，实现了资源分配、节点调度、故障检测等功能。如图7所示，位于头节点(head node)中的集群管理器(cluster manager，简称CM)实现了对系统资源的管理和任务的分配，位于计算节点(compute node)的节点服务器(node service，简称NS)实现了对计算节点的管理和维护。当一个新的数据流任务进入系统被计算时:首先，系统为该任务分配一个全局唯一的查询协调器(query coordinator，简称QC)，查询协调器QC向集群管理器CM请求资源运行任务的数据流DAG；其次，向节点服务器NS请求调度顶点处理器(vertex processes，简称VP)，并实现数据流DAG的构建；再次，实施数据计算；最后，查询协调器QC和顶点处理器VP均会实时地跟踪系统的运行情况，并定期地将相关元数据信息保持到数据库中，在出现系统故障或负载剧烈持续变化的情况时，可以通过这些被永久保存的元数据进行系统状态的恢复和实时动态的调整。

![TimeStream architecture](/images/streamComputing/TimeStreamarchitecture.jpg)

<center>图7 TimeStream系统架构</center>

存在不足:TimeStream系统存在的不足主要包括:数据延迟在秒级，无法满足毫秒级的应用需求；基于依赖关系跟踪的容错机制降低了系统性能，当系统规模为16个节点时，系统吞吐量下降了10%左右。

## Microsoft Naiad

Naiad是一个研究并行数据流的计算框架，本着Dryad和DryadLINQ的精神。Dryad项目主要研究用于为小集群或大型数据中心编写并行和分布式程序的编程模型。DryadLINQ则是用于开发大规模的、运行在大集群上的并行应用，他具有轻便、强大、优雅的编程环境。与Dryad和DryadLINQ有别，Naiad专注于增量计算（低延时的流计算和有环路的计算）。

Naiad是一个用来执行并行数据、循环数据流程序的分布式系统，可以为批处理提供高吞吐，为流处理提供低延时，提供迭代和增量计算。Naiad把这些特性集中在一个平台上。

Timely dataflow是一个基于有向图的计算模型，顶点有状态，可以收发逻辑上的时间戳消息，图中可能包含环。一个Timely dataflow图包括：输入顶点，输出顶点。分别对应外部生产者和外部消费者。包括环，其中有：进入顶点，出口顶点，回溯顶点，如图8所示。Naiad是Timely dataflow在分布式集群中的数据并行计算原型实现。在Naiad的图中，边上承载记录，记录带有逻辑时间戳信息，时间戳带有输入时代（epochs）和循环迭代信息，前者是一种标记，可以用以区分输入顶点不接收的消息，和输出节点不再输出消息；后者是循环计数器，包含k个信息量分别代表k个循环的计数情况。

![Timely Dataflow](/images/streamComputing/Naiad.jpg)

<center>图8 Timely Dataflow计算模型图</center>

在Naiad中定义了四种操作函数：

	v.ONRECV(e : Edge， m : Message， t : Timestamp)；
	v。ONNOTIFY(t : Timestamp)；
	this.SENDBY(e : Edge， m : Message， t : Timestamp)；
	this.NOTIFYAT(t : Timestamp)。
	
这四种操作函数提供了足够的能力进行适应批量、流式、迭代和增量等多种运算，同时也在可靠性上给予了支持。

Naiad中包含很多worker来管理一个timely dataflow 顶点的一部分，本地workers共享内存交换数据，远端则通过TCP链接。每个逻辑上的图有多个阶段，每个阶段对应多个workers，而链接则由边代表。

## 其它

Discretized Stream（Spark Streaming）是在Spark基础上实现的对于流式处理的分布式处理系统，D-Stream的一个特点是可以实现在一个平台上的流处理，批处理，交互式处理同时工作。D-Stream致力于分布式流式处理系统中的容错问题，解决节点错误或者缓慢的问题，提供低消耗的快速恢复机制，D-Streams把流式计算看做一系列无状态确定性的小时间间隔的批计算。实现上面临两个问题：低延时和快速恢复，分别利用RDDs和Parallel recovery来解决。两种有力的恢复技术：parallel recovery 和 speculative execution：parallel recovery是state RDD patitions 和 tasks可以在其他节点上重新计算，当出现节点失效时可以从最近的checkpoint来重新计算执行。对于缓慢节点，如果超过1.4倍则运行一个副本来重新执行，算法Flux和DPC，同样做了Master恢复的工作。D-Stream把每秒或者每毫秒的数据放入一个时间间隔中来运行一个mapreduce操作，并通过数轮的迭代来不断添加和修改结果。每个时间间隔中接收的数据被可靠地存储在集群中组成一个input dataset，一旦时间间隔结束这个数据集被执行（map，reduce，groupby等）并产生一个新的数据集，如果是程序输出则直接推送到外部系统中，如果是中间数据则被存储为RDDs（一种快速存储抽象，可以避免用lineage for recovery来复制）。

Twitter Rainbird是一款基于Zookeeper， Cassandra， Scribe， Thrift的分布式实时统计系统。Twitter Rainbird并不开源。它在Cassandra之上创建，使用Scala编写，依赖于Zookeeper、Scribe和Thrift。每秒可以写入10万个事件，而且都带有层次结构，或者进行各种查询，延迟小于100ms。目前Twitter已经在Promoted Tweets、微博中的链接、短地址、每个用户的微博交互等生产环境使用了Rainbird。Rainbird的设计架构图如图9。整个Rainbird系统中各个组件之间的协调和容灾处理由ZooKeeper负责，Cassandra负责整个数据的存储和统计。Front End中部署了Scribe，收集需要统计的数据，然后将收集到数据实时地发生到Rainbird Aggregator中。Rainbird Aggregator将缓存收集的数据（1M），并将缓存的数据进行一次预处理，然后再将数据一次性批量写入到Cassandra中。这里预处理的作用类似于MapReduce框架中的combiner的作用，在Maper端做Reduce。Rainbird Query接受用户的查询请求，直接到Cassandra中查询已经统计好的数据返回给客户端。

![Rainbird](/images/streamComputing/Rainbird.jpg)

<center>图9 Rainbird设计架构图</center>

银河流数据处理平台由淘宝提出并使用，通用的流数据实时计算系统，但是它并没有开源。它以实时数据产出的低延迟、高吞吐和复用性为初衷和目标，采用actor模型构建分布式流数据计算框架（底层基于akka），功能易扩展、部分容错、数据和状态可监控。银河具有处理实时流数据（如TimeTunnel收集的实时数据）和静态数据（如本地文件、HDFS文件）的能力，能够提供灵活的实时数据输出，并提供自定义的数据输出接口以便扩展实时计算能力。 银河目前主要是为魔方提供实时的交易、浏览和搜索日志等数据的实时计算和分析。

Flume是Cloudera在2009年提供的一个分布式、可靠的、高可用的日志收集系统，用于收集、聚合以及移动大量日志数据，它也是开源。Flume支持在日志系统中定制各类数据发送方，用于收集数据；同时，Flume提供对数据进行简单处理，并写到各种数据接收方（可定制）的能力。

Scribe是Facebook开源的日志收集系统，在2008年被提出。Scribe从各种日志源上收集日志，存储到一个中央存储系统（可以是NFS，分布式文件系统等）上，以便于进行集中统计分析处理。它为日志的“分布式收集，统一处理”提供了一个可扩展的，高容错的方案。Scribe通常与Hadoop结合使用，Scribe用于向HDFS中push日志，而Hadoop通过MapReduce作业进行定期处理。

Samza是一个开源的分布式流处理系统，非常类似于Storm，不同的是它运行在Hadoop之上，并且使用了自己开发的Kafka分布式消息处理系统。它是LinkedIn在2010年提出并开源的。它只有几千行代码，完成的功能就可以和Storm媲美，当然目前还有很多的不足。它和Kafka结合紧密，更方便的处理数据。它运行在Yarn上。

还有一些较早的流式处理系统：Hstreaming构建在Hadoop之上，可以和Hadoop及其生态系统紧密结合起来提供实时流计算服务；Esper & Nesper专门进行复杂事件处理（CEP）的流处理平台，可以方便开发者快速开发部署处理大容量消息和事件的应用系统，不论是历史的还是实时的消息；StreamBase是一个商业流式计算系统，是一个关于复杂事件处理（CEP）、事件流处理的平台。Borealis是Brandeis University、Brown University和MIT合作开发的一个分布式流式系统，由之前的流式系统Aurora、Medusa演化而来，已经停止维护，最新的Release版本停止在2008年。这些系统都没有开源。



