---
title: "Lab-3 Raft"
description: Raft 分布式一致性算法
date: 2024-09-18T02:15:02+08:00
image: 
categories:
    - 分布式
musicid:
---

# Raft
## 脑裂
对于MapReduce、GFS等分布式系统，都是一个Master节点与很多副节点组成的。这种单主结构足够简单，不会出现矛盾（毕竟master自己不会反对自己），但也带来了问题——一旦这个节点崩溃，那么整个系统不能正常工作。

以Test-and-Set为例：

![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/20240917212630.png)

实际上是单点的，我们为了提高系统的容错性，设计一个多副本的Test-and-Set服务（S1和S2）。

现在，我们来假设我们有一个网络，这个网络里面有两个服务器（S1，S2），这两个服务器都是我们Test-and-Set服务的拷贝。这个网络里面还有两个客户端（C1，C2），它们需要通过Test-and-Set服务确定主节点是谁。在这个例子中，这两个客户端本身就是VMware FT中的Primary和Backup虚拟机。

+ 如果我们规定client需要与两个server同时交互，那么会比单点更糟糕，因为一旦一个server崩溃，则不能正常服务，导致有副本系统反而没有容错性；
+ 如果我们规定只与其中一个server交互即可，则会发生脑裂。

当时（1980s）解决脑裂大致两种方法：

+ 构建不会出错的“网络”
+ 人工解决问题，出现问题时，停止提供服务，由人工检查问题



## 过半票决（Majority Vote）
**完成任何操作，都需要过半的服务器（总服务器数的一半以上）批准来完成对应操作。**

+ 服务器总数应该为**奇数**；
+ 假设系统有2*F+1个服务器，那么系统最多可接受F个服务器故障（F+1个服务器正常运行），仍然可以正常工作；
+ 过半票决的一个特性是：**每一个操作对应的过半服务器，必然至少包含一个服务器存在于上一个操作的过半服务器中**。下面的例子说明了这一点：
    - 过半票决系统的一个特性就是，最多只有一个网络分区会有过半的服务器，所以我们不可能有两个分区可以同时完成操作。这里背后更微妙的点在于，如果你总是需要过半的服务器才能完成任何操作，同时你有一系列的操作需要完成，其中的每一个操作都需要过半的服务器来批准，例如选举Raft的Leader，那么**每一个操作对应的过半服务器，必然至少包含一个服务器存在于上一个操作的过半服务器中。也就是说，任意两组过半服务器，至少有一个服务器是重叠的。**实际上，相比其他特性，Raft更依赖这个特性来避免脑裂。例如，当一个Raft Leader竞选成功，那么这个Leader必然凑够了**过半服务器的选票**，而这组过半服务器中，**必然与旧Leader的过半服务器有重叠**。所以，新的Leader必然知道旧Leader使用的任期号（term number），因为新Leader的过半服务器必然与旧Leader的过半服务器有重叠，而旧Leader的过半服务器中的每一个**必然都知道旧Leader的任期号**。类似的，任何旧Leader提交的操作，必然存在于过半的Raft服务器中，而任何新Leader的过半服务器中，必然有至少一个服务器包含了旧Leader的所有操作。这是Raft能正确运行的一个重要因素。



## Raft初探
![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/20240917212635.png)



Raft作为Library运行在服务中，为服务提供一致性保证。如上图，一个服务的副本由两部分组成：上半部分是**应用程序（APP）（上图中是一个KV Server）**，下半部分是**Raft库**。

执行流程：

1. Client发送请求至Leader节点的APP；
2. APP将Client请求向下发送至Raft层，Raft层将操作存入日志；
3. Raft将此操作发送值所有副本；
4. 当Leader得知过半的阶段都有了这个操作的拷贝后，发送消息至上层APP，告知APP可以实际执行这个操作了；此后所有其他副本都将操作提交至本地的APP执行，理想情况下，整个系统的所有副本都执行了此操作，APP与Raft状态应都保持一致。



## Log同步时序
![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/20240917212638.png)

在系统正常工作没有故障时，Raft Log 同步流程的时序图如上：

1. Client发送请求；
2. Leader向其余所有副本同步消息（**AppendEntries RPC**，用于日志同步与心跳信号传递）；
3. 待收到超半数的确认后提交（对于上面三个副本的系统，包括Leader自身只需要额外收到一个节点的确认即可）；
4. Leader将提交的消息同步给其他副本，通知它们可以提交操作（在Raft中，没有单独的提交信息，commit消息是携带在下一次的AppendEntries RPC中的）。



## 日志（Raft Log）
日志的作用：

