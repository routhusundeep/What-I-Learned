[paper](https://github.com/papers-we-love/papers-we-love/blob/main/distributed_systems/dapper-a-large-scale-distributed-tracing-infrastructure.pdf) #distributed_systems #real_world_system 

Two fundamental requirements for Dapper: ubiquitous deployment, and continuous monitoring.
Three concrete design goals result from these requirements:
* Low overhead
* Application-level transparency
* Scalability
An additional design goal is for tracing data to be available for analysis quickly after it is generated: ideally within a minute.

### Distributed Tracing in Dapper
Two classes of solutions have been proposed to aggregate information so that one can associate all record entries with a given initiator.
* Black-box schemes assume there is no additional information other than the message record, and use statistical regression techniques to infer that association. 
* Annotation-based schemes rely on applications or middleware to explicitly tag every record with a global identifier that links these message records back to the originating request.

#### Trace trees and spans
In a Dapper trace tree, the tree nodes are basic units of work which we refer to as spans. The edges indicate a casual relationship between a span and its parent span. Independent of its place in a larger trace tree, though, a span is also a simple log of timestamped records which encode the span’s start and end time, any RPC timing data, and zero or more application-specific annotations

#### Instrumentation points
* When a thread handles a traced control path, Dapper attaches a trace context to thread-local storage
* Most Google developers use a common control flow library to construct callbacks and schedule them in a thread pool or other executor. Dapper ensures that all such callbacks store the trace context of their creator
* Nearly all of Google’s inter-process communication is built around a single RPC framework with bindings in both C++ and Java

#### Annotations
Dapper also allows application developers to enrich Dapper traces with additional information that may be useful

#### Sampling
We control overhead by recording only a fraction of all traces

#### Trace collection
First, span data is written to local log files. It is then pulled from all production hosts by Dapper daemons and collection infrastructure and finally written to a cell in one of several regional Dapper Bigtable repositories.

##### Out-of-band trace collection
in-band collection scheme – where trace data is sent back within RPC response headers – can affect application network dynamics.

#### Security and privacy considerations
RPC Payload is not logged in Dapper in consideration of security.

*Various other important topics are also discussed, read it if you are interested.*

### Experiences
* Used during development for Performance (obvious), correctness (the tree is very useful to identify incorrect calls), Understanding and Testing.
* Exception monitoring can be done.
* Addressing long tail latencies of certain applications and estimating its impact on overall latency and throughput.
* Inferring service dependencies
* Network usage at application level.
* Identifying services using a shared storage.
* Firefighting
