[article](https://postgrespro.com/blog/pgsql/4161321) 

### Index structure
Hash functions in PostgreSQL always return the "integer" type, which is in the range of 232 ≈ 4 billion values. The number of buckets initially equals two and dynamically increases to adjust to the data size. 

$\mathit{bucket}=\mathit{hash(key)}\%\;\mathit{\#buckets}$
This is the bucket where we will put our TID. But this is insufficient, since TIDs matching different keys can be put into the same bucket. It is possible to store the source value of the key in a bucket, in addition to the TID, but this would considerably increase the index size. To save space, instead of a key, the bucket stores the hash code of the key.

When searching through the index, we compute the hash function for the key and get the bucket number. Now it remains to go through the contents of the bucket and return only matching TIDs with appropriate hash codes. This is done efficiently since stored "hash code - TID" pairs are ordered.

However, two different keys may happen not only to get into one bucket, but also to have the same four-byte hash codes - no one has done away with the collision. Therefore, the access method asks the general indexing engine to verify each TID by rechecking the condition in the table row (the engine can do this along with the visibility check).

### Mapping data structures to pages
If we look at an index as viewed by the buffer cache manager rather than from the perspective of query planning and execution, it turns out that all information and all index rows must be packed into pages. Such index pages are stored in the buffer cache and evicted from there exactly the same way as table pages.
- Meta page - page number zero, which contains information on what is inside the index.
- Bucket pages - main pages of the index, which store data as "hash code - TID" pairs.
- Overflow pages - structured the same way as bucket pages and used when one page is insufficient for a bucket.
- Bitmap pages - which keep track of overflow pages that are currently clear and can be reused for other buckets.
Note that hash index cannot decrease in size. If we delete some of indexed rows, pages once allocated would not be returned to the operating system, but will only be reused for new data after VACUUMING. The only option to decrease the index size is to rebuild it from scratch using REINDEX or VACUUM FULL command.


[README](https://github.com/postgres/postgres/tree/master/src/backend/access/hash)


