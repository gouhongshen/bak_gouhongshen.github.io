---
title: 'TiDB: A Raft-based HTAP Database'
date: 2022-09-06 11:34:09
tags:
---

## TiDB 要解决什么问题？
关系型数据库管理系统因提供了关系模型、强事务支持和良好的 SQL 接口而广受欢迎，在如银行业等传统应用场景用得很多。然而以前的关系型数据库不支持拓展性（伸缩性）和高可用性，这时 NoSQL 就应运而生，如 BigTable、DynamoDB 和 Redis. NoSQL 放松了对一致性的要求（事务）而提供了高拓展性和可选的数据模型（如 key-val、图和文本）。那么有没有可能把关系型数据库和 NoSQL 的优点结合在一起呢，同时提供强事务保证、高拓展性和可用性？NewSQL 就在干这件事，如 CockroachDB 和 Spanner.
> NoSQL (not only sql) 一般由以下几种驱动因素：
> * 比关系型数据库更好的伸缩性，包括非常大的数据集或非常高的写吞吐量。
> * 关系模型不能很好的支持一些特殊的查询操作。

> **对于第一点**，可以参考文章：{% post_link 关系型数据库拓展性解决方案 %}。**对于第二点**，考虑存储一份简历。一个人可能有多段求学经历、多段工作经历和一些其他信息。在关系型数据库中，可能需要建立多张表：求学经历表，存储每一个用户的求学经历；工作经历表，存储每一个用户的工作信息；用户表，存储用户的基本信息等。那么当查询某个用户的简历时就需要查询多个表，做一些 join 操作，并发度和性能就有所损失，而像 MongoDB 这类文档数据库可以直接存储 Json、XML 等数据，当要访问简历信息时，将文档存储在一处的局部性就会带来性能优势。另外，关系数据模型将数据存储在关系表中，那么就需要一个转换层将表、行、列等转换成应用程序中的对象模型。（当然了，现阶段的 SQL 标准也增加了对结构化数据和 XML 数据的支持，可以将多值数据存储在单行内，并支持对它们的查询和索引。）

<br>

无论是单机关系型数据库、NoSQL 还是 NewSQL 它们都属于 OLTP（online transaction processing）。用户通过 OLTP 创造了数据，这些数据可能蕴含者巨大的价值，需要进行挖掘。OLAP（online analytical processing）系统就是用来发掘数据价值的。OLTP 所产生的业务数据分散在不同的业务系统中，而 OLAP 往往需要将不同的业务数据集中到一起进行统一综合的分析，这时候就需要根据业务分析需求做对应的数据清洗后存储在数据仓库中，然后由数据仓库来统一提供 OLAP 分析。所以我们常说 OLTP 是数据库的应用，OLAP 是数据仓库的应用。可以将 OLAP 看作是 OLTP 的一种延展，一个让 OLTP 产生的数据发现价值的过程。

OLTP 和 OLAP 这样搭配起来各干各的或看起来不错，但也存在一些问题，比如数据转换需要一些其他的组件（如ETL），需要花精力维护，转换过程持续时间较长，而且数据都是定期成批次地进行转换（量大），OLTP 性能是否会受到影响也需要考虑，此外在**某些**应用中，数据还需要实时分析。所以 HTAP（OLTP & OLAP）系统就产生了，当然了不能仅仅是缝合在一起，还有新的要求：
* 数据新鲜度保证（freshness）。及时分析新数据蕴含着巨大商业价值（如推荐）。
* 性能隔离性保证（isolation）。隔离性表示两个系统同时工作时，需要保证两类系统的性能不会相互影响（或影响很小）。

<br>

总计一下，TiDB 做了啥：
* 保证 OLTP 和 OLAP 负载不会相互干扰。
* OLAP 请求能够读到足够新的，和 OLTP 具有一致性的数据。
* 由于数据是一致的，就能够在 SQL 层（优化器）将不同的语句交由 OLTP 或 OLAP 处理。

<br>

