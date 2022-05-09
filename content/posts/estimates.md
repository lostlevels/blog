---
title: "Bad Row Estimates In PostgreSQL"
date: 2022-05-08T21:28:13-07:00
draft: false
tags: ["postgresql"]
ShowToc: true
weight: 1
---

### Intro

PostgreSQL is an excellent open-source database. It has an advanced planner that can usually generate good execution plans. But there are cases where it may produce suboptimal results.

In order to estimate how many rows a query may return, Postgres relies on different statistics. They are available in the `pg_stats` view for each column of a table and the `pg_class` table and are updated when the *VACUUM* or *ANALYZE* commands are ran.

### Row Estimation

For equality conditions, (`WHERE field = something`). It relies on the `pg_stats.most_common_freqs` and `pg_stats.most_common_vals` fields. These are filled only if certain values appear to be more common than others - for example, they would not be filled for a unique field or a field that is the primary key. There will be up to [default_statistics_target](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-DEFAULT-STATISTICS-TARGET) (default 100) entries.

If a value is one of the most common values, (I'll call it MCV from here on), the formula for the estimate is simply, assuming the value is at the array's first index:

```
  estimated_rows = total_rows * selectivity
  estimated_rows = pg_class.reltuples * pg_stats.most_common_freqs[1]
```

For multiple *AND* conditions of different fields, the frequencies would be multiplied together, unless they are part of any extended statistics.

But what happens if the value is not part of the MCV? For simplicity, we will assume the table has no NULL values. PostgreSQL assumes an **even** distribution of the remaining values. So, in a 1,000,000 row table with the most frequent values taking up 900,000 rows, Postgres will assume the remaining 100,000 are evenly divided among the number of distinct values that don't include the MCV. Let's say there are 10,000 distinct values that are not part of the MCV. Then Postgres will estimate for each distinct value, there are `100,000 / 10,000 = 10` rows.

The formula for the estimate is:

```
  estimated_rows = total_rows * selectivity
  estimated_rows = pg_class.reltuples * (1 - sum(pg_stats.most_common_freqs)) / (pg_stats.n_distinct - array_length(pg_stats.most_common_vals, 1))
```

This can become a problem when distribution of the non MCV values is not uniform.

### Demo Setup

Let's create a single column table where the id grows pseudo-quadratically so the distribution is non uniform. We will create 1000 unique id's from `[1,1000]` (so *n_distinct* should be close to 1000) and each id occurs `pow(id, 1.5)` times. Using the default *default_statistics_target* of 100, id's `[901,1000]` *should* be in the MCV (though due to random sampling numbers with smaller count than 901 might be in it as well, but certainly it would be unusual to see anything lower than around 800 in the MCV field).

```sql
DROP TABLE IF EXISTS abc;
CREATE TABLE abc(abc_id BIGINT);

INSERT INTO abc(abc_id)
  SELECT
    base.id
  FROM
    generate_series(1, 1000) AS base(id)
  JOIN lateral (
    SELECT
      generate_series
    FROM
      generate_series(1, pow(base.id, 1.5))
  ) AS multiplier(multiplicity) ON TRUE
  ORDER BY random();

CREATE INDEX ON abc(abc_id);

VACUUM ANALYZE abc;
```

> A note on the `ORDER BY random()` - Due to the way Postgres samples a table when collecting statistics, correlated ids or ids grouped together may cause a worse `n_distinct` estimate than a random distribution, hence the need for order by. But it can also lead to a more efficient plan for certain queries as the same and similar values are closer together. But that's another post for another day.

Let's get an idea of how frequent each id is with a histogram.
![Image](/images/mcv-plot.svg "Plot")

### Analysis

As we can see, the distribution is uneven. Both ids inside the MCV and outside are unevenly distributed. Let's see what happens when we run a query with a value that is inside the MCV.

```sql
EXPLAIN ANALYZE SELECT * FROM abc WHERE abc_id=950;
---------------------------------------------------------------
 Index Only Scan using abc_id_idx on abc  (cost=0.43..525.40 rows=28284 width=8) (actual time=0.113..4.501 rows=29280 loops=1)
   Index Cond: (abc_id = 950)
```

Postgres estimated *28284* rows and the actual count is *29280* (Your results will vary a bit as the sampling Postgres takes is random). This is a good estimate, inside of an order of magnitude. Estimates will change with each *ANALYZE* due to the aforementioned random sampling.

Now let's try a value outside the MCV.
```sql
EXPLAIN ANALYZE SELECT * FROM abc WHERE abc_id=10;
---------------------------------------------------------------
 Index Only Scan using abc_id_idx on abc  (cost=0.43..210.31 rows=11307 width=8) (actual time=0.043..0.077 rows=31 loops=1)
   Index Cond: (abc_id = 10)
```

Postgres estimated 11307 rows but there were only 31. The estimate is off by nearly 3 orders of magnitude! What's going on here? Because Postgres assumes a uniform distribution for the non MCV values, it assumes that each non MCV value has the following number of rows:

```sql
WITH numerator(value) AS (
  SELECT
    1 - SUM(freq)
  FROM
    pg_stats,
    unnest(most_common_freqs) AS freq
  WHERE
    pg_stats.tablename = 'abc'
)
SELECT
  ROUND(pg_class.reltuples * numerator.value / (pg_stats.n_distinct - ARRAY_LENGTH(pg_stats.most_common_vals, 1))) AS estimated_rows
FROM
  numerator,
  pg_stats,
  pg_class
WHERE
  pg_class.relname = 'abc'
  AND pg_stats.tablename = 'abc'
  AND pg_stats.attname = 'abc_id';
------
= 11307
```

As we can see the estimate of 11307 rows is the exact same estimate as was in our `EXPLAIN ANALYZE`. So, for all non MCV values, the planner will assume it has that many rows. We can see in the graph, the horizontal line represents what the planner believes the number of rows each non MCV id contains.
![Image](/images/mcv-plot-estimate.svg "Plot")

What can we do to solve this? One way is to increase *statistics target* for our given table from the default of default_statistics_target so that more values are included in the MCV.

```
ALTER TABLE abc ALTER COLUMN abc_id SET STATISTICS 1000;
```

Another way is to be aware of these potential estimate issues and modify a query to perform a certain way (or even use the infamous [pg_hint_plan](https://github.com/ossc-db/pg_hint_plan "Hint Plan Extension") extension to force the planner into certain actions).

In our case here, the Index scan is still the optimal choice even with the bad estimate and the query runs quickly enough as each row is not large so many rows can fit on a single page. If we run the original query with the *BUFFERS* option:

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM abc WHERE abc_id=10;
---------------------------------------------------------------
 Index Only Scan using abc_id_idx on abc  (cost=0.43..210.31 rows=11307 width=8) (actual time=0.043..0.064 rows=31 loops=1)
   Index Cond: (abc_id = 10)
   Buffers: shared hit=5
```
We can see each row is only 8 bytes (`width=8`) and Postgres only had to read 5 buffers from the buffer cache (`shared hit=5`), each of which are around 8kB in size, so not much data has to be read.

However, these poor estimates can especially be suboptimal when JOINs are introduced with wider columns more representative of real-world data, resulting in Nested Loops when it underestimates the number of rows or Hash Joins when it overestimates it. More on that in the next post.

### Conclusion

Postgres has a robust optimizer that can usually create good estimates and execution plans. Care should be noted under the circumstances where this is not the case.
