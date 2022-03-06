---
title: "Suboptimal PostgreSQL Queries"
date: 2022-03-05T14:03:38-08:00
draft: false
tags: ["postgresql"]
ShowToc: true
---

### Intro

Listed here are some *potentially* suboptimal PostgreSQL queries I've noticed in a professional setting or just looking at different queries over the years. Note not all are suboptimal under all conditions, and as always, you want to benchmark any queries yourself to confirm any issues. All queries and query problems heavily depend on your data, data distribution, statistics collected, Postgres configuration, and hardware. All queries were ran multiple times to make sure there were as few cache misses as possible and the median times were taken.

---

### Using DISTINCT in a subquery

**Problem:**
You are using a DISTINCT in a subquery.

```sql
  SELECT * FROM abc WHERE id IN (select DISTINCT id FROM def WHERE ...)
```

Ignoring whether you should be using *IN* over a more declarative query to begin with, what's the problem with this?

The issue arises because the *DISTINCT* inside the subquery will cause the rows to be aggregated. Depending on the circumstances, for large tables, this can be a HashAggregate. After this Postgres will do a semi-join, which may be a Hash Semi Join so essentially it will take a hash of the inner query rows twice! This can lead to worse results.

**Solution:**
Don't use *DISTINCT* in the subquery.

```sql
  SELECT * FROM abc WHERE id IN (select id FROM def WHERE ...)
```


> Distinct:
> ```sql
> explain (analyze, costs off) select * from abc where id in (select distinct id from def);
> -----
>  Hash Join (actual time=320.949..557.475 rows=632397 loops=1)
>    Hash Cond: (abc.id = def.id)
>    ->  Seq Scan on abc (actual time=0.015..29.369 rows=1000000 loops=1)
>    ->  Hash (actual time=319.842..319.843 rows=632397 loops=1)
>          Buckets: 1048576 (originally 524288)  Batches: 1 (originally 1)  Memory Usage: 32896kB
>          ->  HashAggregate (actual time=245.059..274.020 rows=632397 loops=1)
>                Group Key: def.id
>                Batches: 1  Memory Usage: 57361kB
>                ->  Seq Scan on def (actual time=0.007..34.256 rows=1000000 loops=1)
>  Planning Time: 0.242 ms
>  Execution Time: 575.312 ms
> ```

> No Distinct:
> ```sql
> explain (analyze, costs off) select * from abc where id in (select id from def);
> -----
> Hash Semi Join (actual time=136.224..401.620 rows=632397 loops=1)
>   Hash Cond: (abc.id = def.id)
>   ->  Seq Scan on abc (actual time=0.012..29.907 rows=1000000 loops=1)
>   ->  Hash (actual time=136.000..136.001 rows=1000000 loops=1)
>         Buckets: 1048576  Batches: 1  Memory Usage: 47255kB
>         ->  Seq Scan on def (actual time=0.007..38.704 rows=1000000 loops=1)
> Planning Time: 0.236 ms
> Execution Time: 415.249 ms
> ```

Note both HashAggregate and HashJoin in the *DISTINCT* query.

---

### Unnecessary self joins on a table

**Problem:**
You are trying to find some relationship within the same table, so you join the table with itself.
```sql
SELECT
  person.name,
  prev_person.name as prev_name,
  next_person.name as next_name
FROM person
LEFT JOIN person AS prev_person ON person.id=prev_person.id+1
LEFT JOIN person AS next_person ON person.id=next_person.id-1;
```
In this case, this will cause 3 sequential scans of the person table.

**Solution:** Use window functions if possible.

```sql
SELECT
  person.name,
  lag(name, 1) OVER (ORDER BY id) as prev_name,
  lead(name, 1) OVER (ORDER BY id) as next_name
FROM
  person;
```

> Self Join:
> ```sql
> explain (analyze, costs off) select person.name, prev_person.name as prev_name, next_person.name as next_name from person left join person prev_person on person.id=prev_person.id+1 left join person next_person on person.id=next_person.id-1;
> ----
> Hash Right Join (actual time=649.125..1002.096 rows=1000000 loops=1)
>   Hash Cond: ((next_person.id - 1) = person.id)
>   ->  Seq Scan on person next_person (actual time=0.023..45.816 rows=1000000 loops=1)
>   ->  Hash (actual time=648.042..648.043 rows=1000000 loops=1)
>         Buckets: 1048576  Batches: 2  Memory Usage: 48865kB
>         ->  Hash Right Join (actual time=195.389..525.978 rows=1000000 loops=1)
>               Hash Cond: ((prev_person.id + 1) = person.id)
>               ->  Seq Scan on person prev_person (actual time=0.010..45.682 rows=1000000 loops=1)
>               ->  Hash (actual time=195.072..195.073 rows=1000000 loops=1)
>                     Buckets: 1048576  Batches: 2  Memory Usage: 39452kB
>                     ->  Seq Scan on person (actual time=0.015..59.047 rows=1000000 loops=1)
> Planning Time: 0.239 ms
> Execution Time: 1027.209 ms
> ```

