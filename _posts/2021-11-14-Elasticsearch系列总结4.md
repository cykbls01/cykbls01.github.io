---
layout:     post
title:      Elasticsearch系列总结4
subtitle:   Elasticsearch进阶知识
date:       2021-11-14
author:     Yikun Chen
header-img: img/background/header.png
catalog: true
tags:
    - elasticsearch

---


# Elasticsearch系列总结4

Elasticsearch进阶知识
--

## Es集群

### 节点类型

Master节点

在Elasticsearch启动时，会选举出来一个Master节点。当某个节点启动后，然后使用Zen Discovery机制找到集群中的其他节点，并建立连接。Master节点主要负责管理索引，分配分片，维护元数据，管理集群节点状态，不负责数据写入和查询，比较轻量级。一个Elasticsearch集群中，只有一个Master节点。在生产环境中，内存可以相对小一点，但机器要稳定。

DataNode节点

在Elasticsearch集群中，会有N个DataNode节点。DataNode节点主要负责，数据写入、数据检索，大部分Elasticsearch的压力都在DataNode节点上。在生产环境中，内存最好配置大一些。

### 分片和副本

分片
一个索引可以存储超出单个结点硬件限制的大量数据。比如，一个具有10亿文档的索引占据1TB的磁盘空间，而任一节点都没有这样大的磁盘空间；或者单个节点处理搜索请求，响应太慢。为了解决这个问题，Elasticsearch提供了将索引划分成多份的能力，这些份就叫做分片。当创建一个索引的时候，可以指定你想要的分片的数量。每个分片本身也是一个功能完善并且独立的“索引”，这个“索引”可以被放置到集群中的任何节点上。

副本
在一个网络/云的环境里，失败随时都可能发生，在某个分片/节点不知怎么的就处于离线状态，或者由于任何原因消失了，这种情况下，有一个故障转移机制是非常有用并且是强烈推荐的。为此目的，Elasticsearch允许你创建分片的一份或多份拷贝，这些拷贝叫做副本分片，或者直接叫副本。因为搜索可以在所有的副本上并行运行，所以可以扩展搜索量。分片和副本的数量可以在索引创建的时候指定。在索引创建之后，可以在任何时候动态地改变副本的数量，但是不能改变分片的数量。

## 工作流程
### 写入原理

![picture1](/img/elasticsearch/write.png)

选择任意一个DataNode发送请求，例如：node2。此时，node2就成为一个协调节点。
计算得到文档要写入的分片`shard = hash(routing) % number_of_primary_shards`routing 是一个可变值，默认是文档的 _id。
coordinating node会进行路由，将请求转发给对应的primary shard所在的DataNode。
node1节点上的Primary Shard处理请求，写入数据到索引库中，并将数据同步到Replica shard。
Primary Shard和Replica Shard都保存好了文档，返回client。

### 搜索原理

![picture1](/img/elasticsearch/search.png)

client发起查询请求，某个DataNode接收到请求，该DataNode就会成为协调节点。
协调节点将查询请求广播到每一个数据节点，这些数据节点的分片会处理该查询请求。
每个分片进行数据查询，将符合条件的数据放在一个优先队列中，并将这些数据的文档ID、节点信息、分片信息返回给协调节点。
协调节点将所有的结果进行汇总，并进行全局排序。
协调节点向包含这些文档ID的分片发送get请求，对应的分片将文档数据返回给协调节点，最后协调节点将数据返回给客户端。

## 准实时索引实现

### 文件系统缓存

当数据写入到ES分片时，会首先写入到内存中，然后通过内存的buffer生成一个segment，并刷到文件系统缓存中，数据可以被检索。ES中默认1秒，refresh一次。

### 写translog保障容错

在写入到内存中的同时，也会记录translog日志，在refresh期间出现异常，会根据translog来进行数据恢复。等到文件系统缓存中的segment数据都刷到磁盘中，清空translog文件。

### flush到磁盘
ES默认每隔30分钟会将文件系统缓存的数据刷入到磁盘。

###  segment合并
Segment太多时，ES定期会将多个segment合并成为大的segment，减少索引查询时IO开销，此阶段ES会真正的物理删除（之前执行过的delete的数据）。

![picture1](/img/elasticsearch/refresh.png)

## 脑裂问题

脑裂问题是同一个集群中的不同节点，对于集群的状态有了不一样的理解，比如集群中存在两个master
如果因为网络的故障，导致一个集群被划分成了两片，每片都有多个node，以及一个master，那么集群中就出现了两个master了。但是因为master是集群中非常重要的一个角色，主宰了集群状态的维护，以及shard的
分配，因此如果有两个master，可能会导致破坏数据。

该参数的作用，就是告诉es直到有足够的master候选节点时，才可以选举出一个master，否则就不要选举出一个master。这个参数必须被设置为集群中master候选节点的quorum数量，也就是大多数。至于quorum的算法，就是：master候选节点数量 / 2 + 1。

如果我们有三个master候选节点，还有100个数据节点，那么quorum就是3 / 2 + 1 = 2，一个生产环境的es集群，至少要有3个节点，同时将这个参数设置为quorum，也就是2。




参考文献
--

