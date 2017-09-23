---
layout: post
title: 大数据图处理系统总结
description: 调研和总结了几个图处理系统——Kineograph，Pregel，GraphLab，PowerGraph
category: 大数据
tags: [分布式系统,图处理]
---

图数据处理主要针对图数据的迭代计算。图广泛适应于许多领域的应用，如社交网络、Web 网络、生物数据分析和软件代码剽窃检测等。图模型正在互联网和社交网络领域得到越来越广泛的应用。随着真实世界中实体规模的扩张，导致对应的图数据规模迅速增长，因而构建高效的大规模图数据处理系统也成为了急需解决的问题。图计算的算法和机器学习算法类似，通常是复杂的、多阶段或迭代的计算。

<!--more-->

## Kineograph

Kineograph是一个分布式系统，在2012年被提出，它使用流式的输入数据来构造一个连续变化的图，可以捕捉数据中的关系。Kineograph支持图的挖掘算法，它用一个时代交换协议（epoch commit protocol）创建一系列的一致性快照，来适应那些在静态图中工作的图挖掘算法。为了适应图的连续更新，Kineograph引入了一个增量式的图计算引擎。

Kineograph主要着眼于三大挑战：第一，Kineograph必须支持连续更新，同时计算必须保证实时性，一般能1-2分钟内；第二，Kineograph需要包含一个图的机构，可以反映不同实体之间的关系，困难在于图很大，同时需要在分布式环境下保证一致性；第三，Kineograph需要在连续变化的图中支持图挖掘算法。

Kineograph和其他系统的关键不同在于图更新和图计算的分离。为了实现这种分离，Kineograph将图结构元数据和应用数据分开存储，图更新仅仅修改元数据。同时快照的方法使得图的更新和计算分离。

因此Kineograph需要支持一个分布式的内存的图存储系统，图引擎支持增量式的迭代的传播支持的图挖掘。这个分布式图存储定期产生可靠的一致性的快照，适应图挖掘算法。Kineograph的graph node有两个层次：计算层和存储层。前者负责数据计算，增量式的图挖掘；后者负责维护图数据，实现快照。存储层中分布式KV存储，一个图会被分割成确定数目（512）的逻辑分区，划分过程是按照点id的哈希。快照不会阻断更新的输入，ingest node分发和graph noede存储更新操作是可以同快照的创建重叠进行的。时代提交机制（Epoch Commit）提供了一种新的并发控制机制，由于图更新的简单性，不需要保证全局串行性。

Kineograph的工作流程如图1所示：

![Kineograph](/images/GraphComputing/Kineograph.jpg)

<center>图1 Kineograph的执行流程说明</center>

1. 原始数据通过一系列ingest nodes进入Kineograph；
2. ingest node分析输入，将输入记录变成一系列操作，创建一个图更新操作的事务(transaction)并分配一个序列号，然后在graph noedes中根据这个号来分布式操作，在graph nodes中用邻接表存储点（这是图结构元数据），应用数据分开存储；
3. graph nodes从ingest nodes中存储图的更新，而后ingest node向progress table报告更新，progress table是一个全局的中心式服务；
4. 有一个snapshooter定期命令所有graph有一个snapshooter定期命令所有graph node根据当前progress table中的序列号码向量做快照，这个向量是全局的逻辑时钟，定义时代（epoch）的结束；
5. graph nodes然后在这个时钟下计算和提交所有存储的本地图的更新，遵从一个预定义顺序，这个时代提交（epoch commit）结果是产生一个图结构的快照；
6. 图结构的更新会在新的快照中触发增量式的图计算来更新相应的权值。

计算的调度机制类似于GraphLab中的Partitioned scheduler，可以看做是Pregel的BSP和GraphLab的动态调度的混合。但是Kineograph不需要强制数据一致性，Kineograph中不允许写邻居，因此没有写写冲突，实验中也没有发现强一致性的需求。

## Google Pregel

Pregel的论文是在2010年发表在SIGMOD上的，Pregel这个名称是为了纪念欧拉，在他提出的格尼斯堡七桥问题中，那些桥所在的河就叫Pregel。最初是为了解决PageRank计算问题，由于MapReduce并不适于这种场景，所以需要发展新的计算模型去完成这项计算任务，在这个过程中逐步提炼出一个通用的图计算框架，并用来解决更多的问题。

