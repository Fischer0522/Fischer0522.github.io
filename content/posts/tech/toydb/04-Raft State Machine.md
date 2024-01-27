---
title: "04-Raft State Machine"
date: 2023-12-30T19:31:59Z
---



在正式写这一篇文章之前，原本的规划是在这一部分介绍sql执行引擎，即如何将Raft，Raft状态机，存储引擎组合起来，为SQL的执行去提供支持，但是这一部分又不可避免的牵扯到SQL执行过程，如果全部展示出来，内容将会非常多，思来想去，还是Raft状态机这一部分比较独立，使用了一个Trait屏蔽掉了底层的存储实现，因此就先将这一模块单独拆出来。

在toydb当中，Raft状态机的实现定义于`src/raft/state.rs`当中。在etcd，对于Raft Package也可以理解成一个状态机，因此，为了方便进行区分，在开头首先声明，在这一章出现的所有状态机的概念都是指构建于Raft上层的状态机，上一章分析的Raft会以Raft模块(Raft Package)来指代。
这里与MIT-6.824有一个非常大的不同是：
- 在MIT-6.824当中，client是与上层状态机进行通信的，命令会先输入给状态机，之后状态机向下创建一条日志，发送给Raft模块去进行共识，当达到quorum共识之后，状态机再给client以回应，告知此次请求执行完成
- 在toydb当中，Client的请求是直接发送给Raft模块的，在上一章当中，我们可以看到，server从client_rx当中获取到了消息之后，直接调用step交给了Raft模块去处理，Raft模块处理完之后会发送给状态机，状态机进行一些逻辑校验，之后再发送Message给Raft模块，之后由Raft模块将消息回复给Client，所以在Raft模块当中，能够看到很多和ClientRequest，ClientResponse相关的内容。

```rust
	// 获取client发送的消息，交给Raft模块去处理，这里的client并不是用户的client，而是要执行命令的sql端
	Some((request, response_tx)) = client_rx.next() => {
		let id = Uuid::new_v4().as_bytes().to_vec();
		let msg = Message{
			from: Address::Client,
			to: Address::Node(node.id()),
			term: 0,
			event: Event::ClientRequest{id: id.clone(), request},
		};
		node = node.step(msg)?;
		requests.insert(id, response_tx);
	}
```

![channel](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/20231228140251.png)

## Driver
在这里，toydb当中的Raft状态机更像是一个比较存粹的状态机，从Raft模块当中接收输入(Instruction)，更新自身状态之后，再给Raft模块输出(Messsage)，不会涉及到网络通信部分的内容，不会与Client进行交互，所以状态机基本上就是起到了一个记录的作用，比如，记录当前接收到了那些请求，各个请求的apply的情况，从而确定什么时候该让Raft模块去回应Client，在这一部分定义了一个`Driver`，用于驱动状态机的执行：
- `state_rx`用于从Raft模块当中接收信息
- `node_tx`用于向Raft模块发送信息
- `notify`：用于当某条日志apply之后，通知Raft模块，让其给Client回应，Raft Leader在收到Mutate类型的ClientRequest之后，会进行propose来复制日志，并且向状态机当中发送一条notify：保存在状态机当中
- `queries`：将只读请求保存到上层状态机，达到quorum时会再告知Leader进行读取，实现了ReadIndex的效果。
```rust
/// Drives a state machine, taking operations from state_rx and sending results via node_tx.
pub struct Driver {
    node_id: NodeID,
    state_rx: UnboundedReceiverStream<Instruction>,
    node_tx: mpsc::UnboundedSender<Message>,
    /// Notify clients when their mutation is applied. <index, (client, id)>
    notify: HashMap<Index, (Address, Vec<u8>)>,
    /// Execute client queries when they receive a quorum. <index, <id, query>>
    queries: BTreeMap<Index, BTreeMap<Vec<u8>, Query>>,
}
```
入口为drive函数，drive函数会不断的从`state.rx`当中不断获取Instuction，然后调用`self.execute`来执行：
```rust
    pub async fn drive(mut self, mut state: Box<dyn State>) -> Result<()> {
        debug!("Starting state machine driver at applied index {}", state.get_applied_index());
        while let Some(instruction) = self.state_rx.next().await {
            if let Err(error) = self.execute(instruction, &mut *state) {
                error!("Halting state machine due to error: {}", error);
                return Err(error);
            }
        }
        debug!("Stopping state machine driver");
        Ok(())
    }
```

