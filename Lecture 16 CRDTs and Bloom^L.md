# CRDTs<a name="CRDTs"></a>

"A solution to the CAP Theorem". This should make you nervous! Indeed, CRDTs should make you nervous, as they contain footguns. [Use with caution](https://www.vldb.org/pvldb/vol16/p856-power.pdf)!

## The State-Based Model
This paper is kind of the opposite of Pat Helland's style -- it prefers notation to description. It's all fairly simple, so the notation feels a bit heavy for the task at hand. In any case, don't let it bother you.

- Each piece of state is replicated. It's easiest to think of state in the context of a single object, but it may be a large, growing object of a collection type (e.g. a set or the like.)
- There are three operations: *query*, *update*, and *merge*
- *Updates* are performed at any replica. (A.k.a. "multi-master"ed data)
- Replicas occasionally (but infinitely often) send a snapshot of state to other replicas, who *merge* the snapshot from the message into the local copy of the object.
    - This is often referred to as "gossip", particularly if the messages are sent at random to neighbors.
    - The idea of gossip + Eventual Consistency goes back at least as far as the [Bayou](https://dl.acm.org/doi/pdf/10.1145/224057.224070) system, which is an important read if you care about loose consistency and data.
- Each node has a local logical clock and timestamps its snapshots of state.
- State in a message may have *preconditions*; messages are deferred so that merge is only applied when the precondition is satisfied. (more on this below)

We can assume that a DAG of causal history of an object is maintained at each node. The causal history of an object is a DAG of timestamped update and merge statements.

The paper notates the causality DAG as an array of histories from the $n$ nodes in the system $[c_1, ..., c_n]$. *Update* at time $k$ at node $i$ (setting the value to be $a$) is denoted $**u_i**^k(a)$. *Merge* at $k$ at node $i$ takes a remote value as its parameter, so it looks like $m_i^k(s_{i'}^{k'})$. Note that the merge drags along the causal history from the remote node and merges (unions) it into the local DAG of causal history.

### Strong Eventual Consistency
We can say an update is *delivered* once it is included in causal history.

If you're into notation, the paper uses two common pieces of notation from linear temporal logic (LTL):

- $\lozenge p$ ("eventually") the predicate $p$ will hold at some future time
- $\square p$ ("always"): the predicate holds now and at all future times

Definition: *Eventual Consistency*. An update delivered to one replica is eventually delivered to all replicas: $\forall i, j: f \in c_i \implies \lozenge f \in c_j$. 

Convergence: Correct replicas that have delivered the same updates eventually reach equivalent state: $\forall i, j: \square (c_i = c_j) \implies \lozenge\square (s_i \equiv s_j)$.

Strong Convergence: Correct replicas that have delivered the same updates 
have equivalent state: $\forall i,j: c_i = c_j \implies s_i \equiv s_j$. (Note this is an "instantaneous" constraint that should hold at all times.)

Definition: *Strong Eventual Consistency*: An object is Strongly Eventually Consistent if it is Eventually Consistent and has Strong Convergence.

> Simply put, an object exhibits Strong Eventual Consistency if it eventually receives all updates, and applying the same causal history of updates produces the same state.

Note that this formalism relies on tracking causality. As we discussed in prior lectures, this has scalability challenges! Fortunately, there's a simpler abstraction on offer as well.

### State-Based CRDTs: Lattices 
One of the nicest things about CRDTs is that they put the intuition of "ACID 2.0" in its classical modern algebra frame. Modern algebra is increasingly relevant in databases these days, so we'll be seeing more soon! If you didn't take that class in undergrad, don't worry -- we'll mostly be using very basic results.

#### Background: Lattices
As you read the definition below, think of the example domain *sets of integers*, i.e. $D = {\mathcal P}({\mathbb Z})$. Most of your intuition about a set comes from the fact that it is an example of a lattice.

> Definition: A *Lattice* $(D, \sqcup, \sqcap)$ has a domain of values $D$. It also has two binary *operators*. The operator $\sqcup$ is called *join* (confusing in database class!) or *least upper bound (LUB)*. The operator $\sqcup$ is called *meet* or *greatest lower bound (GLB)*$.
> The following axioms hold on a Lattice:
> - Associativity: 
>     - $x \sqcup (y \sqcup z) = (x \sqcup y) \sqcup z$
>     - $x \sqcap (y \sqcap z) = (x \sqcap y) \sqcap z$
> - Commutativity:
>     - $x \sqcup y = y \sqcup x$
>     - $x \sqcap y = y \sqcap x$
> - Idempotence: 
>     - $x \sqcup x = x$
>     - $x \sqcap x = x$

> Definition: Given a lattice $L = (D, \sqcup, \sqcap)$, the structure $(D, \sqcup)$ is called a *join semilattice*, and the structure $(D, \sqcap)$ is called a *meet semilattice*.

> Definition: A *bounded lattice* $(D, \sqcup, \sqcap, \top, \bot)$ is a lattice in which $D$ contains two distinguished values $\top$ (*top*) and $\bot$ (*bottom* or *bot*) such that for all $x \in D$: 
> - $x \sqcup \top = \top$
> - $x \sqcap \bot = \bot$
 
Useful things to know about lattices:
> Observation: *every semi-lattice $L$ is a partially ordered set* on the elements of $D$. That is, for all $x, y \in D$:
> - $x <_L y$ if $x \sqcup y = y$
> - $x <_L y$ if $x \sqcap y = x$
>
> Observation: A join semilattice is *monotonic* under $\sqcup$. That is, $\forall x, y \in D, x < x \sqcup y$.

#### Returning to State-Based CRDTs
If we define the *merge* function of our CRDT to be associative, commutative and idempotent (ACI), then our CRDT states form a semilattice, and merge is monotonic. We typically assume a state-based CRDT's semi-lattice includes a $\bot$ element, to capture the state of replicas that have received no messages.

Now we have "Theorem 2.1" from the CRDT paper: *Assuming eventual delivery and termination, any state-based object that satisfies the monotonic semilattice property is strongly eventually consistent*.

An ACI *merge* operation trivially satisfies the theorem, so all that's left is to argue that the *update* operation cannot mess things up. In the proof, they make the offhanded assertion that *update* is "non-mutating". Let's be more formal and assume that updates are *monotonic w.r.t. lattice order $L_{merge}$*. One common way this happens is when we force updates to be elements in $D$ and define *C.update(x)* as *C.merge(x)*.

> Note: A value-based CRDT does not require causal delivery. Physics of networking guarantees that *update* happens-before the update is received and *merge*d into replica state. Since *merge* is ACI, the order of merging has no effect on strong convergence.

#### Examples of State-Based CRDTs:
I find it simplest to think about semi-lattices (i.e. the *merge* operation) and use an *update* operation that comes naturally.

By now there are many fancy CRDTs in the literature, but as our next paper on Bloom$^L$ advocates, most useful CRDTs are built out of simple building blocks:

- *Monotonic Set*: $(\mathcal{P}(T), \cup)$ is a join semi-lattice over sets of objects of type $T$. We can define a state-based CRDT of a monotonically growing set via *merge* = $\cup$ and *update* = $\cup$.
- *Counter*: $(\mathbb{Z}, \max)$ is a join semi-lattice. *Increment* is monotonic wrt that lattice. Hence we can define a state-based CRDT over integers where *merge* = $\max$ and *update* = *increment*.
- *Lattice Product*: Given semi-lattices $L_1 = (D_1, \sqcup_1)$ and $L_2 = (D_2, \sqcup_2)$, we can construct a semi-lattice $L = L_1 \times L_2 = (D_1 \times D_2, \sqcup_L)$ where $\sqcup_L((x_1, x_2), (y_1, y_2)) \stackrel{def}{=} ((x_1 \sqcup_1 x_2), (y_1 \sqcup_2 y_2))$. *It is simple to show that a lattice product is a lattice.* Repeated use of this gives arbitrary-width "tuple" or "vector" semi-lattice constructors.
- *Lattice Map*: Given semi-lattice $L = (D, \sqcup_L)$ and a domain of values $K$ (keys), we can define a domain of maps $M = K \times L$ that pairs keys to lattice-typed values. We can then construct a key-wise join operation $\sqcup_M(m, n) \stackrel{def}{=} \bigcup_{k \in K}\{m[k] \cup_L n[k]\}$. *Note that the key domain $K$ is not a semi-lattice.*


### Operation-Based CRDTs
While state-based CRDTs are simple, they're a bit clumsy: they ask each node to periodically gossip its *entire state*, kind of like physical checkpointing. We might prefer a kind of "log-shipping" approach instead (see discussion of the Quicksand paper).

Recall how the Dynamo shopping cart held a set of *logical commands* like `ADD_TO_CART`. We'd like to formalize shipping a log of such commands.

Op-based CRDTs do not have a *merge* function. Instead they an *update* function that consists of two parts:
- a *prepare-update* operation $t(a)$ that executes once, at the source of the update
- an *effect-update* operation $u(a')$ that executes immediately after $t$ at the source, *and* executes upon arrival at each replica.
  
The *causal history* of an op-based CRDT is defined to record only the *effect-update* operations. Both *query* **and** *prepare-update* operations are ignored.

> Note: Op-based CRDTs *require* causal delivery. For liveness they also require that causality is the precondition for op-based CRDTs.

But wait! Even with causal delivery we may have concurrent updates! Then what? We'll, we'd need them to be commutative!

> Hence Op-based CRDTs *require* commutativity of (concurrent) effect-update operations.

## Equivalence of Op- and State-Based CRDTs
I think about this differently than it's presented:

> Observation 1: A vector clock is a state-based CRDT (a vector of counters).

> Observation 2: If senders follow the vector clock protocol, there is a strict functional dependency from vc to a unique payload (vc uniquely determines payload). Hence vc-stamped messages are semi-lattices: the "merge" for two messages with the *same* vc-stamp is "choose-either", since they're identical.

> Observation 3: Causal broadcast of a message $m$ from sender $i$ to receiver $j$ is achievable with vector clocks (e.g. see the [Schipper-Eggli-Sandoz](https://infoscience.epfl.ch/record/50198/files/SES89.pdf) protocol, or the discussion in this paper.)

> Observation 4: Op-based CRDTs are essentially state-based CRDTs (VCs) with an "op payload".

> In short, all CRDTs are state-based CRDTs. So-called "op-based" CRDTs are just *causal delivery of commutative operations* (and causal delivery is achievable via state-based CRDTs.)

The paper also points out that you can "implement state-based CRDTs via op-based CRDTs", but that direction is (a) entirely mechanical (it basically uses *merge* as the operation being transmitted) and (b) it's over-constrained -- it just forces causal delivery on the state-based CRDT.

## The problems with CRDTs
As pointed out in the Bloom$^L$ paper, a problem with CRDTs is that people tend to implement fancy ones, using traditional programming languages. How do you convince yourself that a large pile of Javascript code is ACI? Did you write any stateful code recently? Is it commutative over its inputs, for example? We don't usually have a good sense of such things.

This critique is not new, any more than ACID 2.0 is new -- the idea of writing ACI code is folklore from long ago, and oft-abandoned for that reason.

### Queries are Footguns
Worse, as documented in [a paper from the hydro group last year](https://www.vldb.org/pvldb/vol16/p856-power.pdf), CRDTs are sort of an uncomfortable halfway point between reasoning about opaque state (general-purpose, 20th century consistency) and reasoning about program outcomes (app-specific, 21st century consistency):

- The CRDT math of monotonic updates and semi-lattice merge is sound, and gives eventually-consistent state.
- There is no "math" for the *query* operation. To reiterate from the Quicksand paper, *"READ is annoying"* and does not commute with other operations. Said differently, revealing the "current state" of a monotone process produces a non-monotone fact!
  - An exception: revealing that the current state is $\top$ *is monotone*! This will turn out to be important.
- Inconsistent *query* results can cause (as in causality) inconsistent updates, so what's the point of the CRDT guarantees for *update*?! CRDTs are not really a closed system of consistency at all!

Many CRDTs have been implemented with even more obviously non-monotone query functions. For example, consider the so-called "2P-Set" CRDT, which is a commonly-used tuple of 2 SetUnion semi-lattices that support updates to Add and Delete elements. One set in the tuple holds elements that have been added, and another holds elements that have been deleted. The *query* interface is essentially $added - deleted$, which of course grows monotonically with Add and shrinks anti-monotonically with Delete, so all in all the query results are non-monotonic and *do not commute with updates*, even though the storage is EC.

## LVars
Concurrent with the CRDT work, Lindsay Kuper was doing a PhD at Indiana with Ryan Newton, introduing the idea of [LVars](https://users.soe.ucsc.edu/~lkuper/papers/lvars-fhpc13.pdf), which use lattices for what they called "deterministic-by-construction" parallel functional programming in Haskell.

Again, LVars are semilattices. The nice addition is the notion of *threshold reads*: read includes a "threshold" parameter, and returns when it's above threshold. This is a bit technical in LVars (look at the definition of *get* and *threshold sets*). Main point: this generalizes the idea that we can reveal $\top$, in that we can also *reveal that we are "greater-than" some value*. More on this in our discussion of Bloom$^L$.

# Bloom$^L$
## Bloom
Bloom was intended to be a programmer-friendly version of Dedalus -- i.e. $Datalog^\neg$ with time and space. It made three main decisions to start:

1. Instead of logic, borrow the functional/comprehension syntax becoming popular in scripting languages like Ruby and Python. (We chose to embed Bloom in Ruby, so it was unfortunate that Python won that battle.)
2. Present a single-transducer programming model, rather than the Dedalus "global database" model. This was a decision based on (a) the confusion we had with global-thinking in NDlog, and (b) a step away from Single-Program Multiple Data (SPMD) to a more actor-like view of potentially different code on different nodes.

Beyond that, much was inherited from Dedalus, but packaged up more intuitively. First, 
Bloom has explicit keywords for the relations used in persistence and communication. All collection types have a named schema, as in SQL. Bloom state declarations appear in a special `state` block and consiste of the following:
- `table`: a persistent relation
- `scratch`: like a table, but persists only until end-of-tick.
- `channel`: like a `scratch`, but includes a *location specifier* field for the destination, where it appears for a single tick
- `periodic`: a channel that auto-produces tuples periodically according to the system clock
- `interface`: a scratch that can be used in the head of one module and referenced in the body of another module

Note that Bloom allows EDB relations in the heads of rules. (This is just syntax sugar).

Bloom rules, like Datalog rules, are unordered declarations, and appear in a `bloom` block. Bloom captures the temporal logic of Dedalus in the merge operators:
- `<=` is *instantaneous* (deductive) merge
- `<+` is *deferred* (inductive) merge
- `<-` is *deferred* (inductive) delete
- `<~` is *async* merge. 
The head of an async rule must be a `channel`, and a `channel` cannot be in the head of any other kind of rule.
  
One of the cool things about Bloom was that it had *automatic CALM analysis of eventual consistency*. It can build dataflow graphs from syntax, with operators labeled monotonic or non-monotonic, and edges downstream of networking labeled non-deterministic. *A non-monotonic operator downstream of a network ingress has inconsistent outputs*. 

Bloom would check these properties transitively and plot the dataflow graphs, with green edges for consistent and deterministically ordered flows, yellow for consistent and non-deterministically ordered, and red for inconsistent. 

## Bloom$^L$
- Inspired by ACID 2.0, LVars and CRDTs: embrace lattices
    - Not just sets/relations -- any lattices!
    - It gets weird trying to write monotonic counters in logic, for example

- Dataflow composition is done via Bloom
    - dataflow operations like `map`, `filter`, `join`, `reduce`, `groupby` on sets of tuples possibly containing lattice-type columns
- Type composition via composable lattice types
    - `lbool`, `lmax`, `lmin`, `lset`, `lpset`
    - `lmap`: a composite lattice mapping key values to lattice values
    - tuple: if al fields are lattices, a tuple is a lattice
- BYO Lattice
    - define `initialize` and `merge` functions

## Lattice Methods, Monotone Functions and Morphisms
A lattice can have arbitrary methods (functions) defined on it. Consider a function $f$ from a lattice $L = (X, \sqcup_X)$ to a lattice $M = (Y, \sqcup_Y)$:

- $f$ is *monotone* if $\forall a,b \in X: a \le_L b \implies f(a) \le_M f(b)$.

- $f$ is a *morphism* if $\forall a, b \in X: f(a \sqcup_X b) = f(a) \sqcup_Y g(b)$. That is, $f$ distributes through $\sqcup$. Said differently, $f$ maps items of $X$ to items of $Y$ in a way that respects the lattice properties.

All morphisms are monotone functions, but not vice versa. E.g. $size(\{1, 2\} \sqcup_{lset} \{2, \3}) \ne size(\{1, \2}) \sqcup_{lmax} size(\{2, 3\})$.

## Example 
```ruby
QUORUM_SIZE = 5
RESULT_ADDR = "example.org"

class QuorumVoteL
  include Bud

  state do
    channel :vote_chn, [:@addr, :voter_id]
    channel :result_chn, [:addr]
    lset    :votes
    lmax    :cnt
    lbool   :quorum_done
  end

  bloom do
    votes       <= vote_chn {|v| v.voter_id}
    cnt         <= votes.size
    quorum_done <= cnt.gt_eq(QUORUM_SIZE)
    result_chn  <~ quorum_done.when_true { [RESULT_ADDR] }
  end
end
```

### Morphisms and Semi-Naive Evaluation
Let's look at this by example: shortest paths.

```ruby
state {
  table :edge, [:from, :to, :cost]
  table :path, [:from, :to, :cost]
  lmap :shortest_path, [:from, :to], [cost]
}

bloom {
  path <= edge
  path <= path.join(edge, :to => :from) {|p, e| 
      [p.from, e.to, p.cost + e.cost]
  }
  shortest_path <= path.group([:from, :to], min(:cost))
}
```
Semi-naive evaluation will execute this in rounds. In round 1, we use the first rule to find $path_1$ -- i.e. all paths of length 1 (i.e. the `edge` relation). In round 2, we join the new paths of length 1 with edges to get $path_2$ -- paths of length 2. In each round $i$ we join the new paths of length $i$ with edges to get $path_{i+1}$. Each path is added in only one round.

But what happens with `shortest_path`? In round 1, the `min` aggregate finds, for each distinct `[:from, :to]` pair, the lowest `:cost` among paths of length 1.  But what happens in round $i$? Here the `group` only looks at "new" tuples -- paths of length $i$. It finds the shortest among these. And then it *merges* the results into the `shortest_path` lattice. By definition this performs $\sqcup_{min}$ on the values for each key. 

Effectively, in semi-naive, we perform 
$$semi\_naive\_cost = min(path_1) \sqcup_{min} ... \sqcup_{min} min(path_n)$$ 
If we did naive evaluation, then in round $n$ we would compute $$naive\_cost = min(path_1 \sqcup_{lset} ... \sqcup_{lset} path_n)$$ 

And we can show that:
$$semi\_naive\_cost = naive\_cost$$
Semi-naive is equivalent to naive in this case because `min` is a morphism to the set lattice (of logic): $min(X \sqcup_{lset} Y) = min(X) \sqcup_{lmin} min(Y)$.  (It should now be clear that monotone functions that are not morphisms do not admit semi-naive evaluation, as the above equivalence would not be guaranteed!)

This idea was revisited in a formal setting just a couple years back, by Khamis, et al: [Convergence of Datalog over (Pre-) Semirings](https://dl.acm.org/doi/10.1145/3517804.3524140) in their extension of datalog with semirings, $datalog^\circ$.