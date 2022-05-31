#cache #cache_coherency #memory_consistency #too_advanced #not_needed

[A primer on Cache Coherency and Memory Consistency](https://www.morganclaypool.com/doi/pdf/10.2200/S00962ED2V01Y201910CAC049)

**CONSISTENCY (MEMORY CONSISTENCY,  MEMORY CONSISTENCY MODEL, OR MEMORY  MODEL)**
Consistency models define correct shared memory behavior in terms of loads and stores (memory reads and writes), without reference to caches or coherence

**COHERENCE (CACHE COHERENCE)**
Cache coherence is the uniformity of shared resource data that ends up stored in multiple local caches. When clients in a system maintain caches of a common memory resource, problems may arise with incoherent data, which is particularly the case with CPUs in a multiprocessing system. 

• Cache coherence does not equal memory consistency.
• A memory consistency implementation can use cache coherence as a useful “black box.”

**SEQUENTIAL CONSISTENCY**
It was first formalized by Lamport, who called a single processor (core) sequential if “the result of an execution is the same as if the operations had been executed in the order specified by the program.” He then called a multiprocessor sequentially consistent if “the result of any execution is the same as if the operations of all processors (cores) were executed in some sequential order, and the operations of each individual processor (core) appear in this sequence in the order specified by its program.”

**Formal Definition**
We adopt the formalism of Weaver and Germond —an axiomatic method to specify consistency—with the following notation: L(a) and S(a) represent a load and a store, respectively, to address a. Orders <p and <m define program and global memory order, respectively. Program order <p is a per-core total order that captures the order in which each core logically (sequen- tially) executes memory operations. Global memory order 
'<m' is a total order on the memory operations of all cores

1. All cores insert their loads and stores into the order <m respecting their program order, regardless of whether they are to the same or different addresses (i.e., a=b or a != b). There are four cases: 
	• If L(a) <p L(b) => L(a) <m L(b)  ||  Load->Load 
	• If L(a) <p S(b) => L(a) <m S(b)  || Load->Store 
	• If S(a) <p S(b) => S(a) <m S(b) || Store->Store 
	• If S(a) <p L(b) => S(a) <m L(b)  || Store->Load 
2. Every load gets its value from the last store before it (in global memory order) to the same address: Value of L(a) = Value of MAX <m {S(a) | S(a) <m L(a)}, where MAX <m denotes “latest in memory order.”

Strictly speaking, this is the safety property for SC implementations (do no harm). SC implementations should also have some liveness properties (do some good). Specifically, a store must become eventually visible to a load that is repeatedly attempting to load that location. This property, referred to as eventual write propagation, is typically ensured by the coherence protocol.