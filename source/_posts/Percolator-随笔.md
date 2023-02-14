---
title: 事务篇六：Percolator 随笔
date: 2022-06-18 13:33:27
tags:
- 分布式事务
- 事务
categories:
- [分布式]
- [事务]
math: true
sticky: 1

# index_img: /img/percolator/percolator_cover.png
---

## 前言
[percolator 论文](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf)
建议先阅读事务并发 snapshot isolation 部分: {% post_link 事务的并发控制 %}

percolator 这个系统是怎么产生的呢？论文中介绍了下背景，谷歌的爬虫应用会周期性的爬取新网页，同时更新网页的索引、排名等信息，这些更新操作往往并发，在庞大的数据之上做一些小的、独立的更改，而且修改跨越多个行、多个表。谷歌现有的系统中，MapReduce 也能完成相同的工作，但由于设计理念，MapReduce 需要扫描整个数据库才能完成这些更新；而且 BigTable 也只支持单行事务，不能提供上面所说的需求，所以就在 BigTable 之上构建了 percolator.

在极其庞大的数据集之上（谷歌是数十PB），每次高效地完成少量的更新被称为**增量处理（incremental processing）**，percolator 便是完成这样的工作。percolator 每次采用事务的方式执行这样的更新，总的来说它采用两阶段提交协议（是对传统两阶段协议的优化，见 {% post_link 两阶段提交协议 %}）。

<font color=red>percolator 包含很多的内容，本文主要集中在事务系统的设计，主要是快照隔离的实现</font>。

## 事务系统的设计
snapshot isolation 有两个阶段：
1. <u>本地阶段</u>，读取在 [0, startTS] 时间内提交的最新的 snapshot，即开始时间戳之前提交的，并在事务的本地空间对数据进行操作。
2. <u>提交阶段（两阶段提交）</u>，该阶段需要先验证是否有冲突产生，snapshot isolation 要求不能有 write-write conflict.

要实现这两个阶段，不仅需要依据时间戳读取数据，还需要辨别出并发事务，percolator 是怎么做的呢？它通过记录一些列信息做到：
* data column. 实际的数据将记录在该列，格式一般为 $startTS(T_i) + data$.
* lock column. 当某个数据处于提交阶段时，更新还不可见，所以记录 lock 信息在该列表示不可见，格式一般为 $startTS(T_i) + point\\_to\\_data$.
* write column. 当写入一条 write 数据在该列时，表示某个数据已提交，其他事务可见，格式一般为 $commitTS(T_i) + point\\_to\\_data$.

下面分别针对读和写两个操作介绍它们是如何工作的：

**当事务 $T_i$ 读取数据时**，它先检查同一数据在 [0, startTS($T_i$)] 区间内是否存在 lock 记录，若存在，说明存在某个与 $T_i$ 并发的事务正在修改该数据，这时，$T_i$ 等待该事务完成。当锁不存在后，$T_i$ 就在 write column 中寻找 [0, startTS($T_i$)] 区间内同一数据最新的 write record，最后依据该记录读取到真正的数据。

<font color=red>这里有一些想不明白，为何 $T_i$ 需要等？</font>该事务只能看到其开始之前提交的数据，很明显，上面占据写锁的事务所做的更新对 $T_i$ 是不可见的（提交在 $T_i$ 开始之后），既然如此，为何 $T_i$ 还必须等待呢？

<br>

**写操作相对复杂一些，因为它涉及提交过程**。percolator 采用两阶段提交事务：
<u>第一阶段称为 Prewrite.</u> 在协调节点提交事务 $T_i$ 前，先获得所有被修改数据的写锁（通过写入 lock 记录实现）。这其实相当于验证阶段，查找冲突是这一阶段的重要过程。冲突来自于与之并发的其他更新事务（假设为 $T_j$），存在两种冲突，如图片所示:
![冲突示意](/img/percolator/conflict.png)
1. 并发的事务已完成提交。即提交时间（write record 的时间戳）在当前事务开始时间戳之后。在 percolator 中，若当前事务检测到一条待修改数据的 write 记录，其时间戳在当前事务开始时间戳之后，那么当前事务 abort.
2. 并发的事务正在提交。即并发事务已经对数据加锁了，锁的时间戳为事务的开始时间戳。在 percolator 中，若当前事务检测到一条修待改数据的 lock 记录，不论其时间戳为多少，当前事务都 abort.

对需要修改的每一个数据，当事务 $T_i$ 遇到这两种冲突时会 abort，即满足了 snapshot isolation 对避免 write-write conflict 的保证。如果没有冲突发生，事务 $T_i$ 会结合事务开始时间戳 startTS($T_i$) 分别将修改的数据（data）和锁记录（lock）写入相应的列（这是一个原子操作，通过 BigTable 的单行事务完成）。

