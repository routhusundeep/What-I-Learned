
[paper](http://web.mit.edu/6.829/www/currentsemester/papers/only-just-works.pdf) #real_world_system #networking #reread 

I have always been fascinated by networks, but I never deep dived into this topic due to a paranoia which stemmed from my Bachelors classes. This is a gold mine of information for a beginner in networks like me, a succint and well written paper describing the history and current state of the internet, the various problems the all-mighty internet has faced and how these problems were solved by various means, most of which have passed the test of time.
Without further ado.

### 1970 - 1993
- ARPAnet - first large scale packet switching network
- The switch over to TCP/IP was done on 1 January 1983
  - This was a flag day, easily done due to its small scale of 400 nodes
- in 1982 DNS was deployed replacing host.txt
- Link state protocols were deployed due to the convergence problems of distance vector routing protocols
- EGP was a direct response to the scaling limitations of intra-domain routing
- Congestion was solved using TCP's congestion control, which is not a complete solution
- BGP was introduced to accommodate policy routing
- The final change to the core of internet is CIDR(1993), introduced to tackle diminishing IP addresses
- ECN is standardised but not widely deployed
- Mobile IP was a adaptation failure
- End to End IP Multicast is under utilized
- quality of service should also be used to mitigate various problems

### The future - stagnation
- significant changes are MPLS and VPN
- IP layer and above are stagnated
- QOS and IP Mobility are not adopted
- End-to-end, there is a decrease in transparency, the main reason is security (firewalls)
- NATs also inhibit transparency
  - them not being part of the internet architecture is a problem, as they cannot be addressed directly
  - reason for them is not IPV4 exhaustion, but ISPs charging per IP address
  - used as security solution, irony is that they act as a poor firewall, only advantage is that they are by default fails closed
  - unnatural to go away even in the advent of IPV6
  - along with firewall, NATs are TCP specific, other protocols are increasingly harder to implement
    - VoIP which uses UDP is tricky
    - has to reverse engineer the NAT functionality to get around it (Skype does this)
- deep packet inspection may make incremental change harder
- no substantial change in layer 3 for a decade
- no substantial change in layer 4 for 2 decades

### Convergence
- VoIP, IPTV etc are needed on the internet now
- This will force the internet to adapt to other use cases

### Architectural problem
#### Short term 
- Spam
  - SPIT - spam over internet telephony
- Security
  - taint tracking
  - the biggest problem
- Denial of service
  - Distributed DOS
  - DNS reflection attacks and many others like this
- Application deployment
  - firewall issue - everything unknown is a potential threat, so new deployment will not work
  - many new applications look like HTTP as a workaround
  - like mentioned, Skype is a good example

#### Medium term
- Congestion Control
  - with increase in link speed, congestion may rise again
  - must support all protocols, not just TCP
  - new scheme must be done incrementally, backward compatible with TCP
- Inter-domain routing
  - BGP will not satisfy
  - I did not understand it, I will read about BGP in the future
- Mobility
  - right now, mobile devices solve it at layer 2, laptops solve it via DHCP
  - need to be addressed in the future, due to increasing mobile devices
- Multi-homing
  - technically, all the systems(even in different ISPs) will have the same prefix
  - address aggregation should be prevented in routing announcements, otherwise routing should be done on the edge nodes, BGP will likely not cope with the increase in the number of advertised routes

#### Longer term
- Address space depletion
  - IPv6 is still rare today
