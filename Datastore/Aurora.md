
[paper](https://www.allthingsdistributed.com/files/p1041-verbitski.pdf) #real_world_system #relational_database #reread 


### Introduction
- Industry need capacity on a flexible on-demand basis
- achieved by typically decoupling storage and compute
- storage is replicated over multiple machines
- doing so, we can handle unreachable hosts, adding replicas, failing over from a writer to a replica, scaling up or down
- there will be amplification of traffic over network, since a write is issued out to a storage fleet in parallel
- synchronous operations require stalls and context switches
- Aurora Architecture briefing
  - multi-tenant scale-out storage service that abstracts a virtualized segmented redo log and is loosely coupled to a fleet of database instances
  - Each instance still includes most of the traditional kernel(query processing, transactions, locking, buffer cache, access methods and undo management), several functions(redo logging, durable storage, crash recovery and backup/restore) are off-loaded to the storage service
- Advantages of this architecture
  - by making storage as independent, fault-tolerant and self-healing service across multiple data-centers, the database is protected from performance variance and transient or permanent failures at either the networking or storage tiers. failure in durability can be modeled as a long lasting availability event, and a availability event can be modeled as a long-lasting performance variation
  - by only writing redo log records to storage, we can reduce network IOPS by an order of magnitude
  - move some of the most complex and critical function(backup and redo recovery) from one-time expensive operations in the database engine to continuous asynchronous operations amortized across the storage fleet

### Durability at Scale
#### Replication and Correlated Failures
- Instances fail, get shut down, new ones get added, so decouple the compute from storage
- storage nodes fail, so replicate
- uses quorum-based voting protocol to tolerate failures
- model used is (W,R,N) = (4,3,6)
- tolerates
  - losing an entire AZ(availability zone) and a node, without losing the data
  - can write even when an AZ is lost
#### Segmented Storage
- MTTF(mean time to failure) can't be reduce for independent failures, so tries to reduce MTTR(mean time to repair)
- database volume is partitioned into segments, of size 10GB.
- segments are replicated 6 ways into PGs(Protection groups) across 3 AZs
- physically implemented over a large fleet of storage nodes that are provisioned as virtual hosts with attached SSDs using EC2
- volume is added to PG dynamically, can support up to 64TB
- A 10 GB segment can be repaired in 10 seconds on a 10GBps network
- so two nodes plus a AZ should fail within this 10 second span for it to be unresponsive
**** Operational Advantage of Resilience
- if resilient to long failures, it is naturally resilient to shorter ones
- this resilience is used for various other operations like
  - heat management, a segment is marked as a hot disk until fixed
  - OS and security patching, brief unavailability event at the segment
  - software upgrades, one AZ at a time and ensure no more than one member of a PG

### The LOG is the Database
#### The Burden of Amplified Writes
- 4/6 write quorum, though resilient, will suffocate the network bandwidth and is not tenable with a straight forward MySQL
- The section explains all the cross instance data transfers happening in a MySQL setup
#### Offloading Redo Processing to Storage
- Transaction commit only writes the log, the data page write is deferred
- only these redo log records are sent across the network
- No pages are ever written from the database tier
- The log applicator is pushed to the storage tier where it can be used to generate database pages in background or on demand
- The log is the database, so these database pages are optional and are simple a cache of the log
- Aurora page materialization is governed by the length of the chain for a given page
- The primary only writes log records to the storage service and streams those log records as well as metadata updates to the replica instances
- The writes to the storage tier are batched at the primary as per the PG
- You can see the benchmark for this in the paper
- The storage tier will see unamplified writes, since it is only one of the size copies
- Moving processing to storage also improved availability by minimizing crash recovery time and eliminates jitter caused by background processes such as checkpointing, background data page writing and backups
- Nothing is required at instance startup, any read request for a data page will require some redo records to be applied if the page is not current
#### Storage Service Design Points
- core design tenet is to minimize the latency of foreground write request
- steps are
  - receive log records and add it to a in-memory queue
  - persists records on disk and acknowledge
  - organize records and identify gaps in the log since some batches may be lost
  - gossip with peers to fill in the gaps
  - coalesce log records into new data pages
  - periodically stage logs and new records to S3
  - periodically GC old versions
  - periodically validate CRC codes on pages

### The LOG Marches Forward
#### Solution Sketch: Asynchronous Processing
- each log record has a sequence number that is monotonically increasing(LSN)
- we maintain points of consistency and durability, and continually advance these points as we receive acknowledgements for outstanding storage requests
- upon restart, before the database is allowed to access the storage, the storage service does its own recovery which is focused not on user-level transactions, but making sure that the database sees an uniform view despite its distributed nature
- The storage determines the highest LSN for which it can guarantee availability of all prior log records, this is known as Volume Complete Number(VCN)
- The database can, however, further constrain a subset of points that are allowable for truncation by tagging log records and identifying them as CPLs or Consistency Point LSNs. We therefore define VDL or the Volume Durable LSN as the highest CPL that is smaller than or equal to VCL and truncate all log records with LSN greater than the VDL.
- In practice
  - Each database-level transaction is broken up into multiple mini-transactions(MTR) that are ordered and must be performed atomically
  - Each MTR is composed of multiple contiguous log records
  - The final record in a MTR is its CPL

#### Normal Operation
- Writes
  - new LSN is given for each log record with a constrain that LSN < VDL + configured number(currently 10 million), this introduces back pressure, in case the storage or network cant keep up
  - Each log record contains a backlink to identify the previous record
  - using these backlinks the storage can determine the complete lineage
  - on encountering an CPL, the storage node will gossip regarding the missing records
- Commits
  - completely asynchronous, gives the record a "commit LSN" as part of separate list of transactions
  - will complete only iff VDL > commit LSN
  - as VDL moves forward, the database will identify qualifying completed transactions and responds to the client via a dedicated thread
- Reads
  - can serve from the cache, no need to contact the storage is most of the cases
  - To prevent dirty reads, the cache page is evicted if the page LSN is greater than VDL
  - on cache miss, it is sufficient to request a page as of version VDL to get its latest durable version
  - The database normally keeps track of the SCL's of the segments, so it does not need a quorum read always, it just needs to send it to the segment with SCL > read-point(VDL when the read was received)
  - As database is aware of outstanding reads, it can compute at any time the Minimum Read Point LSN on a per PG basis, it is called PGMRPL
  - writer gossips with this value with the read replica
  - In other words, a storage node segment is guaranteed that there will be no read-point request with value less than PGMRPL, so the GC older versions
- Replicas
  - a single writer, upto 15 read replicas can be present
  - The log stream generated by writer which is sent to storage nodes is also sent to reader asynchronously
  - log applicator is used on the reader to apply the change just like storage nodes
  - only applies if LSN > VDL
  - log records as part of single MTR are applied atomically
  - typically reader lags by 20 ms

### Recovery
- the system contacts each PG, a read quorum of segments which is sufficient to guarantee discovery of any data that could have reached a write quorum
- once a read quorums are established, it recalculates the VDL above which the data is truncated by generating a truncation range