<u>待所有数据的 lock 和 data 记录写入后，将来到第二个阶段</u>. 在该阶段，事务首先获取一个提交时间戳 commitTS($T_i$)，然后对修改的每一个数据，移除其 lock 记录，同时写入 write 记录（同样是一个原子操作），这时，该更新就对外部可见了。这里要注意下，将 lock 记录改写成 write 前要先检查以下 lock 记录还在不在，原因见后面的 [可用性考虑中的 roll forward](#percolator-对可用性的考虑).

以一个例子阐述上述过程，Bob 和 Joe 账户初始分别有 10 美元和 2 美元，一个事务从 Bob 账户转 7 美元到 Joe 账户。下面几幅图展示了修改过程（加粗表示新写入的记录，x:data 表示 data 是在时间戳为 x 时写入的，primary 后面会提到）：

![example_a](/img/percolator/example_1.jpg)
![example_b](/img/percolator/example_2.jpg)
![example_c](/img/percolator/example_3.jpg)
![example_d](/img/percolator/example_4.jpg)
![example_e](/img/percolator/example_5.jpg)



## percolator 对可用性的考虑
<font color=red>注意传统 2PC 的 blocking 问题在这里是如何解决的。</font>

上面提到事务准备提交前会对数据加锁（写入 lock 记录），这些锁分布在不同的行、不同的表、甚至不同的节点。假如事务在提交过程中系统发生故障，只释放了部分的锁（将 lock 记录改写成 write 记录）或者只写入了部分的 lock 记录，那么就可能阻塞后面新的事务，如果加锁的是热点数据，那系统的可用性就会大大降低，甚至不可用。另外，如果由系统来贸然释放这些遗留的锁，则可能造成数据的不一致，因为事务可能只提交了部分更新。因此锁必须是可恢复的（保存在 stable storage），而且还必须判断这些锁能不能安全的释放。percolator 提出了如下的解决方法。

锁是通过 lock 记录写入 BigTable 中的，能提供 stable storage 的功能。同时，在获取写锁时，事务会指定一行（某个数据）的锁为主锁（primary，如 example_b），其他锁都会包含一个指针指向主锁。事务提交释放锁时，先从主锁开始释放所有锁。

若一新事务准备提交时发现自己欲修改的数据已经被加锁了，**可以猜想该锁是系统崩溃遗留下来的**，此时它有两种选择：
* <u>帮助事务提交（roll forward）</u>。若持有该锁的事务在系统崩溃前已经提交了部分的更新，那么为维护数据的一致性，剩下的更新也应当被提交（其实数据已经写入 data 列了，只不过锁还在，新事务要做的不过是移除该锁，写入 write 记录，使该更新可见）。
* <u>回滚该事务（roll back）</u>。若持有该锁的事务在崩溃时还未提交，那么需要撤销事务所做的更新（删除写入 data 列的数据和 lock 记录）。

听起来是那么回事，但有两个问题还需要解决：
1. 如何判断事务是否已提交了部分更新？
2. 如何判断该锁真的是系统崩溃后遗留的？

**对于第一个问题**（percolator 论文中没有提到 log 文件），primary lock 发挥了重要作用。因为释放锁总是先释放 primary lock，若 primary lock 已经不存在了，说明至少部分更新已经对外可见了，因此该事务必须被完全提交，所以需要 roll forward. 若 primary lock 依旧存在，说明该事务还未提交，roll back 也不会产生什么副作用，<font color=red>所以该 primary lock 是一个同步点（synchronize point）</font>。因为这一点，当事务释放锁时，**应当检查其是否还持有主锁**，看看自己是否已经被 roll back 了。

如果锁不是崩溃后留下来的，贸然 roll back 则会影响系统的性能。percolator 中，是由一系列 worker 执行事务，**对于第二个问题**，所以当新事务遇到锁时，发现该锁属于不活跃的 worker（dead or stuck）时，才会做如上猜想，否则 abort 或者等待。如何判断 worker 不活跃呢？论文中原文如下（偷个懒，不想写了）：
> ...So, a transaction will not clean up a lock unless it suspects that a lock belongs to a dead or stuck worker. Percolator uses simple mechanisms to determine the liveness of another transaction. Running workers write a token into the Chubby lockservice [8] to indicate they belong to the system; other workers can use the existence of this token as a sign that the worker is alive (the token is automatically deleted when the process exits). To handle a worker that is live, but not working, we additionally write the wall time into the lock; a lock that contains a too-old wall time will be cleaned up even if the worker’s liveness token is valid. To handle long- running commit operations, workers periodically update this wall time while committing.


## 总结
论文中没有提到，似乎 percolator 不能避免 read-write conflict.

percolator 论文中还包含很多其他内容，比如通知服务、时间戳、相关工作等等。本文主要讨论其事务的实现。













