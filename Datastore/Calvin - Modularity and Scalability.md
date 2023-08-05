[paper](http://sites.computer.org/debull/A13june/calvin1.pdf)

This is an extension to [[Calvin]], read it if you want to learn actual internal details of the implementation of Calvin.

Calvin's architecture provides several advantages over traditional (nondeterministic) database system designs: 
* **Near-linear scalability:** Since all distributed agreement (both between replicas and between partitions within a replica) needed to commit a transaction occurs before Calvin executes the transaction, no distributed commit protocols are required, ameliorating the primary barrier to scaling ACID systems.
* **Strongly consistent replication:** Multiple replicas of a Calvin deployment remain perfectly consistent as long as they use the same operation request log to drive transaction execution. Once a transaction is written to the log, no further inter-replica coordination is required to synchronize replicas’ states. This allows strongly consistent replication—even across geographically separated data centers—at no cost to transaction throughput.
* **Highly modular system architecture:** By explicitly describing the emulated serial execution in the form of the operation request log, Calvin provides other system components with a declarative specification of the concurrency control mechanisms’ behaviour. Components that traditionally interact closely with the transaction scheduling system in standard nondeterministic database systems (e.g. logging, replication, storage) are thus completely decoupled from Calvin’s concurrency control implementation. Standard interfaces for each Calvin component make it easy to swap in different implementations of each component to quickly prototype systems with different properties.
* **Main-memory OLTP performance over disk-resident data:** When Calvin begins processing a transaction request by adding it to the log, it may also perform a cursory analysis of the transaction request and send a prefetch “hint” to each storage backend component that the transaction will access once it begins executing. The storage engine can therefore begin bringing the transaction’s read- and write-sets into memory even before the request is appended to the log, so that all relevant data will be memory-resident once the transaction begins executing.


### Calvin’s Modular Implementation
#### Log
* **Local log:** The simplest possible implementation of a log component appends new transaction requests to a logfile on the local file system, to which other components have read-only access. This implementation is neither fault-tolerant nor scalable, but it works well for single-server storage system deployments.
* **Sharded, Paxos-replicated log:** At the opposite extreme from the local log is a scalable, fault-tolerant log implementation that we designed for extremely high throughput and strongly consistent WAN replication. Here, a group of front-end servers collect client requests into batches. Each batch is assigned a globally unique ID and written to an independent, asynchronously replicated block storage service such as Voldemort or Cassandra. Once a batch of transaction requests is durable on enough replicas, its GUID is appended to a Paxos “MetaLog”. To achieve essentially unbounded throughput scalability, Paxos proposers may also group many request batch GUIDs into each proposal. Readers can then reassemble the global transaction request log by concatenating batches in the order that their GUIDs appear in the Paxos MetaLog.

##### Log-forwarding
Distributed log implementations may 
* forward the read- and write-sets
* analyze a committed transaction request

##### Recovery Manager
Calvin leverages its deterministic execution invariant to replace physical logging with logical logging. Recovery is performed by loading database state from a recent checkpoint, then deterministically replaying all transactions in the log after this point, which will bring the recovering machine to the same state as any noncrashed replica

#### Separating Transaction Scheduling from Data Storage
Since the storage manager does not make any direct calls to the concurrency control manager, it becomes straightforward to plug any data store implementation into Calvin, from simple key-value stores to more sophisticated NoSQL stores such as LevelDB, to full relational stores.
This is done by building a wrapper around every storage backend that implements two methods: 
* **$RUN(action)$** receives a command in a language that the data store understands (e.g. $get(k)$ or $put(k,v)$ for a key-value store, or a SQL command for a relational store) and executes it on the underlying data store.
* $ANALYZE(action)$ takes the same command, but instead of executing it, returns its read- and write sets—collections of logical objects that will be read or created/updated when RUN is called on the command

Just as the storage backend is pluggable in Calvin, so is the transaction scheduler. Our prototype currently supports three different schedulers that use transactions’ read-/write-sets in different ways:
* **Serial execution a la H-Store/VoltDB:** This scheduler simply ignores the read and write set information about each transaction and never allows two transactions to be sent to the storage backend at the same time.
* **Deterministic locking:** This scheduler uses a standard lock manager to lock the logical entities that are received from calling the ANALYZE method. If the lock manager determines that the logical entities of two transactions do not conflict, they can by executed by the storage backend concurrently
* **Very lightweight locking (VLL):** VLL is similar to the deterministic locking scheduler described above, in that it requests locks in the order of the transactional log, and is therefore deterministic and deadlock free. However, it uses per-data-item semaphores instead of a hash table in order to implement the lock manager.

### Transactional Interface
Clients of a Calvin system deployment may invoke transactions using three distinct mechanisms:
* Built-in Stored Procedures
* Lua Snippet Transactions
* Optimistic Client-side Transactions

