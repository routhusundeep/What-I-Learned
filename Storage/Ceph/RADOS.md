#distributed_file_system #real_world_system

[paper](https://ceph.com/assets/pdfs/weil-rados-pdsw07.pdf)

RADOS, a Reliable, Autonomic Distributed Object Store that seeks to leverage device intelligence to distribute the complexity surrounding consistent data access, redundant storage, failure detection, and failure recovery in clusters consisting of many thousands  
of storage device. It facilitates an evolving, balanced distribution of data and workload across a dynamic and heterogeneous storage cluster while providing applications with  
the illusion of a single logical object store with well-defined safety semantics and strong consistency guarantees.

A RADOS system consists of a large collection of OSD's and a small group of monitors responsible for managing OSD cluster membership. Each OSD includes a CPU, some volatile RAM, a network interface, and a locally attached disk drive or RAID. Monitors are stand-alone processes and require a small amount of local storage

- Monitor Cluster
	- manages the master copy of the cluster map and, through it, the rest of the storage cluster, enabling the system to seamlessly scale from a few dozen to many thousands of devices
- Object storage devices(OSD) 
	- seek to distribute low-level block allocation decisions and security enforcement to intelligent storage devices, simplifying data layout and eliminating I/O bottlenecks by facilitating direct client access to data

#### Cluster Map
The map specifies which OSD's are included in the cluster and compactly specifies the distribution of all data in the system across those devices. It is replicated by every storage node as well as clients interacting with the RADOS cluster.

#### Data Placement
RADOS employs a data distribution policy in which objects are pseudo-randomly assigned to devices. When new storage is added, a random sub-sample of existing data is migrated to new devices to restore balance.
Each object stored by the system is first mapped into a placement group (PG), a logical collection of objects that are replicated by the same set of device.

Each object’s PG is determined by a hash of the object name o, the desired level of replication r, and a bit mask m that controls the total number of placement groups in the system. That is, pgid = (r, hash(o)&m), mask m = 2k −1, constraining the number of PGs by a power of two.

PG's are assigned to OSD's using [CRUSH](https://ceph.com/assets/pdfs/weil-crush-sc06.pdf)

#### Device State
The cluster map includes a description and current state of devices over which data is distributed. This includes the current network address of all OSD's that are currently online and reachable (up), and an indication of which devices are currently down.

#### Map Propagation
differences in map epochs are significant only when they vary between two communicating OSD's (or between a client and OSD), which must agree on their proper roles with respect to the particular PG the I/O references. This property allows RADOS to  
distribute map updates lazily by combining them with existing inter-OSD messages, effectively shifting the distribution burden to OSD's.

### Intelligent Storage Devices
RADOS currently implements n-way replication combined with per-object versions and short-term logs for each PG. Replication is performed by the OSD's themselves: clients  
submit a single write operation to the first primary OSD, who is then responsible for consistently and safely updating all replicas. This shifts replication-related bandwidth to  
the storage cluster’s internal network and simplifies client design. Object versions and short-term logs facilitate fast recovery in the event of intermittent node failure (e.g., a  
network disconnect or node crash/reboot)

#### Replication
clients send I/O operations to a single (though possibly different) OS
- Primary-Copy
	- updates all replicas in parallel, and processes both reads and writes at the primary OS
- Chain
	- writes are sent to the primary (head), and reads to the tail, ensuring that reads always reflect fully replicated update
- Splay
	- combines the parallel updates of primary-copy replication with the read/write role separation of chain replication.

#### Strong Consistency
All RADOS messages—both those originating from clients and from other OSDs—are tagged with the sender’s map epoch to ensure that all update operations are applied in a fully consistent fashion.

#### Failure Detection
A failure on the TCP socket results in a limited number of reconnect attempts before a failure is reported to the monitor cluster. Storage nodes exchange periodic heartbeat messages with their peers to ensure that device failures are detected. OSD's that discover that they have been marked down simply sync to disk and kill themselves to ensure consistent behavior.

#### Data Migration and Failure Recovery
data migration and failure recovery are driven entirely by cluster map updates. Such changes may be due to device failures, recoveries, cluster expansion or contraction, or even complete data reshuffling from a totally new CRUSH replica distribution policy.

### Monitors
A small cluster of monitors is collectively responsible for managing the storage system by storing the master copy of the cluster map and making periodic updates in response to  
configuration changes or changes in OSD state (e.g., device failure or recovery)
The cluster, which is based in part on the Paxos part-time parliament algorithm, is designed to favor consistency and durability over availability and update latency.

#### Workload and Scalability
Most map distribution is handled by storage nodes, and device state changes are normally infrequent.
If each OSD stores μ PGs and f OSD's fail, then an upper bound on the number of failure reports generated is on the order of μf, which could be very large if a large OSD cluster experiences a network partition. To prevent such a deluge of messages, OSD's send heartbeats at semi-random intervals to stagger detection of failures, and then throttle and batch failure reports, imposing an upper bound on monitor cluster load proportional to the cluster size.