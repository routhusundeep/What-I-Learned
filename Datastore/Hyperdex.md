[paper](https://cs.uwaterloo.ca/~bernard/hyperdex.pdf) #nosql  #datastore

### Approach
#### Data Model and API
As in a relational database, HyperDex objects match an application-provided schema that defines the typed attributes of the object and are persisted in tables.
Basic operations 
- get, put, and delete
- search operation enables a user to specify zero or more ranges for secondary attributes

#### Hyperspace Hashing
An object is mapped to a deterministic coordinate in space by hashing each of its attribute values to a location along the corresponding axis
The tessalation of the hyperspace into regions (called the hyperspace mapping), as well as the assignment of the regions to servers, is performed by a fault-tolerant coordinator
Insert and Delete operations are simple.

#### Search Queries
Clients maintain a copy of the hyperspace mapping, and use it to deterministically execute search operations. A client first maps the search into the hyperspace using the mapping.

### Data Partitioning
The drawback of coupling the dimensionality of hyperspace with the number of searchable attributes is that, for tables with many searchable attributes, the hyperspace can be very large since its volume grows exponentially with the number of dimensions. In general, a $D$ dimensional hyperspace will need $O(2^D)$ servers.

We avoid the problems associated with high-dimensionality by partitioning tables with many attributes into multiple lower-dimensional hyperspaces called subspaces.

#### Key Subspace
Provides efficient key-based operations by creating a one-dimensional subspace dedicated to the key.

#### Object Distribution Over Subspaces
Hyperdex does not normalize the data, and each subspace stores a copy of it.

### Consistency and Replication
#### Value Dependent Chaining
Very similar to chain replication.
The head of the chain is called the point leader, and is determined by hashing the object in the key subspace.
Each update flows from the point leader through the chain, and remains pending until an acknowledgement of that update is received from the next server in the chain. When an update reaches the tail, the tail sends an acknowledgement back through the chain in reverse so that all other servers may commit the pending update and clean up transient state.
Updates will have the older version key instances in the chain to remove the data.
Each server independently delays operations which depend upon a destructive operation until the destructive operation, and all that came before it, are acknowledged.

#### Fault Tolerance
New replicas are introduced at the tail of the region’s chain, and servers are bumped forward in the chain as other servers fail.

#### Server and Configuration Management
HyperDex utilizes a logically centralized coordinator to manage system-wide global state. The coordinator encapsulates the global state in the form of a configuration which consists of the hyperspace mapping between servers and regions and information about server failures.

#### Consistency Guarantees
the preceding protocol ensures strong guarantees for applications.
##### Key Consistency 
All actions which operate on a specific key (e.g., get and put) are linearizable, with all operations on all keys. This guarantees that all clients of HyperDex will observe updates in the same order. 

##### Search Consistency
HyperDex guarantees that a search will return all objects that were committed at the time of search. An application whose put succeeds is guaranteed to see the object in a future search. In the presence of concurrent updates, a search may return both the committed version, and the newly updated version of an object matching the search.