在`execute()`当中，就是根据instruction的类型，去调用不同的函数，那我们只需要弄明白每种Instruction的类型和功能，以及对应的发送位置和响应方式即可：
## State Trait
在介绍Raft模块与Raft状态机交互之前，先来看一下Raft状态机的Trait和实现，Raft状态机是以Trait的形式定义的。
```rust
/// A Raft-managed state machine.
pub trait State: Send {
    /// Returns the last applied index from the state machine.
    fn get_applied_index(&self) -> Index;

    /// Applies a log entry to the state machine. If it returns Error::Internal,
    /// the Raft node halts. Any other error is considered applied and returned
    /// to the caller.
    ///
    /// The entry may contain a noop command, which is committed by Raft during
    /// leader changes. This still needs to be applied to the state machine to
    /// properly track the applied index, and returns an empty result.
    ///
    /// TODO: consider using runtime assertions instead of Error::Internal.
    fn apply(&mut self, entry: Entry) -> Result<Vec<u8>>;

    /// Queries the state machine. All errors are propagated to the caller.
    fn query(&self, command: Vec<u8>) -> Result<Vec<u8>>;
}
```
而真正的实现在`src/sql/engine/raft.rs`当中，定义了一个`struct State<E>`来作为Raft状态机的实现，在其中，只有两个变量：
- `last_applied_index`用于记录当前应用的最后的日志
- `super::KV<E>`是一个MVCC Storage的Wrapper(这一部分在02-MVCC当中已经介绍过)，提供了存储元数据和开启事务的功能。
但是这个struct是具体怎么实现这些功能的，由于牵扯到sql执行的一些内容，并且不进行展开也不会影响到后续的内容，就留到下一章了。
```rust
/// The Raft state machine for the Raft-based SQL engine, using a KV SQL engine
pub struct State<E: storage::engine::Engine> {
    /// The underlying KV SQL engine
    engine: super::KV<E>,
    /// The last applied index
    applied_index: u64,
}
```

## Instruction
在Raft State Machine当中，Instruction的地位相当于Raft模块当中Message的地位。在这一部分，主要会分析Raft状态机是如何与Raft模块进行交互的，按照Instruction的类型，理顺各类Instruction的发送和处理的方式与时机。功能实现如下：
- 写请求：Notify
- 只读请求：Query + Vote
- 应用日志：Apply
- 终止请求：Abort
```rust
pub enum Instruction {
    /// Abort all pending operations, e.g. due to leader change.
    Abort,
    /// Apply a log entry.
    Apply { entry: Entry },
    /// Notify the given address with the result of applying the entry at the given index.
    Notify { id: Vec<u8>, address: Address, index: Index },
    /// Query the state machine when the given term and index has been confirmed by vote.
    Query { id: Vec<u8>, address: Address, command: Vec<u8>, term: Term, index: Index, quorum: u64 },
    /// Extend the given server status and return it to the given address.
    Status { id: Vec<u8>, address: Address, status: Box<Status> },
    /// Votes for queries at the given term and commit index.
    Vote { term: Term, index: Index, address: Address },
}
```

### 写请求执行
`Driver.notify`用于记录客户端的一次写请求，在leader接收到客户端的写请求时(类型为 Mutate)，就会向Raft状态机发送一条`Instruction::Notify`，将请求记录到状态机当中，等待之后该请求对应的日志Apply了，再告知Leader，让Leader去响应客户端，返回执行结果:

**发送**

Leader接收到Client发送而来的类型为`Request::Mutate`的`ClientRequest`，在Raft模块层面，会调用`self.propose()`来进行日志复制，在Raft状态机层面，Leader会向Raft状态机发送一条`Instruction::Notify`来记录请求。
```rust
	// Leader.step()
	pub fn step(mut self, msg: Message) -> Result<Node> {
		Event::ClientRequest { id, request: Request::Mutate(command) } => {
			let index = self.propose(Some(command))?;
			self.state_tx.send(Instruction::Notify { id, address: msg.from, index })?;
			if self.peers.is_empty() {
				self.maybe_commit()?;
			}
		}
	}
```

