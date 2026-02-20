# Configuring Spark Driver and Executor Memory for PySpark Python Workers

## Executive summary

Your measured requirementâ€”**~6 GB RAM per Python worker process**â€”almost always means you should **budget memory outside the executor JVM heap**, i.e. via **`spark.executor.memoryOverhead`** and/or **`spark.executor.pyspark.memory`**, not by simply inflating **`spark.executor.memory`** (the JVM heap). Sparkâ€™s own configuration docs explicitly separate executor heap (`spark.executor.memory`) from â€œadditional / non-heapâ€ memory (`spark.executor.memoryOverhead`) and note that this additional memory covers non-JVM processes in the same container, including PySpark when `spark.executor.pyspark.memory` is not set. îˆ€citeîˆ‚turn3view0îˆ‚turn2view0îˆ

For a Python-heavy stage where **every concurrent task runs Python**, the executor will typically run **up to one Python worker per concurrent task**; Sparkâ€™s Python runner code notes the worker pool can grow to the number of concurrent tasks, which is determined by executor cores. îˆ€citeîˆ‚turn27view0îˆ‚turn32view0îˆ With a 6 GB-per-worker peak, a 4-core executor can easily need **~24 GB just for Python workers**, plus JVM heap and non-heap/native overheadâ€”far beyond Sparkâ€™s default overhead calculation (often ~10% of heap, min 384 MiB). îˆ€citeîˆ‚turn2view0îˆ‚turn27view0îˆ

Actionable recommendation for your case (assumptions stated):

- **Assumptions**: (a) you are on a containerized cluster manager that enforces memory limits such as îˆ€entityîˆ‚["organization","Apache Hadoop YARN","resource manager"]îˆ or îˆ€entityîˆ‚["organization","Kubernetes","container orchestration"]îˆ (this is where Sparkâ€™s overhead settings apply), (b) executors run on Linux (so Python `resource` limits are available), and (c) Python runs in tasks (UDFs/RDD ops) rather than in the driver only. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ‚turn28view0îˆ‚turn16view0îˆ  
- **Start by controlling Python concurrency per executor**: prefer **1â€“2 executor cores** for Python-heavy work so you donâ€™t spawn 4 Python workers per executor by default. Sparkâ€™s Python runner associates Python worker concurrency with executor cores. îˆ€citeîˆ‚turn27view0îˆ  
- **Budget Python memory explicitly**:
  - If you want the safest â€œdonâ€™t get container-killedâ€ approach: raise **`spark.executor.memoryOverhead`** until it covers **(concurrent Python workers Ã— 6 GB) + native/JVM overhead + headroom**. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ  
  - If you prefer Python to fail fast with a clearer `MemoryError` rather than a container kill: also set **`spark.executor.pyspark.memory`** (total per executor), which Spark divides by executor cores and enforces via `resource.RLIMIT_AS` inside each Python worker. îˆ€citeîˆ‚turn3view0îˆ‚turn27view0îˆ‚turn28view0îˆ‚turn22view0îˆ  
- A pragmatic starting point (with ~15% headroom over 6 GB, i.e. **7 GiB per Python worker budget**) is:
  - **1-core executor**: `spark.executor.memory=6g`, `spark.executor.pyspark.memory=7g`, `spark.executor.memoryOverhead=2g` (â‰ˆ 15 GiB total container)  
  - **2-core executor**: `spark.executor.memory=6g`, `spark.executor.pyspark.memory=14g`, `spark.executor.memoryOverhead=2g` (â‰ˆ 22 GiB total container)  
  You then increase heap if Spark (JVM-side) memory pressure demands it, but you increase overhead / pyspark memory to satisfy the 6 GB Python worker requirement. îˆ€citeîˆ‚turn3view0îˆ‚turn2view0îˆ‚turn10view0îˆ

## Spark and PySpark memory architecture

Spark applications have a **driver** and one or more **executors**. On YARN, Spark documentation distinguishes deploy modes: in **cluster mode**, the driver runs inside the YARN ApplicationMaster; in **client mode**, the driver runs in the submitting client process and the ApplicationMaster mainly requests resources. îˆ€citeîˆ‚turn5view0îˆ

