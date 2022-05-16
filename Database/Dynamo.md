#Amazon


[paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)

A highly available data storage technology that addresses the needs of a particular set of services. Dynamo has a simple key/value interface, is highly available with a clearly defined consistency window, is efficient in its resource usage, and has a simple scale out scheme to address growth in data set size or request rates.

### Background
#### System Assumption and Requirements  
- Query Model
  - State is stored as binary blobs identified by a unique key
  - No operations across multiple data items
  - objects are usually small ~1 MB
- ACID Properties
  - trading off consistency with availability
  - no isolation guarantees
  - single key updates
- Efficiency
  - configurable to achieve latency and throughput
- Other
  - environment is non-hostile
  - Each service has its own instance

 #### Design Considerations
- Priority on Availability
- Designed to be eventually consistent
- Write conflicts are resolved on read
  - the reason mentioned is, if they are resolved on writes then at each write all the replicas should be available which goes against the availability consideration
- conflict resolution is done by application
- Incremental scalability
  - scale out one node at a time
- Symmetry
  - every node has the same responsibilities
  - nodes are not distinguishable
- Decentralization
  - An extension of symmetry
  - centralized may lead to outages
- Heterogeneity
  - work distribution is dependent on the servers capabilities

### System Architecture
A nice nifty table is provided, stating the major problems, their solutions and reasons

Problem | Techique | Advantage
--------|--------|----------
Partitioning                       | Consistent Hashing                                     | Incremental Scalability                                               |
High Availability for writes       | Vector clocks with reconciliation during reads         | Version size is decoupled from update rates                           |
Handling Temporary failures        | Sloppy Quorums and Hinted Handoff                      | high availability when some replicas are not available                |
Recovering from permanent failures | Anti entropy using Merkel Trees                        | Synchronizes diverging replicas in the background                     |
Membership and failure detection   | Gossip based membership protocol and failure detection | no centralized registry with maintains membership and node liveliness |

#### System Interface
- get(key) and put(key, context, object)
  - context is system metadata of the object like version


#### Partitioning Algorithm
- [Consistent Hashing](https://www.toptal.com/big-data/consistent-hashing) is used, with the hash being the hash(MD5) of the key
- The number of virtual nodes(each point in the ring) assigned to a node is dependent on its capabilities

#### Replication
- data is replicated on multiple(N-configurable) nodes
- Each key(k) has a coordinator node derived from partitioning result
- on top of storing on coordinator, the data is replicated to the next N-1 distinct nodes on the ring
- each node can determine this preference list for any key

#### Data Versioning
- uses [vector clock](https://en.wikipedia.org/wiki/Vector_clock) to capture causality between different versions of the same object
- clock truncation scheme is used to limit vector size, the oldest entry in the vector based on the timestamp is omitted once the number of entries increases beyond a certain threshold
  - this case never occurred in production
- certain failure modes will result in the system having several versions of the data, so conflict resolution is necessary

#### Execution of get and put
- for Both get and put, the client can choose between two strategies to determine the node
  - use a generic load balancer that selects the node based on load information
    - avoids link to Dynamo code
    - once a random node gets the request, it will forward it to appropriate node based on the preference list of that key
  - use a partition aware library to find the node
    - lower latency
- node handling the read/write is called a coordinator
- nodes are given priority based on the order in which they are present in the preference list
- in case of failures, the nodes after N must also be queried
- configurable R and W
- R - min. number of nodes that must participate in read operation
- W - min. number of nodes that must participate in write operation
- R + W > N yields a quorum like system
- for put, coordinator generates a vector clock and broadcasts it to N nodes, if at least W nodes respond then the write is considered successful
- for get, coordinator requests from N nodes, then waits for R responses before returning the response (all versions) to the client

#### Handling failure, Hinted Handoffs
- traditional quorum will be unavailable during server failures and network partitions
- all reads and writes are done on the first N healthy nodes on the preference list
- if a write is forwarded to a non-replica, then it stores it as a hint with metadata containing the actual replica, this hint is forwarded to the actual replica once its comes back alive
- after transfer, it may delete the hinted object

#### Handling Permanent failures: Replica Synchronization
- Hinted replicas may be unavailable before synchronizing, which makes the replica states divergent
- Merkel tree is used to converge the replicas

#### Membership and Failure Detection
- Ring Membership
  - Only Administrator can add or delete the ring nodes through a command line tool or browser
  - the node that serves the request writes the change persistently and propagates it through a gossip protocol
- External Discovery
  - To avoid logical partitions, some seed nodes are present
  - seeds are discovered through some external mechanism and are known to all nodes
- Failure Detection
  - no decentralized failure detection, local detection will suffice
  - a node A will consider node B inoperative, if B does not respond to A

#### Adding/Removing Storage Nodes
- When a new node is added, it takes its key ranges from other nodes
- this will result in a uniform load on other nodes
- similarly when a node is removed

#### Implementation
Three main components
- local persistence engine
  - supported engines are Berkeley Database(BDB), MYSQL, in-memory buffer with persistent backing
- Request Coordinator
  - built on event driven messaging substrate similar to SEDA architecture
  - java nio is used
  - a state machine for each request
- membership

#### Experiences and Lessons learned
- Main Patterns used are
  - Business Logic Specific Reconciliation
  - Timestamp Based Reconciliation
  - High Performance Reads (W=N, R=1)
- Common (N,R,W) = (3,2,2)

#### Balancing Performance and Durability
- write latencies are higher than reads
- Dynamo provides a memory write + async/periodic durable write engine
- coordinator chooses a single persistent replica
- trades off durability for performance

#### Ensuring Uniform Load Distribution
- different partitioning strategies are implemented and tested on production
- read the paper to get a better picture

#### Divergent version: When and how many
- can happen in two scenarios
  - when system is facing failures
  - system is handling large number of concurrent writes
- during 24 hours, 99.94% of requests saw one version, 0.00057% saw 2 versions, 0.00047% saw 3 versions

#### Client driven or Server driven Coordination
- client driven (pull based partitioning information retrieval) works better than server driven due to obvious reasons
- see the paper for concrete numbers

#### Balancing background vs foreground tasks
- background tasks are given a particular number of time slices through an admission control mechanism
