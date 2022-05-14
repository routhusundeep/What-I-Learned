#microkernel #google #networking #did_not_complete
[paper](https://ai.google/research/pubs/pub48630/)

The host networking needs of large-scale Internet services providers are vast and rapidly evolving. Continuous capacity growth demands novel approaches to edge switching and bandwidth management, the rise of cloud computing necessitates rich virtualization features, and high-performance distributed systems continually seek more efficient and lower latency communication.
Our experiences realizing these needs with conventional kernel-based networking were hampered by lengthy development and release cycles.
Hence the need of a user-space networking module(s).

Snap’s architecture is a composition of recent ideas in userspace networking, in-service upgrades, centralized resource accounting, programmable packet processing, kernel bypass RDMA functionality, and optimized co-design of transport, congestion control and routing
- High rate of feature development
- custom packet injection driver and custom CPU scheduler for interoperability
- provides support for OSI layer 4 and 5 functionality through an interface similar to an RDMA-capable “smart” NIC.

#### Snap as a Microkernel Service
The host grants Snap exclusive access to device resources by enabling specific Linux capabilities for Snap (Snap does not run with root privileges) and applications communicate with Snap through library calls that transfer data either asynchronously over shared memory queues (fast path) or synchronously over a Unix domain sockets interface (slow path).