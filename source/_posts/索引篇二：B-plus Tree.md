---
title: 索引篇二：B-plus Tree
date: 2022-06-29 18:41:27
tags:
- 索引
categories:
- [索引]
math: true
sticky: 1
---

## B+ Tree 的结构
![图 1 B+ tree 节点结构](/img/B_plus_tree/1.jpg)
如图 1 所示，B+ 树的每个节点包含 $n-1$ 个 search-key value，$K_1, K_2, ..., K_{n-1}$，和 $n$ 个指针，$P_1, P_2, ..., P_n$. 这些 search-key value 在节点中按序存储，即若 $i < j$, 有 $K_i < K_j$. 在实际中 search-key value 是可能存在重复的，<font color=red>目前只考虑不重复的情况，后面再介绍重复的情况</font>。

B+ 树的节点分为三类，叶子节点（leaf node）、非叶子节点（non-leaf node），下面一个一个介绍。
![图 2 叶子节点](/img/B_plus_tree/2.jpg)

**首先是叶子节点**。如图 2，是一个 n = 4 的叶子节点，search-key 为 instructor 表的 name 属性. 对于 $i=1, 2, ..., n-1$，指针 $P_i$ 指向存储在文件中的、search-key value 等于 $K_i$ 的记录，$P_n$ 指向右边相邻的同级节点或者为 null（如果右边不存在节点）

每一个叶子节点<u>能够容纳的指针</u>的数量为 $[\lceil \frac{n}{2} \rceil,\\ n]$，对于 n = 4，其区间为 [2, 4]. 

对于两个叶子节点 $L_i, L_j$，若 $i < j$，即 $L_i$ 在 $L_j$ 的左边，那么对于所有在 $L_i$ 中的 search-key value，都小于 $L_j$ 中的每一个 search-key value. 

B+ 树是一个多级索引结构，叶子节点常采用稠密索引，也就是说，出现在文件记录中的每一个 search-key value，也会出现在叶子节点中（一个指针对应一条记录）。
![图 3](/img/B_plus_tree/3.jpg)
**然后是非叶子节点**，非叶子节点是对叶子节点建立的多级稀疏索引。非叶子节点和叶子节点的不同是，非叶子节点的指针指向的是树节点而非记录，而且非叶子节<u>能够容纳的指针</u>数量也是 $[\lceil \frac{n}{2} \rceil, n]$. 

在一个非叶子节点中，$P_i\\ \\ 1 < i < n$，指向一个子树，该子树包含的 search-key value 小于 $K_i$，但大于等于 $K_{i-1}$；指针 $P_1$ 指向包含小于 $K_1$ 的 search-key value 的子树；而 $P_n$ 指向其右边的相邻同级节点或者为 null（如果右边不存在节点）。

图 3 展示了一个在 instructor relation 文件上建立的完整 B+ 树，其中 n = 4.

<u>B+ 树是一颗平衡多叉树，即从根节点到每个叶节点的路径长度总是一样的，正是如此，才提供了良好的增删改查性能，而且每个节点至少要半满</u>。

<font color=red>对于 search key value 存在重复的情况</font>，可以有如下的方式解决：
* 叶子节点存储每一个出现的 search key value，无论它是否重复，那么 对于 $i < j$, 就应该有 $K_i <= K_j$.
* 在叶子节点上，对于每一个 search key value，不再存储单个指针，而是一组指针，指向具有相同 search key value 的记录，由于每组指针的数量不确定，节点中的 search key value 个数也就不一定了。

这两种方式都会增加操作的复杂度，不常使用。而绝大多数的数据库系统通过组合多个属性来构成不重复的 search key. 假如某个索引的 search key 为 $a_i$，并且它可能重复（不是主键或候选键），这时系统选择一个不可能重复的属性，假设为 $A_p$，将它们两个组合在一起构成新的 search key $(a_i, A_p)$，称为 composite search-key，也就是联合索引. <u>还有一个问题</u>，联合索引中各属性的顺序如何确定？
> 在创建多列索引时，我们根据业务需求，where 子句中使用最频繁的一列放在最左边，因为索引查询一般会遵循最左前缀匹配的原则，即最左优先，在检索数据时从联合索引的最左边开始匹配。

