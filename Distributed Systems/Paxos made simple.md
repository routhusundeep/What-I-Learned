[paper](https://github.com/papers-we-love/papers-we-love/blob/main/distributed_systems/paxos-made-simple.pdf)

I was so excited to finally learn this. Leslie Lamport is otherworldly.
Highly recommend reading [understanding paxos](https://understandingpaxos.wordpress.com/) too to get a better intuition.

Read the paper to see the proofs, I will only be taking notes on theorems and corollaries.

### The Consensus Algorithm
#### The Problem
Assume a collection of processes that can propose values. A consensus algorithm ensures that a single one among the proposed values is chosen. If no value is proposed, then no value should be chosen. If a value has been chosen, then processes should be able to learn the chosen value. The safety requirements for consensus are: 
* Only a value that has been proposed may be chosen,
* Only a single value is chosen, and 
* A process never learns that a value has been chosen unless it actually has been.

Assume that agents can communicate with one another by sending messages. We use the customary asynchronous, non-Byzantine model, in which:
* Agents operate at arbitrary speed, may fail by stopping, and may restart. Since all agents may fail after a value is chosen and then restart, a solution is impossible unless some information can be remembered by an agent that has failed and restarted. 
* Messages can take arbitrarily long to be delivered, can be duplicated, and can be lost, but they are not corrupted.

#### Choosing a value

> $P1$. An acceptor must accept the first proposal that it receives.

This is not sufficient.

> $P2$. If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.

Not strong enough yet.

>$P2_a$. If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v.

Some more.

> $P2_b$. If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v.

Waiting....

> $P2_c$. For any v and n, if a proposal with value v and number n is issued, then there is a set S consisting of a majority of acceptors such that either (a) no acceptor in S has accepted any proposal numbered less than n, or (b) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the acceptors in S.

There it is!!!

This leads to the following algorithm for issuing proposals. 
1. A proposer chooses a new proposal number n and sends a request to each member of some set of acceptors, asking it to respond with: 
	1.  A promise never again to accept a proposal numbered less than n, and 
	2. The proposal with the highest number less than n that it has accepted, if any. I will call such a request a prepare request with number n. 
2. If the proposer receives the requested responses from a majority of the acceptors, then it can issue a proposal with number n and value v, where v is the value of the highest-numbered proposal among the responses, or is any value selected by the proposer if the responders reported no proposals.

What about an acceptor? It can respond to an accept request, accepting the proposal, iff it has not promised not to. In other words:
> $P1_a$. An acceptor can accept a proposal numbered n iff it has not responded to a prepare request having a number greater than n.

$P1_a$ subsumes $P1$

Putting the actions of the proposer and acceptor together, we see that the algorithm operates in the following two phases. 
**Phase 1.** 
1. A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors. 
2. If an acceptor receives a prepare request with number n greater than that of any prepare request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered proposal (if any) that it has accepted.
**Phase 2.** 
1. If the proposer receives a response to its prepare requests (numbered n) from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered n with a value v, where v is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals. 
2. If an acceptor receives an accept request for a proposal numbered n, it accepts the proposal unless it has already responded to a prepare request having a number greater than n.

### Learning a Chosen Value
We can have the acceptors respond with their acceptances to a distinguished learner, which in turn informs the other learners when a value has been chosen.
More generally, the acceptors could respond with their acceptances to some set of distinguished learners, each of which can then inform all the learners when a value has been chosen.

### Progress
To guarantee progress, a distinguished proposer must be selected as the only one to try issuing proposals. If the distinguished proposer can communicate successfully with a majority of acceptors, and if it uses a proposal with number greater than any already used, then it will succeed in issuing a proposal that is accepted

