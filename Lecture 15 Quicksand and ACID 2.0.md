# Building on Quicksand
## Context: NoSQL
Following on from the CAP theorem, "NoSQL" became a popular battle cry in the early 2000s.

Most NoSQL systems were most importantly "NoACID" systems, but the perception at the time was that SQL and Transactions were tightly coupled, and all that database stuff (complex queries, optimizers, transactions) was slow and unnecessary for many developers. For whatever reason NoSQL is the moniker that stuck for such systems.

The NoSQL offerings partitioned into two camps: *document databases* and *key-value stores*.

### Document Databases
The most influential document database was and is MongoDB. Document databases wer so-called because they targeted web-oriented documents (XML, JSON) as their native data model.
(Even today, the [MongoDB website](https://www.mongodb.com/document-databases) unhelpfully explains "A document is a record in a document database.")

Early on, MongoDB made a lot of claims about scalability, in the spirit of CAP, which [caused some amusement](https://www.youtube.com/watch?v=b2F-DItXtZs). But the real advantage of MongoDB was that it had great ergonomics for JS programmers -- setting up MongoDB and getting JSON in and out of it was very easy. With time it acquired many of the features we associate with a classical DBMS, retaining its JSON-centric data model and a bespoke query language. Meanwhile extensions to PostgreSQL and other systems to support JSON were largely inspired by the success of MongoDB.

### Key-Value Stores
The other class of NoSQL systems were "Key-Value Stores (KVSs). These are essentially durable hashtable APIs. One of the earliest standalone KVSs was BerkeleyDB, which was [Margo Seltzer's](https://www.seltzer.com/margo/) PhD work from the Postgres era, and was for a long time one of the most successful open source databases. It provided B-tree and Hashtable indexes, and was packaged as an embedded library that could be linked into applications. Unlike many subsequent KVSs, BerkeleyDB was a fully transactional system, targeted at a single node.

One of the most influential scalable KVSs was Amazon Dynamo, which they used to replace what was purportedly the world's largest Oracle installation for a bunch of their shopping state. Unlike BerkeleyDB, Dynamo was a distributed, eventually-consistent KVS. [Apache Cassandra](https://cassandra.apache.org) is an open source implementation of the Dynamo design. (Note that AWS DynamoDB is not the same system as Amazon Dynamo). The [Dynamo paper](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) is one of the most-cited examples of an eventually consistent system. It was published at SOSP rather than a database conference. Dynamo itself wasn't especially novel, but the paper is interesting for a few reasons:

1. It is a good example of a large web property replacing a general purpose Relational DBMS with a bespoke database for their own purposes. In a sense Dynamo is to OLTP what MapReduce was to Data Warehousing/OLAP. 
2. It has interesting examples of *application logic that works over an eventually consistent database*.

> The theme of *bespoke vs general DBMS* is unlikely to ever go away. It should not surprise/impress you on its own. The real question to ask: what can you learn from the optimizations chosen for the bespoke setting? Many of these relate to the workload and application logic! Are these repeatable patterns? When and where?

# Building on Quicksand
Rather than read the Dynamo paper, I chose for us to read a paper that focuses on the juicy lessons from Dynamo. This paper is less buttoned down than the Dynamo paper, but it has both more lessons and more open problem statements.

This paper is a fine example of Pat Helland's work, characteristically grounded in real-world experience and exposing "design patterns" and other kitchen wisdom on scalable systems that transcend the simple transaction model we learn in school.

A tax of Pat's writing is that it's the reader's job to distill the wisdom into more exact science or engineering. View this as an opportunity!

>tl;dr: When you leave the walled garden of serializable transactions, anomalies can arise. There are two key contributions in this paper to address this:
> 1. **Memories, Guesses and Apologies**. This is a general-purpose design pattern for *compensating for broken invariants* due to loosely-consistent systems.
> 2. **ACID 2.0**: This is a special-purpose design pattern for *guaranteeing invariants* in special cases: use only Associative, Commutative and Idempotent actions in a Distributed system.

There is also a host of history and perspective in the paper, which is summarized below.

## Distributed Transactions are to be Avoided
*Distributed transactions* using two-phase commit require unanimous votes, i.e. no downtime for any participants.  This is not very available, and at best suffers from tail latency (the slowest not determines your latency).

Single-site transactions are nice building blocks, but you have to string them together somehow into correct application experiences. How??

## Memories, Guesses and Apologies
### Some Lessons From Early Asynchronous DB Environments
1. If you allow uncoordinated asynchrony, you need to tolerate out-of-order "replay" of history --- e.g. when recovering transactions after primary recovers during log shipping.
2. Out-of-order recovery affects the invariants of application logic invoking the transactions -- so-called "business rules" -- and we can no longer guarantee that those invariants are enforced. Invariants become "probabilistic" in the sense that they remain true with probability < 1.0.

> It sure would be nice if "business rules" were *commutative*! In general, they are not.

### Memories, Guesses and Apologies
If business rules fail with some probability, you should assess your (economic) tolerance for that failure.

Example: optimistically promising a Harry Potter book will be available in inventory at Amazon, vs. transactionally guaranteeing a high-value item is available via Sotheby's auction.

Upon failure, what do you do? Ideally you'd like to:
1. run an inverse function for the failed actions
2. run an inverse function for any dependent actions, and 
3. rerun the dependent actions without the dependency
This is the idea behind [SAGAs](https://dl.acm.org/doi/pdf/10.1145/38714.38742), which are enjoying renewed popularity as a design pattern in microservice land.

Realistically, though, you rarely have fully invertible actions. Instead, you *compensate*, often via something that not a direct computational inverse, like sending a coupon.

> You can trade availability---and hence latency---for consistency, on a sliding scale! You "just" need to understand the real-world (application) value of inconsistency and how to compensate for it.

> *"Arguably, all computing falls into three categories: memories, guesses and apologies."*

- **Memories**: Stored, local knowledge that you can choose to disseminate at some cost in latency and bandwidth.
- **Guesses**: Decisions made based on local knowledge.
- **Apologies**: Compensatory actions, whether computational or extra-computational.

> **Open Question, Software:** To my knowledge, nobody has implemented a good software framework for this design pattern. There should be a "Ruby on Rails" for this! What would it look like? 
> 
> **Open Question, Theory:** How does the memories/guesses/apologies pattern relate to CALM, $Datalog^\neg$ and the semantics of negation? We're abandoning stratification here, but what are we putting in its place? Is there a role for fancier negation semantics like the [well-founded](https://en.wikipedia.org/wiki/Well-founded_semantics) semantics? Could such a thing capture practical use cases?

## ACID 2.0
### The Dynamo Shopping Cart: An EC Application Pattern
Dynamo (Cassandra) is a little bit interesting. The Dynamo Shopping Cart is much more interesting: it works correctly despite Dynamo's loose consistency!

Dynamo is a replicated KVS, which asynchronously "gossips" updates among replicas. It promises very little regarding the order of PUT and GET requests. PUTs may be concurrent. GETs may see stale data depending on which replica they contact; they may also see multiple alternate versions. Note that the staleness may propagate: Stale data can cause clients to create and PUT "new" data computed based on that stale data. 

To work with this at application level, the shopping cart works like a *ledger*: it always appends uniquified ADD-TO-CART, DELETE-FROM-CART and CHANGE-NUMBER requests. These are commutative! 

Dynamo may in fact return multiple versions of a key. The Shopping Cart logic can simply union up the requests in the two versions to arrive at a consistent subset of the shopping cart requests.

> Shopping Carts are an example of application logic designed to tolerate eventual consistency. READ/WRITE consistency is not the right way to reason about the correctness of the application!
> Another application that tolerates EC: small-dollar banking! Yes, the classic textbook example of transactions is typically not implemented *fully* transactionally, but rather as something more like an EC set of transactions!

> A theme in the writings of Jim Gray, inherited by Pat Helland, is that *classical bookkeeping techniques from accounting are a rich source of wisdom for implementing reliable data-centric systems*. Ledgers are an interesting multi-transaction example.

### ACID 2.0
Attributed to longtime database veteran Shel Finkelstein, the acronym ACID 2.0 captures *properties of application logic* that make Eventually Consistent databases make sense. These are:

- **Associativity**: The "batching" of requests should not affect their outcome.
- **Commutativity**: The ordering of requests should not affect their outcome
- **Idempotence**: The number of repetition of requests should not affect their outcome
and
- **Distributed**: A word that makes the acronym into "ACID".

Together, ACI allow different replicas to get requests in different orders, with different numbers of retries, and achieve Eventually Consistent state. This is great for tolerating failures and latency.

> If your application can be written in an ACID 2.0 style, you can employ replication to improve latency and fault tolerance. But beware: READs are "annoying" and not ACID 2.0! More on this next.

## Context, Context, Context
This paper is rich with context from the history of database systems and the enterprise transaction processing environments in which they have been deployed over decades. 

> This context is not required to understand the big ideas in the paper, but it's chock full of interesting stuff. If you're in a rush, you can move on to [CRDTs](#CRDTs)

### Idempotent Transactions are good building blocks
Back in the bad days of the early web you'd often see dialog boxes that would say things like "Do Not Click Reload or Close This Window Until Complete!" This would usually crop up after doing something important like buying 100 shares of Pets.com. If you didn't follow instructions, upon reloading the page there was no way of knowing if your transaction went through. And there was some decent chance that it had been resubmitted by reloading the page, and you'd end up buying 200 shares of Pets.com

This is an example of a setting where you want *Idempotent* composition of requests: i.e. $request \cdot request = request$.

The typical scheme for implementing idemotence is to have each request be assigned a unique ID (or "nonce"), and the server to record the nonces it has processed in a lookup table, to ensure that it processes each request once. This can take up quite a bit of space if it's done willy-nilly, but it's not a big overhead on a typical durable transaction.

Idempotence and ACID transctions are a nice mix: the A&D of transactions ensures all-or nothing "attempts", and the Idempotence ensures *at most one* successful attempt.

### The Evolution of Fault Tolerance in Databases
#### The Grandparent of FT: Tandem Computers
Tandem was a super interesting HW/SW co-design in the 1980s. Jim Gray went there after IBM, and was responsible for a lot of their NonStop SQL product.

Tandem was an aggressive exercise in Fault Tolerance. Hardware components were replicated so that they could tolerate a single failure, as were the networks between the components. The OS, Guardian, was build on message passing without shared memory -- easier to scale, and isolate corruption. Processes had "slaves" that would periodically snapshot state and take over on failure. 

NonStop SQL was an early parallel OLTP database design. It didn't do much intra-query parallelism a la Gamma, but it did support scalable transaction processing.

One of Jim Gray's lessons from Tandem was about Heisenbugs and Bohr bugs:
> A Heisenbug is one that does not show up when you try to examine it -- e.g. when you run a debugger or add tracing. It is usually the result of some non-deterministic phenomenon that is perturbed by measurement. It is very difficult to avoid or fix Heisenbugs. A Bohr bug is one that is easily reproduced. We fix Bohr bugs by observing them and fixing them. 

One way to avoid Heisenbugs is to have two different teams implement the same spec separately, without sharing libraries, etc.

A folklore story I heard about Tandem was that they got a support call after an earthquake that "the machine was down" and they wanted help to get it back up. The support people asked the customer if they knew how to reboot the machine; the customer said they did, but it didn't need rebooting! Ut was still running, it had just fallen over, and they wanted to know how to set it upright again. (I found [a copy of this story online](https://availabilitydigest.com/public_articles/0707/tandem_computers_unplugged.pdf)!)

Pat Helland got his start working for Jim Gray at Tandem as a young high school graduate, which perhaps explains the background in this paper.

##### Asynchronous Writes in Transactions: Group Commit
1. Tandem 1984: Checkpoint/Flush on every WRITE. Safe but slow.
2. 1984: DeWitt on sabbatical at Berkeley, writes influential [main-memory database paper](https://dl.acm.org/doi/pdf/10.1145/602259.602261) including Hybrid Hash Join and Group Commit (which is not well-referenced in the Quicksand paper). 
    - Basic idea of Group Commit is to buffer log records from committed transaction into batches to amortize high-latency log writes. Once a commit is buffered, 2-phase locks are released, but the client is not notified until the commit record is flushed. Because it's not strict 2PL any more, cascading aborts are possible.
3. Tandem 1986: Includes version of Group Commit: buffer writes in primary disk process. If that process fails, abort all related transactions.

Sidenote on Group Commit: this is a rare Berkeley-Wisconsin collaboration from DeWitt's sabbatical at Berkeley, which apparently went badly. That said, Stonebraker and DeWitt remained close, personally.

> Note how the ability to abort a transaction allows for timing slack -- i.e. asyncrony. The downside of course is the overhead of transactions, which are not always a good option.

#### Further Asynchrony: Log Shipping
An important trick in database replication: don't replicate the database, replicate (and replay) the log. 
This is very efficient because replication is incremental, rather than full-snapshot. 

Sybase Replication Server innovated in this domain.
ETL tools (Informatica, GoldenGate) would reverse-engineer the log formats of DB vendors to offer 3rd-party replication. This is now often called *Change Data Capture (CDC)*, specifically Log-Based CDC.

DB replication typically is a bit "stale". If the primary copy fails, you lose a bit of history.
The nice thing about log-based CDC is that it's a transactional snapshot---you always see a prefix of the committed transactions.

It is possible, when you recover the old primary, that is has some committed transactions "buffered" inside it. It's entirely unclear how one is supposed to deal with these transactions upon recovery! Oh well?

> Here we see an early admission in the DB industry that it's OK to trade consistency for asynchronous resilience!


#### Sidenote: Escrow Transactions
Suppose we have a workload of increment/decrement tasks, as in managing a bank account or selling tickets (e.g. to an airplane flight). There's typically a *boundary invariant*: account balances must be positive, plane cannot be overbooked.

The increment/decrement commands are *commutative unless they break the boundary invariant*. 

This commutativity can be exploited, with care:
1. Batches of increment/decrement can be declared safe (will not violate boundary invariants *in toto*). Within such a batch, the operations are fully commutative.
2. Note that *READ does not commute with increment/decrement*! This is a bit surprising, since READ is often thought to be less "tricky" than write. As the Quicksand paper notes:
3. Of course WRITE also does not commute with anything. 
> - Storage (READ/WRITE) is an "annoying" abstraction: it is non-commutative.
> - More generally, READ is "annoying".

##### Encouraging EC: Fungibility
Different copies of a Harry Potter book are indistinguishable, they form a *fungible* commodity. The single item at Sotheby's is *non-fungible*. 

Requests for fungible commodities are amenable to Escrow-like approaches.