+ Leader用来对**操作进行排序**，这对于复制状态机十分重要；
+ Follower用于**临时存储操作**，待收到Leader的commit消息执行操作；
+ Leader中存放操作，如果因为网络或其他问题导致副本没有收到消息，可以进行**消息重传**；
+ 所有节点都会将Log持久化，这有助于节点**故障后恢复状**态。



## Leader选举（Leader Election）
+ Raft中设置Leader的好处在于，对于一个请求，存在Leader的情况下，只需一轮交互即可取得过半服务器的确认，而对于无Leader系统，通常需要一轮消息确认一个临时Leader，第二轮消息才能实际确认请求，性能更高；
+ 在Raft生命周期中，通过**任期号（term number）**来区分不同的Leader，每个任期内最多只有一个Leader（可能没有或者有一个）；
+ 每个Raft节点都有一个**选举定时器（Election Timer）**，当此定时器时间耗尽之前，没有收到任何来自Leader的消息，则认为Leader已经下线，并开始一次新的选举；
    - 任何一条AppendEntries消息的到达都会重置选举定时器，因此，在网络正常且节点无故障的情况下，不会进行新的选举；
    - 选举失败可能的情况：
        * 可投票节点不足（网络故障或服务器关机等）
        * 分割选票（Split Vote）：如果若干节点选举定时器同时到期，同时都发出了RequestVote消息，这造成所有这些节点都无法获得足够的选票，就会导致这次选举失败。Raft不能完全避免这种情况发生，但是可以大大降低发生概率：**为选举定时器随机选择超时时间，以降低多个节点同时超时的发生概率。**
            + 超时时间选择：**下限**不能低于一个心跳间隔，否则会一直超时，通常可以设置为2-3个心跳时间；**上限**选择取决于期望的故障恢复速度，同时要保证随机的超时时间间隔足够完成选举。
+ 开始一次选举时，当前服务器会**增加任期号（term number）**，并发出**请求投票（RequestVote）RPC**，这个消息会发送至其他所有节点（因为自己总会投票给自己）；
+ 相同的，选举视为一次操作的话，也需要获得超半数的同意票才能当选。每个Raft节点在一个任期内只会进行一次投票，保证了一次选举最多只会当选一个Leader；
+ 当选的Leader会立刻发送AppendEntries告知其他所有节点，当然不是直接写“我当选了Leader！”，而是附带上当前任期号，而由于Raft规定，对于某个任期，只有Leader才能发出AppendEntries消息。



## 系统故障下的机制
### 日志恢复（Log Backup）
当Raft系统发生故障时，可能系统中不同节点中保存的log不一致，下面举例说明：

| 服务器 | Slot 10 | Slot 11 | Slot 12 | Slot 13 |
| --- | --- | --- | --- | --- |
| S1 | 3 | | | |
| S2 | 3 | 3 | 4 |  |
| S3 | 3 | 3 | 5 | <font style="color:#DF2A3F;">6</font> |


1. 有S1、S2、S3三个服务器，在任期3中，第一次请求三者都成功记录，第二次请求只有S2、S3记录，S1记录失败，包含三种情况（假设S3是任期3的Leader）：
    - 发送给S1的消息丢了
    - S1当时已经关机了
    - S3在向S2发送完AppendEntries之后，在向S1发送AppendEntries之前故障了
2. 继续往后看：
    - S2称为Leader后接受一个请求后崩溃，需要进行重新选举
    - S3成功当选，获得请求，并且崩溃
3. 假设下一个任期是6，并且S3当选Leader，S3会发送任期6的第一个AppendEntries RPC给其他两个follower，其中不仅包括本条log相关信息，还包括：
    - prevLogIndex 前一项log的槽位
    - prevLogTerm 前一项log的Term number
4. S2检查是否匹配，结果发现自己前一项Log的任期为4而非5，因此会拒绝这个AppendEntries RPC，并返回False给Leader S3，S1前一项没有值，因此也将拒绝这个AE RPC。
5. Leader中还维护了Follower的nextIndex，nextIndex的初始值为新当选的Leader的最后一条日志位置，在上例中就是13。而由于Leader收到两个Follower的False，则会减小对应的nextIndex，此时nextIndex=12，prevLogIndex=11，prevLogTerm=3，此时发送的AE消息包含了prevLogIndex后的所有条目（也就是12，13两个槽的内容）。
6. 此时S2发现prevLogIndex和prevLogTerm与自己记录相符，则接受此AE，并将后续的数据写入自己Log的相同位置（删除了Slot 12中的4，这是安全的因为4没有被commit）。
7. S1此时仍不符合，因此还是拒绝。
8. Leader重复前面的流程，直至S1也确认AE，并将正确Log写入。



