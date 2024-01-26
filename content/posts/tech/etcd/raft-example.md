---
title: raft-example
date: 2023-05-31T20:28:32Z
lastmod: 2023-06-06T23:52:43Z
---

# raft-example

在etcd当中，提供了一个raft-example，该程序并非构建了一个完整的Raft模块，而是对Raft模块的的基本使用。并在此基础上构建了一个简单的KV存储结构。而Raft模块是作为一个包的形式存在的，Raft模块只提供如Leader Election Log Replication Snapshot等Raft的基本逻辑功能，而在此之上的存储、网络通信等都交给了使用Raft包的开发者来自行决定。

因此最终的结构就分为了三层：

* 最底层的Raft Package，只提供Raft最基本的逻辑功能
* Raft服务器，调用底层的Raft Package的相关API，自定义日志和快照的存储以及网络通信等
* 内存数据库，和Raft服务器之间通过channel进行通信，向下提交日志，并且接受Raft服务器等Commit信息，之后将其应用于自身的存储模块当中。

![](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/20230606235427.png)

## kvstore

此处的设计基本上与MIT6.824一致，首先看一下kvstore的结构体

```cpp
type kvstore struct {
	proposeC    chan<- string // channel for proposing updates
	mu          sync.RWMutex
	kvStore     map[string]string // current committed key-value pairs
	snapshotter *snap.Snapshotter
}
```

* proposeC用于向下层的Raft服务器提交新的请求，之后下层的Raft服务器拿到之后调用Raft Package当中的相关API封装为日志，之后形成共识
* kvStore即为内存数据库，但是可以通过快照 + WAL日志来提供持久化稳定存储
* snapshotter用于处理快照等相关消息，如Load Save等，在etcdserver/api/snap当中实现

**创建**​

kvStore的创建过程也比较简单，首先尝试从快照当中恢复数据，之后就单独开启一个goroutine，监听从下层Raft传来的日志提交信息，如果提交，就将其应用到自身

```cpp
func newKVStore(snapshotter *snap.Snapshotter, proposeC chan<- string, commitC <-chan *commit, errorC <-chan error) *kvstore {
	s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
	snapshot, err := s.loadSnapshot()
	if err != nil {
		log.Panic(err)
	}
	if snapshot != nil {
		log.Printf("loading snapshot at term %d and index %d", snapshot.Metadata.Term, snapshot.Metadata.Index)
		if err := s.recoverFromSnapshot(snapshot.Data); err != nil {
			log.Panic(err)
		}
	}
	// read commits from raft into kvStore map until error
	go s.readCommits(commitC, errorC)
	return s
}

```

readCommit的实现和MIT6.824当中的实现稍有不同，由于ectd的快照是真实存储的，因此下层的Raft只需要通过一个`nil`​来告知一下上层有了新的快照，之后上层就可以去进行读取，之后就是读取commit信息应用于自身，最后通过close chan的形式通知下层Raft上层应用完毕。

```cpp
func (s *kvstore) readCommits(commitC <-chan *commit, errorC <-chan error) {
	for commit := range commitC {
		// 相比于MIT 6.824 此处的snapshot为真正持久化的，因此无需通过channel传输
		// 因此在chan当中只需要发送一个nil用于通知即可
		if commit == nil {
			// signaled to load snapshot
			snapshot, err := s.loadSnapshot()
			if err != nil {
				log.Panic(err)
			}
			if snapshot != nil {
				log.Printf("loading snapshot at term %d and index %d", snapshot.Metadata.Term, snapshot.Metadata.Index)
				if err := s.recoverFromSnapshot(snapshot.Data); err != nil {
					log.Panic(err)
				}
			}
			continue
		}
		// handle the commit data
		// ....
		close(commit.applyDoneC)
	}
	if err, ok := <-errorC; ok {
		log.Fatal(err)
	}
}
```

除了几个从Snapshot当中加载数据的函数之外，就只剩写入和读取了。写入操作和MIT6.824一样，当提交了一个写入请求之后，向下提交一个日志，此时kvstore会一直阻塞，直至下层的Raft形成了共识并向上传递了commit的信息，之后才会通知客户端写入成功。

