---
title: etcd-raft整体架构
date: 2023-06-07T09:31:59Z
lastmod: 2023-06-07T12:56:03Z
---

# etcd-raft整体架构

> 引用：http://blog.mrcroxx.com/posts/code-reading/etcdraft-made-simple/2-overview/

## RaftNode

就像在raft-example当中分析的那样，Raft Package只负责Raft 的具体逻辑的实现，而存储通信等模块就交给了上层的Raft Server，而二者之间的桥梁即为`interface node`​ Raft Package对于raft进行进一步的封装，对外只暴露一个node借口，开发者可以使用Node接口当中的函数来使用Raft，而对于Node接口，Raft Package由定义了两个实现的结构体，分别是node 和rawnode，二者的主要区别为node通过channel保证了线程安全问题。

```go
type Node interface {
	// Tick increments the internal logical clock for the Node by a single tick. Election
	// timeouts and heartbeat timeouts are in units of ticks.
	Tick()
	// Campaign causes the Node to transition to candidate state and start campaigning to become leader.
	Campaign(ctx context.Context) error
	// Propose proposes that data be appended to the log. Note that proposals can be lost without
	// notice, therefore it is user's job to ensure proposal retries.
	Propose(ctx context.Context, data []byte) error
	// ProposeConfChange proposes a configuration change. Like any proposal, the
	// configuration change may be dropped with or without an error being
	// returned. In particular, configuration changes are dropped unless the
	// leader has certainty that there is no prior unapplied configuration
	// change in its log.
	//
	// The method accepts either a pb.ConfChange (deprecated) or pb.ConfChangeV2
	// message. The latter allows arbitrary configuration changes via joint
	// consensus, notably including replacing a voter. Passing a ConfChangeV2
	// message is only allowed if all Nodes participating in the cluster run a
	// version of this library aware of the V2 API. See pb.ConfChangeV2 for
	// usage details and semantics.
	ProposeConfChange(ctx context.Context, cc pb.ConfChangeI) error

	// Step advances the state machine using the given message. ctx.Err() will be returned, if any.
	Step(ctx context.Context, msg pb.Message) error

	// Ready returns a channel that returns the current point-in-time state.
	// Users of the Node must call Advance after retrieving the state returned by Ready.
	//
	// NOTE: No committed entries from the next Ready may be applied until all committed entries
	// and snapshots from the previous one have finished.
	Ready() <-chan Ready

	// Advance notifies the Node that the application has saved progress up to the last Ready.
	// It prepares the node to return the next available Ready.
	//
	// The application should generally call Advance after it applies the entries in last Ready.
	//
	// However, as an optimization, the application may call Advance while it is applying the
	// commands. For example. when the last Ready contains a snapshot, the application might take
	// a long time to apply the snapshot data. To continue receiving Ready without blocking raft
	// progress, it can call Advance before finishing applying the last ready.
	Advance()
	// ApplyConfChange applies a config change (previously passed to
	// ProposeConfChange) to the node. This must be called whenever a config
	// change is observed in Ready.CommittedEntries, except when the app decides
	// to reject the configuration change (i.e. treats it as a noop instead), in
	// which case it must not be called.
	//
	// Returns an opaque non-nil ConfState protobuf which must be recorded in
	// snapshots.
	ApplyConfChange(cc pb.ConfChangeI) *pb.ConfState

	// TransferLeadership attempts to transfer leadership to the given transferee.
	TransferLeadership(ctx context.Context, lead, transferee uint64)

	// ReadIndex request a read state. The read state will be set in the ready.
	// Read state has a read index. Once the application advances further than the read
	// index, any linearizable read requests issued before the read request can be
	// processed safely. The read state will have the same rctx attached.
	// Note that request can be lost without notice, therefore it is user's job
	// to ensure read index retries.
	ReadIndex(ctx context.Context, rctx []byte) error

	// Status returns the current status of the raft state machine.
	Status() Status
	// ReportUnreachable reports the given node is not reachable for the last send.
	ReportUnreachable(id uint64)
	// ReportSnapshot reports the status of the sent snapshot. The id is the raft ID of the follower
	// who is meant to receive the snapshot, and the status is SnapshotFinish or SnapshotFailure.
	// Calling ReportSnapshot with SnapshotFinish is a no-op. But, any failure in applying a
	// snapshot (for e.g., while streaming it from leader to follower), should be reported to the
	// leader with SnapshotFailure. When leader sends a snapshot to a follower, it pauses any raft
	// log probes until the follower can apply the snapshot and advance its state. If the follower
	// can't do that, for e.g., due to a crash, it could end up in a limbo, never getting any
	// updates from the leader. Therefore, it is crucial that the application ensures that any
	// failure in snapshot sending is caught and reported back to the leader; so it can resume raft
	// log probing in the follower.
	ReportSnapshot(id uint64, status SnapshotStatus)
	// Stop performs any necessary termination of the Node.
	Stop()
}
```

