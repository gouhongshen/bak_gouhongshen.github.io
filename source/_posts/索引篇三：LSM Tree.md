---
title: 索引篇三：LSM Tree
date: 2022-06-30 09:26:48
tags:
- 索引
categories:
- [索引]
math: true
sticky: 1
---

## 前言
考虑这样一种场景，有大量的数据需要通过 B+ 树索引写入（update/insert/delete）数据库，相邻的待写入数据很可能处在不同的叶节点上，而该索引结构又很大，buffer pool 容纳不下所有的叶子节点，那么完成这样的写入，文件 I/O 次数很可能与数据量成正比。<u>若是使用 magnetic disk（磁盘）</u>，当前最快的磁盘能做到 0.05ms 每 block（4KB）的传输速度，seek time（寻道时间）大概是 4ms 每次，也就是单单考虑寻道时间，每秒也只能做 250 次磁盘 I/O. <u>若是使用 flash memory（如 SSD）</u>，因为 flash 支持随机读写，可以不需要寻道，大概 0.01ms 每页（4KB），但是 flash 不支持原地更新（in-place update），更新单位为 erase block（256KB-1MB），通常需要 3ms，就算是只更新一条记录也必须如此，更何况 flash 还有寿命限制，最多能擦除 1,000,000 次。

所以，<u>在每秒存在大量随机 insert/update/delete 场景中</u>，B+ 树并不是十分理想的结构。为此许多 write-optimized 索引结构被提出，LSM（log-structed merge tree）便是其中一种。LSM 有许多变种，本文先介绍基本的 LSM，称为 B-LSM，然后再介绍一种变体，stepped-merge index LSM，以下简称 SMI-LSM.

## Basic LSM
![图 0](/img/LSM/0.jpg)
如上图所示，B-LSM 包含若干个 B+ 树，它们分为两部分：一个在内存中的 B+ 树和若干个在磁盘上的 B+ 树，它们的叶节点都存储实际的记录。在内存中的 B+ 树称为 $L_0$，在磁盘上的 B+ 树称为 $L_1,\\ L_2,\\ ...,\\ L_k$，图中 $k=3$.

当插入一条数据时，数据首先插入到内存中的 $L_0$，当 $L_0$ 的数据达到预设的阈值后，就需要将其写入磁盘。如果 $L_1$ 为空时，整个 $L_0$ 写入磁盘创建一个新的 $L_1$；当 $L_1$ 不为空时，需要将 $L_1$ 按顺序读出与 $L_0$ 的数据归并，最后在归并的数据上以自底向上的方式创建一个新的 $L_1$，此时旧的 $L_1$ 就可以删除了，$L_0$ 也可以清空以插入新的数据。

如果只有两层，那么每次将 $L_0$ 的数据写到磁盘上时都需要与整个 $L_1$ 合并再写入磁盘创建新的 $L_1$，随着数据越来越多，显然会写入过多的额外数据，为此有两种优化：
* 如图 0 所示，分多层，$L_{i+1}$ 层大小是 $L_i$ 层大小的 $k$ 倍，当 $k$ 个 $L_0$ 合并到 $L_1$ 后，$L_1$ 就满了，此时就可以将 $L_1$ 与 $L_2$ 合并，这样一来，每一条数据在每一层最多被写入 $k$ 次（考虑每一层 B+ 树的第一个叶子节点写入的次数），每条记录在每一层平均被写入 $\frac{k}{2} = \frac{k + (k-1) + .. 1}{k}$ 次，即总的写放大平均为 $k * k/2$.
* 如 Stepped-Merge Index 小节所示。该方式进一步将写放大减少为，每一条数据在每一层仅写入一次，总的写放大为层数。

两种方式各有特点，第一种的写放大要比第二种大很多，第二种方式的读放大又比第一种要高，不过在下面一节中也有对读放大的优化。



## Stepped-Merge Index LSM
![图 1](/img/LSM/1.jpg)
如上图，SMI-LSM 的结构大体和 B-LSM 类似，不同在于磁盘中的每一层包含 $k$ 个相同的 B+ 树，且 $L_{i+1}$ 层的 B+ 树的大小（叶子节点数量）是 $L_i$ 的 $k$ 倍，也就是说磁盘上的每一层的所有 B+ 树合并后刚好构成下一层的一个 B+ 树，但 $L_0$ 的 B+ 树和 $L_1$ 的 B+ 树一样大。 

<br>

<font color=dark-green>LSM 的插入操作</font>
当插入数据时，数据首先被插入内存中的 $L_0$ 树，当 $L_0$ 树容量达到限制时，将其写入磁盘并清空，也就是 $L_1$. 当 $L_0$ 再次达到上限后，再将其写入磁盘，以此往复，磁盘中就可能存在多个 $L_1:\\ L_1^1,\\ L_1^2,\\ ...$ 当磁盘中存在 $k$ 个 $L_1$ 时，就可以将这些树合并成一个 $L_2$ B+ 树，当磁盘中存在 $k$ 个 $L_2$ 时，又可以合并成 $L_3$. 这个过程可以一直执行下去。

