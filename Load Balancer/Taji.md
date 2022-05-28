#load_balancer #real_world_system #facebook

[paper](https://research.facebook.com/publications/taiji-managing-global-user-traffic-for-large-scale-internet-services-at-the-edge/) [blog](https://blog.acolyer.org/2019/11/15/facebook-taiji/)

Edge nodes are much smaller in size and are situated close to the end users for two major functions: 
-  reverse proxies for terminating user connections close to their ISPs
-  caching and distribution of static content such as images and video.

For the first decade of Facebook’s existence, we used a static mapping to route user requests from edge nodes to data centers. The static mapping became increasingly difficult to maintain as our services expanded globally. The dynamism and heterogeneity of global user traffic brings the following challenges:

- **Capacity crunch:** If a service’s popularity and capacity needs outpace capacity acquisition plans, we might be unable to service all user requests. This imbalance might result in load shedding or failures during peak load time
- **Product heterogeneity:** As products evolved, the nature of user requests changed, stickiness makes it harder for a static mapping to manage user traffic, as static mapping cannot account for unmovable user sessions that are dynamically established.
- **Hardware heterogeneity:**  A traffic management system must be flexible and adapt routing strategies to infrastructure evolution.
- **Fault tolerance:** A static edge to data center routing strategy is rigid and susceptible to failures when operational issues arise.

Taiji was built with two goals in mind:
- balancing the utilization of data centers and 
- minimizing network latency of user requests.

This paper makes the following contributions:
- Taiji is the first system that manages user requests to dynamic content for Internet services in a user-aware manner
- We describe how to model user traffic routing as an assignment problem which can be solved by constraint optimization solvers.
- We present connection-aware routing, which routes traffic from highly-connected users to the same data centers to achieve substantial improvements in caching effectiveness and backend utilization,

#### Background
First check out the difference between [L4 and L7 load balancers](https://www.a10networks.com/glossary/how-do-layer-4-and-layer-7-load-balancing-differ/)

General Path of a request:
- User's browser/app issues a request to an ISP which maps the domain name to a Virtual IP using the DNS resolver
	- This VIP address points to one of the dozens of globally deployed edge nodes
	- An edge node consists of a small number of servers, typically co-located with a [peering network](https://en.wikipedia.org/wiki/Peering)
- The user request will first hit an L4 load balancer which forwards the request to an L7 edge load balancer (Edge LB) where the user’s [SSL connection is terminated](https://en.wikipedia.org/wiki/TLS_termination_proxy).
	- Each Edge LB runs a reverse proxy that maintains persistent secure connections to all data centers.
	- Edge LBs are responsible for routing user requests to frontend machines in a data center through our backbone network.
- Within a data center, the user request goes through the same flow of an L4 load balancer and an L7 load balancer
	- The Frontend LB proxies the user request to a web server.
	- The web server handling the user request is responsible for returning a response to the edge node which then forwards it to the user

Backbone capacity is not a problem

##### Traffic type
- requests can be stateless or sticky
- web services are stateless
- Interactive services, such as instant messaging, pin a user’s requests to the particular machines that maintain the user session.
- For sticky traffic, a new request can be routed to any data center which will initialize a new session; once a session is established, the subsequent user requests are always routed to the same destination based on cookies in the HTTP header


### Taiji
##### Runtime
The Runtime component is responsible for generating a routing table which specifies the fraction of user traffic that each edge node ought to send to a data center. This routing table is a simple collection of tuples of the form: {edge:{datacenter:fraction}}

**Inputs**
- operational state of the infrastructure such as capacity, health and utilization of edge nodes and data centers
- measurement data such as edge traffic volumes and edge-to-datacenter latency
	- Taiji models traffic load as requests per second (RPS) for stateless traffic and as user sessions for sticky traffic.

**Modeling and Constraint Solving**
Taiji formulates edge-to-datacenter routing as an assignment problem that satisfies a service-specific policy. We simplified the design and implementation of Taiji with constraint-based solving based on a generalization of all the closed-formed solvers we had implemented

A policy specifies constraints and objectives. Policies typically have the constraint of not exceeding predefined data center utilization thresholds to avoid overloading any data center. Our most commonly-used policy specifies the objectives of balancing the utilization of all available data centers, while optimizing network latency.

Taiji employs an assignment solver to solve the problem. Our solver employs a local search algorithm using the “best single move” strategy. It considers all single moves: swapping one unit of traffic between two data centers, identifying the best one to apply, and iterating until no better results can be achieved

**Pacing and Dampening**
Taiji employs several safety guards to limit the volume of traffic change permitted in each update of the routing table

##### Traffic Pipeline
Traffic Pipeline consumes the routing table output by Runtime and generates the specific routing entries in a configuration file using connection-aware routing. It then disseminates the routing configuration out to each Edge LB via a configuration management service. The routing entries specify the buckets based on which each edge node routes to data centers.

**Connection-Aware Routing**
Connection-aware routing is built on the insight that user traffic requesting similar content has locality and can benefit the caching and other backend systems. Connection-aware routing brings locality in traffic routing by routing traffic from highly-connected users to the same data center.

`To understand the social hash algorithm and assignment of buckets to data centers, read the paper`

**Edge LB Forwarding**
For a user request, an Edge LB routes the request to the data center according to the user’s bucket specified in the routing configuration. Note that maintaining every user-to-bucket mapping at each Edge LB is inefficient. Thus, we maintain the fine-grained user-to-bucket mapping in data centers and let Edge LBs route traffic at the granularity of buckets.
The initial user request has no bucket information and therefore the Edge LB routes it to a nearby data center. When the request reaches the data center, the frontend writes the bucket ID in the HTTP cookie based on the user-to-bucket mapping. Then the subsequent requests will have bucket ID in the cookie and the Edge LB will route the request to the data center specified in the routing configuration.