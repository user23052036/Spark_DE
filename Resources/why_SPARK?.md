# 1) Distributed computing — the basics (conceptual + why it matters)

**Short idea:** instead of one big machine doing all work, you split data and work across many machines (nodes). Each node stores part of the data and performs part of the computation in parallel. This gives more storage and more CPU for very large datasets.

**Why we do it**

* Scale storage: many disks across nodes store TBs/PBs.
* Scale compute: many CPUs operate in parallel to finish jobs faster.
* Fault tolerance: copies of data on multiple nodes let cluster survive node failures.

**Core terms**

* **Node / Worker** — a machine in the cluster.
* **Partition** — a subset of the dataset stored/processed on a node.
* **Master / Coordinator** — schedules work and tracks metadata.
* **Shuffle** — network transfer of data between tasks (expensive).

**Simple ASCII diagram**

```
+-----------------------------+
| Client / Driver             |
| - send job plan             |
+-------------+---------------+
              |
      +-------+-------+
      | Cluster nodes |
      +---------------+
     /  |    |    |     \
Node1 Node2 Node3 Node4 Node5
(P1)  (P2)  (P3)  (P4)  (P5)
```

**Key pattern (data-locality):** move the computation to the node that stores the data, not the other way around.

**Example problem:** count words in 1 TB of text.
Instead of loading the whole file to one machine, split it into blocks, each node counts words for its blocks (map), then results are aggregated (reduce).

**Risks / failure modes**

* Network becomes a bottleneck during shuffles.
* Poor partitioning leads to skew (one node overloaded).
* Node failure requires replication and retrying tasks.

**When to use distributed computing**

* Use it for datasets that don’t fit in memory or on disk of a single machine, or for jobs needing more CPU than one server can provide.

---

# 2) What is Apache Spark (and RDDs) — concise conceptual picture

**Short idea:**
Apache Spark is a distributed data-processing engine that runs computations across a cluster. It improves on classic MapReduce by keeping data in memory (when useful) and offering a richer API (RDDs, DataFrames, SQL, MLlib, streaming).

**Core Spark architecture (Driver / Executors / Cluster Manager)**

```
[Client / Notebook]  ---> submits job --->  [Cluster Manager (YARN/K8s/Mesos)]
                                 |
                              allocates
                                 |
                           +------------+
                           |   Driver   |
                           | (schedules)|
                           +------------+
                                 |
                   +-------------+-------------+
                   |             |             |
               Executor1     Executor2     Executor3
               (worker JVM)   (worker JVM)   (worker JVM)
               tasks run      tasks run      tasks run
```

* **Driver**: process that runs the application’s `SparkSession`, builds logical plan, coordinates execution.
* **Executors**: worker JVMs on nodes that execute tasks and store cached data.
* **Cluster manager**: allocates resources (can be YARN, Kubernetes, standalone).

**What is an RDD (Resilient Distributed Dataset)?**

* RDD = a distributed, immutable collection of items partitioned across nodes.
* **Resilient** because Spark keeps lineage (how RDD was derived) so it can recompute lost partitions.
* Operations:

  * **Transformations** (lazy): `map`, `filter`, `flatMap`, `groupByKey` → produce new RDDs.
  * **Actions** (trigger compute): `collect`, `count`, `reduce`, `saveAsTextFile`.
* RDDs give you low-level control: partitioning, custom serialization, fine-grained ops.

**RDD ASCII diagram**

```
RDD1 (file split across nodes)
   map()  -> RDD2  (each partition transformed)
   filter() -> RDD3
action: count() -> triggers tasks on executors -> returns number
```

**Simple PySpark RDD example (word count)** — run in a notebook cell:

```python
rdd = spark.sparkContext.textFile("/path/to/text_large.txt")   # create RDD from HDFS/local
words = rdd.flatMap(lambda line: line.split())                # transformation (lazy)
pairs = words.map(lambda w: (w, 1))
counts = pairs.reduceByKey(lambda a,b: a+b)                   # again lazy
counts.take(10)                                               # action -> triggers job
```

**Assumptions / risks**

* RDDs are low-level and require you to think about partitioning and shuffles.
* RDDs are sometimes slower than DataFrames because they bypass optimizer.

**When to use RDDs**

* When you need fine-grained control not available in DataFrame API.
* When working with unstructured data and writing custom partitioning / custom serialization.

---

# 3) RDD vs DataFrame — comparison, internals, and examples

**Short summary:**

* **RDD** = low-level distributed collection (object-oriented).
* **DataFrame** = higher-level, schema-aware, columnar, optimized (SQL-like). Prefer DataFrame for most analytics and ML.

## Key differences (table)

| Aspect       |                                  RDD | DataFrame                                           |
| ------------ | -----------------------------------: | :-------------------------------------------------- |
| API level    |              Low-level (map, filter) | High-level (select, groupBy, SQL)                   |
| Schema       |        No schema — arbitrary objects | Has schema: columns with names/types                |
| Optimization |                  No global optimizer | Catalyst optimizer (rewrites queries)               |
| Execution    |           Less efficient for SQL ops | Columnar execution, better memory layout (Tungsten) |
| Use cases    | Custom transformations, fine control | ETL, SQL, ML pipelines, most analytics              |
| Ease of use  |                         More verbose | Concise, expressive                                 |

