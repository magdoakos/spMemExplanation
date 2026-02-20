# Configuring Spark Driver and Executor Memory for PySpark Python Workers

## Executive summary

Your measured requirement—**~6 GB RAM per Python worker process**—almost always means you should **budget memory outside the executor JVM heap**, i.e. via **`spark.executor.memoryOverhead`** and/or **`spark.executor.pyspark.memory`**, not by simply inflating **`spark.executor.memory`** (the JVM heap). Spark’s own configuration docs explicitly separate executor heap (`spark.executor.memory`) from “additional / non-heap” memory (`spark.executor.memoryOverhead`) and note that this additional memory covers non-JVM processes in the same container, including PySpark when `spark.executor.pyspark.memory` is not set. citeturn3view0turn2view0

For a Python-heavy stage where **every concurrent task runs Python**, the executor will typically run **up to one Python worker per concurrent task**; Spark’s Python runner code notes the worker pool can grow to the number of concurrent tasks, which is determined by executor cores. citeturn27view0turn32view0 With a 6 GB-per-worker peak, a 4-core executor can easily need **~24 GB just for Python workers**, plus JVM heap and non-heap/native overhead—far beyond Spark’s default overhead calculation (often ~10% of heap, min 384 MiB). citeturn2view0turn27view0

Actionable recommendation for your case (assumptions stated):

- **Assumptions**: (a) you are on a containerized cluster manager that enforces memory limits such as entity["organization","Apache Hadoop YARN","resource manager"] or entity["organization","Kubernetes","container orchestration"] (this is where Spark’s overhead settings apply), (b) executors run on Linux (so Python `resource` limits are available), and (c) Python runs in tasks (UDFs/RDD ops) rather than in the driver only. citeturn2view0turn3view0turn28view0turn16view0  
- **Start by controlling Python concurrency per executor**: prefer **1–2 executor cores** for Python-heavy work so you don’t spawn 4 Python workers per executor by default. Spark’s Python runner associates Python worker concurrency with executor cores. citeturn27view0  
- **Budget Python memory explicitly**:
  - If you want the safest “don’t get container-killed” approach: raise **`spark.executor.memoryOverhead`** until it covers **(concurrent Python workers × 6 GB) + native/JVM overhead + headroom**. citeturn2view0turn3view0  
  - If you prefer Python to fail fast with a clearer `MemoryError` rather than a container kill: also set **`spark.executor.pyspark.memory`** (total per executor), which Spark divides by executor cores and enforces via `resource.RLIMIT_AS` inside each Python worker. citeturn3view0turn27view0turn28view0turn22view0  
- A pragmatic starting point (with ~15% headroom over 6 GB, i.e. **7 GiB per Python worker budget**) is:
  - **1-core executor**: `spark.executor.memory=6g`, `spark.executor.pyspark.memory=7g`, `spark.executor.memoryOverhead=2g` (≈ 15 GiB total container)  
  - **2-core executor**: `spark.executor.memory=6g`, `spark.executor.pyspark.memory=14g`, `spark.executor.memoryOverhead=2g` (≈ 22 GiB total container)  
  You then increase heap if Spark (JVM-side) memory pressure demands it, but you increase overhead / pyspark memory to satisfy the 6 GB Python worker requirement. citeturn3view0turn2view0turn10view0

## Spark and PySpark memory architecture

Spark applications have a **driver** and one or more **executors**. On YARN, Spark documentation distinguishes deploy modes: in **cluster mode**, the driver runs inside the YARN ApplicationMaster; in **client mode**, the driver runs in the submitting client process and the ApplicationMaster mainly requests resources. citeturn5view0

### Executor memory is not a single pool

From Spark’s configuration docs, the executor’s memory footprint (as enforced by YARN/Kubernetes) is composed of multiple pieces:

- **Executor JVM heap**: `spark.executor.memory` (this is the memory Spark uses to size the executor heap; you should not set `-Xmx` via `spark.executor.extraJavaOptions`). citeturn3view0turn31view0  
- **Executor non-heap / “overhead”**: `spark.executor.memoryOverhead`, intended for VM overheads and “other native overheads,” and it is explicitly supported on YARN and Kubernetes. citeturn2view0  
- **Optional executor off-heap**: `spark.memory.offHeap.enabled` + `spark.memory.offHeap.size`; Spark notes off-heap size does not affect heap, and if total memory must fit a hard limit you must account for it. citeturn2view4  
- **Optional explicit PySpark allocation**: `spark.executor.pyspark.memory`; if set, Spark says PySpark memory in each executor is limited to that amount and (for YARN/Kubernetes) is added to resource requests. citeturn3view0

