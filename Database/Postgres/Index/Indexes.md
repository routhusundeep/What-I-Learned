#postgres #relational_database #index

[link](https://postgrespro.com/blog/pgsql/3994098)

In PostgreSQL, indexes are special database objects mainly designed to speed up data access. They are auxiliary structures: each index can be deleted and recreated back from the information in the table.
Despite all differences between types of indexes (also called access methods), each of them eventually associates a key (for example, the value of the indexed column) with table rows that contain this key. Each row is identified by TID (tuple ID), which consists of the number of block in the file and the position of the row inside the block.
It is important to understand that an index speeds up data access at a certain maintenance cost. For each operation on indexed data, whether it be insertion, deletion, or update of table rows, indexes for that table need to be updated too, and in the same transaction. Note that update of table fields for which indexes haven't been built does not result in index update; this technique is called HOT ([[Heap Only Tuples]]).

Extensibility entails some implications. To enable easy addition of a new access method to the system, an interface of the general indexing engine has been implemented. Its main task is to get TIDs from the access method and to work with them:
- Read data from corresponding versions of table rows.
- Fetch row versions TID by TID or in a batch using a prebuilt bitmap.
- Check visibility of row versions for the current transaction, taking into account its isolation level.

All the rest is the task of the access method:
- Implement an algorithm for building the index and map the data into pages (for the buffer cache manager to uniformly process each index).
- Search information in the index by a predicate in the form _indexed-field operator expression_.
- Evaluate the index usage cost.
- Manipulate the locks required for correct parallel processing.
- Generate write-ahead log (WAL) records.

### Indexing engine
Enables PostgreSQL to work with various access methods uniformly, but taking into account their features.

#### Main scanning techniques
##### Index scan
```SQL
postgres=# create table t(a integer, b text, c boolean); 
postgres=# insert into t(a,b,c) 
	select s.id, chr((32+random()*94)::integer), random() < 0.01 
	from generate_series(1,100000) as s(id) 
	order by random(); 
postgres=# create index on t(a); postgres=# analyze t;

postgres=# explain (costs off) select * from t where a = 1;
		  QUERY PLAN          
-------------------------------
 Index Scan using t_a_idx on t
   Index Cond: (a = 1)
(2 rows)
```

In this case, the optimizer decided to use _index scan_. With index scanning, the access method returns TID values one by one until the last matching row is reached. The indexing engine accesses the table rows indicated by TIDs in turn, gets the row version, checks its visibility against multiversion concurrency rules, and returns the data obtained.

##### Bitmap scan
``` SQL
postgres=# explain (costs off) select * from t where a <= 100;

             QUERY PLAN            
------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: (a <= 100)
   ->  Bitmap Index Scan on t_a_idx
         Index Cond: (a <= 100)
(4 rows)
```

The access method first returns all TIDs that match the condition (Bitmap Index Scan node), and the bitmap of row versions is built from these TIDs. Row versions are then read from the table (Bitmap Heap Scan), each page being read only once.

Note that in the second step, the condition may be rechecked (**Recheck Cond**). The number of retrieved rows can be too large for the bitmap of row versions to fully fit the RAM (limited by the _work_mem_ parameter). In this case, the bitmap is only built for pages that contain at least one matching row version. This "lossy" bitmap requires less space, but when reading a page, we need to recheck the conditions for each row contained there. Note that even for a small number of retrieved rows and therefore "exact" bitmap (such as in our example), the "Recheck Cond" step is represented in the plan anyway, although not actually performed.

``` SQL
postgres=# create index on t(b);
postgres=# analyze t;
postgres=# explain (costs off) select * from t where a <= 100 and b = 'a';

                    QUERY PLAN                    
--------------------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: ((a <= 100) AND (b = 'a'::text))
   ->  BitmapAnd
         ->  Bitmap Index Scan on t_a_idx
               Index Cond: (a <= 100)
         ->  Bitmap Index Scan on t_b_idx
               Index Cond: (b = 'a'::text)
(7 rows)
```

##### Sequential scan
``` SQL
postgres=# explain (costs off) select * from t where a <= 40000;

       QUERY PLAN      
------------------------
 Seq Scan on t
   Filter: (a <= 40000)
(2 rows)
```

The thing is that indexes work the better, the higher the condition selectivity, that is, the fewer rows match it. Growth of the number of retrieved rows increases overhead costs of reading index pages.

#### Covering indexes
As a rule, the main task of an access method is to return the identifiers of matching table rows for the indexing engine to read necessary data from these rows. But what if the index already contains all the data needed for the query? Such an index is called _covering_, and in this case, the optimizer can apply the _index-only scan_:

``` SQL
postgres=# vacuum t;
postgres=# explain (costs off) select a from t where a < 100;

             QUERY PLAN            
------------------------------------
 Index Only Scan using t_a_idx on t
   Index Cond: (a < 100)
(2 rows)
```

Indexes in PostgreSQL do not store information that enables us to judge the row visibility. Therefore, an access method returns versions of rows that match the search condition regardless of their visibility in the current transaction.

However, if the indexing engine needed to look into the table for visibility every time, this scanning method would not have been any different from a regular index scan.

To solve the problem, for tables PostgreSQL maintains a so-called _visibility map_ in which vacuuming marks the pages where data was not changed long enough for this data to be visible by all transactions regardless of the start time and isolation level. If the identifier of a row returned by the index relates to such a page, visibility check can be avoided.

Therefore, regular vacuuming increases efficiency of covering indexes. Moreover, the optimizer takes into account the number of dead tuples and can decide not to use the index-only scan if it predicts high overhead costs for the visibility check.

We can learn the number of forced accesses to a table using the EXPLAIN ANALYZE command:

```SQL

postgres=# explain (analyze, costs off) select a from t where a < 100;

                                  QUERY PLAN                                  
-------------------------------------------------------------------------------
 Index Only Scan using t_a_idx on t (actual time=0.025..0.036 rows=99 loops=1)
   Index Cond: (a < 100)
   Heap Fetches: 0 -- Mentions how many table reads were made
 Planning time: 0.092 ms
 Execution time: 0.059 ms
(5 rows)
```

#### NULL
For each access method, the developers make an individual decision whether to index NULLs or not. But as a rule, they do get indexed.

#### Indexes on several fields
``` SQL
postgres=# create index on t(a,b);
postgres=# analyze t;
postgres=# explain (costs off) select * from t where a <= 100 and b = 'a';

                   QUERY PLAN                  
------------------------------------------------
 Index Scan using t_a_b_idx on t
   Index Cond: ((a <= 100) AND (b = 'a'::text))
(2 rows)

postgres=# explain (costs off) select * from t where a <= 100;

              QUERY PLAN              
--------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: (a <= 100)
   ->  Bitmap Index Scan on t_a_b_idx
         Index Cond: (a <= 100)
(4 rows)
```

#### Indexes on expressions
``` SQL
postgres=# create index on t(lower(b));
postgres=# analyze t;
postgres=# explain (costs off) select * from t where lower(b) = 'a';

                     QUERY PLAN                    
----------------------------------------------------
 Bitmap Heap Scan on t
   Recheck Cond: (lower((b)::text) = 'a'::text)
   ->  Bitmap Index Scan on t_lower_idx
         Index Cond: (lower((b)::text) = 'a'::text)
(4 rows)
```

#### Partial indexes
Read the article

#### Sorting
``` SQL
postgres=# set enable_indexscan=off;
postgres=# explain (costs off) select * from t order by a;

     QUERY PLAN      
---------------------
 Sort
   Sort Key: a
   ->  Seq Scan on t
(3 rows)

postgres=# set enable_indexscan=on;
postgres=# explain (costs off) select * from t order by a;

          QUERY PLAN          
-------------------------------
 Index Scan using t_a_idx on t
(1 row)
```

#### Concurrent building
Usually building of an index acquires a SHARE lock for the table. This lock permits reading data from the table, but forbids any changes while the index is being built.
We can make sure of this if, say, during building of an index on the table "t", we perform the query below in another session:

``` SQL
postgres=# select mode, granted from pg_locks where relation = 't'::regclass;

   mode    | granted
-----------+---------
 ShareLock | t
(1 row)
```

If the table is large enough and extensively used for insertion, update, or deletion, this may appear to be inadmissible since modifying processes will wait for a lock release for a long time.

In this case, we can use concurrent building of an index.

```

postgres=# create index concurrently on t(a);
```

This command locks the table in the SHARE UPDATE EXCLUSIVE mode, which permits both reading and update (only changing the table structure is forbidden, as well as concurrent vacuuming, analysis, or building another index on this table).

However, there is also a flip side. First, the index will be built more slowly than usual since two passes across the table are done instead of one, and it is also necessary to be waiting for completion of parallel transactions that modify the data.

Second, with concurrent building of the index, a deadlock can occur, or unique constraints can be violated. However, the index will be built, although nonoperating. Such an index must be deleted and rebuilt. Nonoperating indexes are marked with the INVALID word in the output of the psql \d command, and the query below returns a complete list of those:

``` SQL
postgres=# select indexrelid::regclass index_name, indrelid::regclass table_name
from pg_index where not indisvalid;

 index_name | table_name
------------+------------
 t_a_idx    | t
(1 row)
```