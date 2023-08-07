[paper](http://www.cs.umd.edu/~abadi/papers/vll-vldb13.pdf) #locking

### Introduction
The most common way to implement a lock manager is as a hash table that maps each lockable record’s primary key to a linked list of lock requests for that record. This list is typically preceded by a lock head that tracks the current lock state for that item. For thread safety, the lock head generally stores a mutex object, which is acquired before lock requests and releases to ensure that adding or removing elements from the linked list always occurs within a critical section. Every lock release also invokes a traversal of the linked list for the purpose of determining what lock request should inherit the lock next.

### Very lightweight locking
The VLL protocol is designed to be as general as possible, with specific optimizations for the following architectures:
* Multiple threads execute transactions on a single-server, shared memory system.
* Data is partitioned across processors (possibly spanning multiple independent servers). At each partition, a single thread executes transactions serially.
* Data is partitioned arbitrarily (e.g. across multiple machines in a cluster); within each partition, multiple worker threads operate on data.

#### VLL Algorithm
Each record value will be preceded by a pair of integers ($C{x}$, $C{s}$), where $C{x}$ is the count of pending exclusive locks and $C{s}$ is the count of pending shared locks.
In addition, a global queue of transaction requests (called $TxnQueue$) is kept at each partition, tracking all active transactions in the order in which they requested their locks. If a transaction successfully got its locks then it will be $FREE$ state, otherwise it will be $BLOCKED$

Read the paper for the explanation of the algorithm, it is not difficult to grasp.
