#reread 

[Non Blocking algorithm](https://en.wikipedia.org/wiki/Non-blocking_algorithm)
It is called non blocking if a suspension or failure of a thread cannot cause failure or suspension of another thread.
- lock-free
  - If there is a guarantee of a system wide progress
  - guarantees no deadlock and livelock by definition
  - no mutex
- wait-free
  - If there is a guarantee of per-thread progress
  - Strongest guarantee
  - Freedom from starvation
  - has a bound on the number of steps
  - CAS and LL/SC are not wait-free
  - Every wait-free is lock-free


[paper](https://www.cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf) 
Did not completely understand as it is hard to form an intuition on distributed algorithms unless you spend large enough time on them.
This is the algorithm used by Java's Concurrent Queue