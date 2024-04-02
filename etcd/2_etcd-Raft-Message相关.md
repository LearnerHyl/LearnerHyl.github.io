# Raft Message Structure

当客户端提交操作到raft算法中时，会涉及到日志复制，这包括修改数据库状态、集群成员变更操作等。raft算法本身也有领导选举、投票等操作。在etcd-raft算法库中，无论是哪一种类型的操作，最后一定是封装成一个Message结构体，这些消息可能会从raft node中发送出去，并被别的节点收到处理。

# raft Message的结构体定义

在`raftpb/raft.proto`文件中，定义了raft算法中传输消息的结构体。`message Message`类型，etcd-raft库采取了grpc协议进行节点通信，grpc协议有一个特点，允许自选字段使用，即定义在message中的字段不必全部赋值，只需要选取必要的字段使用即可，当然使用者要清楚每种消息类型都是用什么字段，以便在解析message消息时可以准确的获取到数据，首先从宏观上了解一下Message结构体中定义了哪些可能的字段：

| 成员       | 类型        | 作用                                                         |
| ---------- | :---------- | :----------------------------------------------------------- |
| type       | MessageType | 消息类型                                                     |
| to         | uint64      | 消息接收者的节点ID                                           |
| from       | uint64      | 消息发送者的节点ID                                           |
| term       | uint64      | 节点当前任期ID                                               |
| logTerm    | uint64      | logTerm通常用于向跟随者附加Raft日志。例如，(type=MsgApp，index=100，logTerm=5)表示领导者附加从index=101开始的条目，并且索引100处的条目的期限为5。<br />可以认为是Leader追加日志时的PrevLogTerm |
| index      | uint64      | 参考logTerm注释，可以认为是Leader追加日志时的PrevLogIdx。    |
| entries    | Entry       | 要追加的日志条目。                                           |
| commit     | uint64      | Leader的commitIndex。                                        |
| vote       | uint        | 表示在当前term中该节点投票给了谁。                           |
| snapshot   | Snapshot    | 对于MsgSnap消息，snapshot是非nil且非空的，对于所有其他消息类型，它是nil。<br />但是，运行旧二进制版本的对等节点可能会发送非nil的空值；  用于非MsgSnap消息的快照字段。代码应该准备好处理这样的消息。 |
| reject     | bool        | 表示follower拒绝领导者的追加条目操作                         |
| rejectHint | uint64      | 当follower拒绝leader追加操作时，会给一些提示信息以让leader快速的定位到发送日志的位置。 |
| context    | bytes       | 上下文数据。                                                 |
| Response   | []Message   | 该数据结构由raft节点填充，以指示存储线程在与message关联的工作完成时如何响应以及响应给谁。当有MsgStorageAppend 和 MsgStorageApply消息完成时，填充此结构体。 |

为了便于理解字段，源代码中给出了一些例子解释：

- `logTerm`通常用于向跟随者附加Raft日志。例如，（type=MsgApp，index=100，logTerm=5）表示领导者附加从index=101开始的条目，并且索引100处的条目的期限为5。即prevLogindex是100，prevLogTerm是5。

- （type=MsgAppResp，reject=true，index=100，logTerm=5）表示**跟随者拒绝其领导者的一些条目**，因为它在索引100处已经有一个term=5的条目。

  >   (type=MsgAppResp,reject=true,index=100,logTerm=5) means follower rejects some
  >
  >   entries from its leader as it already has an entry with term 5 at index 100.
  >
  > 这是原文，到底是什么意思，既然follower在index-100处已经有term为5的条目，这不就意味着已经匹配上了？为什么还要拒绝leader的条目？或许可以理解follower已经有了一部分leader发来的日志，只选择了部分entries内容进行复制，所以这里说的是**跟随者拒绝其领导者的一些条目**，而不是全部条目？

- （type=MsgStorageAppendResp，index=100，logTerm=5）表示本地节点将条目写入了稳定存储，索引100处的条目的任期为5。但这并不总是意味着相应的MsgStorageAppend消息是携带这些条目的消息（即prevLogIdx=100，prevLogTerm=5），只是在处理相应的MsgStorageAppend消息时，这些条目已经被持久化了。
- （type=MsgStorageAppend，vote=5，term=10）表示本地节点正在为任期10中的5号节点投票。对于MsgStorageAppends，如果任何字段发生了变化，那么term、vote和commit字段都将被set（以便构造HardState），如果没有任何字段发生变化，那么所有字段都将被unset。
- 对于MsgSnap消息，snapshot是非nil且非空的，对于所有其他消息类型，它是nil。但是，运行旧二进制版本的对等节点可能会发送非nil的空值。用于非MsgSnap消息的快照字段。代码应该准备好处理这样的消息。

# 不同MsgType消息的介绍

raft库使用Protocol Buffer格式发送和接收消息（在raftpb包中定义），使用gRPC协议通信。每个状态（follower、candidate、leader）在使用给定的raftpb.Message推进时实现自己的“step”方法（“stepFollower”、“stepCandidate”、“stepLeader”）。每个step的具体行为由其raftpb.MessageType确定。请注意，每种step方法都由一个公共方法“Step”检查，该方法对节点和传入消息的Term进行安全检查，以防止陈旧的日志条目。

在`raftpb/raft.pb.go`中介绍了多达24种消息类型，接下来分别介绍这些消息的各种类型、消息本身发挥的作用，以及这类消息会用到message中哪些字段。<u>注意，Type、To、From、Term四个字段一般来说会在所有类型消息中出现。有些消息类型在源码中构造时可能没有显示的放上去，但是这些字段一般是必须的，会在receiver端使用，写上没什么问题。</u>

注意，集群成员变更的相关消息类型不在这里介绍，这部分信息会在下一节中介绍集群成员变更机制的时候再具体介绍。

## MsgHup消息

`MsgHup`用于选举。如果节点是follower或candidate，则在“raft”结构中将“tick”函数设置为“tickElection”。如果follower或candidate在选举超时之前没有收到任何心跳，则将“MsgHup”传递给其Step方法，并成为（或保持）候选人以开始新的选举。

使用参数类型如下：

| 成员 | 类型   | 作用                                                 |
| ---- | :----- | :--------------------------------------------------- |
| type | MsgHup | 不用于节点间通信，仅用于发送给本节点让本节点进行选举 |
| from | uint64 | 本节点ID                                             |
| to   | uint64 | 消息接收者的节点ID                                   |

## MsgBeat消息

