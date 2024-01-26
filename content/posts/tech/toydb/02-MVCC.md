---
title: "02-MVCC"

---



## 基础理论

简而言之，实现MVCC的DBMS在内部维持着单个逻辑数据的多个物理版本，当事务修改某条数据时，就创建一个新的版本。当事务读取时，就根据事务的开始时间，去读取事务开始时刻之前的最新版本。

MVCC概括起来就是两句话：
- Writers don't block readers.
- Readers don't block writers.

只读事务无需加锁就可以读取数据库某一时刻的快照，如果保留数据的所有历史版本，DBMS甚至能够支持读取任意历史版本的数据，即time-travel(这一点在toydb当中也得到了实现，即不实现gc，保留之前所有的版本，开发者还特意强调了这是一个feature而不是bug)
### 并发控制
MVCC(Multi-Version Concurrency Control)的名字具有一定的误导性，虽然叫做并发控制，但是本身并不是一个完整的并发控制协议，正如上面所说的，MVCC只能解决R-W之间的冲突问题，但是对于W-W，单靠MVCC本身是无法解决的，需要引入其他的并发协议，根据并发协议的种类，又可以大致分为：
- MV2PL：使用2PL悲观锁的形式来解决W-W冲突问题
- MVOCC：使用基础的时间戳或者创建private workspace的形式(事务分为read,write,validate三个过程)，二者其实有细微的区别的，但是本质都是乐观的形式，就归在一起了
- 还有最简单的形式,如果发现当前事务要修改的record正在被其他事务修改，就放弃并之后重试(又不是不能用 :) )，也算是一种乐观的实现方式吧

这里先看一个toydb当中给出的例子来理解一下，
```rust
//! Time
//! 5
//! 4  a4          
//! 3      b3      x
//! 2            
//! 1  a1      c1  d1
//!    a   b   c   d   Keys
```
- 在t1时刻，某个事务写入了a=a1,c=c1,d=d1并提交
- 在t3时刻，某个事务写入了b=b3,删除了d
- 在t4时刻，某个事务写入了a=a4
- 在t2时刻，开启了事务T1，那么他就能够读取到a=a1,c=c1,d=d1
- 在 t 5 时刻，开启了事务 T 2, 那么他就能够读取到 a=a 4, b=b 3,c=c1

事务的时间或者版本的概念是根据事务begin决定的，比如说T2读取的物理时间可能落后于T5，但是由于T2事务begin早于T5，所以他就能够读取到的数据的版本就早于T5(其实这个也是根据并发控制协议决定的，如果使用OCC的话，事务的时间就是validate的时间)。

记录真正变成可见是根据提交的时刻决定的，在事务未提交前，该事务写入的数据对于自己是可见的，但是对于其他的事务不可见，在看一个例子：
```rust
//! Active set: [2, 5]
//!
//! Time
//! 5 (a5)
//! 4  a4          
//! 3      b3      x
//! 2         (x)     (e2)
//! 1  a1      c1  d1
//!    a   b   c   d   e   Keys
```
事务T2写入的数据，但是并未提交，T2维护在Active set当中,删除c1和写入e2对于自身是可见的，但是对于后面开启的事务T5是不可见的。

### MVCC in Miniob
在介绍toydb的MVCC的实现之前，先看一下Miniob的MVCC实现,虽然存在变更无法一次性对外暴露的问题，但是实现的比较简单，很好理解：

MVCC当中的版本是基于tid的，在开启了MVCC模式之后，每一条record会生成两个sys_field，分别存储begin和end，来标识一个事务的可见性，这里似乎并没有一个统一的标准，只要能满足MVCC协议本身的要求即可，在Miniob当中的设置如下：

record通过begin和end 两个id进行状态管理，begin用于表示事务开始的时间，end为事务结束的时间，对于一条record：
- 当一个事务开始时，新的record的begin设置为-trx_id，end为max_int32,表示事务写入但未提交，而删除时则将end设置为-trx_id，表示删除但未提交
- 写入操作提交时，将begin设置为trx_id，删除操作提交时将end设置为trx_id，最终会产生五种状态

