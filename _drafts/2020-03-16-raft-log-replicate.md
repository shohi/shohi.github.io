---
layout: post
title: Raft中的log复制
category: tech
guid: F23F4DF8-FDA0-4F85-B9A1-3D90F4FFEE2E
tags: [tech, raft]

---
{% include JB/setup %}

本文将尝试结合golang的[hashicorp/raft](https://github.com/hashicorp/raft/)库来解释raft协议中log复制的原理及其实现.


问题:
1. log复制发生的时机
2. log compaction之后, log如何复制
3. Apply是否是并发的?

## 流程
raft node通过NewRaft方法创建并开始工作, NewRaft将创建了3个long-running goroutine. 这3个goroutine完成raft node的所有工作(`api.go`).
    - run
    - runFSM
    - runSnapshots

创建完node以后, 一般来说必须调用`BootstrapCluster`以启动整个cluster. 这是因为NewRaft的参数中没有地方指定cluster的成员, 而`BootstrapCluster`的唯一参数就是cluster成员. 但是有另外一种情况, node可以从snapshotStore从获取成员信息，而snapshotStore对NewRaft是可见的.


### 1. run
这个方法主要用来监控node的状态变化，以启动相应的处理流程.

```go
// run is a long running goroutine that runs the Raft FSM.
func (r *Raft) run() {
	for {
		// Check if we are doing a shutdown
		select {
		case <-r.shutdownCh:
			// Clear the leader to prevent forwarding
			r.setLeader("")
			return
		default:
		}

		// Enter into a sub-FSM
		switch r.getState() {
		case Follower:
			r.runFollower()
		case Candidate:
			r.runCandidate()
		case Leader:
			r.runLeader()
		}
	}
}
```

从这个方法来看，在运行相应的流程中, node会时刻调整自己的状态, 当自己的状态不同于流程所要求的状态时会退出方法.


#### 1.1 runFollower

node在一开始启动的时候, 就处于这个状态. 在这个状态下, node被动地等待leader的RPC, 如HeartBeat/AppendEntries. 但是有一个事情是主动处理的, 就是监控heartbeatTimer, 定时去检查与node是否一直有联系, 如果从上次联系以来的时间间隔超过了heartbeatTimeout, 则将当前状态设为Candidate以重新启动raft election.

```go

    // Check if we have had a successful contact
    lastContact := r.LastContact()
    if time.Now().Sub(lastContact) < r.conf.HeartbeatTimeout {
        continue
    }

    // Heartbeat failed! Transition to the candidate state
    lastLeader := r.Leader()
    r.setLeader("")

    r.logger.Warn("heartbeat timeout reached, starting election", "last-leader", lastLeader)
    metrics.IncrCounter([]string{"raft", "transition", "heartbeat_timeout"}, 1)
    r.setState(Candidate)
    return

```

node在follower状态下，如果有client的请求，则其一律拒绝

```go

    // Reject any operations since we are not the leader
    r.respond(ErrNotLeader)
```

##### processRPC
`processRPC`是raft处理内部peer之间请求的流程, 无论node处于什么状态, 都会监听这个chan. `processPRC`仅仅简单封装了各类处理流程, 具体实现如下.


```go

	switch cmd := rpc.Command.(type) {
	case *AppendEntriesRequest:
		r.appendEntries(rpc, cmd)
	case *RequestVoteRequest:
		r.requestVote(rpc, cmd)
	case *InstallSnapshotRequest:
		r.installSnapshot(rpc, cmd)
	case *TimeoutNowRequest:
		r.timeoutNow(rpc, cmd)
	default:
		r.logger.Error("got unexpected command",
			"command", hclog.Fmt("%#v", rpc.Command))
		rpc.Respond(nil, fmt.Errorf("unexpected command"))
	}
```

从follower的角度来看, `AppendEntriesRequest`是最主要的 - 用来同步leader发来的Log replication.

##### appendEntries
首先检查request是否有效的, 比如req中term不能比目前自己持有的小等等. 检查完之后, 将request中的Log　Entries存到自己的LogStore, 然后调用`processLogs`将这些Log发送给`fsmMutateCh`, 只要发送成功就返回.

#### 1.2 runCandidate
当node的状态是Candidate时, 第一件是发起选举`r.electSelf`, 向各个server发出RequestVote, 让其他server投票给自己, 然后等待其他server的response.

选举有ElectionTimeout, 如果超过了这个时间, node重次重新执行`runCandidate`. node发出request之后，等待各server给它发response,　如果收到的选票大于quoram, 那么成功当选, 将自己设为leader并进入Leader状态, runCandidate退出.


```go

case vote := <-voteCh:
    // Check if the term is greater than ours, bail
    if vote.Term > r.getCurrentTerm() {
        r.logger.Debug("newer term discovered, fallback to follower")
        r.setState(Follower)
        r.setCurrentTerm(vote.Term)
        return
    }

    // Check if the vote is granted
    if vote.Granted {
        grantedVotes++
        r.logger.Debug("vote granted", "from", vote.voterID, "term", vote.Term, "tally", grantedVotes)
    }

    // Check if we've become the leader
    if grantedVotes >= votesNeeded {
        r.logger.Info("election won", "tally", grantedVotes)
        r.setState(Leader)
        r.setLeader(r.localAddr)
        return
    }

```

#### 1.3 runLeader
当进入leader状态，首先广播自己是Leader给监听的notifyCh. 然后初始化内部变量, 主要是`map[ServerID]*followerReplication` -- 用于向follower进行log replication. 之后, 启用replication goroutine - `startStopReplication`. 然后分发了一个No-op的log entry用于更新configuration (不明白).

这些工作做完以后, node开始执行业务逻辑循环去处理各种请求- `leaderLoop`.


##### leaderLoop
每个Leader任命都有有期限了--LeaseTimeout. 过了这个点, 要重新检查自己是否还可以继续当leader,否则应该退回candidate状态, 重新选举 -- `checkLeaderLease`.

for循环中有两类请求特别基本和关键 -- `applyCh`和`commitCh`.

- `applyCh`中是各类请求，如client发的请求, 均以raft.Log的形式出现.
- `commitCh`是指已经commited但还没有应用到FSM的请求, commited是指超过半数的node声明它们commited这个Index.

从`applyCh`取出的Log, 经过batch处理--即尽可能地一次处理多条Log, 调用`dispatchLog`对Log处理--复制分发. 每当一个follower的Log复制完成, 就会向调用leader的commitment来更新各follwer最新的commitedIndex, 如果commitedIndex的中位数比目前的大，那么向`commitCh`发消息让leader启动FSM更新流程-`processLogs`.

Question:
1. 返回给client与更新FSM的先后关系 (见`processLogs`)
2. follower什么时候返回AppendEntries RPC的结果给leader

##### dispatchLog

首先将log写入本地logStore, 如果有出错，那么node直接变成follower并退出.


##### startStopReplication
该方法主要作用是对每个server(除自己外), 启动一个独立的goroutine用于log repliacation - `r.replicate`. `r.leaderState.replState`保存了每个Server及其对应的followerReplication, 它在这个方法中被初始化, 初始化过程中会产生`PeerObservation`事件. 同时其也检查是否有Peer离开, 并将它的replication活动终止, 这个检查过程也可能产生`PeerObservation`事件.


###### replicate
`r.replicate`一开始就使用新goroutine调用`r.heartbeat`--周期性地检测peer是否存活, 接下来不断去取lastestLog, 然后调用`r.replicateTo`将logStore中截至到latestLog的日志都复制给server.

`r.replicate`控制向特定的一个folloer进行log复制, 其主循环中主要监听三个channel

* s.triggerDeferErrorCh - ??
* s.triggerCh - 当leader收到Log之后, 将其存到它的LogStore后, 就向这个ch发消息，以启动复制
* CommitTimout - 这个是定时触发的, 每过一段时间就主动去同取leader的commitIndex, 将其同步到follower上

这3个ch对应的处理方式是一样: 取leader的LastLogIndex, 然后调用`r.replicateTo`. 接着根据返回状态来决定是否启用`pipeline` 复制模式 (下面详细解释).

###### replicateTo
`r.replicateTo`首先从log中取出哪些log需要复制给follower, 如果找不到这些log, 一般来说是follower落后太多, leader已经对log进行了compaction, 这时leader转为使用snapshot发送给follower. 另外有一种情况是AppendEntriesRPC成功了，但是返回的response中follower报告自己有些log没跟上, 这时leader会根据返回的response重新计算应该将哪些log发给follower, 出现这种情况可能是follower短暂地重启了.

leader保存了每个follower下一次复制的起始index, 结合传入的LastLogIndex来筛选log. 筛选Log过程中首先找到nextIndex前的一条Log即prevLog - follower已此来检验Log的完整性. 另外为了平衡AppendEntries有可能由于prevLog不匹配而被follower拒绝，导致不必要的性能损失, 每次真的向follower发送的Entry数最大为`MaxAppendEntries` - 这是个配置, 默认是64. 当nextIndex和lastLogIndex有很多log时, `replicateTo`将多次调用`AppendEntries RPC`, 直至消息发送完. (待确认: follower节点什么时候返回? 收到就返还是将其apply到FSM后?)

如果一次PRC失败, `replicateTo`返回并将在下一次Tick或者有新Log时, 重新触发执行. 如果成功, 将调用`s.updateLastAppended`来更新其commitIndex, 同时启用`pipeline`复制模式 -- 从这可以看出pipeline是更高效的复制方法.　一次调用`replicateTo`会将所有的Log发送过去, 而pipeline模式只要有一次RPC成功就会被设置成`true`, 即使后续有失败的RPC, 这个全局变量将会在`replicateTo`返回后被使用.

`s.updateLastAppended`更新当前follower的nextIndex及其在leader的matchIndexes中的值, 随后调用`s.notifyAll`通知所有的verifyFuture更新状态.

`r.sendLatestSnapshot`从snapshotStore中取出最新的snapshort, 然后调用InstallSnapshot RPC将最新的snapshot发给follower.

###### pipelineReplicate
pipeline工作模式是同时发多个`AppendEntries RPC`, 然后异步地收结果--`pipelineDecode`, 然后更新状态.

Question
1. `configurations`成员的更新没有有lock保护, node在运行期间对其没有data race么?
2. `replicate`如果返回了是否一定说明当前node不是leader了?
3. leader如何保证一条log被quora peer所确认?
4. 在`r.setPreviousLog`中prevLog的Index是entry自己的属性, 而不是`nextIndex-1`, 这是否意味着prevLog自身的index可能不等于`nextIndex-1`, 或者这两者必然相等--所以写哪个都一样. 从inmem_store的实现看，两者是相同的, 也看到在存储多条Log时, 这些Log按照Index从小到大顺序排列的.

#### processLogs
这个方法将committed Log发送给FSM goroutine, 触发FSM状态更新. 每个node都会记录自己上次appliedIndex,方法内部首先会prepareLog-准备FSM所需要的数据结构-封装Log成`commitTuple`, 然后将`commitTuple`给成batch, 然后通过`r.fsmMutateCh`传给FSM. 只要发送成功, `processLogs`就更新appliedIndex并返回.

注意在这个过程中, processLogs并不等FSM处里完才返回. 另外Log也不在processLogs中写reponse, 而是FSM处理Log时才响应其future.

### 2. runFSM
这个方法主要是不断地监听`fsmMutateCh`和`fsmSnapshotCh`, 以将Log应用到FSM和生成snapshot. 这是个long-running方法, raft中用独立的goroutine来执行.

首先来看`fsmMutateCh`, 上文提到`processLogs`会将已将commited但还没applied的Log以`commitTuple`的形式发到这个chan上, `runFSM`收到请求后调用`commitBatch`内部方法, 来更新FSM以及给Log的Future回response.

(另外`fsmMutateCh`还处理`restoreFuture`这类消息,以后再详细分析它的作用)

#### commitBatch
首先检查FSM是否支持batchApply, 如果不支持，那么循环调用commitSingle. 如果支持, 那么该方法内部先组装batch, 然后调用FSM的BatchApply.

#### commitSingle
这个方法将一条特定的Log应用到FSM. 方法内部一开始先准备Log future的response, 然后调用FSM的Apply方法同时设置Log future的response并调用`future.response(nil)`.

也就是说只要在FSM应用时, 才会向Log future发送response!
client发送request到收到response的流程，大致是这样的:
- leader收到request, 将其转成Log
- leader将Log存到自己的LogStore, 然后复制给其他follower
- 其他follower复制成功后
- leader将Log应用到FSM
- 应用到FSM之后, 返回给client

另外有三个问题
1. 复制给follower成功是什么意思? follower接受就算成功,还是放到其LogStore, 还是应用到其FSM?
2. FSM.Apply返回的结果对Log future有什么作用? 在`runFSM`中直接调用的是`future.response(nil)`.
3. 如果Log在给client写response时出错, 那么该条Log会被raft revert么?

关于第二个问题, `FSM.Apply`返回的结果是给主动调用`raft.Apply`方法用的, 调用者可以以些来检查逻辑结果是否正确. `future.response(nil)`的涵义是说, 这条Log已经成功的被FSM处理了, 至于处理的结果是否合符业务逻辑则由`FSM.Apply`的返回值决定. FSM仅是一个接口, 其中`Apply`方法也是需要自己实现的.

对于第一个问题, 需要回到`runFollower`流程中看follower如何处理AppendEntriesRPC. 简单地说, follower收到request之后，将其保存下来并将其放到FSM的queue中, 就成功返回了. 不是等到应用到FSM才返回.

第三个问题, 则需回到`transport`的处理逻辑, 从raft内部来看没有rollback的机制(??).

### 3. runSnapshots
这个后台任务就是定时触发snapshot, 或者接收用户的snapshot请求, 去调用`takeSnapshot`. 定时触发的snapshot会首先检查要不要进行新的snapshot, 以防止新snapshot只新增了几条Log, `SnapshotThreshold`参数可以用来指定新Log的个数只有超过该阈值才会继续.

需要注意的是这里的新Log是指LogStore中存的, 并不是committed -- Log首先会存在本地，然后复制到其他Server, 超过半数server返回才认为是committed.


`r.takeSnapshot`用来向FSM请求创建新的snapshot, 然后将其写入到snapshotStore. 这里面有个需要特别注意的地方: snapshot是目前FSM的最新状态的快照, 但是是snapshot的Persist发在在`runSnapshots`这个goroutine, 而FSM的apply发生在`runFSM`中，它们是并发的. 如果在实现上snapshot仅是一个引用, 而`Persist`时才将FSM的最新状态Copy出来, 就会导致Persist出来的内容包含了`Apply`之后的Log. nats-streaming-server是这样做的, 尽管它使用了Lock来确保`Apply`和`Persist`不会同时发生, 但是`Persist`依然可能包含新的committed Log, 一个可能的过程是这样的

- `runSnapshots`向`runFSM`请求最新快照
- runFSM中调用内部`snapshot`函数, 生成最新的快照(仅是FSM, 并不是immutable copy), 这时snapshot关联的是最新的commitIndex.
- runFSM中调用了`Apply`, 将新的Log应用到FSM中, commitIndex增加
- `runSnapshots`调用`Persist`将snapshot写入到store中, 那么这时写入的状态包括了上一步新Apply的Log, 但是snapshot的index没变.

这种情况会导致compactLogs会少删除Log, 如果系统可以容忍Log的重复Apply, 这没有问题, 但是大部分场景并不是这样.

与`nats-streaming-server`相对的`rqlite`, 其`snapshot`生成的快照就是一个immutable copy, 因此其Log Compaction是十分可靠的.


`compactLogs`
snapshot存到store之后,接下来就会对LogStore进行Compact, 删除已经存在snapshot中的Log. 但是并不是所有的snapshot已存的Log都删除, 有一个阈值--`TrailingLog`, LogStore中至少还要保留这么多. 这是为什么呢? 因为如果存在snapshot中全删除了, 而一个follower仅缺少一两条之前的Log, 那么Leader也需要把整个snapshot发过去, 这其实是很不划算的.

所以LogStore中被删除的范围是[logMinInde, min(snapIndex, logMaxIndex-TrailingLog)]

Question
1. 为何不用committed来检查?
2. LogStore中的Log Index一定是严格顺序的么? 有没有可能中间少几个?
3. `CommitTimeout`与`ApplyTimeout`的区别

## Observation

raft实现中共有4类Observation, 调用者可以注册observer, 如果有相应事件发生时，会能过channel收到消息通知, 它们分别是(`observer.go`)
    - *RequestVoteRequest
	- RaftState
	- PeerObservation
	- LeaderObservation

### 1. RaftState
如果raft node的状态有变化, 就会产生RaftState事件. node在启动时是follower状态.

### 2. LeaderObservation
当node发现leader变化时, 就会产生LeaderObservation事件. node在启动时, 其leader变量是空, 意思是说现在还没有leader.

在实现上RaftState会级联触发LeaderObservation, 会有两次LeaderObservation.
- 将现有的leader置成空, 触发LeaderObservation
- 将现有的leader置成检测到的leader, 触发LeaderObservation

## Transport

### handleCommand
该方法处理transport层的request, 将request解码后变成Command, 然后封装成RPC结构. 根据RPC中command类型的不同, 调用不同的handler, 然后等待RPC的response, 最后将其写回到对端.

如果写RPC出错, 那么hanldeCommand返回, 进而触发handleConn返回, 之后conn就会被close.

```go
	// Wait for response
RESP:
	select {
	case resp := <-respCh:
		// Send the error first
		respErr := ""
		if resp.Error != nil {
			respErr = resp.Error.Error()
		}
		if err := enc.Encode(respErr); err != nil {
			return err
		}

		// Send the response
		if err := enc.Encode(resp.Response); err != nil {
			return err
		}
	case <-n.shutdownCh:
		return ErrTransportShutdown
	}
```

# Links

1. The Raft Consensus Algorithm, <https://raft.github.io/>
2. Raft一致性协议简说, <https://asktug.com/t/raft/1028>
