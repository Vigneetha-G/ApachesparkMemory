# Apache Spark Performance and Memory Management Essentials 🚀

This document summarizes key concepts in **Spark's architecture, memory management, and performance optimization techniques** that lead to robust and efficient data processing.

---

## 1. Spark Architecture: Driver vs. Executors

| Component | Role | Memory Management |
| :--- | :--- | :--- |
| **Driver** | The master process that coordinates all tasks in a Spark job. Holds **metadata**, task scheduling info, and DAGs. | If the **Driver runs out of memory (OOM)**, the job fails. Driver memory is allocated when the Spark job runs. |
| **Executor** | Processes that perform the actual work of a Spark job. | Executors have their own memory management, utilizing both storage and execution memory pools. |

### Driver Memory Allocation
* **Total Driver Memory** = JVM Heap Memory + Overhead Memory.
* **Overhead Memory** (`spark.driver.memoryOverhead`) is used for JVM threads, shared libraries, and off-heap buffers.
* It is typically **10%** of the requested Driver memory (e.g., 1GB for 10GB) or a default of **384MB**.

### Driver OOM Scenarios
| OOM Cause | Description | Solution |
| :--- | :--- | :--- |
| **Large Broadcast Variables** | The Driver's heap memory stores broadcast variables. If the data being broadcast is larger than the heap memory, it causes an OOM error. | Increase the Driver memory size (`spark.driver.memory`). |
| **`df.collect()`** | This action sends **all partitions** from Executors to the Driver. If the combined size of all partitions is greater than the Driver's heap memory, it causes an OOM error. | Use `df.collect()` only when you expect **few records** (e.g., fetching a maximum date or single row result). Increase Driver size if necessary. |

## 2. Executor Memory and Unified Memory Management

Executor memory is internally managed by Spark for efficient processing.

### Memory Pools
1.  **Reserved Memory:** A fixed amount, typically **300MB**, reserved for the Spark engine.
2.  **Spark Memory Pool:** The remaining memory, which is subject to unified management (default 60% of Total - 300MB).
3.  **User Memory:** Memory used for user-defined functions (UDFs) (default 40% of Total - 300MB).

### Unified Memory Management
The Spark Memory Pool is dynamically split with a **flexible boundary** between Execution and Storage memory.

| Memory Area | Purpose | Flexibility/Authority |
| :--- | :--- | :--- |
| **Execution Memory** | Used for **Transformations** (joins, group by, shuffles). **Short-term memory**. | **PRIORITIZED:** Has the authority to **evict Storage Memory** (using LRU) to gain more space for transformations. |
| **Storage Memory** | Used for **Caching/Persist**. **Long-term memory**. | Can **borrow space** from free Execution Memory. **LACKS AUTHORITY:** Cannot evict Execution Memory. If full, it only evicts its own least-used cached data (LRU). |

## 3. Executor OOM and Skewness Solutions

An **Executor Out-of-Memory (OOM)** error typically occurs when a single partition's data is larger than the available Execution Memory.

### Data Spilling
When Execution Memory is full, Spark can use its **Spilling** leverage to avoid OOM.
* It sends **intermediate results** (from operations like `groupBy`) to **disk** to free up memory for a new partition.
* **Constraint:** You cannot split a partition; the **whole partition** must be sent to disk.

### Extreme Skewness and Salting
* **Extreme Skewness:** When one key (e.g., a "Food" category) has a disproportionately large amount of data, leading to a partition size that exceeds Executor memory.
* **Avoid:** Continuously expanding memory size is a "foolish approach".
* **Solution: Salting (or De-Skewing)**:
    1.  Add a temporary **SALT column** to the skewed DataFrame.
    2.  Assign a small set of random numbers (e.g., `[0, 1, 2, 3]`) to the SALT column.
    3.  The subsequent `groupBy` operation is performed on **two columns** (the original skewed column and the SALT column).
    4.  This divides the massive partition into several smaller, manageable partitions.

---

## 4. Spark SQL Engine (Catalyst Optimizer) Flow

This process is critical for converting a user's query into the most efficient executable plan.

| Step | Plan Name | Description | Key Action |
| :--- | :--- | :--- | :--- |
| 1 | **Unresolved Logical Plan** | The query is parsed from SQL or DataFrame/Dataset API code. | Checks for **Syntactic correctness**. |
| 2 | **Resolved Logical Plan** | The plan is passed to the **Analyzer**. | Uses the **Catalog** (metadata store) to check for and resolve table names, column names, and functions. |
| 3 | **Optimized Logical Plan** | The resolved plan is passed to the **Catalyst Optimizer**. | Applies **rule-based optimizations** (e.g., Predicate Pushdown, combining filters/projections). |
| 4 | **Physical Plans** | The optimized plan is converted into one or more candidate physical execution plans. | Multiple execution strategies are generated (e.g., choosing between `SortMergeJoin` or `BroadcastHashJoin`). |
| 5 | **Selected Physical Plan** | Candidate physical plans are evaluated by the **Cost Model**. | Selects the **most optimal physical plan** based on estimated execution time and resources. This plan is executed by the Executors. |

### Catalyst Optimizer Flowchart
![Spark Catalyst Optimizer Flowchart](https://www.databricks.com/wp-content/uploads/2015/04/Screen-Shot-2015-04-12-at-8.41.26-AM.png)

---

## 5. Caching, Persist, and Optimization Techniques

### Caching and Persist
**Caching** and **Persist** are mechanisms to store the result of an operation in Storage Memory for reuse, avoiding re-computation. `df.cache()` is a shortcut for `df.persist(StorageLevel.MEMORY_AND_DISK)`.

| Storage Level | Description |
| :--- | :--- |
| **`MEMORY_ONLY`** | Data stored in RAM as deserialized Java objects. If not enough memory, partitions are recomputed when needed. |
| **`MEMORY_AND_DISK`** | Tries to store in memory; if insufficient, spills the rest to disk. |
| **`DISK_ONLY`** | Stores data only on disk (the slowest option). |
| **`MEMORY_ONLY_2`** | $2\times$ replicated for **fault tolerance**. |

### Adaptive Query Execution (AQE)
AQE optimizes the query plan dynamically during runtime.

1.  **Dynamically Coalesce Partitions:** Tries to combine small partitions to create partitions of a more **similar size**, improving efficiency.
2.  **Optimizes Join Strategy:** Optimizes the join algorithm based on runtime query statistics.
3.  **Skewness Handling:** Automatically handles skewness by splitting a large partition into smaller ones based on the data.

### Partition Pruning
This technique skips scanning unnecessary partitions when reading data.
* If your query filters by date or region, Spark only scans the relevant partition, making the query faster by reading less data.
* **Dynamic Partition Pruning** occurs during a join operation where a filter from one DataFrame is dynamically pushed to the partitioned DataFrame.

---

## 6. Deployment Modes (Edge Node, Client, Cluster)

The deployment mode defines where the **Spark Driver** process runs relative to the cluster.

| Concept | Role | Description |
| :--- | :--- | :--- |
| **Edge Node** | An intermediary machine used by developers to submit applications. | Developers login to this node, and it talks to the **Cluster Manager** to request and assign resources for the Spark job. |
| **Client Mode** | The Driver process is created on the **client machine** (the Edge Node or submission machine). | The application is submitted from the client, and the Driver stays there, which can introduce network latency. |
| **Cluster Mode** | The Driver process is created on one of the **worker nodes** within the cluster. | Generally preferred for production as it reduces network latency by keeping the Driver close to the Executors and frees up the submission machine. |
