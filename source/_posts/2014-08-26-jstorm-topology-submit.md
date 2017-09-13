---
layout: post
title: Jstorm源码分析——Topology的提交和实例化过程
description: 这篇文章分析Jstorm中提交Topology后整个任务是如何上传和启动工作的。
category: 大数据
tags: [JStorm,Storm,分布式系统]
---

## Topology提交过程流程

Topology的上传过程是将完成的Jstorm程序打包成jar再上传到Jstorm中，通过” jstorm jar xxxxxx.jar com.alibaba.xxxx.xx parameter”命令启动JStorm的上传时会通过这个python脚本中main函数调用对应的函数启动相应jar中的功能模块。而后这个java程序会被编译执行并从命令中的，其中的submitTopology()会触发Topology的提交过程。

![topologySubmit](/images/jstorm/image021.png)
 
<center>图1 Topology提交过程流程图</center>

## Nimbus-接受提交的Topology

在Jstorm-client-Backtype-storm下的StormSubmitter.java中定义了提交jar的方法：submitJar()（submitTopology()调用这个方法将topologies提交给cluster）。

Use this class to submit topologies to run on the Storm cluster. You should run your program with the "storm jar" command from the command-line, and then use this class to submit your topologies.

它首先调用nimbus$iface接口的beginFileUpload()，uploadChunk()，和finishFileUpload()，然后它们会利用sendBase向服务端传送消息调用服务端的对应函数。服务端的对应方法（Jstorm-server-Daemon-Nimbus-ServiceHandler.java）来把jar包上传到nimbus服务器上的/inbox目录。
 
![submit Topology client](/images/jstorm/image022.png)
 
beginFileUpload：

客户端
 
![beginFileUpload client](/images/jstorm/image023.png)
 
服务端：
 
 
![beginFileUpload server](/images/jstorm/image024.png)

uploadChunk：

客户端：
 
![uploadChunk client](/images/jstorm/image025.png)

服务端：
 
 ![uploadChunk server](/images/jstorm/image026.png)
 
finishFileUpload：

客户端：
 
 ![finishFileUpload client](/images/jstorm/image027.png)
 
服务端：

![finishFileUpload server](/images/jstorm/image028.png)
 
然后进行运行topology之前的一些校验。topology的代码上传之后服务端的submitTopology(submitTopologyWithOpts)方法会负责对这个topology进行处理， 它首先要对storm本身，以及topology进行一些校验:

* 它要检查storm的状态是否是active的（服务端）
* 它要检查是否已经有同名的topology已经在storm里面运行了（客户端）
* 因为我们会在代码里面给spout, bolt指定id, storm会检查是否有两个spout和bolt使用了相同的id。
* 任何一个id都不能以”__”开头， 这种命名方式是系统保留的。

这一部分没有找到。

如果以上检查都通过了，那么就进入下一步了。建立topology的本地目录：
 
 ![build dir](/images/jstorm/image029.png)
 
共建立了三方面目录：

1. 为这个topology建立它的本地目录。
2. 建立topology在zookeeper上的心跳目录。
3. zookeeper上的/task目录。

## Nimbus-分配任务给supervisor

nimbus对每个topology都会做出详细的预算：需要多少工作量(多少个task)。它是根据topology定义中给的parallelism hint参数， 来给spout/bolt来设定task数目了，并且分配对应的task-id。并且把分配好task的信息写入zookeeper上的/task目录下: 打比方说我们的topology里面一共有一个spout, 一个bolt。 其中spout的parallelism是2, bolt的parallelism是4, 那么我们可以把这个topology的总工作量看成是6， 那么一共有6个task，那么/tasks/{topology-id}下面一共会有6个以task-id命名的文件，其中两个文件的内容是spout的id, 其它四个文件的内容是bolt的id。
 
 ![setup zookeeper task info](/images/jstorm/image030.png)

 ![make task component assignments](/images/jstorm/image031.png)
 
把计算好的工作分配给supervisor去做。

然后nimbus就要给supervisor分配工作了。工作分配的单位是task(上面已经计算好了的，并且已经给每个task编号了), 那么分配工作意思就是把上面定义好的一堆task分配给supervisor来做， 在nimbus里面，Assignment表示一个topology的任务分配信息：

任务分配单独一个线程TopologyAssign（com.alibaba.jstorm.daemon.nimbus）进行操作。调用关系是Run() -> doTopologyAssignment()-> mkAssignment()。

mkAssignment中进行端口分配等工作。

![make assignments for a topology](/images/jstorm/image032.png)
 
其中核心数据就是task->node+port, 它其实就是从task-id到supervisor-id+port的映射， 也就是把这个task分配给某台机器的某个端口来做。工作分配信息会被写入zookeeper的如下目录:

	/-{storm-zk-root}			-- storm在zookeeper上的根目录
	  |
	  |-/assignments			-- topology的任务分配信息
		  |
		  |-/{topology-id}	-- 这个下面保存的是每个topology的assignments
					信息包括： 对应的nimbus上的代码目录,所有
					task的启动时间,每个task与机器、端口的映射

mkAssignment ()中的set_assignment会保存分配情况到zk目录。

## Nimbus-激活Topology