> **为什么要合并呢？**考虑查询操作，当要某条数据时（借助 search key），按照上面所述的步骤，查询过程应该是这样的：先查询 $L_0$，不中，再依次查询 $L_1^1,\\ L_1^2,\\ ...$ 若还是不中，继续查询 $L_2^1,\\ L_2^2,\\ ...$ 也就是说查询一条数据可能会搜索多颗 B+ 树。<u>为了限制查找开销（读放大），有两点优化，其一便是合并</u>，将若干个上一层的树合并成下一层的一个较大的树，这样原本会查询多颗小树的操作就只需要查询一颗更矮更胖的大树。<u>另外一个优化是使用布隆过滤器（bloom filter）</u>。尽管合并在一定程度上降低了查询开销，但仍旧不能避免在某一层需要检索多颗 B+ 树。布隆过滤器能够给出某个集合（这里就是一颗 B+ 树）中包含某一个 search key 的概率：若判断包含，则有一定概率不存在（假阳性），若判断不存在，那一定不存在。所以，可以为每颗 B+ 树分配一个布隆过滤器，查询时可以跳过一些树。布隆过滤器建立在 search key value 上，如果一棵树包含 $n$ 个 search key value，布隆过滤器大小为 $10n\\ bits$，使用 7 个哈希函数，那么假阳性约为 $\frac{1}{100}$，也就是能跳过大量无效查询，开销略微高于常规的 B+ 树索引。<u>但是，对于范围查询</u>，布隆过滤器无能为力，还是需要检索每棵树判断是否有在范围内的数据。<u>除了减少读放大</u>，合并也能够减少写放大。

在合并 $L_i$ 层时，先顺序读取 $L_i$ 层每棵树的叶子节点，然后将这些数据按照 search key value 的顺序合并，最后在上面以 bottom-up construction 的方式构建一颗 $L_{i+1}$ 的树。很明显，合并过程没有多余的随机 I/O 读写，$L_i$ 的每棵树是按顺序从左至右读取每个叶子节点，合并时采用的排序算法（见 {% post_link "查询处理篇：Sorting" %}）也没有多余的随机读写，构建下一层树时，也是顺序读写。

<u>可以使用写放大（write amplification）指标来衡量 SMI-LSM 和 B+ 树的磁盘写开销</u>。当插入一条数据时，需要写磁盘的总数据量（bytes）除以单条数据的大小，其值就是写放大。写放大衡量的主要是，当更新一条数据时，需要多少额外的写开销，若写放大为 10，那么更新一条数据，平均需要引起 10 条数据的写磁盘。

<u>对于 B+ 树索引</u>，假设每个叶子节点写回磁盘前平均有 $x$ 次更新，那么写放大为：叶子节点的字节数（block 大小）除以 $x$ 条数据的字节数。假设一个 block 能够容纳 100 条数据，那么写放大为 $\frac{100}{x}$. <u>对于 SMI-LSM</u>，若 $k = 5$，$L_0$ 的数据条数为 $M$，所有层总共的数据条数为 $I$，且 $100 = \frac{I}{M}$. 那么该 LSM 大概有 $log_5(\frac{I}{M}) = 3$ 层，写放大也就为 3，因为同一条数据只会在一层写入一次：写入 $L_1$ 层的一次来自 $L_0$，写入 $L_2$ 层的一次来自 $L_1$ 层的合并......


<font color=dark-green>LSM 的删除操作</font>
一般来说，删除一条数据首先是找到它，然后再删除它，SMI-LSM 并不采用这种方式，而是通过<u>插入删除标记（deletion entry）</u>的方式来标记某条数据已经被删除了。因此 SMI-LSM 的删除操作过程和插入过程一致，只不过插入的是一条 deletion entry. 

插入 deletion entry 后，LSM 的查找和合并操作有些不一样，先看原文：
> However, lookups have to carry out an extra step. As mentioned earlier, lookups retrieve entries from all the trees and merge them in sorted order of key value. If there is a deletion entry for some entry, both of them would have the same key value. Thus, a lookup would find both the deletion entry and the original entry for that key, which is to be deleted. If a deletion entry is found, the to-be-deleted entry is filtered out and not returned as part of the lookup result.

> When trees are merged, if one of the trees contains an entry, and the other had a matching deletion entry, the entries get matched up during the merge (both would have the same key), and are both discarded.

需要解释下，可以确定，$L_0$ 中不会同时存在 deletion entry 和原数据，因为内存中删除操作很快也简单，当 $L_0$ 满了写入磁盘变成 $L_1$ 的 1 棵树，那么在 $L_1$ 的每棵树中也不会同时存在删除标记和纪录，在不同的树中可能同时存在。同一层中，每棵树的数据新旧程度不一样，后写入的更新一些，当合并时，当遇到 search key value 相同的删除标记和记录（可能有多条）时，若删除标记更加新，那么丢弃记录即可，若记录更新，忽略删除记录即可。这样，在下一层中，同一颗树中，同样不会同时存在删除标记和原记录。

<font color=dark-green>LSM 的更新操作</font>
LSM 的更新操作和删除操作过程类似，并不是找到记录然后更新它，而是插入一条<u>更新标记</u>。查找、合并时的过程和上面叙述的类似。


## 总结

由上面的叙述可知，LSM 在插入数据时（删除、更新一样）不会涉及到将数据读出磁盘然后再写入，它采用只追加（顺序写）的方式，拥有非常好的写性能。这是它最主要的优点。

B-LSM 具有很大的写放大，SMI-LSM 做出了一些改变减少了写放大，但增加了读放大，所以有了布隆过滤器等优化，不过 <u>SMI-LSM 不支持范围查询</u>，因为每层的各个 B+ 树之间会存在 key 范围的重叠，而 B-LSM 是支持的，因为每一层只有一个 B+ 树。

LevelDB 是一个存储引擎，和 B-LSM 结构类似，不过也做了很多优化，在一定程度上减少了写放大，依然支持范围查找，详见文章：{% post_link "LevelDB 中的 LSM"%}. 

badgerDB 也实现了 LSM，进一步减少了写放大，详见文章：{% post_link "badgerDB 中的 LSM" %}.


另外，因为 LSM 的插入、更新、删除操作都是只追加的，<u>所以 LSM 很自然就支持 MVCC</u>：查找时，找到第一个版本就好了，合并时，刚好可以删除过期的版本，不过，内存中的结构也得保存多个版本。


