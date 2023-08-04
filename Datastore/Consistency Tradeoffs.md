[paper](http://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf)

A wonderful non-technical paper written to explicate how influential CAP is in the design of the modern Distributed Data Systems when it quite limited in what it states.

Highly recommend reading the whole paper.

> Ignoring the consistency/latency tradeoff of replicated systems is a major oversight, as it is present at all times during system operation.

### PACELC
A more complete portrayal of the space of potential consistency tradeoffs for DDBSs can be achieved by rewriting CAP as PACELC (pronounced “pass-elk”): if there is a partition (P), how does the system trade off availability and consistency (A and C); else (E), when the system is running normally in the absence of partitions, how does the system trade off latency (L) and consistency (C)?

Note that the latency/consistency tradeoff (ELC) only applies to systems that replicate data. Otherwise, the system suffers from availability issues upon any type of failure or overloaded node. Because such issues are just instances of extreme latency, the latency part of the ELC tradeoff can incorporate the choice of whether or not to replicate data

