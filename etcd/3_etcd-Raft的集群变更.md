# 1. 集群成员变更概要

《CONSENSUS: BRIDGING THEORY AND PRACTICE》的*Chapter 4 Cluster membership changes*介绍了两种成员变更算法，一种是一次操作一个节点的简单算法，另一种是联合共识（joint consensus）算法。两种算法都是为了避免由于节点切换配置时间不同导致的同一term出现不只一个leader的问题。

- 简单成员变更算法限制每次只能增加或移除一个节点。这样可以保证新配置与旧配置的quorum至少有一个相同的节点，因为一个节点在同一term仅能给一个节点投票，所以这能避免多主问题。
- 联合共识算法可以一次变更多个成员，但是需要在进入新配置前先进入一个“联合配置（joint configuration）”，在联合配置的quorum分别需要新配置和旧配置的majority（大多数）节点，以避免多主问题。当联合配置成功提交后，集群可以开始进入新配置。

etcd/raft的`ConfChangeV2`既支持简单的“one at a time”的成员变更算法，也支持完整的联合共识算法。需要注意的是，etcd/raft中的配置的应用时间与论文中的不同。**在论文中，节点会在追加配置变更日志时应用相应的配置，而在etcd/raft的实现中，当节点应用（apply）配置变更日志条目时才会应用相应的配置。**

etcd/raft在“apply-time”应用新配置的方式，可以保证配置在应用前已被提交，因此不需要论文中提到的回滚旧配置的操作。<u>但是这种方式需要引入新机制来解决影响集群liveness的边际情况下的问题。该问题已经被解决，会留出一节专门讨论这个问题。</u>

**需要注意的是，同一时间只能有一个正在进行的配置变更操作，在提议配置变更请求时，如果已经在进行配置变更，那么该提议会被丢弃（被改写成一条无任何意义的日志条目）。**

# 2.Etcd-raft配置的实现

etcd/raft实现的配置是按照**joint configuration**组织的，以自底向上的方法介绍etcd/raft中配置的实现。

## 2.1 MajorityConfig

在Joint Consensus算法中，处于中间状态的C~old,new~ 的quorum集群同时需要Cold和Cnew各自的Majority。Cold或Cnew配置中voter的集合（voter即有投票权的角色，包括candidate、follower、leader，而不包括learner），是通过`MajorityConfig`存储的。**该结构体负责存储某个配置下的具有投票权的节点的集合，即Cnew配置和Cold配置各用一个MajorityConfig结构体存储。**

> 下面会说JointConfig，JointConfig包含了两个MajorityConfig，一个代表Cold，一个代表Cnew。

```go
// MajorityConfig is a set of IDs that uses majority quorums to make decisions.
// MajorityConfig是一个ID集合，使用多数派来做决策。
type MajorityConfig map[uint64]struct{}
```

`MajorityConfig`的实现非常简单，其只是voter节点id的集合，但`MajorityConfig`提供了一些很实用的与majority有关方法，如下表所示（仅给出主要方法）：

|                      方法                      | 描述                                                         |
| :--------------------------------------------: | :----------------------------------------------------------- |
|     `CommittedIndex(l AckedIndexer) Index`     | 根据给定的`AckedIndexer`计算被大多数节点接受了的*commit index* 。 |
| `VoteResult(votes map[uint64]bool) VoteResult` | 根据给定的投票统计计算投票结果。                             |

`CommittedIndex`是根据该`MajorityConfig`计算被大多数接受的*commit index*，其参数`AckedIndexer`是一个接口：

```go
// quorum.go

// AckedIndexer allows looking up a commit index for a given ID of a voter
// from a corresponding MajorityConfig.
type AckedIndexer interface {
	AckedIndex(voterID uint64) (idx Index, found bool)
}

// tracker.go

type matchAckIndexer map[uint64]*Progress

// AckedIndex implements IndexLookuper.
func (l matchAckIndexer) AckedIndex(id uint64) (quorum.Index, bool) {
	pr, ok := l[id]
	if !ok {
		return 0, false
	}
	return quorum.Index(pr.Match), true
}
```

