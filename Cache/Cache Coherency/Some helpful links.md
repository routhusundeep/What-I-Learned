
[This](https://stackoverflow.com/questions/20316124/does-it-make-any-sense-to-use-the-lfence-instruction-on-x86-x86-64-processors) explains LFENCE, SFENCE and MFENCE operations in x86, Intels Specifications are also mentioned.

[This](https://stackoverflow.com/questions/42746793/does-a-memory-barrier-ensure-that-the-cache-coherence-has-been-completed) has very explanation to store buffer and the necessity of fence operations, role of MESI/MOESI/MESIF, differentiates Cache Coherency, Global Visibility and Memory ordering.

[This](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/) talks about all types of barriers with an analogy (which is a bit too simplistic)

[This](https://preshing.com/20120930/weak-vs-strong-memory-models/) give a brief description of the weak and strong memory models.
Sequential consistency is also touched.

[Myths Programmers Believe about CPU Caches](https://software.rajivprab.com/2018/04/29/myths-programmers-believe-about-cpu-caches/)
many of the concepts learnt in cache-coherency are directly applicable to distributed-system-architecture and database-isolation-levels as well. For instance, understanding how coherency is implemented in hardware caches, can help in better understanding strong-vs-eventual consistency

#### Being Coherent
If different cores each have their own private cache, storing copies of the same data, wouldn’t that naturally lead to data mismatches as they start issuing writes? 
The answer: hardware caches on modern x86 CPUs like Intel’s, are kept in-sync with one another. These caches aren’t just dumb memory storage units, as many developers seem to think. Rather, there are very intricate protocols and logics, embedded in every cache, communicating with other caches, enforcing coherency across all threads.