在`Driver.execute()`当中，如果高于目前的appled_index，那么就插入保存，等待之后该条日志apply，否则为出现错误，让Leader告知客户端。
```rust
    /// Executes a state machine instruction. 
    fn execute(&mut self, i: Instruction, state: &mut dyn State) -> Result<()> {
        match i {
            Instruction::Notify { id, address, index } => {
                if index > state.get_applied_index() {
                    self.notify.insert(index, (address, id));
                } else {
                    self.send(address, Event::ClientResponse { id, response: Err(Error::Abort) })?;
                }
            }
        }
        Ok(())
    }
```

在`Leader.step()`当中，会处理由Raft状态机发来的`Message::ClientResponse`，Leader不会做什么处理，直接发送给Client就好，对于其他的Instruction，处理类型也是相同，后面不会再提及。
```rust
	Event::ClientResponse { id, mut response } => {
		if let Ok(Response::Status(ref mut status)) = response {
			status.server = self.id;
		}
		self.send(Address::Client, Event::ClientResponse { id, response })?;
	}
```
### 只读请求优化
在Raft的基础实现当中，无论是写请求还是读请求，都需要创建一条日志进行写入，之后执行时按照日志commit的顺序进行执行，从而提供线性一致性。但这样的问题就在于，即便是读请求也需要创建日志并写入，带来了额外的磁盘IO，从而提高系统延迟。