### Executor memory is not a single pool

From Sparkâ€™s configuration docs, the executorâ€™s memory footprint (as enforced by YARN/Kubernetes) is composed of multiple pieces:

- **Executor JVM heap**: `spark.executor.memory` (this is the memory Spark uses to size the executor heap; you should not set `-Xmx` via `spark.executor.extraJavaOptions`). îˆ€citeîˆ‚turn3view0îˆ‚turn31view0îˆ  
- **Executor non-heap / â€œoverheadâ€**: `spark.executor.memoryOverhead`, intended for VM overheads and â€œother native overheads,â€ and it is explicitly supported on YARN and Kubernetes. îˆ€citeîˆ‚turn2view0îˆ  
- **Optional executor off-heap**: `spark.memory.offHeap.enabled` + `spark.memory.offHeap.size`; Spark notes off-heap size does not affect heap, and if total memory must fit a hard limit you must account for it. îˆ€citeîˆ‚turn2view4îˆ  
- **Optional explicit PySpark allocation**: `spark.executor.pyspark.memory`; if set, Spark says PySpark memory in each executor is limited to that amount and (for YARN/Kubernetes) is added to resource requests. îˆ€citeîˆ‚turn3view0îˆ

Sparkâ€™s own executor overhead doc statement is especially important for PySpark: it notes that â€œadditional memory includes PySpark executor memory (when `spark.executor.pyspark.memory` is not configured)â€ and that the **maximum container memory** for an executor is determined by the sum of `spark.executor.memoryOverhead`, `spark.executor.memory`, `spark.memory.offHeap.size`, and `spark.executor.pyspark.memory`. îˆ€citeîˆ‚turn2view0îˆ

### Unified memory inside the JVM heap

Inside the executor heap, Sparkâ€™s tuning guide describes a unified region for **execution** (shuffles, joins, aggregations) and **storage** (caching), governed by:

- `spark.memory.fraction` (default 0.6 of (heap âˆ’ 300 MiB))  
- `spark.memory.storageFraction` (default 0.5 of the memory fraction region) îˆ€citeîˆ‚turn10view0îˆ‚turn2view4îˆ  

These settings influence how Spark uses **heap** memory; they **do not** directly provision memory for external Python processes (which run outside the JVM). îˆ€citeîˆ‚turn10view0îˆ‚turn2view0îˆ

### How PySpark launches and uses Python workers

Sparkâ€™s PySpark user guide (â€œDebugging PySparkâ€) states:

- On executors, **Python workers** execute Python-native functions/data and are launched **only when needed** (Python UDFs, pandas UDFs, PySpark RDD APIs). îˆ€citeîˆ‚turn16view0îˆ  
- In process listings, Python workers can be seen as processes forked from `pyspark.daemon`. îˆ€citeîˆ‚turn16view1îˆ  

Sparkâ€™s source code explains the launch mechanism: `PythonWorkerFactory` prefers starting a **single Python daemon** (by default `pyspark.daemon`, i.e. `pyspark/daemon.py`) and asking it to **fork new worker processes** for tasks, because forking from Java is expensive; it can fall back to launching workers directly (by default `pyspark.worker`) and daemon mode is UNIX-oriented. îˆ€citeîˆ‚turn15view0îˆ  

Those Python processes are **separate OS processes** from the executor JVM. Their memory consumption is therefore **outside** the JVM heap, but still counts against the executor container/pod memory limit that YARN/Kubernetes enforce. îˆ€citeîˆ‚turn2view0îˆ‚turn18view0îˆ‚turn18view2îˆ  

### Executor memory layout flowchart

The following summarizes how memory â€œfits togetherâ€ for one executor on YARN/Kubernetes, consistent with Sparkâ€™s documented container-memory sum and PySpark worker launch behavior. îˆ€citeîˆ‚turn2view0îˆ‚turn15view0îˆ‚turn16view0îˆ

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