> 最左前缀匹配原则：索引的底层是一颗 B+ 树，那么联合索引的底层也就是一颗 B+ 树，叶子节点存储的 Key 是组合键值。由于构建一棵 B+ 树只能根据一个值来确定索引关系，所以数据库依赖联合索引最左的字段来构建。举个例子，以 (a, b) 组合属性建立联合索引时，会优先使用属性 a 的大小构建索引，当遇到 a 相等时，再按 b 的顺序，那么叶子节点的 Key 就可能是：[1, 3], [1, 4], [2, 8], [3, 1]...

> 对于 (a,b,c) 组合属性构成的联合索引，如果 where 子句中只包含 b 或 c 而不包含 a，那么该查询就不能使用该索引。



## 在 B+ 树上的查询
假设现在要查询一个 search-key 值为 $v$ 的记录，下图 4 展示了其伪代码，比较简单就不解释了：
![图 4](/img/B_plus_tree/4.jpg)

由于每个节点有一个 $P_n$ 指针指向下一个同级节点，所以 B+ 树支持范围查询（range queries），其伪代码如下图 5：
![图 5](/img/B_plus_tree/5.jpg)

在范围查询的伪代码中，先采用单值查询 find(lb) 的方式找到落在 [lb, ub] 中且最小的记录，然后向后遍历（可能跨节点）找到所有合适的记录。<u>需要注意</u>，伪代码中 n 表示的是 $K$ 的个数，和上文中的 n 意义不一样，上文中 n 表示指针的个数，在伪代码中，节点的最后一个指针应该是 $P_{n+1}$.

**现在计算下 B+ 树查询的时间复杂度**。假设文件中共有 $N$ 条记录，从根节点到叶子节点的路径长度不超过 $\lceil log_{\lceil\frac{n}{2}\rceil}(N)\rceil$. 

一般情况下，B+ 树节点的大小和 block 的大小一致，同为 4KB-8KB，假设 search key 的大小为 32B（一般没这么大），指针的大小为 8B，那么一个节点能存储 100 个 search key value. 如果文件中的记录总共包含 1,000,000 个 search key value，查找某个特定的记录最多只需要访问 4 个节点，即最多读 4 次磁盘，远远少于平衡二叉树的次数，最后只需要根据指针的值再读一次磁盘就能得到目标记录。

<u>而范围查询代价更高一些</u>，假设最后返回的指针个数为 M，这 M 个指针在叶子节点上必然是连续的，所以要读取这 M 个指针，最多需要读取 $\lceil\frac{M}{n/2}\rceil + 1$ 次磁盘，因为一个叶子节点最少有 $\frac{n}{2}$ 个指针。为了获取这些指针所指向的记录，还需要计算一些开销，<u>如果是 secondary index</u>，尽管记录的 search key value 是相邻的，但这些记录在文件中却不一定相邻，所以最多需要 M 次磁盘 I/O. <u>如果是 primary index</u>，那么这些记录在文件中也是相邻的，所以一次磁盘 I/O 读取一个 block，其中包含多条记录，磁盘 I/O 大大减少。

<font color=red>假如 search key value 存在重复，如何查找指定的记录？</font>使用上面所说的使用组合 search key 的方式去重，假设组合后的 search key 为 $(a_i, A_p)$，当给定 $a_i = v$ 时，如何在组合索引中找到所有包含 search key 为 $v$ 的记录？使用范围查找，其范围为 $[lb, ub]$，且 $lb =(v, min(A_p)),\\ ub =(v, max(A_p))$.


## B+ 树文件组织方式
本节讨论，如何以 B+ 树结构组织文件，需要与 B+ 树索引区分开。普通的 B+ 树索引是在文件上建立索引（叶子节点存储的是指针），而 B+ 树文件的叶子节点存储实际的记录。
![图 6](/img/B_plus_tree/6.jpg)

如上图 6，在 B+ 树文件组织方式中，叶子节点（叶子 block）存储实际的记录，而不是指向记录的指针，而非叶子节点还和 B+ 树的定义一样。因为记录要比指针大很多，这样一来叶子节点存储的记录就会大大减少，但依旧要求至少半满。