`AckedIndexer`接口中只定义了一个方法`AckedIndex`，该方法用来返回给定id的voter的一种索引的值。通过实现该接口与方法，在调用`CommittedIndex`时，可以根据不同的index来计算被大多数接受的*commit index*。上面的源码中给出了*tracker.go*中的一种`AckedIndexer`实现——`matchAckIndexer`，其实现的`AckedIndex`方法返回了voter的*match index*。etcd/raft在计算*commit index*时，就是根据节点的*match index*来计算的。

`CommittedIndex`的实现也很简单，其通过排序来计算第n/2+1小的索引，即为被大多数节点接受的最小索引。该方法中还有针对小切片的分配优化，感兴趣的读者可以自行阅读源码，这里不再赘述。

`VoteResult`方法的实现也很简单，其根据参数中的投票情况与该`MajorityConfig`中的voter，计算投票结果。投票结果有三种：`VoteWon`表示赢得投票、`VoteLost`表示输掉投票、`VotePending`表示投票还未完成（既没被大多数接受，也没被大多数拒绝），需要继续等待投票。

## 2.2 JointConfig

`JointConfig`表示Joint Consensus下的配置，存储了C~old~和C~new~两个配置，因此他表现为一个拥有两个MajorityConfig元素的数组：

```go
// JointConfig is a configuration of two groups of (possibly overlapping)
// majority configurations. Decisions require the support of both majorities.
//
// JointConfig是两个Group（可能重叠）多数配置的配置。决策需要两个多数的支持。
// 为了支持集群成员变更，Raft引入了JointConsensus机制，具体细节看论文。
type JointConfig [2]MajorityConfig
```

`JointConfig`的索引0处表示C~new~；元素1在joint consensus下表示C~old~，在非joint consensus下应为一个空配置。

`JointConfig`提供的方法与`MajorityConfig`几乎一样，其也提供了`CommittedIndex`方法与`VoteResult`方法。

`JointConfig`的`CommittedIndex`方法会获取两个`MajorityConfig`的`CommittedIndex`的返回值（若`MajorityConfig`为空配置，其返回类型最大值），并返回较小的结果，即被C~old~和C~new~都接受的commit index。

`JointConfig`的`VoteResult`方法也会获取两个`MajorityConfig`的`VoteResult`，如果二者结果相同可以直接返回，否则如果其中之一为`VoteLost`则直接返回`VoteLost`，否则还无法给出结果，返回`VotePending`。

## 2.3 Config(tracker.go)

`Config`记录了全部的成员配置，既包括voter也包括learner。具体解释请见tracker.go中的Config结构体。这里重点说明一下Config中的Learners和NextLearners的设计。

`Config`的`Voter`字段类型即为上文介绍的`JointConfig`，`AutoLeave`字段是标识*joint consensus*情况下离开*joint configuration*的方式，后文介绍配置变更的实现时会对其进行介绍。`Config`中与learner相关的字段有两个，分别是`Learners`和`LearnsNext`。`Learners`是当前的learner集合，`LearnersNext`是在*joint consensus*情况下，在离开*joint configuration*时需要新加入learner集合的节点。

etcd/raft为了简化配置变更的实现，其配置需要满足一条约束：配置的voter集合与learner集合不能有交集。在不做任何更改的*joint consensus*方法下，可能出现如下情况：

1. Cold：`voters: {1 2 3}, learners: {}`，
2. Cnew,old：`voters: {1 2} & {1 2 3}, learners: {3}`
3. Cnew：`voters: {1 2}, learners: {3}`。

这违背了上文中的约束，因为Etcd-raft中规定了voter和learner集合不能有交集，而这种情况下，在JointConsensus状态下（2中），3既是Cold的Voter，也是learner。

为此，`Config`中引入了`LearnsNext`字段，在*joint configuration*中，如果新的learn集合与voter集合有冲突，那么先将该learn加入到`LearnerNext`中，在退出*joint consensus*再将该`LearnersNext`中的节点加入到`Learners`中。例如：