- `spark.executor.memory`: executor JVM memory setting (heap sizing). îˆ€citeîˆ‚turn3view0îˆ‚turn31view0îˆ  
- `spark.executor.memoryOverhead`: executor â€œadditionalâ€/non-heap memory, supported on YARN and Kubernetes; defaults to `executorMemory * spark.executor.memoryOverheadFactor` with a minimum `spark.executor.minMemoryOverhead`. îˆ€citeîˆ‚turn2view0îˆ  
- `spark.executor.memoryOverheadFactor` (default 0.10; ignored if `spark.executor.memoryOverhead` is set): fraction used to compute overhead; Spark notes a higher default (0.40) for â€œKubernetes non-JVM jobs.â€ îˆ€citeîˆ‚turn2view0îˆ‚turn8view1îˆ  
- `spark.executor.minMemoryOverhead` (default 384m): minimum overhead if not set explicitly. îˆ€citeîˆ‚turn2view0îˆ  
- `spark.executor.pyspark.memory`: PySpark memory allocation per executor; if set, PySpark memory is limited and added to YARN/Kubernetes resource requests; Spark notes platform limitations inherited from Pythonâ€™s `resource` module. îˆ€citeîˆ‚turn3view0îˆ‚turn28view0îˆ  

Driver-side (relevant if Python does heavy work on the driver):

- `spark.driver.memory`: driver JVM memory; Spark notes you must not set this through `SparkConf` in client mode because the driver JVM already startedâ€”use `--driver-memory` or a properties file. îˆ€citeîˆ‚turn2view3îˆ  
- `spark.driver.memoryOverhead`: driver non-heap memory in cluster mode supported on YARN and Kubernetes; Spark explicitly includes â€œpython process that goes with a PySpark driverâ€ in this non-heap memory. îˆ€citeîˆ‚turn2view2îˆ‚turn2view3îˆ  

PySpark / Python runtime selection and worker behavior:

- `spark.pyspark.python` and `spark.pyspark.driver.python`: Python executables (executors + driver, respectively). îˆ€citeîˆ‚turn11view4îˆ  
- Environment-variable precedence: Spark notes `spark.pyspark.python` takes precedence over `PYSPARK_PYTHON`, and `spark.pyspark.driver.python` over `PYSPARK_DRIVER_PYTHON`. îˆ€citeîˆ‚turn11view4îˆ  
- `spark.python.worker.reuse` (default true): reuse Python workers; Spark describes this as using a fixed number of Python workers and avoiding forking for every task. îˆ€citeîˆ‚turn11view0îˆ  
- `spark.python.worker.memory` (default 512m): memory used per Python worker â€œduring aggregationâ€; if exceeded, it spills to disk (this is distinct from overall Python process memory limits). îˆ€citeîˆ‚turn11view6îˆ‚turn22view1îˆ  
- `spark.python.worker.killOnIdleTimeout` and related idle pool controls exist in recent Spark and can affect how long idle workers persist. îˆ€citeîˆ‚turn11view0îˆ  

Cluster-manager specifics:

- On Kubernetes, Sparkâ€™s Kubernetes doc states the driver/executor pod memory request/limit is set by the **sum** of `spark.{driver,executor}.memory` and `spark.{driver,executor}.memoryOverhead`. îˆ€citeîˆ‚turn8view0îˆ  
- Kubernetes also has `spark.kubernetes.memoryOverheadFactor` to size non-JVM memory overhead for pods. îˆ€citeîˆ‚turn8view1îˆ  
- On YARN, there is a YARN-specific `spark.yarn.am.memoryOverhead` for the ApplicationMaster in client mode (described as â€œsame as `spark.driver.memoryOverhead`â€ but for the AM). îˆ€citeîˆ‚turn6view0îˆ  

Legacy note: older Spark setups and logs may show `spark.yarn.executor.memoryOverhead` / `spark.yarn.driver.memoryOverhead` as deprecated in favor of `spark.executor.memoryOverhead` / `spark.driver.memoryOverhead`. îˆ€citeîˆ‚turn29search3îˆ‚turn20view0îˆ  

### Units and defaults that matter operationally

