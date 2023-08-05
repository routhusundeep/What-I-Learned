#database #nosql

[paper](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.161.6751&rep=rep1&type=pdf)
[Apache Cassandra Site](https://cassandra.apache.org/doc/latest/cassandra/data_modeling/index.html)

I won't go through the API and data model. 

### Architecture
#### Partitioning
Cassandra partitions data across the cluster using [consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing) but uses an order preserving hash function to do so.
Each data item identified by a key is assigned to a node by hashing the data item’s key to yield its position on the ring, and then walking the ring clockwise to find the first node with a position larger than the item’s position. This node is deemed the coordinator for this key.
The basic consistent hashing algorithm presents some challenges. 
* Random position assignment of each node on the ring leads to non-uniform data and load distribution. 
* Basic algorithm is oblivious to the heterogeneity in the performance of nodes. 
Typically there exist two ways to address this issue: One is for nodes to get assigned to multiple positions in the circle (like in Dynamo), and the second is to analyze load information on the ring and have lightly loaded nodes move on the ring to alleviate heavily loaded nodes as described in. Cassandra opts for the latter, as it makes the design and implementation very tractable and helps to make very deterministic choices about load balancing.

#### Replication
Each data item is replicated at N hosts, where N is the replication factor configured “per-instance”.
Cassandra provides various replication policies such as “Rack Unaware”, “Rack Aware” (within a datacenter) and “Datacenter Aware”. Replicas are chosen based on the replication policy chosen by the application.
Cassandra's system elects a leader amongst its nodes using a system called Zookeeper.
The metadata about the ranges a node is responsible is cached locally at each node and in a fault-tolerant manner inside Zookeeper - this way a node that crashes and comes back up knows what ranges it was responsible for.

#### Membership
Within the Cassandra system Gossip is not only used for membership but also to disseminate other system related control state.

#### Failure Detection
Cassandra uses a modified version of the Φ Accrual Failure Detector
The idea of an Accrual Failure Detection is that the failure detection module doesn’t emit a Boolean value stating a node is up or down. Instead, the failure detection module emits a value which represents a suspicion level for each of the monitored nodes. This value is defined as Φ. The basic idea is to express the value of Φ on a scale that is dynamically adjusted to reflect network and load conditions at the monitored nodes

#### Bootstrapping
When a node starts for the first time, it chooses a random token for its position in the ring. For fault tolerance, the mapping is persisted to disk locally and also in Zookeeper. The token information is then gossiped around the cluster. This is how we know about all nodes and their respective positions in the ring.

#### Scaling the Cluster
The Cassandra bootstrap algorithm is initiated from any other node in the system by an operator using either a command line utility or the Cassandra web dashboard. The node giving up the data streams the data over to the new node using kernel copy techniques.

### Local Persistence
Uses MemTable, Bloom filter and SST, similar to [[BigTable]]

### Implementation Details
Read the paper



