---
title: ElasticSearch 实战（六） 
tags: [ElasticSearch,源码]
date: 2018-09-25 09:16:58
categories: ElasticSearch
---



> 真正的胜利不是打败强大的对手，而是守护自己最重要的东西。 ——杨鑫

### 对于ES索引详解：

ES中有两个库：

- 反向索引库
- 文档库
  - 文档的结构
  - _index
    - _type
      - _id
      - _source
      - _version

一般更新索引的时候我们先把反向索引库删除，

POST、同样的更新，相同的更新，我们再次操作，返回的结果：noop，主要原理就是根据id、source、version

PUT、对于同样的操作，此时就会更新，添加版本号

### 重索引：

> 将一个索引中的数据重索引到另一个索引中，要求源索引中的source是开启的，目标索引的setting、mapping与源索引没有关系。

重索引一个问题，目标索引中存在源索引中的数据，这些数据的version如何处理：

1. 如果没有指定version_type 或 指定为 internal ，则会是采用目标索引中的版本，重索引过程中，执行就是新增、更新操作。
2. 如果想使用源索引中的版本来进行版本控制更新，则设置version_type为external。重索引操作将写入不存在的，更新旧版本的数据。
3. 如果只想从源索引中复制目标索引中不存在的文档数据，可以指定op_type为create，此时存在的文档将处罚版本冲突，可设置conflicts：proceed，跳过继续。
4. 同时可以只索引源索引中的一部分数据，通过type或查询来指定需要的数据
5. 可以限定文档的数量、文档的字段

### 刷新

对于索引、更新、删除操作如果想操作完后立马重刷新可见，可带上refresh参数。

- 未给值或true，则立马会重刷新读索引
- false、相当于没有带fresh参数，遵循内部的定时刷新
- wait_for，登记等待刷新，当登记的请求数到达 index.max_fresh_listeners 参数设定的值时（默认为1000）将触发重刷新。

## 路由详解

### 集群的工作原理

#### 节点启动过程

- 当启动一个主节点的时候，默认节点中存在集群元信息，包括集群名称、集群的节点信息。
- 当我们启动第二个节点的时候，配置 `discovery.zen.ping.unicast.hosts: [10.0.0.1, 10.0.0.2]` ，找到主节点，加入到节点中，之后更新节点，集群中每个节点都有所有集群所有节点信息，这个同步是由主节点同步的。

#### 创建一个索引

- 请求转发给请求master节点
- 主节点选择节点存放分片，副本，记录元信息
- 通知给参与存放索引分片、副本的节点从节点，创建分片，副本
- 参与节点向主节点反馈结果
- 等待时间到了，master向node3反馈结果信息，node3响应请求
- 主节点将元信息广播给所有从节点

#### 索引一个文档

- node2计算文档的路优质得到文档存放的分片（假设路由选定的是分片0）
- 将文档转发给分片0的主分片节点node1
- node1 索引文档，同步给副本节点node3索引文档
- node1 向 node2 反馈结果
- node2 作出响应

#### 文档路由规则

- 决定了文档应该存放到那个分片上，计算公式如下：`shard = hash(routing) % number_of_primary_shards`
- routing 是用来进行hash计算的路由值，默认是使用文档id值。
- 在索引、删除、更新、查询中都可以使用routing参数指定操作的分片。

#### 请求一个文档

- node2 解析查询
- node2 将查询发给索引 s1 的分片、副本（R1，R2，R0）节点
- 各节点执行查询，将结果发给 node2
- node2 合并结果，做出相应

#### 思考

> 关系数据库中有分区表，通过选定分区，可以降低操作的数据量，提高效率。在ES的索引中能不嫩这样做？

可以：通过制定路由值，让一个分片上存放一个区的数据，如果按部门存放数据，则可制定路由值为部门值

Master 节点的工作：主要就是创建索引的时候，需要Master进行同步元数据信息。