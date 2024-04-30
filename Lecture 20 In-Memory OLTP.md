# Lecture 20: In-Memory OLTP
## Background on OLAP and OLTP
- OLTP: On-Line Transaction Processing. Very old term. Sometimes called "Operational Databases".  Textbook transactions as invented by Jim Gray. Typically, simple short-running queries with some updates. Sometimes OLTP tasks are pre-"canned" (SQL "stored procedures" or as our reading says, "one-shot" tasks) because they're just parameterized forms from applications, so the only variation is in the parameters. TPC-C is the classic benchmark.
- OLAP: On-Line Analytic Processing. Originally just called "Decision Support" or "Analytics". Associated with "Data Warehouses", rather than "Operational Databases". Long-running, often complex, read-mostly query workloads with lots of aggregation (statistics).
    - The term OLAP originally had a narrower meaning focuses on Data Cubes. Coined in the early 1990s by one Edgar F Codd, working as a consultant helping to market a product called Essbase. The pitch was supposed to be "revolutionary", with a focus on "multidimensional analysis" -- some attributes are "dimensions" (axes) and some are "measures", forming a "data cube" to be sliced and diced. This narrower flavor of OLAP was the topic of some good research in the 1990s. Mostly reduced to SQL features by now (with extensions like CUBE BY), but some vestiges still exist of the Essbase-style pitch -- e.g. MDX is a "standard" language pushed by Microsoft.
        - A Data Cube is basically a hierarchy of GROUP BY queries. E.g. if you three dimensions like (make, model, color), you might want 
            - sum-of-sales (no GROUP BY)
            - sum-of-sales GROUP BY make
            - sum-of-sales GROUP BY model
            - sum-of-sales GROUP BY color
            - sum-of-sales GROUP BY make, model
            - sum-of-sales GROUP BY make, color
            - sum-of-sales GROUP BY model, color
            - sum-of-sales GROUP BY make, model, color

It is challenging to design a DBMS to do well at both OLTP and OLAP. It is especially challenging to *tune a DBMS deployment* to perform well at both. If you think about the mix of long- and short-running queries, as well as the mix of isolaton-level requirements for the two workloads, you can imagine why: long-running queries and short-running queries don't coexist happily.

Hence conventional wisdom is to buy and/or deploy separate instances for OLTP ("operational DBMSs") and for OLAP ("data warehouse"). This also acknowledges the fact that different OLTP databases tend to "belong" to different applications/devops groups, whereas the data warehouse tends to span all the data and "belong" to cross-cutting "decision makers" -- e.g. management or their "data science" team.

## Where's the MM OLTP Market?
Lots of evidence that the OLTP market has been harder to crack than the OLAP market in the last decade or 2. 

1. I have been told by people involved in the early days of Snowflake that they originally wanted to go after OLTP on Flash, but on analysis there just wasn't a 10-100x win there over conventional technology. Then they looked at OLAP in the cloud and saw opportunity.
2. Viktor Leis at TUM and Andy Pavlo at CMU have done research on (in-memory) OLTP. I asked them about this and they said:
    > Viktor: "the commercial success of in-memory OLTP has been -- let's say -- limited. Almost all the commercial action has been in the analytics space...This is despite lots of innovative research showing that new OLTP designs are orders of magnitude faster than traditional disk-based systems. My personal guess is that this is due to three reasons:
    > - Operational OLTP is a difficult and conservative market.
    > - DRAM prices per GB stopped decreasing. So it turned out that storing everything in DRAM is expensive and does not scale....
    > - The millions of transactions per second that academic in-memory systems can achieve are kind of fantasy numbers. Users don't like to put all their logic into stored procedures and it's very hard to get that many transactions into the DBMS through the network.

    > Andy: "Enterprises still run most of their txn-heavy workloads on Oracle, DB2, Sybase, MSSQL, and NonStop. Postgres is seeing an uptick in adoption but they're not running at the scale of these heavies. HFTs are the only people running hardcore +1m txn/sec workloads but they almost always roll their own systems.
    >
    > Real-world stored-procedures are super rare."

Probably the most widely-deployed in-memory OLTP system is Hekaton, which is built-in to MS SQL Server. I am not aware of how often it gets "used", but I've asked some friends.

## SILO: an in-memory OLTP design
Setup
- In-memory DBMS
- Serializable isolation
- Multicore (of course) with shared memory
- "one-shot" tasks (stored procedures, all parameters known at time of request)
    - No data dependency on external world (user)
    - No user "think time" between BEGIN and COMMIT
- optional snapshots for lighter-weight read-only transactions

Some keys to high-performance
- Infrequent use of centralized coordination
- Never turn local reads into shared-memory writes (e.g. read locks!)

Techniques
- OCC rather than locking
- Relatively infrequent global "epoch counter" rather than transaction counter
- Group Commit at end-of-epoch
- Redo-only logging (after crash there is no DB to UNDO!)
- Co-locate locks with data (a very old design)

Context: H-Store and Masstree
- Previously at MIT, Stonebraker et al's H-Store system, commercialized as VoltDB
- Shard DBMS across cores, hope that transactions shard the same way
    - Works for TPC-C benchmark! (shard by "warehouse")
    - If not, lock entire shards
- Design by benchmarketing? Wishful thinking?
    - I believe VoltDB by now is quite a bit different, but still not a big player