> Window Functions:
> ```sql
> explain (analyze, costs off) select person.name, lag(name, 1) over (order by id) as prev_name, lead(name, 1) over (order by id) as next_name from person;
> ---
> WindowAgg (actual time=0.036..340.874 rows=1000000 loops=1)
>   ->  Index Scan using person_pkey on person (actual time=0.022..59.962 rows=1000000 loops=1)
> Planning Time: 0.084 ms
> Execution Time: 361.980 ms
>```

---

### Using OR for the different fields on joined tables in WHERE clause

**Problem:**
You are trying to find rows in a joined table where a field equals XYZ in one table or a field equals ABC in the other table.
```sql
SELECT
  DISTINCT parent.*
FROM
  parent
LEFT JOIN
  child USING (parent_id)
WHERE
  child.name LIKE '456%'
  OR parent.name LIKE '456%';
```
(Note that the names are numbers because it's easier to generate a random name as numbers than using actual letters).

Unfortunately there is no good way to check this without doing a sequential scan on both tables.

**Solution:** Divide the queries on the child and the parent to make use of any indexes and combine / *UNION* the results.

```sql
SELECT
  parent.*
FROM
  parent
WHERE
  parent.name LIKE '456%'
UNION DISTINCT
SELECT
  parent.*
FROM
  parent
  LEFT JOIN child USING (parent_id)
WHERE
  child.name LIKE '456%';
```

> OR:
> ```sql
> explain (analyze, costs off) select distinct parent.* from parent left join child using (parent_id) where child.name like '456%' or parent.name like '456%';
> ----
> Hash Right Join (actual time=26.318..314.089 rows=2107 loops=1)
>   Hash Cond: (child.parent_id = parent.parent_id)
>   Filter: ((child.name ~~ '456%'::text) OR (parent.name ~~ '456%'::text))
>   Rows Removed by Filter: 997894
>   ->  Seq Scan on child (actual time=0.010..38.909 rows=1000000 loops=1)
>   ->  Hash (actual time=26.144..26.144 rows=100000 loops=1)
>         Buckets: 131072  Batches: 1  Memory Usage: 6702kB
>         ->  Seq Scan on parent (actual time=0.011..9.128 rows=100000 loops=1)
> Planning Time: 0.331 ms
> Execution Time: 314.338 ms
> ```

> UNION:
> ```sql
> explain (analyze, costs off) select parent.* from parent where parent.name like '456%' union distinct select parent.* from parent left join child using (parent_id) where child.name like '456%';
> ----
> HashAggregate (actual time=7.291..7.562 rows=1195 loops=1)
>   Group Key: parent.parent_id, parent.name
>   Batches: 1  Memory Usage: 209kB
>   ->  Append (actual time=0.047..6.632 rows=1203 loops=1)
>         ->  Bitmap Heap Scan on parent (actual time=0.046..0.170 rows=102 loops=1)
>               Filter: (name ~~ '456%'::text)
>               Heap Blocks: exact=94
>               ->  Bitmap Index Scan on parent_name_idx (actual time=0.025..0.026 rows=102 loops=1)
>                     Index Cond: ((name >= '456'::text) AND (name < '457'::text))
>         ->  Nested Loop (actual time=0.626..6.340 rows=1101 loops=1)
>               ->  Bitmap Heap Scan on child (actual time=0.614..2.235 rows=1101 loops=1)
>                     Filter: (name ~~ '456%'::text)
>                     Heap Blocks: exact=1030
>                     ->  Bitmap Index Scan on child_name_idx (actual time=0.310..0.311 rows=1101 loops=1)
>                           Index Cond: ((name >= '456'::text) AND (name < '457'::text))
>               ->  Index Scan using parent_pkey on parent parent_1 (actual time=0.003..0.003 rows=1 loops=1101)
>                     Index Cond: (parent_id = child.parent_id)
> Planning Time: 0.308 ms
> Execution Time: 7.688 ms
> ```

---

### Joining then aggregating (or vice versa)

**Problem:**
You *Join* then *Aggregate* a result when you should do the opposite.

This is a **very** situationally dependent issue that requires understanding your data distribution. For inner joins with aggregates, the order does not matter for the end result, but may cause different performance characteristics.

For this example, there are 10,000,000 click events in an event table, related to user events. There are 1,000,000 users. We want to find the name of the top 10 users with the most events.

```sql
SELECT
  someone.name,
  count(*)
FROM
  someone
JOIN event USING (someone_id)
GROUP BY
  someone_id
ORDER BY
  count(*) DESC LIMIT 10;
```

How many rows does this go through? Potentially 10,000,000 + 1,000,000 to generate the initial JOIN result. Since one of the tables, the `someone` table, has only unique values in the join clause, the maximum possible output rows is `max(10,000,000, 1,000,000)`. Then it aggregates the result of the JOIN so Postgres scans through 10,000,000 rows again.

```
total_rows_scanned = rows_scanned_from_join + rows_scanned_for_aggregate
total_rows_scanned = 10,000,000 + 1,000,000 + 10,000,000 =
                   = 21,000,000 rows
```

21 million rows.

**Solution:** Aggregate then join

```sql
SELECT
  someone.name,
  aggregated.count
FROM
  someone
  JOIN (
    SELECT
      someone_id,
      count(*)
    FROM
      event
    GROUP BY someone_id
    ORDER BY count(*) DESC LIMIT 10
  ) AS aggregated USING (someone_id)
ORDER BY aggregated.count DESC;
```

How many rows will this scan through? 10,000,000 rows to aggregate resulting in 1,000,000 rows which are then sorted and limited, resulting in 10 rows. Then it must join those 10 rows with the 1,000,000 users but at this point. Because there are only 10 rows for the output and an index available in the someone table, Postgres will prefer to do a Nested Loop, with the 10 rows as the outer loop and an index scan over users in the inner loop - 10 times to logarithmically search the 1,000,000 users.

```
total_rows_scanned = rows_from_event + rows_from_aggregate + rows_created_from_aggregated + rows_created_from_aggregated * log(num_rows_in_user)
total_rows_scanned = 10,000,000 + 1,000,000 + 10 + 10 * log2(1,000,000)
                   ~ 11,000,209 rows
```
11 million rows (Note I'm assuming a log base and branching factor, of 2, which is not accurate as the B-Tree used in Postgres' index has a variable branching factor and is usually much larger than 2 but this is good enough for this estimation).

> JOIN then Aggregate:
> ```sql
> explain (analyze, costs off) select someone.name from someone join event using (someone_id) group by someone_id order by count(*) desc limit 10;
> ----
> Limit (actual time=1500.977..1500.979 rows=10 loops=1)
>   ->  Sort (actual time=1500.976..1500.977 rows=10 loops=1)
>         Sort Key: (count(*)) DESC
>         Sort Method: top-N heapsort  Memory: 26kB
>         ->  GroupAggregate (actual time=0.045..1435.221 rows=999954 loops=1)
>               Group Key: someone.someone_id
>               ->  Merge Join (actual time=0.028..1024.564 rows=10000000 loops=1)
>                     Merge Cond: (someone.someone_id = event.someone_id)
>                     ->  Index Scan using someone_pkey on someone (actual time=0.008..64.638 rows=1000000 loops=1)
>                     ->  Index Only Scan using event_someone_id_idx on event (actual time=0.016..409.976 rows=10000000 loops=1)
>                           Heap Fetches: 0
> Planning Time: 0.371 ms
> Execution Time: 1501.023 ms
> ```

> Aggregate then Join:
> ```sql
> explain (analyze, costs off) select someone.name, aggregated.count from someone join (select someone_id, count(*) from event group by someone_id order by count(*) desc limit 10) as aggregated using (someone_id) order by aggregated.count desc;
> ----
> Nested Loop (actual time=889.050..889.080 rows=10 loops=1)
>   ->  Limit (actual time=889.034..889.036 rows=10 loops=1)
>         ->  Sort (actual time=889.033..889.034 rows=10 loops=1)
>               Sort Key: (count(*)) DESC
>               Sort Method: top-N heapsort  Memory: 25kB
>               ->  GroupAggregate (actual time=0.037..846.509 rows=999954 loops=1)
>                     Group Key: event.someone_id
>                     ->  Index Only Scan using event_someone_id_idx on event (actual time=0.025..441.831 rows=10000000 loops=1)
>                           Heap Fetches: 0
>   ->  Index Scan using someone_pkey on someone (actual time=0.004..0.004 rows=1 loops=10)
>         Index Cond: (someone_id = event.someone_id)
> Planning Time: 0.215 ms
> Execution Time: 889.117 ms
> ```

Note there are *MANY* cases where the *JOIN* then *AGGREGATE* is the preferable solution (this is the "default" as aggregates happen after JOINs anyways). For example, joining 5 rows from the user table to the event table then aggregating would be much faster if the number of matches in the event table is small. This all depends on the query and your data distribution.

Postgres already knows the distribution of existing tables and will try to rearrange JOINs in an optimal way, up to [join_collapse_limit](https://www.postgresql.org/docs/current/runtime-config-query.html "Config") tables - it just has less information about potential aggregations. The following query would likely be already optimized by Postgres as it will prefer to join with *another_table* and *someone* first, then use that result to join with the *event* table as it knows that order will result in the smallest intermediate tables.

```sql
-- example Postgres automatically choosing best JOIN order:
-- another_table x someone x event
SELECT
  someone.name,
  count(*)
FROM
  someone
JOIN event USING (someone_id)
JOIN another_table USING (someone_id)
WHERE
  another_table.field = 'some value that returns 5 rows'
GROUP BY
  someone_id
ORDER BY
  count(*) DESC LIMIT 10;
```

---

### Using NOT IN with a subquery

**Problem:**
You use a *NOT IN* clause to perform an anti-join.

```sql
select * from abc where abc_id not in (select def_id from def);
```

While a natural way to think of querying, it is almost always not the most optimal approach in Postgres. Besides the potential issue with [NULL values](https://wiki.postgresql.org/wiki/Don't_Do_This#Don.27t_use_NOT_IN "Postgres Wiki") and not returning the expected results, it is not as quick in Postgres as compared to other equivalent queries. The exact reason, I'm not sure, but I believe it is due to having special logic handling NULLs.

**Solution:** Use LEFT JOIN w/ NULL check or NOT EXISTS which produce the same anti-join execution plan.

```sql
SELECT
  abc.*
FROM
  abc
  LEFT JOIN def
  ON abc.abc_id=def.def_id
WHERE
  def.def_id IS NULL;
--
-- or
--
SELECT
  abc.*
FROM
  abc
WHERE NOT EXISTS (
  SELECT
    *
  FROM
    def
  WHERE 
    def.def_id=abc.abc_id
);
```

> NOT IN:
> ```sql
> explain (analyze, costs off) select * from abc where abc_id not in (select def_id from def);
> ----
> Seq Scan on abc (actual time=260.742..396.429 rows=567873 loops=1)
>   Filter: (NOT (hashed SubPlan 1))
>   Rows Removed by Filter: 432127
>   SubPlan 1
>     ->  Seq Scan on def (actual time=0.029..32.993 rows=1000000 loops=1)
> Planning Time: 0.246 ms
> Execution Time: 408.463 ms
> ```

> NOT EXISTS or LEFT JOIN with NULL:
> ```sql
> explain (analyze, costs off) select * from abc where not exists (select def_id from def where def_id=abc_id);
> explain (analyze, costs off) select abc.* from abc left join def on abc.abc_id=def.def_id where def.def_id is null;
> ----
> Hash Anti Join (actual time=135.047..290.661 rows=567873 loops=1)
>   Hash Cond: (abc.abc_id = def.def_id)
>   ->  Seq Scan on abc (actual time=0.022..25.532 rows=1000000 loops=1)
>   ->  Hash (actual time=133.934..133.934 rows=1000000 loops=1)
>         Buckets: 1048576  Batches: 1  Memory Usage: 47255kB
>         ->  Seq Scan on def (actual time=0.007..38.804 rows=1000000 loops=1)
> Planning Time: 0.376 ms
> Execution Time: 305.370 ms
> ```


---

### Unnecessary ORDER BY in a view

**Problem:**
You add an *ORDER BY* clause to your view.

This is an issue if you plan on sorting the view *afterwards*. This means Postgres has to do 2 sorts, the first one, the one in the view, is pointless as the result will have to be resorted.

**Solution:** Don't add in *ORDER BY* clause in your view

Don't add in *ORDER BY* clause in your view if you plan on sorting it by another field. Of course, if you only plan to use the view as is and need the sorting, then this is fine.

