[paper](http://www.vldb.org/pvldb/vol7/p1259-gupta.pdf) #real_world_system #reread 

Mainly build for ad-serving platform, requirements for the use case are
1. Atomic Updates - obviously
2. Consistency and Correctness - due to business and legal reasons
3. Availability - No single point of failure, for both planned and unplanned
4. Near Real time update throughput - must support continuous updates, since missing even a short span may lead to loss of revenue in a short term
5. Query Performance - since we are serving live customers, low response time is preferred
6. Scalability - must support trillion of rows and petabytes of data
7. Online data and Metadata transformations - clients can change schema, should not affect the ongoing queries
Mesa ingests data generated from up-streams services, aggregates them and serves them when queried. Problems with the existing systems are
- BigTable does not provide the necessary atomicity requirements
- Megastore, Spanner and F1 which are intended for online transaction processing, do provide strong consistency across geo-distributed data, they do not provide peak update throughput needed by our clients.
- However Mesa takes advantage of BigTable and Paxos technology underlying Spanner for metadata storage and maintenance
Key Contributions of this paper are
- can provide ACID and still scale to the high throughput rate of Google's ad metrics
- novel version managing system which batches updates
  - acceptable latencies and high throughput for writes
  - low latency and high throughput for reads
- highly scalable which tolerate failures at a data centre level
  - application data is asynchronously replicated through independent and redundant process
  - critical metadata is synchronously replicated
- Schema changes can be done for a large number of tables without affecting the correctness or efficiency of existing applications
- withstand problems of data corruption due to software errors and hardware faults
- describes some operational challenges of maintaining this system

### MESA Storage Subsystem
Some example queries are 
- How many ad clicks were there for a particular advertiser on a specific day
- How many ad clicks were there for a particular advertiser matching a certain keyword(eg: decaf) during the first week of a October between 8AM and 11AM that were displayed on google.com for user in a specific geographic location using a mobile device
#### Data Model
- Data is maintained as tables
- Each table has a schema
- Schema specified key space(K) and corresponding value space(V), where both K and V are sets
- Schema also specifies an aggregation function F: V X V -> V which is used to aggregate the values corresponding to the same key
- Commutative Aggregation functions are encouraged
- In practice commutative is not always possible, so a total ordering is maintained in the form of an index
- F((...xi...), (...yi...)) = (...f(xi,yi)...)
#### Updates and Queries
- Updates are done in batches
- Batches are produced by an upstream process, typically at a frequency of a few minutes
- An update consists a version number n and a set of rows of the form (table, key, value)
- A query consists of a version number and a predicate P on the key space
- The response contains one row for each key matching P that appears in some update in version 0 to n.
- It supports more queries, but they can be viewed as pre and post processing along this model
- Atomicity is guaranteed at a version level
- The strict ordering of updates(versioning and order within a version) is useful for non commutative aggregate functions
  - Using this, fraudulent ads can be reverted only after the initial ad is logged
#### Versioned data management
Problems being faced are
- Costly to store each version, aggregated information is much smaller
- Going over all the versions to get results at query time is very costly
- Naive pre-aggregation of all versions on every update can be prohibitively expensive
To handle these
- Pre-aggregates certain versioned data and stores it as deltas
- where delta is a set of rows and a delta version represented by [V1, V2], where V1 and V2 are update version numbers with V1<=V2
- the rows in a delta are the rows which appeared in versions between V1 and V2(inclusively)
- The value of each key is aggregated values of each record with same key within the delta
- updates correspond to a singleton delta(V1=V2)
- Mesa can only query at a particular version for only a limited amount of time, the older versions are merged into a base delta [0,B]
- Base compaction happens asynchronously w.r.t other operations
- Base compaction happens once a day, since it is a costly process
- Cumulative delta is [U,V], 0<B<U<V.
- Cumulative deltas are formed with the recent singletons, so that the queries will be done faster
- The delta compaction policy determines the set of deltas maintained
  - what deltas(excluding the singletons) must be generated prior to allowing an updated version to be queried(synchronously inside the update path, which slows down the updates at the expense of faster queries)
  - what deltas should be generated asynchronously outside the update path
  - when a delta can be deleted
- Once a delta is created, it is immutable
#### Physical data and index formats
- The rows in the deltas are stored in sorted order in data files of bounded size
- The rows themselves are divide into row blocks, each of which is transposed and compressed 
- The transposition lays out the data by columns instead of row for better compression
- compression ratio and read/write decompression time are more important than cost of write/ compression times
- For each data file an index file is stored
  - index entry contains a short key which is fixed size prefix of the first key in the row block and the offset of the compressed row block in the data file
- Each table index contains a copy of the data file sorted on the indexed keys

### System Architecture
#### Single Datacenter instance
Each Mesa Instance is composed of two sub-systems: update/maintenance and querying. These sub-systems are decoupled enabling scaling independently. No direct communication is required between these two systems for operational correctness
- Update/Maintenance SubSystem
  - ensure data is correct, up to date and optimized for querying
  - background operations such as loading updates, table compaction, schema changes and maintaining table checksums
  - follows a controller/worker framework
  - Controller
    - can be viewed as large scale table metadata cache, work scheduler and work queue manager
    - at startup loads metadata from BigTable, subscribes to a metadata feed to listen to schema changes
    - is exclusive write of meta data
    - maintains internal queue's for data manipulation like incorporating updates, delta formation, checksums etc
    - Work that required global co-ordination like schema changes are triggered by external process, and controller accepts the process and adds it to the queue
  - Worker
    - accepts the task given by controller
    - Mesa maintains an in-house worker pool scheduler that scales the number of worker based on the percentage of idle workers available
    - if idle it periodically polls the controller
  - A garbage collector runs continuously
- Query Subsystem
  - performs on the fly aggregation of deltas
  - limited support for server side conditional filtering and "group by"
  - The query servers are organized into multiple sets, each of which is collectively capable of serving all the tables known to the controller
  - Mesa prefers to direct queries related to same table to a subset of servers handling that table, this allows better cache hit rate
  - On startup, each query server registers the list of table it actively caches over a global locator service
#### Multi datacenter deployment
Mesa is deployed on multiple datacenters, each have a independent copy of the actual data
- Consistent Update Mechanism
  - Committer is responsible for coordinating between all the datacenters, one version at a time
  - Committer assigns the version number to batch update, publishes associated metadata with the update(like its location) to the versions database(probably spanner)
  - Committer is stateless, with instances running in each datacenters, for high availability

### Enhancements
#### Query server performance optimizations
  - delta pruning is done based the predicate
  - scan-to-seek can be used to avoid full scans when searched on a non-indexed column
  - Mesa attaches a resume key when returning the results in a streaming fashion, so when Query Server becomes un-responsive, the client can re-query from this resume key to get the remaining result
#### Parallelizing Worker Operations
  - Mesa uses Map/Reduce for data processing in workers
#### Schema changes
  - Naive method
    - make a separate copy of the table with data stored in the new schema version at a fixed update version
    - replay any updates to the table generated in the meantime until the new schema is up to date
    - switch schema's once the newer is up to date
    - older queries can continue running on the older schema for some amount of time
  - linked schema change
    - Schema change is immediately visible to queries
    - At query time both the schemas are read
    - Updates happen to the new schema
    - This is not applicable for all schema changes
#### Mitigating Data Corruption Problems
- when writing, row ordering, key range and aggregate value checks are done
- so during query time, the above checks are re-done, which might help in detecting data corruption(and bugs)
- The index files themselves contain checksums of header and index data
- Global offline checks are also done regularly
  - row order dependant checksum and a weak row order independent checksum are calculated
- Global aggregate value checker is done across all instances
### Experiences and Lessons learned
- Distribution, Parallelism and Cloud Computing
- Modularity, Abstraction and Layered Architecture
- Capacity Planning
- Application level assumptions
- Geo replication
- Data corruption and Component Failure
- Testing and Incremental deployment
- Human Factors
