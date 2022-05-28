#real_world_system #google #load_balancer 

[paper](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44824.pdf)
#### Maglev: A Fast and Reliable Software Network Load Balancer

A network load balancer is typically composed of multiple devices logically located between routers and service endpoints.
Network load balancers have traditionally been implemented as dedicated hardware devices, an approach that has several limitations
- their scalability is generally constrained by the maximum capacity of a single unit
- they do not meet Google’s requirements for high availability.
	- Though often deployed in pairs to avoid single points of failure, they only provide 1+1 redundancy.
- they lack the flexibility and programmability needed for quick iteration, as it is usually difficult, if not impossible, to modify a hardware load balance
- they are costly to upgrade

Software load balancing system has many advantages
- We can address scalability by adopting the scale-out model, where the capacity of the load balancer can be improved by increasing the number of machines in the system
	- through [ECMP](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-ecmp-flow-based-forwarding.html) forwarding, traffic can be evenly distributed across all machines
	- Availability and reliability are enhanced as the system provides N+1 redundancy
	- By controlling the entire system ourselves, we can quickly add, test, and deploy new features
		- Meanwhile, deployment of the load balancers themselves is greatly simplified: the system uses only existing servers inside the clusters
- We can also divide services between multiple shards of load balancers in the same cluster in order to achieve performance isolation

