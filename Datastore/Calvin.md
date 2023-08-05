[paper](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)

Wow, This one took me some time to complete!


Calvin is a practical transaction scheduling and data replication layer that uses a deterministic ordering guarantee to significantly reduce the normally prohibitive contention costs associated with distributed transactions.

### Background
* Calvin is designed to run alongside a non-transactional storage system, transforming it into a shared-nothing (near-)linearly scalable database system that provides high availability and full ACID transactions. 
* These transactions can potentially span multiple partitions spread across the shared-nothing cluster. Calvin accomplishes this by providing a layer above the storage system that handles the scheduling of distributed transactions, as well as replication and network communication in the system. 

> The key technical feature that allows for scalability in the face of distributed transactions is a deterministic locking mechanism that enables the elimination of distributed commit protocols.

#### The cost of distributed transactions
2PC is very costly. The problem with holding locks during the agreement protocol is that 2PC requires multiple network round-trips between all participating machines, and therefore the time required to run the protocol can often be considerably greater than the time required to execute all local transaction logic

#### Consistent replication
Distributed database system design has been towards reduced consistency guarantees with respect to replication.
Synchronous updates come with a latency cost fundamental to the agreement protocol, which is dependent on network latency between replicas. This cost can be significant, since replicas are often geographically separated to reduce correlated failures. However, this is intrinsically a latency cost only, and need not necessarily affect contention or throughput.

#### Achieving agreement without increasing contention
Calvin decided the ordering outside transactional boundaries—that is, before they acquire locks and begin executing the transaction.

This paper’s primary contributions are the following:
* The design of a transaction scheduling and data replication layer that transforms a non-transactional storage system into a (near-)linearly scalable shared-nothing database system that provides high availability, strong consistency, and full ACID transactions.
* A practical implementation of a deterministic concurrency control protocol that is more scalable than previous approaches, and does not introduce a potential single point of failure.
* A data prefetching mechanism that leverages the planning phase performed prior to transaction execution to allow transactions to operate on disk-resident data without extending transactions’ contention footprints for the full duration of disk lookups.
* A fast checkpointing scheme that, together with Calvin’s determinism guarantee, completely removes the need for physical REDO logging and its associated overhead.

### Deterministic Database Systems
In traditional distributed transaction, either all nodes involved in the transaction agree to “commit” their local changes or none of them do. Events that prevent a node from committing its local changes fall into two categories: 
* nondeterministic events (such as node failures) and 
* deterministic events (such as transaction logic that forces an abort).
Calvin maintains multiple replicas, and they do not diverge since they perform transactions in a predetermined order. So, if one of them fails, the other replica can still serve the requests. This way, nondeterministic events cannot halt the transaction.
For deterministic events, each node involved in a transaction waits for a one-way message from each node that could potentially deterministically abort the transaction, and only commits once it receives these messages.

### System Architecture
![[Screenshot 2023-08-05 at 4.51.40 PM.png]]

#### Sequencer and replication
Calvin divides time into 10-millisecond epochs during which every machine’s sequencer component collects transaction requests from clients. At the end of each epoch, all requests that have arrived at a sequencer node are compiled into a batch. This is the point at which replication of transactional inputs occurs.
After a sequencer’s batch is successfully replicated, it sends a message to the scheduler on every partition within its replica containing (1) the sequencer’s unique node ID, (2) the epoch number (which is synchronously incremented across the entire system once every 10 ms), and (3) all transaction inputs collected that the recipient will need to participate in.

#### Synchronous and asynchronous replication
Calvin currently supports two modes for replicating transactional input: asynchronous replication and Paxos-based synchronous replication

#### Scheduler and concurrency control
Calvin’s deterministic lock manager is partitioned across the entire scheduling layer, and each node’s scheduler is only responsible for locking records that are stored at that node’s storage component— even for transactions that access records stored on other nodes. The locking protocol resembles strict two-phase locking, but with two added invariants:
* For any pair of transactions A and B that both request exclusive locks on some local record R, if transaction A appears before B in the serial order provided by the sequencing layer then A must request its lock on R before B does. In practice, Calvin implements this by serializing all lock requests in a single thread. The thread scans the serial transaction order sent by the sequencing layer; for each entry, it requests all locks that the transaction will need in its lifetime.
* The lock manager must grant each lock to requesting transactions strictly in the order in which those transactions requested the lock. So in the above example, B could not be granted its lock on R until after A has acquired the lock on R, executed to completion, and released the lock.

