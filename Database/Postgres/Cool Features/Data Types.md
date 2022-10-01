Trivial types are not stated here.

### Range Types

PostgreSQL comes with the following built-in range types:

-   `int4range` — Range of `integer`, `int4multirange` — corresponding Multirange    
-   `int8range` — Range of `bigint`, `int8multirange` — corresponding Multirange
-   `numrange` — Range of `numeric`, `nummultirange` — corresponding Multirange
-   `tsrange` — Range of `timestamp without time zone`, `tsmultirange` — corresponding Multirange
-   `tstzrange` — Range of `timestamp with time zone`, `tstzmultirange` — corresponding Multirange
-   `daterange` — Range of `date`, `datemultirange` — corresponding Multirange

```SQL
CREATE TABLE reservation (room int, during tsrange);
INSERT INTO reservation VALUES
    (1108, '[2010-01-01 14:30, 2010-01-01 15:30)');

-- Containment
SELECT int4range(10, 20) @> 3;

-- Overlaps
SELECT numrange(11.1, 22.2) && numrange(20.0, 30.0);

-- Extract the upper bound
SELECT upper(int8range(15, 25));

-- Compute the intersection
SELECT int4range(10, 20) * int4range(15, 25);

-- Is the range empty?
SELECT isempty(numrange(1, 5));
```

### Timestamps
[This](https://phili.pe/posts/timestamps-and-time-zones-in-postgresql/) explains the differences between `timestamp` and `timezone with time zone` data types.

In a literal that has been determined to be `timestamp without time zone`, PostgreSQL will silently ignore any time zone indication. That is, the resulting value is derived from the date/time fields in the input value, and is not adjusted for time zone.

For `timestamp with time zone`, the internally stored value is always in UTC (Universal Coordinated Time, traditionally known as Greenwich Mean Time, GMT). An input value that has an explicit time zone specified is converted to UTC using the appropriate offset for that time zone. If no time zone is stated in the input string, then it is assumed to be in the time zone indicated by the system's [TimeZone](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-TIMEZONE) parameter, and is converted to UTC using the offset for the `timezone` zone.

When a `timestamp with time zone` value is output, it is always converted from UTC to the current `timezone` zone, and displayed as local time in that zone. To see the time in another time zone, either change `timezone` or use the `AT TIME ZONE` construct (see [Section 9.9.4](https://www.postgresql.org/docs/current/functions-datetime.html#FUNCTIONS-DATETIME-ZONECONVERT "9.9.4. AT TIME ZONE")).

### Enum
```SQL
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');
CREATE TABLE person (
    name text,
    current_mood mood
);
INSERT INTO person VALUES ('Moe', 'happy');
SELECT * FROM person WHERE current_mood = 'happy';
 name | current_mood
------+--------------
 Moe  | happy
```

### Geometric Types
Points, Lines, Segments, ... Polygons, Circles

### Network Types
inet, cidr, macaddr

### Text Search Types
#### tsvector
A `tsvector` value is a sorted list of distinct _lexemes_, which are words that have been _normalized_ to merge different variants of the same word. Sorting and duplicate-elimination are done automatically during input, as shown in this example:
```SQL
SELECT 'a fat cat sat on a mat and ate a fat rat'::tsvector;
                      tsvector
----------------------------------------------------
 'a' 'and' 'ate' 'cat' 'fat' 'mat' 'on' 'rat' 'sat'
```

#### tsquery
A `tsquery` value stores lexemes that are to be searched for, and can combine them using the Boolean operators `&` (AND), `|` (OR), and `!` (NOT), as well as the phrase search operator `<->` (FOLLOWED BY). There is also a variant ``<N>`` of the FOLLOWED BY operator, where `N` is an integer constant that specifies the distance between the two lexemes being searched for. `<->` is equivalent to `<1>`.

### UUID
The data type `uuid` stores Universally Unique Identifiers (UUID) as defined by [RFC 4122](https://tools.ietf.org/html/rfc4122), ISO/IEC 9834-8:2005, and related standards. (Some systems refer to this data type as a globally unique identifier, or GUID, instead.) This identifier is a 128-bit quantity that is generated by an algorithm chosen to make it very unlikely that the same identifier will be generated by anyone else in the known universe using the same algorithm. Therefore, for distributed systems, these identifiers provide a better uniqueness guarantee than sequence generators, which are only unique within a single database.

### JSON
JSON data types are for storing JSON (JavaScript Object Notation) data, as specified in [RFC 7159](https://tools.ietf.org/html/rfc7159). Such data can also be stored as `text`, but the JSON data types have the advantage of enforcing that each stored value is valid according to the JSON rules. 

PostgreSQL offers two types for storing JSON data: `json` and `jsonb`. To implement efficient query mechanisms for these data types, PostgreSQL also provides the `jsonpath`

The `json` and `jsonb` data types accept _almost_ identical sets of values as input. The major practical difference is one of efficiency. The `json` data type stores an exact copy of the input text, which processing functions must reparse on each execution; while `jsonb` data is stored in a decomposed binary format that makes it slightly slower to input due to added conversion overhead, but significantly faster to process, since no reparsing is needed. `jsonb` also supports indexing, which can be a significant advantage.

### ### Composite Types
```SQL
CREATE TYPE inventory_item AS (
    name            text,
    supplier_id     integer,
    price           numeric
);

CREATE TABLE on_hand (
    item      inventory_item,
    count     integer
);

INSERT INTO on_hand VALUES (ROW('fuzzy dice', 42, 1.99), 1000);

SELECT (on_hand.item).name FROM on_hand WHERE (on_hand.item).price > 9.99;
```

### Domain Type
A _domain_ is a user-defined data type that is based on another _underlying type_. Optionally, it can have constraints that restrict its valid values to a subset of what the underlying type would allow.
```SQL
CREATE DOMAIN posint AS integer CHECK (VALUE > 0);
CREATE TABLE mytable (id posint);
INSERT INTO mytable VALUES(1);   -- works
INSERT INTO mytable VALUES(-1);  -- fails
```


