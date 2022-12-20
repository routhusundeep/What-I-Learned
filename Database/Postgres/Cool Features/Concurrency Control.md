#postgres #sql
Internally, data consistency is maintained by using a multiversion model (Multiversion Concurrency Control, MVCC).
The main advantage of using the MVCC model of concurrency control rather than locking is that in MVCC locks acquired for querying (reading) data do not conflict with locks acquired for writing data, and so reading never blocks writing and writing never blocks reading. PostgreSQL maintains this guarantee even when providing the strictest level of transaction isolation through the use of an innovative _Serializable Snapshot Isolation_ (SSI) level.

### Transaction Isolation
Highly encourage you to read the [wiki](https://www.postgresql.org/docs/current/transaction-iso.html)
The phenomena which are prohibited at various levels are:

- **Dirty read:** A transaction reads data written by a concurrent uncommitted transaction.
- **Nonrepeatable read:** A transaction re-reads data it has previously read and finds that data has been modified by another transaction (that committed since the initial read).
- **Phantom read:** A transaction re-executes a query returning a set of rows that satisfy a search condition, and finds that the set of rows satisfying the condition has changed due to another recently-committed transaction.
- **Serialization anomaly:** The result of successfully committing a group of transactions is inconsistent with all possible orderings of running those transactions one at a time.
Isolation Level | Dirty Read | Nonrepeatable Read | Phantom Read | Serialization Anomaly
------------ | ------------ | ------------------- | -------------| --------------------
Read uncommitted | Allowed, but not in PG | Possible | Possible | Possible
Read committed   | Not possible | Possible | Possible | Possible
Repeatable read | Not possible | Not possible | Allowed, but not in PG | Possible
Serializable | Not possible | Not possible | Not possible | Not possible

```SQL
SET TRANSACTION transaction_mode [, ...]
SET TRANSACTION SNAPSHOT snapshot_id
SET SESSION CHARACTERISTICS AS TRANSACTION transaction_mode [, ...]

where transaction_mode is one of:

    ISOLATION LEVEL { SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED }
    READ WRITE | READ ONLY
    [ NOT ] DEFERRABLE
```

### Explicit Lock
#### Table level
```SQL
LOCK [ TABLE ] [ ONLY ] name [ * ] [, ...] [ IN lockmode MODE ] [ NOWAIT ]

where lockmode is one of:

    ACCESS SHARE | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE
    | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE
```

#### Row level
- **FOR UPDATE:** `FOR UPDATE` causes the rows retrieved by the `SELECT` statement to be locked as though for update.
- **FOR NO KEY UPDATE:** Behaves similarly to `FOR UPDATE`, except that the lock acquired is weaker: this lock will not block `SELECT FOR KEY SHARE` commands that attempt to acquire a lock on the same rows. This lock mode is also acquired by any `UPDATE` that does not acquire a `FOR UPDATE` lock.
- **FOR SHARE:** Behaves similarly to `FOR NO KEY UPDATE`, except that it acquires a shared lock rather than exclusive lock on each retrieved row.
- **FOR KEY SHARE:** Behaves similarly to `FOR SHARE`, except that the lock is weaker: `SELECT FOR UPDATE` is blocked, but not `SELECT FOR NO KEY UPDATE`.

#### Deadlocks
The best defense against deadlocks is generally to avoid them by being certain that all applications using a database acquire locks on multiple objects in a consistent order. In the example above, if both transactions had updated the rows in the same order, no deadlock would have occurred. One should also ensure that the first lock acquired on an object in a transaction is the most restrictive mode that will be needed for that object. If it is not feasible to verify this in advance, then deadlocks can be handled on-the-fly by retrying transactions that abort due to deadlocks.

