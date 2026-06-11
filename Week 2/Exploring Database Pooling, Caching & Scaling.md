As part of my backend engineering roadmap, this week I explored database optimization.

In modern applications, performance bottlenecks often come from how we manage database connections, how frequently we access the database, and how efficiently we store and retrieve frequently used data. Concepts like connection pooling and caching exist not because databases are slow, but because resources are limited and applications need ways to use them efficiently.

The goal isn't to provide a complete guide to database optimization. Instead, it's to document what I learned, the assumptions that turned out to be wrong, and the ideas that helped me better understand how backend systems behave under load.

# Connection Pool

So imagine this, you have a store and you are handling it alone and all of a sudden 100 customers came all together, would you be able to handle all of them alone?? You had to manage 1st customer, then 2nd, then 3rd and it'll take hours or maybe more, so what you'll do?? Most probably you call a few of your family members for help or maybe hire a few workers.

Similarly if you simply create a client, and perform queries using that, it would be like a single person handling multiple requests and users have to wait a lot. So what you can do is create a pool of active connections(hire a group of people for your store) and the application can take up one of the active connections and perform their queries. Lets test and see how much performance difference does both the techniques will have:

```cmd
npm init -y
npm install express pg autocannon
```

```js
import { Pool } from "pg";
import { Client } from "pg";
import express from "express";
const app = express();
const port = 3000;
const pool = new Pool({
  max: 2000,
  idleTimeOutMillis: 5000,
  connectionTimeoutMillis: 2000,
  connectionString:
    "postgres://postgres:mysecretpassword@localhost:5432/postgres",
});

app.get("/generalclient", async (req, res) => {
  try {
    const client = await new Client({
      connectionString:
        "postgres://postgres:mysecretpassword@localhost:5432/postgres",
    });
    await client.connect();
    const response = await client.query(
      "SELECT * FROM video_game_sales LIMIT 5;",
    );
    client.end();
    res.send("Hello World");
  } catch (error) {
    console.log(error.message);
  }
});

app.get("/pool", async (req, res) => {
  try {
    const client = await pool.connect();
    const response = await client.query(
      "SELECT * FROM video_game_sales LIMIT 5;",
    );
    client.release();
    res.send("Hello World");
  } catch (error) {
    console.log(error.message);
  }
});
```

```cmd
<!-- NOW WE TEST THE APIs BY BOMBARDING THEM WITH REQUEST -->
THE BELOW COMMAND WILL SEND 1000 PARALLEL REQUESTS AND TEST THE API FOR THE DURATION OF 20 SECONDS
npx autocannon -c 1000 -d 20 http://localhost:3000/generalclient
```

```
Running 20s test @ http://localhost:3000/generalclient
1000 connections


┌─────────┬────────┬─────────┬─────────┬─────────┬────────────┬────────────┬──
────────┐
│ Stat    │ 2.5%   │ 50%     │ 97.5%   │ 99%     │ Avg        │ Stdev      │ M
ax      │
├─────────┼────────┼─────────┼─────────┼─────────┼────────────┼────────────┼──
────────┤
│ Latency │ 696 ms │ 2275 ms │ 9823 ms │ 9928 ms │ 4131.14 ms │ 3082.64 ms │ 1
0166 ms │
└─────────┴────────┴─────────┴─────────┴─────────┴────────────┴────────────┴──
────────┘
┌───────────┬─────┬──────┬─────────┬─────────┬─────────┬─────────┬─────────┐
│ Stat      │ 1%  │ 2.5% │ 50%     │ 97.5%   │ Avg     │ Stdev   │ Min     │
├───────────┼─────┼──────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ Req/Sec   │ 0   │ 0    │ 23      │ 127     │ 48.05   │ 48.74   │ 23      │
├───────────┼─────┼──────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ Bytes/Sec │ 0 B │ 0 B  │ 5.47 kB │ 30.2 kB │ 11.4 kB │ 11.6 kB │ 5.47 kB │
└───────────┴─────┴──────┴─────────┴─────────┴─────────┴─────────┴─────────┘

Req/Bytes counts sampled once per second.
# of samples: 20

3k requests in 23.64s, 229 kB read
1k errors (1k timeouts)
```

