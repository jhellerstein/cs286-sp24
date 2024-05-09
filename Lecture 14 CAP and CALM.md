# CAP Theorem

## Context: NOW, Inktomi, BASE
Back in the late 1990s, the precursor to the Sky lab was called the 
[NOW Project](http://now.cs.berkeley.edu/), for Network of Workstations. 
It was led by faculty members Dave Patterson, David Culler and Tom Anderson. Some of the famous students include Andrea and Remzi Arpaci-Dusseau (Wisconsin), Mike Dahlin (Texas/Google), Kim Keeton (HP/Google), Steve Lumetta (Illinois), Amin Vahdat (Duke/UCSD/Google).
The saw that shared-nothing clusters would be a commodity, and wanted to explore what software would look like. Note that this is post-Gamma, but pre-Google, pre-cloud.

With industry sponsors they filled the lab next to the Woz with computers -- a sizable cluster for its time.

Eric Brewer was also on the faculty, and with his student Paul Gauthier they started building a web search engine on the cluster. At the time, the state of the art in web search was AltaVista, which was invented at DEC and ran on a single very expensive DEC multiprocessor (Wikipedia claims they used 20 of those machines, but I suspect they were full replicas of the entire index.) Inktomi used Gamma-like partitioning to shard the inverted index that was the heart of search at the time.

Brewer and Gauthier founded a company to commercialize Inktomi, and as of the late 1990s it was the presumptive winner of the search engine wars, having displaced AltaVista.  

According to the story, Inktomi Inc tried to use the Informix database for search. Informix was a Gamma-style architecture and considered one of the best parallel DBMSs of its time. It could not handle the scale of the web search workload, and that was blamed in large part on the overhead of transactional concurrency and recovery.

This got Brewer and students like Armando Fox and Steve Gribble thinking about alternatives to transactions, and they started talking about ACID vs BASE semantics: "Basically Available Soft-State Eventually consistent". One of the key lessons there:

> Transactions do not promise Availability

### Major Shifts in Computing: Query Semantics Pre- and Post-Web 
Prior to the web, the unavailability of a transaction system was considered a good thing: far better to be right than to be running. Once online consumer applications became common on the web, the opposite was true: better to be running than right.

It's worth reflecting back on how the rise of web search is entwined in these expectations. Prior to the web, text search (i.e. the field of Information Retrieval) was not of significant academic interest (mostly just Gerald Salton at Cornell, though the roots go to Hans P. Luhn at IBM in the 1950s!). Document search was a small dollar business. 

One of the knocks on search was that it was wildly heuristic: answers to queries were not well defined, there was a soft HCI component of *satisficing* decisions: decisions that satisfy some threshold are sufficiently good, even if they may sacrifice optimality. Much of respectable computer science --- and certainly the database community with its roots in formal logic and top-down design --- looked askance at search.

All of that changed with the rise of the consumer Web. Web search was honestly pretty poor in the early days (and the web was mostly full of useless information), but it shifted public perception -- and academic perception! -- from a view of computers as calculators of correct results to a view of computers as powerful heuristic assistants of human behavior. 

Once you are delivering query results that are satisficing even on perfect data, then clearly you can relax your need for transactional consistency. And given an open system in a commercial context, simple economics drives you to trade in correctness for availability.

Brewer used to tell the story about how Inktomi moved their cluster from Berkeley to Foster City. They shut down half the cluster, drove it Foster City and turned it back on, flipped the switch in DNS, then brought the rest of the cluster to Foster City and turned it on. Approximately half the web was missing from search results for a while and approximately nobody complained!

In many ways this shift was the beginning of a new chapter in computer science, opening the door much wider for both AI and HCI. (Note that Herbert Simon who coined the term Satisficing was a political/social scientist!)


(Historical note on the web search wars: For a time, Inktomi was one of the few superscalar web companies, alongside pioneers like Yahoo and eBay. They expanded from web search into web caching and content distillation (i.e. downscaling images for low-bandwidth modems). But Inktomi was not in the public eye so much: they ran an "arms dealer" business, licensing their tech to other properties. Yahoo was the main web "portal" in those days, and licensed Inktomi for search for a time. Google was founded after Inktomi and copied its sharded design, with PageRank as their initial core improvement. Inktomi lost out to Google fairly quickly. Partly this was timing: the dot-com bubble bursting was harder on a larger company like Inktomi. Also Google chose not to be an arms dealer (though Yahoo licensed it for a time!) -- they figured out how to monetize a branded search website via advertising, which turns out to be a far bigger business than licensing a search engine!  Inktomi was eventually sold in two pieces -- enterprise search to Verity (the leader in that space at the time), and the core web search to Yahoo.)

## CAP Conjecture
Given this experience, Brewer formulated the CAP Principle, which he first discussed in talks, and then in a paper with Armando Fox, and then in a PODC keynote:
> CAP: Strong **C**onsistency, High **A**vailability, **P**artition-resilience: Pick at most 2.

Note: much like ACID, the **P** is kind of redundant -- there's no reason to talk about Availability if you don't expect machines to get disconnected. 
As with ACID, everybody likes to treat the acronym as an axiom system, but that's not a good use of your time.

Given the impact of Brewer's work via Inktomi and academia, as well as the emergence of consumer web infrastructure as a grand challenge in industry, this idea lit up the world of distributed systems.

## What is Consistency Anyway?
Fox and Brewer defined the **C** in CAP as follows:
> strong consistency means single-copy ACID consistency

Here "ACID" is not being unpacked, it's a placeholder for "serializable transactions". Indeed the **C** in ACID (enforcement of database constraints) has no clear definition in web search. But we know that serializability is a strong guarantee.

After Brewer's PODC keynote, 
distributed systems hero Nancy Lynch and her student Seth Gilbert decided to try and formalize the CAP Principle into a Theorem, and they chose to define Consistency as *Linearizability*, proving that Linearizability is not possible under partition. Basically the two sides of a partition cannot agree on the order of events when they reconnect.

The confusion and discussion around the CAP acronym, the Theorem, and Brewer's practical intent significantly amplified interest in the idea. A recurring lesson in the diffusion of ideas: sometimes flaws in a good idea can get that idea discussed/cited even more often!

Before we go on, let's have some new definitions that come from Herlihy and Wing's paper on Linearizability. It's from 1990 -- rather a latecomer relative to serializability from the early 1970's! Very influential in distributed systems and programming languages, where asynchrony and replication are common but transactions are not necessarily used.
 
### Definitions
#### Strict Serializability
Recall that a schedule (H&W use the term "history") is serializable if it has an equivalent serial schedule. That equivalent schedule could have nothing to do with the wall-clock time when transactions started or ended! That's not so good for systems that value fair queueing and/or real time -- think TicketMaster or stock purchases!

Instead assume transactions have some precedence order. Maybe it's a total order, like the wall-clock time of commit, for example.
> A transaction history is *strictly serializable* if the transactions' order in a sequential history is compatible with their precedence order.
Note that 2PL gives strict serializability in exactly one way ... wrt the order of conflicts. MVCC is very flexible -- the timestamper can choose any order it likes, which can choose to be strictly serializable with respect to some ordering your care about, or non-strict.

#### Linearizability
Humbly, H&W describe it as "a special case of strict serializability where transactions are restricted to consist of a single operation applied to a single object". 

An attractive feature of linearizability is that it is a "local" property: if you're linearizable on each object individually, you're linearizable on the set of all objects. This is clearly not the case for serializability.

H&W draw an opinionated distinction. They say that serializability is useful for app developers: a strong abstraction protecting an application's full behavior. By contrast, linearizability is useful for system builders, as it's a more flexible, expressive abstraction that helps reason about subtler control of individual actions.

Like serializability, linearizability assumes that agents are sequential processes that issue one request at a time.
A key distinction between linearizability and serializability is that linearizability assumes "split-phase" actions -- requests and responses are decoupled in time. Requests from two different processes in one order may return responses interleaved in the opposite order.

For linearizability, we want the illusion that each request/response pair happens in sequence, without such interleaving.

> A History is a finite sequence of operation *invocation* and *response events*. 
> - Invocation: $(A: x.op(args\*))$, where $x$ is an object name, $op$ is an operation name, $args\*$ denotes a sequence of argument values, and $A$ is a process name. 
> - Response: $(A: x\ term(res\*) A)$, where term is a termination condition (OK/Err), and $res\*$ is a sequence of results. 
> - A response *matches* an invocation if they share the same object and process names. (Assume sequential)

Much like we have Transaction identifiers (T1, T2, ...) in Serializability, we have Process identifiers (P1, P2, ...) in Linearizability.

Much like serial schedules, we want a "benchmark of correctness" for linearizability:

> A history is *sequential* if:
> 1. The first event is an invocation
> 2. Each invocation, except possibly the last, is immediately followed by a
> matching response. Each response is immediately followed by a matching invocation.

A history that is not sequential is concurrent.

> Two histories are *equivalent* if for they have the same subsequence for every process P.

> In history $H$, operations $o_1$ and $o_2$ are ordered ($o_1 <_H o_2$) if the response of $o_1$ precedes the invocation of $o_2$ in $H$.

> For history $H$, *complete(H)* is the maximal subsequence of H consisting only of invocations and matching responses.

i.e. *complete(H)* drops the outstanding invocations.

> A history is *linearizable* if it can be extended (by appending zero or more response events) to some history *H’* such that:
> - Ll: $complete(H’)$ is equivalent to some legal sequential history $S$, and 
> - L2: $<_H \subseteq <_S$.
>
> $S$ is called a *linearization* of $H$.

(The reason for $H'$ is just to gather responses for outstanding invocations in $H$).

Example of a non-linearizable history of enQ and deQ ops on queue $q$ from processes *A* and *B:
```
History H2:
A: q.enQ(x)
A: OK()
B: q.enQ(y)
A: q.deQ
B: OK()
A: OK(y)
```
Let's try moving that around to sequential subsequences (L1) that preserve the order *A <_H B* (L2)
```
A: q.enQ(x)
A: OK()
A: q.deQ
A: OK(y) <-- NOPE!
B: q.enQ(y)
B: OK()
```

#### Sidenote: Sequential Consistency
Leslie Lamport, 1979.
> A history is *sequentially consistent* if it is equivalent to a legal sequential history. 

This is weaker than linearizability, in the way that serializability is weaker than strict serializability: we don't require condition L2 above, just L1!

Consider this:
```
History H7:
A: q.enQ(x)
A: OK()
B: q.enQ(y)
B: OK()
B: q.deQ()
B: OK(y)  <-- NOPE!
```
Clearly not linearizable. But it *is* sequentially consistent, just reorder *B* before *A*!

## CAP Revisited Paper
For about a decade, people kept writing blog posts about "defeating CAP". One way or another, they are able to stay available under partition and "heal" the system afterwards. How could this be?

In fact, this may have been what Brewer was after the whole time -- go back to the BASE discussion.

There's a great quote in the IEEE paper:
> *The subtle beauty of a consistent system is that the invariants tend to hold even when the designer does not know what they are.*

In short, CAP talking about *conservative* techniques like transactions and linearizability, where the application invariants are unknown and we must reason about the order of operations.

Much ink was spilled about application-aware techniques that could work. Brewer highlights some of the ideas:

- Order-insensitive merging when healing.
  - Turn the happens-before DAG into an arbitrary order
  - Arbitrary is fine if operations are commutative (non-conflicting)!
  - Otherwise ... appeal to a human? Or...
- Compensation ("Memories, Guesses and Apologies")
  - Keep track of any "mistakes": concurrency that violated invariants. E.g. overbooking the plane.
  - Define compensations ("apologies")
      - Computational: e.g. inverse transaction and restart, a la [SAGA](https://dl.acm.org/doi/pdf/10.1145/38714.38742)s
      - Extra-computational: e.g. an airport food voucher.

We'll read about some of this soon, specifically CRDTs.

# CALM Theorem
Similar trajectory: a conjecture in a PODS keynote, followed by a theorem (from Ameloot, Neven and van den Bussche).

## Background
In the late 90's and early 2000's, Eric Brewer and I revamped CS262 (formerly Ousterhout's OS course) to be an intro course spanning Databases and Operating Systems. We agreed that most of the hard problems in systems were about state/data management, but that the DB and OS communities had different philosophies and techniques that every computer scientist should see. 

As it turned out, seeing them side-by-side is especially useful. The UNIX and System R papers are a great kickoff for that, offering distinct "bottom-up" and "top-down" design philosophies, both of which have been wildly successful. (Both papers also put data management at the heart of their responsibilities!)

At the same time Eric was thinking about ACID and BASE, I was thinking about Online Aggregation (which we'll study later), which is a statistical way of trading correctness for efficiency in query processing. Online aggregation was also inspired by the web in a way---specifically by incremental delivery of interlaced images over slow modems. In essence we were both exploring satisficing approaches to data systems. 

Of course the ACID/BASE and CAP discussion was unavoidable in the 20-oughts, as NoSQL rose and subsided in that era. 

At the same time I was working on Declarative Networking and queries with soft state, and seeing that certain problems seemed tolerant of soft state (e.g. routing) and others did not (e.g. the Symphony DHT).

In 2010, as Dedalus was getting formalized, I was invited to keynote PODS. My plan was to give a talk called [The Declarative Imperative](https://speakerdeck.com/jhellerstein/the-declarative-imperative), to try to re-energize the PODS community around its roots in Datalog by explaining the practical things we were doing. I was prompted in part by a blog post from Hennesey and Patterson that bemoaned the end of Moore's Law and the rise of parallelism as a frightening future for software. In my view that's because they didn't understand databases, and their view of the challenges of parallelism were based on sequential languages and the failure of "auto-parallelization of C programs" efforts in 1990's compiler community. To quote Henessey at the time:

> *I would be panicked if I were in industry.*

My lesson was to chill out and embrace declarativity (and, pandering a bit to the PODS audience, Datalog). After all, I was on a mission to implement everything with queries -- even maybe implement a whole database engine with queries, from the transaction layer up! (I'm still on that mission!)

As I started writing the talk, the CALM Conjecture among others, came to mind.

### Monotonicity: Intuition
By this time we had a formal semantics for state management---the *persistence rules* of Dedalus, which came in two forms. In the Dedalus rules below, relation `p` is persistent: once a fact is added to `p`, it remains over time by induction. Relation `q` is also persistent, but it allows deletion: by placing a fact in `del_q`, you break the induction and cause it to not be in `q` in the next timestep:
```prolog
p(X)@next :- p(X).
q(X)@next :- q(X), ¬del_q(x).
```

Said differently, `p` is a monotonic predicate: its data dependencies are all positive, so every fact in `p` will remain true as we assert new facts. By contrast, `q` is a non-monotonic: facts in `q` can be rendered false by the discovery of new facts (in `del_q`).

As I thought about this, it became clear to me why our routing rules worked in NDlog: they were pure monotonic recursion! By contrast, the Symphony DHT was doing aggregation, which is non-monotonic. Maybe ... non-monotonic tasks like counting are the reason for coordination! Weirdly, though, coordination protocols also count.

Our mantra at the time:
> Counting requires waiting; Waiting requires counting!

And when you think about stratified negation, it's awfully like protocols for Transaction Commmit or Paxos: we stop the world and make sure everyone agrees on something before we continue.

### Revisiting MapReduce and Join
Hey -- this hit another of my bugbears. MapReduce conflated the physical operation of "shuffle" (Graefe's Exchange operator) with the logical operation of Reduce (i.e. aggregation). Reduce requires a barrier synchronization in the BSP model -- which makes sense because aggregation, like negation, makes sense when it's stratified! 

But exchange doesn't need a barrier synchronization, and joins with exchange can be purely pipelining! Look at the symmetric hash join! In MapReduce, everybody had to implement joins as Reduce to get parallelism. Grrr...stupid MapReduce!

This makes sense because joins are monotonic! Add more stuff to the input, get more stuff at the output!

Again -- maybe the whole reason for barriers, transactions, consensus... is to achieve stratification for non-monotonic programs! 

### The CALM Conjecture
This led to the CALM Conjecture, which essentially said  that "A program has an eventually consistent, coordination-free implementation iff it is monotonic". (However, again for the benefit of my friends in the PODS audience, I decided to say "A program has an eventually consistent, coordination-free implementation iff it is expressible in (monotonic) Datalog", which turned out to be overly-specific!)

## The CALM Theorem
Tom Ameloot was a grad student at U. Hasselt in Belgium working with PODS heavyweights Frank Neven and Jan van den Bussche (who I seem to remember came up to me after my keynote, but I did not know them at the time!)

Much like Gilbert and Lynch, the primary challenge for Ameloot, et al. was to choose crisp definitions for the terms in my conjecture. Specifically, he needed a model of computation and definitions of the terms *monotonicity*, *consistency*, and *coordination*. 

### Computational Model
Ameloot defines the notion of "machine" in a distributed system as a *relational transducer*. Think of a transducer as state machine that can consume and produce network messages in an event loop, where each iteration of the loop is a program expressed in some language $L$. Relational transducers use relational languages.

### Monotonicity
Monotonicity is canonical and need no further definition. 

### Eventual Consistency
I had offered an application-level definition of consistency in the PODS keynote: 
> Given: a distributed system and a trace of messages
> It exhibits *eventual consistency* if the final state of the system is independent of message ordering

And I noted this wasn't just data (EDB) replica consistency as in the classical literature, this was consistency of application outcomes (IDB!)

A better term for this is *confluence*: all execution orders lead to the same outcome. (The Church-Rosser property.)

### Coordination
But what the heck counts as coordination? What's the difference between a message that is required (e.g. for dataflow) and one that is mere "coordination"? After all the expression `a + b` needs to wait for the values `a` and `b` to get bound, but that waiting is implicit in the program spec, it's not an artifact of implementation!

Here's what I see as Ameloot's big idea: coordination messages are the ones you send because you do not know how the data in the system is distributed.

But how do you identify those?

Well, consider *all* partitionings of the data, in a "many worlds" kind of philosophy. Look at the messages that are sent. Some worlds will be able to achieve the dataflow without sending messages. So the coordination messages are the ones that are sent in "all worlds".

For the clearest intuition, look at the world where all the data happens to be on the same machine. In that world, all dataflow (deduction) is achieved without messages. So what messages would you send? You already can compute the contents of IDB relations, no? But what about messages that help with questions like "am I done with this stratum"? Those are not deduction/dataflow, but they are still required *if you want stratified negation*!

The arguments in the paper are more general/subtle, but this gives you the kernel of the idea.

## Obliviousness
Ameloot's other big idea was to show a third equivalence:

The following classes of queries are the same:

1. Monotonic queries
2. Queries that are distributedly executable by transducers without coordination
3. Programs that are distributedly executable by *oblivious* transducers

What is an oblivious transducer?? It's one that has no access to the relation *All* (which stores the addresses of all transducers) OR *Id* (which stores the address of this transducer).

This is a big extension to the CALM Conjecture!

Some intuitive ways to think about it:
- Monotone <=> Oblivious: An "open world" of agents can eventually compute a monotone query by sending anonymous messages (including not recognizing your own messages)! (Conversely, if you have a monotone query, you don't need to track membership or identity!)
- Coordination <=> Non-Oblivious: Membership is sufficient to implement Coordination, and conversely a Coordination protocol can be used to establish Membership. 
- Non-Coordination <=> Oblivious: A network of oblivious nodes cannot implement coordination protocols (Paxos, 2PC, etc.) (Conversely, in the absence of coordination protocols, you don't need to track membership or identity!)


# CAP and CALM
CAP is an impossibility result about the class of all programs, like many key results in distributed systems.

CALM is a dichotomy result: it says there are 2 classes of problems, and a discriminator between them -- monotone vs. non-monotone (or if you prefer oblivious v. non-oblivious!)

Note that the A&P in CAP induces reordering of messages: during a period of partition when the system is still available, some messages are not propagated, they are delayed until the time the network is healed. The question is whether the results are "consistent".

If we parameterize the C in CAP and CALM, and look at Gilbert/Lynch vs Ameloot, we can see how the two theorems relate.

1. If C is defined as Linearizability -- i.e. an agreed total order -- it's clearly a non-monotonic condition that requires coordination. A total order is an assignment of "predecessor" to each node in a graph. If we decide that `pred(x, z)` is a fact, a new fact could arrive later `pred(x,z)` which forces us to retract one of those two facts as they can't both be true in a predecessor relation!
2. If C is defined as confluence, then we get the CALM dichotomy.

Brewer's quote above is useful here: if we don't know our application semantics (invariants), we might want to choose a strong consistency semantic like Linearizability. 

However, we'd like to move to a world where we do know -- and can prove (e.g. in a compiler) -- that the semantics of some block of code is monotonic, and hence can achieve eventual consistency without coordination.
