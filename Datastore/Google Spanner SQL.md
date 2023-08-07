[paper](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/acac3b090a577348a7106d09c051c493298ccb1d.pdf) #database #relational_database #real_world_system #reread 

### Brief Background
- sharded(range) geo-replicated relational database
- inheritance among tables can exist
- data locality is maintained for interleaved child tables
- uses a replicated write-ahead redo log, and paxos is used for get replicas to agree with each other
- pessimistic locking and timestamps are used for concurrency control
- a special RPC framework called co-processor framework is provided to hide the complexity of locating data
- This framework is responsible for finding the paxos group to query for a key, it forwards the query to the nearest replica which is upto date as per the concurrency mode
- The storage is based on log-structured-merge trees

### Query Distribution
#### Distributed query compilation
- uses relational algebra operator tree
- Distributed union operator is core, so that dynamic re-partition can be accommodated smoothly
- This union operator can be pulled up using standard relational logic

#### Distributed Execution
- Shard pruning is used to avoid querying irrelevant data
- runtime analysis of the sharding key is done to get the relevant ranges

#### Distributed joins
- Row-at-a-time is optimized by Batch-ing a set of rows from the input and unnest-ing the batch into individual rows at the target shard
- join on the batch happens remotely on the shard itself

#### Query Distribution API
- Single consumer
  - query from client is sent to a root server
  - upon first execution, the plan is analysed to detect sharded key expression
  - It is converted into a special pattern that encodes how to obtain sharding keys as a function of query parameters
  - This pattern is cached in the client
- Parallel consumer
  - The result of some queries are prohibitively large to be consumed on a single machine
  - The work is divided into between the desired number of clients
  - The API takes a SQL query and degree of parallelism and returns a set of opaque query descriptor partitions
  - Then the query is executed on the clients using these descriptors
  - For this to work, the query should be root partitionable, that is a distributed union should be present as a root operator

### Query Range Extraction
#### Problem Statement
- Several flavours
  - Distributed range extraction
    - Figure out the table shards referenced by the query
  - Seek range extraction
    - What fragments of the relevant shards to read from the underlying storage
  - Lock range extraction
    - The extracted key ranges determine what ranges to be locked for pessimistic concurrency, or to check potential pending transactions
- Examples are mentioned in the paper

#### Compile time re-writing
- query is normalized and rewrite a filtered scan expression into a tree of correlated self joins that extract the ranges for successive key columns
- see the example in paper for clear picture

#### Filter Tree
- runtime data structure to extract the key ranges via bottom up set operators, and for post filtering the rows emitted by self joins
- The computed range is a general approximation and used as a heuristic
- The data structure is quite simple, see the example in paper

### Query Restarts
- Automatically compensates for failures, resharding, and binary rollouts, affecting latency in a minimal way
#### Usage Scenarios and Benefits
- Hiding transient failures
  - All transient failures are hidden
  - transient failures like network disconnects, machine reboots and process crashes, distributed waits(instead of blocking, query the replica), data movement(re-sharding)
  - simpler programming model - no retry loops
  - Streaming pagination through query results
    - client code may stop consuming results for prolonged periods of time
  - Improved tail latency
  - Forward progress for long running queries
  - Recurrent rolling upgrades
  - Simpler Spanner Internal error handling

#### Contract and requirements
- A restart token is sent along with the query
- query restarts is handling by capturing the distributed state of the query plan generated
- problems are
  - Dynamic Resharding
  - Non-Determinism
  - Restarts across server versions
    - restart token wire format
      - previous and later versions should be parsable by the server
    - query plan
    - Operator behaviours

### Blockwise-Columnar Storage
- Initially used SSTable(same as BigTable)
- Ressi is the new low-level storage format
- Data Layout
  - stores as LSM tree, which is regularly compacted
  - organizes data into blocks in row-major order, but lays the data within a block in column major order
  - divides the values which contains only the most recent values, and an inactive file, which contains older values
- Live data migration
  - Paxos groups are converted as per the new layout, so rollbacks are also possible
  - Once all data is moved to new group from the live group, it takes over the responsibility

### Lessons learned
- SQL is very usable and adoptable
- Transaction guarantees differ widely across applications
- True time is very useful
- Declarative nature of SQL enables many optimizations