1. Cold：`voters: {1 2 3}, learns: {}, next_learns: {}`
2. Cnew,old：`voters: {1 2} & {1 2 3}, learns: {}, next_learners: {3}`
3. Cnew：`voters: {1 2}, learns: {3}, next_learns: {}`

## 2.4 ProgressTracker（tracker.go）

ProgressTracker 跟踪当前活动的配置以及关于其中的节点和学习者的信息。特别是，它跟踪每个raft peer的match index，这反过来又允许推断出已提交的index。

`ProgressTracker`提供了如下方法：

|                             方法                             | 描述                                                         |
| :----------------------------------------------------------: | :----------------------------------------------------------- |
| `func MakeProgressTracker(maxInflight int) ProgressTracker`  | 新建空配置。                                                 |
|                  `ConfState() pb.ConfState`                  | 返回当前激活的配置。                                         |
|                     `IsSingleton() bool`                     | 判断当前配置是否为单节点模式（voter数为0，且不处于*joint configuration*中）。 |
|                     `Committed() uint64`                     | 返回被quorum接受的*commit index*。                           |
|           `Visit(f func(id uint64, pr *Progress))`           | 有序遍历追踪的所有进度，并对其执行传入的函数闭包。           |
|                    `QuorumActive() bool`                     | 判断是否有达到quorum数量的节点处于活跃状态（用于**Check Quorum**）。 |
|                   `VoterNodes() []uint64`                    | 返回有序的voter节点id集合。                                  |
|                  `LearnerNodes() []uint64`                   | 返回有序的learner节点id集合。                                |
|                        `ResetVotes()`                        | 重置记录的选票。                                             |
|               `RecordVote(id uint64, v bool)`                | 记录来自节点`id`的选票。                                     |
| `TallyVotes() (granted int, rejected int, _ quorum.VoteResult)` | 返回选票中赞同数、反对数、与当前投票结果。                   |

# 3. etcd-raft配置变更的实现

**etcd/raft中新配置是在代表配置变更的日志被应用时生效的**，也就是说，如果etcd/raft模块的使用者在处理`Ready`结构体的`CommittedEntries`字段时遇到了实现了`ConfChangeI`接口的消息时（包括`ConfChangeV2`和兼容的旧版本`ConfChange`），需要主动调用`Node`的`ApplyConfChange`方法通知Raft状态机使用新配置。该方法最终会调用`raft`结构体的`applyConfChange`方法，切换到相应的配置。

## 3.1 ConfChange类型

在介绍配置变更的实现之前，本节先介绍一下etcd/raft支持的配置变更类型。由于旧版本的配置变更消息`ConfChange`仅支持"one at a time"的简单算法，这里不再赘述；这里主要对新版本的`ConfChangeV2`类型的成员变更方法进行介绍。

在`ConfChangeV2`中加入了对Joint Consensus算法的支持，`ConfChangeV2`消息启动配置更改，它支持简单的“一次一个”成员更改协议和完整的联合共识，允许对成员资格进行任意更改。若只进行单个节点的更改，那么无论何时都可以使用简单协议。

非简单更改需要使用Joint Consensus，其中会运行两个配置更改。第一个配置更改指定所需更改，并将Raft组转换为联合配置，其中quorum行为需要先前更改和后续更改配置的大多数（即同时需要Cold和Cnew中具有投票权节点的大多数的同意意见）。联合共识避免进入可能危及生存能力的脆弱中间配置。

> 例如，在跨三个可用区并具有三个复制因子的情况下，如果不使用Joint Consensus算法，那么就不可能在不进入一个脆弱的中间配置的情况下替换一个投票者，而这个中间配置无法在一个可用区的故障中生存。

提供的ConfChangeTransition字段指定如何（以及是否）使用联合共识，并将离开联合配置的任务分配给Raft或应用程序。通过propose一个ConfChangeV2消息，并且可以选择性的填充Context字段来离开联合配置。

在raft.pb.go中定义了ConfChangeV2的结构体，具有如下字段：