Pregel是一种面向图算法的分布式编程框架，采用迭代的计算模型：在每一轮，每个顶点处理上一轮收到的消息，并发出消息给其它顶点，并更新自身状态和拓扑结构（出、入边）等。

Pregel主要针对于大规模的图问题中面临的挑战：构建分布式框架，每次引入新算法或数据结构都需要花很大的精力；依赖分布式平台入MapReduce等，存在易用性和性能等问题，不适用于图算法（图算法更适合消息传递模型）；单机无法适应问题规模的扩大；现存的并行图模型系统：没有考虑到大规模系统比较重要的问题如容错性等。

Pregel计算模型基于BSP模型，所有节点构成有向图，采用消息传递模型。Pregel计算包含一连串的Supersteps，在每个Superstep，框架对所有的顶点调用一个用户定义的函数。该函数读取当前节点V 在上一个superstep中接收到的消息，向其它顶点（可以是任何节点，数目可变）发送消息（用于下一个superstep），更改V的状态和出边。在并行化方面，每个顶点独立地进行本地操作，所有的消息都是从第S步发送到第S+1步。图2通过一个简单的例子来说明这些基本概念：给定一个强连通图，图中每个顶点都包含一个值，它会将最大值传播到每个顶点。在每个超级步中，顶点会从接收到的消息中选出一个最大值，并将这个值传送给其所有的相邻顶点。当某个超级步中已经没有顶点更新其值，那么算法就宣告结束。

![pregel](/images/GraphComputing/pregel.jpg)

<center>图2 节点最大值的例子说明Pregel的计算过程</center>

Pregel框架由集群管理系统进行作业调度：完成资源分配、作业重启和转移等，用GFS或BigTable进行持久存储。Pregel采用Master-slave结构，有一个节点（机器）扮演master角色，其它节点通过name service定位该顶点并在第一次时进行注册；master负责对顶点集合进行切分到各节点（也可以用户自己指定，考虑load balance等因素），根据顶ID哈希分配顶点到机器（一个机器可以有多个节点，通过name service进行逻辑区分）；节点间异步传输消息；通过checkpoint机制实行容错（更高级的容错通过confined recovery实现：log），节点向master汇报心跳（ping）维持状态。

## GraphLab

GraphLab是一种新的ML（Machine Learning）框架，利用稀疏结构和通用的ML算法的计算模式。graphlab使ML人员能够轻松地设计和实现高效的可扩展并行算法通过将问题指定计算、数据依赖和调度组合在一起，并提供了GraphLab一种有效的的共享内存的实现，利用它构建了四种流行的机器学习算法的并行版本。GraphLab用于实现高效的正确的并行机器学习算法。在MR的模型上发展而成，可以简洁地表达异步迭代算法，有稀疏的计算依赖性，同时可以确保数据的一致性并实现高度并行性能。

Graphlab的主要贡献有：一个基于图的数据模型，模型展现了数据和计算过程中的依赖；一组并行访问的模式来保证一系列的并行一直性；一种复杂的模块调度机制；它实现和通过实验评估参数学习和图形算法模型的推论。

GraphLab data model包括两个部分：一个有向数据图和一个共享数据表。数据图需要满足稀疏计算结构和直接修改的程序状态，用户可以分配数据到顶点和边，分别表示为Dv和Du->v。共享数据表是一个关联map，kv结构。

GraphLab中有两种计算：update function，代表了本地计算，类似于map但是不同在于它可以允许访问和修改重叠；sync mechanism，代表了全局聚合，类似于reduce但是不同在于它和update同时进行。Sv代表v的邻居和update function的可操作范围。每个GraphLab程序可以有多个update，这与调度模型有关。在sync mechanism中，聚合了图中所有的顶点，用户提供一个key k，一个fold 函数（修改顶点数据），一个apply 函数（最终输出权值）和一个初始权值rk0给SDT，以及一个merge函数来构建并行树的合并剪枝。