查找、插入和删除操作与在 B+ 树上定义的一样。

**当要插入一条 search key value 为 $v$ 的记录时**，搜索该 B+ 树，找到 $<= v$ 的最大的 search key value（或者最小的 $>= v$ 的 search key value），同样也就找到了待插入的 block（叶子节点）。<u>如果该 block 还有空间</u>，那么按 search key value 的顺序插入该记录（可能涉及到记录在 block 中的移动，参考数组的插入操作）。<u>如果没有空间</u>，执行分裂操作（split），将待插入纪录和原记录按照 search key value 顺序平均分成两部分，同时创建一个新的 block（叶子节点），将前一半放在原 block 中，后一半放入新 block中。还需要索引新 block，即放入一条记录 $(key, P_i)$ 到分裂 block 的父节点中，其中 $key$ 是新 block 中最小的 search key value，$P_i$ 则是指向新 block 的指针。很明显，这是一个递归操作，父节点的插入依然可能引起分裂。

**当要删除一条记录时**，同样需要定位记录的 search key value，然后删除该记录（如果存在多个相等的 search key value，需要遍历它们直到找到目标记录），删除后，需要移动后面的记录填补删除记录的空白。由于每个节点都需要满足半满，如果删除一条记录后，节点不满足该要求，就需要从兄弟节点借一些记录，如果两个兄弟都不够，就只能释放 block，合并兄弟，同样地，这些操作也会影响到父节点。

B+ 树文件组织方式还能用于大文件，具体地，将大文件划分成多记录，再采用 B+ 树组织。


## 一些其他的内容
<font color=dark-green>Secondary Indices And Record Relocation</font>
假设在 instructor 表文件上建立了两个索引，$A_{idx}$ 建立在 dept_name 属性上的，$B_{idx}$ 建立 ID 上，很明显 $A_{idx}$ 是一个 secondary index，而 $B_{idx}$ 是 primary index。不妨假设 $B_{idx}$ 采用 B+ 树文件组织方式，$A_{idx}$ 采用何种索引结构都无所谓，但一个文件只能有一种索引结构能够改变记录的存储位置，所以 $A_{idx}$ 的索引项只能存储指向记录的指针。当 $B_{idx}$ 节点分裂时，会有部分的记录移动到新的磁盘位置，这时候，那些 secondary index 就需要更新（记录的位置变了，之前的指针也就失效了），包括 $A_{idx}$. 这样的索引结构可能很多，可能涉及很多的记录，那么就可能需要非常多次的磁盘读写。

<font color=red>一个比较广泛的解决方法是</font>，在 secondary index 中，对于某个 search key value，不再存储指向对应记录的指针，而是存储 primary index 的 search key 在该记录中的值。当使用 secondary index 查找记录时，就需要两个步骤：
* 先在 secondary index 中找到该记录对应的 primary index 的 search key value.
* 然后借助该 value 再到 primary index 中查找具体的记录。

这样就避免了更新大量 secondary index 中指向记录指针。举个例子，假如有一张表 $R$ 包含三个属性列：id，salary, name，其中 id 为主键。假如现在要在 name 上建立一个 secondary index，那么在索引叶子节点中存储的不再是 [$V_{name},\\ ptr$]，而是 [$V_{name},\\ id$]. 如果现在执行如下的 sql 语句：
```sql
1. select id, name, salary from R where id = 10
2. select id name from R where name = "Tim"
3. select id name salary from R where name = "Joe"
```
执行第一条 sql 时直接走关于 id 的 primary index；执行第二条 sql 时，可以直接走关于 name 的 primary index；执行第三条 sql 时，需要先在关于 name 的 primary index 中找到对于的 id（可能有多个），然后再走关于 id 的 primary index，<u>这个过程称为回表</u>，减少回表带来的性能损失可以参与文章{% post_link "索引篇四：Parallel Index" %}。


<br>