| 字段名称   | 类型                 | 作用                                                         |
| ---------- | -------------------- | ------------------------------------------------------------ |
| Transition | ConfChangeTransition | Transition表示使用JointConsensus的方式                       |
| Changes    | []ConfChangeSingle   | ConfChangeSingle是一个单独的配置更改操作（针对一个节点进行的配置更改）。可以通过ConfChangeV2原子地执行多个此类操作。 |
| Context    | []byte               | 提供的Context字段被视为不透明的有效载荷，可以用于将状态机上的操作附加到配置更改提议的应用程序。 |

### 3.1.1 Transition（ConfChangeTransition类型）

`ConfChangeTransition`指定了与联合共识相关的配置更改的行为。`ConfChangeV2`的`Transition`字段表示切换配置的行为，其支持的行为有3种：

1. `ConfChangeTransitionAuto`（默认行为）：当配置变更可以通过简单算法完成时，直接使用简单算法；否则使用`ConfChangeTransitionJointImplicit`行为。
2. `ConfChangeTransitionJointImplicit`：强制使用*joint consensus*，并在适当时间通过`Changes`为空的`ConfChangeV2`消息自动退出*joint consensus*。该方法适用于希望减少*joint consensus*时间且不需要在状态机中保存joint configuration的程序。（注意退出JointConfiguration的操作是由状态机本身自动完成的）
3. `ConfChangeTransitionJointExplicit`：强制使用*joint consensus*，但不会自动退出*joint consensus*，而是需要etcd/raft模块的使用者通过提交`Changes`为空的`ConfChangeV2`消息退出*joint consensus*。该方法适用于希望显式控制配置变更的程序，如自定义了`Context`字段内容的程序。

### 3.1.2 Changes（[]ConfChangeSingle类型）

`ConfChangeSingle`类型的对象代表了对一个节点进行的配置变更操作。

```go
// ConfChangeSingle是一个单独的配置更改操作。可以通过ConfChangeV2原子地执行多个此类操作。
type ConfChangeSingle struct {
	Type   ConfChangeType  // 本次操作的成员变更类型
	NodeID uint64 // 本次变更的目标节点ID
}
```

ConfChangeType支持四种类型的操作：

1. `ConfChangeAddNode`：添加新节点。
2. `ConfChangeRemoveNode`：移除节点。
3. `ConfChangeUpdateNode`：用于状态机更新节点url等操作，etcd/raft模块本身不会对节点进行任何操作。
4. `ConfChangeAddLearnerNode`：添加learner节点。

## 3.2 propose一个ConfChange消息

因为只能由Leader处理配置变更消息：在`stepLeader`方法处理`MsgProp`时，如果发现`ConfChange`消息或`ConfChangeV2`消息，会反序列化消息数据并对其进行一些预处理，具体请见pb.MsgProp消息这块源代码，这里简要的对处理流程进行说明：

1. 若判断到时ConfChange类型的Entry，那么将Entry中的Data数据进行反序列化操作（Unmarshal），将其变为pb.ConfChange对象，无论是该Entry是pb.EntryConfChange还是pb.EntryConfChangeV2的类型，最后都将反序列化的对象存储到统一的抽象对象类型pb.ConfChangeI中（具体可见confchange.go中的内容）。
2. 之后根据当前raft节点的状态，给出下述bool的值是否为true：
   - `alreadyPending`：上一次合法的`ConfChange`还没被应用时为真。
   - `alreadyJoint`：当前配置正处于*joint configuration*时为真。
   - `wantsLeaveJoint`：如果消息（旧格式的消息会转为`V2`处理）的`Changes`字段为空时，说明该消息为用于退出*joint configuration*并转到Cnew的消息。
3. 根据上述三个变量，拒绝当前ConfChange消息的情况有3种：
   - `alreadyPending`：Raft同一时间只能有一个未被提交的`ConfChange`，因此拒绝新提议。
   - `alreadyJoint`为真但`wantsLeaveJoint`为假：处于*joint configuration*的集群必须先退出*joint configuration*并转为Cnew，才能开始新的`ConfChange`，因此拒绝提议。
   - `alreadyJoint`为假单`wantsLeaveJoint`为真，未处于*joint configuration*，忽略不做任何变化空`ConfChange`消息。

