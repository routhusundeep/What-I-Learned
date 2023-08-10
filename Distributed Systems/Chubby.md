[paper](https://github.com/papers-we-love/papers-we-love/blob/main/distributed_systems/the-chubby-lock-service-for-loosely-coupled-distributed-systems.pdf) #real_world_system #locking 

We describe our experiences with the Chubby lock service, which is intended to provide coarse-grained locking as well as reliable (though low-volume) storage for a loosely-coupled distributed system.
The primary goals included reliability, availability to a moderately large set of clients, and easy-to-understand semantics; throughput and storage capacity were considered secondary.

### Design
#### Rationale
* A service is easier to integrate with when compared to using a library(implementing Paxos). 
* Many of the services that elect a primary or that partition data between their components need a mechanism for advertising the results. Such data is stored by Chubby itself.
* A lock-based interface is more familiar to our programmers. Both the replicated state machine of Paxos and the critical sections associated with exclusive locks can provide the programmer with the illusion of sequential programming.
* Distributed-consensus algorithms use quorums to make decisions, so they use several replicas to achieve high availability. Thus, a lock service reduces the number of servers needed for a reliable client system to make progress.

These arguments suggest two key design decisions: 
* We chose a lock service, as opposed to a library or service for consensus, and 
* we chose to serve small-files to permit elected primaries to advertise themselves and their parameters, rather than build and maintain a second service.

Some decisions follow from our expected use and from our environment:
* A service advertising its primary via a Chubby file may have thousands of clients. Therefore, we must allow thousands of clients to observe this file, preferably without needing many servers.
* Clients and replicas of a replicated service may wish to know when the service’s primary changes. This suggests that an event notification mechanism would be useful to avoid polling
* Even if clients need not poll files periodically, many will; this is a consequence of supporting many developers. Thus, caching of files is desirable.
* Our developers are confused by non-intuitive caching semantics, so we prefer consistent caching.
* To avoid both financial loss and jail time, we provide security mechanisms, including access control.

Uses coarse grained locks:
* impose far less load on the lock server
* lock-acquisition rate is usually only weakly related to the transaction rate of the client applications
* coarsegrained locks to survive lock server failures,

#### System structure
A Chubby cell consists of a small set of servers (typically five) known as replicas, placed so as to reduce the likelihood of correlated failure (for example, in different racks).
The replicas use a distributed consensus protocol to elect a master; the master must obtain votes from a majority of the replicas, plus promises that those replicas will not elect a different master for an interval of a few seconds known as the master lease.
Clients find the master by sending master location requests to the replicas listed in the DNS.
Write requests are propagated via the consensus protocol to all replicas; such requests are acknowledged when the write has reached a majority of the replicas in the cell.
If a master fails, the other replicas run the election protocol when their master leases expire; a new master will typically be elected in a few seconds. For example, two recent elections took 6s and 4s, but we see values as high as 30s

#### Files, directories, and handles

> /ls/{cell}/....

The name space contains only files and directories, collectively called nodes. Every such node has only one name within its cell; there are no symbolic or hard links.

#### Locks and sequencers
Each Chubby file and directory can act as a reader-writer lock.
Like the mutexes known to most programmers, locks are advisory. That is, they conflict only with other attempts to acquire the same lock: holding a lock called F neither is necessary to access the file F, nor prevents other clients from doing so. We rejected mandatory locks, which make locked objects inaccessible to clients not holding their locks:
* Chubby locks often protect resources implemented by other services, rather than just the file associated with the lock
* We did not wish to force users to shut down applications when they needed to access locked files for debugging or administrative purposes.
* Our developers perform error checking in the conventional way, by writing assertions such as “lock X is held”, so they benefit little from mandatory checks

Chubby provides a means by which sequence numbers can be introduced into only those interactions that make use of locks.
A lock holder may request a sequencer, It contains the name of the lock, the mode in which it was acquired, and the lock generation number.
The client passes the sequencer to servers (such as file servers) if it expects the operation to be protected by the lock. The recipient server is expected to test whether the sequencer is still valid and has the appropriate mode.

Chubby provides an imperfect but easier mechanism to reduce the risk of delayed or re-ordered requests to servers that do not support sequencers. If a client releases a lock in the normal way, it is immediately available for other clients to claim, as one would expect. However, if a lock becomes free because the holder has failed or become inaccessible, the lock server will prevent other clients from claiming the lock for a period called the lock-delay.

#### Events
Chubby clients may subscribe to a range of events when they create a handle. These events are delivered to the client asynchronously via an up-call from the Chubby library. Events include:

#### API
* $Open()$ - open handle
* $Close()$ - close handle
* $GetContentsAndStat()$ 
* $ReadDir()$ 
* $SetContents()$ 
* $SetACL()$ 
* $Delete()$
* $Acquire(),\ TryAcquire(),\ Release()$
* $GetSequencer()$
* $SetSequencer()$
* $CheckSequencer()$

All the calls above take an operation parameter in addition to any others needed by the call itself. The operation parameter holds data and control information that may be associated with any call. In particular, via the operation parameter the client may: 
* supply a callback to make the call asynchronous, 
* wait for the completion of such a call, and/or 
* obtain extended error and diagnostic information.

#### Caching
To reduce read traffic, Chubby clients cache file data and node meta-data (including file absence) in a consistent, write-through cache held in memory.
When file data or meta-data is to be changed, the modification is blocked while the master sends invalidations for the data to every client that may have cached it; this mechanism sits on top of KeepAlive RPCs.

The caching protocol is simple: it invalidates cached data on a change, and never updates it. It would be just as simple to update rather than to invalidate, but updateonly protocols can be arbitrarily inefficient; a client that accessed a file might receive updates indefinitely, causing an unbounded number of unnecessary updates.

#### Sessions and KeepAlives
A Chubby session is a relationship between a Chubby cell and a Chubby client; it exists for some interval of time, and is maintained by periodic handshakes called KeepAlives.
Each session has an associated lease—an interval of time extending into the future during which the master guarantees not to terminate the session unilaterally.
As well as extending the client’s lease, the KeepAlive reply is used to transmit events and cache invalidations back to the client

#### Fail-overs
the new master must reconstruct a conservative approximation of the in-memory state that the previous master had. It does this partly by reading data stored stably on disc (replicated via the normal database replication protocol), partly by obtaining state from clients, and partly by conservative assumptions

A newly elected master proceeds:
1. It first picks a new client epoch number, which clients are required to present on every call. The master rejects calls from clients using older epoch numbers, and provides the new epoch number. This ensures that the new master will not respond to a very old packet that was sent to a previous master, even one running on the same machine.
2. The new master may respond to master-location requests, but does not at first process incoming session-related operations
3. It builds in-memory data structures for sessions and locks that are recorded in the database. Session leases are extended to the maximum that the previous master may have been using.
4. The master now lets clients perform KeepAlives, but no other session-related operations.
5. It emits a fail-over event to each session; this causes clients to flush their caches (because they may have missed invalidations), and to warn applications that other events may have been lost.
6. The master waits until each session acknowledges the fail-over event or lets its session expire. 
7. The master allows all operations to proceed.
8. If a client uses a handle created prior to the fail-over (determined from the value of a sequence number in the handle), the master recreates the in-memory representation of the handle and honours the call. If such a recreated handle is closed, the master records it in memory so that it cannot be recreated in this master epoch; this ensures that a delayed or duplicated network packet cannot accidentally recreate a closed handle. A faulty client can recreate a closed handle in a future epoch, but this is harmless given that the client is already faulty
9. After some interval (a minute, say), the master deletes ephemeral files that have no open file handles. Clients should refresh handles on ephemeral files during this interval after a fail-over. This mechanism has the unfortunate effect that ephemeral files may not disappear promptly if the last client on such a file loses its session during a fail-over

#### Database implementation
we have written a simple database using write ahead logging and snapshotting similar to the design of Birrell et al.

#### Backup
Every few hours, the master of each Chubby cell writes a snapshot of its database to a GFS file server in a different building

#### Mirroring
Chubby allows a collection of files to be mirrored from one cell to another.

### Mechanisms for scaling
* We can create an arbitrary number of Chubby cells; clients almost always use a nearby cell (found with DNS) to avoid reliance on remote machines.
* The master may increase lease times from the default 12s up to around 60s when it is under heavy load, so it need process fewer KeepAlive RPCs
* Chubby clients cache file data, meta-data, the absence of files, and open handles to reduce the number of calls they make on the server.
* We use protocol-conversion servers that translate the Chubby protocol into less-complex protocols such as DNS and others.

#### Proxies
Chubby’s protocol can be proxied (using the same protocol on both sides) by trusted processes that pass requests from other clients to a Chubby cell. A proxy can reduce server load by handling both KeepAlive and read requests; it cannot reduce write traffic, which passes through the proxy’s cache.

#### Partitioning
Chubby’s interface was chosen so that the name space of a cell could be partitioned between servers

#### Lessons learned
* Developers rarely consider availability
* Fine-grained locking could be ignored
* Poor API choices have unexpected affects
* RPC use affects transport protocols
* 
