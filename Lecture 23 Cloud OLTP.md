# Lecture 23: Cloud OLTP

### Let's start with the hidden assumptions in Aurora
- Single-mastered databases! *What? Is that OK?*
- Stale RO copies! *What? Is that OK?*


### Recall our Cloud Change list from OLAP. What's different for OLTP?
1. Delegate components to good-enough subsystems, and their dev *and ops* teams
   1. Eg storage in S3? **No!** (why not?)
   2. Eg cluster management in k8s?
2. Elasticity is available: what shall we use it for and how? **Impact of single-master**
3. Need to address global reach and geolatencies.
4. Lots of shared work across queries/users
5. Diversity of HW
6. SLOs. E.g. "best effort" vs "reserved"
7. Security concerns: Enclaves?
8. Learn from workloads
9. IaaS pricing models and impact on design

## Many Nice Lessons from the Aurora Paper
Required reading! In fact, so well-written and full of sweet sense that it almost "hides the ball" of its Achilles' Heel(s)!

- Quorum math and AZ-level reasoning
    - Goal: tolerate AZ+1 failures
    - 6 copies: 3 AZs, 2 copies per AZ!
        - write quorum = 4
        - read quorum = 3
- MTTF/MTTR analysis
    - MTTF can only be driven down so far. Focus on MTTR!
    - storage in 10GB *segments*
        - This is the unit of failure/repair
        - 10Gbps network: **10 sec MTTR**
    - *Protection Group (PG)* is 6 segments, 2 per AZ.
    - *Volume* is a set of PGs
    - all stored on SSDs in EC2!
    - Given 10 sec MTTR, need AZ failure and 2 more segment failures within 10 sec. 
    - Q: why not even smaller segments? What's the bottleneck?
- A host of concerns addressed in one mechanism: replication & fast recovery!
    - *hot node?* mark it unavailable, fail over to others
    - *SW patching?* mark it unavailable, fail over to others
        - Do this 1 AZ at a time!
- The log is the database.
    - Postgres storage inverted!
    - The single-node DBMS is a very fancy cache of the log
        - And it (MySQL, PostgreSQL) was already written for us, so no sweat!
    - The only writes that escape the local DBMS are Redo log records!
    - Continuous per-page recovery in the smart storage tier (not S3!)
        - No checkpoints
        - Worst recovery is an individual page with lots of modifications (not the entire log all together)
        - Work of recovery maintainance is paralle, background effort
        - periodically stage log and new pages to S3 (a bit vague but seems for catastrophic or point-in-time recovery)
- Respectable related work discussion for an industry paper

**BUT, BUT, BUT...** it's a single-master non-serializable design!  Sigh.