对需要拒绝的提议的处理非常简单，只需要将该日志条目替换为没有任何意义的普通空日志条目`pb.Entry{Type: pb.EntryNormal}`即可。

而对于合法的`ConfChange`，除了将其追加到日志中外，还需要修改`raft`结构体的`pendingConfIndex`字段，将其设置为当前配置更改消息在日志中应该所处的索引。

## 3.3 应用ConfChange

在ETCD-Raft中，集群成员变更消息是以日志的形式出现的，只有当该日志被提交之后，节点再主动调用Node.ApplyConfChange方法来应用新配置，最终会调用raft.applyConfChange方法。至此，新的配置才会开始生效。

在第一篇的结构说明中，我们已经了解Node负责中转信息，是用于连接应用层与raft状态机的中间媒介；Node还负责接收来自Peer和客户端的消息。当Node收到一条ConfChange类型消息时，会调用raft.applyConfChange()来应用这条pb.ConfChange类型的消息，大致流程如下：

1. 首先创建changer对象保存节点在当前活跃配置下的ProgressTracker对象和LastIndex信息。
2. 之后进行必要的检查：如信息是否指示节点离开JointConfig状态，使用的是Joint Consensus算法还是one at a time类型算法（取决于消息的changes长度是否大于1）。之后调用changer中对应的处理函数，获取新的配置。
   - Joint Consensus算法调用EnterJoint()，这会进入一个JointConfig状态，即论文中描述的Cold_new状态。算法会复制一份Cold配置信息，并在copy的基础上不断应用changes中的修改，应用完毕后，检查当前新配置中的信息是否符合相关规则，若不符合就返回err不为nil的结果。
   - one at a time调用Simple()。整理流程类似于JointConsensus，但是只允许最多一个Voter发生变化，若不符合要求则返回错误。
3. 至此，我们得到了一个JointConfig/或者得到了一个新的Config，以及一个新的tracker信息。最后调用raft.switchToConfig切换到新配置。

### 3.3.1 raft.switchToConfig

具体细节注释还是看源码，都写在源码上了，下面进行简要说明：

该方法会将新的`Config`与`ProgressMap`应用到`ProgressTracker`。应用后，获取激活的新状态，并检查该节点是否在新的`ProgressMap`。如果节点不造`ProgressMap`中，说明节点已被移除集群。根据不同的新状态，执行如下操作：

1. 如果该节点原来为leader，但在新配置中成为了learner或被移除了集群，那么直接返回，该节点会在下个term退位且无法再发起投票请求。此时状态机可以停止`Node`。
2. 如果节点原来不是leader节点，或新配置中不包含任何voter，那么该节点同样不需要进行任何操作，直接返回。
3. 否则，该节点原来为leader且继续运行。需要执行一些操作。

如果该节点原来为leader且继续运行，那么按需广播日志复制请求与*commit index*。最后，该leader检查此时是否在进行**leader transfer**，如果正在进行**leader transfer**但目标节点已被从配置中移除，那么终止**leader transfer**。

## 3.4 Auto Leave

当`ConfChange`的类型为`ConfChangeTransitionAuto`或`ConfChangeTransitionJointImplicit`时，退出*joint configuration*由etcd/raft自动实现：

1. 配置是在用户处理`Ready`结构体时主动调用`ConfChange`方法时生效的，当node处理完代表日志变更的entry之后，意味着此次配置变更操作已经完成了；
2. 当这个Ready结构体已经完成时，node会调用rawNode.acceptReady()来提醒raft状态机这批操作日志已经被应用完成或者追加完成。接下来rawNode会向raft节点发出StorageApplyRespMsg或StorageAppendRespMsg消息。
3. 当raft收到StorageApplyRespMsg消息时，并可以确定确实有新的日志条目被应用了，则会调用AppliedTo推进自身的AppliedIndex，在AppliedTo中内置了AutoLeave的逻辑。
4. 当节点发现自身的应用进度已经超过pendingConfIndex时，且设置了AutoLeave选项，并且自己是Leader，那么主动调用r.Step()向自己propose一条数据为nil的confChangeV2消息。
5. 当应用层在未来的某个ready中读到这条消息后，会调用node.ApplyConfChange，之后就是node层调用raft.applyConfChange()来处理，可以识别到这是一条让自己离开JointConfig的消息，离开完毕。

