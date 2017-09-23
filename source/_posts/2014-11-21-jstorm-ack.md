---
layout: post
title: Jstorm源码分析——Ack机制
description: 这篇文章分析Jstorm中的ack可靠性机制
category: 大数据
tags: [JStorm,Storm,分布式系统]
---

## 可靠性机制原理说明

Storm一个很重要的特性是它能够保证你发出的每条消息都会被完整处理， 完整处理的意思是指：一个tuple被完全处理的意思是： 这个tuple以及由这个tuple所导致的所有的tuple都被成功处理。而一个tuple会被认为处理失败了如果这个消息在timeout所指定的时间内没有成功处理, 而这个timetout可以通过Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS来指定。也就是说对于任何一个spout-tuple以及它的所有子孙到底处理成功失败与否我们都会得到通知。storm里面有个专门的acker来跟踪所有tuple的完成情况。

<!--more-->

在Spout的接口ISpout中，需要实现的参数有：

	public interface ISpout extends Serializable {
		void open(Map conf, TopologyContext context,
				  SpoutOutputCollector collector);
		void close();
		void nextTuple();
		void ack(Object msgId);
		void fail(Object msgId);
	}
	
Ack和fail分别是在ack成功和失败后执行的函数。

发射tuple的时候spout会提供一个message-id, 后面我们通过这个message-id来追踪这个tuple，如果没有message-id则不会启动acker机制。接下来， 这个发射的tuple被传送到消息处理者bolt那里， storm会跟踪由此所产生的这课tuple树。如果storm检测到一个tuple被完全处理了， 那么storm会以最开始的那个message-id作为参数去调用消息源的ack方法；反之storm会调用spout的fail方法。值得注意的一点是， storm调用ack或者fail的task始终是产生这个tuple的那个task。所以如果一个spout被分成很多个task来执行， 消息执行的成功失败与否始终会通知最开始发出tuple的那个task。

对于一个storm的用户，在生成一个tuple的时候需要通知storm，在完成处理一个tuple以后要通知storm。由一个tuple产生一个新的tuple称为anchoring。发射一个新tuple的同时也就完成了一次anchoring，例如在：_collector.emit(tuple, new Values(word))，这样把输入tuple和输出tuple进行了anchoring，即把新的tuple加入到了tuple的处理树中。而_collector.emit(new Values(word))则不会产生anchoring关系（unanchoring）。

一个输出tuple可以被anchoring到多个输入tuple上，这种方式在stream合并或者stream聚合的时候很有用，一个多入口的tuple被处理失败的话，它对应的输入tuple都要被重新执行。例如：

	List<Tuple> anchors = new ArrayList<Tuple>();
	anchors.add(tuple1);
	anchors.add(tuple2);
	_collector.emit(anchors, new Values(1, 2, 3));

多入口tuple把这个新tuple加到了多个tuple树里面去了。

我们通过anchoring来构造这个tuple树，最后一件要做的事情是在你处理完当个tuple的时候告诉storm,  通过OutputCollector类的ack和fail方法来做。每个你处理的tuple， 必须被ack或者fail。因为storm追踪每个tuple要占用内存。所以如果你不ack/fail每一个tuple， 那么最终你会看到OutOfMemory错误。

大多数Bolt遵循这样的规律：读取一个tuple；发射一些新的tuple；在execute的结束的时候ack这个tuple。这些Bolt往往是一些过滤器或者简单函数。Storm为这类规律封装了一个BasicBolt类。发送到BasicOutputCollector的tuple会自动和输入tuple相关联，而在execute方法结束的时候那个输入tuple会被自动ack的。（作为对比，处理聚合和合并的bolt往往要处理一大堆的tuple之后才能被ack， 而这类tuple通常都是多输入的tuple， 所以这个已经不是IBasicBolt可以罩得住的了。）

storm里面有一类特殊的task称为：acker， 他们负责跟踪spout发出的每一个tuple的tuple树。当acker发现一个tuple树已经处理完成了。它会发送一个消息给产生这个tuple的那个task。你可以通过Config.TOPOLOGY_ACKERS来设置一个topology里面的acker的数量， 默认值是一。 如果你的topology里面的tuple比较多的话， 那么把acker的数量设置多一点，效率会高一点。acker task是非常轻量级的， 所以一个topology里面不需要很多acker。你可以通过Strom UI(id: -1)来跟踪它的性能。 如果它的吞吐量看起来不正常，那么你就需要多加点acker了。