结构体当中的注视已经给的较为清晰，挑几个重点说一下：

* 首先是Tick，正如之前所说的那样，etcd当中的raft模块使用的是逻辑时钟，即需要手动调用某些函数来推进时钟，即为此处的Tick，该Tick留给上层在serverChannels当中进行调用。

  在Node当中既有一个Tick()函数的实现，其向一个channel当中发送一个空结构体作为通知，而这个通知就被node.run()当中的一个select接收到，之后调用下层的raft的tick最终驱动时钟

  而rn的Tick()为一个函数类型的字段，会根据当前raftnode的身份而选择不同的Tick()，如TickElection()、TickHeartBeat()等

  ```go
  func (n *node) Tick() {
  	select {
  	case n.tickc <- struct{}{}:
  	case <-n.done:
  	default:
  		n.rn.raft.logger.Warningf("%x A tick missed to fire. Node blocks too long!", n.rn.raft.id)
  	}
  }

  // in node.run()
      select {
  	case <-n.tickc:
  	    n.rn.Tick()
      }
  ```

* Campaign和Propose分别是用来进行选举和将上层kvstore传下来的请求封装为日志，这个留到之后的选举流程和日志再进行分析
* Step()为整个Raft模块的核心，整个Raft状态机的改变或者更新都由Step函数来完成
* Ready()之前已经分析过，在Raft Package当中已经完成了的工作，之后需要上层的raft server进行处理的都通过Ready进行传递，如需要持久化存储的HardState、Entries，已经提交了需要进行Apply的committedEntries，以及需要发送给其他节点的Messages[]
* Advance()，通常和Ready()成对出现，当上层的Raft Server处理完一批通过Ready返回的内容之后，通过Advance()来通知下层的Raft Node已经完成，推进进度
* ReadIndex()用于保证线性一致性，读取状态具有一个读取索引。一旦应用程序的进度超过了读取索引，任何在读取请求之前发出的线性可读请求就可以安全地处理。

## Raft Package

还是先看一下相关的结构体