Spark’s own executor overhead doc statement is especially important for PySpark: it notes that “additional memory includes PySpark executor memory (when `spark.executor.pyspark.memory` is not configured)” and that the **maximum container memory** for an executor is determined by the sum of `spark.executor.memoryOverhead`, `spark.executor.memory`, `spark.memory.offHeap.size`, and `spark.executor.pyspark.memory`. citeturn2view0

### Unified memory inside the JVM heap

Inside the executor heap, Spark’s tuning guide describes a unified region for **execution** (shuffles, joins, aggregations) and **storage** (caching), governed by:

- `spark.memory.fraction` (default 0.6 of (heap − 300 MiB))  
- `spark.memory.storageFraction` (default 0.5 of the memory fraction region) citeturn10view0turn2view4  

These settings influence how Spark uses **heap** memory; they **do not** directly provision memory for external Python processes (which run outside the JVM). citeturn10view0turn2view0

### How PySpark launches and uses Python workers

Spark’s PySpark user guide (“Debugging PySpark”) states:

- On executors, **Python workers** execute Python-native functions/data and are launched **only when needed** (Python UDFs, pandas UDFs, PySpark RDD APIs). citeturn16view0  
- In process listings, Python workers can be seen as processes forked from `pyspark.daemon`. citeturn16view1  

Spark’s source code explains the launch mechanism: `PythonWorkerFactory` prefers starting a **single Python daemon** (by default `pyspark.daemon`, i.e. `pyspark/daemon.py`) and asking it to **fork new worker processes** for tasks, because forking from Java is expensive; it can fall back to launching workers directly (by default `pyspark.worker`) and daemon mode is UNIX-oriented. citeturn15view0  

Those Python processes are **separate OS processes** from the executor JVM. Their memory consumption is therefore **outside** the JVM heap, but still counts against the executor container/pod memory limit that YARN/Kubernetes enforce. citeturn2view0turn18view0turn18view2  

### Executor memory layout flowchart

The following summarizes how memory “fits together” for one executor on YARN/Kubernetes, consistent with Spark’s documented container-memory sum and PySpark worker launch behavior. citeturn2view0turn15view0turn16view0

```mermaid
flowchart TB
  A[Executor container / pod memory limit] --> B[JVM executor process]
  B --> H[JVM heap: spark.executor.memory]
  H --> U[Unified Spark memory (execution + storage)\ncontrolled by spark.memory.fraction/storageFraction]
  B --> NH[JVM non-heap/native (metaspace, stacks, direct buffers)]

  A --> O[Non-heap budget: spark.executor.memoryOverhead]
  O --> NH
  O --> P[Python side when spark.executor.pyspark.memory is NOT set\n(pyspark.daemon + pyspark.worker processes)]

  A --> PS[Optional: spark.executor.pyspark.memory\n(PySpark budget; Linux RLIMIT_AS-based)]
  PS --> P

  A --> OFF[Optional: spark.memory.offHeap.size\n(Tungsten off-heap)]
```

## Configuration knobs and precedence

### The most important knobs for your scenario

Executor-side:

- `spark.executor.memory`: executor JVM memory setting (heap sizing). citeturn3view0turn31view0  
- `spark.executor.memoryOverhead`: executor “additional”/non-heap memory, supported on YARN and Kubernetes; defaults to `executorMemory * spark.executor.memoryOverheadFactor` with a minimum `spark.executor.minMemoryOverhead`. citeturn2view0  
- `spark.executor.memoryOverheadFactor` (default 0.10; ignored if `spark.executor.memoryOverhead` is set): fraction used to compute overhead; Spark notes a higher default (0.40) for “Kubernetes non-JVM jobs.” citeturn2view0turn8view1  
- `spark.executor.minMemoryOverhead` (default 384m): minimum overhead if not set explicitly. citeturn2view0  
- `spark.executor.pyspark.memory`: PySpark memory allocation per executor; if set, PySpark memory is limited and added to YARN/Kubernetes resource requests; Spark notes platform limitations inherited from Python’s `resource` module. citeturn3view0turn28view0  

