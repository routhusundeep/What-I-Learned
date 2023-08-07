[paper](https://www.vldb.org/pvldb/vol7/p181-bailis.pdf) #datastore #availability

## Key Takeaways
  - The default configuration of most of the popular database systems expose a range of anomalies which may not be what the application desire
  - Many of the "weak isolation" models can be achieved without sacrificing high availability if implemented correctly. However, none of the achievable models provide concurrent modifications
  - In addition to providing guaranteed response and horizontal scale out, these HAT models allow one to three order of magnitude lower latencies on current infrastructure
  - For correct behaviours, applications may require combined usage of HAT and non-HAT(sparingly), database designers should take this into consideration

## Why High Availability
Network Partitions are very common at scale, Read the paper to get examples and statistics
In a distributed setting, network failures may prevent database servers from communicating, and, in the absence of failures, communication is slowed by factors like physical distance, network congestion, and routing. Highly available system designs mitigate the effects of network partitions and latency

## ACID in the Wild
The below table summarises only for a few databases, read the paper to find more

> RC - read committed
> RR - repeatable read
> SI - snapshot isolation
> S - serializability
> CS - cursor stability
> CR - consistent reads

| Database           | Default | Maximum |
|--------------------|---------|---------|
| Aerospike          | RC      | RC      |
| IBM DB2 for z/OS   | CS      | S       |
| MySQL 5.6          | RR      | S       |
| MS SQL Sever 2012  | RC      | S       |
| Oracle 11g         | RC      | SI      |
| Oracle Berkeley DB | S       | S       |
| Postgres 9.2.2     | RC      | S       |
| SAP HANA           | RC      | SI      |
| VoltDB             | S       | S       |

## High Availability
Informally, highly available algorithms ensure “always on” operation and, as a side effect, guarantee low latency
If servers are partitioned from one another, they do not need to stall in order to provide clients a “safe” response to operations. The lack of fast path coordination results in the low latency.
Traditionally, a system provides high availability if every user that can contact a correct(non-failing) server eventually receives a response from that server even in the presence of arbitrary, indefinitely long network partitions between the servers of the system.

### Sticky Availability
Clients can contact the same set of logical replica(s) across subsequent operations, this is called stickiness
We say that a system provides sticky availability if, whenever a client's transaction is executed across a copy of database state that reflects all the client's previous operations, it eventually gives a response even in the presence of network partitions.
Any guarantee achievable in a highly available system is achievable in a sticky high availability system but not vice-versa.

### Transactional Availability
We say a transaction has replica availability, if it can contact at least one replica for every item it attempts to access, this may result in “lower availability” than a non-transactional availability requirement (e.g., single-item availability).
Additionally, given the possibility of system-initiated aborts, we need to ensure useful forward progress: a system can trivially guarantee clients a response by always aborting all transactions. However, this is an unsatisfactory system because nothing good (transaction commit) ever happens; we should require a liveness property
We say that a system provides transactional availability if, given replica availability for every data item in a transaction, the transaction eventually commits (possibly after multiple client retries) or internally aborts. A system provides sticky transactional availability if, given sticky availability, a transaction eventually commits or internally aborts

### Highly Available Transactions
HAT systems provide transactions with transactional availability or sticky transactional availability. They offer latency and availability benefits over traditional distributed databases, yet they cannot achieve all possible semantics

We describe ACID, distributed replica consistency, and session consistency levels which can be achieved with high availability (Read Committed isolation, variants of Repeatable Read, atomic reads, and many session guarantees), those with sticky availability (read your writes, PRAM and causal consistency). We also discuss properties that cannot be provided in a HAT system (those preventing Lost Update and Write Skew or guaranteeing recency).

In our examples, we exclusively consider read and write operations, denoting a write of value $v$ to data item $d$ as $w_d(v)$ and a read from data item d returning $v$ as $r_d(v)$. We assume that all data items have the null value $\bot$, at database initialization, and, unless otherwise specified, all transactions in the examples commit.

### Achievable HAT semantic

#### Read Uncommitted
Writes are ordered, but reads are not.
Consider the following example: 

> T1 ∶ $w_x(1)\; w_y(1)$ 
> T2 ∶ $w_x(2)\;w_y(2)$ 

In this example, under Read Uncommitted, it is impossible for the database to order T1’s $w_x(1)$ before T2’s $w_x(2)$ but order T2’s $w_y(2)$ before T1’s $w_y(1)$.
Read Uncommitted is easily achieved by marking each of a transaction’s writes with the same timestamp (unique across transactions; e.g., combining a client’s ID with a sequence number) and applying a “last writer wins” conflict reconciliation policy at each replica.

#### Read Committed
It is particularly important in practice as it is the default isolation level of many DBMSs.
Prohibit [dirty writes and dirty reads](https://en.wikipedia.org/wiki/Write%E2%80%93read_conflict).
For instance, in the example below, T3 should never see a = 1, and, if T2 aborts, T3 should not read a = 3: 
> T1 ∶ $w_x(1)\;w_x(2)$
> T2 ∶ $w_x(3)$
> T3 ∶ $r_x(a)$

Normally provides recency and monotonic properties also


#### Repeatable Reads
if a transaction reads the same data more than once, it sees the same value each time (preventing “Fuzzy Read”).
Will be called "Cut Isolation", "Item Cut Isolation" and "Predicate Cut Isolation"
In the example below, under both levels of cut isolation, T3 must read a = 1: 
> T1 ∶ $w_x(1)$
> T2 ∶ $w_x(2)$
> T3 ∶ $r_x(1)\;r_x(a)$

#### ACID atomicity guarantees
Atomicity, informally guaranteeing that either all or none of transactions’ effects should succeed, is core to ACID guarantees. Although, at least by the ACID acronym, atomicity is not an “isolation” property, atomicity properties also restrict the updates visible to other transactions.
We consider the isolation effects of atomicity, which we call Monotonic Atomic View (MAV) isolation.
Under MAV, once some of the effects of a transaction $T_i$ are observed by another transaction $T_j$, thereafter, all effects of $T_i$ are observed by $T_j$. That is, if a transaction $T_j$ reads a version of an object that transaction $T_i$ wrote, then a later read by $T_j$ cannot return a value whose later version is installed by $T_i$.
Together with item cut isolation, MAV prevents[ Read Skew anomalies](https://vladmihalcea.com/a-beginners-guide-to-read-and-write-skew-phenomena/)
In the example below, under MAV, because T2 has read T1’s write to y, T2 must observe b = c = 1 (or later versions for each key): 
> T1 ∶ $w_x(1)\;w_y(1)\;w_z(1)$ 
> T2 ∶ $r_x(a)\;r_y(1)\;r_x(b)\;r_z(c)$ 

T2 can also observe a = $\bot$, a = 1, or a later version of x

#### Session guarantees
An abstraction of a sequence of operations performed during the execution of the application.
For us, a session describes a context that should persist between transactions: for example, on a social networking site, all of a user’s transactions submitted between “log in” and “log out” operations might form a session.

**Monotonic reads** require that, within a session, subsequent reads to a given object "never return any previous values"
**Monotonic writes** require that, writes become visible in the order they are submitted
**Write Follows Reads**, the "happens before" relationship between transactions

Without stickiness, write may be done on an out of date replica
With stickiness, we will have three more guarantees
- Read your writes
- PRAM((Pipelined Random Access Memory), serialization all operation(both reads and writes) within the session, combination of monotonic reads, monotonic writes and read your writes
- Causal Consistency, PRAM with write-follow-reads

#### Addition Guarantees
**Consistency** A HAT system can make limited application-level consistency guarantees. It can often execute commutative and logically monotonic operations without the risk of invalidating application-level integrity constraints and can maintain limited criteria like foreign key constraints (via MAV)
**Convergence** Under arbitrary (but not infinite delays), HAT systems can ensure convergence, or eventual consistency: in the absence of new mutations to a data item, all servers should eventually agree on the value for each item

### Unachievable HAT semantics

#### Unachievable ACID Isolation
##### Lost Update 
as when one transaction T1 reads a given data item, a second transaction T2 updates the same data item, then T1 modifies the data item based on its original read of the data item, “missing” or “losing” T2’s newer update. 
Consider a database containing only the following transactions: 
> T1 ∶ $r_x(a)\;w_x(a + 2)$
> T2 ∶ $w_x(2)$

If T1 reads a = 1 but T2’s write to x precedes T1’s write operation, then the database will end up with a = 3, a state that could not have resulted in a serial execution due to T2’s “Lost Update.”

##### Write Skew
It occurs when one transaction T1 reads a given data item x, a second transaction T2 reads a different data item y, then T1 writes to y and commits and T2 writes to x and commits. As an example of Write Skew, consider the following two transactions: 
> T1 ∶ $r_y(0) w_x(1)$
> T2 ∶ $r_x(0) w_y(1)$

If there was an integrity constraint between x and y such that only one of x or y should have value 1 at any given time, then this write skew would violate the constraint

Consistent Read, Snapshot Isolation, Cursor Stability are not achievable because they prevent Lost Update

#### Unachievable Recency Guarantees
Distributed data storage systems often make various recency guarantees on reads of data items. Unfortunately, an indefinitely long partition can force an available system to violate any recency bound, so recency bounds are not enforceable by HAT systems. One of the most famous of these guarantees is linearizability, which states that reads will return the last completed write to a data item, and there are several other (weaker) variants such as safe and regular register semantics. When applied to transactional semantics, the combination of one-copy serializability and linearizability is called strong (or strict) one-copy serializability (e.g., Spanner). It is also common, particularly in systems that allow reading from masters and slaves, to provide a guarantee such as “read a version that is no more than five seconds out of date” or similar. None of these guarantees are HAT-compliant.

## Summary
|Availability|Models|
|--|--|
|HA|HA Read Uncommitted (RU), Read Committed (RC), Monotonic Atomic View (MAV), Item Cut Isolation (I-CI), Predicate Cut Isolation (PCI), Writes Follow Reads (WFR), Monotonic Reads (MR), Monotonic Writes (MW)|
|Sticky| Read Your Writes (RYW), PRAM, Causal|
|Unavailable| Cursor Stability (CS), Snapshot Isolation (SI), Repeatable Read (RR), One-Copy Serializability (1SR), Recency, Safe, Regular Linearizability, Strong 1SR|