| begin_xid | end_xid | 自身可见 | 其他事务可见             | 说明                                                 |
| --------- | ------- | -------- | ------------------------ | ---------------------------------------------------- |
| -trx_id   | MAX     | 可见     | 不可见                   | 由当前事务写入，但还未提交，对其他事务不可见         |
| trx_id    | MAX     | 已提交   | 对新事务可见             | 写入并且已经提交，对之后的新事务可见                 |
| 任意正数  | trx_id  | 已提交   | 旧事务可见，新事务不可见 | 在当前事务中进行删除，并事务提交                     |
| 任意正数  | -trx_id | 不可见   | 可见                     | 在当前事务中进行删除，未提交                         |
| -trx_id   | -trx_id | 不可见   | 不可见                   | 由当前事务进行写入，并且又被当前事务删除，并且未提交 |

其中，已提交指的就是当前事务已经结束，自然不存在什么可见不可见的问题。而Miniob当中的隔离级别应当是和toydb一样的快照隔离，因为未提交的事务的 begin < 0，因此是永远无法读取到新写入的record，因此是不存在幻读情况的。

对于record是否可见的判断，在visit_record当中提供了一段代码：
```cpp
if (begin_xid > 0 && end_xid > 0) {
  if (trx_id_ >= begin_xid && trx_id_ <= end_xid) {
    rc = RC::SUCCESS;
  } else {
    rc = RC::RECORD_INVISIBLE;
  }
} else if (begin_xid < 0) {
  // begin xid 小于0说明是刚插入而且没有提交的数据
  rc = (-begin_xid == trx_id_) ? RC::SUCCESS : RC::RECORD_INVISIBLE;
} else if (end_xid < 0) {
  // end xid 小于0 说明是正在删除但是还没有提交的数据
  if (readonly) {
    // 如果 -end_xid 就是当前事务的事务号，说明是当前事务删除的
    rc = (-end_xid != trx_id_) ? RC::SUCCESS : RC::RECORD_INVISIBLE;
  } else {
    // 如果当前想要修改此条数据，并且不是当前事务删除的，简单的报错
    // 这是事务并发处理的一种方式，非常简单粗暴。其它的并发处理方法，可以等待，或者让客户端重试
    // 或者等事务结束后，再检测修改的数据是否有冲突
    rc = (-end_xid != trx_id_) ? RC::LOCKED_CONCURRENCY_CONFLICT : RC::RECORD_INVISIBLE;
  }
}

```

