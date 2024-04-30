# Cloud Data Warehousing
Reminder: Key aspects of **DW workloads**:
- Read-mostly. Can (often) ignore concurrency or at least settle for snapshot semantics on reads, can defer transactional update.
- Relatively high Latency expected
- Bottleneck typically data bandwidth, not compute

Analytics/Data Warehousing conventional wisdom today: 
- SQL is ubiquitous, has largely displaced MapReduce/Spark/etc.
- UDFs important (including async UDFs like ML)
- Some folks *will* want to drop into Spark/MapReduce
- Data Lakes: data in situ, not necessarily cleanly relational. "ELT" beats "ETL".

We know about Gamma. We know about MapReduce. **What changes in the Cloud?**

1. Delegate components to good-enough subsystems, and their dev *and ops* teams
   1. Most inexpensive storage is shared object storage (S3). High latency, high bandwidth. Good for DW!
      1. Also handles encryption/auth in a unified setting
   2. Distributed resource allocation/management/membership handled by K8s and co.
2. Elasticity is available: what shall we use it for and how?
3. Need to address global reach and geolatencies.
4. Lots of shared work across queries/users
5. Diversity of HW
6. SLOs. E.g. "best effort" vs "reserved"


**What technical challenges arise?**
- Runtime scheduling and resource allocation
  - E.g. custom optimized shuffle services, join services. "Disaggregated memory" in BigQuery/Dremel shuffle.
  - "Arguably the biggest engineering challenge in Polaris is orchestration of tasks."
- Query Optimization w/some dynamics
  - Unknown data stats
  - FT and its relation to performance smoothing
  - Matching SLO budget/perf tradeoffs
  - Multiquery optimization and scheduling

**What are some Design Space considerations?**
- As many tiers as you like
  - Physical: memory, low-latency storage, high-latency storage
  - Logical/Materialization: georeplication, storage formats/compression, ELT precomputation, ...
  - Make smart, workload-aware choices
- Fully stateless vs transiently-stateful vs full statefulness a la Gamma
- Monitoring/Scheduler architecture: centralized? (Dremel/BigQuery) Distributed? how distributed?
- Scheduling tricks: batching, sharing, dynamics, warm pools, speculative execution, 
- Query optimization
- Workload-driven heuristics, ML, etc.
- Query processing architecture/orchestration
  - Partitionable dataflow subgraphs as the scheduling tasks, partitionable HW resources as the scheduling units
    - Match tasks to "slots" of compute/memory/storage
    - Break things down smaller than machines to allow flexibility in scheduling, redundancy, restart
    - Predicate pushdown to "storage"

# POLARIS
- Cells of Data, with stats for QO
  - distributions/hashed (for hashing)
  - partitions (for data pruning)
- Cell-aware QO
- Tasks of data/compute work
- Task-level workflow DAG, spans queries
- SQL Server per node scale-up
- "Session" model to support on-demand or reserved modes
- Caching and prefetching
- Stateless workers -- beyond disaggregation


- Cell
  - unit of distribution
    - data extraction from a cell is local-only work
  - cell $C_{ij}$ has all object $r$ st $p(r) = i$, $h(r) = j$. 
    - $h(r)$ is the distribution (hash)
    - $p(r)$ is an optional user partition used for pruning at query time
  - Storage can choose any organization of cells
  - At query time, assignment of cells to compute is in durable metadata state

- Query optimizaer distribution of cells: seems pretty standard
  - an *interesting property* for QO
  - required distribution properties for operators, e.g. $P \bowtie_{a=b} Q: \{P^{[a]} \wedge Q^{[b]}\} \vee P^1 \vee Q^1$
  - Data move enforcers:
    - Hash operator resets $h(r)$ for all objects
    - Broadcast operator
  - best plan for each interesting property
  
- QO
  - Back to Volcano: logical-only memo first
  - Then physical optimization to consider distributed data movement

- QP
  - Some pipelining, but data move (shuffle or broadcast) is always blocking
  - Local tasks given to SQL server in T-SQL to optimize

- Orchestration is tricky -- needs explicit representation!
  - hierarchy of state machines based on task/data dependencies
  - simple states: success, failure, or ready
  - composite states: instantiated or blocked
  - failure states analyzable for potential retry

- Scheduling: Global Workload Graph
  - each node has a *resource demand* vector
  - note distinction between fungible resources (memory, CPU) and more rigid resources (temp space on local disk)
  - Problem: task scheduling with precedence constraints
  - Workpool policies:
    - FIFO, sorted by resource demand (asc or desc)
    - FIFO, sorted by proximity to the root of the DAG
      - jobs close to done can release resources
    - May not be able to get resources for next task, have to try again
  - Target location fixed by data affinity to exploit cache locality