```go
type raft struct {
	id uint64

	Term uint64
	Vote uint64

	readStates []ReadState

	// the log
	raftLog *raftLog

	maxMsgSize         uint64
	maxUncommittedSize uint64
	// TODO(tbg): rename to trk.
	prs tracker.ProgressTracker

	state StateType

	// isLearner is true if the local raft node is a learner.
	isLearner bool

	msgs []pb.Message

	// the leader id
	lead uint64
	// leadTransferee is id of the leader transfer target when its value is not zero.
	// Follow the procedure defined in raft thesis 3.10.
	leadTransferee uint64
	// Only one conf change may be pending (in the log, but not yet
	// applied) at a time. This is enforced via pendingConfIndex, which
	// is set to a value >= the log index of the latest pending
	// configuration change (if any). Config changes are only allowed to
	// be proposed if the leader's applied index is greater than this
	// value.
	pendingConfIndex uint64
	// an estimate of the size of the uncommitted tail of the Raft log. Used to
	// prevent unbounded log growth. Only maintained by the leader. Reset on
	// term changes.
	uncommittedSize uint64

	readOnly *readOnly

	// number of ticks since it reached last electionTimeout when it is leader
	// or candidate.
	// number of ticks since it reached last electionTimeout or received a
	// valid message from current leader when it is a follower.
	electionElapsed int

	// number of ticks since it reached last heartbeatTimeout.
	// only leader keeps heartbeatElapsed.
	heartbeatElapsed int

	checkQuorum bool
	preVote     bool

	heartbeatTimeout int
	electionTimeout  int
	// randomizedElectionTimeout is a random number between
	// [electiontimeout, 2 * electiontimeout - 1]. It gets reset
	// when raft changes its state to follower or candidate.
	randomizedElectionTimeout int
	disableProposalForwarding bool

	tick func()
	step stepFunc

	logger Logger

	// pendingReadIndexMessages is used to store messages of type MsgReadIndex
	// that can't be answered as new leader didn't committed any log in
	// current term. Those will be handled as fast as first log is committed in
	// current term.
	pendingReadIndexMessages []pb.Message
}
```

在其中除了一个raft本身的结构体，还有一个config结构体，在在上层的startRaft当中调用配置Raft的初始信息

```go
type Config struct {
	// ID is the identity of the local raft. ID cannot be 0.
	ID uint64

	// ElectionTick is the number of Node.Tick invocations that must pass between
	// elections. That is, if a follower does not receive any message from the
	// leader of current term before ElectionTick has elapsed, it will become
	// candidate and start an election. ElectionTick must be greater than
	// HeartbeatTick. We suggest ElectionTick = 10 * HeartbeatTick to avoid
	// unnecessary leader switching.
	ElectionTick int
	// HeartbeatTick is the number of Node.Tick invocations that must pass between
	// heartbeats. That is, a leader sends heartbeat messages to maintain its
	// leadership every HeartbeatTick ticks.
	HeartbeatTick int

	// Storage is the storage for raft. raft generates entries and states to be
	// stored in storage. raft reads the persisted entries and states out of
	// Storage when it needs. raft reads out the previous state and configuration
	// out of storage when restarting.
	Storage Storage
	// Applied is the last applied index. It should only be set when restarting
	// raft. raft will not return entries to the application smaller or equal to
	// Applied. If Applied is unset when restarting, raft might return previous
	// applied entries. This is a very application dependent configuration.
	Applied uint64

	// MaxSizePerMsg limits the max byte size of each append message. Smaller
	// value lowers the raft recovery cost(initial probing and message lost
	// during normal operation). On the other side, it might affect the
	// throughput during normal replication. Note: math.MaxUint64 for unlimited,
	// 0 for at most one entry per message.
	MaxSizePerMsg uint64
	// MaxCommittedSizePerReady limits the size of the committed entries which
	// can be applied.
	MaxCommittedSizePerReady uint64
	// MaxUncommittedEntriesSize limits the aggregate byte size of the
	// uncommitted entries that may be appended to a leader's log. Once this
	// limit is exceeded, proposals will begin to return ErrProposalDropped
	// errors. Note: 0 for no limit.
	MaxUncommittedEntriesSize uint64
	// MaxInflightMsgs limits the max number of in-flight append messages during
	// optimistic replication phase. The application transportation layer usually
	// has its own sending buffer over TCP/UDP. Setting MaxInflightMsgs to avoid
	// overflowing that sending buffer. TODO (xiangli): feedback to application to
	// limit the proposal rate?
	MaxInflightMsgs int

	// CheckQuorum specifies if the leader should check quorum activity. Leader
	// steps down when quorum is not active for an electionTimeout.
	CheckQuorum bool

	// PreVote enables the Pre-Vote algorithm described in raft thesis section
	// 9.6. This prevents disruption when a node that has been partitioned away
	// rejoins the cluster.
	PreVote bool

	// ReadOnlyOption specifies how the read only request is processed.
	//
	// ReadOnlySafe guarantees the linearizability of the read only request by
	// communicating with the quorum. It is the default and suggested option.
	//
	// ReadOnlyLeaseBased ensures linearizability of the read only request by
	// relying on the leader lease. It can be affected by clock drift.
	// If the clock drift is unbounded, leader might keep the lease longer than it
	// should (clock can move backward/pause without any bound). ReadIndex is not safe
	// in that case.
	// CheckQuorum MUST be enabled if ReadOnlyOption is ReadOnlyLeaseBased.
	ReadOnlyOption ReadOnlyOption

	// Logger is the logger used for raft log. For multinode which can host
	// multiple raft group, each raft group can have its own logger
	Logger Logger

	// DisableProposalForwarding set to true means that followers will drop
	// proposals, rather than forwarding them to the leader. One use case for
	// this feature would be in a situation where the Raft leader is used to
	// compute the data of a proposal, for example, adding a timestamp from a
	// hybrid logical clock to data in a monotonically increasing way. Forwarding
	// should be disabled to prevent a follower with an inaccurate hybrid
	// logical clock from assigning the timestamp and then forwarding the data
	// to the leader.
	DisableProposalForwarding bool
}
```

