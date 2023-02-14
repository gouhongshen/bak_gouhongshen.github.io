---
title: badgerDB 中的 LSM
date: 2022-08-04 22:07:00
tags:
sticky: 1
---

badgerDB 基本是按照[ WiscKey 论文 ](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf) 实现的，在 {% post_link "LevelDB 中的 LSM" levelDB %} 基础上专门为 SSD 做了优化。

<font color=red>第一个优化，Key 和 Value 分开存储</font>。在 levelDB 中 key-val 是一起插入 LSM 中的，当某两层归并时 key 和 val 都会被读入内存、排序然后再写回磁盘，事实上只有 key 参与了排序，如果将 val 存在另外一个地方，合并时只需要读取 key，就可以有效的减少写放大。

![](/img/badgerDB/1.png)

如上图，badgerDB 将 value 单独存在一个名为 vLog（value log）的文件中，而 LSM 中存储的则是 [key, <vlog_offset, val_size>]，当然了要 addr 远小于 val 时才够划算，算一下每层最坏情况的写放大：
> 假设 key=16B，addr=16B，val=1KB，如果是普通 LSM 写放大为 10，badgerDB 则为 (10 * (16 + 16) + 1024) / (16 + 16 + 1024) = 1.27! 

这样一来，<u>当插入记录时</u>，val 首先会被追加到 vlog 文件，key 和该 val 的地址会被插入普通的 LSM 中；<u>当删除一条记录时</u>，仅仅只是将 [key, addr] 从 LSM 中删除（插入删除标记），不需要变动 vlog，那些没有 key 指向的 val（被删除或被重写）会在垃圾回收（garbage collection）流程中处理；<u>当查询记录时</u>，首先会在 LSM 中查找，若该记录已经被删除了，返回 not found，否则根据 LSM 中查找到的 addr 再去 vlog 中读取实际的数据。
> 这里相比 levelDB 多了一步随机读操作，好像性能会更低，实际上由于 key-val 分开存储后，整个 LSM 存储的 SSTable 会小很多，比如现有 key=16B，addr=16B，val=1KB 的 100GB key-val 记录，如果采用 levelDB，加上各种 index，LSM 会远大于 100GB，如果采用 badgerDB，其 LSM 只有 3GB，首先是层级会更少，其次甚至整个 LSM 都可以缓冲到内存中。

<br>

<font color=red>第二个优化，提升范围检索性能</font>。levelDB 将 key 和 val 放在一起，范围查询时直接顺序扫描即可，而 badgerDB 将 key-val 分开存储，范围扫描时得到的是一组 val 的地址，会存在大量的随机读，下图展示了 SSD 顺序读和随机读的吞吐量比较：

![](/img/badgerDB/2.png)

很明显 badgerDB 必须要使用多线程来提升随机读的性能才能追上顺序读。大致流程是这样的，当上层在迭代器上调用 Next() 接口时，badgerDB 会从 LSM 中预取后面的多个记录，将这些记录中的 val 地址插入到队列中，然后多个线程在后台去 vlog 中读取数据。

如果是 HDD 的话是做不到的，还是会因为随机访问降低性能，LSM 带来的好处荡然无存。

<br>

<font color=red>第三个优化，垃圾回收</font>。所谓垃圾回收就是将那些已经删除或覆写的记录空间腾出来。levelDB 在删除或更新记录时，只是在 LSM 中插入了新的记录或一条删除标记，并不会立即删除过时的数据，真正的空间回收发生在合并的时候，如果合并时发现过时的记录就丢弃。badgerDB 的 [key, addr] 存储在 LSM 中，并不用担心它们的回收，而 val 单独存在 vlog 中，若不删除过时的数据，vlog 将变得越来越庞大。

一个比较朴素的方式是，先扫描一遍 LSM 收集过期的 [key, addr]，然后再根据 addr 去删除 vlog 中的 val. 很明显该方法会严重干扰系统的正常服务，所以 badger 采用如下的方式：
![](/img/badgerDB/3.png)
如上图，badgerDB 对 vlog 存储的记录格式做了一些变更，由 [val] -> [key_size, val_size, key, val]. 可以设置一个阈值，当 vlog 大小达到阈值后开始垃圾回收，也可以定时清理，总之<u>当垃圾回收线程启动后</u>（下称 gcWorker），gcWorker 会从 vlog tail 位置开始一次性读取数个 val 记录，提取其中的 key 去 LSM 中查询该 key 是否过期（可以根据 addr 判断），若过期就将该 val 记录丢弃，若未过期就将该 val 记录 append 到 vlog 的 head 位置。<u>为了防止 append 的时候发生故障</u>，append 的数据会调用 fsync() 同步刷写到 vlog 中，同时将新的 [key，addr] 插入到 LSM 中，还需要将更新后的 tail 插入到 LSM 中，记录格式为 ["tail", \<tail-vLog-offset\>]，head 倒不用记录，因为它就是文件的最后一个字节。

<br>

<font color=red>第四个优化，干掉 WAL 日志</font>。levelDB 中有一个 WAL 日志，新的变更操作都会先刷写进 WAL 后才会进入 memTable，为的是防止 故障发生时，memTable 和 immutable 中的记录丢失。在 badgerDB 中 vlog 其实就已经充当了 WAL 日志，为什么？首先只有当 val 记录写入了 vlog 文件中，[key, addr] 才有可能写到 SSTable 中。当系统重启后，只需要扫描 vlog 文件就能恢复 SSTable. 不过这个方法太低效，需要扫描整个 vlog 文件，所以 badgerDB 会周期性的将 head 记录同步刷写到 LSM 中，格式为 ["head", head-vlog-offset]，这样一来，系统重启后先从 LSM 中查找最近一次的 head，然后从该位置扫描到文件尾进行数据恢复，整个方法有点像 {% post_link "数据库恢复系统" checkpoint %}.


<br>

<u>另外，关于 SSD</u>，最初 LSM 更多用在大量 HDD 写场景的，假设一个高端 HDD，有 10ms 寻道延迟，100MB/s 的数据传输速率，随机访问 1KB 数据的时间大约为 10ms，而顺序读写 1KB 数据的时间只有 10μs，随机读比上顺序读高达 1000:1（写操作也差不多），所以对于存在大量随机写的场景使用 LSM 吞吐量就会好很多，虽然 LSM 会带来写放大，但也远远小于 1000. 

SSD 的随机和顺序读写延迟大概为 200:1，远低于 HDD，似乎 LSM 对 SSD 效果没那么大，其实不是，首先是吞吐量的提升，其次 SSD 是有擦除（写入前必须擦除）次数限制，而且 SSD 后台还会进行垃圾回收：将分散的 block 放在一起，再擦除被移动的 block. LSM 本身就是顺序追加，会大幅度减少垃圾回收的次数。

badgerDB 针对 SSD 擦除寿命限制这个特点，将 key val 分开存储，进一步减小 LSM 的写放大，同时利用 SSD 内部并行性特点，提高范围检索时对 val 读取的吞吐量。

论文中还有很大内容，比如实验对比、崩溃一致性、相关工作等，可以去看看。

