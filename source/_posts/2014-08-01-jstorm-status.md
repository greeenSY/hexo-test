---
layout: post
title: Jstorm源码分析——JStorm的系统状态
description: 初步分析Jstorm的本地目录，zookeeper的本地目录，系统状态等等。
category: 大数据
tags: [JStorm,Storm,分布式系统]
---

## JStorm的系统状态

storm集群里面工作机器分为两种一种是nimbus, 一种是supervisor, 他们通过zookeeper来进行交互，nimbus通过zookeeper来发布一些指令，supervisor去读zookeeper来执行这些指令。

<!--more-->

这篇文章主要介绍JStorm的本地目录，zookeeper的本地目录，以及他们之间如何协调工作。

## ZooKeeper中保存的数据目录结构

Zookeeper在storm中的作用主要有两个方面：

1. Twitter Storm的所有的状态信息都是保存在Zookeeper里面，nimbus通过在zookeeper上面写状态信息来分配任务，supervisor，task通过从zookeeper中读状态来领取任务
2. supervisor, task也会定义发送心跳信息到zookeeper， 使得nimbus可以监控整个storm集群的状态， 从而可以重启一些挂掉的task

Storm在Zookeeper中保存的数据目录结构：

	/-{storm-zk-root}           -- storm在zookeeper上的根目录
	  |
	  |-/assignments          -- topology的任务分配信息
	  |   |-/{topology-id}       -- 这个下面保存的是每个topology的assignments
	  |                  信息包括： 对应的nimbus上的代码目录,所有
	  |                  的启动时间, 每个task与机器、端口的映射
	  |-/tasks             -- 所有的task
	  |   |-/{topology-id}       -- 这个目录下面id为{topology-id}的topology 
	  |       |            所对应的所有的task-id
	  |       |-/{task-id}     -- 这个文件里面保存的是这个task对应的
	  |                  component-id：可能是spout-id或者bolt-id
	  |-/topology             -- 这个目录保存所有正在运行的topology的id
		|                  (这里好像是topology不是storms)
	  |   |-/{topology-id}       -- 这个文件保存这个topology的一些信息，包括
	  |                  topology的名字，topology开始运行时间以及
	  |                  这个topology的状态 (具体看StormBase类)
	  |-/supervisors          -- 这个目录保存所有的supervisor的心跳信息
	  |   |-/{supervisor-id}    	-- 这个文件保存的是supervisor的心跳信息包括: 
	  |                  心跳时间，主机名，这个supervisor上worker
	  |                  的端口号运行时间 (具体看SupervisorInfo类)
	  |-/taskbeats           -- 所有task的心跳
	  |   |-/{topology-id}       -- 这个目录保存这个topology的所有的task的心
	  |       |						跳信息
	  |       |-/{task-id}     -- task的心跳信息，包括心跳的时间，task运行时
	  |                   间以及一些统计信息
	  |-/taskerrors           -- 所有task所产生的error信息
		  |
		  |-/{topology-id}      -- 这个目录保存这个topology下面每个task的出
			  |					错信息
			  |-/{task-id}    -- 这个task的出错信息

源代码主要是: com.alibaba.jstorm.cluster，Jstorm-server中的cluster下的Cluster.java。

![Cluster1](/images/jstorm/image009.png)

![Cluster2](/images/jstorm/image010.png)

状态查询是Cluster.mk_storm_cluster_state()方法定义的，返回类型是StormZkClusterState对象：

* StormClusterState.java中定义了集群状态查询的接口
* StormZkClusterState.java对其进行了实现，其构造函数中，会初始化一些参数，例如将状态ID（state_id）和回调函数关联起来，但是注册的方法定义在Jstorm-client-extension中的Cluster中的ClusterState.java（com.alibaba.jstorm.cluster），它的实现同样在这个目录下的DistributedCluster.java中。此类中还定义了get_data(),get_children等方法，大部分关联在一个zkobj(Zookeeper)的变量上，用到的都是zookeeper的相关方法。这部分内容在Jstorm-client-extension中的Zk中的Zookeeper.java中。

其他查询接口例如：assignment_info()查询分配信息（其中用到Assignment类型数据，定义在Jstorm-server中的Task的Assignment.java），heartbeat_tasks()查看对应目录下存储的心跳，leader_existed()查询leader是否存在等。

Cluster.java中还有一些如topology_task_info()，get_topology_id()等方法也是需要用到StormClusterState对象的对应方法。

## nimbus和supervisor在自己本机存储信息

nimbus机器上面只有/nimbus目录；supervisor机器上面有/supervisor和/workers两个目录。

	/{storm-local-dir}
	  |
	  |-/nimbus
	  |   |-/inbox					-- 从nimbus客户端上传的jar包会在这个目录里
	  |   |  |                         面
	  |   |  |-/stormjar-{uuid}.jar		-- 上传的jar包其中{uuid}表示生成的一个uuid
	  |   |                            
	  |   |-/stormdist
	  |      |
	  |      |-/{topology-id}
	  |         |-/stormjar.jar			-- 包含这个topology所有代码的jar包(从
	  |         |                     nimbus/inbox里面挪过来的)
	  |         |-/stormcode.ser		-- 这个topology对象的序列化
	  |         |-/stormconf.ser		-- 运行这个topology的配置
	  |-/supervisor
	  |   |-/stormdist
	  |   |   |-/{topology-id}
	  |   |      |-/resources			-- 这里保存的是topology的jar包里面的resources
	  |   |      |                     目录下面的所有文件
	  |   |      |-/stormjar.jar			-- 从nimbus机器上下载来的topology的jar包
	  |   |      |-/stormcode.ser		-- 从nimbus机器上下载来的这个topology对象的
	  |   |      |                     序列化形式
	  |   |      |-/stormconf.ser      -- 从nimbus机器上下载来的运行这个topology的
	  |   |                            配置
	  |   |-/localstate					-- supervisor的localstate
	  |   |-/tmp						-- 临时目录，从Nimbus上下载的文件会先存在这
	  |      |                         个目录里面，然后做一些简单处理再copy到
	  |      |                         stormdist/{topology-id}里面去
	  |      |-/{uuid}
	  |         |-/stormjar.jar			-- 从Nimbus上面download下来的工作jar包
	  |-/workers
		  |-/{worker-id}
			  |-/pids					-- 一个worker可能会起多个子进程所以可能会
			  |   |                    有多个pid
			  |   |-/{pid}				-- 运行这个worker的JVM的pid
			  |-/heartbeats			-- 这个supervisor机器上的worker的心跳信息
				 |-/{worker-id}        -- 这里面存的是一个worker的心跳：主要包括
									   心跳时间和worker的id

代码主要包括config.java（Jstorm-client下backtype下storm目录）, StormConfig.java（Jstorm-server中的cluster下的StormConfig.java）。

Config.java中定义了很多字符串变量，定义了配置参数的位置。

StormConfig.java中说明了storm本体目录下nimbus、supervisor、worker的存储内容：

![Nimbus1](/images/jstorm/image011.png)

![supervisor1](/images/jstorm/image012.png)

![stormJar](/images/jstorm/image013.png)

![supervisor2](/images/jstorm/image014.png)

![Worker](/images/jstorm/image015.png)

有一个问题：
Nimbus中的inimbus，作用？
nimbus中还有pids，作用是？
nimbus中还有心跳，作用是？

![Nimbus2](/images/jstorm/image016.png)
						   
									  


