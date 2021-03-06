---
title: ElasticSearch-ES集群管理（十一）
author: HoldDie
tags: [ElasticSearch,搜索,聚合分析]
top: false
date: 2018-10-04 19:43:41
categories: ElasticSearch
---



> 利剑虽强却斩不断流水，微风虽弱却能平息海浪。 ——子房
>

## ES集群管理

### 集群规划

#### 多大规模的集群？

当前数据量有多大？数据增长情况如何？

你的机器配置如何？CPU、内存、硬盘

ES小常识:

- ES JVM heap 最大 32G
- 30G Heap 大概能处理的数据量10T，如果内存很大128G，可以在一台机器上运行多个ES实例。
- 集群规划满足当前数据规模+适量增长规模即可，后续可按需扩展。

两类应用场景：

- 用于构建业务搜索功能模块，且是多个垂直领域的搜索。数据量级几千万到数十亿级别。一般2~4台机器的规模。
- 用于大规模数据的实时OLAP（联机处理分析），经典的ELK stack，数据规模可能达到千亿或者更多。几十到上百节点的规模。

#### 集群中的节点角色如何分配？

节点角色：

- Master
  - `node.master: true`
- DataNode
  - `node.date:true`
- Coordinate Node
  - 协调节点，如果担任协调节点、将上两个配置设为false

角色如何分配

- 小规模集群，不需严格区分
- 中大规模集群，应考虑单独角色充当，不会应该因协调角色负载过高而影响数据节点的能力。

#### 如何避免脑裂问题？

原因：

- 网络原因，阻塞
- 主节点繁忙，对外无法提供服务

尽量避免脑裂，配置：

discovery.zen.minimum_master_nodes：（有master资格节点数/2）+1

常用做法：

- Master 和 DataNode 角色分开，配置奇数个 master
- 单播发现配置，配置master资格节点
  - `discovery.zen.ping.multicast.enabled:false`
  - `discovery.zen.ping.unicast.hosts:["master1","master2","master3"]`
- 配置选举发现数，及时延长Ping master 的等待时长
  - `discovery.zen.ping_timeout:30`
  - `discovery.zen.minimum_master_nodes:2`

#### 索引应该设置多少个分片？

> 分片数指定后不可变，除非重索引。

1、分片对应的存储实体是什么？

索引，

2、分片是不是越多越好？

分片过多的影响：

- 每个分片本质上就是一个Lucene索引，因此会消耗相应的文件句柄，内存和CPU资源。
- 每个请求会调度到索引的每个分片中，如果分片分散在不同的节点到时问题不大，但当分片开始竞争相同的硬件资源时，性能便会逐步下降。
- ES使用词频统计来计算相关性，当然这些统计也会分配到各个分片上，如果大量分片上只维护了很少的数据，则将导致最终的文档相关性较差。

索引的分片设置参考原则：

- ES 推荐最大JVM为30~32G，所以单分片最大容量设置30G，然后再对分片数量做合理估算。
- 在开始阶段，一个好的方案根据你的节点数量按照1.5~3倍的原则来创建。若你有三个节点，推荐你创建的分片数最多不超过9个。当性能下降时，增加节点，ES会平衡分片的放置。
- 给予日期的索引需求，并且对索引数据的搜索场景非常少，也许将这些索引量到达成百上千，每个索引的数量只有1GB甚至更小，对于这种类似场景，建议只有一个分片

#### 分片应该设置几个副本？

1、副本的用途？

备份数据

2、针对它的用途，我们该如何设置它的副本数？

- ES 副本数指的就是数据的副副本数，
- 当我们的副本数为2时，默认主副本数为1，副副本数为2，总共有3个副本。

3、集群规模没变的情况下，副本过多有什么影响？

- 副本数可以随时调整的
- 浪费空间

4、副本设计原则

- 为了保证高可用，副本数设置为2即可，要求集群至少有3个节点，来分开存放主分片，副本。
- 发现并发量大时，查询性能会下降，可增加副本数，来提升并发查询能力。（前提是增加机器数）

### 集群的管理

监控API

可视化（x-pack）

为集群提供安全防护、监控、告警、报告等核心功能