- There was a [multi-master version of Aurora](https://aws.amazon.com/about-aws/whats-new/2020/05/amazon-aurora-multi-master-expands-availability-8-aws-regions/#:~:text=Amazon%20Aurora%20Multi%2DMaster%20is,write%20availability%20through%20instance%20failure.) but [it seems to have been pulled](https://www.reddit.com/r/aws/comments/11vsvow/what_happened_to_aurora_mysql_multimaster_option/) as it was not general-purpose ([only MySQL 5.6](https://stackoverflow.com/questions/77175541/where-can-i-find-the-option-to-use-multi-master-in-database-mysql-aurora-creat), with limitations) and not efficient -- write conflicts are resolved by rollback. It's also not serializable, just read-your-writes.  You can find [a blog post about how it works here](https://www.scaler.com/topics/aws/aurora-multi-master/).

## PolarDB-SCC: Serializable Reads for Aurora-Style Replication using RDMA
- Stale reads suck
    - bad consistency semantics, obviously
    - if you want strong consistency, you overload the RW node even for RO workload
        - defeats the purpose of RO autoscaling
        - RO nodes end up underused
    - No magic bullet
        - Spanner/Cockroach recommends turning off strong consistency
            - they do offer bounded staleness
- Still single-master
- Simple Solutions:
    1. commit-wait: RW node waits for replicas before commit
       - makes RW transactions high latency
    2. read-wait: RO node fetches RW node timestamp, waits for logs to accrue sequentially up to there
       - load on RW node even for RO transactions
       - may be waiting for irrelevant data
- Insight: read-wait can be made better

1. do not wait for irrelevant data, apply entire logs sequentially. instead, track max LSN on fine-grained subsets
    - Hierarchical: global, per-table, per-page max timestamps (LSNs)
    - check from top-down and if success at any level process request
        - else wait for relevant pages to arrive
2. Use "Linear Lamport" timestamps to avoid fetching RW node timestamp
    - RW node is the "timestamp oracle"
    - Can reuse a timestamp fetch for transactions that "happened-before" the fetch request
        - e.g. in the below, 
        ```mermaid
        flowchart LR
        subgraph RO node
            subgraph request 1
            t_1["r_1 arrives"]
            t_4["r_1 needs TS, can reuse TS3rw!"]
            end
            subgraph request 2
            t_2["r_2 fetch TS"]
            t_3["TS response"]
            end
            t_1 --> t_2
        end
        subgraph RW node
            TS3rw
        end
        t_2 --> TS3rw --> t_3
        t_3 --> t_4
        
        ```
    - special case: request arrives before r2's fetch, but needs a TS between the request and response of r2.
        - make it wait for r2's response!
    - The happens-before tracking is done by caching <$TS_{rw}$, $TS_{ro}$>
        - If a request arrives before $TS_{ro}$, use cache
    - Note: now only one TS fetch per transaction per RO node fails to cache.
3. RDMA for log-shipping/timestamp-fetching 
    - low latency
    - sidesteps the CPU at RW node!
    - Doesn't seem to address Aurora's cross-AZ desires?!
    - Detail: hashtable of hierarchical timestamps needs to be at a fixed location/size in memory
        - one TS per *hash value*; collisions OK, keep max (makes readers more conservative)
    - Remote logging a bit complicated
        - RW node has a fixed-size ring buffer for log, one "log writer" thread per RO node
        - need to rate-match:
            1. the buffer at RW node
            2. the buffers at the RO nodes
        - if the rate-matching fails, RO node can fetch log from shared cloud storage (!!)
4. Read-your-writes across SQL statements: 
    - RW node returns an LSN for each write to the proxy
    - Proxy makes sure to route reads to a node that's past that LSN (RO or in worst case RW node)
    

    
# What about Spanner/Cockroach?
Yes, you can implement/use 2PC! The existence of Spanner is a nice proof point, but it's debatable whether it's a nice system.

- In the worst cases (geo-distributed transactions), 2PC makes you wait the speed of light for 1-2 round-trips. In 2013, Peter Bailis worked this out at 106ms Sao Paolo to Singapore, but real ping times were more like 362ms on average, 649ms on 95th %ile. So geo-distributed transactions take something like 1/10 to 1/2 second to commit. That's *not great* -- you would not put that in the inner loop of a system -- and it's *not going to get better*.
- So the name of the game is either
    - avoid geo-distributed updates! (if you can, then Spanner becomes attractive, but maybe so does single-master??)
    - cheat on serializability
        - e.g. snapshot isolation with "bounded staleness" a la TrueTime
- Oh by the way, 2PC is not fault-tolerant
    - Coordinator is a single point of failure.
    - Various attempts to fix that were wrong (3PC, 4PC, ...)
    - You need something like [Paxos Commit](https://lamport.azurewebsites.net/video/consensus-on-transaction-commit.pdf), the great Lamport-meets-Gray sundae.
        - i.e. complicated!
    - The Spanner paper is both complicated and poorly justified -- a hodgepodge of 2PC, Paxos and TrueTime.
        - There is probably work to do in exploring elegant geo-distributed transactions.
        - I haven't taken the time to learn about the variations in CockroachDB, they may have done some nice stuff there. Note: the founder is Berkeley alum Spencer Kimball, author of [Gimp](http://www.gimp.org).

- Here's a [nice read from Eric Brewer](https://research.google/pubs/spanner-truetime-and-the-cap-theorem/) on Spanner, CAP, and how to think about TrueTime (hint: it's more helpful for timestamping snapshots, as well as other low-level mechanisms).