- Silo, in reaction, is not sharded -- all cores access shared data.
- Previously at Harvard, the Masstree in-memory KVS
    - tree/trie hybrid
    - "lock-free", but makes use of atomics (test-and-set) for mutual exclusion
        - not to be confused with "coordination-free"
        - no better than locks under contention (maybe worse)
        - some benefits in the absence of contention: no storage/compute overhead for locks
        - See discussion in [Faleiro/Abadi CIDR 2017](http://www.jmfaleiro.com/pubs/latch-free-cidr2017.pdf)

Main contributions here are the epoch-based OCC protocol.

- Global Epoch number $E$ in shared memory, updated every ~40ms, i.e. 25x per second.
    - Incremented by a dedicated thread, no locking.
- Each worker $w$ maintains a local epoch number $e_w$ s.t. $e_w = E$ or $e_w = E-1$.
    - Epoch-advancing thread must wait on lagging workers
    - Concern for mixed workloads!
- Unusual TIDs: 64 bits containing...

    | E | X | status |
    |---|---|---|

    - **E** is the $E$ at commit time
    - **X** is a unique ID per **E**
    - 3 **status** bits:
        - **lock**: an exclusive lock
        - **latest-version**: is this the latest version of this tuple
        - **absent**: a "tombstone" that this tuple is deleted

    - The tricky one is **X**. Paper says:
        > A worker chooses a transaction’s TID only after verifying that the transaction can commit. At that point, it calculates the smallest number that is (a) larger than the TID of any record read or written by the transaction, (b) larger than the worker’s most re- cently chosen TID, and (c) in the current global epoch. The result is written into each record modified by the transaction.
        
        I am not certain how **X** collisions are avoided!
    - Note: *TID order may not be the serial order, unlike classic MVCC*!


        | T1 | T2 |
        | --- | --- |
        | W(x) | |
        | | R(X) |

        T2 > T1 because (a) T2 reads T1 hence chooses a larger TID.

        But consider anti-dependency:
        | T1 | T2 |
        | --- | --- |
        | R(x) | |
        | | W(X) |

        T1 could be before *or* after T2 in Silo. Guarantee:
            - Each worker assigns monotonically increasing **X** per epoch, match serial order
            - TIDs per record monotonically increasing, match serial order
            - TIDs with different epochs match serial order
            - Hence...?

- Tuple storage
    | TID | Previous | Data... |
    | --- | --- | --- |

    - Data in place if small for cache locality
    - Update in place if possible (doesn't grow field size)


- Commit (OCC Validation) protocol
    - Assumption: no *blind writes* (must read before writing!)
    - Phase 1
        - lock all tuples in *write-set* for transaction
            - Deadlock avoidance: global order on tuples (e.g. by VM address). Why OK?
        - compiler_fence()
        - snapshot $E$ to $e_w$.
        - compiler_fence()
    - Phase 2
        - *Re-fetch and compare* each record in *read-set*
            - Abort if TID changed, latest bit = 0, or lock bit = 1
        - Assign a TID with **E** = $e_w$
    - Phase 3
        - Write modified records to index with updated TID and set lock = 0, record-by-record
            - Note that TID and lock are sharing a memory word so change together atomically
    
    - note on compiler_fence: this is "global coordination", but on x86 and other "Total Store Order" machines, it's "free" ... just that the compiler mustn't reorder code across the fence! On non-TSO architectures this may indeed be a bottleneck.

- Serializability proof: by reduction to Strict 2PL (S2PL)
    - Recall strict 2PL acquires read/write locks in advance of access, drops all atomically at commit
    - Assume no blind writes, so *write-set* $\subseteq$ *read-set*
    - *read-set* tuple: in Phase 2 we verify it's both unchanged & unlocked. Hence we would have successfully acquired read lock in S2PL in advance of access, held it til commit point.
    - *write-set* tuple: would have successfully acquired/held S lock in S2PL, *and* upgraded to write lock successfully at commit time. Recall: S lock check in Phase 2, but also X lock acquisition in Phase 1!

- Note: Epoch order conforms to serial order
    - compiler-fence before fetching $E$ ensures our reads are before any future epoch happens
    - compiler-fence after write ensure transactions in future epoch would observe "at least" the lock bits from Phase 1 (or our committed records from this epoch)

- Why do updates work?
    - Concern: update-in-place, concurrent readers might have cached old versions
    - But updates in Phase 3, so
        - set lock bit
        - memory fence (before/after for all memory updates)
        - stores updated TID/lock
    - Hence concurrent reader sees either a lock=1, or new data/TID
        - if lock=1, spin until lock = 0
        - there's your concurrency control, and it's brief (?? length of commit)

- Logging/Durability
    - record-level physical REDO logging. No UNDO, no logical (operational) logging.
    - Commit entire epochs to disk. Atomic epochs, *no prefix of epoch*!
        - because serial order within epoch is not recoverable from the log!
        - maybe this is why it's OK for 2 workers to choose the same **X**?
    - Log record for a transaction: TID, list of [(table, key, value)] for modified records
    - Per-worker log queue, with global variable $ctid_w$ of high-watermark per worker
    - Logger thread $l$:
        - calculate $t = \min ctid_w$
        - *local durable epoch* $d_l = epoch(t) - 1$
        - append all buffers to end of log file, plus a final record containing $d_l$
        - wait for write to ack
    - Global logger thread:
        - compute and publish *global durable epoch* $D = min d_l$
        - transactions with epoch $< D$ can respond to clients!

See paper for details of:
- Handling index structure modifications. Recall that ARIES uses "nested top actions" for this, but all that's done here is to keep a *node-set* akin to the *read-set*
    - some fussy details about concurrently updating nodes in the tree
- Deletion/GC
- Versions, tuples that don't fit in cache, snapshots
- Evaluation
    - I confess I ran out of time
    - More importantly, recall that SILO did well in the "Tale of 1000 cores" paper by a third party!
