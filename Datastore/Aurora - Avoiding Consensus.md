[paper](https://dl.acm.org/doi/pdf/10.1145/3183713.3196937?download=true) #real_world_system #relational_database #reread 

Contributions are
1. Writes using asynchronous flows, established local consistency points, uses said points for commit processing, and re-establishes on crash recovery
2. avoid reads and reads are scaled out across replicas
3. quorum sets and epochs to make non-blocking reversible membership changes to process failures, grow storage and reduce costs

### Making Writes Efficient
#### Architecture
- Already described in the previous paper
#### Writes
- only redo logs are sent to storage tier and read replicas
- as per boxcar, once it receives its first operation, it starts the network operation and continues to fill up the buffer till the buffer is completely empty
- This ensures that the jitter is less
#### Storage Consistency Points and Commits
- Each storage nodes keeps track of its own Segment complete LSN(SCL), before which it knows all writes with no gaps
- SCL is sent back with the write acknowledgement
- Once database observes SCL advancement at more than 4 storage nodes, it locally advances the Protection Group Complete LSN(PGCL), representing the point at which the protection group has made all write durable
- The database also keeps track of Volume Complete LSN(VCL)
- A commit is acknowledged only when its LSN is below VCL
- so acknowledgements are kept on hold till there is enough progress in VCL
- A dedicated thread takes care of the acknowledgements every time there is an advancement in VCL
#### Crash Recovery
- after crash, contacts atleast read quorum of nodes to compute the VCL and PGCL
- asks to truncate all LSNs beyong VCL
- if unable to establish write quorum for any protection group, it initiates read repair using the read quorum
- the volume epoch is incremented in the storage metadata once the write quorum is established
- this new epoch invalidates old connections since operation carry the epoch with them, so storage nodes can validate it
- no redo replay happens, undo happens in parallel with user activity
### Reads
- A cache miss will require a read from read quorum
- performs poorly compared to other databases
#### Avoiding Read Quorum
- The database is aware of segments which contains the last durable version of a data block and will request it directly from them
#### Scaling Reads Using Read Replicas
- Write replicas send a stream of redo logs to the read replicas
- very efficient setup and tear down of read replicas
- Easy to promote a read replica to write replica
#### Structural Consistency in Aurora Replicas
- Three invariants are maintained
  - replica read views should lag durability consistent points at the write instance, This ensures that writer and reader does not coordinate cache eviction
  - Structural changes like B-Tree Merge and Splits appear atomically, this ensures consistency during block traversal
  - read views on replicas must be anchorable to equivalent points in time on the writer instance
- log records are only shipped from writer in MTR chunks
#### Snapshot Isolation and Read View Anchors in Replicas
- logical consistency is maintained using snapshot isolation
- transactions undo's are also sent by the redo log

### Failures and Quorum Membership
#### Using Qurorum Sets to change membership
- Each membership change is associated with an epoch
- reads and writes are not blocks during the change
- See the exact details in the paper, it uses simple boolean logic to go through the change
#### Using Quorum Sets to Reduce Costs
- Read the paper regarding this
- I think it is an interesting approach to achieve low-cost while keeping the availability, durability without compromising much performance 
