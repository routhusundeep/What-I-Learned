[paper](https://blog.acolyer.org/2019/09/11/procella/) #real_world_system #reread 

unifying serving and analytical data at youtube
Attacking the following patterns
1. Reporting and Dashboarding   
   - Video creators, content holders and stakeholders require access to real time dashboards to understand how their videos and channels are performing
2. Embedded Statistics -
   - Many useful statistics are exposed to the users such as likes or views of a video
3. Monitoring - 
   - This is very similar to Reporting, but the different lies in that, monitoring is used internally by Engineers, so some support to down-sampling, expiry of old data, approximation functions, time series queries etc is required
4. Ad-hoc Analysis -   
   - Business analysts, data scientists will perform complex ad-hoc queries which are typically long running and resource intensive
With exponential data growth and multiple products, they were facing the problems like
- Polyglot persistence is leading to multiple ETL jobs, which incur high development and maintenance cost, data quality issues, hard to tract bugs etc
- data duplication and accessibility issues due to incompatible API's
- underlying components may not be scalable, efficient when dealing with Youtube level data
To solve these problems, Procella has
1. Rich API  
   - Almost complete standard SQL, with additional useful extensions like approximations in aggregations, handling complex nested and recursive schema etc
2. High Scalability  
   - Compute(on Borg, which is a resource manager/scheduler like Yarn) and Storage(on Colossus, which is a distributed data store like Hadoop and Ceph) are separated
3. High Performance  
   - state-of-the-art query execution techniques
4. Data Freshness  
   - High Volume, low latency ingestion is supported in both batch and Stream mode, native support to Lambda Architecture

### Architecture
#### Google Infrastructure
1. Disaggregated Storage -   
   - All data will be stored in Colossus, which differs from normal file systems, like
   - Data is immutable. A file can be opened and appended to, but once finalized, it cannot be modified
   - Common metadata operations have significant latency especially at tail
   - All durable meta data is remote, so reading data will be at the least a RPC cost
2. Shared Compute - 
   - All servers will run on Borg
   - Vertical scaling will be challenging and many tasks may run concurrently on the same machine, so to improve overall performance it is better to run many small tasks than a small number of large tasks
   - Machines can be brought down for maintenance, so tasks should be able to recover quickly, this strengths the argument of small and stateless task
   - Machines can be of different configurations, some better, some worse, Performance of a task will be unpredicatable
#### Procella's Data
1. Data Storage
   - Data is organized as tables
   - table is stored across multiple files, also referred to as tablets or partitions
   - Has its own Columnar format called Artus for most data, but also supports querying the format Capacitor(Colossus's format)
2. Metadata Storage
   - Procella, like many modern analytical systems, does not use B-Tree's for data storage, instead prefers other indexing strategies like zone maps, bitmaps, bloom filters, partitions keys and sort keys. This information is provided to the query engine for better plans
   - Meta store is mostly Big Table(inspiration for HBase) or Spanner(I will cover this in the future)
3. Table Management
   - Registration Server(RgS) handles this
   - Standard SQL DDL commands will work
   - Additional data ingestion methods, sorting information etc will help with better management
   - Both ingestions have different optimizations
4. Batch ingestion
   - Data is sent to the registration server
   - Data is not scanned due to performance reasons
   - Data servers later scans the data lazily and create appropriate indexes
   - RgS ensures that the meta data is synchronized with the file schemas
5. Realtime ingestion
   - Ingestion Server(Igs) handles this
   - can be ingested through RPC or other PubSub
   - Steps it does on receiving data
     - appends to the WAL of colossus
     - in parallel, it sends to the data servers according to the partition scheme of the table
     - data is stored temporarily in the memory of data servers for query processing
     - Buffers are regularly check pointed
     - Can send to multiple Data servers for redundancy
     - Memory reads are similar to dirty reads, can be switched off to ensure consistency, at the cost of additional data latency
6. Compaction
   - Compacts data written by IgS regularly
   - updates RgS with the compacted information
#### Query Life cycle
- Clients send the queries to Root Server(RS)
- RS does parsing, rewrites, planning and optimizations
- plan structure - query blocks are nodes and edges are streams of data
- The Data Server(DS) receives plan from RS or other DS
- does most of heavy lifting, such as reading from Colossus/local memory/RDMA or other DS
- data servers use Stubby/gRPC and RDMA for shuffle(interesting)

### Optimizations
#### Caching
- Colossus Metadata Caching
  - Data servers cache file handles avoiding one or more RPC
- Header Caching
  - Header/Footer of the files are cached, which contain critical index information
  - uses separate LRU
- Data Caching
  - DS cache columnar data is another cache
  - Artus is designed to have same structure in-memory and disk so no need for re-interpretation
  - Some derived information like bloom filters are also cached
  - since colossus files are immutable, cache coherency can be achieved by simply making sure that file names are not reused
- Meta data caching
  - self explanatory
- Affinity Scheduling
  - Caches are more effective when a server stores a small subset of data
  - so Procella makes sure that the data requests are sent to the DS which stores that subset with high probability ensuring high cache hit ratio
  - note that affinity is loose, even if a request is sent to a different DS, the DS can retrieve data from the durable storage, this provides high availability
#### Data Format
First implementation used Capacitor, which is designed for large scans ad-hoc workload, so a new format called Artus is developed, 
- Uses custom encoding, avoid generic ones like LZW, so seek to a single row can be done without a scan, which makes it suitable for small point lookups and range scans
- Multi pass adaptive encoding, parses once to collect lightweight information and uses this information to determine a suitable encoding
  - user can mention their objective function to optimize
  - dictionary encoding, indexer types, run length, delta etc are supported
- Chooses encoding that allows binary search for sorted columns, constant time seeks to a particular row number
  - trivial for indexed columns, fixed length will suffice, but for run length encoding a skip block is maintained after every B(variable) blocks
- Nested structures have similar encoding to parquet
  - I have skipped the details for now, details about parquet, orc, capacitor will be covered in the future
- Dictionary indexes, run length encoding etc are exposed to the query planner for more optimizations
- Rich meta data are stored in the file header like range, bloom filters etc
- inverted indexes are also supported
  - roaring bitmaps are used to store the indices

### Evaluation Engine
normally LLVM will be used to convert the execution plan to native code at query time, but this becomes bottleneck at high QPS, so a evaluation engine is created called Superluminal.
- C++ template meta programming is used extensively for code generation, can avoid large virtual call overhead
- Data is processed in blocks to take advantages of vectorized operations(intel's SIMD operations)
- Operates on the natively encoded data, and preserves the encoding whenever possible during functional operations
- Processed structures are fully columnar format
- Filters and projections are pushed downwards dynamically

### Partitioning and Indexing
- multi level partitioning and clustering is supported
- typically fact tables are partitioned on date
- dimensions are partitioned over key
- metadata server(MDS) has this information

### Distributed operations
#### Distributed Joins
The following joins are supported
- Broadcast
- Co-partitioned, the inner and outer table can be partitioned on the same key
- Shuffle, by the join key
- Pipelines, when the RHS is complex but likely to be small, then RHS is calculated and inlined in the query itself
- Remote Lookup
  - the build side(dimensional) table can be large but partitioned
  - the probe side(fact) table will be small but non-partitioned
  - then all the possible the join keys are sent to the DS containing the probe tables
  - the probe tables with the join keys are sent to the build tables
  - note that filters and projections can also be sent for further optimizations
#### Addressing Tail Latency
- The RS employs an effective backup strategy, it maintains quantiles of DS responses, if a query takes much more time than median then the request will be sent to another DS
- rate limiting and batching queries for the DS
- priority of queries is sent to DS
  - high for small queries and low for large queries
#### Intermediate Merging
For heavy aggregations, intermediate operators are induced into the plan so that the processing can happen in parallel along with the data retrieval

### Query Optimizations
#### Virtual Tables
A common approach to achieve better performance for high QPS  queries is materialized views, some approaches to it are
- Index aware aggregate selection
  - aware of partitioning and clustering, on the query predicate
- Stitched queries
  - stitch multiple tables with union or joins
- Lambda architecture awareness
  - Stitch(union) tables based on time column, so queries will be aware of both batch and stream tables
- Join awareness
#### Query Optimizer
- Rules based standard rewrites at query compile time
- adaptive techniques at query execution time
  - it is stronger than traditional cost based optimizations
  - simpler to implement
  - no need to maintain complex estimation models, which will be used infrequently
- Some stats are collected during shuffles, these will be used for later shuffles
- Adaptive Aggregations
  - partial aggregation is done on a subset of data before deciding the number of shards required for the whole aggregation
- Adaptive join
  - Some data structures to summarize the table information will be used
  - broadcast, size of the table
  - Pruning, bloom filter for one side can be used to prune the columns on the other side
  - Pre-Shuffled, if one side is already partitioned, then the other side is also partitioned on the same condition
  - Full Shuffle, Shards are determined based on the combined size
  - Adaptive Sorting
    - First estimate the number of rows to be sorted and determine the number of shards(n)
    - range partition them based on the n quantiles
    - locally sort on the shard
- Limitation of Adaptive optimizations
  - Works well for large queries
  - for small queries there will be a large overhead
  - so user can provide query hints for small queries
  - Join ordering is not yet supported

### Data Ingestion
Provides an offline tool to generate data in Procella accepted format, It is not necessary to use this

### Embedded Statistics
require millions of QPS for these queries with millisecond latency on a data at the scale of billions of records(funnily, the paper says that this size is "relatively small"), Procella solves this problem by running the instances in "stats serving" mode with specialized optimizations
- When new data is registered, Rgs notifies the data servers so that it can be loaded into memory, this ensures that it can served without disk access
- MDS is compiled into RS, this save RPC overhead between these sub-systems
- all metadata is fully preloaded, slightly stale meta data is acceptable, since the number of tables are small, this is accepted
- Query plans are aggressively cached, it can be done as these are highly predicable plans
- RS batches the queries and sends it to both primary and secondary and faster response is returned, this helps in tail latency
- RS and DS's are monitored for errors and latency fluctuations, the problem outlier tasks are moved to other machines
- Most of the expensive optimizations and operations are disabled

### Performance
you can look into the paper for these details
