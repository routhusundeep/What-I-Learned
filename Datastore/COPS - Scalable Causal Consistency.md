[paper](http://www.cs.cmu.edu/~dga/papers/cops-sosp2011.pdf) #datastore

Wow! This scratched an itch I didn't know I had.

### Introduction
We refer to systems with these four properties—Availability, low Latency, Partition-tolerance, and high Scalability—as ALPS systems.
Given that ALPS systems must sacrifice strong consistency (i.e., linearizability), we seek the strongest consistency model that is achievable under these constraints. Stronger consistency is desirable because it makes systems easier for a programmer to reason about. In this paper, we consider causal consistency with convergent conflict handling, which we refer to as causal+ consistency

> COPS - Clusters of Order-Preserving Servers

COPS executes all put and get operations in the local datacenter in a linearizable fashion, and it then replicates data across datacenters in a causal+ consistent order in the background.

We detail two versions of our COPS system. 
* The regular version, COPS, provides scalable causal+ consistency between individual items in the data store, even if their causal dependencies are spread across many different machines in the local datacenter. These consistency properties come at low cost: The performance and overhead of COPS is similar to prior systems, such as those based on log exchange, even while providing much greater scalability.
* We also detail an extended version of the system, COPS-GT, which also provides get transactions that give clients a consistent view of multiple keys. Get transactions are needed to obtain a consistent view of multiple keys, even in a fully-linearizable system. Our get transactions require no locks, are non-blocking, and take at most two parallel rounds of intra-datacenter requests. These transactions do come at some cost: compared to the regular version of COPS, COPS-GT is less efficient for certain workloads (e.g., write-heavy) and is less robust to long network partitions and datacenter failures.

The contributions in this paper include: 
* We explicitly identify four important properties of distributed data stores and use them to define ALPS systems. 
* We name and formally define causal+ consistency.
* We present the design and implementation of COPS, a scalable system that efficiently realizes the causal+ consistency model. 
* We present a non-blocking, lock-free get transaction algorithm in COPS-GT that provides clients with a consistent view of multiple keys in at most two rounds of local operations. 
* We show through evaluation that COPS has low latency, high throughput, and scales well for all tested workloads; and that COPS-GT has similar properties for common workloads.

### CAUSAL+ CONSISTENCY
Three rules define potential causality, denoted by $\rightarrow$
1. Execution Thread. If a and b are two operations in a single thread of execution, then $a \rightarrow b$ if operation a happens before operation b. 
2. Gets From. If a is a put operation and b is a get operation that returns the value written by a, then $a \rightarrow b$. 
3. Transitivity. For operations a, b, and c, if $a \rightarrow b$ and $b \rightarrow c$, then $a \rightarrow c$.

> We define causal+ consistency as a combination of two properties: causal consistency and convergent conflict handling.

#### Causal+ vs. Other Consistency Models
In decreasing strength, they include: linearizability (or strong consistency), which maintains a global, real-time ordering; sequential consistency, which ensures at least a global ordering; causal consistency, which ensures partial orderings between dependent operations; FIFO (PRAM) consistency, which only preserves the partial ordering of an execution thread, not between threads; per-key sequential consistency, which ensures that, for each individual key, all operations have a global order; and eventual consistency, a “catch-all” term used today suggesting eventual convergence to some type of agreement.

> The causal+ consistency we introduce falls between sequential and causal consistency

### System Design of COPS

#### Overview of COPS
* Each local COPS cluster is set up as a linearizable (strongly consistent) key-value store.
	* Linearizable is acceptable locally unlike WAN
* Each key space is partitioned into N separate linearizable partitions.
	* each of which can reside on a single node or a single chain of nodes
* replication between COPS clusters happens asynchronously to ensure low latency for client operations and availability in the face of external partitions
* System Components: Key-value store and Client library

Goals:
* Minimize overhead of consistency-preserving replication.
* (COPS-GT) Minimize space requirements.
* (COPS-GT) Ensure fast get trans operations

#### The COPS Key-Value Store
COPS must track the versions of written values, as well as their dependencies, in the case of COPS-GT.
For scalability, our implementation partitions the keyspace across a cluster’s nodes using consistent hashing
Every key stored in COPS has one primary node in each cluster. In practice, COPS’s consistent hashing assigns each node responsibility for a few different key ranges.
After a write completes locally, the primary node places it in a replication queue, from which it is sent asynchronously to remote equivalent nodes(in other datacenters)

#### Client Library and Interface
1. ctx_id ← createContext() 
2. bool ← deleteContext(ctx_id) 
3. bool ← put (key, value, ctx_id) 
4. value ← get (key, ctx_id) (In COPS) or 
5. {values} ← get_trans ({keys}, ctx_id) (In COPS-GT)
The client library in COPS-GT stores the client’s context in a table of $(key, version, deps)$ entries

#### Writing Values in COPS and COPS-GT
> ${bool,vers} ← put\textunderscore after (key, val, [deps], nearest, vers=\phi)$

##### Local
The put after operation ensures that val is committed to each cluster only after all of the entries in its dependency list have been written. In the client’s local cluster, this property holds automatically, as the local store provides linearizability
The primary storage node uses a [Lamport timestamp](https://en.wikipedia.org/wiki/Lamport_timestamp) to assign a unique version number to each update.

##### Between clusters
After a write commits locally, the primary storage node asynchronously replicates that write to its equivalent nodes
a node that receives a put after request from another cluster must determine if the value’s nearest dependencies have already been satisfied locally.
> $bool ← dep\textunderscore check (key, version)$


#### Reading Values in COPS
> $value, version, deps ← get\textunderscore by\textunderscore version (key, version=LATEST)$

#### Get Transactions in COPS-GT

Nice and simple Algorithm!
```python
# @param keys list of keys 
# @param ctx_id context id 
# @return values list of values 
function get_trans(keys, ctx_id): 
	# Get keys in parallel (first round) 
	for k in keys 
		results[k] = get_by_version(k, LATEST) 
	# Calculate causally correct versions (ccv) 
	for k in keys 
		ccv[k] = max(ccv[k], results[k].vers) 
		for dep in results[k].deps 
			if dep.key in keys 
				ccv[dep.key] = max(ccv[dep.key], dep.vers) 
	# Get needed ccvs in parallel (second round) 
	for k in keys 
		if ccv[k] > results[k].vers 
			results[k] = get_by_version(k, ccv[k]) 
	# Update the metadata stored in the context 
	update_context(results, ctx_id) 
	# Return only the values to the client 
	return extract_values(results)
```

### Garbage, Faults, and Conflicts
#### Garbage Collection Subsystem
##### Version Garbage Collection. (COPS-GT only)
To enable prompt garbage collection, COPS-GT limits the total running time of get trans through a configurable parameter, trans time (set to 5 seconds in our implementation). (If the timeout fires, the client library will restart the get trans call and satisfy the transaction with newer versions of the keys; we expect this to happen only if multiple nodes in a cluster crash.)
After a new version of a key is written, COPS-GT only needs to keep the old version around for trans time plus a small delta for clock skew.

##### Dependency Garbage Collection. (COPS-GT only)
COPS-GT can garbage collect these dependencies once the versions associated with old dependencies are no longer needed for correctness in get transaction operations
It can be cleaned after trans time seconds after a value has been committed in all datacenters,
Both COPS and COPS-GT can further set the value’s never-depend flag.

##### Client Metadata Garbage Collection. (COPS + COPS-GT)
As with the dependency tracking above, clients need to track dependencies only until they are guaranteed to be satisfied everywhere.
The client library will immediately remove a never-depend item from the list of dependencies in the client context. Furthermore, this process is transitive: Anything that a never-depend key depended on must have been flagged never-depend, so it too can be garbage collected from the context.
The COPS storage nodes remove unnecessary dependencies from put after operations. When a node receives a put after, it checks each item in the dependency list and removes items with version numbers older than a global checkpoint time. This checkpoint time is the newest Lamport timestamp that is satisfied at all nodes across the entire system. The COPS key-value store returns this checkpoint time to the client library

#### Fault Tolerance
Client failures can be ignored.

##### Key-Value Node Failures
Similar to the design of FAWN-KV, each data item is stored in a chain of R consecutive nodes along the consistent hashing ring. put after operations are sent to the head of the appropriate chain, propagate along the chain, and then commit at the tail, which then acknowledges the operation. get by version operations are sent to the tail, which responds directly.




 
 



