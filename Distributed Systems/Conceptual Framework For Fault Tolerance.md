[Link](https://resources.sei.cmu.edu/library/asset-view.cfm?assetid=11747)
#fault_tolerance #taxonomy #definitions

#### System:
A system is the entire set of components, both computer related, and non-computer related, that provides a service to a user.

#### Dependable System:
For a system to be dependable, it must be available (e.g., ready  
for use when we need it), reliable (e.g., able to provide continuity of service while we are using  it), safe (e.g., does not have a catastrophic consequence on the environment), and secure (e.g., able to preserve confidentiality)

**Ways to achieve it:**
- **Fault Avoidance**
Techniques used include design methodologies, verification and validation  methodologies, modelling, and code inspections and walk-throughs
- **Fault Removal**
The range of techniques used for fault removal includes  
unit testing, integration testing, regression testing, and back-to-back testing. *It is generally much more expensive to remove a fault than to avoid a fault.*
- **Fault Tolerance**
For a system to be fault tolerant, it must be able to detect, diagnose, confine, mask, compensate and recover from faults
- **Fault Evasion**
It is possible to observe the behavior of a system and use this information to take action to compensate for faults before they occur.

**Quantitative Goals**
A quantitative reliability goal is usually expressed as the maximum allowed failure-rate.

**Qualitative Goals**
- **Fail-safe**
Design the system so that, when it sustains a specified number of faults, it fails in a safe mode.
- **Fail-op**
Design the system so that, when it sustains a specified number of faults, it still provides a subset of its specified behavior.
- **No single point of failure**
Design the system so that the failure of any single component will not cause the system to fail.
- **Consistency**
Design the system so that all information delivered by the system is equivalent to the information that would be delivered by an instance of a non-faulty system.

#### Faults and Failure
A system is said to have a failure if the service it delivers to the user deviates from compliance with the system specification for a specified period of time.

fault is the adjudged cause of a failure.
An alternate view of faults is to consider them failures in other systems that inter- act with the system under consideration—either a subsystem internal to the system under con- sideration, a component of the system under consideration, or an external system that interacts with the system under consideration

#### Dependency Relations
A component of a system is said to depend on another component if the correctness of the first component’s behavior requires the correct operation of the second component. Tradition- ally, the set of possible dependencies in a system are considered to form an acyclic graph.

```
	    Failures           Users
	-------------          System Boundary

		Faults             Span of Concern

	-------------          Fault Floor
	                       Substrate 
```

