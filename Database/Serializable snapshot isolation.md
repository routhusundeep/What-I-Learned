
### Serializable snapshot isolation 
[paper](https://drkp.net/papers/ssi-vldb12.pdf)
#isolation

- Talks about simple write skews and another 3-Transaction example which are not serializable with SI
- How Seriazability is achieved till now
  - some workloads do not experience Serialization Anomalies like TPC-C
  - Explicit locking can be used like SQL's "LOCK TABLE" and "SELECT FOR UPDATE".
  - conflict can be materialized by creating a dummy row or table and forcing transactions to act on them.
  - The desired result can be achieved through integrity constraints provided by the database like Foreign Key or incremental keys.
  - Static tools can be used on the queries to determine the conflicts, which is not possible for most of the workloads
- Harder to implement it for PG since the necessary locking tools are not present, other databases have SS2PL, so the interfaces could be reused.
- xx-dependency
  - ww, wr and rw(anti)
  - rw-antidependency are the important ones to here, due to their unique property in determining the serializability
    - definition : if T1 writes a version of some object,and T2 reads the previous version of that object, then T1 appears to have executed after T2, because T2 did not see its update
- Every cycle in the serialization history graph contains a sequence of edges T1 -rw-> T2 -rw-> T3 where each edge is an rw antidependency. Furthermore, T3 must be the first transaction to commit.
  - This property is used to implement SSI. The pattern is defined as "Dangerous structure".
  - whenever such T1, T2 and T3 are found, the transactions are aborted to prevent anomaly.
  - It is important to note that the cycle itself may not be present, so there are chances of false positives.
- Optimizations
  - If T1 is read-only then T3 should be committed before T1 has taken a snapshot, otherwise it is safe to continue.
  - A read-only transaction running on Safe snapshot can read data without risk of Serialization failure.
    - definition: A read-only transaction has a safe snapshot if no concurrent r/w transaction has commmitted with a rw-conflict out to a transaction that committed before T's snapshot, or has probability to do so.
  - Deferrable transactions, long running transactions can wait to become safe before proceeding.
- Implementation details
  - Introduces SIREAD lock and SI lock manager
  - wr-conflicts can be detected doing visibility checks on the tuple header which PG stores in the block.
  - rw-conflicts will be detected by SIREAD locks.
    - All pointers to other transactions involved in the rw-conflict are stored, ww and wr are discarded.
    - The decision was made after a lot of consideration, like memory usage.
- Safe Retry
  - If a transaction is aborted, then immediately re-trying the same transaction will not cause it to fail with the same transaction error.
  - Do not abort anything until T3 commits
  - Always choose to abort T2
  - If T2 and T3 and committed, it is safe to abort T1
- Memory usage considerations
  - Safe snapshot and Deferrable Transactions
  - granularity promotion of locks
  - Aggressive cleanup of committed transactions, note that SIREAD are required even after transactions are finished.
  - Summarization of committed transactions at the expence of increased false positive rates.
    


