---
title: 查询处理篇：Parallel Join
date: 2022-07-26 17:07:31
tags:
---

在 Join 操作那篇{% post_link "查询处理篇：Join 操作" 文章中 %}，介绍了几种 Join 算法，但有一个假设：参与 join 的两张表都完整地存放在了一处。在{% post_link "数据分片" %}中谈到了系统为了提高水平拓展能力和降低宕机风险会将数据分片存放在若干个节点上，**那么在这种情况下 join 操作又该如何做呢**？本文就讨论这一点。
> 当然了 parallel join 的用处也不只上面提到的。在 shared-memory 架构中，在多个节点上同时执行 join 操作能够极大地减少算法的时间，如果是 shared-nothing 架构，如果参与 join 的表本身没有分片，采用 parallel join 方式需要将（部分）表复制到其他节点，这样带来的加速比不一定比本地多线程来得快。**所以这一篇文章讨论的是数据分片后的 join 方法，也就是假设了所有的的数据都已经分片了**。

<br>

## 等值连接
假设 join 的两张表分别是 $r$ 和 $s$，有 $m$ 个节点，$N_1,N_2,...,N_m$，参与本次的 join. 算法的主要思想是：将这两张表用同一种分块方法，将元组分成 $m$ 份，$r_1-r_m, s_1-s_m$ 分别发送（复制）到 $m$ 个节点，由于是等值连接，join 属性值相等的元组（记录）一定会被发送到同一个节点，然后在节点本地执行 join 操作，最后汇总结果即可。

分块的方式一般有两种：
> 按范围分（range partitioning）
> 使用哈希分（hash partitioning）

<br>

如上所述，这两张表很可能已经被分片存储在不同的节点上了，那么可以采用下面的步骤：
> 每一个存储了表 $r$ 分片的节点从磁盘读取这部分元组（记录），然后使用选定的分块方法将其中的每一条记录都发送到对应的节点，同时该节点开启另外一个线程接收别的节点发送来的记录。对表 $s$ 执行同样的操作，等到两张表都分块完成，每一个参与 join 的节点上都存在两张表的一个分块，这两个小分块的 join 操作就和{% post_link "查询处理篇：Join 操作" %}中的情况一样了。需要注意，如果分块的方法是哈希，每个节点本地的 join 也是哈希 join，那么这两次的哈希算法不能一样。

<br>

如果分块后在每个节点本地使用的是哈希 join，那么这样的 parallel join 又称为 partitioned parallel hash join，如果每个节点本地使用的是 merge join，那么这样的 parallel join 又称为 partitioned parallel merge join. 当然本地还可以使用 nested loop join 等方法，不再赘述。

当然了，如果数据分片的属性就是本次 join 属性，那么分块这一过程就可以免了，因为分块已经完成了，只需要在每个节点上并行执行 join 操作即可。

## Fragment-and-Replicate Join
上面提到的方法中，分块时不论使用的是范围分块还是哈希分块都只适用于等值连接，这样具有相同 join 属性值的元组才会分到同一个块。那么当 join 条件不再是等值连接时，如 $r \bowtie_{r.a < s.b} s$，算法将不再适用。

幸好还有一种叫做 fragment-and-replicate join，下面简称 FARJ，的方法可以使用。FARJ 有两种，一种被称为非对称的 FARJ（asymmetric farj），先来介绍它。

非对称 FARJ 算法有如下步骤：
> * 系统选择一个表，假设为 $r$，进行分块并发送到相应的参与 join 的节点上，可以使用任何分块方法，包括 round-robin.
> * 然后将另外一张表，假设为 $s$，复制到每一个参与 join 的节点上。
> * 最后在每一个节点上执行 $r_i$  join  $s$，在节点上可以使用任何 join 方法。

> 需要注意，这里的分块并不一定按照 join 属性来，因为没有影响。由于本文假设的是数据已经分片了，那么其实只需要执行后面两个步骤就可以了。这个方法又被称作 broadcast join，**既可以用于等值连接，又可以用于非等值连接**，非常有用。如果 join 的两张表一张很小，一张很大，那么选择将小的那一张表复制到所有参与 join 的节点，这个代价一般要低于重新分块大表的代价。

<br>

谈完非对称 FARJ，下面看看对称版的 FARJ：
> 系统将表 $r$ 分成 $n$ 块，$r_1,r_2,...,r_n$，将表 $s$ 分成 $m$ 块，$s_1,s_2,...,s_m$，$n$ 和 $m$ 的大小没有比如的关系，当然了，这样一来就有 $n*m$ 个节点参与 parallel join. 非对称版本其实是一个特例，即 $m=1$. 

> 假设参与 join 的节点为：$N_{1,1}, N_{1,2},...,N_{1,m}, N_{2,1},...,N_{n,m}$. 那么节点 $N_{i,j}$ 上执行的是分块 $r_i$ 和分块 $s_j$ 的 join 操作。为了实现这一点，$r_i$ 需要发送到 $N_{i,1}, N_{i,2},...,N_{i,m}$ 节点上，$s_j$ 需要发送到 $N_{1, j}, N_{2, j},...,N_{n, j}$ 节点上。分块完成后，在每个节点上执行 join 操作，最后汇总结果即可。

> 对称版的 FARJ 同样适用于任何连接条件。两者的示意图如下：
> ![](/img/parallel_join/1.jpg)
> 对于两者的评价，摘抄书中内容如下：
> Fragment-and-replicate join has a higher cost than partitioning, since it involves replication of both relations, and is therefore used only if the join does not involve equi-join conditions. Asymmetric fragment-and-replicate, on the other hand, is useful even for equi-join conditions, if one of the relations is small, as discussed earlier.

<br>

还有一点需要注意，那就是使用非对称 FARJ 方法计算左外连接（定义见{% post_link "Join 的种类" %}），$r\\ ⟕_{\sigma}\\ s$，时，对于复制和分块表的选择有限制：左外连接要求保留左表中那些未被匹配上的元组，并用空值填充一些字段。如果将表 $r$ 复制，表 $s$ 分块，那么在节点 $N_i$ 上可能存在 $r$ 的一些元组不能匹配 $s_i$，但可以匹配在节点 $N_j$ 上的 $s_j$，这样一来单个节点就不能判断这些不能匹配的元组是否丢弃还是保留并填充空值。但如果将表 $r$ 分块，表 $s$ 复制，就不会存在这样的问题。









