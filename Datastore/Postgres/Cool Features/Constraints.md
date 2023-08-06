#postgres #sql

### Constraints
- Check Constraints
```SQL
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric **CHECK (price > 0)**
);
```
- Not-Null
- Unique
- Primary Keys
- Foreign Keys
- Exclusion Constraints
	- Ensure that if any two rows are compared on the specified columns or expressions using the specified operators, at least one of these operator comparisons will return false or null. The syntax is:
```SQL
CREATE TABLE circles (
    c circle,
    EXCLUDE USING gist (c WITH &&)
);
```