Once a transaction has acquired all of its locks under this protocol (and can therefore be safely executed in its entirety) it is handed off to a worker thread to be executed. Each actual transaction execution by a worker thread proceeds in five phases:
1. **Read/write set analysis:** The first thing a transaction execution thread does when handed a transaction request is analyzed the transaction’s read and write sets, noting (a) the elements of the read and write sets that are stored locally (i.e. at the node on which the thread is executing), and (b) the set of participating nodes at which elements of the write set are stored. These nodes are called active participants in the transaction; participating nodes at which only elements of the read set are stored are called passive participants. 
2. **Perform local reads:** Next, the worker thread looks up the values of all records in the read set that are stored locally. Depending on the storage interface, this may mean making a copy of the record to a local buffer, or just saving a pointer to the location in memory at which the record can be found.
3. **Serve remote reads:** All results from the local read phase are forwarded to counterpart worker threads on every actively participating node. Since passive participants do not modify any data, they need not execute the actual transaction code, and therefore do not have to collect any remote read results. If the worker thread is executing at a passively participating node, then it is finished after this phase
4. **Collect remote read results:** If the worker thread is executing at an actively participating node, then it must execute transaction code, and thus it must first acquire all read results—both the results of local reads (acquired in the second phase) and the results of remote reads (forwarded appropriately by every participating node during the third phase). In this phase, the worker thread collects the latter set of read results. 
5. **Transaction logic execution and applying writes** Once the worker thread has collected all read results, it proceeds to execute all transaction logic, applying any local writes. Nonlocal writes can be ignored, since they will be viewed as local writes by the counterpart transaction execution thread at the appropriate node, and applied there.

#### Dependant Transactions
Are not supported

Instead, Calvin supports a scheme called Optimistic Lock Location Prediction (OLLP), which can be implemented at very low overhead cost by modifying the client transaction code itself. The idea is for dependent transactions to be preceded by an inexpensive, low-isolation, unreplicated, read-only reconnaissance query that performs all the necessary reads to discover the transaction’s full read/write set. The actual transaction is then sent to be added to the global sequence and executed, using the reconnaissance query’s results for its read/write set. Because it is possible for the records read by the reconnaissance query to have changed between the execution of the reconnaissance query and the execution of the actual transaction, the read results must be rechecked, and the process have to may be (deterministically) restarted if the “reconnoitered” read/write set is no longer valid.

### With Disk Based storage
Any time a sequencer component receives a request for a transaction that may incur a disk stall, it introduces an artificial delay before forwarding the transaction request to the scheduling layer and meanwhile sends requests to all relevant storage components to “warm up” the disk-resident records that the transaction will access. If the artificial delay is greater than or equal to the time it takes to bring all the disk-resident records into memory, then when the transaction is actually executed, it will access only memory resident data. Note that with this scheme the overall latency for the transaction should be no greater than it would be in a traditional system where the disk IO were performed during execution (since exactly the same set of disk operations occur in either case)—but none of the disk latency adds to the transaction’s contention footprint.

### Checkpointing
Supports three types of checkpointing
* Naive Synchronous
The Replica will not serve any requests when this happens.
* Zig-Zag Algorithm
General algorithm keeps a copy of each data entry. Read the paper for more details.
Calvin captures a snapshot with respect to a virtual point of consistency, which is simply a pre-specified point in the global serial order. When a virtual point of consistency approaches, Calvin’s storage layer begins keeping two versions of each record in the storage system—a “before” version, which can only be updated by transactions that precede the virtual point of consistency, and an “after” version, which is written to by transactions that appear after the virtual point of consistency. Once all transactions preceding the virtual point of consistency have completed executing, the “before” versions of each record are effectively immutable, and an asynchronous checkpointing thread can begin checkpointing them to disk.
* Take advantage of MVCC
since versioning in built in within the storage system, just checkpoint the whole data before a certain version.

