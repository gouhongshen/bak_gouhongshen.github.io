---
title: 查询处理篇：Join 操作
date: 2022-07-01 16:21:19
tags:
sticky: 1
math: true
---


关于 join 操作，可以看这篇{% post_link "Join 的种类" %}。

为了方便计算，考虑有 student 和 takes 两张表，需要执行它们的 join 操作，即 $student \bowtie takes$. 同时这两张表具有如下的特点：
> student 表的记录数量 $n_{student} = 5000$
> student 表文件的 block 数量 $b_{student} = 100$
> takes 表的记录数量 $n_{takes} = 10,000$
> takes 表文件的 block 数量 $b_{takes} = 400$

### Nested-Loop Join 算法
该算法的伪代码为：
![](/img/join_op/1.jpg)

该算法非常简单易懂，<u>在最坏的情况下</u>，每个表只分配一个 buffer block，那么总共需要传输 $n_r * b_s + b_r$ 个 block，对于外层表 r 的每一条记录都需要完整地扫描一遍内层表 s，如果每个表文件的所有 block 都按顺序存储，那么总共需要 $n_r + b_r$ 次随机 I/O. <u>在最好的情况下</u>，两张表都能放在内存，那么总共需要传输 $b_s + b_r$ 个 block，2 次随机 I/O.

很明显，如果某个表能够完全放入内存中，应该将它放在算法的内层，其代价和两个表都能放进内存时一样。如果都不能，将较小的表放在外层较好。

现在应用上述算法到上面的例子，<u>假设 student 在外层，takes 在内层</u>：
* 最坏情况下，需要传输 $5000*4000+100=2000100$ 个 block，$5000+100=5100$ 次随机 I/O. 
* 最好情况下，需要传输 $100+400=500$ 个 block，2 次随机 I/O. 

<u>假设 takes 在外层，student 在内层</u>：
* 最坏情况下，需要传输 $10000*100+400=1000400$ 个 block，$10000+400=10400$ 次随机 I/O.
* 最好情况下，需要传输 $100+400=500$ 个 block，2 次随机 I/O. 

### Block Nested-Loop Join 算法
该算法的伪代码如下：
![](/img/join_op/2.jpg)
基础的 nested-loop join 算法中，对于外层表的每一条记录就会完整遍历一次内层表，而在 block nested-loop join 算法中，对这一点做了改进：对于外层的每一个 block，访问一次内层表，如上图。

<u>在最坏的情况下</u>，每个表只能各有一个 block 驻留内存，对于外层的每一个 block，内层表的每个 block 只需要访问一次，再加上外层表的 block 数量，所以总共需要传输 $b_r * b_s + b_r$ 个 block. 在每一个外层的 block 需要随机 I/O 一次内层表（block 连续存储），再加上外层表的每一个 block 需要随机 I/O 一次，所以总共需要 $2 * b_r$ 次随机 I/O. <u>在最好的情况下</u>，两个表都能放在内存，总共需要传输 $b_r + b_s$ 个 block，2 次随机 I/O.

很明显，如果某个表能够完全放入内存中，应该将它放在算法的内层，其代价和两个表都能放进内存时一样。如果都不能，将较小的表放入外层较好。

现在应用上述算法到上面的例子，<u>假设 student 在外层（较小），takes 在内层</u>：
* 最坏情况下，需要传输 $100 * 400+100=40100$ 个 block，$2*100=200$ 次随机 I/O.
* 最好情况下，需要传输 $100+400=500$ 个 block，2 次随机 I/O. 

和 nested-loop join 相比，最坏情况下的代价大幅减少，最好情况两种一致。nested-loop join 和 block nested-loop join 两个算法通过<u>以下方式还能提升其性能</u>：
* 如果 join 条件是在内层表主键/候选键上的等值连接，那么内层循环就能在第一条匹配记录找到后结束。
* block nested-loop join 算法中 nested-loop 的单位是 block，如果增大这个单位，代价就能进一步减少。假设有 $M$ 个 buffer block，给外层表分配 $M-2$ 个，内层表一个，那么内层表的扫描次数就会从 $b_r$ 降到 $\lceil b_r /(M-2) \rceil$. 
* buffer block 淘汰是有策略的，如果使用的是 LRU，那么可以采用电梯扫描的方法，尽快复用那些还没换出内存的 block.
* 如果内层表在 join 属性上有索引的话，就可以不用文件扫描的方法了。

### Indexed Nested-Loop Join
这一部分附上原文：
> In a nested-loop join, if an index is available on the inner loop’s join attribute, index lookups can replace file scans. For each tuple $t_r$ in the outer relation r,
the index is used to look up tuples in s that will satisfy the join condition with tuple $t_r$. This join method is called an indexed nested-loop join; it can be used with existing indices, as well as with temporary indices created for the sole purpose of evaluating the join.

> Looking up tuples in s that will satisfy the join conditions with a given tuple $t_r$ is
essentially a selection on s. For example, consider $student\bowtie takes$. Suppose that we have a student tuple with ID "00128". Then, the relevant tuples in takes are those that satisfy the selection "ID = 00128".

> The cost of an indexed nested-loop join can be computed as follows: For each tuple in the outer relation r, a lookup is performed on the index for s, and the relevant tuples are retrieved. In the worst case, there is space in the buffer for only one block of r and one block of the index. Then, $b_r$ I/O operations are needed to read relation r, where $b_r$ denotes the number of blocks containing records of r; each I/O requires a seek and a block transfer, since the disk head may have moved in between each I/O. For each tuple in r, we perform an index lookup on s. Then, the cost of the join can be computed as $b_r*(t_T + t_S ) + n_r *c$, where $n_r$ is the number of records in relation r, and c is the cost of a single selection on s using the join condition. We have seen in {% post_link "查询处理篇：Selection 操作" %} how to estimate the cost of a single selection algorithm (possibly using indices); that estimate gives us the value of c.