As we can see, it was able to handle 3K requests and out of those 1K requests timed-out. You can also see a table with 50%, 97.5%, 99%, what does that mean??

- 50p (50th percentile) latency. Also known as the median latency. This means that 50% of your queries or requests are faster than this time, and 50% are slower.
- 97.5p (97.5th percentile) latency. This indicates that 97.5% of your queries or requests are faster than this time, and 2.5% are slower.
- 99p (99th percentile) latency. This shows that 99% of your queries or requests are faster than this time, and 1% are slower.
  Now lets test the pool method

```
npx autocannon -c 1000 -d 20 http://localhost:3000/pool
```

```cmd
Running 20s test @ http://localhost:3000/pool
1000 connections


┌─────────┬────────┬─────────┬─────────┬─────────┬───────────┬───────────┬────
─────┐
│ Stat    │ 2.5%   │ 50%     │ 97.5%   │ 99%     │ Avg       │ Stdev     │ Max
     │
├─────────┼────────┼─────────┼─────────┼─────────┼───────────┼───────────┼────
─────┤
│ Latency │ 888 ms │ 1055 ms │ 2294 ms │ 2519 ms │ 1186.4 ms │ 345.48 ms │ 266
6 ms │
└─────────┴────────┴─────────┴─────────┴─────────┴───────────┴───────────┴────
─────┘
┌───────────┬─────┬──────┬────────┬────────┬────────┬─────────┬────────┐
│ Stat      │ 1%  │ 2.5% │ 50%    │ 97.5%  │ Avg    │ Stdev   │ Min    │
├───────────┼─────┼──────┼────────┼────────┼────────┼─────────┼────────┤
│ Req/Sec   │ 0   │ 0    │ 849    │ 1,123  │ 829.35 │ 272.82  │ 478    │
├───────────┼─────┼──────┼────────┼────────┼────────┼─────────┼────────┤
│ Bytes/Sec │ 0 B │ 0 B  │ 202 kB │ 268 kB │ 197 kB │ 64.9 kB │ 114 kB │
└───────────┴─────┴──────┴────────┴────────┴────────┴─────────┴────────┘

Req/Bytes counts sampled once per second.
# of samples: 20

18k requests in 20.42s, 3.95 MB read
```

We can clearly see that using pool increases the number of requests the db can handle by a lot.

Now you might have one question, if having bigger pools lets db handle more requests simultaneously then why dont we just set the max property in the pool to the highest we can, maybe a million, or more, but first answer this, would you hire millions of workers in your store?? I guess the answer would be no because then it would be very crowded and no one would be able to do their work properly.

Similarly if u create a very large pool of connections, postgres would have to use lot of compute power in managing those connections, and the queries would become slower. So what is the optimal pool size??
Most people say, a few hundreds, but i dont think there is a exact value for this question, I would suggest to run a few tests with different pool sizes and find the perfect one for your application. Lets test the above code with different pool sizes

```cmd
<!-- TESTING WITH POOL SIZE 2000 -->
npx autocannon -c 1000 -d 20 http://localhost:3000/pool
```

