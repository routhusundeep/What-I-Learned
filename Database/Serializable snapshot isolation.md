
### Serializable snapshot isolation 
[paper](https://arxiv.org/pdf/1208.4179.pdf)
#isolation #serializability

#### Snapshot Isolation
All reads within a transaction see a consistent view of the database, as though the transaction operates on a private snapshot of the database taken before its first read.
Snapshot isolation does not allow the three anomalies defined in the ANSI SQL standard: dirty reads, non-repeatable reads, and phantom reads.
However, it allows several other anomalies.
- **Write Skew**
	- e.g. if two doctors are on-call, both try to remove themselves from on-call only if there is some other doctor on-call, then both will be removed resulting in no one being on-call
- **Batch Processing**
	- See the paper for example

#### Why Serializability
SI is widely used as:
- Some workloads do not experience Serialization Anomalies like TPC-C
- Explict locking can be used to avoid anamoly
- Conflict can be materialized by creating a dummy row or table and forcing transactions to act on them.
- The desired result can be achieved through integrity constraints provided by the database like Foreign Key or incremental keys.
- Static tools can be used on the queries to determine the conflicts, which is not possible for most of the workloads such as ad-hoc queries

#### Serializable Snapshot Isolation
- Harder to implement it for PG since the necessary locking tools are not present, other databases have strict two-phase locking (SS2PL), so the interfaces could be reused.
	- In S2PL, transactions acquire locks on all objects they read or write, and hold those locks until the transaction commits. To prevent phantoms, these locks must be predicate locks, usually implemented using index-range locks.
- Decided to not use S2PL due to performance considerations and the distinct design principles

##### SI Anamolies
- xx-dependency
  - ww, wr and rw(anti)
  - rw-antidependency are the important ones to here, due to their unique property in determining the serializability
    - definition : if T1 writes a version of some object, and T2 reads the previous version of that object, then T1 appears to have executed after T2, because T2 did not see its update
- If a cycle is present in the graph, then the execution does not correspond to any serial order

##### Serializability Theory
**Theorem 1:** Every cycle in the serialization history graph contains a sequence of edges `T1 --rw--> T2 --rw--> T3` where each edge is a rw antidependency. Furthermore, T3 must be the first transaction to commit.

##### SSI
- The above property is used to implement SSI. The pattern is defined as "Dangerous structure".
- If any transaction has both an incoming rw-antidependency and an outgoing one, SSI aborts one of the transactions involved.
- It is important to note that the cycle itself may not be present, so there are chances of false positives.
- The benefit is that it is more efficient. Besides being a less expensive runtime check than cycle testing, dangerous structures are composed entirely of rw-antidependencies, so SSI does not need to track wr- and ww-dependency edges
- This approach can offer greater concurrency than a typical S2PL or optimistic concurrency control (OCC) system
- PSSI (Precisely Serializable Snapshot Isolation) is an extension of SSI that does eliminate all false positives, but rejected it because we felt the costs outweighed the benefits of the reduced false positive abort rate.

#### Read-only Optimization
**Theorem 3.** Every serialization anomaly contains a dangerous structure T1 rw−→ T2 rw−→ T3, where if T1 is read-only, T3 must have committed before T1 took its snapshot.
- This result can be applied directly to reduce the false positive rate, using the following read-only snapshot ordering rule: if a dangerous structure is detected where T1 is read-only, it can be disregarded as a false positive unless T3 committed before T1’s snapshot.

##### Safe Snapshots
- A read-only transaction T has a safe snapshot if no concurrent read/write transaction has committed with a rw-antidependency out to a transaction that committed before T’s snapshot, or has the possibility to do so.
- A read-only transaction running on a safe snapshot can read any data (perform any query) without risk of serialization failure. It cannot be aborted, and does not need to take SIREAD locks.

##### Deferrable Transactions
- Deferrable transactions, a new feature, provide a way to ensure that complex read-only transactions will always run on a safe snapshot.
- long running transactions can wait to become safe before proceeding.
- When a deferrable transaction begins, our system acquires a snapshot, but blocks the transaction from executing. It must wait for concurrent read/write transactions to finish. If any commit with a rw-conflict out to a transaction that committed before the snapshot, the snapshot is deemed unsafe, and we retry with a new snapshot.

### Implementing SSI in Postgresql
- PostgreSQL’s SSI implementation uses existing MVCC data as well as a new lock manager to detect conflicts
	- If the write happens first, then the conflict can be inferred from the MVCC data, without using locks.Whenever a transaction reads a tuple, it performs a visibility check, inspecting the tuple’s xmin and xmax to determine whether the tuple is visible in the transaction’s snapshot. 
		- If the tuple is not visible because the transaction that created it had not committed when the reader took its snapshot, that indicates a rw-conflict: the reader must appear before the writer in the serial order
		- Similarly, if the tuple has been deleted – i.e. it has an xmax – but is still visible to the reader because the deleting transaction had not committed when the reader took its snapshot, that is also a rw-conflict that places the reader before the deleting transaction in the serial order
	- We also need to handle the case where the read happens before the write. This cannot be done using MVCC data alone; it requires tracking read dependencies using SIREAD locks.
		- The SSI lock manager stores only SIREAD locks. It does not support any other lock modes, and hence cannot block. The two main operations it supports are to obtain a SIREAD lock on a relation, page, or tuple, and to check for conflicting SIREAD locks when writing a tuple.

#### Implementation of the SSI Lock Manager
- Uses S2PL
- handles predicate reads using index range locks
- Reads acquire SIREAD locks on all tuples they access, and index access methods acquire SIREAD locks on the “gaps” to detect phantoms.
- SIREAD locks cannot cause blocking. Deadlock detection is unecessary
- Must handle additional situations
	- SIREAD locks must be kept up to date when concurrent transactions modify the schema with data-definition language (DDL) statements
		- rewrite a table, such as RECLUSTER or ALTER TABLE, cause the physical location of tuples to change. As a result, SIREAD locks, are no longer valid; PostgreSQL therefore promotes them to relation-granularity.
	- If Index is removed
		- Index-gap locks are promoted to relation level locks

#### Tracking Conflicts
- Approaches
	- 2 bits per transaction or a single pointer can be used
	- PSSI stored the entire graph, including wr and ww dependencies
		- This is to support cycle testing
- But Postgres only keeps a list of all rw-antidependencies in or out for each transaction, but not wr- and ww-dependencies.
	- Keeping pointers to the other transaction involved in the rw-antidependency, rather than a simple flag, is necessary to implement the commit ordering optimization and read-only optimizations
- also implemented a number of techniques to aggressively discard information about committed transactions to conserve memory

#### Resolving Conflicts: Safe Retry
When a dangerous structure is found, and the commit ordering conditions are satisfied, some transaction must be aborted to prevent a possible serializability violation.
**Safe retry:** if a transaction is aborted, immediately retrying the same transaction will not cause it to fail again with the same serialization failure.

Once a dangenrous structure `T1 --rw--> T2 --rw--> T3` is identified. Specifically, the following rules are used to ensure safe retry
1. Do not abort anything until T3 commits.
	- Dangerous structures may not be resolved immediately when they are detected. As a result, we also perform a check when a transaction commits. If T3 attempts to commit while part of a dangerous structure of uncommitted transactions, it is the first to commit and an abort is necessary. This should be resolved by aborting T2,
2. Always choose to abort T2 if possible. T2 must have been concurrent with both T1 and T3. Because T3 is already committed, the retried T2 will not be concurrent with it and so will not be able to have a rw-conflict out to it, preventing the same error from recurring.
3. If both T2 and T3 have committed when the dangerous structure is detected, then the only option is to abort T1. But this is safe

### Memory Usage Mitigations
Requirements:
- Bounded memory usage
- Graceful degradation

Uses four techniques to limit the memory usage of the SSI lock manager: 
1. Safe snapshots and deferrable transactions can reduce the impact of long-running read-only transactions 
2. Granularity promotion: multiple fine-grained locks can be combined into a single coarse-grained lock to reduce space. 
3. Aggressive cleanup of committed transactions: the parts of a transaction’s state that are no longer needed after commit are removed immediately 
4. Summarization of committed transactions: if necessary, the state of multiple committed transactions can be consolidated into a more compact representation, at the cost of an increased false positive rate

Read the paper for more information.