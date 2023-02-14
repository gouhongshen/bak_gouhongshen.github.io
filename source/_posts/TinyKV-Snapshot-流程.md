---
title: TinyKV Snapshot 流程探秘
date: 2022-06-16 18:56:01
tags: 
- TinyKV

categories:
- [分布式]
- [数据库]
sticky: 1
# index_img: /img/tinykv_snapshot/tinykv_snapshot_cover.png
---


## 前言
在写 TinyKV 时，有一个部分很难理解，那就 Snapshot 的收发过程。
为了防止内存中的日志条目无限扩张，Raft 会定时/定量清理日志（如已经提交了的日志），一些节点可能由于是新加入或者网络等原因，其想要复制的日志已经被 leader 清理出内存了，此时，leader 会给该节点发送一份 Snapshot 使其快速跟上。在实现代码时，Raft 只是使用 Snapshot 的元数据来更新了一些状态，并没有涉及的日志的追加等操作，深感疑惑。而且，Snapshot 一般很大，虽然可以作为普通消息处理，但可能会阻塞正常的流程，所以对它的收发过程也很感兴趣。为了搞清楚这些问题，追踪代码调用，总算是搞清楚了。下面分为 Snapshot 的发送、接收、处理几个方面解密。

## Snapshot 流程总览
这里先给出 snapshot 各个部分的流程示意图，下面会对各个部分详细分析
![TinyKV 的整体架构（代码层面）](/img/tinykv_snapshot/tinykv_arch.png)
![snapshot 创建流程](/img/tinykv_snapshot/fig_for_create.png)
![snapshot 发送流程](/img/tinykv_snapshot/fig_for_send.png)
![snapshot 接收流程](/img/tinykv_snapshot/fig_for_recv.png)
![snapshot 应用流程](/img/tinykv_snapshot/fig_for_apply.png)

## Part 1：Snapshot 的创建
``` golang
func (r *Raft) sendAppend(to uint64) {
    term, err := r.RaftLog.Term(r.Prs[to].Next - 1)
    if err != nil {
        // the peer left too far behind  (or newly join), 
        // send it a snapshot to catch-up
        r.trySendSnapshot(to)
        return
    }
    // something else
}

func (r *Raft) trySendSnapshot(to uint64) {
    snapshot, err := r.RaftLog.storage.snapshot()
    if err != nil {
        return
    }

    r.msgs = append(r.msgs, SnapshotMessage{...})

}
```
第一个函数表明了 Raft 发送 Snapshot 的时机，第二个函数表明了 Snapshot 来自 storage。
> 这个 storage 在 2A部分和之后的部分是不一样的，值得分析下。在 2A 中，storage 接口由 MemoryStorage 实现，这货存在于内存中，文档中写着，放入 storage 的日志是持久化的（stabled），当时很不理解，因为它也是存在内存中的啊，做到后面才发现，这里的 MemoryStorage 主要起着测试的作用，你只需要闭只眼假装它真的持久化了就行。而在后面的部分，storage 接口由 badger.DB（engines.Raft）实现，是实打实的写入磁盘。

那么，调用 storage.snapshot() 实际做了什么？
```golang
func (ps *PeerStorage) Snapshot() (eraftpb.Snapshot, error) {
    var snapshot eraftpb.Snapshot
    if snapshot_is_generating {
        snapshot <- ps.snapState.Receiver
        return snapshot, nil
    }

    // something else

    ch := make(chan *eraftpb.Snapshot, 1)
	ps.snapState = snap.SnapState{
		StateType: snap.SnapState_Generating,
		Receiver:  ch,
	}
	// schedule snapshot generate task
	ps.regionSched <- &runner.RegionTaskGen{
		RegionId: ps.region.GetId(),
		Notifier: ch,
	}

    return snapshot, raft.ErrSnapshotTemporarilyUnavailable
}

```
代码很清楚，如果当前正在生成 snapshot，那么就等待它生成完成并返回，否则，就创建一个 RegionTaskGen 任务发送给 regionSched 通道，并返回暂时不可用错误。那么是谁在接收该任务呢？

