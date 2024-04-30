# In-Memory Analytic DBMS
## History
- Nice first-gen survey by [Garcia-Molina and Salem, TKDE 1992](https://15721.courses.cs.cmu.edu/spring2018/papers/02-inmemory/garciamolina-tkde1992.pdf)
- Like all things, goes back to IBM. See IMS/VS
 FastPath, 1976 (as documented in [This Tech Report](https://www.researchgate.net/publication/220283152_Varieties_of_concurrency_control_in_IMSVS_fast_path/link/543fb3420cf2fd72f99cef50/download?_tp=eyJjb250ZXh0Ijp7ImZpcnN0UGFnZSI6InB1YmxpY2F0aW9uIiwicGFnZSI6InB1YmxpY2F0aW9uIn19))

> *"As DRDB [Disk-Resident DB] perform more and more in-memory optimizations, they become closer to MMDB [Main-Memory DB]. In the future, we expect that the differences between a MMDB and DRDB will disappear: any good database management system will recognize and exploit the fact that some data will reside permanently in memory and should be managed accordingly"*

But what about:
- Concurrency control overheads?
- Memory hierarchy details: Cache vs RAM instead of RAM vs Disk?
- Logging overheads?
- Data representation?
- Indexes?
- Query Processing techniques?
- Recovery?

### Some Early MMDB Systems
Mostly focused on transaction processing (the harder part?)
- IBM Product: IMS/VS FastPath (Dieter Gawlick, D. Kinkade) 1976.
- IBM prototype: Office By Example (Ravi Krishnamurthy, Dina Bitton, Carolyn Turbyfill, and others). 1985
- Wisconsin prototype: MM-DBMS (Toby Lehman, Mike Carey). 1986-87. T-tree indexes (T is for Toby Lehman).
- Various Princeton Prototypes: HALO (Ken Salem, Hector Garcia-Molina) 1987, TPK (Kai LI, Jeff Naughton) 1988, and System M (Ken Salem, Hector Garcia-Molina) 1990 at Princeton
- Bell Labs prototype: Dali. (HV Jagadish, Dan Lieuwen, Rajeev Rastogi, Avi Silberschatz, S. Sudarshan)

We'll read a more modern example via SILO next lecture

### Background/Context
#### "Volcano-style" iterators
- Attributed to Volcano, but pretty standard in lots of DBMSs (e.g. Postgres did this)
- Every operator has `init()`, `next()` and `close()` methods
- Output (top) of the dataflow "pulls" data from children, recursively
- Data flows when popping the call stack (`return()`) in `next()`
- Need a bit of buffering to handle DAGs/Cycles
- Typically assumes "row at a time" processing
    - but of course you can mix-and-match projections and joins as you see fit

#### Columnar Data Representation
Improves analytic performance in both disk-based and MMDB. Originally designed for disk-based systems, but important background for today.

Super simple idea: column-major rather than row-major layout.
- Makes projection cheap, at the cost of "assembly" joins
- Exploit uniformity of type within a column for lightweight compression
- Good RAM Bandwidth, decompress in cache
- Avoid "walking"/interpreting tuple representations
- Naturally vectorizable (as we'll discuss)

History
- [Tutorial by Dan Abadi, Peter Boncz and Stavros Harizopoulos](https://dl.acm.org/doi/pdf/10.14778/1687553.1687625)
- Earliest refs tend to be to Copeland/Khoshafian SIGMOD 1985 *Decomposition Storage Model*
- [According to Wikipedia](https://en.wikipedia.org/wiki/SAP_IQ), initially commercialized by a startup in Boston in a system called Expressway 103, acquired by Sybase and commercialized as Sybase IQ Accelerator in 1995. Most people tip a hat to Sybase IQ as the earliest real columnar database.
    - [Sybase IQ paper in VLDB 2004](http://www.vldb.org/conf/2004/IND8P3.PDF)
- Things begin to get academic, popularized, and fully fleshed out in the early 2000's
    - Adam Sah, Stonebraker student, [Addamark Log Mgmt System](https://www.usenix.org/legacy/events/lisa2002/tech/full_papers/sah/sah.pdf)
    - Peter Boncz PhD thesis with Martin Kersten on MonetDB, 2002
    - C-Store @ MIT: Re-emergence of Mike Stonebraker with large crew (Dan Abadi, Sam Madden, the O'Neils, Mitch Cherniack, Stan Zdonik), 2005. 
- A lot of Google and Hadoop-related storage efforts in this space followed
    - BigTable (2005) is a kind of columnstore
    - Open formats like Parquet and ORC
    - New research in this space

## The European Systems Resurgence: MMDB
Back in the 70s, there was some notable European systems impact, mostly in Germany, on things like B-trees and transactions. But in the 1980s-90s, European DB researchers were *mostly* doing theory with a few exceptions. Systems work centered around US academics and companies.

In the 00's, The Netherlands' CWI (led by Martin Kersten and later Peter Boncz) and Germany's T.U. Munich (led by Thomas Neumann and later also Viktor Leis) emerged as new centers of excellence in main-memory DBMS. Also efforts at Wisconsin (Jignesh Patel's QuickStep), MIT (Stonebraker/Madden et al HStore/Silo) and CMU (Andy Pavlo's Peloton and NoisePage), but even Andy Pavlo gives most credit to the europeans.


Germany's SAP Hana also made a huge bet on in-memory around 2010, and the Hasso Plattner Institute backed that with a bunch of papers.

In terms of direct industrial impact, biggest impact has been on embedded/in-memory:
- CWI: Vectorwise, some Databricks, **DuckDB**
- TUM: HyPer (i.e. **Tableau**)
- SAP **Hana**

Also HStore became VoltDB, Quickstep startup acquired by Pivotal for HAWQ/Greenplum. These infrastructure MMDB plays have been less visible than DuckDB and Hyper/Tableau.

Lots of commercial activity outside this space (e.g. MemSQL/SingleStore, ClickHouse).

## Big Picture: [Everything You Always Wanted to Know](https://www.vldb.org/pvldb/vol11/p2209-kersten.pdf)
- Two approaches:
    - Vectorized query engine, typified by MonetDB/X100
    - Modern compiled queries, typified by HyPer
- Conventional Wisdom after a number of years
    - Generally the compiled (HyPer) approach generates faster code in most cases
    - Vectorized (X100) approach is easier to understand/maintain (??), has 0 compilation overhead
- Some performance tradeoffs
    - Vectorization is better at hiding cache miss latency
    - Compilation requires fewer CPU instructions, benefits cache-resident workloads. 
- This later paper has a a fun, friendly bakeoff
    - Tectorwise vs Typer
    - No single winner across TPC-H queries, though Typer looks a little stronger
    - Much breakdown of details
    - Really nice to see open-mindedness among competing teams!

Theme: 20th century designs were I/O-throttled, ignored in-memory issues
    - Need to optimize memory bandwidth (see X11)
    - Need to optimize register locality (see HyPer)
    - Cache locality addressed in lots of papers, but it turns out it mostly falls out from the above


# MonetDB X100 and Vectorwise

- Arc of innovation
    1. History of Volcano-style, tuple-at-a-time processing has run into I/O problems "up the stack" in the CPU
        - I/O (disk, network, bus) bandwidth bottlenecks resolved
        - Now suffering from memory bandwidth into the CPU: RAM -> Caches -> Registers
        - MySQL profiling, simple TPC-H Query 1:
            - 62% of time spent in `rec_get_nth_field`
            - 28% of time in hashtable management for aggregation
            - Even "real work" is bad: Simple arithmetic goes very slowly: 38 instructions for `+`
    2. What if we do full-column-at-a-time processing (original MonetDB)
        - Also doesn't get good memory bandwidth
            - Better than MySQL but 75 cycles for `*`!!
            - But works great if each column fits in cache!
        - Has to do lots of positional joins that MySQL/Row-oriented does not to "reassemble"
    3. What if we go back to Volcano-style pull, but embrace columnar storage in cache-sized "vectors"?
        - And employ a number of tricks that were developed in column-at-a-time MonetDB!
- Commercialization
    - European startup Vectorwise
    - Eventually acquired by Ingres, of all things! Struggled.
    - Set the stage for DuckDB *many years later*!
        - Timing and patience count for A LOT!
----
- Paper structure: Bottoms-up look at the *problems/opportunities* in "modern" processors
    - boils down to problems of *CPU utilization* relative to memory BW
    - want to grease the memory BW to keep the CPU busy!
- Here's the list of **Solutions/Techniques** in MonetDB/X100
    1. Columnar storage, lightweight compression 
    2. Dataflow of *vectors* (i.e. cache-sized arrays of column volues) rather than tuples
        - Compress right-sized chunks so you can bring from RAM to cache, decompress in cache
            - **[iii](#opt3)**
            > "The CPU cache is the only place where bandwidth doesn't matter"
    3. Vectorized query primitives:
        - Order, Top-N and Select all produce dataflows of vectors of the same *shape* as input!
            - **[iv](#optv)**, **[v](#opt5)**
        - Selection??  Doesn't filter, it creates a *selection vector* of positive positions
            - Preserves offsets in the column for subsequent assembly into tuples via join
            - Downstream ops use selection vector to "skip" negative positions
                - **[v1](#opt6)**, **[ii](#opt2)**
        - GROUP BY/Aggregation
            - "Direct": for small domains, use "identity hash" and a non-colliding table
                - **[vii](#opt7)**
                > GROUP BY on two single-byte characters can never yield more than 65536 combinations, such that their combined bit-representation can be used directly as an array index to the table with aggregation results.
            - Hash: as you'd expect
            - Ordered: as in Selinger 79
              - **[vii](#opt7)**
        - Nested loops join
        - Fetch (Index) Join -- also serves as positional join for "assembling" tuples
        - Expressions/UDFs:
            - Hand-rolled C templating for functions like `sum<double>` vs `sum<double *>`.
                - **[viii](#opt8)**
                - **[ix](#opt9)**
            - Compound expressions as primitives when possible to get more "work" instructions in registers per load/store (to cache)
                - **[i](#opt1)**
    4. Standard Immutable Columnar Storage
        - Deletes go to *delete lists* (tombstones)
        - Inserts/Updates go to un compressed *delta columns*, bundled together for the table til they get big and are re-orged into vertical storage
            - **[x](#opt10)**
        - Lightweight compression, including simple offsets (pointers) to enums as stand-ins for values
        - "summary indices" instead of exact indices, allow for immutable indexing of clustered (mostly-sorted) columns, with false positives

- And now, here's the list of **Problems and Opportunities**:
    1. <a id="opt1">Processor Pipelining:</a> Take advantage of "pipelining" in processors
    2. <a id="opt2">Exploit Super-Scalar Processors:</a> Some processors can process independent instructions in parallel: multiple pipelines per processor. 
        - Special case: *loop pipelining*: This can be done by taking a sequence of dependent ops `(g(f(x)))` and running `f` on many `x`'s, then running `g` on many results.

    3. <a id="opt3">Maximize Memory Bandwidth into Cache:</a> keep the CPU fed with work by greasing any non-cache access!
    4. <a id="opt4">Fixed-Size Units of Work in Cache:</a> avoids (explicit) fragmentation of data in cache, always passing vectors
    5. <a id="opt5">Offset-Based Access to Stored Data:</a> if we keep the column "shape", we can easily reassemble tuples from columns (see join below)
    6. <a id="opt6">Branch Predication:</a> avoid processor branch prediction problems by turning conditionals into multiplication by 0/1. Keeps processor pipelines from "flushing". Also helps super-scalar processors find independent work (**[x](#opt10)**)
    7. <a id="opt7">Avoid Hashing:</a> use positional indexing aggressively for small domains
    8. <a id="opt8">Avoid Function Pointers:</a> don't waste time chasing "vtable" function pointers
    9. <a id="opt9">Instruction Cache Locality:</a> is beneficial
    10. <a id="opt10">Monotonicity on the fast path:</a> Avoid random I/O and contention due to update in place. This pattern of monotonic append on fast path, reorg in background is quite common: Postgres, Log-Structured File System, Log-Structured Merge Trees, many more

# HyPer
Surely a compiler can help us?! Compiled queries are a long tradition in DBMSs, going back to System R!

Where X100 focuses on memory bandwidth, HyPer focuses on register locality!

High-level ideas:

1. Query algebra is for code structuring, but compiler should break that abstraction to keep data in cache
2. "Push" beats "Pull" for register locality (??)
3. In 2011, LLVM is good enough to do what we need, let's use its IR (yuck)! 

Observations:
- Pipeline breaker: pulls tuple out of *registers*.
    - Iterator-style function calls blow registers!
    - A batch of tuples, vector style, also blows registers!
- Full Pipeline breaker: blocking operator (consumes all inputs)
- "Push" from leaf to a pipeline breaker. "Complex control flow logic tends to be outside tight loops, which reduces register pressure." 
    - `produce()`/`consume()` rather than `next()`/`return()`
    - Instruction cache locality is pretty good here: small amounts of code per loop.
    - `produce` has a single consumer (typically) so no conditional logic. By contrast `next()` on a binary op does very different materialization for left and right sides with virtual function calls and frequent memory access.
- Quibbles/Update:
    - The Push/Pull distinction here is *probably* irrelevant if you use a modern compiler like Rust's. Iterators are common in these languages and well handled-- blocks are inlined aggressively. Still, the Push intuition is useful to understand what good machine code looks like.
    - Relatedly, the idea that binary ops with `next()` are alternating sides and using function pointers isn't really a given if you compile iterators intelligently.
    - Would be fun to explore/benchmark this. TIL about [Compiler Explorer](godbolt.org) which makes this stuff relatively easy to play with.
- Main point: code "inlining" boundaries are not at algebraic operator boundaries. 
    - HyPer approach (2011): let us figure this out and generate LLVM
    - Language approach: generate Rust/C++, let the compiler figure this out
        - Would be fun to benchmark, see how to trip up the Rust compiler
        - Note: Higher level languages like Rust/C++ take more time to compile! Not so good for interactive queries
- LLVM IR features
    - Handles register allocation. Just assume unbounded registers that can hold a whole tuple, let the LLVM IR compiler figure it out.
    - "the LLVM assembler is strongly typed, which caught many bugs that were hidden in our original textual C++ code generation." ??
- Lowering compiler overhead:
    - Precompile basic blocks of code for operators (cogwheels), generate LLVM to stitch it together (chains)
    - Could one make Rust/C++ competitive with LLVM codegen today?
- Notes on inlining: it's not going to just be one function because
    1. The LLVM code will most likely call C++ code at some point that will take over the control flow. 
    2. Inlining the complete query logic into a single function can lead to an exponential growth in code. 
- Main point: "one has to make sure that the hot path does not cross a function boundary"
    - > For optimal performance it is important that the hot path, i.e., the code that is executed for 99% of the tuples, is pure LLVM. Calling C++ from time to time (e.g., when switching to a new page) is fine, the costs for that are negligible, but the bulk of the processing has to be done in LLVM. While staying in LLVM, we can keep the tuples in CPU registers all the time, which is about as fast as we can expect to be. When calling an external function all registers have to be spilled to memory, which is somewhat expensive. In absolute terms it is very cheap, of course, as the registers will be spilled on the stack, which is usually in cache, but if this is done millions of times it becomes noticeable.
- Get all this right and new bottlenecks emerge
    - E.g. hashing, so try to do it early and hide latency
    - E.g. branches (and misprediction), so hand-write code to branch less often
    - again, one hopes that compilers in 2024 are better at this than they were in 2011?
- Parallelize:
    - with SIMD instructions where available
        - wide registers
        - multiple tuples per instruction
        - evaluate a chain of predicates on a block rather than one predicate at a time
    - across cores, Gamma/Volcano style


# Recent Work: [InkFuse](https://www.cs.cit.tum.de/fileadmin/w00cfj/dis/papers/inkfuse.pdf)
Hot-off-the-press ICDE24 including TUM and CWI folks!
Goals: best of both
    - 0 compilation time vectorized performance for short queries
    - benefit from compiled code for long queries

# You might ask...specialized HW?
- GPUs?
- ASICs/FPGAs?
- Multicore/NUMA
- RDMA?
- Etc etc?
