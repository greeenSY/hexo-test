---
layout: post
title: Jstorm的Nimbus HA机制
description: 简单调研了Jstorm的Nimbus HA机制，写在这里给自己留个备份。
category: 大数据
tags: [JStorm,Storm,分布式系统]
---

## storm的nimbus 单点问题

众所周知，在Storm集群系统中，zookeeper和supervisor都是多节点，任意一个zookeeper节点宕机或supervisor节点宕机均不会对系统整体运行造成影响，但 nimbus和ui都是单节点 。ui的单节点对系统的稳定运行没有影响，仅提供storm-ui页面展示统计信息。但nimbus承载了集群的许多工作，如果nimbus单节点宕机，将会使系统整体的稳定运行造成极大风险。因此解决nimbus的单点问题，将会更加完善storm集群的稳定性。

虽然Nimbus在设计上是无状态的，进程在挂掉之后立即重启并不会对Storm集群产生太大的影响，但如果在Nimbus挂掉期间再有一个Supervisor节点出现问题，那么Supervisor所负责的任务将无法分配到其它的节点，集群也将处于不稳定的状态。

因此解决Storm集群的Nimbus单点问题也是很必要的。

## HA机制

HA机制指的是双机主备模式，如果主机出现宕机的情况，备用机会顶替上来，从而使得整个集群继续工作。

但Storm本身并不支持HA模式，但是在Jstorm中Nimbus 实现HA：当一台nimbus挂了，自动热切到备份nimbus。

## Jstorm的HA实现

在Jstorm中我们可以启动任意多个Nimbus，Nimbus进程启动后即通过抢占zookeeper的InterProcessMutex锁来竞争成为leader，没有抢到锁的非leader Nimbus进程一直处于block状态，不进行后续工作，当leader宕机时，抢占到锁的下一个Nimbus节点成为新leader，由此解决了Nimbus的单节点问题。

（之前没有调研，我本来单纯的认为Nimbus是配合zookeeper中的心跳来查看nimbus的状态和启动备用nimbus的。。。。）

下面我们看一下Nimbus启动时的源码：

 ![NimbusHA](/images/jstorm/NimbusHA.png)
 
Jstorm的HA机制基本就是这样。