```cmd
Running 20s test @ http://localhost:3000/pool
1000 connections


┌─────────┬───────┬────────┬─────────┬─────────┬──────────┬───────────┬───────
──┐
│ Stat    │ 2.5%  │ 50%    │ 97.5%   │ 99%     │ Avg      │ Stdev     │ Max
  │
├─────────┼───────┼────────┼─────────┼─────────┼──────────┼───────────┼───────
──┤
│ Latency │ 89 ms │ 204 ms │ 3078 ms │ 3748 ms │ 477.4 ms │ 744.96 ms │ 5203 m
s │
└─────────┴───────┴────────┴─────────┴─────────┴──────────┴───────────┴───────
──┘
┌───────────┬─────┬──────┬────────┬────────┬────────┬─────────┬─────────┐
│ Stat      │ 1%  │ 2.5% │ 50%    │ 97.5%  │ Avg    │ Stdev   │ Min     │
├───────────┼─────┼──────┼────────┼────────┼────────┼─────────┼─────────┤
│ Req/Sec   │ 0   │ 0    │ 509    │ 1,098  │ 562.9  │ 351.81  │ 184     │
├───────────┼─────┼──────┼────────┼────────┼────────┼─────────┼─────────┤
│ Bytes/Sec │ 0 B │ 0 B  │ 121 kB │ 261 kB │ 134 kB │ 83.7 kB │ 43.8 kB │
└───────────┴─────┴──────┴────────┴────────┴────────┴─────────┴─────────┘

Req/Bytes counts sampled once per second.
# of samples: 20

13k requests in 20.63s, 2.68 MB read
1k errors (965 timeouts)
```

```cmd
<!-- TESTING WITH POOL SIZE 10000 -->
npx autocannon -c 1000 -d 20 http://localhost:3000/pool
```

```cmd
Running 20s test @ http://localhost:3000/pool
1000 connections


┌─────────┬──────┬──────┬───────┬──────┬──────┬───────┬──────┐
│ Stat    │ 2.5% │ 50%  │ 97.5% │ 99%  │ Avg  │ Stdev │ Max  │
├─────────┼──────┼──────┼───────┼──────┼──────┼───────┼──────┤
│ Latency │ 0 ms │ 0 ms │ 0 ms  │ 0 ms │ 0 ms │ 0 ms  │ 0 ms │
└─────────┴──────┴──────┴───────┴──────┴──────┴───────┴──────┘
┌───────────┬─────┬──────┬─────┬───────┬─────┬───────┬─────┐
│ Stat      │ 1%  │ 2.5% │ 50% │ 97.5% │ Avg │ Stdev │ Min │
├───────────┼─────┼──────┼─────┼───────┼─────┼───────┼─────┤
│ Req/Sec   │ 0   │ 0    │ 0   │ 0     │ 0   │ 0     │ 0   │
├───────────┼─────┼──────┼─────┼───────┼─────┼───────┼─────┤
│ Bytes/Sec │ 0 B │ 0 B  │ 0 B │ 0 B   │ 0 B │ 0 B   │ 0 B │
└───────────┴─────┴──────┴─────┴───────┴─────┴───────┴─────┘

Req/Bytes counts sampled once per second.
# of samples: 19

17k requests in 20.82s, 0 B read
16k errors (0 timeouts)
```

As you can see with a pool size of 200, the db was able to handle 13k requests and 1k returned error and while trying out with the pool size of 2000, the application crashed, so **bigger pool != better performance**

# Caching

So now you have multiple workers in your store, but suppose you are getting a lot of customers for a particular item, would you send your worker again and again to get that item?? No, what you can do is send your worker once and ask him to bring that item in bulk at once maybe 8 or 9 and keep it near the counter, lets say in a big box near the counter(or a cache), so its easily accessible right.

Caching is very similar, once you get your data from db, you save it in-memory storage or a local storage for easy and fast access. There are multiple ways of saving cache, lets discuss about them one by one

## Cache Aside Pattern

It is a caching strategy where the application directly manage the data flow between the cache and the primary db.

**How it works**

1. Application checks the cache first for the data it needs.
2. If the data is present in the cache, it return the data(it is also called a "hit").
3. If the data is not present(also called "miss"), the application then asks the db for it and also saves the received data in the cache.

## Read Through Pattern

It is a caching strategy where the application treats the cache as the primary data store.

**How it works**