## Why DataFrame is faster

* **Catalyst optimizer** rewrites your logical operations into optimized physical plans.
* **Tungsten** execution uses a compact binary format and optimized code paths.
* Columnar execution reduces memory and speeds vectorized operations.

## Concrete examples: word count using RDD and DataFrame

### RDD version (we saw above)

```python
rdd = spark.sparkContext.textFile("/data/docs.txt")
words = rdd.flatMap(lambda line: line.split())
pairs = words.map(lambda w: (w, 1))
counts = pairs.reduceByKey(lambda a,b: a+b)
counts.take(10)
```

### DataFrame version (preferred)

```python
from pyspark.sql.functions import explode, split, col

df = spark.read.text("/data/docs.txt")     # df has column "value" (string line)
words_df = df.select(explode(split(col("value"), "\\s+")).alias("word"))
counts_df = words_df.groupBy("word").count()
counts_df.show(10)
```

**Why the DataFrame version is preferable:**

* Catalyst may push down filters or optimize the groupBy.
* Execution will run faster for large datasets due to internal optimizations and efficient memory layout.
* DataFrame code is shorter and integrates with SQL and ML pipelines.

## Partitioning and performance note

Both RDD and DataFrame are partitioned underneath. You can control partitions:

```python
rdd = rdd.repartition(100)      # more partitions -> more tasks
df = df.repartition(100)
```

But with DataFrames, Catalyst can use partitioning metadata to optimize.

## Advanced: DataFrame → Dataset (typed) (Scala/Java)

* In Scala, Spark has `Dataset[T]` which is typed and combines both performance and type-safety. Python uses DataFrame only.

---

# Diagrams (compact)

## Spark application flow (driver → executors → tasks)

```
Notebook/Script
   |
   v
Driver (SparkSession)
  - builds logical plan
  - asks Cluster Manager for resources
  - schedules tasks
   |
   v
Executors (JVMs on nodes)
  - get tasks, run code on partitions
  - hold cached RDD/DataFrame partitions
   |
   v
Tasks -> read partitions -> compute -> write results
```

## DataFrame logical → physical transformation (Catalyst)

```
User DataFrame API call (SQL) 
   ↓
Logical Plan (unoptimized)
   ↓ Catalyst optimizer rewrites/optimizes
Optimized Logical Plan
   ↓ Physical Planner picks physical operators
Physical Plan (e.g., shuffle, aggregate)
   ↓
Tasks scheduled and executed on Executors
```

---

# Practical rules / best practices (short list)

1. **Prefer DataFrame / Spark SQL** for ETL and analytics. Use RDD only when you need something dataframes can’t do.
2. **Initialize Spark once** at top of notebook. Reuse `spark` session.
3. **Avoid `collect()` on big datasets** — it brings data back to driver and can OOM. Use `show(n)` for preview.
4. **Partition smartly**: use `repartition()` or `coalesce()` when controlling parallelism. Avoid extreme skew.
5. **Cache** intermediate datasets if you reuse them: `df.cache()` then `df.count()` to materialize.
6. **Monitor Spark UI** (4040) to inspect stages, tasks, shuffle size — that tells you where time is spent.

---

# Failure modes & tradeoffs (quick)

* **Too many partitions**: overhead of scheduling many tiny tasks → slowdown.
* **Too few partitions**: tasks aren’t parallel enough → CPUs idle.
* **Large shuffle**: network I/O heavy → slow; avoid wide transformations where possible or tune partitioning.
* **Using Python UDFs**: slow; prefer builtin functions or Pandas UDFs for vectorized performance.
* **Memory OOM**: increase executor memory or tune joins (use broadcast joins for small tables).

---

# Exercises (do these in your VS Code notebook)

1. **Create an RDD and count words**

```python
rdd = spark.sparkContext.parallelize(["hello world", "hello spark", "big data"])
rdd.flatMap(lambda l: l.split()).map(lambda w: (w,1)).reduceByKey(lambda a,b: a+b).collect()
```

2. **Same with DataFrame and compare**

```python
df = spark.createDataFrame([("hello world",),("hello spark",),("big data",)], ["line"])
from pyspark.sql.functions import explode, split, col
words_df = df.select(explode(split(col("line"), "\\s+")).alias("word"))
words_df.groupBy("word").count().show()
```

3. **Partition experiment**

```python
big = spark.range(20000000).repartition(8)  # adjust 8 -> number of cores
res = big.groupBy((big.id % 10).alias("bucket")).count()
res.explain(True)   # shows physical plan and shuffles
res.show()
```

Open Spark UI, inspect stages and shuffle size.

---

# Final quick checklist for you

* Know difference: RDD = manual, DataFrame = optimized.
* Remember: transformations are lazy; actions trigger execution.
* Use Spark UI to debug performance.
* Prefer DataFrames for most real work.

---

