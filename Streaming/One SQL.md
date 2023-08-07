[One SQL to rule them all](https://arxiv.org/pdf/1905.12133.pdf) 

This paper aims to achieve stream processing by SQL alone, current SQL cant do this, they propose some language extensions and primitives to achieve their goal.  
These are
1. Time varying relations as a foundation for classical tables as well as streaming data
2. Event time semantics
3. A limited set of optional keyword extensions to control the materialization of time varying results

Foundations
- Time varying relations
  - Do not differentiate between classical relations and streams, both will work on a more general abstraction called TVR(time varying relations)
  - For a relational database user, a relation is already a TVR since it will be updated over time, but right now the user is denied to operate on these relations with respect to a time dimension unless he explicitly stores it as a property for the relations
- Event time Semantics
  - Do no assume that the events are ordered over time
  - Proposed concepts
    - Explicit event times in the relation
    - Watermarks
      - a mechanism to deterministically or heuristically defining a temporal margin of completeness for a timestamped event
- Materialization Control
  - Stream materialization
    - A space-efficient way of describing the evolution of TVR over time
    - can be implemented using a change log, which in turn can be represented as a TVR
  - Materialization Delay
    - like mentioned a queries output is a TVR, but most the time the users do not need a TVR, they only need the result at some point in time or at regular intervals etc, so by making this delay explicit we can do further optimizations at the implementation level
- All these three ideas are explained in detail using an example
- Lessons learned in practice
  - Some operations only work efficiently in the presence of watermarks
  - Operators may erase watermark alignment of event time attributes
  - TVR's might have more than one event time attribute
    - like join of two TVR
    - holding back the watermark is be a possible solution in this case
  - Reasoning about what can be done with an event time attribute can be difficult for the user
  - Reasoning about size of query state is a necessary evil
  - It is useful for users to distinguish between streaming and materialization operators
  - Torrents of updates are inefficient
- Extending the existing SQL Standard
  - Existing support
    - Queries are on table snaphots
    - Logical and Materialized view map a query pointwise over a TVR
    - Temporal tables are tables trying to emulate TVR
    - Match Recognize - very important to Complex event processing
  - TVR
    - no need for any change, as existing operations cleanly maps TVR as they did for relation
  - Event time
    - a timestamp column should be added for all tables
  - Watermarks
    - a distinguished timestamp column should be added to represent it
  - Group By
    - when group by happens on event time, any grouping where the key is less than the watermark is declared complete and further events are dropped(reasonable time can be configured)
  - Event time Windowing functions
    - Add built in table values functions Tumble and Hop which takes a relation and event time column descriptor as input and returns a relation with additional event time interval columns as output
    - establish a convention on the output column names
    - Tumble
      - Tumble(data , timecol , dur , [offset ])
        - data is table
        - timecol is event time descriptor
        - dur is duration of window
        - offset is optional, which tells the begin of tumble
    - Hop/Hopping/Sliding
      - an additional parameter hopsize will be used
        - if < dur, then intersecting windows
        - if > dur, then gaps in windows
  - Materialization Controls
    - Stream materialization
      - add EMIT STREAM operation
      - the result will be a stream instead of a traditional table
      - will contain additional columns
        - undo - whether it is a retraction of a previous record
        - ptime - processing time offset
        - ver - sequence number telling the revision of this record
    - Materialization delay
      - Completeness delay
        - emit only once the group is complete
        - EMIT AFTER WATERMARK
      - Periodic delays
        - rows are emitted at a periodic interval
        - EMIT AFTER DELAY d
      - Combined delays
        - has both watermark and periodic delay