Driver-side (relevant if Python does heavy work on the driver):

- `spark.driver.memory`: driver JVM memory; Spark notes you must not set this through `SparkConf` in client mode because the driver JVM already started—use `--driver-memory` or a properties file. citeturn2view3  
- `spark.driver.memoryOverhead`: driver non-heap memory in cluster mode supported on YARN and Kubernetes; Spark explicitly includes “python process that goes with a PySpark driver” in this non-heap memory. citeturn2view2turn2view3  

PySpark / Python runtime selection and worker behavior:

- `spark.pyspark.python` and `spark.pyspark.driver.python`: Python executables (executors + driver, respectively). citeturn11view4  
- Environment-variable precedence: Spark notes `spark.pyspark.python` takes precedence over `PYSPARK_PYTHON`, and `spark.pyspark.driver.python` over `PYSPARK_DRIVER_PYTHON`. citeturn11view4  
- `spark.python.worker.reuse` (default true): reuse Python workers; Spark describes this as using a fixed number of Python workers and avoiding forking for every task. citeturn11view0  
- `spark.python.worker.memory` (default 512m): memory used per Python worker “during aggregation”; if exceeded, it spills to disk (this is distinct from overall Python process memory limits). citeturn11view6turn22view1  
- `spark.python.worker.killOnIdleTimeout` and related idle pool controls exist in recent Spark and can affect how long idle workers persist. citeturn11view0  

Cluster-manager specifics:

- On Kubernetes, Spark’s Kubernetes doc states the driver/executor pod memory request/limit is set by the **sum** of `spark.{driver,executor}.memory` and `spark.{driver,executor}.memoryOverhead`. citeturn8view0  
- Kubernetes also has `spark.kubernetes.memoryOverheadFactor` to size non-JVM memory overhead for pods. citeturn8view1  
- On YARN, there is a YARN-specific `spark.yarn.am.memoryOverhead` for the ApplicationMaster in client mode (described as “same as `spark.driver.memoryOverhead`” but for the AM). citeturn6view0  

Legacy note: older Spark setups and logs may show `spark.yarn.executor.memoryOverhead` / `spark.yarn.driver.memoryOverhead` as deprecated in favor of `spark.executor.memoryOverhead` / `spark.driver.memoryOverhead`. citeturn29search3turn20view0  

### Units and defaults that matter operationally

- `spark.executor.memory` and `spark.driver.memory` are specified as JVM memory strings with suffixes like `m` and `g`. citeturn3view0turn2view3  
- `spark.executor.memoryOverhead` is described “in MiB unless otherwise specified” (so a bare number is interpreted as MiB; explicit units like `384m` are also used in docs). citeturn2view0  
- `spark.executor.pyspark.memory` is described “in MiB unless otherwise specified” and depends on Python’s `resource` module; Spark notes Windows does not support resource limiting and macOS does not actually limit resources. citeturn3view0turn28view0  

### Precedence and where to set what

Spark’s config docs provide a clear precedence:

1. Properties set directly on the **SparkConf** (highest precedence)  
2. Then `--conf` flags or `--properties-file` passed to `spark-submit` / `spark-shell`  
3. Then `spark-defaults.conf` citeturn20view0turn20view1  

Spark also notes that if you pass a `--properties-file`, Spark does not load `conf/spark-defaults.conf` unless you add `--load-spark-defaults`. citeturn20view0turn20view1  

## Sizing methodology for a 6 GB Python worker

### Step one: compute how many Python workers can run concurrently per executor

In PySpark, the relevant quantity is not “Python workers per executor over time,” but **peak concurrent Python workers**, because that determines peak memory consumption.

Two source-backed facts help:

- Spark’s Python runner notes the worker pool “will grow to the number of concurrent tasks, which is determined by the number of cores in this executor.” citeturn27view0  
- Spark’s scheduler uses `spark.executor.cores` and `spark.task.cpus` to compute “number of slots” in at least some scheduling contexts. citeturn32view0  

A practical approximation for Python-heavy stages is therefore:

**`N_py_concurrent ≈ floor(spark.executor.cores / spark.task.cpus)`**, and in the common case `spark.task.cpus=1`, **`N_py_concurrent ≈ spark.executor.cores`**. citeturn32view0turn27view0  

