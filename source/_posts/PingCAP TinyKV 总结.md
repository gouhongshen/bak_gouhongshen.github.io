---
title: PingCAP TinyKV 总结
date: 2022-07-22 14:19:47
tags:
sticky: 1
---

## TiDB

TalentPlan TinyKV 2022 学习营马上结束了，项目也写得差不多了，此篇文章作为一个总结，也不免我两个月的 coding.

本来呢我是想写一篇关于 TiKV（TinyKV 是它的简单版本）的学习记录，但在看 PingCAP 官网的文档时越看越有意思（它们的的文档写得真好，详细），于是决定干脆学习下 TiDB 好了，看看现代数据库都干了什么事。所以这篇文章记录一些 TiDB 学习过程中的东西，基本就是 TiDB 文档的搬运工，汇总一下方便阅读。
> 另外，知乎上这个[如何评价 TiDB](https://www.zhihu.com/question/58767602/answer/1662140715)问题下有很多比较好的回答可以看看。而且还炸出了很大大佬，可以关注下。

<br>

![](/img/tinyKV/2.png)
上图是 TiDB 的架构图，下面单独介绍各个组成部分

### TiDB Server
> 来看看它具体干了什么事情，下面是它的架构图：
> ![](/img/tinyKV/3.png)
> 用户的 SQL 请求会直接或者通过 Load Balancer 发送到 TiDB Server，TiDB Server 会解析 MySQL Protocol Packet，获取请求内容，对 SQL 进行语法解析和语义分析，制定和优化查询计划，执行查询计划并获取和处理数据。数据全部存储在 TiKV 集群中，所以在这个过程中 TiDB Server 需要和 TiKV 交互，获取数据。最后 TiDB Server 需要将查询结果返回给用户。TiDB 层本身是无状态的（不存储数据），实践中可以启动多个 TiDB 实例，通过负载均衡组件（如 LVS、HAProxy 或 F5）对外提供统一的接入地址，客户端的连接可以均匀地分摊在多个 TiDB 实例上以达到负载均衡的效果。

> <u>一个问题</u>：TiDB 是分布式关系型数据库，而 TiKV 是 key-val 存储引擎，TiDB 是如何将库表中的数据映射到 TiKV 中的 key-val 键值对的？[参考文档](https://docs.pingcap.com/zh/tidb/stable/tidb-computing) 


### PD (placement driver) Server
> <u>PD (Placement Driver) Server</u>. 整个 TiDB 集群的元信息管理模块，负责存储每个 TiKV 节点实时的数据分布情况和集群的整体拓扑结构，并为分布式事务分配事务 ID. PD 不仅存储元信息，同时还会根据 TiKV 节点实时上报的数据分布状态，下发数据调度命令给具体的 TiKV 节点（region 分裂/合并/迁移），可以说是整个集群的“大脑”。此外，PD 本身也是由至少 3 个节点构成，拥有高可用的能力。建议部署奇数个 PD 节点。

> TiKV 的每一个 raft group 都是一个 region 的冗余复制集，而 region 数据不断增减时，它的大小也会不断发生变化，因此必须支持 region 的分裂和合并，才能确保 TiKV 能够长时间稳定运行。region split 会将一段包含大量数据的 range 切割成多个小段，并创建新的 raft Group 进行管理，如将 [a, z) 切割成 [a, h), [h, x) 和 [x, z)，并产生两个新的 raft group。region merge 则会将 2 个相邻的 raft group 合并成一个，如 [a, h) 和 [h, x) 合并成 [a, x）。这些逻辑也在 raftstore 模块中实现。这些特殊管理操作也作为一个特殊的写命令走一遍上节所述的 raft propose/commit/apply 流程。为了保证 split/merge 前后的写命令不会落在错误的范围，我们给 region 加了一个版本的概念。每 split 一次，版本加一。假设 region A 合并到 region B，则 B 的版本为 max(versionB, versionA + 1) + 1。更多的细节实现包括各种 corner case 的处理在后续文章中展开。

> 一些参考文档，主要是调度相关，其实是牵扯到 multi-raft 内容:
> * [TiDB 数据库的调度，主要是介绍调度考虑的内容，是一个概述。](https://docs.pingcap.com/zh/tidb/stable/tidb-scheduling)
> * [这篇文章也简述了迁移/分裂/合并的过程，可以一看。](https://docs.pingcap.com/zh/tidb/stable/tikv-overview#region-%E4%B8%8E-raft-%E5%8D%8F%E8%AE%AE)
> * [当一个 region 的数据量太大时，会分裂 region，可以参考这篇文章。](https://pingcap.com/zh/blog/tikv-source-code-reading-20)
> * [当两个相邻的 region 的数据量太小时，会合并它们。](https://pingcap.com/zh/blog/tikv-source-code-reading-21)

### TiKV Server
> <u>TiKV Server</u>. 负责存储数据，从外部看 TiKV 是一个分布式的提供事务的 key-value 存储引擎。存储数据的基本单位是 region，每个 region 负责存储一个 key range（从 startKey 到 endKey 的左闭右开区间）的数据，一个 region 有多个备份，使用 raft 协议实现副本一致性，一个 region 备份会存在某个 TiKV 节点上，一个 TiKV 可能存在多个 region 备份（属于不同的 region）.

> TiKV Server 包含这样几个重要的部分，[一篇概览性文章](https://docs.pingcap.com/zh/tidb/stable/tikv-overview)：
> * 分布式事务的实现，可以参考[文章](https://docs.pingcap.com/zh/tidb/stable/optimistic-transaction)。
> * multi-raft 部分，上面 PD 已经涉及到了。
> * key-val 存储引擎部分。key-val 存储引擎有两个，rocksdb 和 titan. rocksDB 是在 {% post_link "LevelDB 中的 LSM" levelDB %} 基础上开发的，主要的改变应该是 column family，可以参考[文档](https://docs.pingcap.com/zh/tidb/stable/rocksdb-overview). titan 也是老朋友了，其实现和 {% post_link "badgerDB 中的 LSM" badgerDB %} 差不多，可以参考[文档](https://docs.pingcap.com/zh/tidb/stable/titan-overview).
> * 还包括一些计算，是 TiDB Server 下推来的，可以参考 TiDB Server 部分是外链。

### TiFlash
> <u>TiFlash</u>：TiFlash 是一类特殊的存储节点。和普通 TiKV 节点不一样的是，在 TiFlash 内部，数据是以列式的形式进行存储，主要的功能是为分析型的场景加速，它会实时地从 TiKV 拷贝数据以列式存储用于分析。
> 对这个不太熟，有时间可以了解下，参考[文档](https://docs.pingcap.com/zh/tidb/stable/tiflash-overview).


## 总结

TiDB 属于 NewSQL，至于 SQL、NoSQL 和 NewSQL 之间的区别，因为还没有太多的实际生成经验所以也没有太多的体会，只能摘录一些别人的总结（慢慢更新）：
> SQL 一般是指传统的单机关系型数据库，它们的特点是数据存储采用关系型模型，支持事务，能够提低时延和高并发的事务处理，是典型的 OLTP. 如果业务量不大一般这类数据库是第一选择。SQL 的主要缺点在于拓展性，为什么？（后面慢慢补）
> 拓展性的一方面是指数据模型固定，不太支持图片、文档等无结构/半结构的数据类型，于是 NoSQL 出世了？
> NewSQL 又可以叫做分布式关系型数据库，同时继承了传统关系型数据库的事务特性和 NoSQL 的拓展性。

<br>

未整理得参考资料
> [关系型数据库横向扩展的三种方法](https://blog.csdn.net/stevensxiao/article/details/51872795)