- `spark.executor.memory` and `spark.driver.memory` are specified as JVM memory strings with suffixes like `m` and `g`. îˆ€citeîˆ‚turn3view0îˆ‚turn2view3îˆ  
- `spark.executor.memoryOverhead` is described â€œin MiB unless otherwise specifiedâ€ (so a bare number is interpreted as MiB; explicit units like `384m` are also used in docs). îˆ€citeîˆ‚turn2view0îˆ  
- `spark.executor.pyspark.memory` is described â€œin MiB unless otherwise specifiedâ€ and depends on Pythonâ€™s `resource` module; Spark notes Windows does not support resource limiting and macOS does not actually limit resources. îˆ€citeîˆ‚turn3view0îˆ‚turn28view0îˆ  

### Precedence and where to set what

Sparkâ€™s config docs provide a clear precedence:

1. Properties set directly on the **SparkConf** (highest precedence)  
2. Then `--conf` flags or `--properties-file` passed to `spark-submit` / `spark-shell`  
3. Then `spark-defaults.conf` îˆ€citeîˆ‚turn20view0îˆ‚turn20view1îˆ  

Spark also notes that if you pass a `--properties-file`, Spark does not load `conf/spark-defaults.conf` unless you add `--load-spark-defaults`. îˆ€citeîˆ‚turn20view0îˆ‚turn20view1îˆ  

## Sizing methodology for a 6 GB Python worker

### Step one: compute how many Python workers can run concurrently per executor

In PySpark, the relevant quantity is not â€œPython workers per executor over time,â€ but **peak concurrent Python workers**, because that determines peak memory consumption.

Two source-backed facts help:

- Sparkâ€™s Python runner notes the worker pool â€œwill grow to the number of concurrent tasks, which is determined by the number of cores in this executor.â€ îˆ€citeîˆ‚turn27view0îˆ  
- Sparkâ€™s scheduler uses `spark.executor.cores` and `spark.task.cpus` to compute â€œnumber of slotsâ€ in at least some scheduling contexts. îˆ€citeîˆ‚turn32view0îˆ  

A practical approximation for Python-heavy stages is therefore:

**`N_py_concurrent â‰ˆ floor(spark.executor.cores / spark.task.cpus)`**, and in the common case `spark.task.cpus=1`, **`N_py_concurrent â‰ˆ spark.executor.cores`**. îˆ€citeîˆ‚turn32view0îˆ‚turn27view0îˆ  

### Step two: understand what memory â€œbucketâ€ Python uses

If you do nothing special, a Python workerâ€™s RSS is **not part of `spark.executor.memory`**; it is memory â€œoutside the JVM,â€ which Spark expects to fit under overhead/non-heap space and the container limit. Spark explicitly documents that executor â€œadditional memoryâ€ includes PySpark executor memory when `spark.executor.pyspark.memory` is not configured. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ  

### Step three: choose your control strategy

There are two mainstream strategies (they can be combined):

#### Strategy A: fund Python using `spark.executor.memoryOverhead`

- You set `spark.executor.memoryOverhead = baseline_nonheap + N_py_concurrent Ã— python_worker_peak + headroom`.  
- This is the simplest mental model and works because Spark requests container memory as a sum that includes `spark.executor.memoryOverhead`. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ  

Trade-off: Spark does not inherently stop Python from consuming beyond â€œits share,â€ so an RSS spike can still lead to a **container kill**.

#### Strategy B: set `spark.executor.pyspark.memory` to cap Python

Spark added `spark.executor.pyspark.memory` to limit Pythonâ€™s addressable memory space (using `resource.RLIMIT_AS`) and avoid situations where YARN kills containers because Python doesnâ€™t know its constraints. îˆ€citeîˆ‚turn22view0îˆ‚turn28view0îˆ  

The mechanics matter:

- In Sparkâ€™s Scala code, the executorâ€™s PySpark memory allocation is divided by **executor cores** so â€œeach python worker gets an equal part of the allocation,â€ and it sets `PYSPARK_EXECUTOR_MEMORY_MB` accordingly. îˆ€citeîˆ‚turn27view0îˆ  
- In the Python worker, Spark reads `PYSPARK_EXECUTOR_MEMORY_MB` and uses `resource.RLIMIT_AS` to set the limit (with platform caveats). îˆ€citeîˆ‚turn25view1îˆ‚turn28view0îˆ  