`MsgBeat`是一个内部类型，用于通知领导者发送“MsgHeartbeat”类型的心跳。如果节点是领导者，则在“raft”结构中将“tick”函数设置为“tickHeartbeat”，并触发领导者定期向followeres发送“MsgHeartbeat”消息。

| 成员 | 类型    | 作用                                                         |
| ---- | :------ | :----------------------------------------------------------- |
| type | MsgBeat | 不用于节点间通信，仅用于leader节点在heartbeat定时器到期时向集群中其他节点发送心跳消息 |
| from | uint64  | 本节点ID                                                     |
| to   | uint64  | 消息接收者的节点ID                                           |

## MsgProp消息

一般都是Leader节点收到消息后，将MsgProp发给自己。Leader可能已经过期了。

`MsgProp`建议将数据附加到其日志条目。这是一种特殊类型，用于将提案重定向到领导者。因此，send方法使用其HardState的Term覆盖raftpb.Message的Term，以避免将其本地Term附加到“MsgProp”。

- 当“MsgProp”传递给领导者的“Step”方法时，在通过一系列的检查之后，领导者调用“appendEntry”方法将条目附加到其日志中，然后调用“bcastAppend”方法将这些条目发送到他的peers。
- **当传递给候选人时，“MsgProp”被丢弃。**
- 当传递给follower时，“MsgProp”由send方法存储在该follower的邮箱（msgs）中。它与发送者的ID一起存储，稍后由rafthttp包转发给leader。首先会检查集群内是否有leader存在，如果当前没有leader存在说明还在选举过程中，这种情况忽略这类消息；否则转发给leader处理。

这里的意思是，MsgProp消息只能由Leader处理。follower会将该消息重定向到Leader，而candidate会直接将该消息丢弃。**为什么candidate丢弃消息却不转发给Leader**

![image-20240305165723447](./assets/image-20240305165723447.png)

问了一下群友，这里还有一些模糊，后面会深入思考一下。

| 成员    | 类型    | 作用                                            |
| ------- | :------ | :---------------------------------------------- |
| type    | MsgProp | raft库使用者提议（propose）数据，只能Leader处理 |
| from    | uint64  | 本节点ID                                        |
| to      | uint64  | 消息接收者的节点ID                              |
| entries | Entry   | 日志条目数组                                    |

具体代码逻辑可见raft.go中的step相关方法，Leader通过step将消息发给自己，前面还有一系列的检查操作，通过之后可以进行相应的处理。

## MsgApp类型

`MsgApp`消息是`MsgProp`消息成功后的结果：`MsgApp`包含要复制的日志条目。领导者调用bcastAppend，该方法调用sendAppend，该方法以“MsgApp”类型发送即将复制的日志。当“MsgApp”传递给candidate的Step方法时，候选人会恢复为follower，因为它表示有一个有效的领导者发送“MsgApp”消息。候选人和follower以“MsgAppResp”类型响应此消息。

| 成员    | 类型   | 作用                                     |
| ------- | :----- | :--------------------------------------- |
| type    | MsgApp | 用于leader向集群中其他节点同步数据的消息 |
| to      | uint64 | 消息接收者的节点ID                       |
| from    | uint64 | 本节点ID                                 |
| index   | uint64 | 代表prevLogIndex                         |
| LogTerm | uint64 | 代表prevLogTerm                          |
| Entries | Entry  | 要发送的日志                             |
| Commit  | uint64 | 当前Leader的commitindex                  |

## MsgAppResp

Leader将MsgApp类型消息发给followers，followers处理完消息后，返回给Leader类型为MsgAppResp类型的消息，因此这里有两个阶段需要说明，第一个是follower处理来自Leader的MsgApp消息的过程，第二个是Leader处理来自follower的MsgAppResp消息的过程。

### follower处理来自Leader的MsgApp

`MsgAppResp`是对日志复制请求（'MsgApp'）的响应。当“MsgApp”传递给candidate或follower的Step方法时，它通过调用“handleAppendEntries”方法响应，该方法将“MsgAppResp”消息发送到raft邮箱(msgs)。

| 成员       | 类型       | 作用                                                         |
| ---------- | :--------- | :----------------------------------------------------------- |
| type       | MsgAppResp | 集群中其他节点针对leader的MsgApp消息的应答消息               |
| from       | uint64     | 本节点ID                                                     |
| to         | uint64     | 消息接收者的节点ID                                           |
| index      | uint64     | 日志索引ID，不同情况下回复的值有不同含义，见下文。           |
| reject     | bool       | 是否拒绝同步日志的请求                                       |
| rejectHint | uint64     | 拒绝同步日志请求时返回的当前节点的日志索引，用于被拒绝方快速定位到下一次合适的同步日志位置 |
| LogTerm    | uint64     | follower日志中的rejectHint索引处日志的任期                   |

若想详细了解这里字段的含义，需要先理解peers是如何处理来自Leader的MsgApp消息的。先阅读了raft.go中的`handleAppendEntries(m pb.Message)`函数。

follower/candidate收到来自Leader的MsgApp消息后，candidate先将状态转为follower，接下来：

1. 首先从MsgApp中提取出要本次MsgApp操作携带的日志条目，prev配对信息，Leader任期。这 些用logSlice结构体进行封装。接下来做一些判断工作。

2. 如果`prevLogIndex < follower's commitedIndex`，则说明Leader发送的日志已经过期，当前节点不会接受Leader的append操作。回复Leader，此时index字段设置为follower当前的commitIndex值。

3. follower尝试将日志追加到自己的日志中，若追加成功，回复Leader，此时index字段设置为follower当前日志的lastIndex值（即追加日志的最后一条日志索引）。

   > 最好去看一下raftLog中的Term实现，查看一下在不同情况下是如何处理获取Term的。

4. 若追加失败，意味着prev信息匹配失败，此时我们需要向Leader返回一个Hint，一个关于最大匹配的(index, term)的猜测。通过搜索跟随者的日志，**找到最大的(index, term)对**，其中term <= MsgApp的LogTerm，index <= MsgApp的Index。这可以帮助跳过跟随者未提交的尾部中所有term大于MsgApp的LogTerm的索引。

   总结来说，返回的是当前follower节点日志中**小于等于**prevLogTerm的最大的index索引—即hintIndex；以及对应索引的Term—hintTerm。此时可以返回，Reject为True，RejectHint设置为hintIndex，LogTerm设置为hintTerm。将消息返回给Leader。

   此时Index字段设置为本次MsgApp消息的prevLogIndex值（m.Index值）。

follower将处理完后的形成的MsgAppResp消息放到Ready结构体的msgs结构体中，等待上层状态机将消息发送给Leader节点。

### Leader收到MsgAppResp的反应

