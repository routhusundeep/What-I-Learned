[paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43864.pdf)

## Abstract

We as a field must stop trying to groom unbounded datasets into finite pools of information that eventually become complete, and instead live and breathe under the assumption that we will never know if or when we have seen all of our data, only that new data will arrive, old data may be retracted, and the only way to make this problem tractable is via principled abstractions that allow the practitioner the choice of appropriate tradeoffs along the axes of interest: correctness, latency, and cost.

## Introduction
Problems with existing systems:
- Batch systems such as MapReduce (and its Hadoop variants, including Pig and Hive), FlumeJava, and Spark suffer from the latency problems inherent with collecting all input data into a batch before processing it. 
- For many streaming systems, it is unclear how they would remain fault-tolerant at scale. Those that provide scalability and fault-tolerance fall short on expressiveness or correctness vectors.
- Many lack the ability to provide exactly-once semantics (Storm, Samza, Pulsar), impacting correctness.
- Others simply lack the temporal primitives necessary for windowing2 (Tigon), or provide windowing semantics that are limited to tuple or processing-time-based windows (Spark Streaming, Sonora, Trident).
- Most that provide event-timebased windowing either rely on ordering (SQLStream), or have limited window triggering semantics in event-time mode (Stratosphere/Flink).
- CEDR and Trill are noteworthy in that they not only provide useful triggering semantics via punctuations, but also provide an overall incremental model that is quite similar to the one we propose here; however, their windowing semantics are insufficient to express sessions, and their periodic punctuations are insufficient for some of the use cases
- MillWheel and Spark Streaming are both sufficiently scalable, fault-tolerant, and low-latency to act as reasonable substrates, but lack high-level programming models that make calculating event-time sessions straightforward.
- The only scalable system we are aware of that supports a high-level notion of unaligned windows4 such as sessions is Pulsar, but that system fails to provide correctness, as noted above.
- Lambda Architecture systems can achieve many of the desired requirements, but fail on the simplicity axis on account of having to build and maintain two systems.
- Summingbird ameliorates this implementation complexity by abstracting the underlying batch and streaming systems behind a single interface, but in doing so imposes limitations on the types of computation that can be performed, and still requires double the operational complexity

The conceptual contribution of this paper is a single-unified model which:
- Allows for the calculation of event-time5 ordered results, windowed by features of the data themselves, over an unbounded, unordered data source, with correctness, latency, and cost tunable across a broad spectrum of combinations.
- Decomposes pipeline implementation across four related dimensions, providing clarity, composability, and flexibility: 
	- What results are being computed.
	- Where in event time they are being computed.
	- When in processing time they are materialized.
	- How earlier results relate to later refinements.
- Separates the logical notion of data processing from the underlying physical implementation, allowing the choice of batch, micro-batch, or streaming engine to become one of simply correctness, latency, and cost.

Concretely, this contribution is enabled by the following:
- A windowing model which supports unaligned eventtime windows, and a simple API for their creation and use
- A triggering model that binds the output times of results to runtime characteristics of the pipeline, with a powerful and flexible declarative API for describing desired triggering semantics
- An incremental processing model that integrates retractions and updates into the windowing and triggering models described above
- Scalable implementations of the above atop the MillWheel streaming engine and the FlumeJava batch engine, with an external reimplementation for Google Cloud Dataflow, including an open-source SDK that is runtime-agnostic
- A set of core principles that guided the design of this model
- Brief discussions of our real-world experiences with massive-scale, unbounded, out-of-order data processing at Google that motivated development of this model

### Unbounded/Bounded vs Streaming/Batch
When describing infinite/finite data sets, we prefer the terms unbounded/bounded over streaming/batch, because the latter terms carry with them an implication of the use of a specific type of execution engine. In reality, unbounded datasets have been processed using repeated runs of batch systems since their conception, and well-designed streaming systems are perfectly capable of processing bounded data.

### Windowing
Windowing is effectively always time based; while many systems support tuple-based windowing, this is essentially time-based windowing over a logical time domain where elements in order have successively increasing logical timestamps. Windows may be either aligned, i.e. applied across all the data for the window of time in question, or unaligned, i.e. applied across only specific subsets of the data (e.g. per key) for the given window of time.
- Fixed windows (sometimes called tumbling windows) are defined by a static window size, e.g. hourly windows or daily windows. They are generally aligned, i.e. every window applies across all the data for the corresponding period of time. For the sake of spreading window completion load evenly across time, they are sometimes unaligned by phase shifting the windows for each key by some random value.
- Sliding windows are defined by a window size and slide period, e.g. hourly windows starting every minute. The period may be less than the size, which means the windows may overlap. Sliding windows are also typically aligned
- Sessions are windows that capture some period of activity over a subset of the data, in this case per key. Typically they are defined by a timeout gap. Any events that occur within a span of time less than the timeout are grouped together as a session. Sessions are unaligned windows.

### Time Domains
- Event Time, which is the time at which the event itself actually occurred, i.e. a record of system clock time (for whatever system generated the event) at the time of occurrence.
- Processing Time, which is the time at which an event is observed at any given point during processing within the pipeline, i.e. the current time according to the system clock. Note that we make no assumptions about clock synchronization within a distributed system.