Therefore, to support ~6 GB per Python worker with `C` executor cores (and thus up to `C` concurrent workers), you typically want:

**`spark.executor.pyspark.memory â‰ˆ C Ã— (python_worker_peak + headroom)`**. îˆ€citeîˆ‚turn27view0îˆ‚turn3view0îˆ  

### Step four: compute total executor container memory

Per Spark docs, on YARN/Kubernetes, the executor container memory upper bound is:

**`M_executor_container = spark.executor.memory + spark.executor.memoryOverhead + spark.memory.offHeap.size + spark.executor.pyspark.memory`**. îˆ€citeîˆ‚turn2view0îˆ‚turn2view4îˆ  

If you are not using off-heap and not setting `spark.executor.pyspark.memory`, Python must fit in overhead; if you do set `spark.executor.pyspark.memory`, it is added to requests as well. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ  

### Numeric examples and a configuration table for your 6 GB worker

Assumptions for the table:

- Peak Python worker RSS measured: **6 GB** (your measurement).  
- Headroom: **~15%** (rounded up to **7 GiB per worker budget**) as a practical starting point (tune based on real peaks).  
- Baseline non-heap/native/JVM overhead: **2 GiB** (thread stacks, metaspace, direct buffers, plus miscellaneous non-JVM processes). Spark describes overhead as accounting for â€œVM overheadsâ€¦ other native overheads.â€ îˆ€citeîˆ‚turn2view0îˆ‚turn2view2îˆ  
- Executor JVM heap (`spark.executor.memory`): **6 GiB** as a baseline; tune upward if Spark JVM-side spills/OOMs occur, but keep in mind GC trade-offs. îˆ€citeîˆ‚turn31view0îˆ‚turn10view0îˆ‚turn30view1îˆ  

Table interpretation: â€œTotal containerâ€ corresponds to Sparkâ€™s documented sum for executors. îˆ€citeîˆ‚turn2view0îˆ‚turn2view4îˆ  

| Executor cores (â‰ˆ max concurrent Python workers) | Recommended `spark.executor.memory` (heap) | Recommended `spark.executor.memoryOverhead` | Optional `spark.executor.pyspark.memory` (Python cap) | Resulting total executor container memory |
|---:|---:|---:|---:|---:|
| 1 | 6g | 2g + (1 Ã— 7g) = **9g** (Strategy A) **or** **2g** (Strategy B) | 7g (Strategy B) | **15g** |
| 2 | 6g | 2g + (2 Ã— 7g) = **16g** (Strategy A) **or** **2g** (Strategy B) | 14g (Strategy B) | **22g** |
| 4 | 6g | 2g + (4 Ã— 7g) = **30g** (Strategy A) **or** **2g** (Strategy B) | 28g (Strategy B) | **36g** |

Why the overhead numbers dwarf Spark defaults: by default, `spark.executor.memoryOverhead` is computed as `executorMemory Ã— spark.executor.memoryOverheadFactor` (default 0.10) with a minimum `spark.executor.minMemoryOverhead` (384m). With a 6g heap, the default overhead is on the order of hundreds of MiBâ€”nowhere near a 6 GB Python process. îˆ€citeîˆ‚turn2view0îˆ  

#### Should you increase executor heap or memoryOverhead for a 6 GB Python process?

- The **6 GB Python worker** should be funded by **`spark.executor.memoryOverhead`** and/or **`spark.executor.pyspark.memory`**, because Python workers are **separate processes** and Sparkâ€™s overhead description explicitly covers non-JVM memory and PySpark (when pyspark memory is not configured). îˆ€citeîˆ‚turn2view0îˆ‚turn16view0îˆ‚turn15view0îˆ  
- You increase **executor heap** (`spark.executor.memory`) when the **JVM side** needs more headroom: caching, shuffle-heavy joins/aggregations/sorts, large broadcast materialization, etc., whose memory behavior is governed by Sparkâ€™s unified memory manager and GC behavior. îˆ€citeîˆ‚turn10view0îˆ‚turn30view1îˆ  

