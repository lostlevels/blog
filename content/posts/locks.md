---
title: "PostgreSQL Lock Contention For Frequently Updated Rows"
date: 2022-02-20T20:16:37-08:00
draft: false
tags: ["postgresql"]
ShowToc: true
---

### Intro

A transaction that updates a row in Postgres acquires a *RowExclusiveLock* on the table it updates. The transaction will hold this lock until the end of the transaction where it will either commit or abort. No other transaction can update this row while this lock is held. This can potentially lead to lock contention for transactions that are trying to concurrently update the same row. How much impact does this have and what can you do to alleviate this if there is an issue?

### Setup

In order to simulate this, we will prepare a voting setup under two scenarios. One where lock contention can take place because votes increment a shared counter, and another where they can't at the expense of more rows stored.

The setup:

```sql
-- init.sql
CREATE TABLE vote (
  vote_id BIGINT PRIMARY KEY
);

CREATE TABLE person (
  person_id BIGINT PRIMARY KEY
);

CREATE TABLE vote_counter (
  vote_id BIGINT PRIMARY KEY REFERENCES vote(vote_id),
  counter BIGINT DEFAULT 0 NOT NULL
);
CREATE INDEX ON vote_counter(vote_id);

CREATE TABLE person_vote (
  person_id BIGINT REFERENCES person(person_id) NOT NULL,
  vote_id BIGINT REFERENCES vote(vote_id) NOT NULL
);
CREATE INDEX ON person_vote(person_id, vote_id);

INSERT INTO vote(vote_id)
  SELECT generate_series FROM generate_series(1, 4);

INSERT INTO vote_counter(vote_id, counter)
  SELECT generate_series, 0 FROM generate_series(1, 4);

INSERT INTO person(person_id)
  SELECT generate_series FROM generate_series(1, 10000);

```

We generate 10,000 different people who can choose among 4 different votes any number of times they wish for the purposes of this demo.

Of course, there are only a few different types of votes in this table. A more active table might have many different votes for different events at once with many votes at once. For simplicity, we will keep only 4 votes, assuming an event that is the only event happening at the time.

The actual script we'll run to update the votes will be:

#### Counter scenario
```sql
-- counter.sql
WITH id(vote_id) AS (
  SELECT floor(1 + random()*4)
)
UPDATE vote_counter SET counter = counter + 1 WHERE vote_id = (select vote_id from id);
```

#### Non-Counter scenario:
```sql
-- non-counter.sql
INSERT INTO
  person_vote(vote_id, person_id)
VALUES (
	floor(1 + random()*4),
	floor(1 + random()*10000)
);
```

### Benchmark

As I'm on an 8 core device, I will run the benchmark with 8 threads and 32 connections. Tuning connections is definitely system and workload dependent but the fine folks at [Cockroach Labs](https://www.cockroachlabs.com/docs/stable/connection-pooling.html#sizing-connection-pools "Cockroach Labs") have determined a **small** multiplier of your available cores offers the best throughput so as to prevent too much context switching, which from personal benchmarks, I agree with.

We will run the following commands each time and take the median of 5 tries. We will also take note of how much WAL is generated. We perform the following steps:

- Clear and initialize the **bench** database.
- Take note of the starting WAL location.
- Run the benchmark for 4 minutes.
- Note how much WAL was generated.

```bash
# counter
dropdb --if-exists bench && createdb bench && psql bench < init.sql
START_WAL="$(psql -qAt bench -c 'select pg_current_wal_insert_lsn()')"
pgbench -j 8 -T 240 -n -c 32 -f counter.sql bench
psql bench -c "select pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_insert_lsn(), '$START_WAL'))"

# non-counter
dropdb --if-exists bench && createdb bench && psql bench < init.sql
START_WAL="$(psql -qAt bench -c 'select pg_current_wal_insert_lsn()')"
pgbench -j 8 -T 240 -n -c 32 -f non-counter.sql bench
psql bench -c "select pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_insert_lsn(), '$START_WAL'))"
```

