# Predictive Interaction & Lessons from Wrangler/Trifacta
- [Potter's Wheel](http://control.cs.berkeley.edu/pwheel-vldb.pdf) was a Berkeley research system in the early days following [Online Aggregation](http://control.cs.berkeley.edu/online/online.pdf) in the [CONTROL project](https://control.cs.berkeley.edu/ieeecomputer.pdf).
    - DSL for data wrangling
    - Scalable Spreadsheet + DSL-Menu UX
    - Type inference algorithm including user-defined types
    - Anomaly detection algorithm
- [Wrangler](http://vis.stanford.edu/papers/wrangler) and [Profiler](http://vis.stanford.edu/papers/profiler) were the follow-ons, in concert with "real" HCI/DataVis folks (Jeff Heer and Sean Kandel)
    - **Predictive Interaction** in place of menus
        - Arguably the main contribution over Potter's Wheel
        - A paradigm for human/AI interaction, targeted at this use case
        - Articulated more fully in [CIDR paper on Predictive Interaction](https://idl.cs.washington.edu/files/2015-PredictiveInteraction-CIDR.pdf)
    - Interactive, NL-based History
    - Compilation of DSL to other languages (JS, Python, SQL)
    - Anomaly detection and data quality visualizations
    - Sample-based spreadsheet

- Trifacta was the commercialization of Wrangler/Profiler
    - Originally targeted at Big Data (Hadoop, etc)
    - Subsequently made into serverless SaaS, over SQL, Big Data, Pandas, etc.
    - Many bells and whistles
    - Acquired by Alteryx (market leader in end-user data prep) in 2022

## Slides on Predictive Interaction
- [Powerpoint](https://docs.google.com/presentation/d/13B9DP7hKRRyZEEyy6m8ZRMMka9015h0O/edit?usp=drive_link&ouid=106010423489779239860&rtpof=true&sd=true)
- [PDF](https://drive.google.com/file/d/13JjaciWKU5j3dMdiM_cSwNu1Nnc-AiX_/view?usp=sharing)
- I don't currently have access to Alteryx, so we'll have to settle for a [Video Demo](https://drive.google.com/file/d/139GPT_iRLwvrx3z71R_sBfLaz0JOwfMN/view?usp=sharing)

# LLMs meet Databases: Data, Dreams and People
- It would be foolish to ignore the potential of LLMs in 2024.
- Seems like three classes of usage:
    - **Rote work** like office text/stock-image generation, code assistants. *Grounded* by human review.
    - **Dreams**, include image and story generation, NPC behavior. *Bounded* by rules/contraints.
    - **Text search/summarization**, *grounded* by data (citations/links)
- In the end, LLMs are *dream machines*
    - Suggestive, seeded with recorded data, but not bound by reality in the way they search/recombine.
    - In high-value real-world applications, LLMs need to be grounded (by data) or bounded (by logic), and reviewed by humans
    - There is a wide range of computer science to be done here, from HCI to DB to AI to PL to Systems.
- Caveat to this aspect of the lecture: 
    - We are in early days, it's impossible to keep up with everything, and I'm not trying. 
    - I wish I had time to review these slides with colleagues/students *before* class, but hopefully a framework for followup
    - This is neither complete nor well-attributed. More of a zeitgeist/brainstorm than a scholarly talk.


# LLMs for Data Wrangling?
- I still think this is an ideal Petri Dish for grounded research in this space
    - Side steps tasks humans are "good at": verifying prose/images
    - Raises all the questions of:
        - Soft boundary between human exploration and specification
        - Verifying outputs that humans are not so good at assessing
        - Code synthesis vs. Output synthesis
        - Tight feedback loops between AI, People, and deterministic SW (checkers, summarizers, data visualization, etc.)
    - [EPIC Data Lab](https://epic.cs.berkeley.edu) is pursuing!
- Many wrangling tasks can benefit from LLM inference
    - Data restructuring
    - Standardizing formats
    - Type induction
    - Entity resolution
    - Deduplication
    - Anomaly detection
    - Data imputation
    - Mock data generation
    - Target schema mapping suggestions
    - Join suggestions
    - Data augmentation
    - ...
- All this is approximately useless without the right frame
    - UX feedback loop for human validation: 
        - Predictive Interaction
        - Guide/Decide
        - Data quality visualization
    - Program synthesis for reuse, scaling, verification

# From Chatbots to RAG
- Chatbots compress and dream in two ways:
    1. Compress a large corpus into a computational structure
    2. Dream up a (more) useful form/starting point for each user query.
    3. Dream up a relevant "answer" progressively, starting with the query.


- Consider how this works for RAG. Scenario: an assistant for PostgreSQL users
        - This assistant should give excellent answers to relevant questions.
        - This assistant shouldn't engage in irrelevant discussion ("how are you?", "how do I do this in Oracle?")
        - Certainly shouldn't say something like "this is supported in DuckDB, but not PostgreSQL"
        - Exploring these ideas in practice at [RunLLM](https://runllm.com)

- Here's the basic RAG structure
    1. Train LLM, index documents in a (vector) database. Documents need to be chunked "well".
    2. Rewrite query using both sources and dreams
        - RA: First, take query and load nearby documents (data) into the query prompt (context) as well.
        - HYDE (Hypothetical Document Embedding): Given the user query, dream up a generated answer, and use *that* instead. 
    3. Generate answer(s) from the LLM
    4. Enhance trust by Grounding in citations: Check proximity of answers to the corpus (e.g. keyword overlap density) and annotate answers with citations -- if no good citations say "I don't know"!

# NL Querying of Relational Data a la RAG 
## i.e., relational data, LLM-driven query processing
- One can just do text-to-SQL and use the relational DBMS
- Consider how we might use RAG-style design:
    1. LLM + compressed database? How to embed DB tables?
    2. Taking NL input and turning into something better.
        - one or more formal queries in SQL?
        - more carefully-worded NL?
        - both?
    3. Generate answer tuples
        - using an LLM?
        - using a DBMS?
        - both? cheaper LLMs for subtasks?
    4. Enhance trust by Grounding/Bounding
        - Ground with provenance
            - Not a great general UX (often TMI, esp with negation/aggregation)
        - Bound with constraints (e.g. the full SQL text? and SQL results? fragment predicates? A NL description of logic constraints?)

# Relational Querying on Unstructured Data
## i.e., LLM-driven data, relational query processing
- Traditional knock on relational data is simply that most information isn't relational
- Problem solved! Any data can be featurized into tuples/vectors with virtual semantic "attributes"
    - NL text, Images, Video, Motion Trajectories, etc.
    - Can be done with cheap classifiers or with an LLM
- In principle can run SQL on whatever virtual atrributes we choose
    - E.g. "how many video clips in Alameda County between midnight and 5AM show a car turning left near a pedestrian"
    - Can be an interesting end-solution, i.e. analytics on multimedia
    - Can be used underneath applications for classical data independence reasons
        - E.g. in AI pipelines
        - Separate your data+featurization from your quantitative calculations
- Note: The schema and data are unbounded!
    - Do we close the world and run a query? Or does the query dictate its own schema?
    - Something more iterative/possible-worldsy?
- What's the best way to choose/populate features?
    1. Strawman: Eager ETL for featurization, followed by SQL?
        - probably too expensive
    2. Strawman: Lazy featurization on demand?
    3. Hybrids (ETL the easy features to help decide what to do for (2))?
- Trust: The queries are deterministic SQL, but how do we trustground/bound the featurization?
    4. Role of approximate query processing? The data is noisy after all!
    
# Full Hybrid: Relational and LLM-driven data, relational and LLM query processing

- RAG provides some context here, blending vector search and LLM inference
    - but vector search language is very limited, computationally
- Where do we use what tool?
- In principle, relational lens serves to 
    1. ground/bound the dreaming and
    2. Calculate deterministic things more cheaply than an LLM
- Imagine a langchain pipeline with SQL-like query plans embedded inside
    - Are these optimizable for efficiency/quality tradeoffs?

# Generic efficiency/quality tricks
- HYDE-style "canontextualization" of queries
- Ground: Citations/provenance
- Bound: logic synthesis/checking
- Testing: 
    - Human feedback (e.g. from production)
    - Adversarial tests: perturb data and queries, look for sensitivities
- Cascades: redundant filters using increasingly expensive models with false positives. 
    - Assume $p \subset p_1 \subset p_2 \subset p_3$
    - then $p \approx p_3 \wedge p_2 \wedge p_1$
    - and $p = p_3 \wedge p_2 \wedge p_1 \wedge p$
- Funky data chunking/embedding
    - E.g. chunk documents by structure
    - E.g. chunk tables sorted-columnwise and row-wise
    - E.g. hierarchically chunk to capture multiple granularities
- Sampling and other query approximation schemes

# LLMs as Program Synthesis Engines
- I love this direction: convert dreams into reliable building blocks
- CoPilot + Python is probably a bad example of where this goes
    - Humans have nearly full responsibility to verify correctness
    - CoPilot + Rust engages the AI with rich type system for deterministic correctness checks
    - Even stronger proof assistants would be nice
        - Rust seems too fussy and imperative as an end-state for this research.
- Emphasizes value of declarative spec languages
    - Amenable to stronger proof techniques
    - Is Text-to-SQL better than Text-to-Python? Unclear to me. But maybe that's just a problem of SQL per se.
    - Admits other modes of debugging: e.g. example traces or data provenance examples
- A new generation of *read-mostly* programming languages?
    - Checkable, yes
    - New emphasis on *rendering* those languages for human review
        - What is a good computer language to *read*? See examples of Regex rendering in Predictive Interaction paper
    - Seems that legacy languages are not necessarily an albatross for LLMs, so we're not stuck with Python


# ML for Systems stuff
- Many classic systems heuristics lose to instance-trained model
    - E.g. cache eviction policies, process scheduling, etc.
    - These were always kind of primitive and mediocre.
    - Yawn.
- Query Optimization is a more interesting playpen
    - Hard, heuristic selectivity estimation can be improved with better models
    - Exponential, exhaustive DP search can be replaced with heuristic RL
    - Maybe end-to-end approaches? (Im not yet convinced)
- Core data structures or algorithms can be instance optimized, but should they?
    - Learned indexes? 
        - Seems a bit silly for B+-trees.
        - Maybe for multidimensional indexes (which are heuristics that have exponential worst-cases)
    - If we've already got poly-time human algorithms, do learned structures really help? Or maybe we can learn to tune our deterministic structures!
- Clearly this space has some use, but
    1. Probably best pursued in industry on real workloads where low %-age wins are verifiable and worth the effort
        - I don't trust research papers with low %-age wins
    2. Seems unlikely to supplant algorithm/architecture design
    3. What might we learn from these efforts? Can we translate back into engineering?
    4. Keep an eye on predictability. As we've learned with Query Optimizers, users often hate regression of any kind.

# Is this worth your time?
- There are a million tweaks in this space, and each one will get a paper
    - Undifferentiated sea of student papers
- Industry may have better data sets
    - Although maybe only for the common knowledge part, so can build on top?
    - Query/Usage corpora are the *real* advantage
- One option: identify new Computer Science problems that LLMs open up
    - Anything related to efficiently/correctly Grounding and Bounding results
    - Then perhaps reflect that back into more efficient interaction with LLMs
        - "correctness pushdown" into the LLM
    - Probably best explored in a "vertical" application domain
        - E.g. RAG, Program Synthesis, Query Answering, etc.
    - Like *all of CS*, I think the HCI/UX aspects are very important.
- Is the 19th century "Great Man Theory of History" out of date in AI research?
    - Maybe it's more like experimental physics?
    - Maybe that's OK?
    - I'd still encourage you to try and find Big Problems to solve

# In sum
- I don't believe LLMs will replace CS any time soon
    - If anything, they accelerate the need for new CS research that the AI community is not addressing
- End-User issues in data tools (wrangling, etc) are a great Petri Dish for this area
- Core DB + LLM question-answering is a very rich research space
    - Certainly a DB renaissance: expands the scope of relational querying to all data
        - Including common knowledge!

# Final Thoughts