### Step two: understand what memory “bucket” Python uses

If you do nothing special, a Python worker’s RSS is **not part of `spark.executor.memory`**; it is memory “outside the JVM,” which Spark expects to fit under overhead/non-heap space and the container limit. Spark explicitly documents that executor “additional memory” includes PySpark executor memory when `spark.executor.pyspark.memory` is not configured. citeturn2view0turn3view0  

### Step three: choose your control strategy

There are two mainstream strategies (they can be combined):

#### Strategy A: fund Python using `spark.executor.memoryOverhead`

- You set `spark.executor.memoryOverhead = baseline_nonheap + N_py_concurrent × python_worker_peak + headroom`.  
- This is the simplest mental model and works because Spark requests container memory as a sum that includes `spark.executor.memoryOverhead`. citeturn2view0turn3view0  

Trade-off: Spark does not inherently stop Python from consuming beyond “its share,” so an RSS spike can still lead to a **container kill**.

#### Strategy B: set `spark.executor.pyspark.memory` to cap Python

Spark added `spark.executor.pyspark.memory` to limit Python’s addressable memory space (using `resource.RLIMIT_AS`) and avoid situations where YARN kills containers because Python doesn’t know its constraints. citeturn22view0turn28view0  

The mechanics matter:

- In Spark’s Scala code, the executor’s PySpark memory allocation is divided by **executor cores** so “each python worker gets an equal part of the allocation,” and it sets `PYSPARK_EXECUTOR_MEMORY_MB` accordingly. citeturn27view0  
- In the Python worker, Spark reads `PYSPARK_EXECUTOR_MEMORY_MB` and uses `resource.RLIMIT_AS` to set the limit (with platform caveats). citeturn25view1turn28view0  

Therefore, to support ~6 GB per Python worker with `C` executor cores (and thus up to `C` concurrent workers), you typically want:

**`spark.executor.pyspark.memory ≈ C × (python_worker_peak + headroom)`**. citeturn27view0turn3view0  

### Step four: compute total executor container memory

Per Spark docs, on YARN/Kubernetes, the executor container memory upper bound is:

**`M_executor_container = spark.executor.memory + spark.executor.memoryOverhead + spark.memory.offHeap.size + spark.executor.pyspark.memory`**. citeturn2view0turn2view4  

If you are not using off-heap and not setting `spark.executor.pyspark.memory`, Python must fit in overhead; if you do set `spark.executor.pyspark.memory`, it is added to requests as well. citeturn2view0turn3view0  

### Numeric examples and a configuration table for your 6 GB worker

Assumptions for the table:

- Peak Python worker RSS measured: **6 GB** (your measurement).  
- Headroom: **~15%** (rounded up to **7 GiB per worker budget**) as a practical starting point (tune based on real peaks).  
- Baseline non-heap/native/JVM overhead: **2 GiB** (thread stacks, metaspace, direct buffers, plus miscellaneous non-JVM processes). Spark describes overhead as accounting for “VM overheads… other native overheads.” citeturn2view0turn2view2  
- Executor JVM heap (`spark.executor.memory`): **6 GiB** as a baseline; tune upward if Spark JVM-side spills/OOMs occur, but keep in mind GC trade-offs. citeturn31view0turn10view0turn30view1  

Table interpretation: “Total container” corresponds to Spark’s documented sum for executors. citeturn2view0turn2view4  

| Executor cores (≈ max concurrent Python workers) | Recommended `spark.executor.memory` (heap) | Recommended `spark.executor.memoryOverhead` | Optional `spark.executor.pyspark.memory` (Python cap) | Resulting total executor container memory |
|---:|---:|---:|---:|---:|
| 1 | 6g | 2g + (1 × 7g) = **9g** (Strategy A) **or** **2g** (Strategy B) | 7g (Strategy B) | **15g** |
| 2 | 6g | 2g + (2 × 7g) = **16g** (Strategy A) **or** **2g** (Strategy B) | 14g (Strategy B) | **22g** |
| 4 | 6g | 2g + (4 × 7g) = **30g** (Strategy A) **or** **2g** (Strategy B) | 28g (Strategy B) | **36g** |