## Dataflow Model

### Core Primitives
The Dataflow SDK has two core transforms that operate on the (key, value) pairs flowing through the system
- ParDo for generic parallel processing. Each input element to be processed (which itself may be a finite collection) is provided to a user-defined function (called a DoFn in Dataflow), which can yield zero or more output elements per input. For example, consider an operation which expands all prefixes of the input key, duplicating the value across them: $$ ParDo((fix, 1), (f it, 2)) \Rightarrow (f, 1),(f i, 1),(f ix, 1),(f, 2),(f i, 2),(f it, 2)$$
- GroupByKey for key-grouping (key, value) pairs $$GroupByKey((f, 1),(f i, 1),(fix, 1),(f, 2),(fi, 2),(fit, 2)) \Rightarrow (f, [1, 2]),(fi, [1, 2]),(ix, [1]),(fit, [2])$$
### Windowing
Systems which support grouping typically redefine their GroupByKey operation to essentially be GroupByKeyAndWindow. Our primary contribution here is support for unaligned windows, for which there are two key insights. The first is that it is simpler to treat all windowing strategies as unaligned from the perspective of the model, and allow underlying implementations to apply optimizations relevant to the aligned cases where applicable. The second is that windowing can be broken apart into two related operations:
- Set $AssignWindows(T datum)$, which assigns the element to zero or more windows. This is essentially the Bucket Operator from Li.
- Set $MergeWindows(Set\ windows)$, which merges windows at grouping time. This allows data driven windows to be constructed over time as data arrive and are grouped together.

#### Window Assignment
From the model’s perspective, window assignment creates a new copy of the element in each of the windows to which it has been assigned.

#### Window Merging
It occurs as part of the $GroupByKeyAndWindow$ operation. Look at the example in the paper to get better understanding.

#### API
```java
PCollection<<KV<String, Integer>> input = IO.read(...);
PCollection<<KV<String, Integer>> output = input.apply(Sum.integersPerKey());

// adding session window to above global window
PCollection<<KV<String, Integer>> input = IO.read(...); 
PCollection<<KV<String, Integer>> output = input
		.apply(Window.into(Sessions.withGapDuration( Duration.standardMinutes(30))))
		.apply(Sum.integersPerKey());
```

### Triggers & Incremental Processing
- We need some way of providing support for tuple and processing-time-based windows, otherwise we have regressed our windowing semantics relative to other systems in existence.
- We need some way of knowing when to emit the results for a window. Since the data are unordered with respect to event time, we require some other signal to tell us when the window is done.

As to window completeness, an initial inclination for solving it might be to use some sort of global event-time progress metric, such as watermarks. However, watermarks themselves have two major shortcomings with respect to correctness:
- They are sometimes too fast, meaning there may be late data that arrives behind the watermark. For many distributed data sources, it is intractable to derive a completely perfect event time watermark, and thus impossible to rely on it solely if we want 100% correctness in our output data.
- They are sometimes too slow. Because they are a global progress metric, the watermark can be held back for the entire pipeline by a single slow datum. As a result, using watermarks as the sole signal for emitting window results is likely to yield higher latency of overall results.

For these reasons, we postulate that watermarks alone are insufficient. A useful insight in addressing the completeness problem is that the Lambda Architecture effectively sidesteps the issue: it does not solve the completeness problem by somehow providing correct answers faster; it simply provides the best low-latency estimate of a result that the streaming pipeline can provide, with the promise of eventual consistency and correctness once the batch pipeline runs.

In a nutshell, triggers are a mechanism for stimulating the production of GroupByKeyAndWindow results in response to internal or external signals. They are complementary to the windowing model, in that they each affect system behaviour along a different axis of time:
- Windowing determines where in event time data are grouped together for processing.
- Triggering determines when in processing time the results of groupings are emitted as panes.

In addition to controlling when results are emitted, the triggers system provides a way to control how multiple panes for the same window relate to each other, via three different refinement modes:
- **Discarding**: Upon triggering, window contents are discarded, and later results bear no relation to previous results.
- **Accumulating**: Upon triggering, window contents are left intact in persistent state, and later results become a refinement of previous results.
- **Accumulating & Retracting**: Upon triggering, in addition to the accumulating semantics, a copy of the emitted value is also stored in persistent state. When the window triggers again in the future, a retraction for the previous value will be emitted first, followed by the new value as a normal datum.

#### Examples
Just read the paper dude!

## IMPLEMENTATION & DESIGN

### Implementation
We have implemented this model internally in FlumeJava, with MillWheel used as the underlying execution engine for streaming mode; additionally, an external reimplementation for Cloud Dataflow is largely complete at the time of writing. One interesting note is that the core windowing and triggering code is quite general, and a significant portion of it is shared across batch and streaming implementations; that system itself is worthy of a more detailed analysis in future work.

### Design Principles
- Never rely on any notion of completeness
- Be flexible, to accommodate the diversity of known use cases, and those to come in the future.
- Not only make sense, but also add value, in the context of each of the envisioned execution engines.
- Encourage clarity of implementation.
- Support robust analysis of data in the context in which they occurred.

### Motivating Experiences
Just read the paper dude!