上面说到 Raft 调用生成 snapshot 的接口，该接口的实现（PeerStorage）会创建一个 RegionTaskGen 任务发送给 regionSched 通道。该通道的消费者实际上是 regionWorker：
```golang
func (w *Worker) Start(handler TaskHandler) {
	go func() {
		for {
			Task := <-w.receiver
			if _, ok := Task.(TaskStop); ok {
				return
			}
			handler.Handle(Task)
		}
	}()
}

func (r *regionTaskHandler) Handle(t worker.Task) {
	switch t.(type) {
	case *RegionTaskGen:
		task := t.(*RegionTaskGen)
		// It is safe for now to handle generating and applying snapshot concurrently,
		// but it may not when merge is implemented.
		r.ctx.handleGen(task.RegionId, task.Notifier)
	case *RegionTaskApply:
		task := t.(*RegionTaskApply)
        // apply received snapshot
		r.ctx.handleApply(task.RegionId, task.Notifier, task.StartKey, task.EndKey, task.SnapMeta)
	case *RegionTaskDestroy:
		task := t.(*RegionTaskDestroy)
		r.ctx.cleanUpRange(task.RegionId, task.StartKey, task.EndKey)
	}
}
```
regionWorker 创建一个协程，处理接收到的各种任务。其中有两种任务是本文需要关注的：
1）RegionTaskGen（生成 snapshot）
2）RegionTaskApply（应用从其他 peer 接收来的 snapshot）

继续看 RegionTaskGen 任务是如何执行的：
```golang
func (snapCtx *snapContext) handleGen(...) {
	snap, err := doSnapshot(...)
	if err != nil {
		notifier <- nil
	} else {
        // notify task done to task creator
		notifier <- snap
	}
}

func doSnapshot(...) (*eraftpb.Snapshot, error) {
	log.Debugf("begin to generate a snapshot. [regionId: %d]", regionId)
    // kvDB !!!
	txn := engines.Kv.NewTransaction(false)
    //...
	err = s.Build(txn, ...)
    //...
	return snapshot, err
}
```
该 Build() 函数最终会调用 snapBuilder.build()，函数会扫描 PeerStorage.kvDB 的数据，创建一份快照：
```golang
func (b *snapBuilder) build() error {
	defer b.txn.Discard()
	startKey, endKey := b.region.StartKey, b.region.EndKey

    // all data will store in b.cfFiles
	for _, file := range b.cfFiles {
		cf := file.CF
		sstWriter := file.SstWriter

		it := engine_util.NewCFIterator(cf, b.txn)
		for it.Seek(startKey); it.Valid(); it.Next() {
			item := it.Item()
			key := item.Key()
			
			value, err := item.Value()
			if err != nil {
				return err
			}

			cfKey := engine_util.KeyWithCF(cf, key)
            // store data
			if err := sstWriter.Add(cfKey, y.ValueStruct{
				Value: value,
			}); err != nil {
				return err
			}
		}
		it.Close()
	}
	return nil
}
```
自此，Snapshot 的创建过程分析完成
## Part 2：Snapshot 的发送
当 sendAppend() 函数获取到创建的 snapshot 后，会将其封装在 pb.MessageType_MsgSnapshot 消息中，等待 RaftStorage 层调用 rawNode.Ready()：
```golang
func (d *peerMsgHandler) HandleRaftReady() {
	// Your Code Here (2B).
	rd := d.RaftGroup.Ready()
	d.sendMessageToPeers(rd.Messages)
    // ...
	d.RaftGroup.Advance(rd)
}
```
sendMessageToPeers() 最终会调用 WirteData() 函数：
```golang
func (t *ServerTransport) WriteData(...) {
	if msg.GetMessage().GetSnapshot() != nil {
		t.SendSnapshotSock(addr, msg)
		return
	}
	if err := t.raftClient.Send(storeID, addr, msg); err != nil {
		log.Errorf("send raft msg err. err: %v", err)
	}
}
```
可以看到，在 WriteData 中对 snapshot 消息做了一个拦截，采用另外的方式单独处理：
```golang
func (t *ServerTransport) SendSnapshotSock(...) {
	t.snapScheduler <- &sendSnapTask{
		addr:     addr,
		msg:      msg,
	}
}
```
这下明白了，由于 snapshot 比较大，会采用分块传输，对它的发送操作与普通的消息分开，由 sendSnapTask 异步完成。
继续探寻该任务是如何被执行的，该任务被 snapWorker 接收，并调用 Handle() 处理：
```golang
func (r *snapRunner) Handle(t worker.Task) {
	switch t.(type) {
	case *sendSnapTask:
		r.send(t.(*sendSnapTask))
	case *recvSnapTask:
		r.recv(t.(*recvSnapTask))
	}
}

func (r *snapRunner) send(t *sendSnapTask) {
	t.callback(r.sendSnap(t.addr, t.msg))
}

func (r *snapRunner) sendSnap(...) error {
    // ...
	buf := make([]byte, snapChunkLen) // snapChunkLen = 1024 * 1024
	for remain := snap.TotalSize(); remain > 0; remain -= uint64(len(buf)) {
		_, err := io.ReadFull(snap, buf)
		if err != nil {
			return errors.Errorf("failed to read snapshot chunk: %v", err)
		}
		err = stream.Send(&raft_serverpb.SnapshotChunk{Data: buf})
		if err != nil {
			return err
		}
	}
    // ...
}
```
可以看到最终 snapshot 以 snapChunkLen 为单位分块发送出去的，后面的事情就是 gRPC 的工作了，探秘自此结束。 