阅读raft.go中的stepLeader函数，Leader收到该follower的MsgAppResp消息后，说明该follower是活跃的，因此保存节点状态的RecentActive成员置为true。接下来，再根据msg.Reject的返回值，即节点是否拒绝了这次数据同步，来区分两种情况进行处理。

#### msg.Reject为True的情况

<u>因为follower节点拒绝了这次数据同步，所以节点的状态可能存在一些异常，此时如果leader上保存的节点状态为ProgressStateReplicate，那么将切换到ProgressStateProbe状态（关于这几种状态，下面会谈到）。</u>

如果msg.Reject为true，说明follower节点拒绝了Leader前面发送的MsgApp消息，follower需要提供一些消息，以让Leader节点可以做快速日志索引。

RejectHint是建议用于附加的下一个基本条目（即我们尝试附加条目RejectHint+1），LogTerm是跟随者在索引RejectHint处的任期。旧版本的此库不会为rejections填充LogTerm，并且对于具有空日志的跟随者，它为零。显然RejectHint是下一次发送MsgApp消息是的PrevLogIndex。该机制一定程度上加速了follower的日志同步速度，如果没有RjectHint机制，leader只能在每次被拒绝数据同步后都递减1进行下一次数据同步，显然这样是低效的。

在正常情况下，领导者的日志比跟随者的日志长，跟随者的日志是领导者的前缀（即跟随者的日志没有发散的未提交后缀）。在这种情况下，第一个探测揭示了跟随者的日志结束的地方（RejectHint=跟随者的最后索引），随后的探测操作将会成功，即随后可以成功追加日志。

但是，当网络被分区或系统超载时，可能会出现大量发散的日志尾部。朴素的尝试，按降序逐个探测条目，这种方式花费时间的数量级将是发散尾部的长度和网络往返延迟的乘积，这很容易导致数小时的探测时间，甚至可能导致完全的中断。 因此，探测被优化如下所述：

1. 当LogTerm>0时，意味着follower回复的RejectHint索引是有效的，Leader会调用`findConflictByTerm(RejectHint,RejectTerm)`去确定下一次给该follower发送MsgApp时要使用的nextProbeIndex（nextIndex）。函数细节不再阐述，看源码会更清晰，总的来说，该函数会返回Leader日志中小于等于RejectTerm的最大的index索引，`index<=RejectHint`。将该index作为`nextProbeIndex`值。
2. 之后，调用`MaybeDecrTo(lastPrevLogindex, nextProbeIndex)`：如果当前follower的状态处于stateReplicate，那么可能当前reject请求可能是过期的，这里会做出对应的处理。若follower处于非replicating状态，消息可能过期，最后返回最终的nextProbeIndex，设置相关元数据。
3. 若上述函数执行成功，且当前follower的状态处于StateReplicate，则转变为StateProbe状态。之后向该follower发送消息。

在源码部分，举出了在快速日志回溯的场景下可能碰到的各种复杂情况，以上述描述的正常情况和网络分区或系统超载情况为基础，这些例子十分有助于我们了解系统可能碰到的各种混沌场景。

> 此外，Leader根据follower接收日志的情况，将progress中该follower的状态标记为StateProbe、StateReplicate、StateSnapshot三种，下面会详细说明这三种状态的含义。

#### msg.Reject为False的情况

这种情况说明该节点通过了leader的这一次数据同步请求，这种情况下根据msg.Index来判断在leader中保存的该节点日志数据索引是否发生了更新，如果发生了更新那么就说明这个节点通过了新的数据。

首先，当回复消息满足如下任一条件时，Leader可以对该follower进行状态转换：

- follower回复的Index成功的更新了Leader中相应的Progress数据（Matchindex和NextIndex）；
- follower回复的`LastIndex==progress.MatchIndex`，且当前的`progress.State\==StateProbe`（这意味着可以将当前follower的状态转化为StateReplicate，因为Leader已经知道了当前follower的LastIndex）。
  - follower的回复存储在msg.Index字段中，当follower认为leader发送的MsgApp消息过期时，m.index代表的是commitIndex；若follower接受日志并追加后，m.index代表的是LastIndex值，理论上总是lastIndex值。为什么呢，我们分情况分析。
  - 若follower成功接受了MsgApp，并马上提交，就会有lastIndex=commitIndex，此时lastIndex或者commitIndex都是一样的；
  - 若follower成功接受了MsgApp，但还未提交，那么此时commitIndex<lastindex，他会先追加日志，之后返回的值依然是LastIndex。
  - 若follower认为消息过期了，因为prev<commitIndex，此时意味着follower已经更新了commitIndex，即该MsgApp对应的消息已经被提交过了，发生在之前的情况中，此时返回的m.index值(commitIndex)会小于Leader保存的matchIndex，不可能进入下一步中，因为不满足m.Index == matchIndex。
  - 综上所述，当进入状态更新步骤时，返回的总是新鲜复制到follower的lastIndex。

当满足上述任意一个条件时，进入对应follower节点的progress状态转换流程：

1. 如果该节点之前在ProgressStateProbe状态，说明之前处于探测状态，现在Leader已经知道该节点的LastIndex值了，此时可以切换到ProgressStateReplicate，开始正常的接收leader的同步数据了。该状态代表Follower会积极地接受Leader发来的日志，而不存在发散的日志段。

2. 如果该节点之前处于ProgressStateSnapshot状态，即还在同步Leader当前的快照副本，说明该follower之前可能落后leader数据比较多才采用了接收快照副本的状态。这里还需要多做一点解释，因为在节点落后leader数据很多的情况下，可能leader会多次通过snapshot同步数据给节点，而当 pr.Match +1  >= r.raftLog.firstIndex()的时候，说明通过快照来同步数据的流程完成了，这时可以进入正常的接收同步数据状态了，即意味着Leader可以开始给这个follower同步日志信息了。首先主动变为Probe状态，同步快照信息到NextIndex和MatchIndex，之后主动变为Replicate状态，意味着消除了发散的日志条目，可以开始正常复制日志条目了。

   > 这里我们没有考虑PendingSnapshot，只考虑了稳定存储中的Snapshot，至于原因，请见Progress结构体中PendingSnapshot结构体的注释。

3. 如果之前处于ProgressStateReplicate状态，因为m.Index代表此时该follower的LastIndex的消息已经成功发送并收到了follower肯定的回复，在Inflights中该lastIndex代表的MsgApp消息以及在他之前发送的且还未收到回复的MsgApp消息都可以删除了，以释放空间，让Leader可以继续给该follower发送新的Append操作。