1. The application requests data directly from the cache.
2. If the data exists, the cache returns it instantly to the application.
3. If the data is missing, the cache layer itself fetches the data from the database.
4. The cache stores the retrieved data for future requests and returns it to the application.

## Write Through

It is a caching strategy where the data is written to the cache and the database simultaneously.

**How it works**

1. The application sends a write or update request to the cache layer.
2. The cache layer immediately writes the data to the underlying database.
3. Once the database acknowledges the write, the cache updates its own copy and returns a success confirmation to the application.

## Write Back

A caching strategy where the application writes data directly to a cache, receives an immediate success acknowledgment, and lets the cache asynchronously persist the data to a database in the background

**How it works**

1. The application sends data to the cache. The cache saves it, and an acknowledgment is returned to the user almost instantly.
2. Database updates happen in batches or on schedule — occasionally

## Write Around

A caching strategy where write operations update the primary database directly while entirely bypassing the cache. The cache is only updated later on demand, when a read request occurs and experiences a cache miss.

1. When the application needs to save or update data, it writes directly to the database layer. The cache is intentionally ignored.
2. When an application subsequently requests the newly written data, it first checks the cache.
3. Because the cache was bypassed on write, this results in a "cache miss". The system then retrieves the data from the database, updates the cache with the retrieved value, and serves it to the user.

# Cache Eviction and Invalidation

Now that we know strategies to save cache we also need to learn how to remove or update cache

## Least Recent Used(LRU)

LRU evicts the item that hasn’t been accessed for the longest time.

**How it Works**

- Read (Get): Moves an accessed item to the "front" (most recently used).
- Write (Put): Adds new data to the front. If the cache hits maximum capacity, the item at the "back" is evicted

## Least Frequent Used(LFU)

LFU removes items with the fewest accesses over time. It prioritizes keeping frequently accessed items, assuming high access counts predict future requests.

**How it Works**:

1. Tracking: Every item has an access counter. When you access an item, its frequency count increases.
2. Eviction: When the cache is full and new data arrives, the algorithm discards the element with the lowest access count.

## First In, First Out(FIFO)

FIFO evicts items in the order they were added, regardless of access patterns. It’s the simplest policy, treating the cache like a queue.

**How it Works**:

1. Insertion: When you add a new item to the cache, it is placed at the end of the queue
2. Capacity Limit: If the cache is full when a new item arrives, the item at the very front of the queue (the oldest entry) is immediately evicted

## Time-To-Live(TTL)

TTL evicts items after a fixed duration. Each cache entry has an expiration timestamp, and items are automatically removed once their TTL expires.

## Adaptive Replacement Cache (ARC)

ARC dynamically balances between LRU and LFU based on workload. It maintains two lists — one for recently accessed items and another for frequently accessed items — adapting the balance as access patterns change.

**How it Works**

ARC divides cache memory into four primary lists, continuously monitoring which data is accessed:

1. $T_1$(Recency / Probation): Holds items that have only been seen once recently.
2. $T_2$(Frequency / Protected): Holds items that have been seen multiple times recently
3. $B_1$(Ghost of $T_1$): A metadata-only list recording the keys of pages that were evicted from \(T\_{1}\).
4. $B_2$(Ghost of $T_2$): A metadata-only list recording the keys of pages that were evicted from $T_2$

ARC uses a continuous "learning rule" that dictates exactly which list loses a page when the cache is full. The balance is controlled by a variable $p$.

- On a $T_1$ Miss but a $B_1$ Hit: The system realizes it just evicted an item that it previously saw once. This implies recency is favored. The algorithm adjusts dynamically by increasing \(p\) to give more space to the $T_{1}$ list.
- On a $T_2$ Miss but a $B_2$ Hit: The system realizes it just evicted an item it previously used frequently. Frequency is favored. The algorithm adjusts by decreasing \(p\) to allocate more space to the $T_2$) list.
- Eviction from $T_1$ or $T_2$: When the cache is full and space is needed, ARC evicts from the tail of the appropriate list based on its current list size parameters.