## 3.5 Learner角色

### 公开“学习者”节点类型给“MemberAdd” API。

etcd客户端在“MemberAdd” API中添加了一个标志，用于指示学习者节点。etcd服务器处理器使用pb.ConfChangeAddLearnerNode类型的成员变更条目应用成员更改条目。一旦命令被应用，服务器将以etcd --initial-cluster-state=existing标志加入集群。这个学习者节点既不能投票，也不能作为法定人数。

etcd服务器不能将领导权转移给学习者，因为它可能仍然滞后，而且不算作法定人数。etcd服务器限制集群可以拥有的学习者数量为一个：学习者越多，领导者就必须传播的数据越多。客户端可以与学习者节点通信，但学习者除了可串行化读取和成员状态API之外拒绝所有请求。

这是为了初始实现的简单性。将来，学习者可以扩展为一个连续镜像集群数据的只读服务器。客户端负载均衡器必须提供帮助函数来排除学习者节点的端点。否则，发送给学习者的请求可能会失败。客户端同步成员调用应考虑学习者节点类型。客户端端点更新调用也应该考虑。

MemberList和MemberStatus响应应指示哪个节点是学习者。

### 添加“MemberPromote” API。

在Raft内部，对学习者节点的第二个MemberAdd调用会将其提升为投票成员。领导者会维护每个跟随者和学习者的进度。如果学习者尚未完成其快照消息，则拒绝晋升请求。只有在以下情况下才接受晋升请求：学习者节点处于健康状态。学习者与领导者同步，或者增量在阈值内（例如，要复制到学习者的条目数少于快照计数的1/10，这意味着即使在晋升后，领导者也不需要向学习者发送快照）。所有这些逻辑都是在etcdserver包中硬编码的，而不可配置。

# 4. etcd-raft membership change's liveness problems

# Abstract

