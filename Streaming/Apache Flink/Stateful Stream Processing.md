[link](https://nightlies.apache.org/flink/flink-docs-master/docs/concepts/stateful-stream-processing/)
Flink needs to be aware of the state in order to make it fault tolerant using [checkpoints](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/checkpointing/) and [savepoints](https://nightlies.apache.org/flink/flink-docs-master/docs/ops/state/savepoints/).
[Queryable state](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/fault-tolerance/queryable_state/) allows you to access state from outside of Flink during runtime.

### Keyed State
Keyed state is maintained in what can be thought of as an embedded key/value store. The state is partitioned and distributed strictly together with the streams that are read by the stateful operators. Hence, access to the key/value state is only possible on _keyed streams_, i.e. after a keyed/partitioned data exchange, and is restricted to the values associated with the current event’s key.
Aligning the keys of streams and state makes sure that all state updates are local operations, guaranteeing consistency without transaction overhead. This alignment also allows Flink to redistribute the state and adjust the stream partitioning transparently.

### State Persistence
Flink implements fault tolerance using a combination of **stream replay** and **checkpointing**. A checkpoint marks a specific point in each of the input streams along with the corresponding state for each of the operators. A streaming dataflow can be resumed from a checkpoint while maintaining consistency _(exactly-once processing semantics)_ by restoring the state of the operators and replaying the records from the point of the checkpoint.
In case of a program failure (due to machine-, network-, or software failure), Flink stops the distributed streaming dataflow. The system then restarts the operators and resets them to the latest successful checkpoint. The input streams are reset to the point of the state snapshot. Any records that are processed as part of the restarted parallel dataflow are guaranteed to not have affected the previously checkpointed state.

### Checkpointing
It is described in this [paper](https://arxiv.org/pdf/1506.08603.pdf) and inspired from [chandy-lamport algorithm](https://www.microsoft.com/en-us/research/uploads/prod/2016/12/Determining-Global-States-of-a-Distributed-System.pdf)

#### Barriers
A core element in Flink’s distributed snapshotting are the _stream barriers_. These barriers are injected into the data stream and flow with the records as part of the data stream. Barriers never overtake records, they flow strictly in line. A barrier separates the records in the data stream into the set of records that goes into the current snapshot, and the records that go into the next snapshot. Each barrier carries the ID of the snapshot whose records it pushed in front of it. Barriers do not interrupt the flow of the stream and are hence very lightweight. Multiple barriers from different snapshots can be in the stream at the same time, which means that various snapshots may happen concurrently.

Operators that receive more than one input stream need to _align_ the input streams on the snapshot barriers.
-   As soon as the operator receives snapshot barrier _n_ from an incoming stream, it cannot process any further records from that stream until it has received the barrier _n_ from the other inputs as well. Otherwise, it would mix records that belong to snapshot _n_ and with records that belong to snapshot _n+1_.
-   Once the last stream has received barrier _n_, the operator emits all pending outgoing records, and then emits snapshot _n_ barriers itself.
-   It snapshots the state and resumes processing records from all input streams, processing records from the input buffers before processing the records from the streams.
-   Finally, the operator writes the state asynchronously to the state backend.

#### Snapshotting Operator State
Operators snapshot their state at the point in time when they have received all snapshot barriers from their input streams, and before emitting the barriers to their output streams.

### Unaligned Checkpointing 
How an operator handles unaligned checkpoint barriers:
-   The operator reacts on the first barrier that is stored in its input buffers.
-   It immediately forwards the barrier to the downstream operator by adding it to the end of the output buffers.
-   The operator marks all overtaken records to be stored asynchronously and creates a snapshot of its own state.
Unaligned checkpointing ensures that barriers are arriving at the sink as fast as possible. It’s especially suited for applications with at least one slow moving data path, where alignment times can reach hours. However, since it’s adding additional I/O pressure, it doesn’t help when the I/O to the state backends is the bottleneck.

#### Unaligned Recovery
Operators first recover the in-flight data before starting processing any data from upstream operators in unaligned checkpointing. Aside from that, it performs the same steps as during [recovery of aligned checkpoints](https://nightlies.apache.org/flink/flink-docs-master/docs/concepts/stateful-stream-processing/#recovery).


### State Backends
The exact data structures in which the key/values indexes are stored depends on the chosen [state backend](https://nightlies.apache.org/flink/flink-docs-master/docs/ops/state/state_backends/). One state backend stores data in an in-memory hash map, another state backend uses [RocksDB](http://rocksdb.org/) as the key/value store. In addition to defining the data structure that holds the state, the state backends also implement the logic to take a point-in-time snapshot of the key/value state and store that snapshot as part of a checkpoint. State backends can be configured without changing your application logic.

### Savepoints
All programs that use checkpointing can resume execution from a **savepoint**. Savepoints allow both updating your programs and your Flink cluster without losing any state.
[Savepoints](https://nightlies.apache.org/flink/flink-docs-master/docs/ops/state/savepoints/) are **manually triggered checkpoints**, which take a snapshot of the program and write it out to a state backend. They rely on the regular checkpointing mechanism for this.
Savepoints are similar to checkpoints except that they are **triggered by the user** and **don’t automatically expire** when newer checkpoints are completed.

### Exactly Once vs. At Least Once
The alignment step may add latency to the streaming program. Usually, this extra latency is on the order of a few milliseconds, but we have seen cases where the latency of some outliers increased noticeably. For applications that require consistently super low latencies (few milliseconds) for all records, Flink has a switch to skip the stream alignment during a checkpoint. Checkpoint snapshots are still drawn as soon as an operator has seen the checkpoint barrier from each input.

When the alignment is skipped, an operator keeps processing all inputs, even after some checkpoint barriers for checkpoint _n_ arrived. That way, the operator also processes elements that belong to checkpoint _n+1_ before the state snapshot for checkpoint _n_ was taken. On a restore, these records will occur as duplicates, because they are both included in the state snapshot of checkpoint _n_, and will be replayed as part of the data after checkpoint _n_.