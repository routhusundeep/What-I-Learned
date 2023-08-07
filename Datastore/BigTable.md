[paper](https://github.com/papers-we-love/papers-we-love/blob/main/datastores/bigtable-a-distributed-storage-system-for-structured-data.pdf)

#real_world_system #datastore 

I have read it a couple of times. A classic!


### Data Model 
A Bigtable is a sparse, distributed, persistent multidimensional sorted map. The map is indexed by a row key, column key, and a timestamp; each value in the map is an uninterpreted array of bytes.
> (row:string, column:string, time:int64) → string

#### Rows
The row keys in a table are arbitrary strings (currently up to 64KB in size, although 10-100 bytes is a typical size for most of our users).
Bigtable maintains data in lexicographic order by row key. The row range for a table is dynamically partitioned. Each row range is called a tablet, which is the unit of distribution and load balancing. As a result, reads of short row ranges are efficient and typically require communication with only a small number of machines

#### Columns Families
Column keys are grouped into sets called column families, which form the basic unit of access control. All data stored in a column family is usually of the same type (we compress data in the same column family together). A column family must be created before data can be stored under any column key in that family; after a family has been created, any column key within the family can be used. 
A column key is named using the following syntax: family:qualifier

#### Timestamps
Each cell in a Bigtable can contain multiple versions of the same data; these versions are indexed by timestamp. Bigtable timestamps are 64-bit integers.

### API
``` c++
// Open the table Table 
*T = OpenOrDie("/bigtable/web/webtable"); 
// Write a new anchor and delete an old anchor 
RowMutation r1(T, "com.cnn.www"); 
r1.Set("anchor:www.c-span.org", "CNN"); 
r1.Delete("anchor:www.abc.com"); 
Operation op; 
Apply(&op, &r1);
```

```c++
Scanner scanner(T); 
ScanStream *stream; 
stream = scanner.FetchColumnFamily("anchor"); 
stream->SetReturnAllVersions(); 
scanner.Lookup("com.cnn.www"); 
for (; !stream->Done(); stream->Next()) {
	printf("%s %s %lld %s\n", 
		scanner.RowName(), 
		stream->ColumnName(), 
		stream->MicroTimestamp(), 
		stream->Value()); 
}
```

### Building Blocks
The Google SSTable file format is used internally to store Bigtable data. An SSTable provides a persistent, ordered immutable map from keys to values, where both keys and values are arbitrary byte strings. Operations are provided to look up the value associated with a specified  key, and to iterate over all key/value pairs in a specified key range.
A lookup can be performed with a single disk seek: we first find the appropriate block by performing a binary search in the in-memory index, and then reading the appropriate block from disk. Optionally, an SSTable can be completely mapped into memory, which allows us to perform lookups and scans without touching disk.

Bigtable relies on a highly-available and persistent distributed lock service called Chubby. A Chubby uses the Paxos algorithm.

Bigtable uses Chubby for a variety of tasks: to ensure that there is at most one active master at any time; to store the bootstrap location of Bigtable data; to discover tablet servers and finalize tablet server deaths; to store Bigtable schema information (the column family information for each table); and to store access control lists. If Chubby becomes unavailable for an extended period of time, Bigtable becomes unavailable

### Implementation
The Bigtable implementation has three major components: a library that is linked into every client, one master server, and many tablet servers.
The master is responsible for assigning tablets to tablet servers, detecting the addition and expiration of tablet servers, balancing tablet-server load, and garbage collection of files in GFS.
Each tablet server manages a set of tablets (typically we have somewhere between ten to a thousand tablets per tablet server).

**Highly recommend reading the paper**

### Refinements
#### Locality Groups
Clients can group multiple column families together into a locality group. A separate SSTable is generated for each locality group in each tablet. Segregating column families that are not typically accessed together into separate locality groups enables more efficient reads.

#### Compression
Clients can control whether or not the SSTables for a locality group are compressed, and if so, which compression format is used. The user-specified compression format is applied to each SSTable block (whose size is controllable via a locality group specific tuning parameter).

### Caching for read performance
To improve read performance, tablet servers use two levels of caching. The Scan Cache is a higher-level cache that caches the key-value pairs returned by the SSTable interface to the tablet server code. The Block Cache is a lower-level cache that caches SSTables blocks that were read from GFS.

#### Bloom filters
We reduce the number of accesses by allowing clients to specify that Bloom filters [7] should be created for SSTables in a particular locality group. A Bloom filter allows us to ask whether an SSTable might contain any data for a specified row/column pair

#### Commit-log implementation
If we kept the commit log for each tablet in a separate log file, a very large number of files would be written concurrently in GFS. Depending on the underlying file system implementation on each GFS server, these writes could cause a large number of disk seeks to write to the different physical log files. In addition, having separate log files per tablet also reduces the effectiveness of the group commit optimization, since groups would tend to be smaller. To fix these issues, we append mutations to a single commit log per tablet server, co-mingling mutations for different tablets in the same physical log file

#### Speeding up tablet recovery
If the master moves a tablet from one tablet server to another, the source tablet server first does a minor compaction on that tablet.

#### Exploiting immutability
SST are immutable.






