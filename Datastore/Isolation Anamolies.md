[blog](http://dbmsmusings.blogspot.com/2019/06/correctness-anomalies-under.html) #reread 
- P - Possible
- NP - Not Possible

System Guarantee              | Dirty Read | Non Repeatable Read | Phantom Read | Write Skew | Immortal Write | Stale read | Causal reverse 
------------ | ------------ | ------------------- | -------------| --------------------|--------------------|--------------------|--------------------
Read UnCommitted              | P          | P                   | P            | P          | P              | P          | P              |
Read Committed                | NP         | P                   | P            | P          | P              | P          | P              |
Repeatable read               | NP         | NP                  | P            | P          | P              | P          | P             
Snapshot Isolation            | NP         | NP                  | NP           | P          | P              | P          | P              |
Serializable                  | NP         | NP                  | NP           | NP         | P              | P          | P              |
Strong Write Serializable     | NP         | NP                  | NP           | NP         | NP             | P          | NP             |
Strong Partition Serializable | NP         | NP                  | NP           | NP         | NP             | NP         |P   
Strict Serializable           | NP         | NP                  | NP           | NP         | NP             | NP         | NP             |


###  Isolation levels

#### Read Uncommitted 
can read the data of uncommitted transactions

#### Read Committed
will read the data of only committed transactions

#### Repeatable reads
Two same reads in a single transaction will give the same result

#### Snapshot Isolation
all reads in a transaction will see a consistent snapshot of the database

#### Serializable
This isolation level specifies that all transactions occur in a completely isolated fashion; i.e., as if all transactions in the system had executed serially, one after the other.

#### Strong write Serializable / One-Copy Serializable / Strong Session Serializable
guarantee strict serializability for all transactions that insert or update data, but only one-copy serializability for read-only transactions. 

#### Strong Partition Serializable
strict serializability only on a per-partition basis. Data is divided into a number of disjoint partitions. Within each partition, transactions that access data within that partition are guaranteed to be strictly serializable. But otherwise, the system only guarantees one-copy serializability.

#### Strong Serializable
Relevant info on [Bailis blog](http://www.bailis.org/blog/linearizability-versus-serializability/) and the [actual Herlihy Paper](http://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf)

### Anomalies
#### Dirty Read
can read uncommitted data

#### Non Repeatable Read
A result of a read cannot be repeated

#### Phantom Read
A phantom read occurs when, in the course of a transaction, two identical queries are executed, and the collection of rows returned by the second query is different from the first.
A special case of Non Repeatable read, see the [Wiki Page](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Phantom_reads) for more information

#### Write Skew
two transactions (T1 and T2) concurrently read an overlapping data set (e.g. values V1 and V2), concurrently make disjoint updates (e.g. T1 updates V1, T2 updates V2), and finally concurrently commit, neither having seen the update performed by the other. Were the system serializable, such an anomaly would be impossible, as either T1 or T2 would have to occur "first", and be visible to the other.

#### Immortal Write
A write can never be modified, notice that it is very convenient strategy to maintain serializability

#### Stale Read
When you read an old data

#### Causal Reverse
A later write which was caused by an earlier write, time-travels to a point in the serial order prior to the earlier write

Notice that the last three anomalies can only be defined when we take real time of the events into consideration
