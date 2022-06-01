[wiki](https://www.postgresql.org/docs/current/functions-datetime.html)

### Intervals
-   ‘1 day’::interval
-   ‘5 days’::interval
-   ‘1 week’::interval
-   ‘30 days’::interval
-   ‘1 month’::interval
...

```sql
SELECT *
FROM users
WHERE created_at >= now() - '1 week'::interval
```

### Date functions
if we wanted to find the count of users that signed up per week:
```SQL
SELECT date_trunc('week', created_at), 
       count(*)
FROM users
GROUP BY 1
ORDER BY 1 DESC;
```
This gives us a nice roll-up of how many users signed up each week. What’s missing here though is if you have a week that has no users. In that case because no users signed up there is no count of 0, it just simply doesn’t exist. If you did want something like this, you could generate some range of time and then do a cross join with it against users to see which week they fell into. To do this, first you’d generate a series of dates:
``` SQL
with weeks as (
  select week
  from generate_series('2017-01-01'::date, now()::date, '1 week'::interval) week
)

SELECT weeks.week,
       count(*)
FROM weeks,
     users
WHERE users.created_at > weeks.week
  AND users.created_at <= (weeks.week - '1 week'::interval)
GROUP BY 1
ORDER BY 1 DESC;
```

