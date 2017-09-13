---
layout: post
title: Jstorm源码分析——Jstorm的启动过程
description: 分析Jstorm的启动过程。
category: 大数据
tags: [JStorm,Storm,分布式系统]
---
## Jstorm的启动命令

与Storm类似，Jstorm的启动过程需要两个部分：

* 在nimbus 节点上执行 “nohup jstorm nimbus &” 
* 在supervisor节点上执行 “nohup jstorm supervisor &”

bin/Jstorm是一个python写的脚本，是Jstorm的程序入口，Jstorm的启动以及Topology的提交等过程都需要用到这个脚本。

## Nimbus

首先我们先看nimbus的启动。在bin/Jstorm脚本中，通过jstorm nimbus调用将触发脚本中的nimbus()方法，其中调用的是"com.alibaba.jstorm.daemon.nimbus.NimbusServer"。

NimbusServer的工作包括：

* 清除中断的Topology（删除本地目录/storm-local-dir/nimbus/topologyid/stormdis和zk上的/storm-zk-root/storms/topologyid）
* 设置/storm-zk-root/storms/topology中的Topology状态为active
* 启动一个monitor线程，每nimbus.monitor.reeq.secs检查/storm-zk-root/storms中所有Topology状态，如果Topology中有task是不活动的则讲Topology状态转换为monitor（这个状态下nimbus会重新分配workers）
* 启动一个cleaner线程，每nimubs.cleanup.inbox.freq.secs清除无用的jar

NimbusServer启动的时候，它首先创建一个nimbusServer的对象，并调用其launchServer(config,iNimbus)启动NimbusServer。
 
![NimbusServer](/images/jstorm/image017.png)
 
在launchServer中，它需要创建本地pids目录（createPid(conf)），初始化关闭nimbus的hook（initShutdownHook()，其实是启动线程调用cleanup()），准备nimbus工作槽（inimbus.prepare(conf, StormConfig.masterInimbus(conf))），创建NimbusData（com.alibaba.jstorm.daemon.nimbus.NimbusData，所有nimbus的数据），初始化followerTread（FollowerTread线程是com.alibaba.jstorm.schedule.FollowerRunnable，作用是定时去检查更新nimbus的工作情况，并且必要时替换新的nimbus），启动Httpserver，初始化initContainerHBThread（心跳HeartBeat相关），然后等待自己成为leader（在NimbusData中的tryTobeLeader()方法中会改变Leader的情况），如果是则载入配置（init(conf)），否则持续等待。
 
![LaunchServer](/images/jstorm/image018.png)
 
在init(Map conf)中，会初始化initTopologyAssign()、initTopologyStatus()（前者分配Topology并设置状态为active，后者将active转换为startup但是什么都不做），同时初始化Monitor线程（initMonitor(conf)）、Cleaner线程（initCleaner(conf)）分别承担Monitor和clean的工作，初始化一个serviceHandler对象，它会接受客户端传递过来的消息并调用相关方法进行Topology的提交等工作（见5.1），最后如果不是本地模式则还需要初始化分组（initGroup(conf)）和Thrift（initThrift(conf)）。
 
![init](/images/jstorm/image019.png)
 
## Supervisor

在bin/Jstorm脚本中，通过jstorm nimbus调用将触发脚本中的Supervisor()方法，其中调用的是" com.alibaba.jstorm.daemon.supervisor.Supervisor"。

Supervisor是一个持续运行的主控线程（见com.alibaba.jstorm.daemon.supervisor下Supervisor.java），Supervisor的工作有：

* 向zookeeper中写入Supervisor信息
* 每supervisor.monitor.frequency.secs运行一次SynchronizeSupervisor，其中这个线程先下载新的Topology，而后释放没有用的worker，然后分配新的task到localstate，最后添加一个syncProcesses到队列中
* syncProcesses是具体的执行动作，它杀掉无用的worker，开始新的worker
* 创建心跳线程，每supervisor.heartbeat.frequency.secs向zookeeper中写入一次信息。

该线程启动的时候：

* 调用mkSupervisor，启动一个supervisorManager。其中，首先清空本地/storm-local-dir/supervisor/tmp的文件，建立zk集群状态、localStat，获取Supervisor id，创建线程队列，创建心跳线程HeartBeat，创建SyncSupervisorEvent线程（用于从Zookeeper获取Event，详见“第6章Topology的提交与实例化过程”），创建和开始httpserver线程，最后返回一个SupervisorManager对象。这里注意，用到一个AsyncLoopThread（和AsyncLoopRunnable），这是一个特殊的一个线程，根据回调函数的返回值决定线程休眠时间，或者根据异常结束线程。
* Supervisor线程每1s查看一次SupervisorManager状态是否已经结束。Supervisor线程是一个主控线程，主要查看SupervisorManager状态的状态。具体的处理功能都由它所创建的其他线程完成。比如SyncSupervisorEvent线程每supervisor.monitor.frequency.secs s就运行一次（EventManager每隔一段时间加入一个event到队列中，同时其不断从队列里取出event进行处理）。

![Supervisor](/images/jstorm/image020.png)