之后，调用`maybeCommit()`判断是否有新的日志被提交了，`maybeCommit`中同样考虑到了集群成员变更期间出现新旧两个配置的问题，采用的是JointConsensus算法，总的来说，会选择两个配置中提交索引的最小索引作为当前新的commitIndex值。

- 如果可以确定已经有新的日志可以被提交了， 即存在日志已经被成功复制到大多数节点上了，推进commitIndex值，之后可以判断是否可以处理一些可能被挂起的ReadIndex请求；然后马上向peers发送MsgApp消息，推进各个peer的commitIndex值。

若没有新日志被提交，且发现该raft peer之前因为某种原因被暂停发送MsgApp消息，从而无法收到新的日志，那么这个peer可能会缺少一些日志，leader会选择马上发送日志。

之后，经过上面流程，已经更新了和该raft peer相关的Inflights消息，可能已经释放了一些已经接收到肯定回复的消息，此时有空间给改peer发送新的消息，Leader会尝试给改peer发送消息，调用maybeSendAppend方法，看看是否有新的条目/日志需要发送，直到没有东西需要发送时，可以退出该尝试同步的流程。

> Inflights：Leader使用该结构体存储已经发送给某个Peer的、但是还未收到回复的MsgApp消息。注意只有当follower处于StateReplicate状态时，Leader才会给其发送日志。

最后，在此背景下，Leader还要做一个和Leader Transfer相关的操作： 如果该follower是正在被Leader Tranfer的对象，并且该follower的matchIndex已经和当前Leader的LastIndex相同，说明该follower已经完全跟上了Leader的进度，此时该Leader会调用`SendTimeoutNow()`发送一个超时消息给这个follower，提示他要马上开始选举成为新的领导人，之后该Leader会主动下台。            

## MsgVote/MsgPreVote消息

`MsgVote`请求选举投票。当节点是follower或candidate，并且将“MsgHup”传递给其Step方法时，然后节点调用“campaign”方法来竞选成为领导者。一旦调用了“campaign”方法，节点就会成为候选人，并向集群中的对等方发送“MsgVote”以请求投票。

- 当传递给领导者或候选人的Step方法并且消息的Term低于领导者或候选人的Term时，“MsgVote”将被拒绝（返回带有Reject true的“MsgVoteResp”）。如果领导者或候选人收到了更高term的“MsgVote”，它将恢复为follower。
- 当“MsgVote”传递给follower时，只有当发送者的最后一个term大于MsgVote的term或发送者的最后一个term等于MsgVote的term，但发送者的最后一个提交的索引大于或等于follower's时，它才为发送者投票。

| 成员    | 类型               | 作用                                                         |
| ------- | :----------------- | :----------------------------------------------------------- |
| type    | MsgVote/MsgPreVote | 节点投票给自己以进行新一轮的选举                             |
| to      | uint64             | 消息接收者的节点ID                                           |
| from    | uint64             | 本节点ID                                                     |
| term    | uint64             | 任期ID（如果是PreVote，则是r.Term+1，注意节点本身任期不改变，只是发出的这个PreVote的term=r.Term+1） |
| index   | uint64             | 日志索引ID，当前candidate的lastIndex                         |
| logTerm | uint64             | lastIndex处日志的Term                                        |
| context | bytes              | 上下文数据（Prevote中用于指明该candidate是否处于compaignTranfer阶段，从而帮助peer做出决断） |

处理流程与MsgPreVote类似，这块处理流程会在下面和MsgPreVote一起说明。

## MsgVoteResp

`MsgVoteResp`包含投票请求的响应。当`MsgVoteResp`传递给候选人时，候选人计算它赢得了多少票。如果它超过了多数（法定人数）。它将成为领导者并调用`bcastAppend`。如果候选人收到了否决的多数票，它将恢复为follower。

| 成员   | 类型                       | 作用               |
| ------ | :------------------------- | :----------------- |
| type   | MsgVoteResp/MsgPreVoteResp | 投票应答消息       |
| to     | uint64                     | 消息接收者的节点ID |
| from   | uint64                     | 本节点ID           |
| reject | bool                       | 是否拒绝           |

处理流程与MsgPreVoteResp类似，这块处理流程会在下面和MsgPreVoteResp一起说明。

## MsgPreVote和MsgPreVoteResp

 `MsgPreVote`和`MsgPreVoteResp`在可选的两阶段选举协议中使用。当Config.PreVote为true时，首先进行预选举（使用与常规选举相同的规则），除非预选举表明竞选节点将获胜，否则没有节点会增加其Term号。**当分区节点重新加入集群时，PreVote机制可以最小化中断。**

MsgPreVote和MsgPreVoteResp的消息字段与上面的MsgVote和MsgVoteResp相同。下面重点说一下PreVote阶段的一些具体流程。

当节点需要发起选举时，节点调用raft.campaign函数进行投票给自己进行一次新的选举，其中的参数CampaignType有以下几种类型：

1. `campaignPreElection`：campaignPreElection 表示Config.PreVote为true时正常选举的第一阶段，即PreVote阶段。
2. `campaignElection`：campaignElection 表示正常（基于时间的）选举（Config.PreVote为true时的第二阶段）
3. `campaignTransfer`：由于leader迁移发生的选举。如果是这种类型的选举，那么msg.Context字段保存的是“CampaignTransfer”字符串，这种情况下会强制进行leader的迁移。

MsgVote还需要带上几个与本节点日志相关的数据（Index、LogTerm），因为raft算法要求，一个节点要成为leader的一个必要条件之一就是这个节点上的日志数据是最新的。

### 为什么我们需要PreVote

前面提到，**当分区节点重新加入集群时，PreVote机制可以最小化集群中断问题。**考虑以下场景：

当出现网络分区的时候，A、B、C、D、E五个节点被划分成了两个网络分区，A、B、C组成的分区和D、E组成的分区，其中的D节点，如果在选举超时到来时，都没有收到来自leader节点A的消息（因为网络已经分区），那么D节点认为需要开始一次新的选举了。

正常的情况下，节点D应该把自己的任期号term递增1，然后发起一次新的选举。由于网络分区的存在，节点D肯定不会获得超过半数以上的的投票，因为A、B、C三个节点组成的分区不会收到它的消息，这会导致节点D不停的由于选举超时而开始一次新的选举，而每次选举又会递增任期号。

在网络分区还没恢复的情况下，这样做问题不大。但是当网络分区恢复时，由于节点D的任期号大于当前leader节点的任期号，这会导致集群进行一次新的选举，即使节点D肯定不会获得选举成功的情况下（因为节点D的日志落后当前集群太多，不能赢得选举成功）。这样会使集群陷入不稳定的状态，很显然这种没有必要的选举是可以避免的。