### Results

> Counter:
> ```pgbench -j 8 -T 240 -n -c 32 -f counter.sql bench```
> ```
> query mode: simple
> number of clients: 32
> number of threads: 8
> duration: 240 s
> number of transactions actually processed: 1988577
> latency average = 3.862 ms
> initial connection time = 23.109 ms
> tps = 8286.060484 (without initial connection time)
> WAL generated: 318 MB
> ```

> Non-Counter
> ```pgbench -j 8 -T 240 -n -c 32 -f non-counter.sql bench```
> ```
> query mode: simple
> number of clients: 32
> number of threads: 8
> duration: 240 s
> number of transactions actually processed: 10287528
> latency average = 0.747 ms
> initial connection time = 22.058 ms
> tps = 42863.040843 (without initial connection time)
> WAL generated: 4072 MB
> ```

![Image](/images/counter-noncounter.svg "Plot")

A few things here. We can see the throughput with non-counter is nearly 5x greater of that with a counter. It also generates much more WAL. The latency for the counter version is also nearly 5x longer, indicating some lock contention as each transaction waits for a lock to be released. Also interesting to note is how long it takes to establish a connection - over 20 milliseconds. This indicates why you should reuse a connection or use a pooler that does this for you whenever possible to avoid costly connection establishment.

Should you always avoid a counter? As with any technical decision, the answer is, it depends. It's very specific to your situation and requirements. Your table will be structured differently from this, with many more or many less columns along with different access patterns. As such, you may encounter situations where the counter is faster, and there are benefits to using a counter. It is simple and we get the count for "free" for each vote with the counter. With the non-counter, we'd have to aggregate it or precompute its value. It also generates a lot less WAL.

### Potential Improvements

Is there any way to use a counter and still maintain throughput similar to the non-counter version? We could batch the counts up and increment it periodically. It's also possible to have a counter of counters to limit lock contention. For example, each vote would have multiple "bins" that can be used to vote. If we had 10 bins, then each vote would have 10 rows in the vote_counter table, reducing lock contention. Let's try with a quick example. Let's assign 10 bins for each vote.

```sql
-- init.sql
DROP TABLE IF EXISTS vote_counter;
CREATE TABLE vote_counter (
  vote_id BIGINT REFERENCES vote(vote_id),
  bin_id BIGINT,
  counter BIGINT DEFAULT 0 NOT NULL,
  PRIMARY KEY (vote_id, bin_id)
);
INSERT INTO vote_counter(vote_id, bin_id, counter)
  SELECT floor((generate_series - 1) / 10) + 1, (generate_series % 10) + 1, 0 FROM generate_series(1, 40);

-- bin-counter.sql
WITH id(vote_id, bin_id) AS (
  SELECT floor(1 + random()*4), floor(1 + random()*10)
)
UPDATE vote_counter SET counter = counter + 1 WHERE (vote_id, bin_id) = (select vote_id, bin_id from id);
```

> Bin-Counter:
> ```pgbench -j 8 -T 240 -n -c 32 -f bin-counter.sql bench```
> ```
> query mode: simple
> number of clients: 32
> number of threads: 8
> duration: 240 s
> number of transactions actually processed: 7544260
> latency average = 1.018 ms
> initial connection time = 23.669 ms
> tps = 31435.414610 (without initial connection time)
> WAL generated: 946 MB.
> ```

![Image](/images/counter-bincounter.svg "Plot")

As we can see, the bin-counter improved throughput by nearly 4 times over the regular counter. It is still 30% slower than the non-counter approach but the WAL usage is less than 4 times the non-counter. Clearly there are tradeoffs with all approaches.

### Conclusion

Counters are simple way to implement voting or counts in Postgres but may encounter lock contention under heavy writes. There are different ways to alleviate this issue and only your own requirements and findings can determine if they are a good fit for you.
