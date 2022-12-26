#postgres 

[link](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT)

### Heap Only Typles
The Heap Only Tuple (HOT) feature eliminates redundant index entries and allows the re-use of space taken by DELETEd or obsoleted UPDATEd tuples without performing a table-wide vacuum.  It does this by allowing single-page vacuuming, also called "defragmentation".

#### Challenges
- Page at a time vaccuum is impractical, because it costs a lot to figure out what which tuples can be recalimed
- Standard vaccuuming scans the index and amortizes the cost, which does not scale.
- In principle, one could recompute the index keys and do standard index searches
	- this is risky in the presence of possibly-buggy user-defined functions in functional indexes
	- We want to do vaccuuming without invoking user defined functions

**HOT solves it for a restricted but useful special case: where a tuple is repeatedly updated in ways that do not change its indexed columns.**
An additional property of HOT is that it reduces index size by avoiding the creation of identically-keyed index entries. This improves search speeds.

#### Update Chains With a Single Index Entry
- A new tuple placed on the same page and with all indexed columns the same as its parent row version does not get new index entries
- Only one index entry for the entire update chain
- An index-entry-less tuple is marked with the HEAP_ONLY_TUPLE flag.
- The prior row version is marked HEAP_HOT_UPDATED, and (as always in an update chain) its t_ctid field links forward to the newer version.
- Heap only tuple can be found by traversing the root pointer and since we restrict the HOT chain to lie within a single page, there won't be performance penalty.
- Eventually, the root pointer won't be visible to any transaction and can be reclaimed.
	- But it can't be removed since it is linked to HOT tuple 
	- So, it is handled by making the line pointer as redirecting-line-pointer which links to the HOT tuple
- If there is no more space in the page, then the chain ends and a new chain begins on the new page.

#### Abort Cases
- If heap-only tuples xmin is aborted, we can remove it
- If heap-only tuples xmax is aborted, we don't follow the chain
	- There might be race cases here, read the paper to understand them.

#### Index/Sequential Scans
When doing an index scan, whenever we reach a HEAP_HOT_UPDATED tuple whose xmax is not aborted, we need to follow its t_ctid link and check that entry as well; possibly repeatedly until we reach the end of the HOT chain. 
Sequential scans do not need to pay attention to the HOT links because they scan every line pointer on the page anyway. The same goes for a bitmap heap scan with a lossy bitmap.

#### Pruning
HOT pruning means updating line pointers so that HOT chains are reduced in length, by collapsing out line pointers for intermediate dead tuples.  Although this makes those line pointers available for re-use, it does not immediately make the space occupied by their tuples available.

#### Defragmentation
Centralizes unused space.  After we have converted root line pointers to redirected line pointers and pruned away any dead intermediate line pointers, the tuples they linked to are free space. But unless that space is adjacent to the central "hole" on the page (the pd_lower-to-pd_upper area) it cannot be used by tuple insertion. Defragmentation moves the surviving tuples to coalesce all the free space into one "hole".  This is done with the same PageRepairFragmentation function that regular VACUUM uses.

#### When can/should we prune or defragment?
A heuristic is being utilized, see the paper to understand it

#### VACUUM 
There is little change to regular vacuum.  It performs pruning to remove dead heap-only tuples, and cleans up any dead line pointers as if they were regular dead tuples.

#### CREATE INDEX AND CREATE INDEX CONCURRENTLY
Highly encourage you to read the paper.

[Useful link](https://www.2ndquadrant.com/en/blog/create-index-concurrently/) explains Create Index Concurrently

