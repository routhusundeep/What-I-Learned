### [Transaction Isolation](https://www.postgresql.org/docs/9.5/transaction-iso.html)
#postgres #isolation

These are the available isolation levels in PG, and the types of phenomenon each of them avoid.  

| Isolation Level  | Dirty Reads  | Non-Repeatable Reads | Phantom Reads | Serialization Anomaly |
|------------------|--------------|----------------------|---------------|-----------------------|
| Read Uncommitted | Possible*    | Possible             | Possible      | Possible              |
| Read Committed   | Not Possible | Possible             | Possible      | Possible              |
| Repeatable read  | Not Possible | Possible             | Possible*     | Possible              |
| Serializable     | Not Possible | Possible             | Not Possible  | Not Possible          |
`* - is allowed in general as per the definition in ANSI SQL, but are not possible in PG


**** Brief description of the phenomenon are
- *Dirty Reads* - A transaction can read another uncommitted transactions writes
- *Non-Repeatable Reads* - A read done in a transaction cannot be repeated by another same read in the same transaction
- *Phantom Read* - The result of a query in a transaction cannot be repeated by another same query in the same transactions
- *Serialization Anomaly* - The resulting database state after executing a series of transactions is not similar to any of the states produced by sequentially executing them in any order.

#### Read Committed Isolation
- Internally *Read Uncommitted* is mapped to *Read Committed* as they provide same guarantees
- It is the default
- 

**** Key points to note                   
- RR is achieved by Snapshot isolation using *Multiversion Concurrency Control (MVCC)*. 
  - This is the reason Phantoms do not occur
  - *write skews* are possible in case of concurrent transactions, so Sequential consistency is not attainable
- *Serializable* is achieved by [[Serializable snapshot isolation]]
 