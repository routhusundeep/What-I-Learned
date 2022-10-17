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
- ***Dirty Reads*** - A transaction can read another uncommitted transactions writes
- ***Non-Repeatable Reads*** - A read done in a transaction cannot be repeated by another same read in the same transaction
|Transaction 1| Transaction 2|
|-----------|--------------|
| **Query 1**:<br>SELECT * FROM users WHERE id = 1 | |
||**Query 2**:<br> UPDATE users SET age = 21 WHERE id = 1; COMMIT;|
|**Query 1**:<br>SELECT * FROM users WHERE id = 1; COMMIT;||

- ***Phantom Read*** - The result of a query in a transaction cannot be repeated by another same query in the same transactions
|Transaction 1| Transaction 2|
|-----------|--------------|
| **Query 1**:<br>SELECT * FROM users WHERE age BETWEEN 10 AND 30; | |
||**Query 2**:<br> INSERT INTO users(id, name, age) VALUES (3, 'Bob', 27); COMMIT;|
|**Query 1**:<br>SELECT * FROM users WHERE age BETWEEN 10 AND 30; COMMIT;||
- ***Serialization Anomaly*** - The resulting database state after executing a series of transactions is not similar to any of the states produced by sequentially executing them in any order.

#### Read Committed
- Internally *Read Uncommitted* is mapped to *Read Committed* as they provide same guarantees
- It is the default
- SELECT see's only committed data and also uncommitted data within the transaction
- UPDATE, DELETE etc. will see committed rows, if target row changes before commit, then the updated waits for the modifying transaction to commit or rollback.

#### Repeatable Reads
- only sees data committed before the transaction began; it never sees either uncommitted data or changes committed during transaction execution by concurrent transactions
- User should be prepared for failure
	- ERROR: could not serialize access due to concurrent update
- is achieved by Snapshot isolation using *Multiversion Concurrency Control (MVCC)*. 
	- This is the reason Phantoms do not occur
- *write skews* are possible in case of concurrent transactions, so Sequential consistency is not attainable

#### Serializable Isolation Level
- *Serializable* is achieved by [[Serializable snapshot isolation]]
- Should be prepared for failure
- uses _predicate locking_, which means that it keeps locks which allow it to determine when a write would have had an impact on the result of a previous read from a concurrent transaction, had it run first.
	- Do not block, so no deadlock
	- they are used to identifying impact and flag dependencies

For optimal performance when relying on Serializable transactions for concurrency control, these issues should be considered:
- Declare transactions as READ ONLY when possible.
- Control the number of active connections, using a connection pool if needed.
- Don't put more into a single transaction than needed for integrity purposes.
- Don't leave connections dangling "idle in transaction" longer than necessary.
- Eliminate explicit locks



 