为了避免这种无意义的选举流程，节点可以有一种PreVote的状态，在这种状态下，想要参与选举的节点会首先连接集群的其他节点，只有在超过半数以上的节点连接成功时，才能真正发起一次新的选举。

所以，在PreVote状态下发起选举时，并不会导致节点本身的任期号递增1，而只有在进行正常选举时才会将任期号加1进行选举。

### MsgVote/MsgPreVote处理流程

当节点本身启动启动了PreVote机制之后，选举会首先从PreVote阶段开始，这里与普通的选举的唯一个差别是：

- Vote流程：是直接将当前节点的Term递增，之后用当前的信息发送投票。
- PreVote流程：发送PreVote消息时，节点本身的Term不会发生变化，但是发送出去的PreVote消息的`Msg.Term=r.term+1`，因此称之为预选举阶段。

处理投票选举相关的部分被单独封装到了`func (r *raft) Step(m pb.Message)`函数中，当raft Peer调用Step()处理一个消息时，首先会进行必要的任期检查，之后，当传入的msg.Type不是pb.MsgVote, pb.MsgPreVote, pb.MsgStorageAppendResp, pb.MsgStorageApplyResp时，才会调用r.step(r, m)进入常规处理流程，即上面我们写到的stepLeader()，stepCandidate()，stepFollower()这些。相关注释已经加到了raft.go文件的r.Step()消息中。这里简单的介绍一下流程：

第一步：首先该函数会判断msg.Term是否大于本节点的Term：

- 如果消息的任期号更大则说明是一次新的选举。这种情况下将根据msg.Context是否等于“CampaignTransfer”字符串来确定是不是一次由于leader迁移导致的强制选举过程。同时也会根据当前的electionElapsed是否小于electionTimeout来确定是否还在租约期以内。如果既不是强制leader选举又在租约期以内，那么节点将忽略该消息的处理，直接返回。

- 在论文4.2.3部分论述这样做的原因，是为了避免已经离开集群的节点在不知道自己已经不在集群内的情况下，仍然频繁的向集群内节点发起选举导致耗时在这种无效的选举流程中。

第二步：判断msg.Term是否小于本节点的Term，若小于，则说明消息是过期消息：

- 且在开启checkQuorum或PreVote机制的情况下收到了HeartBeat或MsgApp信息，那么直接回复MsgAppResp即可，这可能是来自过期Leader的消息；
- 若收到了过期的PreVote消息，同样要回复该发送方，否则会陷入死锁。
- 若收到了本地append storage线程处理完Append消息后回复的MsgStorageAppendResp消息，若是过期的index内容，给出提示；若是过期快照，照常应用即可。**即使快照在不同的term下应用，它的应用仍然有效。快照携带已提交的（与term无关的）状态。**
- 其余类型的消息直接无视。之后退出函数。

到这，意味着msg的Term等于节点当前Term；或者msg.Term大于当前节点Term，且是LeaderTransfer类型或者当前节点已经不在Lease内。可以按照正常流程处理消息。这里只说明和vote相关的：

- 若消息类型是MsgVote/MsgPreVote，且满足下面两个条件，才可以给该节点投票并回复肯定的消息。
  - 当前没有给任何节点进行过投票（r.Vote == None ），或者消息的任期号更大（m.Term > r.Term ），或者是之前已经投过票的节点（r.Vote == m.From)）。这个条件是检查是否可以还能给该节点投票。
  - 同时该节点的日志数据是最新的（r.raftLog.isUpToDate(m.Index, m.LogTerm) ）。这个条件是检查这个节点上的日志数据是否足够的新。 只有在满足以上两个条件的情况下，节点才投票给这个消息节点，将修改raft.Vote为消息发送者ID。如果不满足条件，将应答msg.Reject=true，拒绝该节点的投票消息。

若上面的都不是，则进入不需要m.From进度的`StepFollower/StepCandidate/StepLeader`等这些函数。

### MsgVoteResp/MsgPreVoteResp的处理流程

当candidate收到来自peer的回复消息之后，会调用stepCandidate()函数来处理，流程中规定了处于StatePreCandidate状态的节点只能处理PreVoteMsg类型的消息；处于PreCandidate状态的节点只能处理VoteMsg类型的消息。无论什么状态的candidate，当他收到来自当前Leader的消息时（日志、心跳、快照），会马上转变为follower状态。当收到VoteResp时：

节点调用raft.poll函数，其中传入msg.Reject参数表示当前收到的msg消息是否投票给自己，将这个消息代表的投票结果计入总数，并计算自己当前是否获得大多数选票、被拒绝、pending等等，即poll只会返回给当前candidate三种状态：VoteWon、VoteLost、VotePending。

注意在计算投票结果时，同样需要考虑新旧两个集群的态度，只有两个集群都同意其成为Leader之后，这次选举才算成功，有一方不同意就是Pending状态。candidate只会处理VoteWon和VoteLost两种状态，这两种分别意味着成功与失败两种情况。

VotePending意味着节点当前收集的投票结果还不足以证明成功或失败，可能还有没有收到的回复。

如果在收到某次选票后，得到了最终投票结果，分为VoteWon（成功）和VoteLost（失败）两种。

- 如果VoteWon，赢得选举，若当前选举类型是PreVote，则意味着该节点可以开始正式的选举了，马上进入compaignElection状态。若当前选举类型是Vote，直接成为Leader并广播消息。
- 如果是VoteLost，选举失败。无论选举类型是PreVote还是Vote，马上成为follower，注意任期是节点所处的当前任期（r.Term），因为PreVote状态发起的选举是r.Term+1的。

## MsgSnap

`MsgSnap`请求安装快照消息。当节点刚刚成为领导者或领导者收到`MsgProp`消息时，它调用`bcastAppend`方法，然后调用`sendAppend`方法到每个follower。**在`sendAppend`中，如果领导者未能获取到必要的Previous相关的term或条目，领导者将通过发送`MsgSnap`类型消息来请求快照。**

这就是Leader需要发送快照的唯一时机，只要Leader在给某个follower发送Append消息时，发现获取PrevLogEntry消息失败了，可能意味着Leader当前在PrevIndex处的日志已经被快照数据覆盖了，这部分日志已经被GC了。在raft.go文件中搜索`maybeSendSnapshot()`函数可以很容易看到发送快照的一些时机。

MsgSnap消息需要的参数如下：