#### Frontend Serving Architecture
Every Google service has one or more Virtual IP addresses (VIPs). A VIP is different from a physical IP in that it is not assigned to a specific network interface, but rather served by multiple service endpoints behind Maglev. Maglev associates each VIP with a set of service endpoints and announces it to the router over [BGP](https://www.cloudflare.com/learning/security/glossary/what-is-bgp/); the router in turn announces the VIP to Google’s backbone. Aggregations of the VIP networks are announced to the  
Internet to make them globally accessible.

When a user tries to access a Google service, her browser first issues a DNS query,  
which gets a response (possibly cached) from one of Google’s authoritative DNS servers. The DNS server assigns the user to a nearby frontend location taking into account both her geolocation and the current load at each location, and returns a VIP belonging to the selected location in response. The browser will then try to establish a new connection with the VIP.

When the router receives a VIP packet, it forwards the packet to one of the Maglev machines in the cluster through ECMP, since all Maglev machines announce the VIP with the same cost. When the Maglev machine receives the packet, it selects an endpoint from the set of service endpoints associated with the VIP, and encapsulates the packet using Generic Routing Encapsulation ([GRE](https://en.wikipedia.org/wiki/Generic_Routing_Encapsulation)) with the outer IP header destined to the endpoint.

When the packet arrives at the selected service endpoint, it is decapsulated and consumed. The response, when ready, is put into an IP packet with the source address being the VIP and the destination address being the IP of the user. We use Direct Server Return ([DSR](https://www.loadbalancer.org/blog/direct-server-return-is-simply-awesome-and-heres-why/)) to send responses directly to the router so that Maglev does  
not need to handle returning packets, which are typically larger.

#### Maglev Configuration
- **Controller:** responsible for announcing VIP's
	- the controller periodically checks the health status of the forwarder. Depending on the results, the controller decides whether to announce or withdraw all the VIPs via BGP.
- **Forwarder:** Forwards VIP requests to the machines
	- each VIP is configured with one or more backend pools
	- A backend pool may contain the physical IP addresses of the service endpoints; it may also recursively contain other backend pools, so that a frequently used set of backends does not need to be specified repeatedly
	- Each backend pool, depending on its specific requirements, is associated with one or more health checking methods with which all its backends are verified; packets will only be forwarded to the healthy backends.
- Config manager is responsible for parsing and validating config objects before altering the forwarding behavior. All config updates are committed atomically.

#### Forwarder Design and Implementation
##### Overall Structure
The forwarder receives packets from the NIC (Network Interface Card), rewrites them with proper GRE/IP headers and then sends them back to the NIC. The Linux kernel is not involved in this process.

Packets received by the NIC are first processed by the steering module of the forwarder, which calculates the 5-tuple hash of the packets and assigns them to different receiving queues depending on the hash value. Each receiving queue is attached to a packet rewriter thread. 

The packet thread first tries to match each packet to a configured VIP. Then it recomputes the 5-tuple hash of the packet and looks up the hash value in the connection tracking table. We do not reuse the hash value from the steering module to avoid cross-thread synchronization. The tuple is composed of the source IP, source port, destination IP, destination port, and protocol type

The connection table stores backend selection results for recent connections. If a match is found and the selected backend is still healthy, the result is simply reused. Otherwise the thread consults the consistent hashing module and selects a new backend for the packet; it also adds an entry to the connection table for future packets with the same 5-tuple. The forwarder maintains one connection table per packet thread to avoid access contention. After a backend is selected, the packet thread encapsulates the packet with proper GRE/IP headers and sends it to the attached transmission queue. The muxing module then polls all transmission queues and passes the packets to the NIC

The steering module performs 5-tuple hashing instead of round-robin scheduling for two reasons. 
- It helps lower the probability of packet reordering within a connection caused by varying processing speed of different packet threads. 
- With connection tracking, the forwarder only needs to perform backend selection once  
for each connection, saving clock cycles and eliminating the possibility of differing backend selection results caused by race conditions with backend health updates.  

In the rare cases where a given receiving queue fills up, the steering module falls back to round-robin scheduling and spreads packets to other available queues.

##### Fast Packet Processing
When Maglev is started, it pre-allocates a packet pool that is shared between the NIC and the forwarder. Both the steering and muxing modules maintain a ring queue of pointers pointing to packets in the packet pool.

Receiving side
- received pointer: advanced by NIC
- processed pointer: advanced by steering module
- reserved pointer: advanced by steering module

Sending side
- send pointer: advanced by NIC
- ready pointer: advanced by muxing module
- recycled pointer: advanced by muxing module

To reduce the number of expensive boundary-crossing operations, we process packets in batches whenever possible. In addition, the packet threads do not share any data with each other, preventing contention between them. We pin each packet thread to a dedicated CPU core to ensure the best performance.

##### Backend Selection
- A new form of Consistent Hashing is used to map VIP to backend
- selection is stored in connection tracking table, for TCP stickiness. Just a map from 5-tuple hash to backend is insufficient, as the router before Maglev does not confirm connection affinity. 
	- e.g.: when doing rolling upgrade, the set of Maglev machines keeps on changing and traffic can be drained and rerouted.
	- In theory, the size of the map is finite.

##### Consistent Hashing
To overcome the above problem, we could use a distributed hash table, but that's clearly inefficient.
We use local consistent hashing. The idea is to generate a large lookup table, with each backend taking a number of entries in the table.
- Load balancing: each backend will receive an almost equal number of connections.
- Minimal disruption: when the set of backends changes, a connection will likely be sent to the same backend as it was before

But consistent hashing values minimal disruption over load balancing, Maglev takes the opposite approach.

Read the paper for actual implementation - I did not understand it completely

#### Evolution of Maglev
##### Failover
Initially deployed as active-passive, connections are not interrupted due to Maglev Hashing, but it has some drawbacks
- Inefficient resources
- VIP can only be assigned to single Maglev instance
- Coordination is complex

Moved to ECMP model.

##### Packet Processing
Originally used Linux network stack, later moved to kernel by-pass mechanism and used NIC directly.

##### VIP Matching
Specific to Google's setup of clusters.

##### Fragment handling
[IP Fragments](https://packetpushers.net/ip-fragmentation-in-detail/) require special treatment because Maglev performs 5-tuple hashing for most VIPs, but fragments do not all contain the full 5-tuple. Thus when Maglev receives a non-first fragment, it cannot make the correct forwarding decision based only on that packet’s headers.

Upon receipt of a fragment, Maglev computes its 3-tuple hash using the L3 header and forwards it to a Maglev from the pool based on the hash value. Since all fragments belonging to the same datagram contain the same 3-tuple, they are guaranteed to be redirected to the same Maglev. We use the GRE recursion control field to ensure that fragments are only redirected once.

Maglev uses the same backend selection algorithm to choose a backend for unfragmented packets and second-hop first fragments (usually on different Maglev instances.) It maintains a fixed-size fragment table which records forwarding decisions for first fragments. When a second-hop non-first fragment is received by the same machine, Maglev looks it up in the fragment table and forwards it immediately if a match is found; otherwise it is cached in the fragment table until the first one is received or the entry expires.