Why the overhead numbers dwarf Spark defaults: by default, `spark.executor.memoryOverhead` is computed as `executorMemory × spark.executor.memoryOverheadFactor` (default 0.10) with a minimum `spark.executor.minMemoryOverhead` (384m). With a 6g heap, the default overhead is on the order of hundreds of MiB—nowhere near a 6 GB Python process. citeturn2view0  

#### Should you increase executor heap or memoryOverhead for a 6 GB Python process?

- The **6 GB Python worker** should be funded by **`spark.executor.memoryOverhead`** and/or **`spark.executor.pyspark.memory`**, because Python workers are **separate processes** and Spark’s overhead description explicitly covers non-JVM memory and PySpark (when pyspark memory is not configured). citeturn2view0turn16view0turn15view0  
- You increase **executor heap** (`spark.executor.memory`) when the **JVM side** needs more headroom: caching, shuffle-heavy joins/aggregations/sorts, large broadcast materialization, etc., whose memory behavior is governed by Spark’s unified memory manager and GC behavior. citeturn10view0turn30view1  

### Driver sizing when Python runs on the driver

Spark’s driver overhead doc is explicit: driver non-heap memory includes memory used by other driver processes, such as “python process that goes with a PySpark driver,” and the driver container limit (cluster mode on YARN/Kubernetes) is the sum of `spark.driver.memory` and `spark.driver.memoryOverhead`. citeturn2view2turn2view3  

Practical implication:

- If your pipeline does significant driver-side Python work (e.g., `collect()` then pandas processing, heavy local modeling, large `toPandas()`), you must budget **driver memory + driver overhead** accordingly, especially in cluster mode. citeturn2view2turn2view3  
- In client mode, Spark warns you cannot effectively set driver heap via `SparkConf` after startup; you should set it via `--driver-memory` or your properties file. citeturn2view3turn20view0  

## Best practices and trade-offs

### Avoid “fixing” Python memory problems by inflating JVM heap

Over-allocating `spark.executor.memory` may increase the default overhead fraction (if you rely on the default overhead factor), but it carries GC trade-offs. Spark’s tuning guide emphasizes that GC cost is proportional to object count and discusses strategies like serialized caching to reduce GC overhead; it also notes Spark’s default GC (G1GC) and that large heaps may require GC tuning. citeturn30view0turn30view2  

A better approach is to keep heap “reasonable” for the JVM workload and allocate **separately** for Python via overhead / pyspark memory. citeturn2view0turn3view0  

### Understand failure modes: JVM OOM vs “container killed”

On enforced cluster managers, exceeding the container limit is catastrophic even if the JVM heap itself is fine:

- In YARN strict cgroup enforcement, Hadoop documentation states containers can be preempted “right away” using the kernel OOM killer when reaching memory limits. citeturn18view0  
- In Kubernetes, the official docs explain a container is not allowed to use more memory than its limit; if it continues to consume beyond its limit, it can be terminated. citeturn18view2  

This is exactly why Spark added a Python memory limit feature and described it as helping avoid YARN killing containers when Python doesn’t manage memory under constraints. citeturn22view0turn28view0  

### Prefer fewer executor cores for Python-heavy stages

Given Spark’s Python runner behavior (worker pool growth aligned with executor cores) and your per-worker memory requirement, **executor core count becomes a first-class memory control lever**. citeturn27view0  

A single 4-core executor that runs 4 Python-heavy tasks simultaneously implies you may need ~24–32 GiB of Python memory budget alone, plus heap and overhead. The same total parallelism can often be achieved by running **more 1–2 core executors** instead, trading higher per-executor baseline overhead for much lower peak per-executor Python concurrency. citeturn27view0turn2view0  

### Use `spark.executor.pyspark.memory` when you want predictable Python limits

If you set `spark.executor.pyspark.memory`, Spark enforces a per-worker RLIMIT derived from an executor-level allocation divided by cores. citeturn27view0turn25view1turn28view0  

This can turn “mysterious container killed” outcomes into Python-side `MemoryError` failures, which may be easier to debug and tune. citeturn22view0turn28view0  

Caveat: Spark documents platform limitations (Windows unsupported; macOS not truly limiting). citeturn3view0turn28view0  

### Treat off-heap and Arrow as first-class memory consumers

If you enable Spark off-heap (`spark.memory.offHeap.enabled=true`, `spark.memory.offHeap.size>0`), Spark notes this has no impact on heap usage and you must ensure total memory fits your hard limit. citeturn2view4  

