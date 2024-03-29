---
title: Raft 算法问答录
date: 2022-06-14 19:31:45
tags: 
- raft
- 分布式一致性协议

categories:
- [分布式]
sticky: 1
math: true

# index_img: /img/raft/raft_cover.png
---


## 前言
前段时间学习了 CMU-15445 的课程，也写完了 project，了解了数据库内核的基本知识。这段时间在做 TinyKV，刚好看了 raft，细节很多，所以来总结下。

关于 raft 网上有很多资料：
[raft 小论文](https://raft.github.io/raft.pdf)
[raft 博士论文](http://files.catwell.info/misc/mirror/2014-ongaro-raft-phd.pdf)
[raft 博客](https://tanxinyu.work/raft/)
[etcd raft 实现](https://www.codedump.info/post/20180922-etcd-raft/)

所以，这里并不是对 raft 算法本身的细节记录（可能存在部分），而是自己阅读、实现时的一些疑问和解答。

<u>看完 raft 小论文后，强烈推荐阅读 raft 博士论文（虽然 258 页）</u>，因为它会结合工业实现阐述遇到的问题和解决方式，里面包含了很多细微的点，是一份能大致了解分布式系统的资料。本文绝大多数内容来自这篇博士论文。

## Q0：raft 算法有哪些优化？
1. PreVote 见 Q3.
2. Lease 见 Q6.
3. Read-Only 见 Q9.
4. raft 活性考虑 见 Q10.
5. 批处理和流水线优化 见 Q11.
6. 高级的 multiRaft 见 Q12.


## Q1：raft 算法解决了什么问题？
raft 是一个分布式共识协议（算法），其主要作用是让集群中的节点对某件事情达成一致，如客户端发起更新请求，为了保证多个节点的数据状态一致，就需要让该更新请求在所有节点上都应用成功，否则更新请求失败。raft 算法可以看作一个黑匣子，当某个节点接收到客户端的请求后，首先将该请求交给 raft 模块，由 raft 模块负责节点间的协商，最后将结果返回给节点，节点再反馈客户端。如下图的复制状态机所示。
> 分布式系统的共识算法会将数据的写入复制到多个副本，从而在网络隔离或节点失败的时候仍然提供可用性。

![](/img/raft/rsm.png)

图中的 consensus module 就相当于 raft 黑匣子，state machine 可以理解为 kv 键值数据库。还需要注意两点：1）日志记录和协商同步进行；2）图中是分层的，表示多个客户端和集群节点。

## Q2：在实际实现中，整个分布式系统的流程如何？
这个问题实际是关于
* 集群节点如何与客户端交互？
* 集群节点如何与 raft 模块交互？
* raft 模块如何与其他 raft 模块交互？

要解答这些问题，需要借助现有的成熟的工业系统，比如 etcd、tikv 等等，因为比较熟悉 TinyKV，所以以它为例。


TinyKV 的设计参照了 etcd，将 raft 模块设计成独立的部分，raft 需要的网络、存储服务由上层（非 raft)提供，比较具有灵活性，具体流程如下：
* 客户端向节点发送请求（commands：put/get/delete...)
* 节点准备 WAL(write-ahead-log)
* 节点将 commands 封装成 entry 发送给 raft 模块，开始执行共识协议。
* 某个时间点，节点获取 raft 算法的输出，主要有以下部分：
    * raft 需要存储的日志记录（unstable entry 和一些 raft 自身状态信息）
    * committed entry（已经在大多数节点间达成一致的 entry）
    * messages，需要发往其他 raft 模块的消息
* 当上层模块获取到 raft 的输出后，按其类型做一些操作，对于 unstable entry 和状态信息，执行持久化操作；对于 committed entry，它们已处于安全状态，可以应用其中的 commands 于数据库了，并且可以就这些 commands（请求）向客户端反馈成功；对于messages，将其发送到对应的 raft 模块。完成以上操作后，通知 raft 模块。

<u>为什么 raft 会存储 unstable entry 和状态信息，比如 peers，是为了从崩溃中恢复</u>

<strong>总结一下</strong>，raft 模块被独立实现，其算法输入来自上层（这里的输入可能是客户端请求，也可能是其他 raft 模块的消息），其算法输出由上层负责处理（存储、发送等），从这里也能知道，raft 根本不关心 entry 中的具体请求，那是上层逻辑的责任，它只需要采取办法能够唯一标识一条 entry 即可（Term、Index）。

## Q3：当网络分区发生时，raft算法有什么表现？
该问题比较大，需要分类讨论：
* 单个 follower 节点被隔离，恢复后，会发生什么？
* 网络分区发生时，leader 处在少数部分，恢复后会发生什么？
* 网络分区发生时，leader 处在多数部分，恢复后会发生什么？

<font color=red>对于第一个问题</font>，先看被隔离节点的表现：因为是 follower，它只能被动应答，在一段时间内没有异常发生。等到 election_timeout 后，它自增 Term，发起选举请求，由于网络问题，其他节点接收不到该请求，然后再次等到 election_timeout，再次自增 Term，发起选举请求......它重复该操作，直到从隔离中恢复。

该节点（记为 A）从隔离中恢复后，可能会先收到 leader 发来的 AppenEntriesRPC 或者 HeartBeatRPC，但 leader.Term < A.Term，根据算法，这些 RPC 对 A 没有影响，但 leader 会受到影响（response 中的 Term 比 leader 的 Term 大），leader 会转变为 follower。集群中先超时的节点会率先发起选举请求，由于存在选举限制：<strong>要获取到大多数的选票，就必须具有最新的日志记录</strong>：
> Raft determines which of two logs is more up-to-date by comparing the index and term of <u>the last entries in the logs</u>. If the logs have last entries with different terms, then the log with the later term is more up-to-date. If the logs end with the same term, then whichever log is longer is more up-to-date.

这样的选举可能会持续多次，但无论如何节点 A 都不可能当选 leader，因为它被隔离，没有后面新追加的日志。也就是说，节点 A 的重新加入造成了系统不必要的抖动，其原因在于，节点 A 在隔离期间盲目地自增 Term。

etcd 是如何解决该问题的呢？采用 PreVote 机制，即当一个节点超时后，它并不急于自增 Term，而是先发起选举请求，如果能获取到大多数的选票，再自增 Term 重新发起选举。这样，当 follower 从隔离中恢复后，就不会因为 Term 过大干扰集群的正常流程。下面是 Raft 博士论文中的叙述：
> The Pre-Vote algorithm solves the issue of a partitioned server disrupting the cluster when it rejoins.

<u>然而，还有这样一种情况</u>，当 A 从隔离中恢复后，由于其未收到 heartbeat 超时，发起选举，虽然其 Term 没有增加（PreVote 的限制），但是它在大多数节点中仍然具有较新的日志（可能是隔离期间没有新的请求，也可能是隔离时间比较短，新的日志还没有复制到大多数的节点上），依然可以当选 leader。集群当然可以正常工作，但旧 leader 上还未来得及复制的新日志就会被覆盖，客户端也就需要超时重试，这实际上也算一个干扰，该问题的解决方式会在 Q6 中问题三提到（租约期-Lease）。

<br>

<font color=red>对于第二个问题</font>，leader 处在少数节点分区部分，根据 raft 要求，一条 entry 能被提交，该entry 至少需要被 N/2 + 1 个节点安全复制，因此上层交付的任何 proposal 都无法被提交，自然无法被应用到数据库和反馈客户端，客户端会出现请求超时。后面的客户端请求可能会被路由到另外一个节点，直到请求能够被正常执行。

当网络分区恢复后，该 leader 会接受来自新 leader 的 RPC 请求，转换成 follower，开始正常的日志复制。该种情况下，是否会出现第一个问题中的场景呢？是有可能的，比如四个节点，每个分区中存在两个节点，包含 leader 的分区不会触发新的选举，但另外一个分区会发起多次的选举（或预选举），<strong>这种情况下，整个系统瘫痪，无法对外服务</strong>。

<br>

<font color=red>对于第三个问题</font>，这种情况相对较简单，系统可正常对外服务，少数分区可能存在多次选举，但分区恢复后，可以开始正常的日志复制，具体过程在前两个问题中已经提及。

## Q4：leader commit 日志之前崩溃了，会发生什么？
该问题在 raft 论文中有论述，是关于如何处理前任 leader 复制的日志。<strong>我当时的疑问是</strong>：leader 将最新的日志复制到了一部分节点后，或许是还未满足大多数原则，或许是 commit 之前就崩溃了，这些最新的日志会被怎么处理？raft 协议有一个规定：
> 一个 leader 只能提交当前任期的日志，不能提交之前任何任期的日志。当 leader 提交一条当前任期的记录时，之前的所有日志记录都会被提交（被动）。

<font color=red>下面先来看一下，遵守该规则时，raft 协议的表现</font>：

![](/img/raft/Q4_1.png)

图中方块中的数字标识 Term，上方的数字标识 Index。
* (a)中，S1 为 Term2 的 leader，在将 Index=2 的日志复制到 S2 后崩溃。
* (b)中，S1 崩溃后，S5 在 Term3 当选 leader（S5 获得 S3、S4 和 S5 的选票），并追加了一条日志。
* (c)中，S5 崩溃，S1 在 Term4 当选 leader，并继续复制日志，此时它将 Index=2 的日志成功复制到了大多数节点上，但还未提交。
* (d1) 和 (d2) 描述两种情况：
    第一种是(d1)，以前任期复制的日志（未提交）被后面新的日志覆盖。客户端等待响应超时，会重新发起请求（我之前还在担心，这会不会造成数据丢失，太天真了）。对应的是 S1 再次崩溃，在 (c) 的局面下，S5 再次当选 leader（图中未画出新的任期，S5 可以获得 S2-4 的选票），由于复制日志以 leader 的日志为准，所以 Index=2 的以前任期的日志会被 S5 的 Index=2 的日志覆盖。

    第二种是(d2)，以前任期复制的日志可以被后面新的 leader 提交（属于被动提交，因为 raft 中，提交一条日志，就表示该条日志之前的所有日志都已被提交）。对应的是，S1 在崩溃之前将日志复制到了大多数节点上，此时 S5 已经不可能再当选，新的 leader 只能在 S1-3 之中。假设，S1 未崩溃，那么，S1 通过提交 Index=3 的日志，之前的日志也就一起被动提交了；就算 S1 在提交之前崩溃了，新的 leader 通过提交当前任期的日志也能提交以前任期的所有日志。

<strong>从这里，应当认识到两点</strong>：
* 复制的日志可能会被覆盖，客户端会重试。
* raft 算法中，leader 会强制要求其他节点的日志与自己一致，对安全性的考虑应该结合选举限制一起理解。

<font color=red>那如果不遵守该协议，raft 表现又是怎么样的呢？</font>
* 从 (b) 开始可以以另外一种方式解读，假设 S5 在当 Term = 3 当选后，向日志中追加了一条 Index = 2 的记录后就崩溃了
* (c) 中，S1 在 Term = 4 成功当选（先不看 Index = 3 的粉红色方块，假设它不存在），然后继续复制本地 Index = 2，Term = 2 的记录，最终复制到了大多数节点上（S1，S2，S3），然后提交该记录（注意该条记录不属于 S1 的当前任期）。
* 提交后 S1 就挂了，这时来到了 (d1)，S5 恢复后成功当选（日志一样长，但 Term 更大），它继续复制自己本地 Index = 2 的日志记录，其他节点的日志当然会与该记录发生冲突，然而 raft 复制日志是以 leader 为准，<u>所以 Index = 2 的记录会被覆盖，尽管它们已经被提交了</u>。同一个 Index 的记录可被提交了两次，一次 Term = 2，一次 Term = 3.
* (d2) 自然描绘的是正常的情况，S1 没挂，并且还有新的请求（Term = 4，Index = 3）到达，它们都被复制到了大多数节点。

也就是说，如果一个 leader 主动提交了不属于当前任期的日志记录，那么已经提交的记录依然可能被覆盖，raft 是禁止覆盖已提交的日志的，如果覆盖已提交的日志会造成数据丢失。

<u>现在进一步分析</u>，当新 leader 当选后，以前任期的记录会在有当前任期记录提交时被动提交，那如果迟迟没有新的请求到来，以前任期的记录也就长时间不能被提交，为了能够快速提交以前任期的记录，raft 协议要求每个新 leader 当选后都要向日志中追加一条 no-op entry（如 (c) 中 Index = 3，Term = 4 的粉红色方块），然后复制给其他节点，最后提交它。

<font color=red>leader 选举成功后会先提交一条 no-op 日志是非常重要的内容</font>，除了上述提到的问题，还涉及到很多其他方面，总结如下：
> * 如果没有这一步，已经存在于大多数节点的以前任期的记录不能及时地被提交，那么就可能增加客户端地响应时间，同时，也面临被新 leader 覆盖地的风险。
> * Q9 提到的 read-only 优化需要。
> * 防止单步成员变更过程中出现脑裂，见 Q5.



## Q5：Raft 成员变更过程如何理解？
<font color=dark-green>Raft 成员变更（Cluster Membership Change）</font>
raft 集群中的每一个节点都会记录集群节点的信息，即 peer 配置，如果节点发起选举或是 leader 复制日志都需要该配置，以便将请求发送给其他节点。<u>成员变更本质上就是 peer 配置的变化</u>：向集群中新增节点，就是向 peer 配置中新增节点信息；从集群中移除节点，就是从 peer 配置中删除节点信息。

raft 博士毕业论文中设计了两种算法来处理成员变更：
* 方法一：一次变更只包括一个节点加入集群或从集群中移除；
* 方法二：一次变更包括多个节点加入集群或从集群中移除。

由于对安全性的考虑，第二种方法会引入额外的复杂度，不如第一种简单，考虑如下情况（见下图）：
![](/img/raft/Q5-1.png)
当集群中成员发生变更时，该变更不可能同时应用在所以节点上，上图集群中有 3 个节点，现新增 2 个，在变更的过程中，有些节点更新了自己的 peer 配置，感知到的是 5 个节点，如 server 3，而其他节点可能还未得到更新，记录的仍是 3 个节点，如 server 1 和 server 2。<u>如果此时发生选举，就可能会选出两个 leader</u>：
* server 3：获得 server 3、4、5 的选票。
* server 1：获得 server 1、2 的选票。

由于 server 1 的 peer 配置还未更新，认为集群中只有 3 个节点，那么它已经得到大多数的选票了，可以当选为 leader；而 server 3，根据它的 peer 配置同样也得到了大多数的选票，可以当选为 leader. 同时移除多个节点也会导致该问题。也就是说一次涉及多个节点加入或移除的变更是不安全的，不能保证唯一的 leader。

为了解决这个问题（会产生两个大多数群体），第二种方法会引入一个过渡状态，被称为 joint consensus（共同一致），有时间再讨论这种方法。<font color=red>本节只讨论第一种方法</font>。

<u>为什么一次变更只包含一个节点的情况（方法一）不会引发上述问题呢？</u>考虑如下场景：
![](/img/raft/Q5-2.png)

**如上图 case 1 所示（case 2 类似）**，集群中刚开始有 A、B、C 三个节点，$T_0$ 时刻节点 D 加入了该集群。

$T_1$ 时刻，节点 C 感知到新节点的加入，更新了 peer 配置，此时，集群分为了两部分：更新了 peer 的节点（C、D）和未更新 peer 的节点（A、B），但这并不会将集群分裂成两个大多数群体，C 或 D 要想当选 leader，至少需要 3 票（包含自身选票），A 或 B 要想当选，至少需要获得 2 两票（包含自身选票），但这两种情况不可能同时发生，前者的 3 票与后者的 2 票存在重合。

$T_2$ 时刻，B 节点也感知到了新节点的加入，更新了 peer 配置，此时集群中依然存在两个部分：更新了 peer 的节点（B、C、D）和未更新 peer 的节点（A），这种情况下同样不会产生两个 leader.

$T_3$ 时刻，A 节点也感知到了新节点，至此整个集群都完成了 peer 配置的更新。

**再来看 case 4 所示（case 3 类似）**，集群中刚开始有 A、B、C、D 四个节点，$T_0$ 时刻节点 D 被移除了该集群。

$T_1$ 时刻，节点 C 感知到了节点的移除，更新了 peer 配置，此时集群分为两部分：更新了 peer 配置的节点（C）和未更新 peer 配置的节点（A、B），同样，如果此时需要选举 leader，不会产生两个，若 C 想要当选 leader，至少需要 2 票（包含自身选票），若 A 或 B 想要当选，至少需要 3 票（包含自身选票），前者的 2 票与后者的 3 票存在重合，不可能同时满足。

$T_2$ 时刻，节点 B 感知到了节点的移除，更新了 peer 配置，虽然集群中还是存在两个部分，同样也不会产生两个 leader.

$T_3$ 时刻，节点 A 也感知到了节点的移除，至此整个集群都完成了 peer 配置的更新。

<font color=red>因此，若一次变更只包含单个节点的加入或删除，那么不会造成集群的分裂，这种情况下变更是安全的。</font>

<br>

**如果存在多个节点需要加入或删除怎么办？**

论文指出，对于这种情况，可以一次一个节点地变更。由上面的讨论可知，若一次变更只涉及一个节点，那么当其他节点感知到变更时，就可以更新并应用新的 peer 配置，并且不会产生安全性问题。下面是论文描述的单个节点的变更过程：
> * leader 接收到成员变更请求（一种特殊的 entry：$C_{new}$）后，将该 entry 存在日志中，并应用该配置，同时将 $C_{new}$ 复制给其他 follower 节点。
> * follower 节点收到 $C_{new}$ 后，同样存入日志中，并应用该新配置。
> * 当一个节点（candidate 或 leader）需要给其他所有节点发送请求时，使用的是从日志中检索到的最新的配置。
> * 当 $C_{new}$ 在所有节点中达成一致后被提交，此次变更就完成了，leader 反馈变更成功，可以开始下一次的变更。

上述过程其实和普通的 raft 过程没什么区别，主要的不同在与配置是即时生效的，<font color=red>但单步成员变更存在一个问题</font>，[如下例子](https://zhuanlan.zhihu.com/p/359206808)：
> ![](/img/raft/Q5-3.png)
> $t_0$：节点 abcd 的成员配置为 $C_0$；
> $t_1$：节点 abcd 在 Term 0 选出 a 为 Leader，b 和 c 为 Follower；
> $t_2$：节点 a 同步成员变更日志 $C_u$，只同步到 a 和 u，未成功提交；
> $t_3$：节点 a 宕机；
> $t_4$：节点 d 在 Term 1 被选为 Leader，b 和 c 为 Follower；
> $t_5$：节点 d 同步成员变更日志 $C_v$，同步到 c、d、v，成功提交；
> $t_6$：节点 d 同步普通日志 E，同步到 c、d、v，成功提交；
> $t_7$：节点 d 宕机；
> $t_8$：节点 a 在 Term 2 重新选为 Leader，u 和 b 为 Follower;
> $t_9$：节点 a 同步本地的日志 $C_u$ 给所有人，造成已提交的 $C_v$ 和 E 丢失。

> 为什么会出现这样的问题呢？根本原因是上一任 Leader 的成员变更日志还没有同步到多数派就宕机了，新 Leader 一上任就进行成员变更，使用新的成员配置提交日志，之前上一任 Leader 重新上任之后可能形成另外一个多数派集合，产生脑裂，将已提交的日志覆盖，造成数据丢失。

> Raft 作者在发现这个问题之后，也给出了修复方法。修复方法很简单, 跟 Raft 的日志 Commit 条件类似：新任 Leader 必须在当前 Term 提交一条日志之后，才允许同步成员变更日志。也即 Leader 在当前 Term 还未提交日志之前，不允许同步成员变更日志。

> 按照这个修复方法，最简单的实现就是 Leader 上任后先提交一条 no-op 日志，然后再同步成员变更日志。这条 no-op 日志可以保证跟上一任 Leader 未提交的成员变更日志至少有一个节点交集，这样可以发现上一任 Leader 的日志是旧的，从而阻止上一任 Leader 重新选为 Leader，进而阻止了脑裂的产生。

> 对应上面这个例子，就是 $L_1$ 当选 Leader 后必须先提交一条 no-op 日志，然后才能开始同步 $C_v$ 和 E，以便能发现 $L_2$ 的日志是旧的，从而阻止 $L_2$ 当选 Leader。

> 另一种方法是使用 Joint Consensus 成员变更，没有这样的正确性问题。

我在做 TinyKV 项目时发现，$C_{new}$ 并不是即时生效的，而是上层模块处理 committedEntries 时若发现 $C_{new}$ 被提交了，才会更新节点的配置，也是算是搞定了这个问题。



## Q6：如何提升成员变更中的集群可用性？
要想解答问题首先得了解问题，成员变更过程中集群的可用性为何会受影响？分为三个部分：
1. <u>新加入一个节点到集群中时</u>，该新节点的日志大概率非常落后于 leader（甚至为空），在追赶上（catch up）leader 之前，这个新节点不能参与任何新日志提交的决策，甚至是拖累。比如一个初始有 3 个节点的集群新加入了一个节点，在新节点追赶的过程中，初始 3 个节点中的某个节点崩溃下线了，那么该集群在一段时间内都处于不可用（无法提交任何新的日志），因为达成一致需要大多数节点（$\frac{3}{4}$）成功复制了该日志条目，而新节点还在追赶中。
2. <u>当 leader 节点收到移除自己的请求时</u>。论文中到了该问题（因为 Q5 中方法二需要处理该问题），其实不算一个大问题，解决方法很多，后面会提到。
3. <u>被移除的节点可能会发起选举干扰集群的正常流程</u>。当 leader 收到移除某个节点的请求后，leader 不再给被移除的节点发送任何日志（包括移除请求日志 $C_{new}$），同样也不会发送心跳包。被移除的节点感到很无辜啊，它并不知道自己已经不属于该集群了，所以等待它的必然是超时，它自然会发起选举请求，也就干扰到了集群。

<font color=red>对于第一个问题</font>，解决方法很简单，新节点刚开始加入时，将其当作 non-voter，即不被算作大多数，当其追赶上 leader 的日志后，由 leader 再一次的将其作为正式成员加入到集群中。

还有一个问题需要解决：<u>如何定义“追赶上”？</u>当新节点还在复制日志时，新的日志又会到达 leader，如果是完全相同，基本不可能。细想一下，新节点加入集群影响可用性主要是因为它需要较长的时间复制日志，那如果这个时间比较短，对可用性的影响自然就可以接受，论文中建议这个时间低于 election timeout，所以当新节点与 leader 的日志差距小到复制的时间开销低于 election timeout，那么就可以将其正式加入集群了。<u>具体算法是这样的</u>：
> 新节点从 leader 处复制日志被划分为多个 round. 在每个 round 中复制 leader 在该次 round 开始时具有的所有日志，复制过程中新到达的日志会在下一个 round 中复制。leader 可能会等待几个 round（比如 10），若最后一次 round 所花费的时间少于 election timeout，那么 leader 就可以将该新节点正式加入集群中。
> 如果新节点出问题了，或者太慢了（无论是哪种都会超时），那么 leader 会主动中止本次成员更改，返回失败，后面节点可以继续请求加入，这时，它已经具有一部分日志了，成功的概率大大地增加。

<br>

<font color=red>对于第二个问题</font>，有两种方法可以解决：
1. 如 Q5 中提到的，可以在该 $C_{new}$ 提交后再应用新的配置。但需要注意一点，当 leader 接受到移除自己的请求 $C_{new}$ 后，leader 不应该参与任何日志的提交决策，即统计大多数时不应该算上自己，比如集群中只有两个节点时，当 leader 提交 $C_{new}$ 后移除自己，另外一个节点很可能根本没有复制 $C_{new}$，集群就陷入不可用状态了。
2. 使用 Q7 中提到的 leadership transfer.

<br>

<font color=red>对于第三个问题</font>，粗看起来有点像 Q3 的问题一，采用 PreVote 优化就能解决。实际中，很有可能当一个节点被移除后超时发起选举时，在大多数节点中它依然具有较新的日志，也就是它依然能够当选 leader. 有意思的事情发生了，被移除的节点居然还能自己回归（当然移除节点请求会超时返回失败，稍后可能会重试）。如果移除请求会不停重试，该节点始终会被移除集群，然而这个过程就耗费了太多时间，而且旧 leader 的一些日志还可能会被覆盖，导致客户端请求超时失败。
> 需要解释下，如果被移除的节点发起选举时，新的配置还未提交，那么被移除的节点有可能回归，该次配置也就失败了；如果新的配置已经提交了，那么这样的选举请求会干扰集群，使他们选举新的 leader，影响可用性。其实吧，可不可以在节点回复其他节点请求时先判断一下请求来源方到底在不在本集群中呢？

<br>

PreVote 能够阻止部分的干扰，但显然还不够，<u>因此 raft 引入了租期（Lease）的概念</u>：如果一个 follower 能够在 election timeout 时间内收到 leader 的信息（AppendEntries 或 HeartBeat），则该 follower 处在租期内。当一个 follower 收到一个选举投票请求时，如该 follower 还处在租期内，那么 follower 会拒绝或者直接忽视该投票请求。

下面代码是 etcd 对 Lease 的实现。引入上面的方法后，尽管被移除的节点在大多数节点中具有较新的日志，只要 leader 正常工作，被移除的节点就不可能当选，自然也就不会干扰到集群。然而，<u>Lease 可能会影响 Q7 中的 leadership transfer</u>，所以代码中有一个特殊的变量 force，来判断是不是处于 leadership transfer.

```golang
func (r *raft) Step(m pb.Message) error {
	// Handle the message term, which may result in our stepping down to a follower.
	switch {
	case m.Term == 0:
		// local message
	case m.Term > r.Term:
		if m.Type == pb.MsgVote || m.Type == pb.MsgPreVote {
			force := bytes.Equal(m.Context, []byte(campaignTransfer))
			inLease := r.checkQuorum && r.lead != None &&
                       r.electionElapsed < r.electionTimeout
			if !force && inLease {
				// If a server receives a RequestVote request within the minimum election timeout
				// of hearing from a current leader, it does not update its term or grant its vote
				r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] ignored %s from %x [logterm: %d,
                index: %d] at term %d: lease is not expired (remaining ticks: %d)",
					r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm,
                    m.Index, r.Term, r.electionTimeout-r.electionElapsed)
				return nil
			}
            // ...
		}
        // ...
    }
    // ...
}
```

**其实这里我有个疑问**：实现 Lease 后，岂不是会增加正常选举的时长？假设某时刻 leader 维修下线了，一段时间后，某个 follower 率先超时向其他所有发送选举请求，由于其他节点都还未超时（还在租期内），该 follower 一定不会当选，且只有当集群中超过半数的节点都超时后才可能会有 leader 产生。


## Q7：Leadership Transfer 过程是怎样的？
raft 通过该过程，允许一个节点将其 leader 的身份移交给另外一个节点，<u>移交 leadership 在以下两个场景中很有用</u>：
* 当前 leader 节点需要下线。leader 节点可能需要下线维护或集群移除了该 leader 节点。无论是哪一种，当 leader 节点下线后，集群必须等待一段时间后才能选举出新的 leader，对外提供服务，采用 leadership transfer 就可能避免等待这样一段时间。
* 节点不适合继续承担 leader 角色（或有更好的选择）。一直处于高负载的节点不适合做 leader，远离 datacenter 的节点也不太适合做 leader...可以通过 leadership transfer 将 leadership 身份移交给合适的节点。

结合 raft 协议的基本流程可知，要想将 leader 身份转移给另外一个节点，<u>需要保证两点</u>：1) 这个节点有足够新的日志（至少在大多数节点中较新），2) 该节点还得率先发起选举。raft 通过以下步骤保证这两点：
1. 当前 leader 停止接受新的客户端请求。
2. 当前 leader 通过常规的日志复制，将其日志全部复制到候选节点，使其具有最新的日志记录。
3. 上述步骤完成后，当前 leader 发送 TimeoutNow 请求给候选节点，候选节点收到后发起选举（不用等待 election timeout）。<u>这里与 Q6 中的 Lease 机制有冲突</u>，需要在投票请求中添加一个字段，暗示这样的选举是合法的。

当前 leader 收到候选节点的投票请求时，当前 leader 就会变成 follower（步骤 1 和 2 能保证这一点），此时他就可以下线了，transfer 过程就完成了。这个过程当然可能出问题，下面是论文中关于异常的叙述：
> It is also possible for the target server to fail; in this case, the cluster must resume client operations. If leadership transfer does not complete after about an election timeout, the prior leader aborts the transfer and resumes accepting client requests. If the prior leader was mistaken and the target server is actually operational, then at worst this mistake will result in an extra election, after which client operations will be restored.

**我当然又有问题了**，步骤 1 很明显会导致集群一段时间内不可用，transfer 带来的好处能不能抵消这个缺点？？？但是论文中说他们还没有测试过...

## Q8：Raft 之外的一致性问题？

关于分布式系统一致性内容，可以参考文章{% post_link "分布式一致性模型" %}。

如 Q2 中所说，leader 节点收到来自客户端的请求（command）后会将它们封装为 entry，然后 propose 给 raft 模块开始共识协议，当 entry 达成一致被提交后，上层模块就会将其中的 command 应用到数据库（复制状态机）中，至此，raft 协议就应该结束了，分布式节点（应该是大多数节点）的状态都达到了一致。

当上层模块将 command 应用后，应该就该请求反馈客户端，<u>如果节点在回复客户端前崩溃了</u>，那么客户端很可能会超时重试，意味着相同的请求会再次被提交和应用，这种现象被称为 at-least-once semantics，会导致不正确的结果和不一致的状态。如下图。
![](/img/raft/Q8-1.png)

为了避免上述问题，就必须实现 linearizable semantics（线性语义），也就是每个请求只执行一次，无论重复多少次（同一条指令）都只会返回第一次执行的结果。

<u>如何是实现线性语义呢？论文中采用如下方法</u>：
* 每个客户端启动时会向集群注册自己，集群给每个客户端一个独一无二的 id（标识符），此后客户端会给自己的每一个请求分配一个唯一的 serial_number，并将该序号和自己的标识符封装在请求中。同时，集群中的每个节点为每一个（应用过其请求的）客户端都维护一个 session = [id, serial_number, response]（需持久化）.
* 当集群的中的节点（leader）接收到一个请求时，首先会根据请求中的 id 和 serial_number 查找是否存在相应的 session，如果存在，直接回复，如果不存在，就开始正常的 raft 协议（id 和 serial_number 也会封装在 entry 中，这样应用这个请求的每一个节点就都能维护一个对应的 session 了）。
* 如果一个客户端一次只发送一个请求，那么只需要维护最近的一个 session 即可，当新的 serial_number 来到时，旧的就可以删除了；如果一次发送多个请求，那么需要维护一组 session，客户端的请求中可以包含最小的没有接收到回复的 serial_number，那么节点就可以删除小于这个序号的所有 session 了。

> Given this filtering of duplicate requests, Raft provides linearizability.

<br>

<u>上面的方法看起来不错，但还需要考虑一个问题</u>：
因为内存有限，session 不可能永远的存储下去，除了根据客户端的序号清理过期的外，还需要主动清理一些不太活跃的 session. 其实可以将 session 保存在 stable storage 中，采用 B+ 树索引文件组织。

<u>如果要主动清理，那么又会涉及两个问题</u>：
1. 如何在所有节点中对清理某一 session 达成一致（要么所有节点——至少大多数节点——都清理，要么都不清理），不然，有些节点清理了，会可能应用重复的 command，而另外一些没有清理，则不会应用重复的 command，整个集群就处于不一致状态。
2. 当清理了不应该清理的 session 后应该怎么办？当节点接收到某个请求后，如果不存在该客户端的任何 session 记录，那么节点无法判断请求是否是重复的。

<u>对于第一个问题</u>，一种方法是超时清理，若某个 session 对应的客户端在一段时间内（寿命）不活跃，那么节点清理该客户端的 session. 如何在集群节点间对超时达成协议呢？每个 entry 可以再加入一个时间戳，表示该请求的受理时间，这样 session 的起始时间就一致了。客户端可以在不活跃的时候发送保活请求（同样需要执行共识协议），应用该请求会更新 session 的寿命。

<u>对于第二个问题</u>，前面提到客户端第一次启动时需要向集群注册（RegisterRPC），这个注册实际上也是一个请求，同样需要执行共识协议，为它创建第一个 session. 当节点遇到没有任何 session 记录的客户端请求（非注册请求）时就直接返回失败，abort 客户端，然客户端自己处理异常（比如发送回滚请求等）。

## Q9：只读请求（read-only request）的优化？
前面提到，对于客户端发来的所有请求，都会走 raft 流程，包括 append log、replicate log、commit、apply，最后再回复客户端，会有较大的延迟，因为要日志落盘、网络传输等。<u>只读请求不修改复制状态机，对数据的一致性不会产生影响</u>，按理来说可以不走 raft 流程，而实际上在大多数的系统中读请求数量会远高于写请求，那么有没有方法优化 read-only 请求呢？

当然有了，介绍之前先讨论下如果只读请求不走 raft 流程会产生什么问题。

<u>先考虑只有 leader 能接受请求</u>。如果 leader 一收到只读请求就直接执行并回复，那它可能返回过期的数据，为什么？有以下两个原因：
* leader 身份过期。leader 处于网络分区中，其他分区早已经选出新 leader 并接受了新的请求。那么当前的 leader 的数据可能已经过期了。
* 读请求涉及的数据可能已经 commit 但还未 apply。

能不能读取过期的数据完全取决于系统的类型，允许读取过期数据实际上破坏了{% post_link "分布式一致性模型" %}中的线性一致性，具体可参阅文章。这里讨论如何避免读取过期的数据。

原因找到了，leader 采用下面的步骤解决上述问题：
1. raft 算法可以保证，leader 在选举成功时一定具有所有 committed 的日志，但也可能拥有存在于大多数节点而还未 commit 的日志，leader 为了确定在自己任期的 commit index，它会在选举成功后 append 一条 noop-entry，如 Q4，这样他就能得知最新的 commit index 了。
2. leader 另外采用一个变量 readIndex 来记录当前的 commit index.
3. 为了确保自己的身份没有过期，leader 会广播一轮心跳（即便心跳还未超时），如果收到了大多数节点的回复，说明自身还未过期，也就是 readIndex 是当前最大的 commit index.
4. leader 等待它的复制状态机 apply 日志，等待 apply 到 readIndex 时，就能保证复制状态机中的数据是最新的
5. 这时 leader 执行这些只读请求然后回复。这样就一定能避免读到过期的日志，也就保证了线性一致性。

可以看到这 5 个步骤解决了上述的两个问题。当只读请求到达时，leader 总是会重复执行 3 - 5 步骤，尤其是步骤 3，开销还是很大，所以 leader 可以通过累计一定数量的只读请求，让它们平摊开销。

<u>如果 follower 也能执行只读请求</u>就能进一步提升系统的读吞吐量，也能减轻 leader 的负载。follower 不一定具有最新的数据，它只能通过询问 leader 最新的 readIndex 是多少，但该 leader 自己的数据有可能过期，所以可以采用如下的方法：当 follower 收到只读请求后，向 leader 发送消息请求最新的 readIndex，leader 收到请求后执行上述的 1-3 步骤，返回给 follower 最新的 readIndex，最后再由 follower 执行步骤 4-5.


<u>raft 博士论文还提到了一种优化</u>，但他们不推荐，这种优化可以避免 leader 发送心跳包来确认自己的身份。这个优化基于如下的思考：只要集群中未发起选举，那么当前 leader 一定拥有最新的数据，其他节点要想发起选举必须要等到 election_timeout 超时，如果此时距 leader 上次收到大多数节点回复的时间在一个阈值内，那么就没有必要再次发送心跳来确认身份，因为在这期间不可能有节点超时发起选举，这个阈值是多少呢？
$$
start + \frac{election\\ timeout}{clock\\ drift\\ bound}
$$

<u>这个方法和 Q6 中的 lease 本质是一样的</u>，对于 follower，只要自己的 election_timeout 还未超时，那么它就认为 leader 还处于它自己的租期中；对于 leader，只要没有任何 follower 超时选举，它就依然处于租期中。考虑到各个机器的时钟有差异，所以有一个参数 clock drift bound. 这个方法最难的地方在于时钟的猜想不一定能保证。



## Q10：极端情况下的活性问题
<u>考虑这样一种情况</u>：一个 raft 集群有 4 个节点，其中一个节点与其他 3 个的网络连接不太稳定，不妨假设该节点不能收到其他节点的消息但能给其他节点发送消息。由于收不到 leader 的心跳包，该节点自增 Term 发起选举，导致正常工作的集群 leader 下线重新选举，这种情况会一直反复导致集群无法正常工作。

Q3 中提到的 PreVote 机制可以避免这种情况，然而<u>问题并没有解决</u>，再考虑下面的情况：
![](/img/raft/Q10_1.png)
> 图中描述的情况是：leader 节点（D）与节点 B、E 之间存在可靠连接，与节点 A、C 之间存在不稳定的连接，节点 A、B、C 之间存在可靠的连接。初始时 leader 能收到大多数（包括自己）节点的回复，整个集群正常工作，而此时节点 E 崩溃下线。

很明显，A、C 两个节点不能按时收到 D 的心跳包，会超时发起选举，假设 A 率先超时向 B、C 发起 PreVote 请求，但由于节点 B 能收到心跳包，所以不会同意 PreVote 请求（因为 PreVote 包含的 Term 和节点 B 一样大，但节点 B 的日志更新，或者它们的日志一样新，但 B 在 lease 中），因此节点 A、C 不可能获取到多数选票当选 leader。<u>该集群的问题是</u>不能选出新的 leader，而当 leader（节点 D）的 AppendEntries 又只能达到两个节点（包括自己）不满足多数原则，因此整个集群无法取得进展，依然不满足活性。

PreVote 机制本来是为了增强 raft 集群可用性（活性）而设计的，而在这里却<u>起了反作用</u>：5 个节点的集群应该可以容忍 2 两个节点的失败，但增加 PreVote 后反而无法容忍仅仅一个节点的失败。如果没有 PreVote，节点 A/C 是有可能当选新 leader 的，整个集群也能正常服务。

因此 raft 需要一种机制让 leader 主动下台：如果 leader 没能收到大多数节点的回复，那么就主动下台，等待集群节点超时重新选举，<u>etcd 把这一优化叫做 checkQuorum</u>. PreVote 确保了一旦 leader 选举成功，整个系统将是稳定的，再结合 checkQuorum，就可以解决 raft 算法的活性问题了。


## Q11：Raft 批处理和流水线是什么
![来自深入理解分布式系统](/img/raft/Q11_1.jpg)
如上图，leader 在收到请求后一般是先落盘然后再并行向 follower 复制日志，其实 leader 的落盘可以和发送复制请求并行，在 TinyKV 项目中，日志的落盘是上层模块负责的，看样子是实现了这个优化的。

再者，leader 可以累计一部分日志后在将它们复制给 follower，也就是在一次 RPC 中尽量发送更多的日志，同时日志落盘时也可以采用 batch 方式一次写入多条日志，TinyKV 中也是这样做的。

如果只是以批处理的方式发送消息，那么领导者还是需要等待跟随者返回才能继续后面的流程，<u>对此可以使用流水线 (pipeline)</u> 来进行加速。前文提到，领导者会维护一个变量 nextIndex 来表示下一个给跟随者发送的日志位置，在大部分情况下，只要领导者与跟随者建立了连接，我们都会认为网络是稳定互通的，网络分区只是小概率事件。所以，当领导者给跟随者发送了一批日志消息之后，它可以直接更新 nextIndex，并且立刻发送后续的日志，不需要等待跟随者的返回。如果网络出现了错误，或者跟随者返回了一些错误，那么领导者可以重新调整 nextIndex 的值,然后重新发送日志。

Appendentries RPC 对于日志一致性的检查保证了流水线的安全性。如果 RPC 失败或超时了，那么领导者就要将 nextIndex 递减并重试一次日志复制。如果 AppendEntriesRPC 一致性检查还是失败，那么领导者可能进一步递减 nextIndex 并重试发送前一个记录，或者等待前一个记录被确认。最初的线程架构阻碍了流水线的实现，因为它只支持串行地向每个跟随者发送一个 RPC 请求。想要<u>支持流水线，领导者必须以多线程的方式与一个跟随者建立多个连接</u>。

如果领导者不使用多线程，那么效果会是怎样的呢？其实这样的流水线和批处理没有多大区别，在 TCP 网络层面，消息已经是串行的了，TCP 本身就有滑动窗口来做批处理优化，同时单条连接保证了消息很少会乱序。<u>使用多线程连接是安全的</u>。即使因为在多个连接中请求不能保证请求有序，但在大部分情况下还是先发送的请求先到达。即使后发送的请求先到达了，由于存在AppendEntriesRPC 的一致性检查，后发送的请求自然会失败，失败后重试即可。

<u>raft 算法系统的整体性能在很大程度上取决于如何安排批处理和流水线</u>。如果在高负载的情况下，一个批处理中积累的请求数量不够，那么整体处理效率就会很低，导致低吞吐量和高延迟。另一方面，如果在一个批处理中积累了太多的请求，那么延迟将不必要地变高，因为早期的请求要等待后来的请求到达才会发送批处理请求。



## Q12：MultiRaft 是个啥
{% post_link "PingCAP TinyKV 总结" %}


## Q13：Raft 的日志压缩
一般日志采用 LSM 存储引擎存储，可以参考文章 {% post_link "索引篇三：LSM Tree" %}.