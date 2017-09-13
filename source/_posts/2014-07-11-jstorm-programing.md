---
layout: post
title: Jstorm源码分析——Topology
description: 初步分析Jstorm的Topology和基本编程结构
category: 大数据
tags: [JStorm,Storm,分布式系统]
---


JStorm 是一个分布式实时计算引擎。JStorm 是一个类似Hadoop MapReduce的系统， 用户按照指定的接口实现一个任务，然后将这个任务递交给JStorm系统，Jstorm将这个任务跑起来，并且按7 * 24小时运行起来，一旦中间一个worker 发生意外故障， 调度器立即分配一个新的worker替换这个失效的worker。因此，从应用的角度，JStorm 应用是一种遵守某种编程规范的分布式应用。从系统角度， JStorm一套类似MapReduce的调度系统。 从数据的角度， 是一套基于流水线的消息处理机制。实时计算现在是大数据领域中最火爆的一个方向，人们对数据的要求越来越高，实时性要求也越来越快，传统的Hadoop Map Reduce，逐渐满足不了需求，因此在这个领域Storm 和JStorm 展露头角。JStorm 是用java 完全重写Storm内核， 并重新设计了调度、采样、监控、HA，并对Zookeeper和RPC 进行大幅改良，让性能有30%的提升， 从而JStorm比storm更稳定， 更快，功能更强。

这篇文章主要介绍JStorm的Topology和基本编程结构，编程结构基本类似于Storm，结合源码简单说一下，也做一个备份。

之后还会有几篇文章来分析JStorm的源码。

## Topology示例

我们从一个例子来说明Jstorm中如何定义一个Topology，然后再深入说明其中的调用关系：
	
	//Step 1. Create topology object
	//创建topology的生成器
	TopologyBuilder builder = new TopologyBuilder();

	//创建Spout， 其中
	// - new SequenceSpout() 为真正spout对象，
	// - SequenceTopologyDef.SEQUENCE_SPOUT_NAME 为spout的名字，注意名字中不要含有空格
	int spoutParal = get("spout.parallel", 1); //获取spout的并发设置
	SpoutDeclarer spout = builder.setSpout(SequenceTopologyDef.SEQUENCE_SPOUT_NAME,
					new SequenceSpout(), spoutParal);

	//创建bolt， 
	// - SequenceTopologyDef.TOTAL_BOLT_NAME 为bolt名字，
	// - TotalCount 为bolt对象，
	// - boltParal为bolt并发数，
	//shuffleGrouping（SequenceTopologyDef.SEQUENCE_SPOUT_NAME），表示接收
	//SequenceTopologyDef.SEQUENCE_SPOUT_NAME的数据，并且以shuffle方式，
	//即每个spout随机轮询发送tuple到下一级bolt中
	int boltParal = get("bolt.parallel", 1); //获取bolt的并发设置
	BoltDeclarer totalBolt = builder.setBolt(SequenceTopologyDef.TOTAL_BOLT_NAME, new TotalCount(),boltParal).shuffleGrouping(SequenceTopologyDef.SEQUENCE_SPOUT_NAME);

	//Step 2. Create Config object
	//topology所有自定义的配置均放入这个Map
	Map conf = new HashMp();

	//设置表示acker的并发数
	int ackerParal = get("acker.parallel", 1);
	Config.setNumAckers(conf, ackerParal);

	//表示整个topology将使用几个worker
	int workerNum = get("worker.num", 10);
	conf.put(Config.TOPOLOGY_WORKERS, workerNum);

	//设置topolog模式为分布式，这样topology就可以放到JStorm集群上运行
	conf.put(Config.STORM_CLUSTER_MODE, "distributed");

	//Step 3. Submit Topology
	//提交topology
	StormSubmitter.submitTopology(streamName, conf, builder.createTopology());

其中，TopologyBuilder（backtype.storm.topology.TopologyBuilder）是Topology的生成器，一个Topology的创建工作必须由它来控制。其中关键的方法有： createTopology()，setBolt()，setSpout()，BoltGetter.grouping()。

createTopology()中会获取在应用代码中已经定义好的bolt、spout和对于的连接关系，分别放入boltSpecs、spoutSpecs中，然后由此建立一个StormTopology对象并返回。StormTopology就是一个Topology对象，包括bolt和spout等信息，通过submitTopology()方法直接上传。

![createTopology](/images/jstorm/image001.png)

setBolt()是添加一个Bolt到Topology中的过程，有四种重载的形式：

* public BoltDeclarer setBolt(String id, IRichBolt bolt)
* public BoltDeclarer setBolt(String id, IRichBolt bolt,Number parallelism_hint)
* public BoltDeclarer setBolt(String id, IBasicBolt bolt)
* public BoltDeclarer setBolt(String id, IBasicBolt bolt,Number parallelism_hint)

其最终都是利用其中第二个方法进行实现，其中包括的工作有：验证id是否可用，初始化构成（initCommon()，连接关系的结构和并行度等），放入bolt队列。

![setBolt](/images/jstorm/image002.png)

![initCommon](/images/jstorm/image003.png)

setSpout()与setBolt()类似，不再赘述。

grouping()定义在BoltGetter中，它是TopologyBuilder的一个嵌套类，也是setBolt()的返回类型，grouping()中定义了bolt、spout的连接关系和分组方式，其中分组方式有：fields，shuffle，all，none，direct，custom_object，custom_serialized，local_or_shuffle这八种，分别对应fieldsGrouping()、shuffleGrouping()等方法来连接两个component（bolt/spout），而它们也是调用了grouping()方法来实现。grouping(String componentId, String streamId, Grouping grouping)会把传入的component（bolt/spout）放入当前的component（bolt）的input队列中表示前者输出的tuple会送入后者，同时会带有一个grouping的参数表示分组方式。