理解storm的可靠性的最好的方法是来看看tuple和tuple树的生命周期， 当一个tuple被创建， 不管是spout还是bolt创建的， 它会被赋予一个64位的id，而acker就是利用这个id去跟踪所有的tuple的。每个tuple知道它的祖宗的id(从spout发出来的那个tuple的id), 每当你新发射一个tuple， 它的祖宗id都会传给这个新的tuple。所以当一个tuple被ack的时候，它会发一个消息给acker，告诉它这个tuple树发生了怎么样的变化。具体来说就是：它告诉acker： 我呢已经完成了， 我有这些儿子tuple, 你跟踪一下他们吧。storm使用一致性哈希来把一个spout-tuple-id对应到acker， 因为每一个tuple知道它所有的祖宗的tuple-id， 所以它自然可以算出要通知哪个acker来ack。

当一个spout发射一个新的tuple， 它会简单的发一个消息给一个合适的acker，并且告诉acker它自己的id(taskid)， 这样storm就有了taskid-tupleid的对应关系。 当acker发现一个树完成处理了， 它知道给哪个task发送成功的消息。

acker task并不显式的跟踪tuple树。对于那些有成千上万个节点的tuple树，把这么多的tuple信息都跟踪起来会耗费太多的内存。相反， acker用了一种不同的方式， 使得对于每个spout tuple所需要的内存量是恒定的（20 bytes) .  这个跟踪算法是storm如何工作的关键，并且也是它的主要突破。一个acker task存储了一个spout-tuple-id到一对值的一个mapping。这个对子的第一个值是创建这个tuple的taskid， 这个是用来在完成处理tuple的时候发送消息用的。 第二个值是一个64位的数字称作：”ack val”, ack val是整个tuple树的状态的一个表示，不管这棵树多大。它只是简单地把这棵树上的所有创建的tupleid/ack的tupleid一起异或(XOR)。当一个acker task 发现一个 ack val变成0了， 它知道这棵树已经处理完成了。 因为tupleid是随机的64位数字， 所以， ack val碰巧变成0(而不是因为所有创建的tuple都完成了)的几率极小。

所有可能的失败场景:

* 由于对应的task挂掉了，一个tuple没有被ack： storm的超时机制在超时之后会把这个tuple标记为失败，从而可以重新处理。
* Acker挂掉了：这种情况下由这个acker所跟踪的所有spout tuple都会超时，也就会被重新处理。
* Spout挂掉了：在这种情况下给spout发送消息的消息源负责重新发送这些消息。比如Kestrel和RabbitMQ在一个客户端断开之后会把所有”处理中“的消息放回队列。

如果可靠性对你来说不是那么重要 — 你不太在意在一些失败的情况下损失一些数据， 那么你可以通过不跟踪这些tuple树来获取更好的性能。不去跟踪消息的话会使得系统里面的消息数量减少一半， 因为对于每一个tuple都要发送一个ack消息。并且它需要更少的id来保存下游的tuple， 减少带宽占用。

有三种方法可以去掉可靠性。第一是把Config.TOPOLOGY_ACKERS 设置成 0. 在这种情况下， storm会在spout发射一个tuple之后马上调用spout的ack方法。也就是说这个tuple树不会被跟踪。第二个方法是在tuple层面去掉可靠性。 你可以在发射tuple的时候不指定messageid来达到不跟粽某个特定的spout tuple的目的。最后一个方法是如果你对于一个tuple树里面的某一部分到底成不成功不是很关心，那么可以在发射这些tuple的时候unanchor它们。 这样这些tuple就不在tuple树里面， 也就不会被跟踪了。

## Acker工作机制

storm里的acker用来跟踪所有tuple的完成情况。acker对于tuple的跟踪算法是storm的主要突破之一， 这个算法使得对于任意大的一个tuple树， 它只需要恒定的20字节就可以进行跟踪了。原理很简单：acker对于每个spout-tuple保存一个ack-val的校验值，它的初始值是0， 然后每发射一个tuple/ack一个tuple，那么tuple的id都要跟这个校验值异或一下，并且把得到的值更新为ack-val的新值。那么假设每个发射出去的tuple都被ack了， 那么最后ack-val一定是0(因为一个数字跟自己异或得到的值是0)。