### Merge Join 算法
<u>算法可用于等值和自然连接</u>，其伪代码如下：
![](/img/join_op/3.jpg)

该算法基于这样一个思想：如果参与 join 操作的两个表都已经各自在 join 操作所涉及的属性上有序了，那么完成等值/自然连接就只需要各扫描两张表一遍。

在算法的执行过程中，对于每一个 join 属性值，算法会第一个表维护一个集合 $S_s$，保存了所有具有该 join 属性值的记录，然后对于第二个表的每一条具有该 join 属性值的记录与 $S_s$ 中的所有记录 join 操作。

假设表已经<u>有序了且每个 $S_s$ 都能放进内存</u>，那么算法共需要传输 $b_r + b_s$ 个 block，如果给每个表分配 $b_b$ 个 buffer block，那么共需要 $\lceil b_r/b_b \rceil + \lceil b_s/b_b \rceil$ 次随机磁盘 I/O. 如果将该算法应用在上面例子上，同时令 $b_b=1$，那么共需要传输 $400+100=500$ 个 block，$400+100=500$ 次随机 I/O.

如果<u>未排序</u>，那么就需要先使用{% post_link "查询处理篇：Sorting" 外部排序算法 %}排序。如果 $S_s$ <u>不能放进内存</u>，那么可以再套一个 block nested-loop join，当然需要加上它们的代价。

### Hash Join 算法
<u>为了减少无效的比较次数</u>，除了先排序再 join 和使用索引外，hash 也是一种选择，它的基本思想是这样的：使用同样的哈希函数分别将两个表的元组映射（在 join 属性值上映射）为多个分区（partition），那么具有相同 join 属性值的元组一定会被映射到同一个分区，那么 join 操作只需要在两个表的具有相同下标的分区之间进行就可以了，开始细节描述前，先定义如下的符号：
> 使用 $h$ 表示哈希函数，将 join 属性值映射为整数 $\\{0,1,...,n_h\\}$.
> $\\{r_0,r_1,...,r_{n_h}\\}$ 是表 r 映射的分区，元组 $t_r$ 将放进 $r_i$，$i=h(t_r[JoinAttrVal])$
> $\\{s_0,s_1,...,s_{n_h}\\}$ 是表 s 映射的分区，元组 $t_s$ 将放进 $s_i$，$i=h(t_s[JoinAttrVal])$

下面是算法伪代码和示意图：
![](/img/join_op/4.jpg)
![](/img/join_op/5.jpg)

比较简单，<u>该算法可以用于等值连接和自然连接</u>。从代码中可以看到，$n_h$ 的选值很重要，如果选的好，那么 build relation（表 s）的每一个分区 $s_i$ 和在其上的 hash index 就都能放进内存，而 probe relation（表 r）的分区可以不做此要求。

假设当前系统有 $M$ 个空闲 buffer block，build relation 有 $b_s$ 个 block，如果哈希函数能够满足均匀和随机特性，join 属性值会被平均映射到各个分区，为了能让 build relation 的每个分区都能放进内存，那么 $n_h >= \lceil b_s/M \rceil$.

很可能算出来的 $n_h$ 过大，build relation 不能一次性完成 partition（因为 buffer block 不够用），那么就需要多次 partition，下面是示意图：
![](/img/join_op/6.jpg)
图中展示了这样一种情况，为了使 build relation 的某个分区能放进内存，$n_h$ 的最小取值为 9，但空闲 buffer block 只有 3 个，一轮不能将完成全部的映射，所以采用 recursive partitioning.

其实可以计算，要满足什么样的情况就可以不用 recursive. 当 $M > n_h + 1$ 时，就可以不用，等价地，$M >(b_s/M) + 1$，继续可以近似为 $M > \sqrt{b_s}$. 假设空闲 buffer pool 总容量为 12MB，可以分成 3072 个 4KB 的 block，那么这样的 buffer pool 最多能够支撑 3072 * 3072 * 4KB = 36GB 的 build relation 映射时不需要 recursive.

<u>那么 hash join 的开销又是多少呢</u>? 假设不需要 recursive partitioning，每个分区都能装进内存。首先，r 和 s 都需要分别读入内存 partition，再然后将分区写回磁盘，这过程需要传输 $2 * (b_r + b_s)$ 个 block. 在 build 和 probe 阶段，所有分区又都需要读入内存一次，这个过程需要传输 $b_r + b_s$ 个 block. 考虑到 partition 后，映射到一个分区的记录不一定能填满一个 block，也就是读入写出分区时，也必须传输没有填充的 block 部分，两个表加起来大约有 $4 * n_h$ 额外的 block（每个分区半满），所以一共需要传输的 block 数量为：
$$3 * (b_r+b_s) + 4 * n_h$$

假设共有 $b_b$ 个 block 分配给 partitioning 阶段，两个表读出和写入共需要 $2(\lceil b_r/b_b\rceil + \lceil b_s/b_b\rceil)$ 次随机 I/O. 在 build 和 probe 阶段，读出（不需要再写入）一个分区需要一次随机 I/O，那么对于两个表的所有分区，共需要 $2 * n_h$ 次随机 I/O，所有总共的随机 I/O 次数为：
$$
2(\lceil b_r/b_b\rceil + \lceil b_s/b_b\rceil) + 2 * n_h
$$

[看看 TiDB 的 hash join 是怎么做的](https://pingcap.com/zh/blog/tidb-source-code-reading-9)
[看看云和恩墨的 hash join 是怎么做的](https://mp.weixin.qq.com/s/9EWY2w-il1c-jPvE9VQRwg)