## Least Recently/Frequently Used (LRFU)

LRFU combines LRU and LFU by assigning weights to recency and frequency. It calculates a score for each item based on both when it was last accessed and how frequently it’s been accessed.

**How it Works**
LFRU splits the cache into two distinct partitions to handle different access patterns:

1. . Unprivileged (Probationary) Partition: This is typically managed using an approximated LFU (Least Frequently Used) algorithm. Newly added data enters this section with a low access count.
   If an item in this section is requested frequently enough, it graduates. If it is rarely requested, it gets evicted quickly to avoid cache pollution.

2. Privileged (Protected) Partition: This highly secure section stores your application's "hot" data. It is generally managed by an LRU (Least Recently Used) policy.
   When data becomes popular, it is promoted here. Data in the privileged partition is rotated or removed based on how long it has been since it was last accessed, ensuring only actively relevant items stay

## Cache Invalidation: Keeping Data Fresh

Cache invalidation is the process of removing or marking cache entries as invalid when underlying data changes. Unlike eviction (driven by capacity), invalidation is driven by data changes to maintain consistency.

- **Time-Based Invalidation**: Cache entries are invalidated after a fixed time period, similar to TTL eviction. This ensures data freshness but may invalidate entries that haven’t changed.

- **Event-Based Invalidation**: Cache entries are invalidated when specific events occur, such as data updates or deletions. This maintains better consistency but requires tracking dependencies between data and cache entries.

- **Manual Invalidation**: Cache entries are explicitly invalidated by application code or administrators. This provides full control but requires careful management to avoid stale data.

- **Tag-Based Invalidation**: Cache entries are tagged with metadata, and invalidation occurs by tag. This allows invalidating multiple related entries simultaneously, useful for hierarchical or related data.

# Redis

Redis is an in-memory data store that runs as its own server process. It stores data across requests and users, making it ideal for server-side caching and shared state management.

**Core Characteristics**

- In-Memory Storage: Keeps all data in RAM for instantaneous read/write operations
- Data Structure Server: Unlike typical key-value stores, it supports rich, native data types like strings, hashes, lists, sets, sorted sets, streams, and geospatial indexes
- Persistence: Offers optional on-disk persistence (RDB snapshots or AOF append-only files) so in-memory data survives restarts or crashes
- High Availability & Scalability: Features built-in replication, clustering, and automated failover (Redis Sentinel) to ensure maximum uptime
- Atomic Operations: Guarantees atomicity for its operations, allowing safe concurrent data modification.

**Use Cases**

- Database & API Caching: Temporarily stores frequently accessed data or complex query results in RAM to reduce backend database load and accelerate API response times.
- Session Management: Persists user login, cart, and state data for web applications. It handles this instantly and avoids data loss if a single server goes down.
- Real-Time Analytics & Leaderboards: Uses atomic counters and sorted sets to effortlessly track real-time metrics (e.g., page views, rate limits) and display live rankings
- Pub/Sub Messaging: Powers real-time chat applications, notification services, and event-driven pipelines using its lightweight publish/subscribe engine and Redis Streams.
  And a lot more