<font color=dark-green>Search Key Value 长度可变</font>
如果 B+ 树的 search key 为字符串，长度可变，那么每个节点的容量就不一定，出度不一定。同时，插入、删除等过程也会发生变化，不再根据 search key value 的个数判断是否指向分裂还是合并，而是 block 是否能够容纳新的内容。为了增加每个节点能够容纳的 search key value 的数量（尽量使树又矮又胖），<u>一种前缀压缩（prefix compression）的技术被提出来</u>。

对于每个非叶子节点，在索引子树时（或叶子节点），不需要存储完整的 search key value，而是只存储一部分前缀，只要在节点内部，能够区分各个子树即可，如下图 7。
![图 7](/img/B_plus_tree/7.jpg)


<font color=dark-green>B+ 树的高效构造</font>
考虑这样这种场景，在一个非常大的表上（很多记录）建一个 secondary index 类型的 B+ 树索引，表和索引都很大，内存是放不下的。<u>按照最简单的方式构建</u>，扫描表的每一条记录，然后往 B+ 树中插入条目，可能是 search key value 和指向记录的指针，或者是前面提到的 search key value 和 primary index 中该记录对应的 search key value. 由于是 non-clustering index，所以相邻的记录可能要插入到不同的节点中（block），也就是可能存在大量的磁盘读写（表文件可以顺序扫描，每个 block 只需要一次磁盘 I/O，而 B+ 树的各个节点 block 可能需要多次磁盘 I/O，因为不同的记录需要插入到不同的节点，而 buffer pool 的容量有限，这些节点已经替换出了内存）。

像上面这样的大量插入条目的操作被称为该索引的 bulk loading，<u>通常采用的高效的方法是</u>，创建一个临时的索引条目文件。扫描表文件，为每条记录创建一条索引条目，追加到该临时文件，然后对该临时文件排序（见 {% post_link 查询处理篇：Sorting %}），最后扫描已经排好序的文件（非此临时文件，具体见连接），将这些索引条目插入到 B+ 树中。

<u>先排序再插入有什么好处？</u>上面提到的大量的磁盘 I/O 是因为，这些在表文件中相邻的记录的索引条目不一定在索引结构中相邻（不在同一个节点 block 中），那如果先将这些索引条目按照 search key value 排好序，那么插入时，属于同一个节点的条目就能一次性插入该节点，也就是说每个节点 block 只需要一次磁盘 I/O 即可。

<u>性能上还能更近一步</u>，如果索引条目要插入的 B+ 树本身是空的，那么在排完序后，不需要采用插入的方式，而是直接在排完序后的文件上构建。一个文件包含多个 block，这些 block 就是叶子节点，然后取出这些 block 中的最小的 search key value，加上一个指向 block 的指针，就可以构成上一层的条目（分配新的 block，将这些条目写入），继续下去直到创建根节点。<u>这种方式称为 bottom-up B+ tree construction</u>，能够减少大量的磁盘读写（全部的叶节点也就创建好了，也写好了）。 


## 总结
这里总结下 B+ 树的各种操作的时间复杂度，假设，文件中共有 $N$ 条记录，B+ 树的每个节点能存储 $n$ 个 search key value（也就是 $n+1$ 个指针），并且 search key value 不重复：

<u>对于单个记录的查询操作</u>，找到目标记录的（如果存在）search key value 最多需要 $\lceil log_{\lceil \frac{n}{2}\rceil}(N)\rceil$ 次磁盘 I/O，如果是采用 B+ 树组织文件，那么不需要额外的磁盘读写，如果是指针或 primary index 的 search key value 则还需要其他磁盘读写，这里考虑最简单的情况，采用 B+ 树组织文件，下同。

<u>对于单个记录的插入操作</u>，找到待插入的节点（block），最多需要 $\lceil log_{\lceil \frac{n}{2}\rceil}(N)\rceil$ 次磁盘 I/O，插入后，这些 block 还需要写回磁盘，所以一共需要 $\lceil log_{\lceil \frac{n}{2}\rceil}(N)\rceil + 1 + x$ 次磁盘 I/O，其中 1 表示写回更新的 block，$x$ 表示可能分裂引起的 block 更新的数量（包括新分配的 block）。

<u>对于单个记录的删除操作</u>，同插入操作。