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