## Message

整个Raft的状态机的切换都是通过Step函数，而Step函数调用时会传入一个message，因此不妨先看一下message

grpc当中的消息传输为protobuf的形式，message通过grpc进行传输，首先在raft.proto当中定义，之后可以生成对应的go结构体。

所有的rpc的消息都通过Message来承载，如RequestVote,AppendEntries，通过一个MessageType进行区分。

Message也并不完全用于消息传输，对于本机的操作涉及到修改状态机的也可以通过封装成一个Message来交给Step处理

```go
type Message struct {
	Type MessageType `protobuf:"varint,1,opt,name=type,enum=raftpb.MessageType" json:"type"`
	To   uint64      `protobuf:"varint,2,opt,name=to" json:"to"`
	From uint64      `protobuf:"varint,3,opt,name=from" json:"from"`
	Term uint64      `protobuf:"varint,4,opt,name=term" json:"term"`
	// logTerm is generally used for appending Raft logs to followers. For example,
	// (type=MsgApp,index=100,logTerm=5) means leader appends entries starting at
	// index=101, and the term of entry at index 100 is 5.
	// (type=MsgAppResp,reject=true,index=100,logTerm=5) means follower rejects some
	// entries from its leader as it already has an entry with term 5 at index 100.
	LogTerm    uint64   `protobuf:"varint,5,opt,name=logTerm" json:"logTerm"`
	Index      uint64   `protobuf:"varint,6,opt,name=index" json:"index"`
	Entries    []Entry  `protobuf:"bytes,7,rep,name=entries" json:"entries"`
	Commit     uint64   `protobuf:"varint,8,opt,name=commit" json:"commit"`
	Snapshot   Snapshot `protobuf:"bytes,9,opt,name=snapshot" json:"snapshot"`
	Reject     bool     `protobuf:"varint,10,opt,name=reject" json:"reject"`
	RejectHint uint64   `protobuf:"varint,11,opt,name=rejectHint" json:"rejectHint"`
	Context    []byte   `protobuf:"bytes,12,opt,name=context" json:"context,omitempty"`
}
```

