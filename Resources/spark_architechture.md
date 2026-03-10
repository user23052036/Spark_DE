# 1. Spark Execution Architecture

A Spark application always contains **3 main components**:

```
Spark Application
│
├── Driver
│
├── Executors
│
└── Tasks
```

### What each one does

| Component | Role                          |
| --------- | ----------------------------- |
| Driver    | Brain of the application      |
| Executors | Workers that run computations |
| Tasks     | Small units of work           |

---

# 2. The Full Architecture Diagram

```
                Your Laptop / Notebook
                ----------------------

           Python / Scala Code
                    │
                    ▼
            +----------------+
            |     DRIVER     |
            |----------------|
            | SparkSession   |
            | Job Scheduler  |
            | DAG Builder    |
            +----------------+
                     │
                     │ send tasks
                     ▼

        Cluster (many machines)

   +-------------+   +-------------+   +-------------+
   |  EXECUTOR 1 |   |  EXECUTOR 2 |   |  EXECUTOR 3 |
   |-------------|   |-------------|   |-------------|
   | Task        |   | Task        |   | Task        |
   | Task        |   | Task        |   | Task        |
   | Task        |   | Task        |   | Task        |
   +-------------+   +-------------+   +-------------+
```

Key idea:

```
Driver → splits job into tasks
Executors → run tasks in parallel
```

---

# 3. Example: Your code

You ran:

```python
df = spark.range(10000000)
df.count()
```

What Spark internally does:

### Step 1 — Driver builds plan

```
spark.range(10000000)
```

Driver creates logical plan.

```
Create dataset
10M numbers
split into partitions
```

---

### Step 2 — Partition the data

Example:

```
10,000,000 numbers

Partition 1 → 0 - 2.5M
Partition 2 → 2.5M - 5M
Partition 3 → 5M - 7.5M
Partition 4 → 7.5M - 10M
```

---

### Step 3 — Tasks created

Driver converts partitions → tasks.

```
Task 1 → count partition 1
Task 2 → count partition 2
Task 3 → count partition 3
Task 4 → count partition 4
```

---

### Step 4 — Executors run tasks

```
Executor 1 → Task 1
Executor 2 → Task 2
Executor 3 → Task 3
Executor 4 → Task 4
```

All run **in parallel**.

---

### Step 5 — Results returned to driver

Executors return results:

```
2.5M
2.5M
2.5M
2.5M
```

Driver aggregates:

```
Total = 10M
```

Then your notebook prints:

```
10000000
```

---

# 4. Where JVMs run

Important concept.

Spark is written in **Scala → runs on JVM**.

Even when you write Python code.

### Architecture when using PySpark

```
Python Process
     │
     │ Py4J bridge
     ▼
JVM Spark Driver
     │
     ▼
Executors (JVM)
```

Diagram:

```
           Python Notebook
                │
                ▼
         Python Driver
                │
         Py4J bridge
                │
                ▼
         JVM Spark Driver
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
 Executor JVM  Executor JVM  Executor JVM
```

So:

```
Python = API layer
Spark engine = JVM
```

---

# 5. Now let's connect the commands you used

## 1. pyspark

You ran:

```
pyspark
```

This starts:

```
Python shell
+
Spark driver JVM
```

Architecture:

```
Terminal
   │
   ▼
Python REPL
   │
   ▼
Spark Driver JVM
   │
   ▼
Executors (local threads)
```

Because you ran locally:

```
master = local[*]
```

Spark uses **CPU cores as executors**.

---

# 6. spark-shell

Command:

```
spark-shell
```

Architecture:

```
Scala REPL
   │
   ▼
Spark Driver JVM
   │
   ▼
Executors
```

Difference:

| Command     | Language |
| ----------- | -------- |
| spark-shell | Scala    |
| pyspark     | Python   |

Both run Spark.

---

# 7. spark-submit

Used for production.

Example:

```
spark-submit job.py
```

Architecture:

```
Submit job
   │
   ▼
Driver launched
   │
   ▼
Cluster manager allocates executors
   │
   ▼
Executors run tasks
```

Diagram:

```
spark-submit
      │
      ▼
+----------------+
|    DRIVER      |
+----------------+
      │
      ▼
Cluster Manager
(YARN / Kubernetes)
      │
      ▼
Executors on machines
```

---

# 8. Your current setup

You are running **Spark in local mode**.

So everything is on your laptop.

```
Laptop
│
├── Driver
│
└── Executors (threads using CPU cores)
```

Diagram:

```
Laptop
┌───────────────────────────┐
│ Driver                    │
│                           │
│ Executor thread 1         │
│ Executor thread 2         │
│ Executor thread 3         │
│ Executor thread 4         │
└───────────────────────────┘
```

---

# 9. What Spark UI (port 4040) shows

When you run Spark you saw:

```
http://localhost:4040
```

Spark UI shows:

```
Jobs
Stages
Tasks
Executors
Storage
```

Example view:

```
Job
 └─ Stage
     └─ Tasks
```

Example:

```
count()

Stage 1
 ├ Task 1
 ├ Task 2
 ├ Task 3
 └ Task 4
```

---

# 10. Mental model you should keep

Remember this simple chain.

```
Code
 ↓
Driver builds plan
 ↓
Plan split into stages
 ↓
Stages split into tasks
 ↓
Executors run tasks
 ↓
Results return to driver
```

---

# 11. The most important Spark idea

Spark is basically:

```
Distributed task scheduler
+
Distributed memory engine
```

Everything else (SQL, MLlib, streaming) is built on top.

Your earlier diagram shows that:

```
Spark SQL
Spark Streaming
MLlib
GraphX
```

All run **on top of the same Spark engine**.

---

# 12. Where you currently are in the learning path

You now understand:

✔ Distributed computing
✔ Hadoop HDFS
✔ MapReduce idea
✔ Spark engine
✔ Driver / Executor / Task
✔ RDD vs DataFrame

You are **ahead of most beginners already**.

---
