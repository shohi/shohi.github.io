---
layout: post
title: memberlist源码分析
category: tech
guid: 994A81CC-2294-488A-AE42-CFB9BB897186
tags: [tech,gossip,go]

---
{% include JB/setup %}

[memberlist](https://github.com/hashicorp/memberlist)是基于gossip协议来进行节点相互发现的Go Package, 节点发现需要解决以下两个问题:
* 如何发现其他节点
* 如何判断其他节点是否依然存活

memberlist对此的解决方案是使用`gossip`发现及使用`probe`判断, 本文对`memberlist`的关键源码进行了分析, 以期了解其具体的实现细节, 来更为合理地使用.

`memberlist`中使用`Create(*Config)`方式创建一个新的Node, 新节点创建之后就已经开始工作了.一般来说,新节点要加入cluster, 需要首先与cluster中的一个成员建立连接--即调用`Join(existings []string)`. 在实际应用中, 经常会选择几个节点当种子节点(seed), 让新成员都与它们相连, 进而达到整个cluster相互发现的目的.

对于seed, 它其实是不需要调用`Join`, 等待其他节点主动来连就行了. 下面将简述`Create`及`Join`内部的工作流程.

## 流程

### 1. Create

- 调用`newMemberlist`创建新的节点
    + 创建监听服务, 同时监听tcp与udp端口
    + 启动tcp连接处理程序- `m.streamListen`
        * 每个tcp连接都会新启一个单独的goroutine来处理
    + 启动udp连接处理程序- `m.packetListen`
        * 由于udp包一般不涉及连接,udp包都被统一放到`transport.PacketCh`
    + 启动单独的goroutine处理, 来循环处理不同的packet - `m.packetHandler`
        * `packetListen`处理udp包，然后消息的类型-或者直接处理或者放到PriorityMsgQueue
        * 一旦有消息放到PriorityMsgQueue, `packetHandler`内部循环就会被触发(通过`m.handoffCh`)
    + 由于监听服务都是被动接受请求的，所以驱使它们工作的是其它节点发的probe和gossip.

- 将本节点设置成alive状态 - `setAlive`
    + 调用`Delegate`, 添加用户自定义的metadata
        * `Delegate`是memberlist提供的hook, 可以在memberlist的流程中加入用户
        * 自定义的功能或逻辑, 一般场景下用户都会实现`Delegate`接口, 使memberlist
        * 与业务更好的结合.
    + 创建新的`alive`消息
    + 调用aliveNode, 以更新本地的节点map
        * 这是十分重要的方法,被用来更新节点的状态
        * 如果`alive`消息中的主体是自己, 那么它还可能再向外广播一个消息
            - 自己并没有dead, 同时增加自己的版本号(incarnation)
            - 所有低版本的Suspect等消息

- 启动调度程序-周期性的执行下面任务, 每个任务以独立的goroutine启动 (memberlist.go#m.schedule)
    + 探测已发现的节点状态-`probe` (state.go#m.probe),
    + 与邻居节点全量同步所发现的节点 - `pushPull` (state.go#m.pushPullTrigger)
    + 广播自己已发现的节点- `gossip` (state.go#m.gossip)

**消息类型**
memberlist之间常发的UDP消息类型如下, 更为详细的内容在`m.packetHandler`中

- suspect - 怀疑某个节点已挂了
- dead - 确认某个节点已挂
- alive - 确认某个节点活着

这三种消息分别对应节点的三个状态 - `nodeState`(state.go), 是当前节点观察到的其他节点(包括自己)的状态. 当前节点认为自己的状态只可能有两种 -- `alive`和`dead`, 其中`dead`也是主动调用`m.Leave`才会进入的状态. 对于节点的每一种状态变更，都会触发相应的函数-以将这种变更传播出去(gossip).

节点的状态有四种

- StateAlive
- StateSuspect
- StateDead - target节点不可达, 多次Ping都不通.
- StateLeft - target节点主动Leave时, 当前节点会将其的状态置成Left.

#### 1.1 aliveNode

`aliveNode`的作用是将targe节点设成alive并广播消息, 其触发时机

* 当前节点新加入cluster, 会将target节点设置成自己
* 当节点收到alive消息
* 当前节点主动调用UpdateNode, 利用Delegate机制更新自己的metadata, 进而广播出去
* 在与邻居节点全量同步状态时, 如果邻居节点的列表中自己没有的alive节点, 那么当前节点会调用aliveNode

流程如下:
- 从自己的node map中取出targe节点的状态信息
- 如果target节点找不到, 那么会在map中新建一条记录
- 如果target节点存在, 判断是否更新target节点的信息
    * 如果target节点的Addr有变化(IP或者Port与之前的不同), 进而判断能不能reclaim
        + target节点的状态为Left, 或者
        + 设置了`DeadNodeReclaimTime`, 且状态为Dead, 同时上一次的状态变化到现在的时间长于
        + `DeadNodeReclaimTime`
    + 可以reclaim的话, 就更新map中对应的记录
    * 如果不能reclaim, memberlist就会打印一条Conflict错误信息并返回
- 检查target节点是否是当前节点, 如果是的话，就调用`refute`, 向外驳斥自己还活着
    + 自增自己的版本信息
    + 广播更新后的含最新版本信息的alive消息
- 如果target节点不是当前节点, 那么更新节点状态并调用`encodeBroadcastNotify`, 广播这条消息


注意: 在K8s中pod重启会导致其地址变化, 为了让重启后的pod尽快加入cluster, `DeadNodeReclaimTime`应该设置且设为一个的值.

#### 1.2 suspectNode

`suspectNode`的作用是将targe节点设成suspect状态并广播消息, 其触发条件是

* 当节点收到suspect消息
*


#### 1.3 deadNode

当节点收到dead消息或者当前主动调用Leave时, 会触发suspectNode调用.


### 2. Join
节点要加入cluster, 需要首调用Join方法与已在cluster中的节点建立连接. 如上所述，通常的做法是每个新节点都与指定的seed相连. 当然, Join之前首先需要Create创建节点. Join主要做一件事情 - `m.pushPullNode`

* 与每个指点定的节点建立TCP连接, 然后发Push/Pull请求, 交换两个节点的全量信息

为什么是Push/Pull? 这是因为建立TCP连接后,当前节点首先将自己的状态Push给对端, 然后请对端节点将其记录的节点信息发过来(可以看作当前节点Pull对端的信息).


### 3. Probe

Probe通常使用UDP包发送探测所用的Ping消息, 期待目标节点回Ack(state.go#probeNode). 当直接UDP ping失败后, 当前节点会请其他邻居节点帮其探测--这种间接探测被称为--IndirectPing, 因为这个Ping还是当前节点用于判断目标节点是否可达. IndirectPing的次数由IndirectChecks配置控制, 默认是3次.

如上所述, 节点在Create过程中就会启动定时probe, 探测的间隔及超时控制分别由`ProbeInterval`与`ProbeTimeout`指定, 默认值为1s和500ms.

Probe执行主体是`m.probe`, 流程如下
- 随机从当前获得的members中找一个Node, 当作target Node
- 如果target是自己或者其状态是dead, 那么回到开始重新选
- 选定之后, 调用`m.probeNode`对这个target进行探测
    + 创建一个ping消息
    + 初始化针对这个ping消息的ackHandler
        - 这个ackHandler放在内部变量`m.ackHandlers`中
        - 当超时或者成功返回, `m`将从这个集合中删除该handler
    + 如果target节点是alive状态, 直接发ping由UDP发送
    + 否则, 将新增一个suspect消息, 连同ping一起由UDP发送
    + 等待ack消息
    + 如果ack成功返回, 那么probe返回
    + 否则, 随机从当前memberlist中选择给定数量的邻居节点, 进行IndirectPing
        - 数量为IndirectChecks
    + 如果有邻居节点能ping到target, 那么probeNode返回
    + 如果连IndirectPing也失效, 那么尝试使用TCP进行ping
    + 如果TCP ping成功, 那么输出一条warning而且probeNode返回
        - 输出warning是因为TCP ping能成功, 而UDP全部失败的, 很有可能是配置问题, 因而输出一条log
    + TCP也失败, 那么调用`m.suspectNode`流程

#### 3.1 suspectNode
当当前节点无法成功ping target节点时, 将会继而调用`suspect`流程. 除了probe会触发`suspectNode`之外，当前接点收到suspectMsg也会触发.

suspectNode主要是广播suspect消息, 并更新本地保存的target节点状态. 具体流程如下
- 检查被怀疑的节点是否在本地列表中存在, 或者版本是否较新
- 如果回答是否定的, 那么将忽略这个消息, 因为消息已经过时了
- 检查是否应该再次广播这个消息
    + 由suspicion来负责
    + 消息是否再次广播, 有很大的影响
        - 如果重复广播次数太多, 会造成消息flooding, 导致网络拥塞效率低
        - 如果广播次数太少, 则消息传播的速度就会下降, 导致收敛慢
        - memberlist根据概率模型来判断是否应该再次广播

#### Question
1. 帮忙的邻居节点可不可以利用IndirectPing, 来判断其与target之间的联通性, 还是必须得再次走一遍probe流程?

### 4. Broadcast

所有需要广播的消息都会直接或者间接地调用`encodeBroadcastNotify`, 这个方法只有一个作用--将消息放到broadcasts的queue中--`queueBroadcast`, queue中的消息会被`gossip` goroutine定时读取, 然后通过UDP发送出去. memberlist中每个消息的都会被发送`N`次 - `N`是根据概率模型计算得出的, 这是为了尽快地将消息扩散到整个cluster同时又不至于造成message flooding, 因此memberlist内部自定义了特殊的queue结构
-- `TransmitLimitedQueue` 来满足这种需求.


## Timeout及Interval
memberlist的配置中包含多种时间设置, 包括Timeout和Interval(config.go)

- TCPTimemout: 建立Stream连接的Timeout, 也即TCP连接建立的超时时间, 局域网场景下默认是10s.
    下文中所指的默认值都是针对局域网而言(也即config.go#DefaultLANConfig). memberlist同时
    使用TCP/UDP进行消息传输.

- PushPullInterval: 使用TCP连接，与邻居节点进行信息全同步的时间间隔, 默认是30s.
- ProbeTimeout: 一次Probe的超时时间, 默认是500ms.
- ProbeInterval: Probe执行的间隔, 默认是1s.

## 问题
### 1. Memberlist启动之后

### 2. Push/Pull什么时候触发
- 节点调用Join方法加入cluster时, 主动向seed发Push/Pull请求
- 每个节点(包括seed), 都会定期地随机选择几个邻居节点, 发送Push/Pull请求来交换全量信息
    * `m.pushPullTrigger`
- `Initiating push/pull sync`

### 2. Delegate的`LocalState`什么时候被调用