消息的类型可以在![MessageType](https://pkg.go.dev/go.etcd.io/etcd/raft/v3#hdr-MessageType)当中查看

```go
type MessageType int32
// MsgHup用于进行选举，当超时是向step函数传入MsgHup
// MsgBeat用于让Leader向其follower去发送心跳信息
// MsgProp用于AE，传入step之后Leader首先先向自己写入Entries，之后向follower广播
// MsgApp当中包含要添加的Entries，如果candidate收到之后，会将自身转换为follower
// 
const (
	MsgHup            MessageType = 0
	MsgBeat           MessageType = 1
	MsgProp           MessageType = 2
	MsgApp            MessageType = 3
	MsgAppResp        MessageType = 4
	MsgVote           MessageType = 5
	MsgVoteResp       MessageType = 6
	MsgSnap           MessageType = 7
	MsgHeartbeat      MessageType = 8
	MsgHeartbeatResp  MessageType = 9
	MsgUnreachable    MessageType = 10
	MsgSnapStatus     MessageType = 11
	MsgCheckQuorum    MessageType = 12
	MsgTransferLeader MessageType = 13
	MsgTimeoutNow     MessageType = 14
	MsgReadIndex      MessageType = 15
	MsgReadIndexResp  MessageType = 16
	MsgPreVote        MessageType = 17
	MsgPreVoteResp    MessageType = 18
)

```

## Step

etcd当中将所有和Raft状态机相关的操作全部集中于Step当中，而具体应当进行何种操作则根据Step当中传入的Message当中的MessageType进行判断，而对于Message的调用，一般在Node当中或直接或间接的对其进行调用。

对于Step，首先定义一个统一的Step函数，在其中进行一些预处理或者综合处理，即无论当前的节点的身份为何都可能需要进行的操作，之后在单独定义几个step函数，针对不同身份的节点。在Step当中，综合处理完之后仍需处理的就调用单独的step。而如果有一些操作就是针对某些特定身份，也可以直接调用对应的stepxxx，无需调用Step。

针对特定身份的step如下，作为一个函数类型的变量定义在结构体当中，在状态切换时进行设置

```go
func stepLeader(r *raft, m pb.Message) error {...}
func stepCandidate(r *raft, m pb.Message) error {...}
func stepFollower(r *raft, m pb.Message) error {...}
```

```go
func (r *raft) becomeFollower(term uint64, lead uint64) {
	r.step = stepFollower
	r.reset(term)
	r.tick = r.tickElection
	r.lead = lead
	r.state = StateFollower
	r.logger.Infof("%x became follower at term %d", r.id, r.Term)
}
```

```go

	switch m.Type {
	case xxx:
        // ...
	default:
		err := r.step(r, m)
		if err != nil {
			return err
		}
```

‍

`Campaign`:Campaign只会由candidate出发，对`step`​的直接调用，最终会调用到`stepCandidate`​

```go
func (n *node) Campaign(ctx context.Context) error {
 return n.step(ctx, pb.Message{Type: pb.MsgHup}) 
}
```

`Tick`​：算是对Step的间接调用，主要顺序如下：

1. 调用Tick时向channel当中发送一条通知
2. 该通知被运行node.run()当中的协程获取到，调用下层raft的Tick，该Tick同样为一个函数类型的变量，如果当前为Leader，那么就会调用到`tickHeartBeat`​
3. 在`tickHeartBeat`​当中调用到最终的Step，对状态机进行操作

```go
func (n *node) Tick() {
	select {
	case n.tickc <- struct{}{}:
	case <-n.done:
	default:
		n.rn.raft.logger.Warningf("%x A tick missed to fire. Node blocks too long!", n.rn.raft.id)
	}
}
```

```go
case <-n.tickc:
		n.rn.Tick()
```

```go
func (rn *RawNode) Tick() {
	rn.raft.tick()
}
```

```go
func (r *raft) tickHeartbeat() {
	r.heartbeatElapsed++
	r.electionElapsed++

	if r.electionElapsed >= r.electionTimeout {
		r.electionElapsed = 0
		// 检查当前活跃的follower是否满足quorum 即大多数
		if r.checkQuorum {
			r.Step(pb.Message{From: r.id, Type: pb.MsgCheckQuorum})
		}
		// If current leader cannot transfer leadership in electionTimeout, it becomes leader again.
		if r.state == StateLeader && r.leadTransferee != None {
			r.abortLeaderTransfer()
		}
	}

	if r.state != StateLeader {
		return
	}

	if r.heartbeatElapsed >= r.heartbeatTimeout {
		r.heartbeatElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgBeat})
	}
}
```

‍


‍