到现在为止，任务都分配好了，那么我们可以正式启动这个topology了，在源代码里面，启动topology其实就是将Topology的状态设置为active，与此同时向zookeeper上面该topology所对应的目录写入这个topology（即zk中的topology目录下）的信息（stormClusterState.activate_storm (topologyId, stormBase)进行这个工作）。StormBase即zk中存储的Topology的内容。

注意，在任务分配中（doTopologyAssignment()）进行了topology的启动工作，将其状态activate并写入zk。
 
 ![doTopologyAssignment](/images/jstorm/image033.png)
 
到这里为止nimbus的工作算是差不多完成了，下面就看supervisor的了。

## Supervisor 接受和处理Nimbus的指派

SyncSupervisorEvent每supervisor.monitor.frequency.secs s就运行一次（EventManager每隔一段时间加入一个event到队列中，同时其不断从队列里取出event进行run）。
 
 ![supervisor thread](/images/jstorm/image034.png)

SyncSupervisorEvent获取zk上所有的Assignment，再读取本地topology，获取zk上的对应本Supervisor的Assignment（不同于storm），并将其写入localstate，然后下载本地没有下载过的Assignment，之后移除无用的topology，最后启动syncProcessEvent。
 
![SyncSupervisorEvent](/images/jstorm/image035.png)

SyncProcessEvent执行两个工作： 

1. kill bad worker;
2. start new worker。第一步从localstate获取assigned tasks，第二步获取本地的worker状态（心跳），第三步移除无效或者killed的worker同时将有效的worker放入keepPorts，最后开始新的workers，这里它会找到assignedTask但是不在keeperports中的tasks，通过launchWorker产生新的worker承担这个assignment。

![SyncProcessEvent](/images/jstorm/image036.png)
 
## Worker的创建和初始化

每个Worker只针对一个Topology，负责该Topology中某些并行化Task在该机器上的执行。worker的主要任务有:

* 管理Task实例，Task对象管理着RunnableCallBack，用于处理Tuple。
* 接收外部tuple的消息，转发给Task；
* 向外发送tuple消息，发送给下游Task。
* 发送心跳消息
* ……

com.alibaba.jstorm.daemon.worker中从main进入，首先调用mk_worker()，建立一个新的worker，其中实例化一个worker对象（其中实例化一个workerData对象）在workerData中会初始化一些参数，包括当前worker的task list。
 
![workerData](/images/jstorm/image037.png)
 
然后执行worker.execute()，对Worker进行初始化。在worker.execut()中，

* 实例化所对应的task（调用createTasks(),Task.mk_task(worerData,tasked)）
* 创建虚拟端口对象（WorkerVirtualPort）来绑定连接端口。WorkerVirtualPort用于接收Tuple，并通过zeroMQ将Tuple转发给Task对象.
* 建立task对应输出流的连接（makeRefreshConnections()）
* 激活zk中的activate状态
* 建立heartbeat等。
  
  ![worker execute](/images/jstorm/image038.png)
  
  ![create tasks](/images/jstorm/image039.png)
  
其中，在第1步创建task时，com.alibaba.jstorm.task的mk_task()函数实例化一个Task对象。

## Task的创建和初始化

Task在其构造函数中会获取对应的spout/bolt对象。

![task](/images/jstorm/image040.png)
 
在Worker创建Task时，mk_task()函数创建task对象之后将执行其execute()函数。Task.execute()中：

* 创建heartbeat
* 创建线程来接受zeroMQ中的tuple并将tuple转交给bolt/spout处理。调用关系是：execute() -> mkExecutor () -> mk_executors ()，mk_executors()会根据类型创建新的bolt或者spout（BoltExecutors或SpoutExecutors）。
 
 ![make executors](/images/jstorm/image041.png)

## Topology终止

除非你显式地终止一个topology, 否则它会一直运行的，可以用下面的命令去终止一个topology：

	storm kill {stormname}
	
在Jstorm-client中，backtype.storm.command定义了Kill_topology命令的工作：它根据参数调用killTopology(topologyName)或killTopologyWithOpts(topologyName, options)，而后客户端将参数传入服务端调用相应方法（com.alibaba.jstorm.daemon.nimbus中的ServiceHandler.java中的killTopologyWithOpts()）。它会首先检查topology状态，然后把状态转换为killed，通过回调函数KillTransitionCallback()（com.alibaba.jstorm.callback.impl）在2 * Timeout seconds后将状态转换为remove，再调用RemoveTransitionCallback删除zk中topology的相关信息。（这里注意状态转换对应回调函数需要查看stateTransitions的map关系）
 
 ![kill transition callback](/images/jstorm/image042.png)
 
 ![remove transition callback](/images/jstorm/image043.png)
 
 ![remove storm](/images/jstorm/image044.png)
 
上面的代码会把zookeeper上面/tasks, /assignments, /storms下面有关这个topology的数据都删除了。这些数据(或者目录）之前都是nimbus创建的。还剩下/taskbeats以及/taskerrors下的数据没有清除， 这块数据会在supervisor下次从zookeeper上同步数据的时候删除的（supervisor会删除那些已经不存在的topology相关的数据)。这样这个topology的数据就从storm集群上彻底删除了。