### Driver sizing when Python runs on the driver

Sparkâ€™s driver overhead doc is explicit: driver non-heap memory includes memory used by other driver processes, such as â€œpython process that goes with a PySpark driver,â€ and the driver container limit (cluster mode on YARN/Kubernetes) is the sum of `spark.driver.memory` and `spark.driver.memoryOverhead`. îˆ€citeîˆ‚turn2view2îˆ‚turn2view3îˆ  

Practical implication:

- If your pipeline does significant driver-side Python work (e.g., `collect()` then pandas processing, heavy local modeling, large `toPandas()`), you must budget **driver memory + driver overhead** accordingly, especially in cluster mode. îˆ€citeîˆ‚turn2view2îˆ‚turn2view3îˆ  
- In client mode, Spark warns you cannot effectively set driver heap via `SparkConf` after startup; you should set it via `--driver-memory` or your properties file. îˆ€citeîˆ‚turn2view3îˆ‚turn20view0îˆ  

## Best practices and trade-offs

### Avoid â€œfixingâ€ Python memory problems by inflating JVM heap

Over-allocating `spark.executor.memory` may increase the default overhead fraction (if you rely on the default overhead factor), but it carries GC trade-offs. Sparkâ€™s tuning guide emphasizes that GC cost is proportional to object count and discusses strategies like serialized caching to reduce GC overhead; it also notes Sparkâ€™s default GC (G1GC) and that large heaps may require GC tuning. îˆ€citeîˆ‚turn30view0îˆ‚turn30view2îˆ  

A better approach is to keep heap â€œreasonableâ€ for the JVM workload and allocate **separately** for Python via overhead / pyspark memory. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ  

### Understand failure modes: JVM OOM vs â€œcontainer killedâ€

On enforced cluster managers, exceeding the container limit is catastrophic even if the JVM heap itself is fine:

- In YARN strict cgroup enforcement, Hadoop documentation states containers can be preempted â€œright awayâ€ using the kernel OOM killer when reaching memory limits. îˆ€citeîˆ‚turn18view0îˆ  
- In Kubernetes, the official docs explain a container is not allowed to use more memory than its limit; if it continues to consume beyond its limit, it can be terminated. îˆ€citeîˆ‚turn18view2îˆ  

This is exactly why Spark added a Python memory limit feature and described it as helping avoid YARN killing containers when Python doesnâ€™t manage memory under constraints. îˆ€citeîˆ‚turn22view0îˆ‚turn28view0îˆ  

### Prefer fewer executor cores for Python-heavy stages

Given Sparkâ€™s Python runner behavior (worker pool growth aligned with executor cores) and your per-worker memory requirement, **executor core count becomes a first-class memory control lever**. îˆ€citeîˆ‚turn27view0îˆ  

A single 4-core executor that runs 4 Python-heavy tasks simultaneously implies you may need ~24â€“32 GiB of Python memory budget alone, plus heap and overhead. The same total parallelism can often be achieved by running **more 1â€“2 core executors** instead, trading higher per-executor baseline overhead for much lower peak per-executor Python concurrency. îˆ€citeîˆ‚turn27view0îˆ‚turn2view0îˆ  

### Use `spark.executor.pyspark.memory` when you want predictable Python limits

If you set `spark.executor.pyspark.memory`, Spark enforces a per-worker RLIMIT derived from an executor-level allocation divided by cores. îˆ€citeîˆ‚turn27view0îˆ‚turn25view1îˆ‚turn28view0îˆ  

This can turn â€œmysterious container killedâ€ outcomes into Python-side `MemoryError` failures, which may be easier to debug and tune. îˆ€citeîˆ‚turn22view0îˆ‚turn28view0îˆ  

Caveat: Spark documents platform limitations (Windows unsupported; macOS not truly limiting). îˆ€citeîˆ‚turn3view0îˆ‚turn28view0îˆ  

### Treat off-heap and Arrow as first-class memory consumers

If you enable Spark off-heap (`spark.memory.offHeap.enabled=true`, `spark.memory.offHeap.size>0`), Spark notes this has no impact on heap usage and you must ensure total memory fits your hard limit. îˆ€citeîˆ‚turn2view4îˆ  

