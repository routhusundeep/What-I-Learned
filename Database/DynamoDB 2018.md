[link](https://www.youtube.com/watch?v=yvBR71D0nAQ)

DynamoDB has changed a lot when compared with the [[Dynamo|original]] paper.

Request path:
User → Network → Request Router(RR) → Storage Node(SN) (Coordinator/Primary)

#### Request Router
- Does the authentication
- are stateless
- Multiple RR's exist in an Availability zones (AZ's)
- Will request Partition Metadata service to get the leader of the partition

#### Storage Node
- Primary Storage Node is the leader of the partition
	- is the primary source of data and is up-to date.
	- Is elected via [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science)) (used to be quorum)
	- There will be two other replicas/peers
	- Heartbeat every 1.5 sec
	- Will wait for response from at least one peer before acknowledging the put
- Multiple SN's in an AZ
- Each AZ will have one SN per partition

**Internals of SN**
B-Tree (for data) and Replication log (similar to WAL)

#### Auto Admin
- Updates Partition metadata system
- partition repair
	- Copy B-Tree and replication log for an active peer to replica
- table provisioning, table creation, Split Partition etc...

 #### Log Propagator
 - Watches replication log on SN and propagates to Secondary Index partition
	 - Might get amplification to writes

#### Provisioning Table Capacity
- Read and Write Capacity Unit (RCU and WCU)
	- 1 RCU = 4kb
- Will be throttled with the load is more than RCU/WCU
- Each partition will have same RCU = total/number of partitions
- Token Bucket Algorithm
	- bucket fill rate = RCU rate
	- if no token then throttle
	- capacity = 300 * RCU
		- extra 5 mins for tokens as it can fill up
	- Can burst into capacity
	- Adaptive Capacity is introduced to avoid hot partitions (imbalance in key space)
		- Adaptive capacity multiplier (ACM) is calculated
		- [PID Controller](https://en.wikipedia.org/wiki/PID_controller) is used to calculate ACM
			- inputs - Consumed Capacity, Provisioned Capacity, Throttling rate and current ACM
			- output - new ACM
- Auto Scaling
	- target utilization, lower and upper bound on capacity
	- Metrics are sent to Cloudwatch
	- Auto-scaling service sets alarms on the metrics in cloudwatch
	- Auto-scaling is responsible to increase or decrease the capacity by telling dynamo

#### Backup and Restore
- Point in Time and On Demand
- Stored in S3
- Snapshot of B-Tree and log is uploaded to S3 without any coordination between partitions
- Snapshot at a particular time is calculated for each partition using the latest snapshot and the log at e
- Should account for the partition split
- Point in time informs the leaders to upload the replication logs

#### DynamoDB streams
- All table mutations
- No duplicates
- In order by key
- New and Old Item image Available
- On top of Kinesis
	- shards, records, check pointing
- Leader SN writes to the Shard asynchronously

#### Global Tables
- Tables in multiple Regions
- streams are used for replication
- Multi-master and Multi-region Replication