### 快速恢复（Fast Backup）
如果Follower进度和Leader差太多（例如：差1000个任期），此时进行日志恢复就会造成巨大的时间开销（需要发送1000条AE）。

一种快速恢复策略如下：

+ 在新Leader发送第一个AE后，每个Follower回复都附带上一些额外信息：
    - XTerm：Follower中与Leader冲突的Log对应的任期号。在之前有介绍Leader会在prevLogTerm中带上本地Log记录中，前一条Log的任期号。如果Follower在对应位置的任期号不匹配，它会拒绝Leader的AppendEntries消息，并将自己的任期号放在XTerm中。如果Follower在对应位置没有Log，那么这里会返回 -1。
    - XIndex：这个是Follower中，对应任期号为XTerm的第一条Log条目的槽位号。
    - XLen：如果Follower在对应位置没有Log，那么XTerm会返回-1，XLen表示空白的Log槽位数。



下面假设三个场景  
![](https://cdn.jsdelivr.net/gh/Messiz/image-bed/test/20240917212648.png)

+ Case 1：Follower（S1）会返回XTerm=5，XIndex=2。Leader（S2）发现自己没有任期5的日志，它会将自己本地记录的，S1的nextIndex设置到XIndex，也就是S1中，任期5的第一条Log对应的槽位号。所以，如果Leader完全没有XTerm的任何Log，那么它应该回退到XIndex对应的位置（这样，Leader发出的下一条AppendEntries就可以一次覆盖S1中所有XTerm对应的Log）。
+ Case 2。Follower（S1）会返回XTerm=4，XIndex=1。Leader（S2）发现自己其实有任期4的日志，它会将自己本地记录的S1的nextIndex设置到本地在XTerm位置的Log条目后面，也就是槽位2。下一次Leader发出下一条AppendEntries时，就可以一次覆盖S1中槽位2和槽位3对应的Log。
+ Case 3。Follower（S1）会返回XTerm=-1，XLen=2。这表示S1中日志太短了，以至于在冲突的位置没有Log条目，Leader应该回退到Follower最后一条Log条目的下一条，也就是槽位2，并从这开始发送AppendEntries消息。槽位2可以从XLen中的数值计算得到。







### 选举约束（election Restriction）
1. 候选人最后一条Log条目的任期号**大于**本地最后一条Log条目的任期号；
2. 或者，候选人最后一条Log条目的任期号**等于**本地最后一条Log条目的任期号，且候选人的Log记录长度**大于等于**本地Log记录的长度



### 持久化（Persistence）
Raft持久化存储三种数据：

1. Log：Log记录APP的所有操作，Raft并没有规定持久化APP的运行状态，因此，Log是唯一保存APP状态的数据，当从故障恢复时，可以从Log逐条读取数据恢复APP状态；
2. currentTerm：与voteFor保证Leader选举的正确性，新的Leader需要知道正确的Term number，假设前一个Leader故障，而剩下的两个Follower缺失一些Log，导致不知道正确Term，那么重新选举会造成新Leader存储错误Term number；
3. voteFor：不存储voteFor会造成重复投票，造成脑裂。S1为S2投完票后故障，恢复后又收到S3的投票请求，而认为自己没有投过票，又给S3投票，可能造成两个Leader的情况发生。



### 日志快照（Log Snapshot）
Raft系统长时间运行会造成Log内容过多的情况，这对于持久化以及故障后恢复的效率会产生较大影响。因此，提出快照（Snapshots）。

当Raft判断Log大小达到某个阈值时，会要求程序在Log的特定位置上，对其状态做一个快照，然后Raft丢弃这一点之前的Log。

可能出现的问题：当Follower落后太多，以至于Leader产生快照丢弃的Log无法恢复时，Leader和Follower的一致性会遭到破坏。

Raft的解决方案如下：

引入一个新的消息类型，InstallSnapshot RPC

当Follower刚刚恢复，如果它的Log短于Leader通过 AppendEntries RPC发给它的内容，那么它首先会强制Leader回退自己的Log。在某个点，Leader将不能再回退，因为它已经到了自己Log的起点。这时，Leader会将自己的快照发给Follower，之后立即通过AppendEntries将后面的Log发给Follower。



### 线性一致（Linearizability）
线性一致或者说强一致，定义了一个分布式系统正确的处理逻辑，在客户端看来，就和与单点服务交互一样。

**定义：一个系统是线性一致的，****如果执行历史整体可以按照一个顺序排列，且排列顺序与客户端请求的实际时间相符合；并且，每一个读操作都看到的是最近一次写入的值。**