首先，我们需要注意的是acker实现了IBolt接口，换言之在工作时Acker是作为一个Bolt Task运行的。在提交Topology时，ServiceHandler（org.act.tstream.daemon.nimbus）的submitTopologyWithOpts()中的setupZkTaskInfo会生成TaskInfo，其中创建Task的assignment，首先在创建Topology时同时生成acker（之后分别生成bolt和spout）。

	submitTopologyWithOpts(name, uploadedJarLocation, jsonConf, topology,options);
	——》setupZkTaskInfo(conf, topologyId, stormClusterState);
	——》Map<Integer, String> taskToComponetId = mkTaskComponentAssignments(conf, topologyId);
	——》StormTopology topology = Common.system_topology(stormConf, stopology);
	——》add_acker(ackercount, ret);
	——》IBolt ackerbolt = new Acker();
	
之后为acker创建bolt和spout的输入输出stream。
 
 ![add acker 1](/images/jstorm/image057.png)
 
 ![add acker 2](/images/jstorm/image058.png)
 
在每个bolt/spout的task建立时，在workerData中也会调用Common。System_topology(stormConf, rawTopology)，其中add_acker。

在Acker Task创建时，通过BoltExecutors（org.act.tstream.task.execute. BoltExecutors）的构造函数调用bolt.prepare()。默认timeout时间10s。
 
 ![bolt prepare](/images/jstorm/image059.png)

在Spout发送tuple时，会根据是否需要ack采用不同的策略，如果需要ack则创建一个带有anchoring关系的tuple，并加入pending队列，而后发送给acker一个消息，消息的格式为：(spout-tuple-id, tuple-id, task-id)，消息的streamId是__ack_init(ACKER-INIT-STREAM-ID)，这是告诉acker, 一个新的spout-tuple出来了， 你跟踪一下，它是由id为task-id的task创建的(这个task-id在后面会用来通知这个task：你的tuple处理成功了/失败了)。
   
![send spout message 1](/images/jstorm/image060.png)

![send spout message 2](/images/jstorm/image061.png)
	
如果不使用ack机制（直接ack成功）。SpoutExecutors有一个pending负责对tuple的ack进行监控，如果超时则调用SpoutTimeoutCallBack进行fail操作（向Spout的接收队列中加入一个fail的message）。
 
 ![expire](/images/jstorm/image062.png)
 
对于acker，处理完这个消息之后， acker会在它的pending这个map(类型为TimeCacheMap)里面添加这样一条记录: {spout-tuple-id {:spout-task task-id :val ack-val)}，这就是acker对spout-tuple进行跟踪的核心数据结构， 对于每个spout-tuple所产生的tuple树的跟踪都只需要保存上面这条记录。acker后面会检查:val什么时候变成0，变成0， 说明这个spout-tuple产生的tuple都处理完成了。
 
 ![execute](/images/jstorm/image063.png)
 
对于Bolt来说，处理完一个tuple后，它发送给下一个bolt消息，同时发送给acker ack消息。它调用BoltCollector.boltEmit()时，会检查anchors关系，它会把要ack的tuple的id, 以及这个tuple新创建的所有的tuple的id进行异或运算，然后通过ack把结果发送给acker。在Bolt调用ack时，将pending中异或的结果封装为tuple发送给acker。
 
 ![ack 1](/images/jstorm/image064.png)
 
每个tuple在被ack的时候，会给acker发送一个消息，消息格式是: 

	(spout-tuple-id, tmp-ack-val)
	
消息的streamId是
	
	__ack_ack(ACKER-ACK-STREAM-ID)
	
注意，这里的tmp-ack-val是要ack的tuple的id与由它新创建的所有的tuple的id异或的结果：

	tuple-id ^ (child-tuple-id1 ^ child-tuple-id2 ... )

这时，acker接受到这个tuple后，会更新ack-val值。

Tuple处理失败的时候会给acker发送失败消息，acker会忽略这种消息的消息内容(消息的streamId为ACKER-FAIL-STREAM-ID), 直接将对应的spout-tuple标记为失败。
 
 ![ack 2](/images/jstorm/image065.png)
 
在acker接受到消息并处理完后，acker会检查ack-val值，如果为0则删掉这个tuple树对应pending项，并向对应spout task发送一个tuple，stream-id为Acker. ACKER_ACK_STREAM_ID，表示tuple被处理完成。否则如果这个spout-tuple被标记失败（被主动fail）则同样删掉pending项，并向spout task发送一个tuple，stream-id为Acker.ACKER_FAIL_STREAM_ID，表示fail。
 
 ![ack 3](/images/jstorm/image066.png)
 
在Spout收到tuple后会查看stream-id，如果ack则调用ISpout.ack()，如果是fail则调用ISpout.fail()。

完成一个tuple树的可靠传输。

注意，重发的过程，用户可以自己放在fail()中进行。