![grouping](/images/jstorm/image004.png)

## Bolt

### IRichBolt

在setBolt()中我们需要传入一个Bolt，比如例子中的TotalCount()，它继承了IRichBolt接口。

IRichBolt接口是最为简单的Bolt接口，它实现了IBolt和IComponent两个接口，前者定义了void prepare(Map stormConf, TopologyContext context, OutputCollector collector)，void execute(Tuple input)，void cleanup()三个函数，后者定义了void declareOutputFields(OutputFieldsDeclarer declarer)和Map<String, Object> getComponentConfiguration()两个函数。

![TotalCount](/images/jstorm/image005.png)

注意：

* bolt对象必须是继承Serializable， 因此要求bolt内所有数据结构必须是可序列化的；
* bolt可以有构造函数，但构造函数只执行一次，是在提交任务时，创建bolt对象，因此在task分配到具体worker之前的初始化工作可以在此处完成，一旦完成，初始化的内容将携带到每一个task内（因为提交任务时将bolt序列化到文件中去，在worker起来时再将bolt从文件中反序列化出来）。
* prepare是当task起来后执行的初始化动作
* cleanup是当task被shutdown后执行的动作
* execute是bolt实现核心， 完成自己的逻辑，即接受每一次取消息后，处理完，有可能用collector 将产生的新消息emit出去。在executor中，当程序处理一条消息时，需要执行collector.ack，当程序无法处理一条消息时或出错时，需要执行collector.fail ，详情可以参考 ack机制；
* declareOutputFields， 定义bolt发送数据，每个字段的含义
* getComponentConfiguration 获取本bolt的component 配置

### IBasicBolt

很多bolt有些类似的模式:

* 读一个输入tuple
* 根据这个输入tuple发射一个或者多个tuple
* 在execute的方法的最后ack那个输入tuple

遵循这类模式的bolt一般是函数或者是过滤器, 这种模式太常见，storm为这类模式单独封装了一个接口: IBasicBolt。IBasicBolt继承Icomponent接口，而自己定义了prepare、execute和cleanup三个函数。

区别就在于IBasicBolt的execute需要传入一个BasicOutputCollector对象，其中定义了不同的emit函数，通过这种方式定义的bolt会在自动ack。

![IBasicBolt](/images/jstorm/image006.png)

## Spout

### IRichSpout

与Bolt类似，IRichSpout是最为简单的Spout接口，它实现了ISpout和Icomponent两个接口。其中ISpout接口中定义了：void open(Map conf, TopologyContext context, SpoutOutputCollector collector)，void close()，void activate()，void deactivate()，void nextTuple()。void ack(Object msgId)，void fail(Object msgId)。

注意：

* spout对象必须是继承Serializable， 因此要求spout内所有数据结构必须是可序列化的
* spout可以有构造函数，但构造函数只执行一次，是在提交任务时，创建spout对象，因此在task分配到具体worker之前的初始化工作可以在此处完成，一旦完成，初始化的内容将携带到每一个task内（因为提交任务时将spout序列化到文件中去，在worker起来时再将spout从文件中反序列化出来）。
* open是当task起来后执行的初始化动作
* close是当task被shutdown后执行的动作
* activate 是当task被激活时，触发的动作
* deactivate 是task被deactive时，触发的动作
* nextTuple 是spout实现核心， nextuple完成自己的逻辑，即每一次取消息后，用collector 将消息emit出去。
* ack， 当spout收到一条ack消息时，触发的动作，详情可以参考 ack机制
* fail， 当spout收到一条fail消息时，触发的动作，详情可以参考 ack机制
* declareOutputFields， 定义spout发送数据，每个字段的含义
* getComponentConfiguration 获取本spout的component 配置

### IRichStateSpout

这里还定义了一种IRichStateSpout，不清楚功能.

## Topology的生命周期

这部分主要参照的代码：

* com.alibaba.jstorm.daemon.nimbus.StatusType
* com.alibaba.jstorm.cluster.StormStatus

Topology的状态存储在zookeeper中，包括类型有：kill, killed, monitor, inactive, inactivate, active, activate, startup, remove, rebalance, rebalancing, do-rebalance。其中killed、inactive、active、rebalancing为状态(status)，其他为状态动作(action)。

![StatusType](/images/jstorm/image007.png)

状态说明和动作的触发：

* kill：只有当前状态为active/inactive/killed，当client kill Topology时会触发这个动作。
* monitor：如果当前状态为active，则每Config.NIMBUS_MONITOR_FREQ_SECS seconds会触发这个动作一次，nimbus会重新分配workers。
* inactivate：当前状态为active时，client会触发这个动作。
* activate：当前状态为inactive时，client会触发这个动作。
* startup：当前状态为killed/rebalancing，当nimbus启动时也会触发这个动作。
* remove：当前状态为killed，在client提交kill命令的一段时间之后触发这个动作
* rebalance：当前状态为active/inactive，当client提交 rebalance命令后触发这个动作。
* do-rebalance：当前状态为rebalancing，当client提交rebalance命令一段时间后触发这个动作。

这里需要说明的是，Jstorm中有四种Topology的状态操作命令：activate、deactivate、kill_topology、Rebalance。分别对应了四种状态。

![Topology](/images/jstorm/image008.png)

<center>Topology的状态转移图</center>