GraphLab中提供了三种数据一致性：full consistency，保证Sv的数据不被读和修改；edge consistency，v的邻接边；vertex consistency，仅仅点v。GraphLab实现了一个sequential consistency，即如果串行是正确的则可以保证并行正确性，三个条件是：full是满足的，edge在update函数不修改邻接点的数据满足，vertex在update只修改顶点数据时满足。如图3所示。

![GraphLab](/images/GraphComputing/GraphLab.jpg)

<center>图3 GraphLab中的数据一致性对应的不同数据控制范围</center>

GraphLab的update调度描述了顶点上update函数的执行顺序，用调度器来表示，这是一个动态的任务列表。Jacobi style 算法使用同步调度器，确保所有顶点同时更新，Gauss-Seidel style算法使用循环调度器确保所有顶点依次使用最近可用数据。GraphLab提供两类任务调度器，FIFO调度器只允许任务创建而不允许重新排序，优先调度器允许任务根据负载增加情况重排，同时提供了放松的版本。同时GraphLab提供了splash 调度器，沿着生成树来执行任务。GraphLab提供了一种集合调度器，使得用户可以自定义update调度，形如：(S1，f1)...S1表示执行f1的顶点集合，按照顺序执行，需要根据图的结构来构建执行计划。而对于结束条件的评估有两种：第一种根据调度器，在没有任务时给出结束信号；第二种根据用户提供的结束函数来测试SDT。

## PowerGraph

PowerGraph是GraphLab的后续版，使得它有效地处理自然图形或幂律（power-law）图——这是有大量不良连接点和少量良好连接点的图，其中少数点的度很高，大多数的点的度较低，比如微博中的大V。PowerGraph提供了一种在分布式环境下的新的快速方式实现幂律图数据框架。

PowerGraph面向的是稀疏的有向图，同时同Pregel一样是顶点编程。PowerGraph虽然是GraphLab的后续版本，但是在一定程度也可以看做是GraphLab和Pregel的结合，它一方面沿用了GraphLab中的数据图（Data-graph）和共享内存（Shared-memory）的结构，另一个方面类似于Pregel使用了GAS模型，即：gather收集邻接点和边的信息，apply更新中心点的value，scatter向邻接的边发送新的value。gather阶段只读，apply对顶点只写，scatter对边只写。由于vertex计算会频繁调用Gather阶段操作，而大多数相邻的vertex的值其实并不会变化，为了减少计算量，PowerGraph提供了Delta Cache机制。同时，在顶点分割中PowerGraph跟据图的整体分布概率密度函数计算顶点切割的期望值，根据该期望值指导对顶点进行切割，并修改了传统的通信过程，并把其中随机选择一个作为master。

PowerGraph提供了同步和异步两种计算方式：

同步执行流程：

1. Excange message阶段，master接受来自mirror的消息；
2. Receive Message阶段，master接收上一轮Scatter发送的消息和mirror发送的消息，将有message的master激活， 对于激活的顶点，master通知mirror激活，并将vectex_program同步到mirrors；
3. Gather阶段，多线程并行gather， 谁先完成，多线程并行localgraph中的顶点，mirror将gather的结果到master；
4. Apply阶段，master执行apply，并将apply的结果同步到mirror 
5. Scatter阶段，master和mirror基于新的顶点数据，更新边上数据，并以signal的形式通知相邻顶点。

![PowerGraph-sync](/images/GraphComputing/PowerGraphSync.jpg)

<center>图4 PowerGraph同步执行流程</center>

异步执行流程：

1. 在每一轮执行开始时，Master从全局的调度器(Sceduler)获取消息，获取消息后，master获得锁，并进入Locking状态。同时，master通知mirror获取锁，进入Locking状态。
2. master和mirror分别进行Gathering操作，mirror将gathering结果汇报给master，由master完成汇总。
3. master完成applying之后，将结果同步到mirror上。
4. master和mirror独立的执行scattering，执行完成之后释放锁进入None状态，等待新的任务到来。
5. mirror在scattering状态时，可能再次接收到来自master的locking请求，这种情况下，mirror在完成scattering后将不会释放锁直接进入下一轮任务。

![PowerGraph-async](/images/GraphComputing/PowerGraphAsync.jpg)

<center>图5 PowerGraph异步执行流程</center>


