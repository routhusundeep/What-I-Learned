```SQL
CREATE TABLE cities (
    name            text,
    population      float,
    elevation       int     -- in feet
);

CREATE TABLE capitals (
    state           char(2)
) INHERITS (cities);
```

In this case, the `capitals` table _inherits_ all the columns of its parent table, `cities`. State capitals also have an extra column, `state`, that shows their state.

``` SQL
SELECT name, elevation
    FROM cities
    WHERE elevation > 500;


   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
 Madison   |       845
```
This above query searches in capitals too.

```SQL
SELECT name, elevation
    FROM ONLY cities
    WHERE elevation > 500;

   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
```
`ONLY` Limits the search tables.


There are a lot of caveats, read the documentation for more details