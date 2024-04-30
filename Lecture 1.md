<!-----

You have some errors, warnings, or alerts. If you are using reckless mode, turn it off to see inline alerts.
* ERRORs: 0
* WARNINGs: 0
* ALERTS: 7

Conversion time: 3.135 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs to Markdown version 1.0β36
* Tue Apr 30 2024 14:18:37 GMT-0700 (PDT)
* Source doc: Lecture 1: System R Classics
* Tables are currently converted to HTML tables.
* This document has images: check for >>>>>  gd2md-html alert:  inline image link in generated source and store images to your server. NOTE: Images in exported zip file from Google Docs may not appear in  the same order as they do in your doc. Please check the images!

----->


<p style="color: red; font-weight: bold">>>>>>  gd2md-html alert:  ERRORs: 0; WARNINGs: 0; ALERTS: 7.</p>
<ul style="color: red; font-weight: bold"><li>See top comment block for details on ERRORs and WARNINGs. <li>In the converted Markdown or HTML, search for inline alerts that start with >>>>>  gd2md-html alert:  for specific instances that need correction.</ul>

<p style="color: red; font-weight: bold">Links to alert messages:</p><a href="#gdcalert1">alert1</a>
<a href="#gdcalert2">alert2</a>
<a href="#gdcalert3">alert3</a>
<a href="#gdcalert4">alert4</a>
<a href="#gdcalert5">alert5</a>
<a href="#gdcalert6">alert6</a>
<a href="#gdcalert7">alert7</a>

<p style="color: red; font-weight: bold">>>>>> PLEASE check and correct alert issues and delete this message and the inline alerts.<hr></p>


**Selinger ‘79 Access Path Selection…: The Bible of Query Optimization**

The Setup