In [#7625 (comment)](https://github.com/etcd-io/etcd/issues/7625#issuecomment-489232411), [@tbg](https://github.com/tbg)证明了应用时配置更改的安全性。但仍然存在两个存活性问题。我们讨论了这些问题并提出了解决方案。最后，我们证明了解决方案是可行的，存活性可以得到保证。

# Problem 1

尽管在配置更改期间有多数的Cold,new可用，但仍然可能永远无法选举出领导者。

## Example 1

联合共识

Cold => 投票者: {A, B, C}，学习者: {D} 

Cnew => 投票者: {A, B, D}，学习者: {C} 

现在配置为Cold。A是领导者，B崩溃了。 A提议Cold,new和其他一些条目。D接收了所有这些条目，但是C由于网络问题只接收了Cold,new和其他一些条目的一部分。在C向A发送附加响应之后，A知道Cold,new已经提交，因为Cold的大多数已经有了这个条目（即A和C）。C知道Cold,new已经提交，而D不知道，但是D的条目比C的更为更新。 （这可能发生是因为C通过心跳知道了提交索引，或者A由于流控制而无法将所有条目发送给C）

A: ... Cold,new, E … En … Em 

B: ... 

C: ... Cold,new, E … En 

D: ... Cold,new, E … En … Em

现在A崩溃了，B恢复了。 C在竞选之前应用了Cold,new。然后C向A、B、D发送了投票请求，因为它的配置是Cold,new。<u>显然，B会向C授予投票权，而D不会。D不知道Cold,new已经提交，所以它仍然是学习者，并且不会竞选。 因此，即使Cold,new的多数可用（即B、C、D），也可能永远无法选举出领导者。</u>

## Example 2

Single Membership Change
Cold => voters: {A, B, C}, learners: {D}
Cnew => voters: {A, B, C, D}
Now the configuration is Cold. A is leader and B crashes.
A proposes Cnew to promote learner D and some other entries. The subsequent process is the same as example 1. D’s entry is more up-to-date than C's but C knows Cnew is committed.
After A crashes and B recovers, no leader can be elected even if both the majority of Cold and Cnew is available.
Note that this kind of issue can not happen if the number of voters in Cold is even because the quorum number does not change and a leader can be elected without the new voter.

## Example 3

Joint Consensus
Cold => voters: {A, B, C}, learners: {D}
Cnew => voters: {A, B, D}, learners: {C}
Now Cold,new is committed. A is leader and B crashes.
A proposes Cnew. C and D receive Cnew and responds to A. A knows Cnew is committed and then proposes some other entries. C receives these entries and knows Cnew is committed while D does not.

A: ... Cold,new, Cnew, E …
B: ...
C: ... Cold,new, Cnew, E …
D: ... Cold,new, Cnew

Now A crashes and B recovers.
C applies the Cnew and becomes learner. D sends vote requests to A, B, C because its configuration is Cold,new. Obviously, B will grant its vote to D while C won’t.
Thus no leader can be elected even if the majority of Cold,new is available(B, C, D).

## Example 4

Single Membership Change
Cold => voters: {A, B, C, D}
Cnew => voters: {A, B, D}, learners: {C}
Now the configuration is Cold. A is leader and B crashes.
The subsequent process is the same as example 3.
No leader can be elected even if both the majority of Cold and Cnew is available.
Note that this kind of issue can not happen if the number of voters in Cold is odd because the quorum number does not change and a leader can be elected without the new learner.

For example 1 and 2, there is a learner D who has an up-to-date log but does not know the configuration is committed.
For example 3 and 4, there is a learner C who has an up-to-date log and also knows the configuration is committed but other voters don’t know it.

## Solution

经过与 [@NingLin-P](https://github.com/NingLin-P) 的讨论，我们发现这个问题可以通过使用投票请求和响应来解决。

- 投票请求和响应应携带配置更改的最后提交索引和任期。当节点接收到投票请求或响应时，如果存在具有相同任期和索引的本地条目，则应该将其提交索引提升。

例如 1 和 2，D 将收到来自 C 的投票请求，因此它将其提交索引提升到 Cold,new。**然后 D 应用 Cold,new，并成为 follower。**在发起竞选后，它成为 leader。

例如 3 和 4，C 将收到来自 D 的投票请求，因此它将发送带有 Cnew 提交索引和任期的投票响应。然后 D 将提升其提交索引，应用 Cnew，并在发起竞选后成为 leader。

在 [#7625 (comment)](https://github.com/etcd-io/etcd/issues/7625#issuecomment-551041747) 中，[@tbg](https://github.com/tbg) 在 CockroachDB 中提到，在 [#11284 (comment)](https://github.com/etcd-io/etcd/issues/11284#issue-510417965) 中，他们解决了可用性问题2，通过不直接删除投票者。而是先将其降级然后再删除。

我认为这不应该仅仅是客户端的一种解决方法。我们必须在 raft 库中禁止直接删除投票者，否则，活跃性保证将会被破坏。									

# Problem 2

There is a confusing thing that the client can not know whether this cluster is available when a majority of configuration is alive after it receives the success response of this configuration.

In other words, when can we believe the quorum is always majority of this new configuration?
The answer is after the majority of configuration knows the configuration entry is committed.

But this problem does not exist in some cases. Next, we will discuss different situations.

## Add voter/Promote learner

### Joint Consensus

Cold,new does this work. It’s not a problem because any quorum of Cold,new include a certain quorum of Cold, the cluster is available when the majority of Cold,new is alive.

### Single Membership Change

If number of voters in Cold is odd, this problem does not exist because any quorum of Cnew include a certain quorum of Cold, the cluster is available when the majority of Cnew is alive.
But if it is even, this problem exists.
For example:
Cold => voters: {A, B}, learners: {C}
Cnew => voters: {A, B, C}
Leader A proposes Cnew(promote C). A commits and applies Cnew. B and C have this entry but don’t know it’s committed. Then A crashes. No leader can be elected because both B and C do not know Cnew is committed.

## Demote voter

### Joint Consensus

Cnew does this work.

- If we want to demote voters to learners and then remove them, the majority of Cnew knows Cnew is committed due to append invariant after the Cremove (removing learner(s)) is committed. Now the quorum is always majority of Cnew, the learner can be safely removed. Note that this method has a little disadvantage that we must keep the majority of Cold,new available until the Cremove is committed.
- If we just want to demote voters to learners, this problem exists. For example 3 in problem 1, if C is isolated, no leader can be elected after A crashes because both B and D do not know Cnew is committed. C is a learner but it's still needed for the majority of Cnew.

### Single Membership Change

Like add voter/promote learner, if number of voters in Cold is odd, this problem does not exist.
If it is even, it has the same situations as joint consensus.

## Solution

领导者可以追踪跟随者保存的提交索引。（raft-rs已经实现了这个功能 [tikv/raft-rs#366](https://github.com/tikv/raft-rs/pull/366)）。在大多数跟随者的提交索引大于配置索引后，我们可以在 Ready 中向客户端提供这些信息。

另一种方法是：在配置被提交或应用后，领导者可以提出另一个日志。如果此日志被提交，则由于追加不变性，这意味着大多数配置知道该配置已经提交。

# Liveness argument

Three conditions mentioned above.

1. Vote request and response should carry the commit index and term of the last configuration change.
2. Forbid removing a voter directly.
3. If the majority of configuration knows the configuration entry is committed, the quorum is always the majority of the new configuration.(i.e. leader can be elected if majority of configuration is available)

Under these conditions, we can prove a leader can be elected at all steps of configuration change.

## Joint Consensus

We assume that a majority of the old configuration is available (at least until the majority of Cnew knows Cnew is committed) and that a majority of the new configuration is available.

1. If Cold,new is not committed
   (a) The available peer with the most up-to-date log can collect votes from a majority of Cold and become leader.
2. If Cold,new is committed and Cnew is not committed
   (a) For a certain quorum of Cold,new, if no peer knows Cold,new is committed, it’s the same as situation 1.
   (b) Otherwise, this peer who knows Cold,new is committed must be voter after applying Cold,new. This voter sends vote request to other voters in this quorum of Cold,new. Each voter who has Cold,new entry can know Cold,new is committed through vote request due to condition 1. One of them with the most up-to-date log can collect votes from a majority of Cold,new and become leader.
3. If Cnew is committed
   (a) For a certain quorum of Cold,new, if no peer knows Cnew is committed, it’s the same as situation 2.b because there is at least one peer in any quorum of Cold,new knows Cold,new is committed. (append invariant)
   (b) Otherwise, because of condition 2, no peer can be removed even if it knows Cnew is committed and applies Cnew. Each voter who has Cnew entry but does not know Cnew is committed sends vote request to other voters in this Cold,new’s quorum. They will know Cnew is committed through vote response due to condition 1. One of them with the most up-to-date log can collect votes from a majority of Cnew and become leader.
4. If majority of Cnew knows C
   (a) For a certain quorum of Cnew, at least one of them knows the Ce  committed. So it’s the same as situation 3.b

## Single Membership Change

We assume that a majority of the old configuration is available (at least until the majority of Cnew knows Cnew is committed) and that a majority of the new configuration is available.

If Cnew is not committed
(a) The available peer with the most up-to-date log can collect votes from a majority of Cold and become leader.

1. If Cnew is committed
   (a) For a certain quorum of Cold, if no peer knows Cnew is committed, it’s the same as situation 1.
   (b) Otherwise, there is a peer A in Cold knows Cnew is committed. Because of condition 2, no peer can be removed even if it knows Cnew is committed and applies Cnew. Next, there must be a voter B in quorum of Cnew has Cnew entry because the intersection of any quorum of Cold and any quorum of Cnew is non-empty. If peer A and voter B are different peer, voter B will know Cnew is committed through vote response from peer A due to condition 1. Then each voter in Cnew who has Cnew entry can know Cnew is committed through vote request of peer B. One of them with the most up-to-date log can collect votes from a majority of Cnew and become leader.
2. If majority of Cnew knows Cnew is committed
   For a certain quorum of Cnew, at least one of them knows the Cnew is committed. It’s similar to situation 2.b.
