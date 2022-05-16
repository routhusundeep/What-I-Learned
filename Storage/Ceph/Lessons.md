#distributed_file_system #real_world_system 

#### [[Lessons#BlueStore A Clean-Slate Approach|BlueStore]]
A new backend designed to run directly on raw storage devices. By running in user space and fully controlling the I/O stack, it has enabled space-efficient metadata and data checksums, fast overwrites of erasure-coded data, inline compression, decreased performance variability, and avoided a series of performance pitfalls of local file systems. Finally, it makes the adoption of backwards-incompatible storage hardware possible, an important trait in a [[Shingled magnetic recording|changing storage ecosystem]]

**Problems Ceph faced.**
- Efficient Transactions
	- Existing Approaches have perf overhead, limited functionality, complex
	- can't leverage internal FS transaction mechanism
- Fast Metadata Operations
	- a key challenge that the Ceph team faced was enumerating directories with millions of entries fast, and the lack of ordering in the returned result
- Support for novel, backward incompatible storage hardware
	- SMR works well will zone interfaces similar to SSD's eliminating [FTL](https://en.wikipedia.org/wiki/Flash_memory_controller)

Read the paper to understand all the problems in detail.


 **Novelties of BlueStore**
 - storing low-level file system metadata, such as extent bitmaps, in a key-value store, thereby avoiding on-disk format changes and reducing implementation complexity
 - optimizing clone operations and minimizing the overhead of the resulting extent reference-counting through careful interface design
 - BlueFS—a user space file system that enables RocksDB to run faster on raw storage devices
 - a space allocator with a fixed 35 MiB memory usage per terabyte of disk space


### Essentials of distributed storage backend?
A distributed file system provides a unified view over aggregated storage from multiple physical machines. It should offer high bandwidth, horizontal scalability, fault tolerance, and strong consistency. The _storage backend_ is the software module directly managing the storage device attached to physical machines.

#### Ceph Distributed Storage System Architecture
You can read about [[RADOS|here]]
- Objects in RADOS are stored in logical partitions called pools.
- Pools can be configured to provide redundancy either through replication or erasure coding
- Within a pool, the objects are sharded among aggregation units called placement groups
- PG's are mapped to multiple OSD's using CRUSH
- Each OSD processes client I/O requests from librados clients and cooperates with peer OSD's to replicate or erasure code updates, migrate data, or recover from failures.
- Data is persisted to the local device via the internal ObjectStore interface, which provides abstractions for objects, object collections, a set of primitives to inspect data, and transactions to update data.
	- A transaction combines an arbitrary number of primitives operating on objects and object collections into an atomic operation

#### Problems in Detail
**Efficient Transactions**
- Leveraging File System Internal Transactions
	- Rollbacks not allowed
	- Most of them are deprecated due to high barrier of entry
- Implementing WAL in User space
	- Slow Read-Modify-Write
		- This is common use case
		- Should perform three steps for each transaction
			- Serialize transaction and write to log
			- fsync to commit transaction
			- Perform operation specified in the transaction
		- So write should wait till all three steps of read operation are completed
	- Non-Idempotent Operation
		- Replaying WAL after crash in non-idempotent(read paper for an example)
		- Guards can be used but verifying correctness becomes difficult
	- Double Writes
		- Two writes happen, first to WAL and then to File System
- Using KVS as the WAL
	- RocksDB can be used
	- Slow Read-Modify-Write is avoided since we can read new state without waiting for its completion
	- Non-Idempotent Operation is avoided
		- for e.g. clone a → b, if 'a' is small, it is copied and inserted into transaction, if the object is large, a copy-on-write mechanism is used, which changes both a and b to point to the new data and initial data is read only
	- double writes is avoided
		- because the object namespace is now decoupled from the file system state. Therefore, data for a new object is first written to the file system and then a reference to it is atomically added to the database
	- Causes Journaling of journal issue
		- Creating an object has two steps
			- writing to a file and calling fsync
			- writing the object metadata to RocksDB synchronously
		- Ideally, the fsync in each step should issue one expensive FLUSH CACHE command to disk but with a journaling file system each fsync issues two flush commands, after writing the data and after committing the metadata change to the journal.

**Fast Metadata Operation**
slow directory enumeration (readdir) operations on large directories, and the lack of ordering in the returned result

**Support for New Hardware**
- drive-managed SMR drives that are backward compatible have unpredictable performance
- host-managed SMR drives with zone interfaces are backward-incompatible
- Zoned Namespaces (ZNS) defines an interface for managing SSDs without an FTL
	- Eliminating the FTL results in many advantages, such as reducing the write amplification, improving latency outliers and throughput, reducing  overprovisioning by an order of magnitude, and cutting the cost by reducing DRAM—the highest costing component in SSD after the NAND flash.


### BlueStore: A Clean-Slate Approach
Some main goals of BlueStore were:
1. Fast metadata operations 
2. No consistency overhead for object writes
3. Copy-on-write clone operation
4. No journaling double-writes
5. Optimized I/O patterns for HDD and SSD

It's not POSIX complete and implemented in user space
BlueStore runs directly on raw disks. A space allocator within BlueStore determines the location of new data, which is asynchronously written to disk using direct I/O. Internal metadata and user object metadata is stored in RocksDB, which runs on BlueFS, a minimal user space file system tailored to RocksDB. The BlueStore space allocator and BlueFS share the disk and periodically communicate to balance free space.

#### BlueFS and RocksDB
RocksDB abstracts out its requirements from the underlying file system in the Env interface. BlueFS is an implementation of this interface in the form of a user space, extent-based, and journaling file system. 

`Check out the actual implementation in the paper.`