| 成员     | 类型     | 作用                                     |
| -------- | :------- | :--------------------------------------- |
| type     | MsgSnap  | 用于leader向follower同步数据用的快照消息 |
| to       | uint64   | 消息接收者的节点ID                       |
| from     | uint64   | 本节点ID                                 |
| snapshot | Snapshot | 快照数据                                 |

可能会存在快照暂时不可用的情况，或者快照是empty的情况，详情见`maybeSendSnapshot()`。

### follower收到MsgSnap

follower收到MsgSnap消息后，会调用handleSnapshot函数处理。follower在收到快照之后，会调用restore()函数，该函数会先进行快照信息有效性的检测再决定是否做快照的应用操作：

1. 若快照的lastIndex<=commitIndex，说明快照已经过期，不接受该快照。可以类比follower会丢弃prevLogIndex<commitIndex的MsgApp消息。
1. 我们应该确保调用restore()方法的raft peer始终处于follower状态，即不能以leader状态应用快照，因此在方法中加入了检查来确保这个条件始终成立。此外，不在当前配置中的raft peer也不能应用快照，若发现当前peer不在配置中，则直接返回。
1. 通过以上所有的raft peer的合理性检查后，接下来可以检查该snapshot携带的lastEntry是否合理了，对比includeIndex和IncludeTerm是否与该peer日志中的相匹配，若匹配，说明当前peer已经拥有了这个快照携带的状态，不需要再次应用，只需要推进一下commitIndex即可，直接返回。
1. 否则说明该raft peer没有这个快照携带的信息，需要根据快照内容重置自己的相关状态：重置unstable部分状态、重置当前节点的配置信息。

之后为了方便查看应用完快照后的节点状态，可以打印一下当前的节点信息。

之后，若节点真的应用了这个快照，那么会打印相关信息，并发送MsgAppResp类型消息回复给Leader，Index值设置为自己当前的lastIndex值。

若节点由于某种原因忽略了这个快照，那么同样会回复MsgAppResp类型消息，Index值设置为自己当前的commitIndex。

## MsgHeartbeat

`MsgHeartbeat`从leader发送心跳。当`MsgHeartbeat`传递给候选人并且消息的term高于候选人的term时，候选人将恢复为follower，并从此心跳中更新其commitIndex。然后将消息发送到其邮箱(raft.msgs)。当`MsgHeartbeat`传递给follower的Step方法并且消息的term高于follower的term时，follower将其leaderID更新为消息中的ID。

| 成员    | 类型         | 作用                                     |
| :------ | :----------- | :--------------------------------------- |
| type    | MsgHeartbeat | 用于leader向follower发送心跳消息         |
| to      | uint64       | 消息接收者的节点ID                       |
| from    | uint64       | 本节点ID                                 |
| commit  | uint64       | 提交日志索引                             |
| context | bytes        | 上下文数据，在这里保存一致性读相关的数据 |

leader定时向集群中的成员发送心跳消息，除了探测节点的存活情况，还是为了进一步推进集群中节点的日志commit进度，注意commit值不能超过目标peer的当前matchIndex值。而context值是上下文数据，和线性一致性读相关，后续会解释。

无论是candidate还是follower，都会调用handleHeartbeat处理来自leader的`MsgHeartbeat`消息:

- 首先尝试推进自己的commitIndex值。
- 发送`MsgHeartbeatResp`回复给leader。

## MsgHeartbeatResp

`MsgHeartbeatResp`是对`MsgHeartbeat`的响应。当`MsgHeartbeatResp`传递给leader的Step方法时，leader知道哪个follower做出了响应。只有当leader的最后提交的索引大于follower的Match索引时，leader才运行`sendAppend`方法。具体见stepLeader中的处理。

| 成员    | 类型             | 作用                                     |
| ------- | :--------------- | :--------------------------------------- |
| type    | MsgHeartbeatResp | 用于follower向leader应答心跳消息         |
| to      | uint64           | 消息接收者的节点ID                       |
| from    | uint64           | 本节点ID                                 |
| context | bytes            | 上下文数据，在这里保存一致性读相关的数据 |

此外，Leader在收到`MsgHeartbeatResp`回复后，会尝试处理ReadIndex消息，这部分还看不大懂，等后面学习到ReadIndex相关内容时会回来学习。

# RawNode和对应的raft层相关消息

下面的消息一般是上层RawNode主动发送给自己下层的raft layer，用于节点协调内部通信。首先注意RawNode是一个线程不安全的Node。RawNode的方法对应于Node的方法，并在Node结构体中有更详细的描述。Node是rawNode的上层接口。

## MsgUnreachable（暂时没发现具体的代码调用场景）

由上层RawNode发送`MsgUnreachable`消息给raft层，说明远程节点不可达。  

> **那么谁调用RawNode提供的 ReportUnreachable(id uint64)接口呢？**

`MsgUnreachable`表示请求（消息）未被传递。当`MsgUnreachable`传递给leader的Step方法时，leader发现发送此`MsgUnreachable`的follower不可达，通常表示`MsgApp`丢失。 当follower的progress状态为replicate时，leader将其设置回probe。

| 成员 | 类型           | 作用                                       |
| ---- | :------------- | :----------------------------------------- |
| type | MsgUnreachable | 用于应用层向raft库汇报某个节点当前已不可达 |
| to   | uint64         | 消息接收者的节点ID（一般是自己）           |
| from | uint64         | 不可用的节点ID                             |

## MsgSnapStatus消息（暂时没发现具体的代码调用场景）

由上层RawNode发送`MsgSnapStatus`消息给raft层，说明远程节点不可达。 

> **那么谁调用RawNode提供的ReportSnapshot(id uint64, status SnapshotStatus)接口呢？**

`MsgSnapStatus`告诉快照安装消息的结果。当follower拒绝`MsgSnap`时，它表示 `MsgSnap`的快照请求由于网络问题而失败，这导致网络层无法将快照发送给其followers。然后leader将该follower的状态视作为StateProbe(此时应该没有实际set该follower的state为StateProbe)。当`MsgSnap`没有被拒绝时，它表示快照成功，leader将follower的状态设置为StateProbe，并恢复其日志复制。

| 成员   | 类型          | 作用                                           |
| ------ | :------------ | :--------------------------------------------- |
| type   | MsgSnapStatus | 用于应用层向raft库汇报某个节点当前接收快照状态 |
| to     | uint64        | 消息接收者的节点ID                             |
| from   | uint64        | 节点ID                                         |
| reject | bool          | 是否拒绝                                       |

## MsgStorageAppend