但是对于读取操作，在raft-example当中并没有通过Raft日志来保证强一致性，而是直接在Leader处进行本地读的操作,可以提高读操作的qps，但是相对的，强一致性就无法得到保证。

```cpp
func (s *kvstore) Lookup(key string) (string, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	v, ok := s.kvStore[key]
	return v, ok
}

func (s *kvstore) Propose(k string, v string) {
	var buf bytes.Buffer
	if err := gob.NewEncoder(&buf).Encode(kv{k, v}); err != nil {
		log.Fatal(err)
	}
	// 将kv写入到下层，等待raft完成共识
	s.proposeC <- buf.String()
}
```

在kvstore之上还有一层httpApi，用于对外提供网络服务，比较简单就放一下代码

```cpp
func (h *httpKVAPI) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	key := r.RequestURI
	defer r.Body.Close()
	switch {
	case r.Method == "PUT":
		v, err := ioutil.ReadAll(r.Body)
		if err != nil {
			log.Printf("Failed to read on PUT (%v)\n", err)
			http.Error(w, "Failed on PUT", http.StatusBadRequest)
			return
		}

		h.store.Propose(key, string(v))

		// Optimistic-- no waiting for ack from raft. Value is not yet
		// committed so a subsequent GET on the key may return old value
		w.WriteHeader(http.StatusNoContent)
	case r.Method == "GET":
		if v, ok := h.store.Lookup(key); ok {
			w.Write([]byte(v))
		} else {
			http.Error(w, "Failed to GET", http.StatusNotFound)
		}
	case r.Method == "POST":
		url, err := ioutil.ReadAll(r.Body)
		if err != nil {
			log.Printf("Failed to read on POST (%v)\n", err)
			http.Error(w, "Failed on POST", http.StatusBadRequest)
			return
		}

		nodeId, err := strconv.ParseUint(key[1:], 0, 64)
		if err != nil {
			log.Printf("Failed to convert ID for conf change (%v)\n", err)
			http.Error(w, "Failed on POST", http.StatusBadRequest)
			return
		}

		cc := raftpb.ConfChange{
			Type:    raftpb.ConfChangeAddNode,
			NodeID:  nodeId,
			Context: url,
		}
		h.confChangeC <- cc

		// As above, optimistic that raft will apply the conf change
		w.WriteHeader(http.StatusNoContent)
	case r.Method == "DELETE":
		nodeId, err := strconv.ParseUint(key[1:], 0, 64)
		if err != nil {
			log.Printf("Failed to convert ID for conf change (%v)\n", err)
			http.Error(w, "Failed on DELETE", http.StatusBadRequest)
			return
		}

		cc := raftpb.ConfChange{
			Type:   raftpb.ConfChangeRemoveNode,
			NodeID: nodeId,
		}
		h.confChangeC <- cc

		// As above, optimistic that raft will apply the conf change
		w.WriteHeader(http.StatusNoContent)
	default:
		w.Header().Set("Allow", "PUT")
		w.Header().Add("Allow", "GET")
		w.Header().Add("Allow", "POST")
		w.Header().Add("Allow", "DELETE")
		http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
	}
}
```

## Raft

Raft模块，或者说Raft服务器，构建于Raft的包之上，Raft包提供的Leader Eelction Log Replication功能之上，再提供日志和快照的存储形式、节点之间的通信方式等功能，构建出一个完整的Raft。

首先还是看一下Raft的结构体

```cpp
type raftNode struct {
	proposeC    <-chan string            // proposed messages (k,v)
	confChangeC <-chan raftpb.ConfChange // proposed cluster config changes
	commitC     chan<- *commit           // entries committed to log (k,v)
	errorC      chan<- error             // errors from raft session

	id          int      // client ID for raft session
	peers       []string // raft peer URLs
	join        bool     // node is joining an existing cluster
	waldir      string   // path to WAL directory
	snapdir     string   // path to snapshot directory
	getSnapshot func() ([]byte, error)

	confState     raftpb.ConfState
	snapshotIndex uint64
	appliedIndex  uint64

	// raft backing for the commit/error channel
	node        raft.Node
	raftStorage *raft.MemoryStorage
	wal         *wal.WAL

	snapshotter      *snap.Snapshotter
	snapshotterReady chan *snap.Snapshotter // signals when snapshotter is ready

	snapCount uint64
	transport *rafthttp.Transport
	stopc     chan struct{} // signals proposal channel closed
	httpstopc chan struct{} // signals http server to shutdown
	httpdonec chan struct{} // signals http server shutdown complete

	logger *zap.Logger
}
```

