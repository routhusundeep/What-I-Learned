[paper](file:///Users/sundeep.routhu/Downloads/brewers-conjecture.pdf)

I don't know why I was afraid of this paper till now. Nancy Lynch is a genius.
**Read the paper for the proofs**, I will only summarize them.


### Formal Model
### Atomic Data Objects
Under this consistency guarantee, there must exist a total order on all operations such that each operation looks as if it were completed at a single instant

### Available Data Objects
For a distributed system to be continuously available, every request received by a non-failing node in the system must result in a response. That is, any algorithm used by the service must eventually terminate.

#### Partition Tolerance
In order to model partition tolerance, the network will be allowed to lose arbitrarily many messages sent from one node to another. When a network is partitioned, all messages sent from nodes in one component of the partition to nodes in another component are lost.



### Asynchronous Networks
**Theorem 1:** It is impossible in the asynchronous network model to implement a read/write data object that guarantees the following properties: 
* Availability
* Atomic consistency in all fair executions (including those in which messages are lost)

**Corollary 1.1:** It is impossible in the asynchronous network model to implement a read/write data object that guarantees the following properties: 
* Availability, in all fair executions, 
* Atomic consistency, in fair executions in which no messages are lost

### Solutions in the Asynchronous Model 
While it is impossible to provide all three properties: atomicity, availability, and partition tolerance, any two of these three properties can be achieved.

### Partially Synchronous Networks
a partially synchronous model in which every node has a clock, and all clocks increase at the same rate. However, the clocks themselves are not synchronized, in that they may display different values at the same real time. In effect, the clocks act as timers: local state variables that the processes can observe to measure how much time has passed. A local timer can be used to schedule an action to occur a certain interval of time after some other event. Furthermore, assume that every message is either delivered within a given, known time: ${t_msg}$, or it is lost. Also, every node processes a received message within a given, known time: $t_{local}$, and local processing takes zero time.

**Theorem 2:** It is impossible in the partially synchronous network model to implement a read/write data object that guarantees the following properties: 
* Availability
* Atomic consistency in all executions (even those in which messages are lost).

#### Weaker Consistency Conditions
**Definition 3:** A timed execution, α, of a read-write object is Delayed-t Consistent if: 
1. P is a partial order that orders all write operations, and orders all read operations with respect to the write operations. 
2. The value returned by every read operation is exactly the one written by the previous write operation in P (or the initial value, if there is no such previous write in P). 
3. The order in P is consistent with the order of read and write requests submitted at each node. 
4. (Atomicity) If all messages in the execution are delivered, and an operation θ completes before an operation φ begins, then φ does not precede θ in the partial order P, 
5. (Weakly Consistent) Assume there exists an interval of time longer than t in which no messages are lost. Further, assume an operation, θ, completes before the interval begins, and another operation, φ, begins after the interval ends. Then φ does not precede θ in the partial order P.

**Theorem 4:** The modified centralized algorithm is Delayed-t consistent.
