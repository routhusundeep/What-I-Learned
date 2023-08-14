[paper](https://dsf.berkeley.edu/papers/UCB-lattice-tr.pdf) #reread 

I will have to read the paper more thoroughly later.

### Introduction
#### Convergent Modules
Shapiro, et al. recently proposed a formalism for these approaches called Conflict-Free Replicated Data Types ([CRDTs](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)), which casts these ideas into the algebraic framework of [semilattices](https://en.wikipedia.org/wiki/Semilattice)

The problems with Convergent Modules present a scope dilemma: a small module makes lattice properties easy to inspect and test, but provides only simple semantic guarantees. Large CRDTs provide higher-level application guarantees but require the programmer to ensure lattice properties hold for a large module, resulting in software that is difficult to test, maintain, and trust.

#### Monotonic Logic
Intuitively, a monotonic program makes forward progress over time: it never “retracts” an earlier conclusion in the face of new information. We proposed the CALM theorem, which established that all monotonic programs are confluent (invariant to message reordering and retry) and hence eventually consistent

The CALM theorem obviates any scoping concerns for convergent monotonic logic, but it presents a type dilemma. Sets are the only data type amenable to CALM analysis, but the programmer may have a more natural representation of a monotonically growing phenomenon.

### $Bloom^L$ : Logic and Lattices
Provides three main improvements in the state of the art of both Bloom and CRDTs:
1. It solves the type dilemma of logic programming: CALM analysis in $Bloom^L$ is able to assess monotonicity for arbitrary lattices, making it significantly more liberal in its ability to test for confluence. It can validate the coordination-free use of common constructs like timestamps and sequence numbers.
2. BloomL solves the scope dilemma of CRDTs by providing monotonicity-preserving mappings between lattices via morphisms and monotone functions,
3. For efficient incremental execution, we extend the standard Datalog semi-naive evaluation scheme to support arbitrary lattices

### Adding Lattices to Bloom
#### Definitions
A bounded join semilattice is a triple $\{S, \sqcup, \perp\}$, where $S$ is a set, $\sqcup$ is a binary operator (called “join” or “least upper bound”), and $\perp\in S$. The operator $\sqcup$ is associative, commutative, and idempotent. The $\sqcup$ operator induces a partial order $\leq_S$ on the elements of $S : x \leq_S y$ if $x \sqcup y = y$. Note that although  $\leq_S$ is only a partial order, the least upper bound is defined for all elements $x, y \in S$. The distinguished element $\perp$ is the smallest element in $S$ : $x \sqcup \perp = x$ for every $x \in S$.

A monotone function from [poset](https://en.wikipedia.org/wiki/Partially_ordered_set) $S$ to poset $T$ is a function $f : S \to T$ such that $\forall a, b \in S : a \leq_S b ⇒ f(a) \leq_T f(b)$. That is, $f$ maps elements of $S$ to elements of $T$ in a manner that respects the partial orders of both posets.

A morphism from lattice $\{X, \sqcup_X, \perp_X\}$ to lattice $\{Y, \sqcup_Y, \perp_Y\}$ i is a function $g : X \to Y$ such that, $\forall a, b \in X : g(a \sqcup_X b) = g(a) \sqcup_Y g(b)$. That is, $g$ allows elements of $X $to be mapped to elements of $Y$ in a way that preserves the lattice properties. Note that morphisms are monotone functions but the converse is not true in general.


**Recommend reading the paper for the language details like type and their operations. The implementation is also briefly discussed.**

