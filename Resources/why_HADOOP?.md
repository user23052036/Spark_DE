# 1. The core problem Hadoop was created to solve

Assumption: PostgreSQL runs on **one machine**.

Example:

```
PostgreSQL server
CPU
RAM
Disk
```

If your dataset is:

```
10 GB
50 GB
100 GB
```

Postgres handles it easily.

But what if your dataset is:

```
10 TB
100 TB
1 PB
```

Two major problems appear.

### Problem 1 — Storage

One machine cannot store petabytes reliably.

### Problem 2 — Computation

Even if storage works, **one CPU cannot process that much data in reasonable time**.

---

### Hadoop’s idea

Instead of **one big machine**, use **many cheap machines**.

Example cluster:

```
Machine1
Machine2
Machine3
Machine4
Machine5
```

Each machine contributes:

```
CPU
RAM
Disk
```

So total resources become:

```
Total CPU = sum of CPUs
Total Storage = sum of disks
```

This is called **distributed computing**.

---

# 2. Hadoop ecosystem overview

Hadoop is not one program. It is an **ecosystem**.

Core parts:

```
Hadoop
 ├── HDFS        (storage layer)
 ├── MapReduce   (processing layer)
 └── YARN        (resource manager)
```

Think of it like:

| Layer     | Role                    |
| --------- | ----------------------- |
| HDFS      | distributed storage     |
| MapReduce | distributed computation |
| YARN      | job scheduler           |

---

# 3. HDFS (Hadoop Distributed File System)

Let’s compare with PostgreSQL storage.

### PostgreSQL storage

```
Table
Rows
Blocks on disk
```

Everything stored on **one server disk**.

---

### HDFS storage

HDFS stores files across **many machines**.

Example file:

```
1 TB log file
```

HDFS splits it into blocks.

Example:

```
Block size = 128 MB
```

File becomes:

```
Block1
Block2
Block3
Block4
...
Block8000
```

These blocks are distributed across machines.

Example cluster:

```
Node1 → Block1 Block4 Block9
Node2 → Block2 Block7 Block10
Node3 → Block3 Block5 Block6
```

---

### Replication (important concept)

HDFS replicates blocks for fault tolerance.

Default replication:

```
3 copies
```

Example:

```
Block1 stored on
Node1
Node3
Node5
```

If one machine dies → data still exists.

---

### HDFS components

Two main roles exist.

#### NameNode

```
Master
```

Stores metadata:

```
file.txt
→ Block1 → Node1 Node3 Node5
→ Block2 → Node2 Node4 Node6
```

NameNode knows **where every block lives**.

It does NOT store the data itself.

---

#### DataNode

```
Worker machines
```

They store the actual blocks.

Example:

```
Node1
Block1
Block8
Block14
```

---

# 4. Why not just use PostgreSQL cluster?

You could.

But relational databases are optimized for:

```
structured data
transactions
indexes
joins
```

Big data systems are optimized for:

```
huge sequential data
log processing
analytics
batch computation
```

Typical big data workloads:

```
web logs
click streams
sensor data
social media
images
videos
```

These datasets can be **hundreds of terabytes**.

---

# 5. MapReduce (processing model)

Now assume we stored 1 TB data in HDFS.

How do we process it?

Instead of moving data to computation, Hadoop does:

```
Move computation to where the data is
```

This is the **key idea**.

---

### MapReduce has two phases

```
MAP
REDUCE
```

---

# 6. Map phase

Each node processes the blocks it holds.

Example dataset:

```
documents
```

Task:

```
count word frequency
```

Example documents:

```
doc1: hello world
doc2: hello hadoop
doc3: big data world
```

---

Map step converts each word into key-value pairs.

Example:

```
hello → 1
world → 1
hello → 1
hadoop → 1
big → 1
data → 1
world → 1
```

---

# 7. Shuffle phase (automatic)

Hadoop groups same keys together.

```
hello → [1,1]
world → [1,1]
hadoop → [1]
big → [1]
data → [1]
```

---

# 8. Reduce phase

Now reducer aggregates values.

```
hello → 2
world → 2
hadoop → 1
big → 1
data → 1
```

This produces the final output.

---

# 9. Why MapReduce was revolutionary

Before Hadoop:

```
single machine processing
```

After Hadoop:

```
thousands of machines process data in parallel
```

Example cluster:

```
1000 nodes
```

Each processes a small part of the data.

Result:

```
1 TB job finishes in minutes instead of hours.
```

---

# 10. Limitations of MapReduce

MapReduce has problems:

1. Disk heavy (writes intermediate results)
2. Slow for iterative algorithms
3. Hard to program
4. Poor for machine learning

Because of these limitations, **Apache Spark was created**.

Spark improves:

```
in-memory processing
faster pipelines
DataFrame API
ML pipelines
```

That is why your course is using **Spark instead of MapReduce directly**.

---

# 11. How Spark fits into the ecosystem

Modern architecture:

```
Storage → HDFS / S3
Compute → Spark
```

Spark replaces MapReduce for most workloads.

But it can still **read data from HDFS**.

---

# 12. Final mental model

Think of it like this:

```
HDFS      → distributed hard drive
MapReduce → distributed CPU
Spark     → modern faster distributed engine
```

---