For pandas/Arrow workflows, Sparkâ€™s config surface includes multiple Arrow-related settings and concurrency controls that can change memory behavior, and Python workers are only launched when Python execution is required. îˆ€citeîˆ‚turn16view0îˆ‚turn1view0îˆ  

### Monitoring and validation

Sparkâ€™s PySpark debugging guide shows practical OS-level validation:

- On executors, you can identify Python worker processes forked from `pyspark.daemon` and inspect memory usage via `ps` / `top`. îˆ€citeîˆ‚turn16view1îˆ  

You should also validate that Spark picked up your intended configuration via the Spark UI â€œEnvironmentâ€ tab; Spark docs explicitly recommend using the UI for verification and describe how configs are loaded. îˆ€citeîˆ‚turn20view0îˆ‚turn16view0îˆ  

## Concrete configuration examples

### Per-application `spark-submit` examples

These examples emphasize executor sizing for a 6 GB Python worker case on an enforced cluster manager (YARN or Kubernetes), using explicit overhead and (optionally) pyspark memory.

Example A: 2-core executors (â‰ˆ 2 concurrent Python workers), Python capped to ~7g per worker (14g total), plus 2g overhead, 6g heap:

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

The logic behind the numbers follows Sparkâ€™s documented container-memory sum and its PySpark memory semantics. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ‚turn27view0îˆ‚turn11view0îˆ‚turn11view4îˆ  

Example B: 1-core executors (â‰ˆ 1 concurrent Python worker), smaller per-executor footprint:

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

If you do **not** set `spark.executor.pyspark.memory`, then Python should be budgeted inside `spark.executor.memoryOverhead` instead. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ  

Sparkâ€™s docs confirm typical `spark-submit` usage patterns and how it loads options and properties files. îˆ€citeîˆ‚turn20view1îˆ‚turn20view0îˆ  

### `spark-defaults.conf` cluster defaults vs per-application overrides

Sparkâ€™s configuration docs show `spark-submit` reads `conf/spark-defaults.conf` and explain the precedence rules; `--conf` overrides defaults, and a `--properties-file` can replace defaults unless `--load-spark-defaults` is used. îˆ€citeîˆ‚turn20view0îˆ‚turn20view1îˆ  

An example `spark-defaults.conf` baseline for Python-intensive workloads (adjust to your clusterâ€™s per-container limits):

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

Spark documents the meanings of each property above (including PySpark interpreter precedence and worker reuse). îˆ€citeîˆ‚turn3view0îˆ‚turn2view0îˆ‚turn11view0îˆ‚turn11view4îˆ  

Per-application override example (using `--conf`):

```bash
spark-submit \
  --conf spark.executor.cores=1 \
  --conf spark.executor.pyspark.memory=7g \
  --conf spark.executor.memoryOverhead=2g \
  your_app.py
```

This override precedence is described explicitly in Sparkâ€™s configuration docs. îˆ€citeîˆ‚turn20view0îˆ‚turn20view1îˆ  

### Kubernetes and YARN-specific operational reminders

- Kubernetes: Sparkâ€™s Kubernetes page states memory request/limit is derived from `spark.{driver,executor}.memory + spark.{driver,executor}.memoryOverhead`, and Kubernetes itself enforces memory limits via container termination when exceeded. îˆ€citeîˆ‚turn8view0îˆ‚turn18view2îˆ  
- YARN: Sparkâ€™s YARN page explains deploy modes and that executors / application masters run inside YARN â€œcontainersâ€; Hadoopâ€™s YARN docs describe memory enforcement modes and how strict enforcement can preempt containers at the limit. îˆ€citeîˆ‚turn5view0îˆ‚turn18view0îˆ  

In practice, regardless of YARN vs Kubernetes, the critical point is that **Python worker RSS counts against the same container/pod memory limit** that Spark must request upfront, and Sparkâ€™s own executor memory accounting includes heap, overhead, off-heap, and pyspark memory in the container budget. îˆ€citeîˆ‚turn2view0îˆ‚turn3view0îˆ‚turn18view2îˆ
