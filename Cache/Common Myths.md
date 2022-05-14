#cache #cache_coherency #MESI

[Myths Programmers Believe about CPU Caches](https://software.rajivprab.com/2018/04/29/myths-programmers-believe-about-cpu-caches/)
many of the concepts learnt in cache-coherency are directly applicable to distributed-system-architecture and database-isolation-levels as well. For instance, understanding how coherency is implemented in hardware caches, can help in better understanding strong-vs-eventual consistency

#### Being Coherent
If different cores each have their own private cache, storing copies of the same data, wouldn’t that naturally lead to data mismatches as they start issuing writes? 
The answer: hardware caches on modern x86 CPUs like Intel’s, are kept in-sync with one another. These caches aren’t just dumb memory storage units, as many developers seem to think. Rather, there are very intricate protocols and logics, embedded in every cache, communicating with other caches, enforcing coherency across all threads.

*in-sync is [complicated](https://en.wikipedia.org/wiki/Consistency_model) term*

The most common protocol that’s used to enforce coherency amongst caches, is known as the [MESI protocol](https://en.wikipedia.org/wiki/MESI_protocol).
Each line of data sitting in a cache, is tagged with one of the following states:
1.  Modified (M)
    1.  This data has been modified, and differs from main memory
    2.  This data is the source-of-truth, and all other data elsewhere is stale
2.  Exclusive (E)
    1.  This data has not been modified, and is in sync with the data in main memory
    2.  No other sibling cache has this data
3.  Shared (S)
    1.  This data has not been modified, and is in sync with the data elsewhere
    2.  There are other sibling caches that (may) also have this same data
4.  Invalid (I)
    1.  This data is stale, and should never ever be used

