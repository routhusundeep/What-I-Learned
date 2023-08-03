[paper](http://www.cs.umd.edu/~abadi/papers/determinism-vldb10.pdf)
#relational_database 

Very fascinating read. 

The agnosticism of serialization guarantees to which serial order is emulated generally means that this order is never determined in advance; rather it is dependant on a vast array of factors entirely orthogonal to the order in which transactions may have entered the system, including:
* thread and process scheduling
* buffer and cache management
* hardware failures
* variable network latency
* deadlock resolution schemes
Nondeterministic behaviour in database systems causes complications in the implementation of database replication and horizontally scalable distributed databases.

#### Replication
Three properties are desirable in a replication scheme: 
* consistency across replicas 
* currentness of all replicas
* low overhead 
Commonly used replication schemes generally fall into one of three families, each with its own subtleties, variations, and costs.
* Post-write replication
This is typically implemented via log shipping the master sends out the transaction log to be replayed at each replica
* Active replication with synchronized locking
The disadvantage of this scheme is the additional latency due to the network communication for the lock synchronization.
* Replication with lazy synchronization
Multiple active replicas execute transactions independently— possibly diverging temporarily—and reconcile their states at a later time. Lazy synchronization schemes enjoy good performance at the cost of consistency.

#### Horizontal scalability
In order to satisfy ACID’s guarantees, distributed transactions generally use a distributed commit protocol, typically two-phase commit.
Deterministic database systems that use active replication only need to use a one-phase commit protocol. This is because replicas are executing transactions in parallel, so the failure of one replica does not affect the final commit/abort decision of a transaction

#### Contribution
In this paper, we present a transactional database execution model with the property that any transaction Tn’s outcome is uniquely determined by the database’s initial state and a universally ordered series of previous transactions {T0, T1, ..., Tn−1}

### Maintaining Equivalence to a Predetermined Serial Order
Predetermined serial order equivalence (as well as deadlock freedom) can be achieved by restricting the set of valid executions to only those satisfying the following properties:
* **Ordered locking:**:For any pair of transactions Ti and Tj which both request locks on some record r, if i < j then Ti must request its lock on r before Tj does. Further, the lock manager must grant locks strictly in the order that they were requested.
* **Execution to completion:** Every transaction that enters the system must go on to run to completion— either until it commits or until it aborts due to deterministic program logic. Therefore, if a transaction is delayed in completing for any reason (e.g. due to a hardware failure within a replica), that replica must keep that transaction active until the transaction executes to completion or the replica is killed—even if other transactions are waiting on locks held by the blocked one.

### The Case of Non Determinism
Suppose a transaction enters the system, acquires some locks, and becomes blocked for some reason (examples of this include deadlock, access to slow storage, or, if our hypothetical database spans multiple machines, a critical network packet being dropped). Whatever the cause of the delay, this is a situation where a little bit of flexibility will prove highly profitable. In the case of deadlock or hardware failure, it might be prudent to abort the transaction and reschedule it later. In other cases, there are many scenarios in which resource allocation can be vastly improved by shuffling the serial order to which our (non-serial) execution promises equivalence.

Read the paper for more lucid quantitative arguments.

### A Prototype
Read the paper


