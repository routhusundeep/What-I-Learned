#langsec #design_pattern 
[paper](http://langsec.org/papers/langsec-cwes-secdev2016.pdf)

Input-handling bugs share two common patterns: insufficient recognition, where input-checking logic is unfit to validate a program’s assumptions about inputs, and parser differentials, wherein two or more components of a system fail to interpret input equivalently

**Input-driven exploitation**
invalid input is processed instead of being rejected.

What are the properties of input  that need to be checked and can be relied upon? What coherent sets of such properties can scale up to be implemented correctly by large groups of programmers? To what extent are the pitfalls properties of the input specifications themselves?  
The LangSec methodology seeks to answer these questions

**Langsec**
Language-theoretic security (LangSec) is the idea that many security issues can be avoided by applying a standard process to input processing and protocol design: the acceptable input to a program should be well-defined (i.e., via a grammar), as simple as possible (on the Chomsky scale of syntactic complexity), and fully validated before use (by a dedicated parser of appropriate but not excessive power in the Chomsky hierarchy of automata).

### Anti-patterns
- **Shotgun Parsing**
Shotgun parsing is a programming antipattern whereby parsing and input-validating code is mixed with and spread across processing code—throwing a cloud of checks at the input, and hoping, without any systematic justification, that one or another would catch all the “bad” cases.

- **Non-Minimalist Input-handling Code**
Input-handling code should be minimalist in computing power. A regular language should be handled by a finite automaton implementa- tion, not by a pushdown one, nor by a more powerful model.

- **Input Language More Complex than Deterministic Context-Free**
We recommend not letting language complexity go above deterministic context-free (DCF) first and foremost because of the issue of parser equivalence.

- **Differing Interpretations of Input Language**
Whether or not an input language is complex, different programs, different implementations of the same input language, and even different components of the same program in the same runtime context can interpret input differently, both from each other and from the specification.

- **Incomplete Protocol Specification**
Attempting to write equivalent parsers is of course impossible if the language itself is ill-defined.

- **Overloaded Field in Input Format**
The reuse of data fields for different purposes can be a good indication of ad-hoc constructions or hasty additions—an obvious road to complexity and mistakes.

- **Permissive Processing of Invalid Input**
The traditional “robustness principle” dictates that one should “be liberal in what you accept” . After leading developers to implement vulnerable programs for decades, this principle has attracted considerable discussion. We argue that one should not be liberal, but definite—or explicit—about what is accepted.

### Remidies
- **Completely separate input validation from application  logic**
Avoid writing shotgun parsers. Input should be fully validated by a machine expressible as a deterministic push-down automaton.
Parsing of a basic interchange format into an internal  representation is sometimes referred to as canonicalization.

- **Minimize complexity of pre-validation code**
In general, the code responsible for input canonicalization and validation should be constrained to just those functions. This is not the place to introduce application logic.

- **Avoid defining complex input languages**
In essence, the most preferable language is one that can be fully validated by a regular expression. Programmers should try to use such languages, but of course this is not always possible, as these languages do not support recursive nesting of data structures

- **Be clear about specifications**
The practice of making and following clear specifications will remedy both the differing interpretations of input language and incomplete protocol specification problems

- **Avoid overloading fields**
Do not use special values in fields to have special meanings

- **Do not transparently correct for invalid input**
If input does not validate correctly, either because it cannot be canonicalized, required entities are missing, or illegal entities