For pandas/Arrow workflows, Spark’s config surface includes multiple Arrow-related settings and concurrency controls that can change memory behavior, and Python workers are only launched when Python execution is required. citeturn16view0turn1view0  

### Monitoring and validation

Spark’s PySpark debugging guide shows practical OS-level validation:

- On executors, you can identify Python worker processes forked from `pyspark.daemon` and inspect memory usage via `ps` / `top`. citeturn16view1  

You should also validate that Spark picked up your intended configuration via the Spark UI “Environment” tab; Spark docs explicitly recommend using the UI for verification and describe how configs are loaded. citeturn20view0turn16view0  

## Concrete configuration examples

### Per-application `spark-submit` examples

These examples emphasize executor sizing for a 6 GB Python worker case on an enforced cluster manager (YARN or Kubernetes), using explicit overhead and (optionally) pyspark memory.

Example A: 2-core executors (≈ 2 concurrent Python workers), Python capped to ~7g per worker (14g total), plus 2g overhead, 6g heap:

```bash
spark-submit \
  --master <yarn-or-k8s-master-url> \
  --deploy-mode cluster \
  --executor-cores 2 \
  --executor-memory 6g \
  --conf spark.executor.memoryOverhead=2g \
  --conf spark.executor.pyspark.memory=14g \
  --conf spark.python.worker.reuse=true \
  --conf spark.pyspark.python=python3 \
  your_app.py
```

The logic behind the numbers follows Spark’s documented container-memory sum and its PySpark memory semantics. citeturn2view0turn3view0turn27view0turn11view0turn11view4  

Example B: 1-core executors (≈ 1 concurrent Python worker), smaller per-executor footprint:

```bash
spark-submit \
  --master <yarn-or-k8s-master-url> \
  --deploy-mode cluster \
  --executor-cores 1 \
  --executor-memory 6g \
  --conf spark.executor.memoryOverhead=2g \
  --conf spark.executor.pyspark.memory=7g \
  your_app.py
```

If you do **not** set `spark.executor.pyspark.memory`, then Python should be budgeted inside `spark.executor.memoryOverhead` instead. citeturn2view0turn3view0  

Spark’s docs confirm typical `spark-submit` usage patterns and how it loads options and properties files. citeturn20view1turn20view0  

### `spark-defaults.conf` cluster defaults vs per-application overrides

Spark’s configuration docs show `spark-submit` reads `conf/spark-defaults.conf` and explain the precedence rules; `--conf` overrides defaults, and a `--properties-file` can replace defaults unless `--load-spark-defaults` is used. citeturn20view0turn20view1  

An example `spark-defaults.conf` baseline for Python-intensive workloads (adjust to your cluster’s per-container limits):

```properties
# Executor sizing
spark.executor.cores                2
spark.executor.memory               6g
spark.executor.memoryOverhead       2g
spark.executor.pyspark.memory       14g

# PySpark runtime
spark.pyspark.python                python3
spark.python.worker.reuse           true
```

Spark documents the meanings of each property above (including PySpark interpreter precedence and worker reuse). citeturn3view0turn2view0turn11view0turn11view4  

Per-application override example (using `--conf`):

```bash
spark-submit \
  --conf spark.executor.cores=1 \
  --conf spark.executor.pyspark.memory=7g \
  --conf spark.executor.memoryOverhead=2g \
  your_app.py
```

This override precedence is described explicitly in Spark’s configuration docs. citeturn20view0turn20view1  

### Kubernetes and YARN-specific operational reminders

- Kubernetes: Spark’s Kubernetes page states memory request/limit is derived from `spark.{driver,executor}.memory + spark.{driver,executor}.memoryOverhead`, and Kubernetes itself enforces memory limits via container termination when exceeded. citeturn8view0turn18view2  
- YARN: Spark’s YARN page explains deploy modes and that executors / application masters run inside YARN “containers”; Hadoop’s YARN docs describe memory enforcement modes and how strict enforcement can preempt containers at the limit. citeturn5view0turn18view0  

In practice, regardless of YARN vs Kubernetes, the critical point is that **Python worker RSS counts against the same container/pod memory limit** that Spark must request upfront, and Spark’s own executor memory accounting includes heap, overhead, off-heap, and pyspark memory in the container budget. citeturn2view0turn3view0turn18view2
