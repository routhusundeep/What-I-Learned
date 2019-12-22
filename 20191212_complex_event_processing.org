#+TITLE: Daily Notes Thursday, 12/12/2019
** [[https://people.cs.umass.edu/~yanlei/publications/sase-sigmod08.pdf][Efficient pattern matching over event streams]]   :complex_event_processing:
The paper presents a formal evaluation model that offers precise semantics for this new class of queries and an evaluation framework permitting optimizations in a principled fashion.
Challenges faced when pattern matching over streams are
1. Richer Language - We require a significantly richer language than standard regular expressions to represent our queries. Some constructs necessary are sequencing notion, Kleene Closure, complex predicates, negation etc
2. Efficiency - new algorithms and optimizations are required for efficient pattern matching
The proposed query evaluation framework has three principles
1. Formal Evaluation Model - supports a full set of pattern queries, A NFA(b) model is proposed which combines a finite state automaton with a match buffer
2. Runtime Complexity Analysis - using the proposed model, they identify the key issue in runtime evaluation, the different times of non-determinism, to develop a good intuition for the worst case scenarios
3. Runtime Algorithms and Optimizations - New Data Structures and Algorithms are developed
A few examples of queries in different language framework are explained, Please read those to get an understanding about the event matching domain.
- Event selection Strategy
  - how to select relevant events from an input stream which typically is a mixture of relevant and irrelevant events
  - Strict Contiguity
    - The relevant events are contiguous in the stream
  - Partition Contiguity
    - The relevant events are contiguous with respect to a partition strategy
    - Strict can be treated as a single partition
  - Skip till next match
    - all irrelevant events can be skipped
  - Skip till any match
    - relaxes the previous one by further allowing non-deterministic actions on relevant events
    - when a new relevant event comes
      - it may skip it and maintain the current state
      - it may include this event too to the current state
- output format
  - default format
    - return all matches of a pattern
  - non-overlapping format
    - only one match among those that belong to the same partition and overlap in time

*** Formal Semantic Model
Please read the model in the paper, I am not briefing it here because latex support is abysmal in Git markdown, maybe it is not intended to work.   
More details about the run time optimizations are also given in the paper, which I am ignoring since it wont make sense without the model.