虽然 TiDB 融合了 OLTP 和 OLAP，但采用了两套存储引擎（TiKV 和 TiFlash）来分别支持事务和分析请求。TiKV 是行式存储（row-format），用于存储事务处理中的数据，TiFlash 是列式存储（column-format），用于给分析型请求提供支持。TiDB 整个架构如下：
![](/img/TiDB-Paper/2.jpg)
其中 PD 相当于 Master 集群，主要存储数据分片元数据、负载均衡、分配时间戳等。TiSpark 是用于连接 Hadoop 生态的组件。

**这里先说一下 TiDB 是如何做到 HTAP 功能的**：
> 在 TiKV 中数据是分片存储的，每一个分片是一段连续的 key 空间，称之为 region. 每一个 region 又包含多个副本，副本之间的数据同步采用 raft 协议，因此整个系统包含多个 region，也就是多个 raft 组，这又称之为 multi-raft.

> 为了保证 OLTP 和 OLAP 不会相互干扰，一般这两种类型的查询执行在不同的机器上，最主要的困难还是如何从 OLTP 中拿到足够新的数据副本给 OLAP，而且还得保证这些副本数据是一致的（当然这样也保证了高可用性）。因此啊，TiDB 在 raft 中引入了一个新的角色：learner. 它只从 leader 那里复制日志，不参与选举、不参与投票（也就是不算做大多数），因此这个日志复制是异步的。在 TiFlash 中会为 TiKV 中的每一个 region 设置一个 learner. learner 从 leader 复制过来的日志会在 TiFlash 中重放处理后，再转换成列式存储在 TiFlash 中供分析型查询使用。

> 由于复制的时间很短，能够保证新鲜度，raft 协议也能保证数据的一致性。在不同的存储引擎上执行不同类型的查询，TP 和 AP 的性能也不会相互干扰。TiKV 与 TiFlash 关系如下图，可以看到 TiKV 中的多个 region 中的数据可能存储在 TiFlash 中的一个 partition 中。
> ![](/img/TiDB-Paper/3.jpg)


上面的描述看起来好像 TiDB 的创新也不是很多嘛，这里总结下（论文）：
> * 构建了一个基于 raft 的开源 HTAP，为 HTAP 负载提供了高可用性、一致性
伸缩性、AP 数据实时性和 AP/TP 性能的隔离性。
> * 为 raft 协议引入了一个 learner 角色，为 AP 查询提供实时的列式存储数据。
> * 实现 multi-raft，并优化了其读写性能。
> * 实现了一个 SQL 引擎层，能够依据不同的 HTAP 查询选择合适的存储模式。
> * 大量的对比实验。

<br>

其实更多的还是工程实现上的困难：
> 1. 首先是 raft 和 multi-raft 相关问题：① 如何优化 raft 流程提升读写请求性能；② 分片数据的迁移、分裂、合并等。
> 2. 如何及时地（低延迟）将日志同步到 learner 以保证新鲜度？正在进行的事务可能会生成一些非常大的日志，这些日志需要在 learner 中快速重放和实例化（materialization），以便分析型查询读取新数据。将日志数据转换为列式可能会由于模式（schema）不匹配而遇到错误，这同样可能会延迟日志同步。
> 3. 同时进行 TP 和 AP 时如何保证各种的性能？大型 TP 需要读取写入多个服务器上的大量数据。AP 同样也消耗大量资源。为了减少执行开销，SQL 层还需要为这两类工作制定合适的执行计划，选择合适的存储引擎。

<br>

就像论文中说的那样，TiDB 提供了一种思路将 NewSQL 进化为 HTAP. 整篇论文都在将怎么进化的，效果怎么样。 


## TiDB 中的 Raft 算法
基本的 raft 算法流程大概是下面这样的：
> 1. 一个 region leader 收到来自 SQL 层的请求。
> 2. leader 将请求追加到自己的日志中（包括写磁盘）。
> 3. leader 将这些新的日志条目发送到 follower（并行），它们也将这些条目追加到自己的日志中。
> 4. leader 等待 follower 回复，一旦超过大多数（包括自己）回复成功，leader 就会提交这些日志，并将它们应用到复制状态机中。
> 5. leader 将应用请求的结果返回客户端（SQL 层）并继续提供服务。