I would suggest you to go through the [docs](https://redis.io/docs/latest/develop/) to learn about how to use redis.
Ohkk so now we know how, where and duration for which we should cache but the bigger question is WHAT? What things should we actually cache, we cannot just cache everything, as in-memory storage are not very cheap.

# What to Cache and When

Ok so now we know how to cache, and also update and delete it, but what exactly are we going to cache?? We cannot just cache everything can we right??

To find exactly what to cache, we need to analyse our workload patterns

## Read/Write Ratio Analysis

The read/write ratio analysis is fundamental to understanding workload patterns in database systems. This ratio illustrates the frequency of read operations (like SELECT queries) compared to write operations (INSERT, UPDATE, DELETE).

A high read ratio indicates that data is frequently accessed but only sometimes changed. This is ideal for caching because it doesn't require frequent invalidation or updates once data is cached. A high write ratio suggests more dynamic data, which can lead to frequent cache invalidations and reduced cache effectiveness.

1. **Quantify operations**. Count the number of read and write operations over a given period. This can be done through script automation or database monitoring tools.
2. **Calculate the ratio**: Compute the proportion of reads to writes. A simple formula could be: Read/Write Ratio = Number of Reads / Number of Writes.

In Postgres, we can use the pg_stat_statements extension:

```
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

And then query pg_stat_statements:

```
SELECT query, calls, rows
FROM pg_stat_statements
WHERE query LIKE 'SELECT %' OR query LIKE 'INSERT %' OR query LIKE 'UPDATE %' OR query LIKE 'DELETE %';
```

This will show you the frequency of SELECT statements versus INSERT, UPDATE, and DELETE statements. You can then calculate the read/write ratio from these counts.

High read/write ratios (e.g., 10:1) indicate that the data is read ten times more often than it is written. This data is a good candidate for caching, you can cache and set a longer Time-To-Live (TTL). A low read/write ratio (e.g., 1:2) indicates more writes than reads. You need to be cautious about caching this data for this type of data, as the cache would need frequent updates or invalidations.

## Temporal Locality and Hotspots

It is based on the principle that recently accessed data will likely be accessed again soon. This pattern is a key factor in identifying 'hotspots' in your data - areas where frequent access suggests a high potential benefit from caching. Understanding and identifying these hotspots allows for a more targeted and efficient caching strategy, improving performance and resource utilization.

To identify hotspots you need to, **Monitor access patterns**. Use database monitoring tools to track access frequency to different data elements. Look for patterns where specific rows, tables, or queries are accessed repeatedly. Frequent execution of the same query, especially within short time frames, indicates a hotspot.

To understand the access frequency, you must configure your database to log detailed query information. In your postgresql.conf, you can set the following parameters:

```
logging_collector = on # enable logging.
log_directory = 'pg_logs' # specify the directory for log files.
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # set the log file naming convention.
log_statement = 'all' # log every executed statement.
```

You can use pgBadger to parse the log and create an HTML report.

## Query Analysis and Profiling

By using query analysis tools and techniques like the EXPLAIN command and examining query execution plans, you can gain insights into how queries are executed, which are resource-intensive, and how they can be optimized.

An execution plan shows how the database engine executes a query, including steps like scans, joins, sorts, and aggregations. The EXPLAIN command in SQL provides the execution plan for a query. It reveals the database's operations and their CPU and I/O cost.

To use EXPLAIN with Your Query, just prepend your SQL query with EXPLAIN. For instance:

```
SELECT * FROM video_game_sales LIMIT 5;
```

This will output the execution plan without actually running the query.

## Query Latencies

The time taken to execute database queries are a crucial metric in workload analysis. They provide insight into the performance of the database and are instrumental in identifying which queries might benefit most from caching. High latencies often indicate bottlenecks or inefficiencies that can be alleviated through strategic caching.

We can set log_min_duration_statement in postgresql.conf to log queries that exceed a specified execution time.

```
set log_min_duration_statement=1000;
```

# Closing Thoughts

This week taught me that database optimization is often less about making queries faster and more about managing resources efficiently.

Before diving into connection pooling and caching, I assumed that handling more traffic mostly meant adding more resources or increasing configuration limits. Running experiments showed me that things aren't that simple. Increasing pool sizes doesn't always improve throughput, caching introduces its own trade-offs, and every optimization comes with questions about consistency, memory usage, and system complexity.

This week helped me build a stronger foundation for understanding how real systems handle increasing load and why seemingly simple decisions can have a significant impact on performance.

As I continue this backend engineering roadmap, I'll keep sharing notes, experiments, mistakes, and lessons learned along the way. If you've spotted something I've misunderstood or have suggestions for further reading, I'd be happy to learn from them.