## Part 3：Snapshot 的接收
当使用 gRPC 发送 snapshot 时，对应 peer 也就进入了接收流程。上面提到的 snapWorker 也会处理接收操作，这里不再赘述。当所有的 snapshot 分块都接受完成后，就会给 raftWorker 监听的管道发送消息，最后调用 rawNode.Step() 让 raft 调用 handleSnapshot() 处理。 

## Part 4：应用来自其他 Peer 的 Snapshot
handleSnapshot() 接收到 snapshot 后只是更新了一些元数据，并将 snapshot 赋值给 pendingSnapshot，等待上层调用 Ready() 获取 pendingSnapshot：
```golang
func (d *peerMsgHandler) HandleRaftReady() {
	// Your Code Here (2B).
	rd := d.RaftGroup.Ready()
	applySnapResult, _ := d.peerStorage.SaveReadyState(&rd)
	//...
	d.RaftGroup.Advance(rd)
}

func (ps *PeerStorage) SaveReadyState(ready *raft.Ready) {
	// Your Code Here (2B/2C).
	applySnapResult, err := ps.ApplySnapshot(&ready.Snapshot, ...)
	if err != nil {
		panic(err)
	}
    // ...
}

func (ps *PeerStorage) ApplySnapshot(snapshot *eraftpb.Snapshot, ...) {
    // ...
	// send runner.RegionTaskApply task to region worker through 
    // PeerStorage.regionSched and
	// wait until region worker finishes
	ch := make(chan bool, 1)
	ps.regionSched <- &runner.RegionTaskApply{
		RegionId: snapData.Region.GetId(),
		Notifier: ch,
		SnapMeta: snapshot.Metadata,
		StartKey: snapData.Region.StartKey,
		EndKey:   snapData.Region.EndKey,
	}

	// waiting
	<-ch
    return ...
}
```
上面代码很明显了，上层获取到 Ready.Snapshot 后，会创建 RegionTaskApply 任务通过 regionSched 通道发送给 RegionWorker -> handle() -> handleApply() 处理：
```golang
func (r *regionTaskHandler) Handle(t worker.Task) {
	switch t.(type) {
	case *RegionTaskGen:
		task := t.(*RegionTaskGen)
		// It is safe for now to handle generating and applying snapshot concurrently,
		// but it may not when merge is implemented.
		r.ctx.handleGen(task.RegionId, task.Notifier)
	case *RegionTaskApply:
		task := t.(*RegionTaskApply)
		r.ctx.handleApply(task.RegionId, task.Notifier, task.StartKey, task.EndKey, task.SnapMeta)
	case *RegionTaskDestroy:
		task := t.(*RegionTaskDestroy)
		r.ctx.cleanUpRange(task.RegionId, task.StartKey, task.EndKey)
	}
}

func (snapCtx *snapContext) handleApply(...) {
	err := snapCtx.applySnap(regionId, startKey, endKey, snapMeta)
	// ...
}

func (snapCtx *snapContext) applySnap(...) {
	applyOptions := snap.NewApplyOptions(snapCtx.engines.Kv, &metapb.Region{
		Id:       regionId,
		StartKey: startKey,
		EndKey:   endKey,
	})
	if err := snapshot.Apply(*applyOptions); err != nil {
		return err
	}
}
```
可以看到，上面一步步调用，最后调用 snapshot.Apply()，注意这里传入的是 badger.kvDB。
snapshot.Apply() 和上面提到的 snapshot 创建过程的 snapBuilder.build() 执行的是相反的步骤，即，将 snapshot 中的内容写入到磁盘:
```golang
func (s *Snap) Apply(opts ApplyOptions) error {
	externalFiles := make([]*os.File, 0, len(s.CFFiles))
	for _, cfFile := range s.CFFiles {
		file, err := os.Open(cfFile.Path)
		if err != nil {
			log.Errorf("open ingest file %s failed: %s", cfFile.Path, err)
			return err
		}
		externalFiles = append(externalFiles, file)
	}
    // write to DB
	n, err := opts.DB.IngestExternalFiles(externalFiles)
    // ...
}
```
snapshot 的应用分析自此结束。

## 总结
通过上面的分析，可以得到以下信息：
1. snapshot 的创建、发送、接收和处理都与 Raft 无关，它无需关系具体数据（除了元数据）。
2. snapshot 的发送和接收都采取了单独的 RPC 异步处理。
3. 生成 snapshot 需要从 kvDB 中读取数据，然后返回给 Raft，最后通过 Ready 交给上层发送。
4. snapshot 接收后，需要先交给 Raft 更新一些元数据，然后通过 Ready 交给上层写到 kvDB 中。
