---
title: LevelDB 中的 LSM
date: 2022-08-04 22:06:45
tags:
sticky: 1
---

## LevelDB - LSM

![](/img/levelDB/1.png)
上图右边大致是 LevelDB 实现的 LSM，以下简称 LDB.

LDB 是一个采用 LSM 思想实现的存储引擎，分为内存和磁盘两部分，内存中的结构包括 memTable 和 immutable，它们使用跳表实现。<u>当要插入一条数据时</u>，先写 log（WAL），然后插入 memTable，当 memTable 达到预设的上限时，该 memTable 就变成 immutable，不再发生修改（只读），同时创建一个新的 memTable 继续服务。后台线程将该 <u>immutable 序列化</u>然后 dump 到磁盘生成一个 $L_0$ 层的 SSTable 文件，SSTable（sorted string table）文件由索引、布隆过滤器和数据构成，最后会介绍，这个过程称为 Minor Compaction。

上述大概是 LDB 的插入流程，$L_0$ 层的每一个 SSTable 都是 immutable 序列化后生成的，虽然每个 SSTable 内部 key 都是有序的，但它们中 key 范围存在交叉，当查询数据时既有较大的读放大也不支持范围查找。

像 {% post_link "索引篇三：LSM Tree" SMI-LSM %} 中那样 LDB 将 $L_0$ 层 SSTable 合并到下层来提升查找效率，但 LDB 同时实现了范围查找。

从 $L_1$ 到 $L_6$ 每一层的 SSTable 和 $L_0$ 的不太一样，$L_0$ 是 immutable 序列化后直接 dump 而成，而 $L_{i},\\ i >= 1$ 层的所有 SSTable 都是一个大文件按 key 范围拆分出来的，每一个大小都差不多，总大小是上一层总大小的 10 倍左右（每一层的 SSTable 文件大小是不同的）。

<u>当 $L_0$ 层的 SSTable 数量达到了预设的上限后</u>（8 个），就会另起一个线程将它们合并到 $L_1$ 层：合并过程是多路归并算法（见 {% post_link "查询处理篇：Sorting" %}），所有的 $L_0$ 层 SSTable 都参与归并，而 $L_1$ 只有部分会参与，因为它们是按 key 切分的，不存在交叉，所以只需要 key 值与 $L_0$ 层有交叉的参与合并即可，合并后再按 key 值范围切分为多个 $L_1$ 层的 SSTable，不影响未参与合并的。这个过程叫做 Major Compaction.

<u>这样一来查找过程就应该是</u>，先到 memTable 中查找，若不中，到 immutable 中查找，若再不中，就到 $L_0$ 层查找，因为 $L_0$ 层的 SSTable 的 key 值可能存在交叉，所以每一个都需要查找，若还未中，就继续往下层查找，从 $L_1$ 开始的每一层中，各个 SSTable key 范围理论上没有交叉所以只需要查找一个即可，为了实现这个理论，LDB 创建了一个名为 MANIFEST 的文件记录了各层各个 SSTable 文件 key 范围等元数据，每次合并完会更新它。


<u>计算一下 LDB 的写放大和读放大</u>，对于写放大，它其实在 {% post_link "索引篇三：LSM Tree" B-LSM 和 SMI-LSM  %} 之间，按照最坏来算，就和 B-LSM 一样，每层最多为 10，平均为 5. 对于读放大，查找一条记录最坏情况下需要遍历 $L_0$ 的所有 SSTable，$L_1-L_6$ 层各一个 SSTable，一共有 14 个 SSTable，遍历时需要将一个 SSTable（或部分）读入内存，包括 index block、bloom-filter block 和 data block，假设读入时的 block 为 4KB，查找一个 1KB key-value 记录最坏需要读入 $\frac{14\\ *\\ 4KB\\ +\\ SSTable\\ -\\ 4KB}{1KB} \approx 56$. 假如一个 B+ 树高为 5，每个节点的大小为 4KB，查找一条 1KB key-value 记录需要访问 6 个节点，读放大为 $\frac{4\\ *\\ 6KB}{1KB} = 24$.  


## LevelDB - SSTable
[参考文章](https://blog.csdn.net/ws1296931325/article/details/86635751/)

