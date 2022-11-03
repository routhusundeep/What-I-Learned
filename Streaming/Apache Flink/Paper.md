#distributed_systems #stream_processing #batch_processing

[paper](https://kth.diva-portal.org/smash/get/diva2:1059537/FULLTEXT01.pdf)

Flink is built on the philosophy that many classes of data processing applications, including real-time analytics, continuous data pipelines, historic data processing(batch) and iterative algorithms(machine learning and graph analytics) can be expressed and executed as pipelined fault-tolerant dataflows.

Apache Flink follows a paradigm that embraces data-stream processing as the unifying model for real-time analysis, continuous streams, and batch processing both in the programming model and in the execution engine. In combination with durable message queues that allow quasi-arbitrary replay of data streams (like Apache Kafka or Amazon Kinesis), stream processing programs make no distinction between processing the latest events in real-time, continuously aggregating data periodically in large windows, or processing terabytes of historical data


The contributions of this paper are as follows: 
- we make the case for a unified architecture of stream and batch data processing, including specific optimizations that are only relevant for static data sets
- we show how streaming, batch, iterative, and interactive analytics can be represented as fault-tolerant streaming dataflows
- we discuss how we can build a full-fledged stream analytics system with a flexible windowing mechanism
- as well as a full-fledged batch processor on top of these dataflows, by showing how streaming, batch, iterative, and interactive analytics can be represented as streaming dataflows.

### System Architecture
The core of Flink is the distributed dataflow engine, which executes dataflow programs. A Flink runtime program is a DAG of stateful operators connected with data streams. There are two core APIs in Flink: the DataSet API for processing finite data sets, and the DataStream API for processing potentially unbounded data streams. Flink’s core runtime engine can be seen as a streaming dataflow engine, and both the DataSet and DataStream APIs create runtime programs executable by the engine.

A Flink cluster comprises three types of processes: the client, the Job Manager, and at least one Task Manager.
- **Client**:
	- The client takes the program code, transforms it to a dataflow graph, and submits that to the JobManager. 
	- This transformation phase also examines the data types (schema) of the data exchanged between operators and creates serializers and other type/schema specific code.
	- DataSet programs additionally go through a cost-based query optimization phase
- **Job Manager**:
	- It tracks the state and progress of each operator and stream, schedules new operators, and coordinates checkpoints and recovery. 
	- In a high-availability setup, the JobManager persists a minimal set of metadata at each checkpoint to a fault-tolerant storage, such that a standby JobManager can reconstruct the checkpoint and recover the dataflow execution from there
- **Task Manager**:
	- A TaskManager executes one or more operators that produce streams, and reports on their status to the JobManager. 
	- The TaskManagers maintain the buffer pools to buffer or materialize the streams, and the network connections to exchange the data streams between operators

### The Common Fabric: Streaming Dataflows

#### Dataflow Graphs
The dataflow graph is a directed acyclic graph (DAG) that consists of:
- stateful operators and
- data streams that represent data produced by an operator and are available for consumption by operators.

#### Data Exchange through Intermediate Data Streams
An intermediate data stream represents a logical handle to the data that is produced by an operator and can be consumed by one or more operators. Intermediate streams are logical in the sense that the data they point to may or may not be materialized on disk. The particular behavior of a data stream is parameterized by the higher layers in Flink

##### Pipelined and Blocking Data Exchange
- **Pipelined**:
	- intermediate streams exchange data between concurrently running producers and consumers resulting in pipelined execution.
	- propagate back pressure from consumers to producers, modulo some elasticity via intermediate buffer pools, in order to compensate for short-term throughput fluctuations
- **Blocking**:
	- buffers all of the producing operator’s data before making it available for consumption, thereby separating the producing and consuming operators into different execution stages.
	- They are used to isolate successive operators against each other (where desired) and in situations where plans with pipeline-breaking operators, such as sort-merge joins may cause distributed deadlocks.

##### Balancing Latency and Throughput
Flink’s data-exchange mechanisms are implemented around the exchange of buffers. When a data record is ready on the producer side, it is serialized and split into one or more buffers that can be forwarded to consumers. A buffer is sent to a consumer either as soon as it is full or when a timeout condition is reached. This enables Flink to achieve high throughput by setting the size of buffers to a high value, as well as low latency by setting the buffer timeout to a low value.

##### Control Events
These are special events injected in the data stream by operators, and are delivered in-order along with all other data records and events within a stream partition. The receiving operators react to these events by performing certain actions upon their arrival. Flink uses lots of special types of control events, including:
- **checkpoint barriers**: that coordinate checkpoints by dividing the stream into pre-checkpoint and post-checkpoint
- **watermarks**: signaling the progress of event-time within a stream partition
- **iteration barriers**: signaling that a stream partition has reached the end of a superstep, in Bulk/StaleSynchronous-Parallel iterative algorithms on top of cyclic dataflows

Streaming dataflows in Flink do not provide ordering guarantees after any form of repartitioning or broadcasting and the responsibility of dealing with out-of-order records is left to the operator implementation

#### Fault Tolerance
Flink offers reliable execution with strict exactly-once-processing consistency guarantees and deals with failures via checkpointing and partial re-execution. The general assumption the system makes to effectively provide these guarantees is that the data sources are persistent and replayable.
To bound recovery time, Flink takes a snapshot of the state of operators, including the current position of the input streams at regular intervals.
The core challenge lies in taking a consistent snapshot of all parallel operators without halting the execution of the topology. In essence, the snapshot of all operators should refer to the same logical time in the computation. The mechanism used in Flink is called Asynchronous Barrier Snapshotting. Barriers are control records injected into the input streams that correspond to a logical time and logically separate the stream to the part whose effects will be included in the current snapshot and the part that will be snapshotted later.
An operator receives barriers from upstream and first performs an alignment phase, making sure that the barriers from all inputs have been received. Then, the operator writes its state to durable storage. Once the state has been backed up, the operator forwards the barrier downstream.

#### Iterative Dataflows
Iterations in Flink are implemented as iteration steps, special operators that themselves can contain an execution graph. To maintain the DAG-based runtime and scheduler, Flink allows for iteration “head” and “tail” tasks that are implicitly connected with feedback edges. The role of these tasks is to establish an active feedback channel to the iteration step and provide coordination for processing data records in transit within this feedback channel.

### Stream Analytics on Top of Dataflows

#### The Notion of Time
Flink distinguishes between two notions of time: event-time, which denotes the time when an event originates and processingtime, which is the wall-clock time of the machine that is processing the data.

In distributed systems, there is an arbitrary skew between event-time and processing-time. This skew may mean arbitrary delays for getting an answer based on event-time semantics. To avoid arbitrary delays, these systems regularly insert special events called low watermarks that mark a global progress measure. In the case of time progress for example, a watermark includes a time attribute t indicating that all events lower than t have already entered an operator. The watermarks aid the execution engine in processing events in the correct event order and serialize operations, such as window computations, via a unified measure of progress.

#### Stateful Stream Processing
In Flink state is made explicit and is incorporated in the API by providing: operator interfaces or annotations to statically register explicit local variables within the scope of an operator and an operator-state abstraction for declaring partitioned key-value states and their associated operations. Users can also configure how the state is stored and checkpointed using the StateBackend abstractions provided by the system, thereby allowing highly flexible custom state management in streaming applications. Flink’s checkpointing mechanism guarantees that any registered state is durable with exactly-once update semantics

#### Stream Windows
Apache Flink incorporates windowing within a stateful operator that is configured via a flexible declaration composed out of three core functions: a window assigner and optionally a trigger and an evictor. All three functions can be selected among a pool of common predefined implementations (e.g., sliding time windows) or can be explicitly defined by the user (i.e., user-defined functions).

#### Asynchronous Stream Iterations

### Batch Analytics on Top of Dataflows
A bounded data set is a special case of an unbounded data stream. Thus, a streaming program that inserts all of its input data in a window can form a batch program and batch processing should be fully covered by Flink’s features that were presented above. However, i) the syntax can be simplified and ii) programs that process bounded data sets are amenable to additional optimizations, more efficient book-keeping for fault-tolerance, and staged scheduling.
Flink approaches batch processing as follows: 
- Batch computations are executed by the same runtime as streaming computations. The runtime executable may be parameterized with blocked data streams to break up large computations into isolated stages that are scheduled successively.
- Periodic snapshotting is turned off when its overhead is high. Instead, fault recovery can be achieved by replaying the lost stream partitions from the latest materialized intermediate stream (possibly the source).
- Blocking operators (e.g., sorts) are simply operator implementations that happen to block until they have consumed their entire input. The runtime is not aware of whether an operator is blocking or not. These operators use managed memory provided by Flink (either on or off the JVM heap) and can spill to disk if their inputs exceed their memory bounds.
- A dedicated DataSet API provides familiar abstractions for batch computations, namely a bounded faulttolerant DataSet data structure and transformations on DataSets (e.g., joins, aggregations, iterations). 
- A query optimization layer transforms a DataSet program into an efficient executable.

#### Query Optimization
Flink’s optimizer enumerates different physical plans based on the concept of interesting properties propagation, using a cost-based approach to choose among multiple physical plans. The cost includes network and disk I/O as well as CPU cost. To overcome the cardinality estimation issues in the presence of UDFs, Flink’s optimizer can use hints that are provided by the programmer.

#### Memory Management
Operations such as sorting and joining operate as much as possible on the binary data directly, keeping the serialization and deserialization overhead at a minimum and partially spilling data to disk when needed. To handle arbitrary objects, Flink uses type inference and custom serialization mechanisms. By keeping the data processing on binary representation and off-heap, Flink manages to reduce the garbage collection overhead, and use cache-efficient and robust algorithms that scale gracefully under memory pressure

