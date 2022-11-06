[link](https://nightlies.apache.org/flink/flink-docs-master/docs/concepts/time/)

Timely stream processing is an extension of [[Stateful Stream Processing]] in which time plays some role in the computation. Among other things, this is the case when you do time series analysis, when doing aggregations based on certain time periods (typically called windows), or when you do event processing where the time when an event occurred is important.

## Notions of Time
#### Processing Time
Processing time refers to the system time of the machine that is executing the respective operation.
Processing time is the simplest notion of time and requires no coordination between streams and machines. It provides the best performance and the lowest latency. However, in distributed and asynchronous environments processing time does not provide determinism, because it is susceptible to the speed at which records arrive in the system (for example from the message queue), to the speed at which the records flow between operators inside the system, and to outages (scheduled, or otherwise).

#### Event Time
Event time is the time that each individual event occurred on its producing device. This time is typically embedded within the records before they enter Flink, and that _event timestamp_ can be extracted from each record. In event time, the progress of time depends on the data, not on any wall clocks. Event time programs must specify how to generate _Event Time Watermarks_, which is the mechanism that signals progress in event time.

### Event Time and Watermarks
_have a look at the articles below._
-   [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) by Tyler Akidau
-   The [Dataflow Model paper](https://research.google.com/pubs/archive/43864.pdf)

A stream processor that supports _event time_ needs a way to measure the progress of event time. For example, a window operator that builds hourly windows needs to be notified when event time has passed beyond the end of an hour, so that the operator can close the window in progress.

The mechanism in Flink to measure progress in event time is **watermarks**. Watermarks flow as part of the data stream and carry a timestamp _t_. A _Watermark(t)_ declares that event time has reached time _t_ in that stream, meaning that there should be no more elements from the stream with a timestamp _t’ <= t_ (i.e. events with timestamps older or equal to the watermark).

### Watermarks in Parallel Streams
As the watermarks flow through the streaming program, they advance the event time at the operators where they arrive. Whenever an operator advances its event time, it generates a new watermark downstream for its successor operators.
Some operators consume multiple input streams; a union, for example, or operators following a _keyBy(…)_ or _partition(…)_ function. Such an operator’s current event time is the minimum of its input streams’ event times. As its input streams update their event times, so does the operator.

## Lateness
It is possible that certain elements will violate the watermark condition, meaning that even after the _Watermark(t)_ has occurred, more elements with timestamp _t’ <= t_ will occur. In fact, in many real world setups, certain elements can be arbitrarily delayed, making it impossible to specify a time by which all elements of a certain event timestamp will have occurred. Furthermore, even if the lateness can be bounded, delaying the watermarks by too much is often not desirable, because it causes too much delay in the evaluation of event time windows.

For this reason, streaming programs may explicitly expect some _late_ elements. Late elements are elements that arrive after the system’s event time clock (as signaled by the watermarks) has already passed the time of the late element’s timestamp. See [Allowed Lateness](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/operators/windows/#allowed-lateness) for more information on how to work with late elements in event time windows.

## Windowing
Aggregating events (e.g., counts, sums) works differently on streams than in batch processing. For example, it is impossible to count all elements in a stream, because streams are in general infinite (unbounded). Instead, aggregates on streams (counts, sums, etc), are scoped by **windows**, such as _“count over the last 5 minutes”_, or _“sum of the last 100 elements”_.

Windows can be _time driven_ (example: every 30 seconds) or _data driven_ (example: every 100 elements). One typically distinguishes different types of windows, such as _tumbling windows_ (no overlap), _sliding windows_ (with overlap), and _session windows_ (punctuated by a gap of inactivity).