由上层RawNode发送给raft layer，以指示raft对应raft节点附加日志条目，一般是由一个新的Ready结构体驱动，写入更新的硬状态并应用快照。与`AsyncStorageWrites`一起使用。一般是`func newStorageAppendMsg(r *raft, rd Ready)`

| 成员      | 类型                | 作用                                                         |
| :-------- | :------------------ | :----------------------------------------------------------- |
| type      | pb.MsgStorageAppend | 指示raft层附加条目，更新hardstate，并应用快照                |
| from      | uint64              | 节点ID                                                       |
| to        | uint64              | 这里是固定值：LocalAppendThread，定义为maxUint,以代表是本地消息 |
| Entries   | []pb.Entry          | 本次追加消息需要附加的条目                                   |
| Responses | []pb.Message        | pb.MsgStorageAppendResp类型消息，当这条Append消息被处理完后，这个response会被回复给节点 |

`MsgStorageAppend`是节点发送给它自己本地的append storage线程的消息，用于将条目、硬状态和/或快照写入稳定存储。该消息将携带一个或多个响应，其中一个将是返回给自己的`MsgStorageAppendResp`。响应还可以包含`MsgAppResp`、`MsgVoteResp`和`MsgPreVoteResp`消息。此消息用于AsynchronousStorageWrites。

### pb.MsgStorageAppendResp

该消息在当前Ready（以及所有先前的Ready结构中）中的不稳定日志条目、硬状态和快照已保存到稳定存储之后，该消息应该被返回给节点，注意节点会抛弃不属于当前Term的回复。

| 成员 | 类型                | 作用                                                         |
| :--- | :------------------ | :----------------------------------------------------------- |
| type | pb.MsgStorageAppend | 指示raft层附加条目，更新hardstate，并应用快照                |
| from | uint64              | 这里是固定值：LocalAppendThread，定义为maxUint,以代表是本地消息 |
| to   | uint64              | r.ID                                                         |
| Term | uint64              | 节点当前的Term                                               |

注意，如果当前节点的unstable中有待持久化的日志，或者ready中有待应用的快照：

- unstable log情况下会启用Index和LogTerm参数，对应当前节点日志中的lastIndex和lastLogterm。
- 有待应用的快照的情况下会启用snapshot参数。

## MsgStorageApply

`MsgStorageApply`是节点发送给它自己本地的apply storage线程的消息，用于应用已提交的条目。该消息将携带一个响应，这将是返回给自己的`MsgStorageApplyResp`。用于`AsynchronousStorageWrites`。

| 成员      | 类型               | 作用                                                         |
| :-------- | :----------------- | :----------------------------------------------------------- |
| type      | pb.MsgStorageApply | 指示raft层附加条目，更新hardstate，并应用快照                |
| from      | uint64             | 节点ID                                                       |
| to        | uint64             | 这里是固定值：LocalApplyThread，定义为maxUint-1              |
| Term      | uint64             | 已提交的entries不会在特定的term下被应用，因此Term=0。        |
| Entries   | []pb.Entry         | 本次追加消息需要应用的条目                                   |
| Responses | []pb.Message       | pb.MsgStorageApplyResp类型消息，当这条Apply消息被处理完后，这个response会被回复给节点 |

### MsgStorageApplyResp

在当前Ready（以及所有先前的Ready结构中）中的已提交的条目已经应用到本地状态机之后，该消息应该被返回给节点。

| 成员    | 类型               | 作用                                                  |
| :------ | :----------------- | :---------------------------------------------------- |
| type    | pb.MsgStorageApply | 指示raft层附加条目，更新hardstate，并应用快照         |
| from    | uint64             | 这里是固定值：LocalApplyThread，定义为maxUint-1       |
| to      | uint64             | 节点ID                                                |
| Term    | uint64             | 已提交的entries不会在特定的term下被应用，因此Term=0。 |
| Entries | []pb.Entry         | 本轮消息已经应用的日志条目                            |

## MsgCheckQuorum消息

注意，虽然所有raft节点都是由上层`rawnode`的`rawnode.tick()`驱动的，但是Leader和followers/candidates的具体tick函数是不同的，Leader的tick()函数是`tickHeartbeat()`，而followers/candidates的tick()函数是`tickElection()`。

leader的定时器函数：`tickHeartbeat()`，在到达选举时间时，如果当前打开了raft.checkQuorum开关，那么leader将给自己发送一条MsgCheckQuorum消息，对该消息的处理是：检查集群中当前所有节点的状态，如果没有收到超过半数以上的节点的回复（包括当前Leader节点自身），那么该Leader将会自动下台成为follower。

| 成员 | 类型           | 作用                           |
| ---- | :------------- | :----------------------------- |
| type | MsgCheckQuorum | 用于leader检查集群可用性的消息 |
| to   | uint64         | 消息接收者的节点ID             |
| from | uint64         | 节点ID                         |

## MsgTransferLeader消息

raft论文中描述的Leader Transfer机制，在Etcd-Raft库中得以实现。上层Node提供了接口TransferLeader(transferee uint64)接口，供应用层主动调用。

如果不小心在follower上调用了此类消息，且follower知道集群当前的Leader是谁，那么follower会主动将该消息转发给leader处理，因为follower没有更改集群配置状态的权限。见StepFollower流程。

| 成员 | 类型              | 作用                                                         |
| ---- | :---------------- | :----------------------------------------------------------- |
| type | MsgTransferLeader | 用于迁移leader                                               |
| to   | uint64            | 消息接收者的节点ID                                           |
| from | uint64            | 注意这里不是发送者的ID了，而是准备迁移过去成为新leader的节点ID |

当Leader收到这类消息后，会进行如下的处理流程，代码见StepLeader()，首先是必要的检查工作：

1. 若当前Leader节点的角色状态是一个Learner，即还未开始提供服务，那么不能处理TransferLeader信息。
2. 如果当前Leader节点的raft.leadTransferee成员不为空，说明有正在进行的leader迁移流程，若正在迁移的目标ID与这条消息携带的目标ID相同，说明相同的领导权迁移行为已经在进行中了，忽略这条消息。若正在迁移的目标ID与这条消息携带的目标ID不同，则会终止之前的迁移流程，开始处理新的领导权迁移任务。
3. 如果该消息携带的迁移目标ID就是当前Leader节点的ID，说明迁移已经成功了，停止处理消息。因为节点不能把领导权转让给自己。

到此检查工作已经完毕，正式开始处理这个迁移请求：

