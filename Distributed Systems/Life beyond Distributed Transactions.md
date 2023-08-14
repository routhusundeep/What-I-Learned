[paper](https://github.com/papers-we-love/papers-we-love/blob/main/distributed_systems/life-beyond-distributed-transactions-an-apostates-opinion.pdf)

This is a position paper, and most of the ideas are well-founded and insightful. The pith being, distributed transactions are not needed or generally avoided in almost all highly scalable systems, even when needed, they do not span across entities which are scale agnostic application level objects.

I feel that paper has some relation to domain driven design, especially with entity being a foundational object for an application and messages are delivered to and from these entities. These entities are disjoint sets of data and therefore will be treated as an abstraction which can be acted upon depending on the messages. The abstraction of the entity represents both the business logic and the durable data encapsulated by it.

The crux of the paper is:
> Atomic Transactions Cannot Span Entities

This is a very felicitous phrase and I can relate with it. In my experience, If you ever need transaction across entities, it is almost always due to ill-designed entities and their nebulous boundaries, one such example is, the entity may span across multiple microservices with each application owning part of the business logic, then all these applications should coordinate with each other to keep the entity in a consistent state. Normally, such scenarios can be reoriented by splitting the entities into multiple atomic sub-entities.

One more assumption which acts as the source of headache for budding architects is:
>  Most Applications Use “At-Least-Once” Messaging

This means that the recipient entity must be prepared in its durable state to be assailed with redundant messages that must be ignored. 
Idempotency is the most unchallenging tool in the arsenal to accommodate these pestering redundant messages.

**It is a very easy read, suggest reading the whole paper.**