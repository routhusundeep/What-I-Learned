[paper](http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/36632.pdf) #datastore #column_store #real_world_system 

Supports interactive analysis of very large data over shared clusters of commodity machines. Unlike traditional databases, it is capable of operating on in place nested data structures. It builds its architecture from web search serving tree, which pushes the query down the tree and rewrites it at every step. It executes the query natively without converting them into map reduce jobs. Lastly, and more importantly, it uses a novel column striped storage which tremendously reduces the amount of data read from secondary storage and provided cheaper compression which in-turn reduces the CPU cost.

The main contributions of this paper are
1. New columnar storage format which is the inspiration for [Parquet]([[http://parquet.apache.org/)
2. New query language and execution engine which operates efficiently on column-striped data.
3. How execution trees used in web search systems can be used for query processing
4. Experiments on huge data with the described system

### Data Model
It is based on strongly typed nested records which originated from [Protocol Buffers](https://developers.google.com/protocol-buffers)

### Nested Column Storage
The challenges addressed are 
1. lossless representation of record structure in a columnar format
2. fast encoding
3. efficient record assembly

#### Repetition and Definition Levels
Values alone do not convey the structure of a record. Repetition level and definition level concepts are used
**Repetition level:** It tells us at what repeated field in the fields path the value has repeated
**Definition level:** how many fields in a path could be undefined(optional or repeated) but are actually present in the record
**Encoding:**
  - stored as a set of blocks
  - each block will have definition level, repetition level and compressed value
  - Null's are not stored explicitly, as any definition level less than the number of repeated or optional fields in the path represents a null
  - Definition level is not stored for values always defined
  - repetition is stored only when required, for example definition level of 0 implies a repetition level of 0.

#### Splitting records into Columns
The algorithm is mentioned in the appendix of the paper

#### Record Assembly 
You can find this also in the appendix. This algorithm helps you to develops a good intuition about levels and their usage, I am surprised that it could be done so elegantly, brilliant.

### Query language
Out of scope of the paper, only a sample is described

#### Query Execution
##### Tree Architecture
- Multi level serving tree
- root level receives the query, gets the metadata and routes the query to the serving nodes
- The leaf level communicates with the storage layer
- All levels rewrite their received query

##### Query dispatcher
- Schedules queries as per their priority and balances the load
- critical for fault tolerance
- keeps track of query processing times of each tablet in the form of a histogram
- If a tablet takes a disproportional amount of time, it re-schedules on a different server
- at leaf node, each stripe is fetched asynchronously, the read-ahead cache typically achieves a 95% hit rate
- It honors the percentage of tablets to be scanned before returning a result

### Experiments
Read the paper to see the results

### Observations
- Scan based queries can be executed at interactive speeds on disk resident datasets of up to a trillion records
- Near linear scalability in the number of columns and servers is achievable for a system containing thousands of nodes
- map reduce is beneficial for columnar storage
- record assembly and parsing are expensive. Software layers should be optimized to directly consume columnar data
- map reduce and query processing can be used together
- in multi-user system, larger system can benefit from economies of scale
- if trading speed against accuracy can be traded, we can do it even by seeing large percentage of it
- The bulk of a web-scale dataset can be scanned fast. Getting to the last few percent within a tight time bound is hard

Code is written in 100K lines of C++, Python and Java code