上述过程最大的问题是性能不高，所有步骤都是串行执行的，因此在 TiDB 中有如下的优化：
* 步骤 2 和 3 可以并行。这两个过程没有依赖关系，因此 leader 可以在追加日志的同时将日志发送给 follower. 如果 leader 持久化日志失败了，但大多数 follower 成功复制了日志，依然满足大多数原则，日志依旧可以被提交。
* 另外，步骤 3 中 leader 给 follower 发送日志时可以按批次发送。而且也不用等待某个 follower 回复成功后再发送后面的，而是假设上次成功了，修改 nextIdx，继续发送后面的日志。如果这个 follower 失败了，调整 nextIdx 即可。
* 步骤 4 中的 apply 也可以异步来做。当日志提交后，就可以用另外一个线程来 apply 这些提交了的日志。

优化后，TiDB 中 raft 算法流程大致如下：
> 1. leader 收到来自 SQL 层的请求。
> 2. leader 将请求追加到自己的日志中，同时（并行）将这些日志发送给 follower.
> 3. leader 继续接收请求，重复步骤 1，2.
> 4. leader 提交日志，并将提交的日志发送给另外一个线程 apply.
> 5. apply 后，leader 返回结果给客户端（SQL 层）。

<br>

另外还有读优化。TiDB 实现了 readIndex、lease Read 和 follower read. 


## TiDB 的事务处理
TiDB 在事务中实现了悲观锁和乐观锁。**乐观事务的流程大致如下**：
> 1. 从客户端接收到 "begin" 命令后，SQL 层向 PD 申请一个时间戳作为本次事务的开始时间戳 start_ts。
> 2. 对于每一条来自客户端的 DML，SQL 层从 TiKV 中读取 [0, start_ts] 区间内最新的数据，然后在事务本地内存中操作数据。
> 3. 当 SQL 层收到客户端的 "commit" 命令后开启 2PC 流程。它先随机地选择一个 key 作为 primary key，然后锁住所有的 key（锁住 keys 是为了保证事务的原子性），最后发送 prewrites 给 TiKV. 因为会牵扯到很多的 key，它们可能存在不同的节点上，所有发送 prewrites 之前，需要向 PD 查询路由信息，然后分组发送.
> 4. 当所有的 prewrite 都成功后，SQL 层向 PD 申请一个时间戳作为本次事务的提交时间戳 commit_ts. 然后 SQL 层发送命令给 TiKV 提交 primary key.
> 5. primary key 提交成功后，SQL 层即可向客户端回复。
> 6. SQL 层异步提交其他的 key，并清理锁。

> DML = Data Manipulation Language，包括 insert、update、delete、select.

关于乐观事务更加详细的描述可以参考文章: [TiDB 乐观事务模型](https://docs.pingcap.com/zh/tidb/dev/optimistic-transaction)。这个过程和 percolator 大致一样，不过 TiDB 是异步提交 secondary keys. 乐观事务实现了快照隔离。

**悲观事务模型**和乐观事务模型相差不大，最大的区别在于加锁的时机。一般来说，乐观模型是一种验证模型，待所有的更新操作完成后，再验证这些更新能否写入数据库；而悲观则是先取得要更新数据的锁，再更新数据。TiDB 的悲观模型有点不一样，可以参考文章：[TiDB 悲观事务模式](https://docs.pingcap.com/zh/tidb/dev/pessimistic-transaction)。在这个模型中实现了可重复读（RR）和读已提交（RC）隔离级别。


关于 PD（placement driver）的运作机理，可以参考这篇文章：[TiDB 6.0：让 TSO 更高效](https://cn.pingcap.com/blog/tidb-6.0-make-tso-more-efficient)


## TiDB 中的 TiFlash
待续。。。

## TiDB 的性能
![](/img/TiDB-Paper/4.jpg)
![](/img/TiDB-Paper/5.jpg)
![](/img/TiDB-Paper/6.jpg)
> TC = transaction client, AC = analysis client. 
> TPS = Transactions Per Second，指一台服务器每秒能够处理的事务数量. 
> QPS = Queries Per Second，指一台服务器每秒能够响应的查询次数.