如果想进一步了解Miniob的事务模块的话，可以看这个：[miniob-transaction.md](https://github.com/oceanbase/miniob/blob/main/docs/src/design/miniob-transaction.md)
除此之外，23fall的15-455同样也提供了MVCC，基于MVOCC实现的，以单个Field为单位实现的多版本，笔者目前即没做也没细看，读者如果感兴趣的话可以看以下链接：[Project #4 - Concurrency Control | CMU 15-445/645 :: Intro to Database Systems (Fall 2023)](https://15445.courses.cs.cmu.edu/fall2023/project4/)

## 实现
首先补充点理论：
- 在toydb当中，MVCC所提供的隔离级别为快照隔离，事务只能看到数据库的一个一致性快照，而这个快照是根据事务创建的时间决定的，即事务只能够看到事务创建前的最新的数据，以及由自己写入的新数据，目前还未提交的活跃事务之间相互隔离互不影响。
- Toydb 并没有实现 GC 功能，会保存数据的所有版本，因此就可以支持 time travel query，即传入一个时间戳，然后获取一个那时的快照，进行只读请求 (由于基于旧版本进行写请求会扰乱当前数据库的状态，如进行 set x = x + 1，原本 x = 3，但是目前已经是 x = 5 了，因此 time travel 只支持只读事务)，开发者特意强调了这是一个 feature 而不是 bug，不过感觉多少有点欲盖弥彰了。

好了，接下来来看一下具体的实现，在toydb当中，事务的相关逻辑全部定义在`src/storage/mvcc.rs`当中，只有一个文件，其他的像是`debug.rs`，`keycode.rs`只是提供一些辅助支持，用到的时候再看看。

在介绍MVCC以及事务是如何实现之前，先梳理一下定义的结构体和之间的关系
### Transaction
事务最基础的结构体为`Transaction`：
- Engine为一个Trait，在其中提供了基础的CRUD功能，而上一章介绍的Bitcask和Memory都实现了这个Trait，具体使用的哪个上层应用无需关心
- TransactionState用于表示事务的状态
```rust
/// An MVCC transaction.
pub struct Transaction<E: Engine> {
    /// The underlying engine, shared by all transactions.
    engine: Arc<Mutex<E>>,
    /// The transaction state.
    st: TransactionState,
}
```

#### TransactionState
在注释当中，对于TransactionState的设计理念做了比较详细的说明，简而言之就是，事务状态的设计使得事务可以在不同的组件之间安全地传递，并且可以在不直接引用事务本身的情况下被引用，有助于简化事务管理。

TransacationState当中提供了一个函数，用于判断给定的version对于当前事务是否可见，逻辑如下：
1. 如果version来自活跃事务，即处于active_set当中，那么代表为新写入的，不可见
2. 如果为只读事务，那么能看到小于version的(之前事务创建的)
3. 如果是普通事务，那么可以看到之前的和自身写入(<=)
>A Transaction's state, which determines its write version and isolation. It is separate from Transaction to allow it to be passed around independently of the engine. There are two main motivations for this: 
>- It can be exported via Transaction.state(), (de)serialized, and later used to instantiate a new functionally equivalent Transaction via Transaction::resume(). This allows passing the transaction between the  storage engine and SQL engine (potentially running on a different node)  across the Raft state machine boundary. 
>- It can be borrowed independently of Engine, allowing references to it  in VisibleIterator, which would otherwise result in self-references.

```rust
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct TransactionState {
    /// The version this transaction is running at. Only one read-write
    /// transaction can run at a given version, since this identifies its
    /// writes.
    pub version: Version,
    /// If true, the transaction is read only.
    pub read_only: bool,
    /// The set of concurrent active (uncommitted) transactions, as of the start
    /// of this transaction. Their writes should be invisible to this
    /// transaction even if they're writing at a lower version, since they're
    /// not committed yet.
    pub active: HashSet<Version>,
}

impl TransactionState {
    /// Checks whether the given version is visible to this transaction.
    ///
    /// Future versions, and versions belonging to active transactions as of
    /// the start of this transaction, are never isible.
    ///
    /// Read-write transactions see their own writes at their version.
    ///
    /// Read-only queries only see versions below the transaction's version,
    /// excluding the version itself. This is to ensure time-travel queries see
    /// a consistent version both before and after any active transaction at
    /// that version commits its writes. See the module documentation for
    /// details.
    fn is_visible(&self, version: Version) -> bool {
        if self.active.get(&version).is_some() {
            false
        } else if self.read_only {
            version < self.version
        } else {
            version <= self.version
        }
    }
}
```

#### MVCC
MVCC可以认为是一个`wrapper`，具体的逻辑是由上面的`Transaction`来实现的，调用`Transaction`当中对应的函数，在其中定义了一个存储引擎的shared_ptr，在调用时会传递给`Transaction`,为什么需要使用`Mutex`也在注释当中给出。
```rust
/// An MVCC-based transactional key-value engine. It wraps an underlying storage
/// engine that's used for raw key/value storage.
///
/// While it supports any number of concurrent transactions, individual read or
/// write operations are executed sequentially, serialized via a mutex. There
/// are two reasons for this: the storage engine itself is not thread-safe,
/// requiring serialized access, and the Raft state machine that manages the
/// MVCC engine applies commands one at a time from the Raft log, which will
/// serialize them anyway.
pub struct MVCC<E: Engine> {
    engine: Arc<Mutex<E>>,
}

impl<E: Engine> MVCC<E> {
    /// Creates a new MVCC engine with the given storage engine.
    pub fn new(engine: E) -> Self {
        Self { engine: Arc::new(Mutex::new(engine)) }
    }

    /// Begins a new read-write transaction.
    pub fn begin(&self) -> Result<Transaction<E>> {
        Transaction::begin(self.engine.clone())
    }

    /// Begins a new read-only transaction at the latest version.
    pub fn begin_read_only(&self) -> Result<Transaction<E>> {
        Transaction::begin_read_only(self.engine.clone(), None)
    }

    /// Begins a new read-only transaction as of the given version.
    pub fn begin_as_of(&self, version: Version) -> Result<Transaction<E>> {
        Transaction::begin_read_only(self.engine.clone(), Some(version))
    }

    /// Resumes a transaction from the given transaction state.
    pub fn resume(&self, state: TransactionState) -> Result<Transaction<E>> {
        Transaction::resume(self.engine.clone(), state)
    }

    /// Fetches the value of an unversioned key.
    pub fn get_unversioned(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        self.engine.lock()?.get(&Key::Unversioned(key.into()).encode()?)
    }

    /// Sets the value of an unversioned key.
    pub fn set_unversioned(&self, key: &[u8], value: Vec<u8>) -> Result<()> {
        self.engine.lock()?.set(&Key::Unversioned(key.into()).encode()?, value)
    }

    /// Returns the status of the MVCC and storage engines.
    pub fn status(&self) -> Result<Status> {
        let mut engine = self.engine.lock()?;
        let versions = match engine.get(&Key::NextVersion.encode()?)? {
            Some(ref v) => bincode::deserialize::<u64>(v)? - 1,
            None => 0,
        };
        let active_txns = engine.scan_prefix(&KeyPrefix::TxnActive.encode()?).count() as u64;
        Ok(Status { versions, active_txns, storage: engine.status()? })
    }
}
```

####  Status
不怎么重要，看一下就好，作为函数返回值来表示当前事务的状态，其中的storage是存储引擎的storage，在上一章介绍过了
```rust
/// MVCC engine status.
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct Status {
    /// The total number of MVCC versions (i.e. read-write transactions).
    pub versions: u64,
    /// Number of currently active transactions.
    pub active_txns: u64,
    /// The storage engine.
    pub storage: super::engine::Status,
}
```

#### Key
能够借助kv存储引擎实现MVCC的核心，实现上采用enum，rust当中的枚举非常强大，在 Rust 中，枚举是一种数据类型，它可以有不同的值（称为变体），但在任何给定时刻只能有其中一个值。每个枚举变体可以关联不同类型和数量的数据。

借助rust的enum，就可以在创建key时向其中传递一个值，既可以表示当前的动作或者状态，又可以获得这个动作需要的值。举个例子，在需要写入一个key的新的version时，那么就需要以当前的key + version来作为key，那么就可以很自然的使用enum来表示一个复合的key，之后encode进行存储：
```rust
    /// A versioned key/value pair.
    Version(
        #[serde(with = "serde_bytes")]
        #[serde(borrow)]
        Cow<'a, [u8]>,
        Version,
    ),
        // 某处调用
    session.set(&Key::Version(key.into(),self.st.version).encode()?, bincode::serialize(&value)?)

```
而对于next_version而言，并不需要额外的值，只需要key为next_version，而value为一个u64即可，那么就不定义附加的值：
```rust
    /// The next available version.
    NextVersion,
    // 某处调用
    let versions = match engine.get(&Key::NextVersion.encode()?)? {
            Some(ref v) => bincode::deserialize::<u64>(v)? - 1,
            None => 0,
        };
```

Key的完整定义如下：
```rust
/// MVCC keys, using the KeyCode encoding which preserves the ordering and
/// grouping of keys. Cow byte slices allow encoding borrowed values and
/// decoding into owned values.
#[derive(Debug, Deserialize, Serialize)]
pub enum Key<'a> {
    /// The next available version.
    NextVersion,
    /// Active (uncommitted) transactions by version.
    TxnActive(Version),
    /// A snapshot of the active set at each version. Only written for
    /// versions where the active set is non-empty (excluding itself).
    TxnActiveSnapshot(Version),
    /// Keeps track of all keys written to by an active transaction (identified
    /// by its version), in case it needs to roll back.
    TxnWrite(
        Version,
        #[serde(with = "serde_bytes")]
        #[serde(borrow)]
        Cow<'a, [u8]>,
    ),
    /// A versioned key/value pair.
    Version(
        #[serde(with = "serde_bytes")]
        #[serde(borrow)]
        Cow<'a, [u8]>,
        Version,
    ),
    /// Unversioned non-transactional key/value pairs. These exist separately
    /// from versioned keys, i.e. the unversioned key "foo" is entirely
    /// independent of the versioned key "foo@7". These are mostly used
    /// for metadata.
    Unversioned(
        #[serde(with = "serde_bytes")]
        #[serde(borrow)]
        Cow<'a, [u8]>,
    ),
}
```
此外，还有一个KeyPrefix用于进行前缀匹配，和Key差不多，这里就不做介绍了。

### MVCC Impl
终于，要开始分析MVCC的实现了，这一部分其实就是`Transaction`的impl,在这一部分，笔者会把重点放在MVCC与存储引擎的交互上，即如何使用KV存储引擎来实现MVCC。

像前面说的那样，toydb支持time travel的只读事务，因此在开启事务这块，提供了两个函数，分别对应read-write的事务和read-only的事务
#### Begin
begin用于开启一个read-write的事务，大致干了一下几件事：
1. 获取一个Version作为当前事务的tid,或者可以视为一个时间戳，之后+1写回，由于toydb并没有实现buffer pool + wal，因此这里的NextVersion是存储在存储引擎当中的，采用的是同步写入的方式(这里单指bitcask,使用memory的话就没有什么持久化可言了，不过在MVCC模块当中，只需要在意engine的trait，通常不太需要考虑底层)
2. 从存储引擎当中扫描，恢复出当前的active_set，这里active_set采用的是分布存储的，即没当开启一个事务后，就向存储引擎当中写入一条`Key::TxnActive`，带上自己的version，之后扫描出所有`Key::TxnActive`的key，恢复出active_set，个人认为这样设计的原因是bitcask本身是一个append-only的存储引擎，就算是将value设置为完整的active_set，那么每次写入也是追加写入，并且需要完整的写入整个active_set，写入量反而增大，不如采用分布存储，代价是读取时需要进行一个扫描，不过active_set只会在事务begin的时候进行读取，也无伤大雅
3. 根据active_set来生成一个snapshot，用于进行time-travel read，time travel需要做的是读取到给定时间戳(version)时数据库的完整状态，不能够简单的通过版本进行读取，因为有些key虽然是由给定version之前的事务写入的，但是事务未提交，那么该key就不可见，这也是snapshot存在的意义，用于恢复某一时间的事务隔离情况，判断数据的可见性
4. 将自己的version写入，补充active_set
```rust
    /// Begins a new transaction in read-write mode. This will allocate a new
    /// version that the transaction can write at, add it to the active set, and
    /// record its active snapshot for time-travel queries.
    fn begin(engine: Arc<Mutex<E>>) -> Result<Self> {
        let mut session = engine.lock()?;

        // Allocate a new version to write at.
        let version = match session.get(&Key::NextVersion.encode()?)? {
            Some(ref v) => bincode::deserialize(v)?,
            None => 1,
        };
        session.set(&Key::NextVersion.encode()?, bincode::serialize(&(version + 1))?)?;

        // Fetch the current set of active transactions, persist it for
        // time-travel queries if non-empty, then add this txn to it.
        let active = Self::scan_active(&mut session)?;
        if !active.is_empty() {
            session.set(&Key::TxnActiveSnapshot(version).encode()?, bincode::serialize(&active)?)?
        }
        session.set(&Key::TxnActive(version).encode()?, vec![])?;
        drop(session);

        Ok(Self { engine, st: TransactionState { version, read_only: false, active } })
    }
```

#### Begin_read_only
`begin_read_only`用于进行read only的事务，如果传入了一个version,那么就进行time-travel，否则根据数据库最新的状态进行读取：
1. 获取数据库最新的version，如果传入了as_of就替换成传入的version，用于进行time travel(传入的version是不能大于数据库最新的version的)
2. 之后如果是time travel，就根据version去读取snapshot，来恢复active_set，否则和begin一样，去扫描获取最新的active_set
```rust
    /// Begins a new read-only transaction. If version is given it will see the
    /// state as of the beginning of that version (ignoring writes at that
    /// version). In other words, it sees the same state as the read-write
    /// transaction at that version saw when it began.
    fn begin_read_only(engine: Arc<Mutex<E>>, as_of: Option<Version>) -> Result<Self> {
        let mut session = engine.lock()?;

        // Fetch the latest version.
        let mut version = match session.get(&Key::NextVersion.encode()?)? {
            Some(ref v) => bincode::deserialize(v)?,
            None => 1,
        };

        // If requested, create the transaction as of a past version, restoring
        // the active snapshot as of the beginning of that version. Otherwise,
        // use the latest version and get the current, real-time snapshot.
        let mut active = HashSet::new();
        if let Some(as_of) = as_of {
            if as_of >= version {
                return Err(Error::Value(format!("Version {} does not exist", as_of)));
            }
            version = as_of;
            if let Some(value) = session.get(&Key::TxnActiveSnapshot(version).encode()?)? {
                active = bincode::deserialize(&value)?;
            }
        } else {
            active = Self::scan_active(&mut session)?;
        }

        drop(session);

        Ok(Self { engine, st: TransactionState { version, read_only: true, active } })
    }

```

#### Write && Delete
由于MVCC的append-only的特性(没有gc)，对于Write和Delete进行统一封装，Delete视为写入一个tombstone(MVCC层面的tombstone，调用的存储引擎还是set接口，不会在存储引擎当中删除)，底层都是通过`write_version`来实现的:
```rust
    /// Deletes a key.
    pub fn delete(&self, key: &[u8]) -> Result<()> {
        self.write_version(key, None)
    }

    /// Sets a value for a key.
    pub fn set(&self, key: &[u8], value: Vec<u8>) -> Result<()> {
        self.write_version(key, Some(value))
    }
```

**write_version**

在write_version当中，首先进行写入冲突的检查，即检查是否有其他的事务正在对当前的key进行写入操作，实现方法也很简单，扫描(key,active.min)到(key,u64::MAX)范围内的key，对于扫描出来的key当中最新的版本：
- 如果对于当时事务可见，那么就证明为自身写入的，或者是比自己早并且已经提交的事务写入的，不存在冲突
- 如果对当前事务不可见，那么就是其他的活跃事务写入的，证明有其他事务在并发写入，存在冲突(只要version存在于active_set当中就是不可见的，否则再根据version大小去判断)
在判断没有冲突之后，分别写入一条`TrnWrite`和`Version`，`TxnWrite`用于标志当前的事务进行写入，用于进行回滚，而`Version`才是真正存储数据的key

不过有个问题是，为了实现MVCC，即便是当前要删除的key不存在，也会去写入一条version，给存储引擎带来额外的负担
```rust
    /// Writes a new version for a key at the transaction's version. None writes
    /// a deletion tombstone. If a write conflict is found (either a newer or
    /// uncommitted version), a serialization error is returned.  Replacing our
    /// own uncommitted write is fine.
    fn write_version(&self, key: &[u8], value: Option<Vec<u8>>) -> Result<()> {
        if self.st.read_only {
            return Err(Error::ReadOnly);
        }
        let mut session = self.engine.lock()?;

        // Check for write conflicts, i.e. if the latest key is invisible to us
        // (either a newer version, or an uncommitted version in our past). We
        // can only conflict with the latest key, since all transactions enforce
        // the same invariant.
        let from = Key::Version(
            key.into(),
            self.st.active.iter().min().copied().unwrap_or(self.st.version + 1),
        )
        .encode()?;
        let to = Key::Version(key.into(), u64::MAX).encode()?;
        if let Some((key, _)) = session.scan(from..=to).last().transpose()? {
            match Key::decode(&key)? {
                Key::Version(_, version) => {
                    if !self.st.is_visible(version) {
                        return Err(Error::Serialization);
                    }
                }
                key => return Err(Error::Internal(format!("Expected Key::Version got {:?}", key))),
            }
        }

        // Write the new version and its write record.
        //
        // NB: TxnWrite contains the provided user key, not the encoded engine
        // key, since we can construct the engine key using the version.
        session.set(&Key::TxnWrite(self.st.version, key.into()).encode()?, vec![])?;
        session
            .set(&Key::Version(key.into(), self.st.version).encode()?, bincode::serialize(&value)?)
    }
```

#### Get
Get的逻辑就很简单了，找到一个key的可能看见的版本(从0到自己写入的，version范围对应[0,self.version])，从新到旧遍历，找到第一个自己能够看见的版本，返回。要是全部遍历完都没有那就是不存在能够读取到的版本
```rust
    /// Fetches a key's value, or None if it does not exist.
    pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
        let mut session = self.engine.lock()?;
        let from = Key::Version(key.into(), 0).encode()?;
        let to = Key::Version(key.into(), self.st.version).encode()?;
        // 调用rev从新的key开始遍历
        let mut scan = session.scan(from..=to).rev();
        while let Some((key, value)) = scan.next().transpose()? {
            match Key::decode(&key)? {
                Key::Version(_, version) => {
                    if self.st.is_visible(version) {
                        return bincode::deserialize(&value);
                    }
                }
                key => return Err(Error::Internal(format!("Expected Key::Version got {:?}", key))),
            };
        }
        Ok(None)
    }
```

#### Commit && Rollback
对于read only事务，由于其对系统不会产生任何影响，也不会把自己添加到active_set当中，因此直接返回即可。

**commit**

commit需要做的有：
1. 扫描出来所有由当前事务写入的`Key::TxnWrite`，`TxnWrite`是用于事务回滚时撤销写入的，既然事务提交了就不需要了，所以全部删除
2. 将自己从active_set当中移除，删除掉`TxnActive(self.version)`即可，这也是active_set分布存储的好处，更改只需要操作一个key
```rust
    /// Commits the transaction, by removing it from the active set. This will
    /// immediately make its writes visible to subsequent transactions. Also
    /// removes its TxnWrite records, which are no longer needed.
    pub fn commit(self) -> Result<()> {
        if self.st.read_only {
            return Ok(());
        }
        let mut session = self.engine.lock()?;
        let remove = session
            .scan_prefix(&KeyPrefix::TxnWrite(self.st.version).encode()?)
            .map(|r| r.map(|(k, _)| k))
            .collect::<Result<Vec<_>>>()?;
        for key in remove {
            session.delete(&key)?
        }
        session.delete(&Key::TxnActive(self.st.version).encode()?)
    }
```

**rollback**

在事务执行过程中，无论是写入还是删除，每次都是写入一个version，同时写入一个`Key::TxnWrite`，用于标志事务对数据的更改，因此在回滚时，只需要当前事务之前写入的`Key::TxnWrite`全部读取出来，转换为`Key::Version`，然后删除这个key的对应version，就完成了undo的动作。之后再将自己从active_set当中移除即可。

还是要提一下，这里的删除是mvcc层面的删除，只会删除相同user_key相同version的Key，不会像bitcask那样写入一个tombstone前面所有的key都不可达，分析mvcc就要屏蔽掉底层的存储引擎，只将其视为一个简单的kv存储。
```rust
    /// Rolls back the transaction, by undoing all written versions and removing
    /// it from the active set. The active set snapshot is left behind, since
    /// this is needed for time travel queries at this version.
    pub fn rollback(self) -> Result<()> {
        if self.st.read_only {
            return Ok(());
        }
        let mut session = self.engine.lock()?;
        let mut rollback = Vec::new();
        let mut scan = session.scan_prefix(&KeyPrefix::TxnWrite(self.st.version).encode()?);
        while let Some((key, _)) = scan.next().transpose()? {
            match Key::decode(&key)? {
                Key::TxnWrite(_, key) => {
                    rollback.push(Key::Version(key, self.st.version).encode()?) // the version
                }
                key => return Err(Error::Internal(format!("Expected TxnWrite, got {:?}", key))),
            };
            rollback.push(key); // the TxnWrite record
        }
        drop(scan);
        for key in rollback.into_iter() {
            session.delete(&key)?;
        }
        session.delete(&Key::TxnActive(self.st.version).encode()?) // remove from active set
    }
```

#### 原子性变更
到这里，MVCC的实现方式基本上已经分析完了，现在来说一下Miniob当中遗留的一个问题，就是事务在提交或者回滚时作出的更改没有办法被一次性读到。

对于传统的2PL，通过加锁的方式，阻止其他事务进行读取，从而保证在释放锁时一次性的将更新暴露给其他的事务，而Miniob引入MVCC就是为了避免读写冲突，因此不会加锁，在写入的过程中，其他事务自然可以进行读取，从而读取到不应该存在的中间态。

再来说一下toydb是怎样解决这个问题的，toydb同样没有引入复杂的并发控制，对于W-W冲突，解决方案也是简单的进行重试。但是在可见性上，toydb引入了额外的限制，即如果对应的version存在于active_set当中，那么就是不可见的(对其他事务).

因此即便是将新的key非原子性的写入到存储引擎当中，只要不从active_set当中删除，那么就是不可见的。而更改active_set的这个动作是原子性的，因此就可以保证事务提交时作出的更改一次性的对外可见。

**可重复读**

事务在begin时，会从存储引擎当中读取并建立出active_set，并且之后在事务执行过程当中，都以这个active_set为准，因此即便是其他事务提交了，写入的记录对当前事务还是不可见的，就保证了可重复读的问题。

这里的设计还是非常巧妙的，用简单的方法解决了问题，active_set既保证了原子性的提交，同时提供了可重复读

## Summary
toydb借助一个kv存储实现了mvcc，对于MVCC而言笔者认为有几个关键的问题，toydb也分别给出了对应的解决方案：
- 隔离级别：在toydb当中提供了快照隔离，在这种隔离级别下，保证了可重复读，并且避免了幻读的问题，但是代价是对于W-W冲突，会产生比较频繁的重试问题
- 并发控制：这里的实现与Miniob一样，W-R冲突通过MVCC本身解决，W-W冲突采用了最简单的冲突重试
- 原子性提交：记录写入并不是原子性的，但是active_set的更新是原子性的，通过active_set的更新保证记录能够一次性对外全部暴露出来
- 垃圾回收：没有垃圾回收，这是 feature 而不是 bug:)，借助此特点，实现了任意时间的 time travel

toydb当中Key的设计也是MVCC的重要支持，通过复合类型Key的形式，toydb实现了类似leveldb当中(user_key,sequence)的形式，但是得益于rust当中枚举类型的强大，不仅可以携带上版本信息，并且能够表示key的不同意义，如`TxnWrite`,`TxnActive`等。有了复合类型Key，便可以很轻松的借助kv存储引擎来实现MVCC。

在整个MVCC模块当中，KV存储既用于存储实际的数据，即一个个version，同时还起到了log的作用，会记录Txn::Write用于进行事务回滚，并且还会存储NextVersion,active set这样的元数据 ，可谓是用处多多了，凡事涉及到存储或者持久化的内容，都丢进kv engine

toydb的MVCC实现的非常简洁，真正与MVCC相关的逻辑只有400余行，除了上面分析的，还有一个Iterator的实现，用于支持范围扫描和前缀扫描，逻辑并不是很复杂，笔者就不额外展开了，此外toydb当中也提供了较为完善的测试，通过测试也可以更好的理解MVCC，rust的调试也很方便，无需任何配置，读者可自行阅读调试。