在Raft当中，为了保证线性一致性，需要保证能够读到目前最新的commit_index。如果想对Log Read进行优化，那么就需要想办法绕过写入Log对这个过程，比较常见的两种优化方式是ReadIndex和Lease Read。由于Raft的读写都经过Leader(不考虑Follower Read的优化)，那么只要能够确认当前的Leader身份有效，就可以直接从Leader本地进行读取，为了确认Leader的有效身份，ReadIndex和Lease Read采用了两种不同的方式：
- ReadIndex:ReadIndex记录收到请求时的commit index，然后通过发送一轮心跳的方式来确认Leader的合法身份，如果能确认收到Quorum数量的响应，那么当前的Leader身份就是合法的，当Leader的apply index >= 之前保存的commit index时，就可以根据commit index去读取。
- Lease Read:Lease Read的核心思想是利用了一个选举时间差，当收到Leader心跳信息之后，follower就会重置选举超时时间，那么直到下一次选举超时之前，目前Leader的身份一定是合法的，不会有其他的节点通过选举成为Leader，因此就可以直接读取Leader的commit index。不过Lease Read会受到时钟偏移的影响，一种比较简单的解决方法是将Lease时间设置为比超时时间短一点。(有那么点TrueTime的意思)
如果想进一步了解只读请求优化，可以看这一篇：[深入浅出etcd/raft —— 0x06 只读请求优化 - 叉鸽 MrCroxx 的博客](http://blog.mrcroxx.com/posts/code-reading/etcdraft-made-simple/6-readonly/#11-log-read)

在toydb当中，只读请求的优化采用的是ReadIndex，在`Leader.step()`当中，会处理Client发送来的`Event::ClientRequest`类型的请求，其中request类型为`Request::Query`代表只读请求：
1. Leader首先创建一条`Instruction::Query`将只读请求记录到状态机当中
2. 之后再发送一条`Instruction::Vote`来表示记录当前节点确认了Leader的身份
3. 发送Heartbeat，来尝试证明自身为合法的Leader
```rust
    pub fn step(mut self, msg: Message) -> Result<Node> {
		Event::ClientRequest { id, request: Request::Query(command) } => {
			let (commit_index, _) = self.log.get_commit_index();
			self.state_tx.send(Instruction::Query {
				id,
				address: msg.from,
				command,
				term: self.term,
				index: commit_index,
				quorum: self.quorum(),
			})?;
			self.state_tx.send(Instruction::Vote {
				term: self.term,
				index: commit_index,
				address: Address::Node(self.id),
			})?;
			self.heartbeat()?;
		}
	}
```

Follower收到了Heartbeat信息之后，先更新自身的commit进度和进行apply，之后会发送一条`Message::ConfirmLeader`来认可Leader的身份，发送给Leader,`Message::ConfirmLeader`当中会携带Follower的commit进度，用于给状态机来计算quorum，相当于原本Leader计算commit index。
```rust
    // Follower.step()
    pub fn step(mut self, msg: Message) -> Result<Node> {
		// 先处理commit和apply
		// ....
		// 确认Leader身份
		self.send(msg.from, Event::ConfirmLeader { commit_index, has_committed })?;
	}
```

Leader会统计`Message::ConfirmLeader`信息，每收到一条就会向状态机发送`Message::Vote`，状态机会进行记录和计算是否达到quorum。
```rust
    pub fn step(mut self, msg: Message) -> Result<Node> {	
		// A follower received one of our heartbeats and confirms that we
		// are its leader. If it doesn't have the commit index in its local
		// log, replicate the log to it.
		Event::ConfirmLeader { commit_index, has_committed } => {
			let from = msg.from.unwrap();
			// 与上层状态机进行交互
			self.state_tx.send(Instruction::Vote {
				term: msg.term,
				index: commit_index,
				address: msg.from,
			})?;
			if !has_committed {
				self.send_log(from)?;
			}
		}
	}
```

在状态机当中，只读请求使用的是一个嵌套的BTreeMap管理的，对应`<index, <id, query>>`,表示在当前的commit index下，都有哪些请求，这里的id是由Client发送而来的，表示请求的唯一性ID，而不是NodeID。
```rust
struct Query {
    id: Vec<u8>,
    term: Term,
    address: Address,
    command: Vec<u8>,
    quorum: u64,
    votes: HashSet<Address>,
}

pub struct Driver {
	// ...
    /// Execute client queries when they receive a quorum. <index, <id, query>>
    queries: BTreeMap<Index, BTreeMap<Vec<u8>, Query>>,
}

    fn execute(&mut self, i: Instruction, state: &mut dyn State) -> Result<()> {

		Instruction::Query { id, address, command, index, term, quorum } => {
			self.queries.entry(index).or_default().insert(
				id.clone(),
				Query { id, term, address, command, quorum, votes: HashSet::new() },
			);
		}
	}
```

在状态机收到了`Instruction::Vote`之后，分别调用了两个函数：
```rust
    /// Executes a state machine instruction. 
    fn execute(&mut self, i: Instruction, state: &mut dyn State) -> Result<()> {
        debug!("Executing {:?}", i);
        match i {
            Instruction::Vote { term, index, address } => {
                self.query_vote(term, index, address);
                self.query_execute(state)?;
            }
        }
        Ok(())
    }
```

在`self.query_vote`当中会根据`Instruction::Vote`当中的commit index去记录投票，只能对`commit_index <= Vote.commit_index`的Query进行投票。这样，只有Leader的apply_index >= commit_index之后才能过通过quorum检查。
```rust
    /// Votes for queries up to and including a given commit index for a term by an address.
    fn query_vote(&mut self, term: Term, commit_index: Index, address: Address) {
        for (_, queries) in self.queries.range_mut(..=commit_index) {
            for (_, query) in queries.iter_mut() {
                if term >= query.term {
                // 使用HashSet记录保证同一个节点(使用Address表示)，只能对同一个commit_index投票一次
                    query.votes.insert(address);
                }
            }
        }
    }
```

在`self.query_execute`当中，会获取出所有`index <= applied_index`的只读请求，然后调用状态机的实现来执行只读请求，然后告知Raft模块的Leader去响应Client：
```rust
    /// Executes any queries that are ready.
    fn query_execute(&mut self, state: &mut dyn State) -> Result<()> {
	    // self.query_ready()会获取出所有的达到quorum并且index <= applied_index的Query，并且从self.queries当中移除
        for query in self.query_ready(state.get_applied_index()) {
            debug!("Executing query {:?}", query.command);
            let result = state.query(query.command);
            if let Err(error @ Error::Internal(_)) = result {
                return Err(error);
            }
            self.send(
                query.address,
                Event::ClientResponse { id: query.id, response: result.map(Response::Query) },
            )?
        }
        Ok(())
    }
```

### Abort
当Client发送了一条请求之后，Raft模块就会将请求保存到Raft状态机当中，之后等待日志commit并apply了，再给Client响应。

如果当前的Leader收到了更高的Term，那么就会退位，不是Leader自然就不能够处理请求，因此存在状态机当中还未处理完的请求就需要全部放弃，这一部分逻辑定义在`leader.become_follower()`当中：
```rust
		// If we receive a message for a future term, become a leaderless
	// follower in it and step the message. If the message is a Heartbeat or
	// AppendEntries from the leader, stepping it will follow the leader.
	if msg.term > self.term {
		return self.become_follower(msg.term)?.step(msg);
	}

    /// Transforms the leader into a follower. This can only happen if we find a
    /// new term, so we become a leaderless follower.
    fn become_follower(mut self, term: Term) -> Result<RoleNode<Follower>> {
        assert!(term >= self.term, "Term regression {} -> {}", self.term, term);
        assert!(term > self.term, "Can only become follower in later term");

        info!("Discovered new term {}", term);
        self.term = term;
        self.log.set_term(term, None)?;
        self.state_tx.send(Instruction::Abort)?;
        Ok(self.become_role(Follower::new(None, None)))
    }
```

在Raft状态机当中，收到了旧Leader发送的`Instruction::Abort`之后，在`execute()`当中就会调用`notify_abort()`和`query_abort()`，将原本用于存储请求的notify重置，同时清除用于保存只读请求的Query，并且再封装一条Message发送回已经退位的Leader，让其告知Client请求已经被取消执行。
```rust
    fn execute(&mut self, i: Instruction, state: &mut dyn State) -> Result<()> {
        debug!("Executing {:?}", i);
        match i {
            Instruction::Abort => {
                self.notify_abort()?;
                self.query_abort()?;
            }
            // ...
            // ...
        }
    }
    
    
    
    /// Aborts all pending notifications.
    fn notify_abort(&mut self) -> Result<()> {
        for (_, (address, id)) in std::mem::take(&mut self.notify) {
            self.send(address, Event::ClientResponse { id, response: Err(Error::Abort) })?;
        }
        Ok(())
    }
    
    /// Aborts all pending queries.
    fn query_abort(&mut self) -> Result<()> {
        for (_, queries) in std::mem::take(&mut self.queries) {
            for (id, query) in queries {
                self.send(
                    query.address,
                    Event::ClientResponse { id, response: Err(Error::Abort) },
                )?;
            }
        }
        Ok(())
    }
```

### Apply
当一条日志在Raft模块当中得到了大多数的共识之后，就视其为`commit`，而`commit`了的日志会`Apply`到Raft状态机，此时代表完成了一次写入，写入结果对外可见。在toydb当中，`Commit`和`Apply`的动作是连贯的，一旦获取到了新的commit_index，就会立即向状态机发送进行apply的commit entry:
- Leader通过自身进行计算
- Follower通过Heartbeat得知当前的commit进度，再结合自身的日志复制进度得出一个commit_index
```rust
	// Leader
	
	if commit_index > prev_commit_index {
		self.log.commit(commit_index)?;
		// TODO: Move application elsewhere, but needs access to applied index.
		let mut scan = self.log.scan((prev_commit_index + 1)..=commit_index)?;
		while let Some(entry) = scan.next().transpose()? {
			self.state_tx.send(Instruction::Apply { entry })?;
		}
	}
	
	// Follower
	Event::Heartbeat { commit_index, commit_term } => {
		// ....


		// Advance commit index and apply entries if possible.
		let has_committed = self.log.has(commit_index, commit_term)?;
		let (old_commit_index, _) = self.log.get_commit_index();
		if has_committed && commit_index > old_commit_index {
			self.log.commit(commit_index)?;
			let mut scan = self.log.scan((old_commit_index + 1)..=commit_index)?;
			while let Some(entry) = scan.next().transpose()? {
				self.state_tx.send(Instruction::Apply { entry })?;
			}
		}
		self.send(msg.from, Event::ConfirmLeader { commit_index, has_committed })?;
	}
```

当Raft状态机接收到了`Instruction::Apply`之后，调用定义在`sql/engine/raft`当中Raft状态机的实现来进行Apply，然后调用`self.notify_applyed()`，将Apply的情况通知给下层的Raft模块，此时Raft模块就可以给Client回应了。

并且某些只读请求也可能因为apply_index的推进，从而可以执行了，调用`query_execute`尝试执行。`query_execute()`在上文已经介绍过了，会获取所有达到index <= apply_index的Query尝试执行。

```rust
    fn execute(&mut self, i: Instruction, state: &mut dyn State) -> Result<()> {
        match i {
        // ...
            Instruction::Apply { entry } => {
                self.apply(state, entry)?;
            }
        /// ...
        }
    }

    /// Applies an entry to the state machine.
    pub fn apply(&mut self, state: &mut dyn State, entry: Entry) -> Result<Index> {
        // Apply the command.
        debug!("Applying {:?}", entry);
        match state.apply(entry) {
            Err(error @ Error::Internal(_)) => return Err(error),
            result => self.notify_applied(state.get_applied_index(), result)?,
        };
        // Try to execute any pending queries, since they may have been submitted for a
        // commit_index which hadn't been applied yet.
        self.query_execute(state)?;
        Ok(state.get_applied_index())
    }


    /// Notifies a client about an applied log entry, if any.
    fn notify_applied(&mut self, index: Index, result: Result<Vec<u8>>) -> Result<()> {
        if let Some((to, id)) = self.notify.remove(&index) {
            self.send(to, Event::ClientResponse { id, response: result.map(Response::Mutate) })?;
        }
        Ok(())
    }
```

## Others
在文章的最后，再补充一些零散的内容
### Driver启动
Driver用于驱动状态机，调用`Driver.drive()`之后，就会不断处理由Raft模块发送而来的`Instruction`，`Driver.drive()`会在`Node.new()`当中调用，随着Node的创建一同启动
```rust
impl Node {
    /// Creates a new Raft node, starting as a follower, or leader if no peers.
    pub async fn new(
        id: NodeID,
        peers: HashSet<NodeID>,
        mut log: Log,
        mut state: Box<dyn State>,
        node_tx: mpsc::UnboundedSender<Message>,
    ) -> Result<Self> {
        let (state_tx, state_rx) = mpsc::unbounded_channel();
        // Driver用于和Raft状态机进行交互，启动状态机运行
        let mut driver = Driver::new(id, state_rx, node_tx.clone());
        driver.apply_log(&mut *state, &mut log)?;
        tokio::spawn(driver.drive(state));

        let (term, voted_for) = log.get_term()?;
        let mut node = RoleNode {
            id,
            peers,
            term,
            log,
            node_tx,
            state_tx,
            role: Follower::new(None, voted_for),
        };
        if node.peers.is_empty() {
            info!("No peers specified, starting as leader");
            // If we didn't vote for ourself in the persisted term, bump the
            // term and vote for ourself to ensure we have a valid leader term.
            if voted_for != Some(id) {
                node.term += 1;
                node.log.set_term(node.term, Some(id))?;
            }
            let (last_index, _) = node.log.get_last_index();
            Ok(node.become_role(Leader::new(HashSet::new(), last_index)).into())
        } else {
            Ok(node.into())
        }
    }
}
```

### 信息传递
Raft状态机和Raft模块之间使用`Message`和`Instruction`进行交互，Raft模块向Raft状态机发送`Instruction`，Raft状态机回应`Message`给Raft模块，Message的发送定义如下：
```rust
    /// Sends a message.
    /// 状态机使用node_tx进行消息发送，那么对应的接收端就是node_rx
    fn send(&self, to: Address, event: Event) -> Result<()> {
        // TODO: This needs to use the correct term.
        let msg = Message { from: Address::Node(self.node_id), to, term: 0, event };
        debug!("Sending {:?}", msg);
        Ok(self.node_tx.send(msg)?)
    }
```

### Follower
如6.824一样，Follower当然也是有状态机的，只不过当中的逻辑非常简单，就没有额外介绍，Follower只需要不断的接受日志，然后更新commit_index然后向状态机发送`Instruction::Apply`就好，代码在上文也有展示。
## Summary
在toydb当中，相比于MIT-6.824实现了一个比较纯粹的状态机，只需要以Raft模块发送的`Instruction`作为输入，内部进行状态更新，之后以`Message`为输出即可，主要起到了一个记录和逻辑判断，通知的作用，不会进行对外的网络交互。Leader在状态机的指示下去回应Client，完成一条请求的执行。







