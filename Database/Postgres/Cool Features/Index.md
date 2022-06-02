### Types
#### B-Tree
B-trees can handle equality and range queries on data that can be sorted into some ordering. In particular, the PostgreSQL query planner will consider using a B-tree index whenever an indexed column is involved in a comparison using one of these operators:
`<   <=   =   >=   >`

Constructs equivalent to combinations of these operators, such as `BETWEEN` and `IN`, can also be implemented with a B-tree index search. Also, an `IS NULL` or `IS NOT NULL` condition on an index column can be used with a B-tree index.

#### Hash
Hash indexes store a 32-bit hash code derived from the value of the indexed column. Hence, such indexes can only handle simple equality comparisons. The query planner will consider using a hash index whenever an indexed column is involved in a comparison using the equal operator: `=`

#### GIST
GiST stands for Generalized Search Tree. It is a balanced, tree-structured access method, that acts as a base template in which to implement arbitrary indexing schemes. B-trees, R-trees and many other indexing schemes can be implemented in GiST. Supports operators:
`<<   &<   &>   >>   <<|   &<|   |&>   |>>   @>   <@   ~=   &&`
[This](https://www.postgresql.org/docs/9.3/functions-range.html) is the overview of these operators


#### SP-GIST
SP-GiST is an abbreviation for space-partitioned GiST. SP-GiST indexes, like GiST indexes, offer an infrastructure that supports various kinds of searches. SP-GiST permits implementation of a wide range of different non-balanced disk-based data structures, such as quadtrees, k-d trees, and radix trees (tries). As an example, the standard distribution of PostgreSQL includes SP-GiST operator classes for two-dimensional points, which support indexed queries using these operators:
`<<   >>   ~=   <@   <<|   |>>`

#### GIN
GIN indexes are “inverted indexes” which are appropriate for data values that contain multiple component values, such as arrays. An inverted index contains a separate entry for each component value, and can efficiently handle queries that test for the presence of specific component values. Supports operators:
`<@   @>   =   &&`

#### BRIN
BRIN indexes (a shorthand for Block Range Indexes) store summaries about the values stored in consecutive physical block ranges of a table. Thus, they are most effective for columns whose values are well-correlated with the physical order of the table rows. Supports:
`<   <=   =   >=   >`


### Multicolumn
Currently, only the B-tree, GiST, GIN, and BRIN index types support multiple-key-column indexes.

A multicolumn B-tree index can be used with query conditions that involve any subset of the index's columns, but the index is most efficient when there are constraints on the leading (leftmost) columns. The exact rule is that equality constraints on leading columns, plus any inequality constraints on the first column that does not have an equality constraint, will be used to limit the portion of the index that is scanned.

A multicolumn GiST index can be used with query conditions that involve any subset of the index's columns. Conditions on additional columns restrict the entries returned by the index, but the condition on the first column is the most important one for determining how much of the index needs to be scanned. A GiST index will be relatively ineffective if its first column has only a few distinct values, even if there are many distinct values in additional columns.

A multicolumn GIN index can be used with query conditions that involve any subset of the index's columns. Unlike B-tree or GiST, index search effectiveness is the same regardless of which index column(s) the query conditions use.

### Order By
By default, B-tree indexes store their entries in ascending order with nulls last (table TID is treated as a tiebreaker column among otherwise equal entries). This means that a forward scan of an index on column `x` produces output satisfying `ORDER BY x` (or more verbosely, `ORDER BY x ASC NULLS LAST`). The index can also be scanned backward, producing output satisfying `ORDER BY x DESC` (or more verbosely, `ORDER BY x DESC NULLS FIRST`, since `NULLS FIRST` is the default for `ORDER BY DESC`).
You can adjust the ordering of a B-tree index by including the options `ASC`, `DESC`, `NULLS FIRST`, and/or `NULLS LAST` when creating the index; for example:
```SQL
CREATE INDEX test2_info_nulls_low ON test2 (info NULLS FIRST);
CREATE INDEX test3_desc_index ON test3 (id DESC NULLS LAST);
```

### Combining Indexes
A single index scan can only use query clauses that use the index's columns with operators of its operator class and are joined with `AND`. For example, given an index on `(a, b)` a query condition like `WHERE a = 5 AND b = 6` could use the index, but a query like `WHERE a = 5 OR b = 6` could not directly use the index.

Fortunately, PostgreSQL has the ability to combine multiple indexes (including multiple uses of the same index) to handle cases that cannot be implemented by single index scans.

To combine multiple indexes, the system scans each needed index and prepares a _bitmap_ in memory giving the locations of table rows that are reported as matching that index's conditions. The bitmaps are then ANDed and ORed together as needed by the query. Finally, the actual table rows are visited and returned. The table rows are visited in physical order, because that is how the bitmap is laid out; this means that any ordering of the original indexes is lost, and so a separate sort step will be needed if the query has an `ORDER BY` clause. For this reason, and because each additional index scan adds extra time, the planner will sometimes choose to use a simple index scan even though additional indexes are available that could have been used as well.

### Unique Index
Indexes can also be used to enforce uniqueness of a column's value, or the uniqueness of the combined values of more than one column.
```SQL
CREATE UNIQUE INDEX name ON table (column [, ...]);
```

### Indexes on Expressions
```SQL
CREATE INDEX test1_lower_col1_idx ON test1 (lower(col1));
SELECT * FROM test1 WHERE lower(col1) = 'value';
```

### Partial Index
A _partial index_ is an index built over a subset of a table; the subset is defined by a conditional expression (called the _predicate_ of the partial index). The index contains entries only for those table rows that satisfy the predicate. Partial indexes are a specialized feature, but there are several situations in which they are useful.

One major reason for using a partial index is to avoid indexing common values. Since a query searching for a common value (one that accounts for more than a few percent of all the table rows) will not use the index anyway

### Index-Only Scans and Covering Indexes
Highly encourage you to read the [wiki](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)
All indexes in PostgreSQL are _secondary_ indexes, meaning that each index is stored separately from the table's main data area (which is called the table's _heap_ in PostgreSQL terminology). This means that in an ordinary index scan, each row retrieval requires fetching data from both the index and the heap. The heap-access portion of an index scan thus involves a lot of random access into the heap, which can be slow.
To solve this performance problem, PostgreSQL supports _index-only scans_, which can answer queries from an index alone without any heap access.
1. The index type must support index-only scans. B-tree indexes always do. GiST and SP-GiST indexes support index-only scans for some operator classes but not others.
2. The query must reference only columns stored in the index. For example, given an index on columns `x` and `y` of a table that also has a column `z`, these queries could use index-only scans:
```SQL
SELECT x, y FROM tab WHERE x = 'key'; -- works
SELECT x FROM tab WHERE x = 'key' AND y < 42; -- works

SELECT x, z FROM tab WHERE x = 'key'; -- does not work
SELECT x FROM tab WHERE x = 'key' AND z < 42; -- does not work
```

covering index:
```SQL
CREATE INDEX tab_x_y ON tab(x) INCLUDE (y);
```

### Operator classes
An index definition can specify an _operator class_ for each column of an index.
```SQL
CREATE INDEX name ON table (column opclass [ ( opclass_options ) ] [sort options] [, ...]);
```
he operator class identifies the operators to be used by the index for that column. For example, a B-tree index on the type `int4` would use the `int4_ops` class; this operator class includes comparison functions for values of type `int4`.

There are also some built-in operator classes besides the default ones:
- The operator classes `text_pattern_ops`, `varchar_pattern_ops`, and `bpchar_pattern_ops` support B-tree indexes on the types `text`, `varchar`, and `char` respectively.
[This](https://dba.stackexchange.com/questions/291248/is-there-a-difference-between-text-pattern-ops-and-collate-c/291250#291250) explains why collation is a better alternative.

