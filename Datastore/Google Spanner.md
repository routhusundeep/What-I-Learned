[paper](https://blog.acolyer.org/2015/01/08/spanner-googles-globally-distributed-database/) #relational_database #real_world_system #reread

Many projects at goggle use BigTable, but some are not satisfied with, such projects generally have complex, evolving schemas, require strong consistency in the presence of wide area replication. Many applications prefer to use Megastore because of its semi relational data model and support for synchronous replication, despite its poor write throughput. As a result Spanner has evolved from BigTable like versioned key-value store into a temporal multi-version database. Data is stored in a schematized semi-relational tables; data is versioned, and each version is automatically timestamped with its current time; old versions of data are subject to configurable garbage-collection policies; and applications can read data at old timestamps.
Some interesting features are 
1. replication configuration can be controlled dynamically by the applications
2. control read latency by mentioning the datacenters on which the data should be present
3. constrain the number of replica of replicas
4. Data movement between the datacenters is dynamic and transparent
5. provides external consistent reads and writes, and globally-consistent reads across the database at a time stamp
6. These features allow Spanner to support consistent backups, consistent MapReduce executions, and atomic schema updates, all at global scale, and even in the presence of ongoing transactions

### Implementation
- Spanner deployment is called a deployment
- organized as a set of zones, where each zone is the rough analog of a deployment of BigTable servers
- zones are the locations across which data can be replicated and are physically isolated
- zone has one zonemaster and hundreds of spanservers
- The universe master and placement driver are singletons
- The universe master is primarily a console that displays status information of all zones for interactive debugging
- The placement driver handles automated movement of data across zones on the timescale of minutes

#### Spanserver Software Stack
- Each one handles between 100 to 1000 instances of data structure called a tablet
- A tablet will have data with a timestamp, so Spanner is similar to multi version database
- A tablet state is stored in set of B-tree-like files and a write-ahead log, all on Colossu
- Paxos state machine is implemented on top of each tablet to support replication
- Each state machine stores its metadata and log in its corresponding tablet
- Variant Paxos is used which supports long lived leaders which are time based whose length defaults to 10 seconds
- The current implementation logs every Paxos write twice, once in Paxos log and once in tablets log
- writes much initiate Paxos protocol, while reads access state directly from any tablet which is sufficiently up to date
- At every replica which is a leader, spanserver implements a concurrency control through a lock table
- The lock table contains the state for two-phase locking: it maps ranges of keys to lock state
- transaction manager is implemented at the leader
- if a transaction involves only one Paxos group then it can bypass the manager otherwise a two phase lock is required between the participating groups(technically, it is between the leaders)

#### Directories and Placement
- Supports a bucketing abstraction called a directory
- directory is a set of contiguous keys with a common prefix
- It is the unit of data placement, when data moves it moves the directory by directory
- All data in a directory has the same replication configuration and can be specified by the user
- Paxos group can contain multiple directories, so a tablet can encapsulate multiple partitions in the row space, this decision is made so that frequently accessed data can be co-located
- Movedir handles directory movement and add/remove replicas, data movement is not atomic, when all but a nominal data is moved, it moves the remaining data and updates the group information transactionally
- directory can be split into fragments if their size increases more than a threshold

### Data Model
- Very similar to relational, but not pure
- every table is required to have an ordered set of one or more primary keys
- hierarchy in the schemas are supported

### True Time
- API
  - TT.now() - TTInterval:[earliest, latest]
  - TT.after(t) - true if t has definitely passed
  - TT.before(t) - true if t has definitely not arrived
- underlying time reference used are GPS and Atomic clocks
  - GPS reference-source vulnerabilities include antenna and receiver failures, local radio interference, correlated failures and GPS system outages
  - Atomic clocks fail in ways uncorrelated to GPS and each other, and over long periods of time can drift significantly due to frequency error
- Implemented by a set of time master per datacenter and timeslave daemons per machine
- Majority of masters have GPS receivers with dedicated antennas, these masters are separated physically to reduce the effects of antenna failures, radio interference and spoofing
- The remaining masters are equipped with atomic clocks
- All masters time reference are regularly compared against each other
- Each master also cross-checks the rate at which its reference advances time against its local time, and evicts itself if substantial divergence
- Between synchronizations, atomic clocks advertise a slowly increasing time uncertainty that is derived from conservatively applied worst-case drifts
- GPS masters advertise uncertainty close to zero
- Every daemon polls multiple masters to reduce vulnerability to error from one master, Marzullo's algorithm is detect and reject liars, and to synchronize itself with the non-liars
- Between synchronizations, the daemon advertises a slowly increasing time uncertainty, which is conservative worst case estimate

### Concurrency Control
True time is used to guarantee the correctness property, implement features like externally consistent transactions, lock free read-only transactions and non-blocking reads in the past
**** Timestamp Management
- supports
  - read-write transactions
  - read-only transactions(predeclared snapshot-isolation transactions)
  - snapshot reads
- RW and RO and internally retried
- RO will be executed on any replica that is sufficiently up to date without blocking the writes
- SR will happen an any replica that is sufficiently up to date, the clients mention an exact time or upper bound of the timestamp
- for RO and SR a read position will be given to the client so that the queries are retryable

#### Paxos Leader Leases
- uses timed leases to make leaderships long lived
- upon receiving a quorum of lease votes, the leader is said to have a lease
- A replica extends its lease vote implicitly upon write
- leader request lease-vote extensions if they are near expiration
- the following disjoint invariant will be held
  - for each Paxos groups: each Paxos leader's lease interval is disjoint from every other leader's
  - read the appendix to find out how

#### Assigning Timestamps to RW transactions
- RW uses two phase locking
- so, they can assigned timestamps at any time when all locks have been acquired, before any locks have been released
- For a given transaction, Spanner assigns it the timestamp that Paxos assigns to the Paxos write that represents the transaction
- It depends on the following invariant
  - within each Paxos group, Spanner assigned timestamp to Paxos writes is monotonically increasing, even across leaders
  - easy to prove using the disjoint invariant
- Other external variant is maintained
  - if start of T2 occurs after the commit of T1 then commit timestamp of T2 is greater than the commit timestamp of T1

#### Serving Reads at a Timestamp
- Every replica keeps track of a safe time
- safe time = minimum of( safe time of Paxos state machine, safe time of transaction manager)
  - paxos safe is nothing but the time of previous Paxos write
  - transaction safe is the lower bound sent the latest two-phase transaction start
- This safe time can be used to determine whether the transaction can be served or not
***** Assigning timestamp to RO
- if the scope of read is across a single group then the request is forwarded to the Paxos leader which assigns it timestamp equal to the latest write(LastTs())
- if the scope is across multiple groups then
  - ideally negotiation should happen across all the leaders but it is time consuming
  - so, TT.now().latest is assigned, but the replica should wait till this has passed the safe time to deliver the result

####  Details
#### RW transactions
- writes are buffered till commit, so reads cant see the writes
- reads use wound wait to prevent deadlocks
- client issues reads to the leader of appropriate group
- when read is completed and writes are buffered, it begins two-phase commit
- Client chooses the coordinator group and sends the commit message to each leader with the identity of the coordinator
- A non-coordinator leader first acquires the write locks, logs a prepare record through Paxos, notifies the coordinator of the its prepare timestamp
- The coordinator leader acquire write locks but skips prepare phase
  - It waits for all the non-coordinator timestamps, and assigns a timestamp greater than all of them and any timestamp it has assigned before
  - logs a commit record through Paxos
  - waits for this time to pass, then sends this timestamp to the non-coordinators

##### RO transactions
Already discussed
#### Schema change transactions
- prepare phase explicitly assigns a timestamp in the future(t)
- for reads and writes which implicitly depend on the schema, continues for the previous schema before t but are blocked before the change if after t

### Refinements
- safe time of transaction manager can be fine grained to include key ranges, otherwise irrelevant reads must wait for the writes
- similarly LastTs()
- safe Paxos time also has a weakness, a SR at t cannot execute on a Paxos group whose last write happened before t
  - keeps track of the minimum timestamp assigned to the next leader
  - in the absence of prepared transactions, replicas can perform read if before this time
