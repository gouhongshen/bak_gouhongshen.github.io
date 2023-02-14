---
title: 查询处理篇：Sorting
date: 2022-07-01 16:20:55
tags:
sticky: 1
math: true
---

在数据库系统中会经常读取**大量记录**，如果这些记录有序的话，在一些场景中非常有用，比如：
* query sql 语句指定输出有序。
* 一些 Join 算法的高效实现，见 {% post_link "查询处理篇：Join 操作" %}。
* 向 B+ 树中插入大量的索引条目或高效地构造 B+ 树，见 {% post_link "索引篇二：B-plus Tree" %}.

<u>再区分下一些相关的概念</u>。如果文件上面已经建立了索引，那么使用该索引也能实现顺序读取记录，但这样的顺序可能只是记录的逻辑顺序，而非在磁盘上存储的物理顺序（如 secondary index），所以可能每条记录都会涉及 block 的传输。

另外本文讨论的是排序算法的<u>输入数据很大，内存不能一次装下</u>，会涉及多次的 block 传输，这种问题被称为 external sorting，比较常用的算法是 external sort-merge（外部归并排序），这也是本文的主角。


## 外部归并排序
假设 buffer pool 中空闲 buffer（用来缓冲 block）的个数为 $M$，需要排序的表为 $A$，该算法可以分为两个阶段：

<font color=red>初始化阶段</font>
![](/img/query_processing_sort/0.png)
> 该阶段的主要工作是创建多个 *run* 文件，每个 *run* 文件存储的都是排好序的记录。

> 先读取 $M$ 个表 $A$ 的 block（有可能不足 $M$ 个）到内存中，然后采用如快排等方法将这 $M$ 个 block 的记录排好序，最后将这 $M$ 个 block 的记录全部写入一个 *run* 文件中，记为 $R_i$. 重复该过程直到扫描完整个表 $A$，创建了若干个 *run* 文件。为什么这里不需要留一个 buffer 作为写磁盘缓冲呢？因为快排不需要额外的空间，这 $M$ 个 buffer 本身就可以作为缓冲。


<br>

<font color=red>归并阶段</font>
![](/img/query_processing_sort/1.png)
> 这一阶段的主要工作就是合并上一阶段创建的 *run* 文件。

> <u>先来考虑最简单的情况（如上图）</u>，假设上一阶段产生了 $N$ 个 *run* 文件，且 $N < M$，这样就能给每个 *run* 文件分配至少一个 block，还能剩下至少一个 buffer 缓冲输出。在内存中采用归并排序合并这 $N$ 个 *run* 文件，并将已合并部分的记录写入输出缓冲 buffer，假设输出文件为 $F$，以减少磁盘 I/O. 由上一阶段可知，这里的每个 *run* 文件最多有 $M$ 个 block，所以，当某个 *run* 文件在内存中的 block 为空时，还需要去磁盘中读取剩余的 block，直到该 *run* 文件被读取完。直到所有 *run* 文件都被扫描完，文件 $F$ 即为排好序的表 $A$.

> <u>然而，世界并不总是如此美好</u>。如果表很大，上一阶段产生的 *run* 文件数量很大，即 $N > M$，那么就需要重复多次上面的过程。对于当前的 $N$ 个 *run* 文件，每次将其中的 $M-1$ 个（可能不足）合并成一个新的 *run* 文件，待所有 $N$ 个 *run* 文件合并完成后（最多需要 $\lceil \frac{N}{M-1} \rceil$ 次），就产生了一组新的 *run* 文件，记为 $N_1$，继续重复该过程，直到只产生一个 *run* 文件。将 $N_i -> N_j$ 这个过程称为一次 *pass*. <u>下图描述了这个过程</u>，假设一个 block 中只有一条记录，内存中有 3 个空闲 buffer，总共需要经历两次 *pass*.

![pass-example](/img/query_processing_sort/2.png)



## 复杂度分析
本节主要讨论外部归并算法的磁盘访问开销，下面汇总下用到的变量：
* $M$：buffer pool 中空闲的 buffer 数量。
* $b_r$：表 $A$ 文件包含的总 block 数量。
* $b_b$：在上面归并阶段中，会给每一个 *run* 文件分配一个 buffer（block），合并完后再读取该 *run* 的下一个 block，这样的话会涉及多次随机 I/O（因为有多个 *run* 文件，磁头会经常来回移动），所以这里并不做这个限制，而是将分配给每个 *run* 文件的 buffer 数量设置为 $b_b$ 参数，这样一次就有 $\lfloor M/b_b\rfloor-1$ 个 *run* 文件同时参与合并。

由上可知，初始的 *run* 文件有 $\lceil b_r / M \rceil$ 个，每次 *pass* 后，*run* 文件的数量都以 $\lfloor M/b_b\rfloor-1$ 因子减少，所以，<u>总共的 *pass* 次数为</u>：
$$
\lceil log_{\lfloor M\\ /\\ b_b\rfloor\\ -\\ 1}\\ (b_r\\ /\\ M) \rceil
$$
在每一次 *pass* 中，表 $A$ 的每一个 block 都会被从磁盘读取一次和写入一次磁盘，最后一次 *pass* 可以不用写入磁盘（某些场景中）再加上第一阶段的传输次数（读取全部 block 一次，再写一次），<u>那么对该表排序，总共的 block 传输（读+写）次数为</u>（减去了最后一次的写）：
$$
b_r(2\lceil log_{\lfloor M\\ /\\ b_b\rfloor\\ -\\ 1}\\ (b_r\\ /\\ M) \rceil + 1)
$$

还需要计算随机 I/O 的次数，在每一次 *pass* 中，磁头会来回移动从不同的 *run* 文件中读取 block，所以随机读大概需要的 I/O 次数为 $\lceil b_r / b_b \rceil$，假设写入的磁盘和输入的磁盘是同一个，那么随机写的 I/O 次数和随机读一样，所以每一次 *pass* 总的随机读写 I/O 次数为 $2\lceil b_r / b_b \rceil$. 再加上第一阶段的随机读写 I/O 次数，<u>那么对该表排序，总共的随机 I/O 次数为（减去最后一次的写随机 I/O 次数）</u>：
$$
2\lceil b_r\\ /\\ M \rceil + \lceil b_r\\ /\\ b_b \rceil\\ (2\lceil log_{\lfloor M\\ /\\ b_b\rfloor\\ -\\ 1}\\ (b_r\\ /\\ M) \rceil\\ -\\ 1)
$$

如果以上面 pass-example 图中的数据为例，假设 $b_b$ = 1，那么需要的总的 block 传输数量和随机 I/O 次数分别为：$12 * (4 + 1) = 60$，$8 + 12 * (2 * 2 - 1) = 44$.










