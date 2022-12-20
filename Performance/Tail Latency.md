#distributed_systems #easy_to_read #performance

[link](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext) 

Temporary high-latency episodes (unimportant in moderate-size systems) may come to dominate overall service performance at large scale.

#### Why Variability Exists?
- Shared Resources
- Daemon
	- Background daemons may use only limited resources on average but when scheduled can generate multi-millisecond hiccups
- Global resource sharing
	- Applications running on different machines might contend for global resources
- Maintenance activities
- Queuing
- Power Limits
- Garbage collection
- Energy management

#### Component-Level Variability Amplified By Scale
A common technique for reducing latency in large-scale online services is to parallelize sub-operations across many machines. Parallelization happens by fanning out a request from a root to numerous leaf servers. These sub-operations must all complete within a strict deadline for the service to feel responsive.

Over-provisioning of resources, careful real-time engineering of software, and improved reliability can all be used at all levels and in all components to reduce the base causes of variability.

#### Reducing Component Variability
**Differentiating service classes and higher-level queuing**
Differentiated service classes can be used to prefer scheduling requests for which a user is waiting over non-interactive requests. Keep low-level queues short so higher-level policies take effect more quickly; for example, the storage servers in Google's cluster-level file-system software keep few operations outstanding in the operating system's disk queue, instead maintaining their own priority queues of pending disk requests. This shallow queue allows the servers to issue incoming high-priority interactive requests before older requests for latency-insensitive batch operations are served.

**Reducing head-of-line blocking**
High-level services can handle requests with widely varying intrinsic costs. It is sometimes useful for the system to break long-running requests into a sequence of smaller requests to allow interleaving of the execution of other short-running requests; for example, Google's Web search system uses such time-slicing to prevent a small number of very computationally expensive queries from adding substantial latency to a large number of concurrent cheaper queries.

**Managing background activities and synchronized disruption**
Background tasks can create significant CPU, disk, or network load; examples are log compaction in log-oriented storage systems and garbage-collector activity in garbage-collected languages. A combination of throttling, breaking down heavyweight operations into smaller operations, and triggering such operations at times of lower overall load is often able to reduce the effect of background activities on interactive request latency.

#### Within Request Short-Term Adaptations
**Hedged requests**. A simple way to curb latency variability is to issue the same request to multiple replicas and use the results from whichever replica responds first. We term such requests "hedged requests" because a client first sends one request to the replica believed to be the most appropriate, but then falls back on sending a secondary request after some brief delay. The client cancels remaining outstanding requests once the first result is received.

**Tied requests**. The hedged-requests technique also has a window of vulnerability in which multiple servers can execute the same request unnecessarily. Permitting more aggressive use of hedged requests with moderate resource consumption requires faster cancellation of requests.
A common source of variability is queuing delays on the server before a request begins execution. For many services, once a request is actually scheduled and begins execution, the variability of its completion time goes down substantially. [The tail at scale](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext#R10) said allowing a client to choose between two servers based on queue lengths at enqueue time exponentially improves load-balancing performance over a uniform random scheme. We advocate not choosing but rather enqueing copies of a request in multiple servers simultaneously and allowing the servers to communicate updates on the status of these copies to each other. We call requests where servers perform cross-server status updates "tied requests." The simplest form of a tied request has the client send the request to two different servers, each tagged with the identity of the other server ("tied"). When a request begins execution, it sends a cancellation message to its counterpart. The corresponding request, if still enqueued in the other server, can be aborted immediately or de-prioritized substantially.

An alternative to the tied-request and hedged-request schemes is to probe remote queues first, then submit the request to the least-loaded server.It can be beneficial but is less effective than submitting work to two queues simultaneously for three main reasons: load levels can change between probe and request time; request service times can be difficult to estimate due to underlying system and hardware variability; and clients can create temporary hot spots by all clients picking the same (least-loaded) server at the same time.

#### Cross-Request Long-Term Adaptations
Although many systems try to partition data in such a way that the partitions have equal cost, a static assignment of a single partition to each machine is rarely sufficient in practice for two reasons: First, the performance of the underlying machines is neither uniform nor constant over time, for reasons (such as thermal throttling and shared workload interference) mentioned earlier. And second, outliers in the assignment of items to partitions can cause data-induced load imbalance (such as when a particular item becomes popular and the load for its partition increases).

**Micro-partitions**. To combat imbalance, many of Google's systems generate many more partitions than there are machines in the service, then do dynamic assignment and load balancing of these partitions to particular machines.

**Selective replication**. An enhancement of the micro-partitioning scheme is to detect or even predict certain items that are likely to cause load imbalance and create additional replicas of these items. Load-balancing systems can then use the additional replicas to spread the load of these hot micro-partitions across multiple machines without having to actually move micro-partitions. Google's main Web search system uses this approach, making additional copies of popular and important documents in multiple micro-partitions. 

**Latency-induced probation**. By observing the latency distribution of responses from the various machines in the system, intermediate servers sometimes detect situations where the system performs better by excluding a particularly slow machine, or putting it on probation. The source of the slowness is frequently temporary phenomena like interference from unrelated networking traffic or a spike in CPU activity for another job on the machine, and the slowness tends to be noticed when the system is under greater load.

#### Large Information Retrieval Systems
**Good enough**. In large IR systems, once a sufficient fraction of all the leaf servers has responded, the user may be best served by being given slightly incomplete results in exchange for better end-to-end latency.
for example, results from ads or spelling-correction systems are easily skipped for Web searches if they do not respond in time.

**Canary requests**. Another problem that can occur in systems with very high fan-out is that a particular request exercises an untested code path, causing crashes or extremely long delays on thousands of servers simultaneously. To prevent such correlated crash scenarios, some of Google's IR systems employ a technique called "canary requests"; rather than initially send a request to thousands of leaf servers, a root server sends it first to one or two leaf servers. The remaining servers are only queried if the root gets a successful response from the canary in a reasonable period of time.

#### Mutations
Tolerating latency variability for operations that mutate state is somewhat easier for a number of reasons: First, the scale of latency-critical modifications in these services is generally small. Second, updates can often be performed off the critical path, after responding to the user. Third, many services can be structured to tolerate inconsistent update models for (inherently more latency-tolerant) mutations. And, finally, for those services that require consistent updates, the most commonly used techniques are quorum-based algorithms (such as Lamport's Paxos); since these algorithms must commit to only three to five replicas, they are inherently tail-tolerant.