* Imagine I give you a domain-specific language
    1. Now write a compiler!
    2. Oh … and imagine you’re living in the 1970’s.
        * This is nigh impossible for modern computer scientists to imagine.
        * The nature of innovation in applied CS at that time was completely different
            * Very unclear what was “important”.
            * Time-consuming to try and validate ideas in practice.
            * Very few prior designs in the literature and minimal reusable code.
                * That’s why UNIX and Ingres SO important as a result
            * Communication was via printed papers, physical travel to conferences, telephone, and [limited forms of email](https://en.wikipedia.org/wiki/History_of_email), mostly internal. (SMTP dates to 1983).
* Let’s be agile! What are some initial ideas for the first sprint to get the compiler to help?
    1. Divide and conquer: Break expressions down into subproblems, optimize those.
        * INGRES “one-variable query processor” and DECOMP
        * Pick smallest table. For each tuple, recursively run the query remainder.
    2. Transformation rules
        * Can push filters before joins, will make things faster!
            * filters cheap, joins slow!
* OR … how about we build an oracle.
    1. Ask it a question, it instantly “looks up” the best answer.
    2. There’s a finite space of possible “binaries” (plans) for our query.
    3. _Optimization as Search!_
        * **Enumerate** all possible plans  that implement the program.
        * **Choose** the one that optimizes your objective_ in_ _your context_
* This is a ridiculous idea. It will never work. CERTAINLY not in 1979!

    

<p id="gdcalert2" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image2.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert3">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image2.png "image_tooltip")


    8. Search space size: n! join orders
        * Heuristic pruning can help: no cross products
    9. Objective function: empirical. Pick one or more: latency, utilization
    10. So the Oracle will need to run each possible plan and measure it?
        * No, we need a cost predictor!
        * Worse .. this is not a static compilation problem. It’s entirely context (data) dependent!

OK but .. let’s try anyway?



* Our _n _is often small. Let’s crush it for those use cases.
* How hard could the prediction problem be? Let’s be agile on that front.
    11. Build a noisy predictor and make it better over time.
* Break the problem down
    12. Define the search space (and maybe some pruning heuristics)
    13. Build a noisy predictor
    14. Design an intelligent search algorithm

    We can iterate on each of these independently!!! (And the community has continued to do so for ~50 years!)


An initial cut



* A. Search space
    * A “Physical” relational algebra
    * Modern Algebra is lovely: typically a small # of operators _with axioms_ _that define a search space!_
        * In modern algebra the number of ops is like 1 or 2.
        * Relational alg is like 6, but we mostly focus on 3 (and really just 2)
            * JOIN and UNION. I.e. x and +. Semirings!
* B. The System R search algorithm:
    * Core observation:
    * 

<p id="gdcalert3" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image3.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert4">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image3.png "image_tooltip")

        * Bellman’s “principle of optimality”. Dynamic Programming.
    * DP expanded with context (“interesting orders”)
        * 

<p id="gdcalert4" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image4.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert5">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image4.png "image_tooltip")

    * You should be able to explain Selinger algorithm for a simple 3-way join query
        * SELECT *  \
  FROM students, enroll, students \
 WHERE student.id = enroll.sid \
    AND enroll.cid = classes.id
    * Still (3<sup>n</sup> - 2<sup>n+1</sup> + 1) / 2 [Ono/Lohman 1990!]
        * No cross-products?
            * (n-1)*2<sup>n-2</sup> for star queries 
            * n<sup>3</sup>-n / 6 for chain queries
* C. Cost predictor:
    * The cost of the algorithms isn’t that hard, let’s be agile.
    * The context (data) is a giant headache.
        * cost(O1(O2(R, S), T) – input to O1 are outputs of O2.
    * _Selectivity estimation _problem
        * Unfortunately it’s a really hard statistical problem
        * (Bad) simplifying assumptions
            * Assume uniform distribution in each column
            * Assume data values are independent across columns/tables
            * Assume query predicates are independent
        * Removing these assumptions altogether is very hard
            * Only very recent techniques starting to get accurate
    * Wanna feel better about it?
        * We don’t need actual costs, just ranking of costs!
        * 

<p id="gdcalert5" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image5.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert6">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image5.png "image_tooltip")


Extra goodies



* Clever handling of subqueries. 
* Presages handling of “expensive” UDFs
* Leaves open the question of subquery flattening

Takeaways for modern researchers



* Top-Down SW research
    * See the big problem in its fullness, and factor it.
        * Only then be “agile” on the pieces
    * As opposed to bottom-up design celebrated in the UNIX philosophy
    * (It’s fun to compare the UNIX and System R papers … many common goals, many good but competing lessons to weigh!)
        * You should develop opinions about when to lean on which tradition!
* Theme: DB is a cross-cut of computing
    * Note: this is a classic “AI” problem, without the pomp and circumstance
        * Search (DP is popular in AI)
        * Statistical prediction
        * Decades before AI was viewed as “useful”
    * Database research is not limited to “Systems” or “Theory” or “Inferential reasoning” or suchlike tribalisms. It’s a license to do Computer Science.
* Understand your constants
* Some topics are both useful _and _open problems for decades
    * What are your favorites?

At a high level, all respectable DBMSs implement a version of this very optimizer.



* Search space made extensible (BYO operators/axioms)
* Search algorithms tuned (top down vs bottom up)
* Prediction techniques improved

**Grey, et al. ‘76: Granularity of Locks and Degrees of Consistency**

This is a beautiful and flawed paper. (Well, two papers).

**Lock Granularity**

You all remember 2PL, right? Let’s review.

But what to lock?? Apparent Tradeoff: Concurrency vs. overhead



* To minimize false contention, we want fine-grained locks.
    * Lock every bit?!
* To minimize overhead, want each transaction to take out very few locks
    * Lock the entire database with 1 lock!
* Idea: what if I just lock as much as I use?
    * Jim reads EMP, so S-locks EMP
    * Pat updates Jim’s EMP record, X-locks just that record
    * How do we ensure two transactions “rendezvous” and check for conflicts?
        * Idea: define a _hierarchy_, leave bread-crumbs as you walk it top-down
* Simple version: 
    * Table > Block > Tuple
    * Walk from the top down, leaving bread-crumbs of your plans 
        * request “intent locks”
            * IX, IS, SIX
            * in addition to S and X
    * When you get to the unit you want to lock, request a “traditional” lock
        * Recurse no further to save overhead!
    * Rendezvous ensured relatively cheaply
        * Can’t miss – everybody requests a lock at the root
        * Intent locks at coarse-grained levels of hierarchy, so minimal overhead
    * Next question: when should the lock manager make you wait?
        * You should be able to puzzle out the lock compatibility matrix
        * Lattice of locks: IS &lt; {S, IX} &lt; SIX &lt; X
* Extension from a tree of locks to a DAG
    * E.g. DB > Table > {File, Index} > Tuple
    * How to ensure rendezvous…?
* Further extension to dynamic lock graphs
    * Stated goal: locking key intervals
    * Not aware this was ever used. 
        * Locking key intervals doesn’t work nicely. Why??
    * I wouldn’t spend your cycles on this

**Degrees of Consistency**

 \
WAIT … is this paper defining a “transaction”? Is it defining two-phase locking? **_Oh heck yes!_**



* I had always assumed some theoretician had defined transactions in prior work and the Jim Gray and co were “systems people” who figured out how to make them go fast.
* Not so! Theory and Practice all came from the System R group.
* The Gray/Reuter book has long, painful prehistory of transaction-like ideas.

First, note the definition of Consistency: the DB satisfies all assertions (invariants).



* The “replica consistency” of CAP and CALM is a special case of database consistency
* Confusion often ensues

Second: Discussion of system-enforced vs user/app-enforced lock requests.



* The right abstraction was not self-evident. Maybe there should be more than one? 
* “User controlled locking results in potentially fewer locks due to the users knowledge of the semantics of the data. On the other hand, user controlled locking requires difficult and potentially unreliable application programming.”
* Challenge: Can adjust the system vs user (automatic vs custom) locking control flexibly?
    * Even better: Can we isolate one transactions choices on this from another’s?

Next: another classic System R Top-Down design goal.



1. Define a semantics for multiple degrees of consistency, in terms of what a transaction can “observe”. An “isolation level” (the I in ACID) or “degree of consistency” (let the confusion commence!)
2. Then define mechanisms (locking protocols) that provide the desired semantics

Semantic definition from this paper follows. First, note distinction between “dirty” (non-atomic) and committed.


    

<p id="gdcalert6" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image6.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert7">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image6.png "image_tooltip")



    Locking Protocols to achieve these degrees of consistency follow.


    **Definition**: a transaction is <span style="text-decoration:underline;">well-formed</span> if it always locks an entity in appropriate mode (exclusive or shared) before the corresponding action (write or read, respectively). 


    **Definition: **a transaction is two-phase if it does not lock an entity after unlocking any other entity.


    

<p id="gdcalert7" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image7.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert8">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image7.png "image_tooltip")



    Assertions (humble mini-theorems): \




1. The locking protocols ensure the desired semantics.
2. If all transactions run at least Degree 0 locking, then all transactions will observe their desired degree of consistency.

Pull-quote: “We wish to emphasize that this system is a vehicle for research in

data base architecture, and does not indicate plans for future IBM products.”

Takeaways for modern researchers:



* Gray & co. crushed transactions in their first system.
    * Was built with lots of context from earlier IBM systems
* Ambition of top-down design, separation of semantics from implementation
* Problems defined and solved by embracing combo of definitional/correctness design issues and implementation/performance issues. 
    * Locking overheads very much on their minds