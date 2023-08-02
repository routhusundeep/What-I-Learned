#index #postgres
[link](https://postgrespro.com/blog/pgsql/4161264)

```SQL
postgres=# select amname from pg_am;
 amname
--------
 heap
 btree
 hash
 gist
 gin
 spgist
 brin
(7 rows)
```

Although sequential scan can rightfully be referred to access methods, it is not on this list for historical reasons.

In PostgreSQL versions 9.5 and lower, each property was represented with a separate field of the "pg_am" table. Starting with version 9.6, properties are queried with special functions and are separated into several layers:

- `pg_index_column_has_property` ( _`index`_ `regclass`, _`column`_ `integer`, _`property`_ `text`) → `boolean`
	Tests whether an index column has the named property. Common index column properties are listed in [here](https://www.postgresql.org/docs/current/functions-info.html#FUNCTIONS-INFO-INDEX-COLUMN-PROPS "Table 9.72. Index Column Properties"). (Note that extension access methods can define additional property names for their indexes.)
- `pg_index_has_property` ( _`index`_ `regclass`, _`property`_ `text`) → `boolean`
	Tests whether an index has the named property. Common index properties are listed in [here](https://www.postgresql.org/docs/current/functions-info.html#FUNCTIONS-INFO-INDEX-PROPS "Table 9.73. Index Properties").
- `pg_indexam_has_property` ( _`am`_ `oid`, _`property`_ `text`) → `boolean`
	Tests whether an index access method has the named property. Access method properties are listed in [here](https://www.postgresql.org/docs/current/functions-info.html#FUNCTIONS-INFO-INDEXAM-PROPS "Table 9.74. Index Access Method Properties").

#### Index Access Method Properties
```SQL
postgres=# select a.amname, p.name, pg_indexam_has_property(a.oid,p.name) from pg_am a, unnest(array['can_order','can_unique','can_multi_col','can_exclude','can_include']) p(name) where a.amname = 'btree' order by a.amname;
 amname |     name      | pg_indexam_has_property
--------+---------------+-------------------------
 btree  | can_order     | t
 btree  | can_unique    | t
 btree  | can_multi_col | t
 btree  | can_exclude   | t
 btree  | can_include   | t
(5 rows)
```

|Name|Description|
|---|---|
|`can_order`|Does the access method support `ASC`, `DESC` and related keywords in `CREATE INDEX`?|
|`can_unique`|Does the access method support unique indexes?|
|`can_multi_col`|Does the access method support indexes with multiple columns?|
|`can_exclude`|Does the access method support exclusion constraints?|
|`can_include`|Does the access method support the `INCLUDE` clause of `CREATE INDEX`?|

#### Index Properties
```SQL
postgres=# select p.name, pg_index_has_property('t_a_idx'::regclass,p.name)                                                                                                                                           from unnest(array[                                                                                                                                                                                                           'clusterable','index_scan','bitmap_scan','backward_scan'                                                                                                                                                            ]) p(name);
     name      | pg_index_has_property
---------------+-----------------------
 clusterable   | t
 index_scan    | t
 bitmap_scan   | t
 backward_scan | t
(4 rows)
```
|Name|Description|
|----|-----|
|`clusterable`|Can the index be used in a `CLUSTER` command?|
|`index_scan`|Does the index support plain (non-bitmap) scans?|
|`bitmap_scan`|Does the index support bitmap scans?|
|`backward_scan`|Can the scan direction be changed in mid-scan (to support `FETCH BACKWARD` on a cursor without needing materialization)?|


#### Index Column Properties
```SQL
postgres=# select p.name,
postgres-#      pg_index_column_has_property('t_a_idx'::regclass,1,p.name)
postgres-# from unnest(array[
postgres(#        'asc','desc','nulls_first','nulls_last','orderable','distance_orderable',
postgres(#        'returnable','search_array','search_nulls'
postgres(#      ]) p(name);
		 
name        | pg_index_column_has_property
--------------------+------------------------------
 asc                | t
 desc               | f
 nulls_first        | f
 nulls_last         | t
 orderable          | t
 distance_orderable | f
 returnable         | t
 search_array       | t
 search_nulls       | t
(9 rows)
```

|Name|Description|
|-----|-----|
|`asc`|Does the column sort in ascending order on a forward scan?|
|`desc`|Does the column sort in descending order on a forward scan?|
|`nulls_first`|Does the column sort with nulls first on a forward scan?|
|`nulls_last`|Does the column sort with nulls last on a forward scan?|
|`orderable`|Does the column possess any defined sort ordering?|
|`distance_orderable`|Can the column be scanned in order by a “distance” operator, for example `ORDER BY col <-> constant` ?|
|`returnable`|Can the column value be returned by an index-only scan?|
|`search_array`|Does the column natively support `col = ANY(array)` searches?|
|`search_nulls`|Does the column support `IS NULL` and `IS NOT NULL` searches?|


### [Operator classes and families](https://www.postgresql.org/docs/current/indexes-opclass.html)
An index definition can specify an _operator class_ for each column of an index.

The operator class identifies the operators to be used by the index for that column. For example, a B-tree index on the type `int4` would use the `int4_ops` class; this operator class includes comparison functions for values of type `int4`. 
In practice the default operator class for the column's data type is usually sufficient. 
The main reason for having operator classes is that for some data types, there could be more than one meaningful index behavior. For example, we might want to sort a complex-number data type either by absolute value or by real part. We could do this by defining two operator classes for the data type and then selecting the proper class when making an index. 
The operator class determines the basic sort ordering (which can then be modified by adding sort options `COLLATE`, `ASC`/`DESC` and/or `NULLS FIRST`/`NULLS LAST`).

There are also some built-in operator classes besides the default ones:

-   The operator classes `text_pattern_ops`, `varchar_pattern_ops`, and `bpchar_pattern_ops` support B-tree indexes on the types `text`, `varchar`, and `char` respectively. The difference from the default operator classes is that the values are compared strictly character by ccharacter,rather than according to the locale-specific collation rules. This makes these operator classes suitable for use by queries involving pattern matching expressions (`LIKE` or POSIX regular expressions) when the database does not use the standard “C” locale. As an example, you might index a `varchar` column like this:

```SQL
postgres=# select opfname, opcname, opcintype::regtype from pg_opclass opc, pg_opfamily opf where opf.opfname = 'integer_ops' and opc.opcfamily = opf.oid and opf.opfmethod = ( select oid from pg_am where amname = 'btree' );

   opfname   | opcname  | opcintype
-------------+----------+-----------
 integer_ops | int2_ops | smallint
 integer_ops | int4_ops | integer
 integer_ops | int8_ops | bigint
(3 rows)
```


An operator class is actually just a subset of a larger structure called an _operator family_. In cases where several data types have similar behaviors, it is frequently useful to define cross-data-type operators and allow these to work with indexes. To do this, the operator classes for each of the types must be grouped into the same operator family. The cross-type operators are members of the family, but are not associated with any single class within the family.

``` SQL
postgres=# explain (costs off) select * from t where b like 'A%';

         QUERY PLAN          
-----------------------------
 Seq Scan on t
   Filter: (b ~~ 'A%'::text)
(2 rows)

postgres=# create index on t(b text_pattern_ops);
postgres=# explain (costs off) select * from t where b like 'A%';

                           QUERY PLAN                          
----------------------------------------------------------------
 Bitmap Heap Scan on t
   Filter: (b ~~ 'A%'::text)
   ->  Bitmap Index Scan on t_b_idx1
         Index Cond: ((b ~>=~ 'A'::text) AND (b ~<~ 'B'::text))
(4 rows)
```


Refer to the [catalog](https://postgrespro.com/docs/postgresql/9.6/catalogs) for detailed system tables.