1. Leader Transfer操作应该在一个electionTimeout内完成，所以首先重置r.electionElapsed。并设置当前r.leadTransferee = leadTransferee，说明自己正在处理一个领导权转移的任务。
2. 若当前目标节点的MatchIndex等于当前节点的lastIndex，说明日志已经同步到最新，直接发送MsgTimeoutNow消息，通知目标节点立即发起选举。否则，发送MsgApp消息，让目标节点尽快同步日志。

此外，当旧leader处于转让leader状态时，将停止接收新的prop消息，这样就避免出现在转让过程中新旧leader一直日志不能同步的情况。

## MsgTimeoutNow消息

`MsgTimeoutNow`类型消息用于LeaderTranfer流程。当Leader节点需要执行领导权禅让时，如果可以确定目标peer已经完全复制了自己当前的日志，就会发送`MsgTimeoutNow`消息给目标peer，目标peer收到消息后会马上开始一次选举，此时会调用campaign函数，传入的参数是campaignTransfer，表示这是一次由于迁移leader导致的选举流程。

| 成员 | 类型          | 作用                                                         |
| ---- | :------------ | :----------------------------------------------------------- |
| type | MsgTimeoutNow | leader迁移时，当新旧leader的日志数据同步后，旧leader向新leader发送该消息通知可以进行迁移了 |
| to   | uint64        | 新的leader ID                                                |
| from | uint64        | 旧的leader的节点ID                                           |

## MsgReadIndex和MsgReadIndexResp消息

这两个消息一一对应，使用的成员也一样，在后面分析读一致性的时候再详细解释。

| 成员    | 类型         | 作用               |
| ------- | :----------- | :----------------- |
| type    | MsgReadIndex | 用于读一致性的消息 |
| to      | uint64       | 接收者节点ID       |
| from    | uint64       | 发送者节点ID       |
| entries | Entry        | 日志条目数组       |

其中，entries数组只会有一条数据，带上的是应用层此次请求的标识数据，在follower收到MsgReadIndex消息进行应答时，同样需要把这个数据原样带回返回给leader，详细的线性读一致性的实现在后面展开分析。

详情请见第三篇文章。

# 节点状态概要

每个raft的节点，分为以下三种状态：

1. candidate：候选人状态，节点切换到这个状态时，意味着将进行一次新的选举。
2. follower：跟随者状态，节点切换到这个状态时，意味着选举结束。
3. leader：领导者状态，所有数据提交都必须先提交到leader上。

每一个状态都有其对应的状态机，每次收到一条提交的数据时，都会根据其不同的状态将消息输入到不同状态的状态机中。同时，在进行tick操作时，每种状态对应的处理函数也是不一样的。

所以raft结构体中将不同的状态，及其不同的处理函数独立出来几个成员变量：

| 成员     | 作用                                         |
| -------- | :------------------------------------------- |
| state    | 保存当前节点状态                             |
| tick函数 | tick函数，每个状态对应的tick函数不同         |
| step函数 | 状态机函数，同样每个状态对应的状态机也不相同 |

raft库中提供几个成员函数becomeCandidate、becomeFollower、becomeLeader分别进入这几种状态的，这些函数中做的事情，概况起来就是：

1. 切换raft.state成员到对应状态。
2. 切换raft.tick函数到对应状态的处理函数。
3. 切换raft.step函数到对应状态的状态机。

# 选举流程

## 发起选举的节点

只有在candidate或者follower状态下的节点，才有可能发起一个选举流程，而这两种状态的节点，其对应的tick函数都是raft.tickElection函数，这个函数的主要流程是：

1. 将选举超时递增1。
2. 当选举超时到期，同时该节点又在集群中时，说明此时可以进行一轮新的选举。此时会向本节点发送HUP消息，这个消息最终会走到状态机函数raft.Step中进行处理。

明白了raft.tickElection函数的作用，可以来看选举流程了：

- 节点启动时都以follower状态启动，同时随机选择自己的选举超时时间。之所以每个节点随机选择自己的超时时间，是为了避免同时有两个节点同时进行选举，这种情况下会出现没有任何一个节点赢得半数以上的投票从而这一轮选举失败，继续再进行下一轮选举
- 在follower的tick函数tickElection函数中，当选举超时到时，节点向自己发送HUP消息。
- 在状态机函数raft.Step函数中，在收到HUP消息之后，节点首先判断当前有没有没有apply的配置变更消息，如果有就忽略该消息。其原因在于，当有配置更新的情况下不能进行选举操作，即要保证每一次集群成员变化时只能同时变化一个，不能同时有多个集群成员的状态发生变化。
- 否则进入campaign函数中进行选举：首先将任期号+1，然后广播给其他节点选举消息，带上的其它字段包括：节点当前的最后一条日志索引（Index字段），最后一条日志对应的任期号（LogTerm字段），选举任期号（Term字段，即前面已经进行+1之后的任期号），Context字段（目的是为了告知这一次是否是leader转让类需要强制进行选举的消息）。
- 如果在一个选举超时之内，该发起新的选举流程的节点，得到了超过半数的节点投票，那么状态就切换到leader状态，成为leader的同时，leader将发送一条dummy的append消息，目的是为了提交该节点上在此任期之前的值（见疑问部分如何提交之前任期的值）

## 收到选举消息的节点

当收到任期号大于当前节点任期号的消息，同时该消息类型如果是选举类的消息（类型为prevote或者vote）时，会做以下判断：

1. 首先会判断一下该消息是否为强制要求进行选举的类型（context为campaignTransfer，context为这种类型时表示在进行leader转让，流程见下面的leader转让流程）
2. 判断当前是否在租约期以内，判断的条件包括：checkQuorum为true，当前节点保存的leader不为空，没有到选举超时，前面这三个条件同时满足。

如果不是强制要求选举，同时又在租约期以内，那么就忽略该选举消息返回不进行处理，这么做是为了避免出现那些离开集群的节点，频繁发起新的选举请求（见论文4.2.3）。

- 如果不是前面的忽略选举消息的情况，那么除非是prevote类的选举消息，在收到其他消息的情况下，该节点都切换为follower状态。
- 此时需要针对投票类型中带来的其他字段进行处理了，需要同时满足以下两个条件：
  1. 只有在没有给其他节点进行过投票，或者消息的term任期号大于当前节点的任期号，或者之前的投票给的就是这个发出消息的节点
  2. 进行选举的节点，它的日志是更新的，条件为：logterm比本节点最新日志的任期号大，在两者相同的情况下，消息的index大于等于当前节点最新日志的index，即总要保证该选举节点的日志比自己的大。

只有在同时满足以上两个条件的情况下，才能同意该节点的选举，否则都会被拒绝。这么做的原因是：保证最后能胜出来当新的leader的节点，它上面的日志都是最新的。
