[paper](https://static.googleusercontent.com/media/research.google.com/en/us/archive/gfs-sosp2003.pdf) #google  #real_world_system  #distributed_file_system 

We have reexamined traditional choices and explored radically different points in the design space. Some key departures from earlier files systems are.
- Component failures are the norm rather than the exception.
- Files are huge by traditional standards. Multi-GB files are common
	- design assumptions, and parameters such as I/O operation and block sizes have to be revisited.
- Most files are mutated by appending new data rather than overwriting existing data
- co-designing the applications and the file system API benefits the overall system by increasing our flexibility
	- we have relaxed GFS’s consistency model to vastly simplify the file system without imposing an onerous burden on the applications

### Design Overview
#### Assumptions
- The system is built from many inexpensive commodity components that often fail. It must constantly monitor itself and detect, tolerate, and recover promptly from component failures on a routine basis
- The system stores a modest number of large files. Small files must be supported, but we need not optimize for them.
- The workloads primarily consist of two kinds of reads: large streaming reads, and small random reads
- The workloads also have many large, sequential writes that append data to files.
- The system must efficiently implement well-defined semantics for multiple clients that concurrently append to the same file.
- High sustained bandwidth is more important than low latency

#### Interface
- Files are organized hierarchically in directories and identified by path names.
- Support the usual operations to create, delete, open, close, read, and write files.
- Has snapshot and record append operations

#### Architecture
- A cluster consists of a single master and multiple chunkservers are accessed by multiple clients
- Files are divided into fixed-size chunks.
- Each chunk is identified by an immutable and globally unique 64 bit chunk handle assigned by the master at the time of chunk creation
- Chunkservers store chunks on local disks as Linux files
- each chunk is replicated on multiple chunkservers
- The master maintains all file system metadata
	- This includes the namespace, access control information, the mapping from files to chunks, and the current locations of chunks.
	- It also controls system-wide activities such as chunklease management, garbage collection of orphaned chunks, and chunkmigration between chunkservers.
	- Communicates with each chunkserver in HeartBeat messages

#### Single Master
- Simplifies design
- Clients never read and write file data through the master
	- Instead, a client asks the master which chunkservers it should contact. It caches this information for a limited time and interacts with the chunkservers directly for many subsequent operations.

#### Chunk Size
We have chosen 64 MB, which is much larger than typical file system blocksizes. A large chunksize offers several important advantages.
- Reduces clients’ need to interact with the master
- since on a large chunk, a client is more likely to perform many operations on a given chunk
- reduces the size of the metadata stored on the master

Disadvantages of large chunk size
- A small file consists of a small number of chunks, perhaps just one. The chunkservers storing those chunks may become hot spots if many clients are accessing the same file.

#### Metadata
The master stores three major types of metadata: the file and chunk namespaces, the mapping from files to chunks, and the locations of each chunk’s replicas.
The master does not store chunklocation information persistently. Instead, it asks each chunkserver about its chunks at master startup and whenever a chunkserver joins the cluster.

##### In-Memory Data Structures
Since metadata is stored in memory, master operations are fast. Furthermore, it is easy and efficient for the master to periodically scan through its entire state in the background. This periodic scanning is used to implement chunkgarbage collection, re-replication in the presence of chunkserver failures, and chunkmigration to balance load and diskspace usage across chunkservers.
One potential concern for this memory-only approach is that the number of chunks and hence the capacity of the whole system is limited by how much memory the master has.

##### Chunk Locations
The master does not keep a persistent record of which chunkservers have a replica of a given chunk. It simply polls chunkservers for that information at startup. The master can keep itself up-to-date thereafter because it controls all chunkplacement and monitors chunkserver status with regular HeartBeat messages.

##### Operation Log
The operation log contains a historical record of critical metadata changes
Not only is it the only persistent record of metadata, but it also serves as a logical timeline that defines the order of concurrent operations. Files and chunks, as well as their versions, are all uniquely and eternally identified by the logical times at which they were created.
We replicate it on multiple remote machines and respond to a client operation only after flushing the corresponding log record to disk both locally and remotely.

### Consistency Model
#### Guarantees by GFS
- File namespace mutations (e.g., file creation) are atomic
	- handled exclusively by the master: namespace locking guarantees atomicity and correctness
- The state of a file region after a data mutation depends on the type of mutation, whether it succeeds or fails, and whether there are concurrent mutations.


--- | Write | Record Append
------------ | ------------- | -------------
Serial Success | defined | defined interspersed with inconsistent
Concurrent Success | consistent but undefined| same as above
Failure | Inconsistent | Inconsistent

- A file region is **consistent** if all clients will always see the same data, regardless of which replicas they read from. 
- A region is **defined** after a file data mutation if it is consistent and clients will see what the mutation writes in its entirety.

- A **write** causes data to be written at an application-specified file offset.
- A **record append** causes data (the “record”) to be appended atomically at least once even in the presence of concurrent mutations, but at an offset of GFS’s choosing

After a sequence of successful mutations, the mutated file region is guaranteed to be defined and contain the data written by the last mutation. GFS achieves this by 
- applying mutations to a chunk in the same order on all its replicas 
- using chunkversion numbers to detect any replica that has become stale because it has missed mutations while its chunkserver was down 

### SYSTEM INTERACTIONS
#### Leases and Mutation Order
The master grants a chunklease to one of the replicas, which we call the primary. The primary picks a serial order for all mutations to the chunk. All replicas follow this order when applying mutations. Thus, the global mutation order is defined first by the lease grant order chosen by the master, and within a lease by the serial numbers assigned by the primary.

Control flow of writes:
1. The client asks the master which chunkserver holds the current lease for the chunk and the locations of the other replicas. If no one has a lease, the master grants one to a replica it chooses. 
2. The master replies with the identity of the primary and the locations of the other (secondary) replicas. The client caches this data for future mutations. It needs to contact the master again only when the primary becomes unreachable or replies that it no longer holds a lease
3. The client pushes the data to all the replicas. A client can do so in any order. Each chunkserver will store the data in an internal LRU buffer cache until the data is used or aged out. By decoupling the data flow from the control flow, we can improve performance by scheduling the expensive data flow based on the network topology, regardless of which chunkserver is the primary.
4. Once all the replicas have acknowledged receiving the data, the client sends a write request to the primary. The request identifies the data pushed earlier to all the replicas. The primary assigns consecutive serial numbers to all the mutations it receives, possibly from multiple clients, which provides the necessary serialization. It applies the mutation to its own local state in serial number order.
5. The primary forwards the write request to all secondary replicas. Each secondary replica applies mutations in the same serial number order assigned by the primary
6. The secondaries all reply to the primary indicating that they have completed the operation.
7. The primary replies to the client. Any errors encountered at any of the replicas are reported to the client. In case of errors, the write may have succeeded at the primary and an arbitrary subset of the secondary replicas. (If it had failed at the primary, it would not have been assigned a serial number and forwarded.) The client request is considered to have failed, and the modified region is left in an inconsistent state. Our client code handles such errors by retrying the failed mutation. It will make a few attempts at steps (3) through (7) before falling back to a retry from the beginning of the write.

#### Data Flow
To fully utilize each machine’s network bandwidth, the data is pushed linearly along a chain of chunkservers rather than distributed in some other topology (e.g., tree). Thus, each machine’s full outbound bandwidth is used to transfer the data as fast as possible, rather than divided among multiple recipients.
To avoid network bottlenecks and high-latency links (e.g., inter-switch links are often both) as much as possible, each machine forwards the data to the “closest” machine in the network topology that has not received it.

#### Atomic Record Appends
This is similar to writing to a file opened in O APPEND mode in Unix without the race conditions when multiple writers do so concurrently.

#### Snapshot
The snapshot operation makes a copy of a file or a directory tree (the “source”) almost instantaneously, while minimizing any interruptions of ongoing mutations

We use standard copy-on-write techniques to implement snapshots. 
- When the master receives a snapshot request, it first revokes any outstanding leases on the chunks in the files it is about to snapshot. This ensures that any subsequent writes to these chunks will require an interaction with the master to find the leaseholder.
- Master logs the operation to disk. It then applies this log record to its in-memory state by duplicating the metadata for the source file or directory tree. The newly created snapshot files point to the same chunks as the source files.
- The first time a client wants to write to a chunk after the snapshot operation, it sends a request to the master to find the current leaseholder. The master notices that the reference count for chunk is greater than one. It defers replying to the client request and instead picks a new chunk handle. It then asks each chunkserver that has a current replica to create a new chunk By creating the new chunko n the same chunkservers as the original, we ensure that the data can be copied locally, not over the network


### MASTER OPERATION
#### Namespace Management and Locking
GFS logically represents its namespace as a lookup table mapping full pathnames to metadata. With prefix compression, this table can be efficiently represented in memory. Each node in the namespace tree (either an absolute file name or an absolute directory name) has an associated read-write lock
Each master operation acquires a set of locks before it runs. Typically, if it involves /d1/d2/.../dn/leaf, it will acquire read-locks on the directory names /d1, /d1/d2, ..., /d1/d2/.../dn, and either a read lock or a write lockon the full pathname /d1/d2/.../dn/leaf.

#### Replica Placement
We must also spread chunkreplicas across racks. This ensures that some replicas of a chunk will survive and remain available even if an entire rack is damaged or offline. It also means that traffic, especially reads, for a chunkcan exploit the aggregate bandwidth of multiple racks. On the other hand, write traffic has to flow through multiple racks, a tradeoff we make willingly.

#### Creation, Re-replication, Rebalancing
When the master creates a chunk, it chooses where to place the initially empty replicas. It considers several factors. 
- We want to place new replicas on chunkservers with below-average diskspace utilization.
- We want to limit the number of “recent” creations on each chunkserver. Although creation itself is cheap, it reliably predicts imminent heavy write traffic because chunks are created when demanded by writes, and in our append-once-read-many workload they typically become practically read-only once they have been completely written.
- As discussed above, we want to spread replicas of a chunkacross racks.

Re-replication happens by taking the same above factors into consideration.

The master rebalances replicas periodically: it examines the current replica distribution and moves replicas for better diskspace and load balancing. Also through this process, the master gradually fills up a new chunkserver rather than instantly swamps it with new chunks and the heavy write traffic that comes with them.

#### Garbage Collection
When a file is deleted by the application, the master logs the deletion immediately, just like other changes. However, instead of reclaiming resources immediately, the file is just renamed to a hidden name that includes the deletion timestamp. During the master’s regular scan of the file system namespace, it removes any such hidden files if they have existed for more than three days (the interval is configurable). Until then, the file can still be read under the new, special name and can be undeleted by renaming it back to normal. When the hidden file is removed from the namespace, its in memory metadata is erased. This effectively severs its links to all its chunks. 
In a similar regular scan of the chunknamespace, the master identifies orphaned chunks (i.e., those not reachable from any file) and erases the metadata for those chunks. In a Heartbeat message regularly exchanged with the master, each chunkserver reports a subset of the chunks it has, and the master replies with the identity of all chunks that are no longer present in the master’s metadata. The chunkserver is free to delete its replicas of such chunks.

#### Stale Replica Detection
For each chunk, the master maintains a chunk version number to distinguish between up-to-date and stale replicas Whenever the master grants a new lease on a chunk, it increases the chunkversion number and informs the up-to- date replicas. The master and these replicas all record the new version number in their persistent state. This occurs before any client is notified and therefore before it can start writing to the chunk. If another replica is currently unavailable, its chunkversion number will not be advanced. The master will detect that this chunkserver has a stale replica when the chunkserver restarts and reports its set of chunks and their associated version numbers. If the master sees a version number greater than the one in its records, the mas- ter assumes that it failed when granting the lease and so takes the higher version to be up-to-date.

### FAULT TOLERANCE AND DIAGNOSIS
#### Availability
Both the master and the chunkserver are designed to restore their state and start in seconds, no matter how they terminated. In fact, we do not distinguish between normal and abnormal termination; servers are routinely shut down just by killing the process.

#### Master Replication
The master state is replicated for reliability. Its operation log and checkpoints are replicated on multiple machines. A mutation to the state is considered committed only after its log record has been flushed to disk locally and on all master replicas.

#### Data Integrity
Each chunkserver uses checksumming to detect corruption of stored data.
During idle periods, chunkservers can scan and verify the contents of inactive chunks. This allows us to detect corruption in chunks that are rarely read.