|proposeC|从 kvstore当中接受新的请求，创建为日志|
| ---------------| -------------------------------------------------------------|
|confChangeC|Raft集群配置更新的相关消息|
|commitC|向kvstore发送日志提交的信息|
|errorC|传递错误信息|
|snapshotIndex|快照化的最后一条日志的Index，用于在故障恢复之后找到Index|
|appliedIndex|应用于kvstorte的最后一条日志Index，用于故障恢复|
|node|Raft Package对外提供的API接口|
|raftStorage|稳定存储日志，由于使用了WAL，因此可以使用内存的形式稳定存储|
|wal|对WAL日志操作的封装|
|transport|raft节点之间的通信|

此外还有三个struct{}的 chan用于进行消息通知，接收方使用select阻塞，发送方向其中发送一个空结构体即可使其从阻塞当中恢复。

### Raft的创建

定义一个newRaftNode供上层的kvstore进行调用，传入一个propose的channel用于接受kvstore的新的请求，之后初始化raftNode的部分值，并将commitC与errorC返回交给kvstore。之后再单独开启一个goroutine调用startRaft来启动一些raft的服务。

```cpp
func newRaftNode(id int, peers []string, join bool, getSnapshot func() ([]byte, error), proposeC <-chan string,
	confChangeC <-chan raftpb.ConfChange) (<-chan *commit, <-chan error, <-chan *snap.Snapshotter) {

	commitC := make(chan *commit)
	errorC := make(chan error)

	rc := &raftNode{
		proposeC:    proposeC,
		confChangeC: confChangeC,
		commitC:     commitC,
		errorC:      errorC,
		id:          id,
		peers:       peers,
		join:        join,
		waldir:      fmt.Sprintf("raftexample-%d", id),
		snapdir:     fmt.Sprintf("raftexample-%d-snap", id),
		getSnapshot: getSnapshot,
		snapCount:   defaultSnapshotCount,
		stopc:       make(chan struct{}),
		httpstopc:   make(chan struct{}),
		httpdonec:   make(chan struct{}),

		logger: zap.NewExample(),

		snapshotterReady: make(chan *snap.Snapshotter, 1),
		// rest of structure populated after WAL replay
	}
	go rc.startRaft()
	return commitC, errorC, rc.snapshotterReady
}
```

### 初始化

**startRaft**​

在startRaft当中继续完成一部分初始化的工作，当前的节点有可能是之前宕机之后重启，因此需要首先检查快照与写入的WAL，从中读取快照并通知上层的kvstore去应用快照恢复内存数据库，和恢复Raft日志到memoryStorage当中。

之后再对底层的Raft Package进行一些相关的配置，如设置心跳信息等

> etcd当中使用的为逻辑时钟，即通过Tick来推进时钟，如将heartbeat的间隔设置为1次tick，选举的时间间隔设置为10次tick

之后再定义raft节点之间的传输协议之后，初始化基本完成，之后分别开启一个协程去负责节点之间的通信和一个用于处理和kvstore的channel和raft package的channel

