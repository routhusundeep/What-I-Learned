[paper](https://arxiv.org/abs/1901.01930#:~:text=CALM%20is%20an%20acronym%20for,be%20expressed%20in%20monotonic%20logic.) #distributed_systems 

We consider two nearly identical classical distributed systems problems involving graph reachability—one coordination-free, one not.
* Distributed Deadlock Detection: does not need coordination
* Distributed Garbage Collection: needs coordination
Easy to see why.

#### The Crux of Consistency: Monotonicity
> Question: What is the family of problems that can be consistently computed in a distributed fashion without coordination, and what problems lie outside that family?

The set of satisfying paths that exist is monotonic in the information received:
> Definition 1. A program P is monotonic if for any input sets S,T where S ⊆ T , P(S) ⊆ P(T ).

Monotonicity is the key property underlying the need for coordination to establish consistency, as captured in the CALM Theorem: 
>Theorem 1. Consistency As Logical Monotonicity (CALM). A program has a consistent, coordination-free distributed implementation if and only if it is monotonic.

Intuitively, monotonic programs are “safe” in the face of missing information, and can proceed without coordination. Non-monotonic programs, by contrast, must be concerned that truth of a property could change in the face of new information. Therefore they cannot proceed until they know all information has arrived, requiring them to coordinate. 
Additionally, because they “change their mind”, non-monotonic programs are order-sensitive: the order in which they receive information determines how they toggle state back and forth, which in turn determines their final state. By contrast, monotonic programs simply accumulate beliefs; their output depends only on the content of their input, not the order in which is arrives.

### CALM: A PROOF SKETCH
#### Program Consistency: Confluence
> A practical consistency question is this: “Does my program produce deterministic outcomes despite non-determinism in the runtime system?”

In the context of nondeterministic message delivery, an operation on a single machine is confluent if it produces the same set of outputs for any nondeterministic ordering and batching of a set of inputs.

The rest of the paper delves into why some design patterns and why CALM is useful. I recommend reading the paper if you are interested.