```go
func (rc *raftNode) startRaft() {
	// handle snapshot and wal
	// ...
	c := &raft.Config{
		ID:                        uint64(rc.id),
		ElectionTick:              10,
		HeartbeatTick:             1,
		Storage:                   rc.raftStorage,
		MaxSizePerMsg:             1024 * 1024,
		MaxInflightMsgs:           256,
		MaxUncommittedEntriesSize: 1 << 30,
	}
	// 在创建节点时，如果通过原本的WAL日志进行log和snapshot的加载
	// 或者为中途加入到集群当中的节点相比于新节点少了通过bootstrap进行初始化加载
	if oldwal || rc.join {
		rc.node = raft.RestartNode(c)
	} else {
		rc.node = raft.StartNode(c, rpeers)
	}

	rc.transport = &rafthttp.Transport{
		Logger:      rc.logger,
		ID:          types.ID(rc.id),
		ClusterID:   0x1000,
		Raft:        rc,
		ServerStats: stats.NewServerStats("", ""),
		LeaderStats: stats.NewLeaderStats(zap.NewExample(), strconv.Itoa(rc.id)),
		ErrorC:      make(chan error),
	}

	rc.transport.Start()
	for i := range rc.peers {
		if i+1 != rc.id {
			rc.transport.AddPeer(types.ID(i+1), []string{rc.peers[i]})
		}
	}
	// serveRaft对外监听端口提供服务
	// serveChannels处理存储层发起的请求，和raftNode所传来的相关信息
	go rc.serveRaft()
	go rc.serveChannels()
}
```

### channels处理

重点看一下serveChannels

在serverChannels当中，又单独开了一个goroutine，其中通过select 监听kvstore的propose channel和集群配置的channel，调用Raft Package的API交给下层的Raft Package进行处理。

```cpp
	go func() {
		confChangeCount := uint64(0)

		for rc.proposeC != nil && rc.confChangeC != nil {
			select {
			case prop, ok := <-rc.proposeC:
				if !ok {
					rc.proposeC = nil
				} else {
					// blocks until accepted by raft state machine
					rc.node.Propose(context.TODO(), []byte(prop))
				}

			case cc, ok := <-rc.confChangeC:
				if !ok {
					rc.confChangeC = nil
				} else {
					confChangeCount++
					cc.ID = confChangeCount
					rc.node.ProposeConfChange(context.TODO(), cc)
				}
			}
		}
		// client closed channel; shutdown raft if not already
		close(rc.stopc)
	}()
```

而serverChannels自身的goroutine用于处理Raft Package处理完毕的消息，共select 两个channel，一个用于处理定时器，如果通过这个channel收到了消息那么就调用Tick函数推进逻辑时钟。

另外一个Channel是通过raftnode.Ready()函数返回的一个 `chan Ready`​

Ready的结构体定义如下：

```cpp
type Ready struct {
	// The current volatile state of a Node.
	// SoftState will be nil if there is no update.
	// It is not required to consume or store SoftState.
	*SoftState

	// The current state of a Node to be saved to stable storage BEFORE
	// Messages are sent.
	// HardState will be equal to empty state if there is no update.
	pb.HardState

	// ReadStates can be used for node to serve linearizable read requests locally
	// when its applied index is greater than the index in ReadState.
	// Note that the readState will be returned when raft receives msgReadIndex.
	// The returned is only valid for the request that requested to read.
	ReadStates []ReadState

	// Entries specifies entries to be saved to stable storage BEFORE
	// Messages are sent.
	Entries []pb.Entry

	// Snapshot specifies the snapshot to be saved to stable storage.
	Snapshot pb.Snapshot

	// CommittedEntries specifies entries to be committed to a
	// store/state-machine. These have previously been committed to stable
	// store.
	CommittedEntries []pb.Entry

	// Messages specifies outbound messages to be sent AFTER Entries are
	// committed to stable storage.
	// If it contains a MsgSnap message, the application MUST report back to raft
	// when the snapshot has been received or has failed by calling ReportSnapshot.
	Messages []pb.Message

	// MustSync indicates whether the HardState and Entries must be synchronously
	// written to disk or if an asynchronous write is permissible.
	MustSync bool
}
```

此处我们只需要HardState、Entries、Snapshot、Messages、CommitedEntries

就像上文所说的那样，Raft Package并不负责通信和存储，这两部分都要交给RaftServer处理，因此将HardState、Entries写入到WAL当中，Snapshot同样进行保存。之后再把Entries添加到raftStorage当中。

而Message即为当前节点产生的所有的需要发送给其他节点的消息，都需要在此处进行发送，通过之前定义的transport模块进行发送。

而对于底层Raft Package已经达成共识认定为committed 的Entries，通过`publishEntries`​处理后通过commitC通知上层的kvstore。

