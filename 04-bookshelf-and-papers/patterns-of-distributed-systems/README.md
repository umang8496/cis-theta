<!-- markdownlint-disable MD001 -->
<!-- markdownlint-disable MD024 -->
<!-- markdownlint-disable MD025 -->
<!-- markdownlint-disable MD026 -->
<!-- markdownlint-disable MD029 -->
<!-- markdownlint-disable MD033 -->
<!-- markdownlint-disable MD040 -->
<!-- markdownlint-disable MD056 -->
<!-- markdownlint-disable MD060 -->

# Patterns of Distributed Systems [Unmesh Joshi & Martin Fowler]

## Content

### [Part I] Narratives

- [Chapter 01: Promise and Perils of Distributed Systems](#chapter-01-promise-and-perils-of-distributed-systems)
- [Chapter 02: Replication, Logs, and Leadership](#chapter-02-replication-logs-and-leadership)

### [Part II] Partitions of Data Replication

- [Chapter 03: Write-Ahead Log](#chapter-03-write-ahead-log)
- [Chapter 04: Segmented Log](#chapter-04-segmented-log)
- [Chapter 05: Low-Water Mark](#chapter-05-low-water-mark)
- [Chapter 06: Leader and Followers](#chapter-06-leader-and-followers)
- [Chapter 07: HeartBeat](#chapter-07-heartbeat)
- [Chapter 08: Majority Quorum](#chapter-08-majority-quorum)
- [Chapter 09: Generation Clock](#chapter-09-generation-clock)
- [Chapter 10: High-Water Mark](#chapter-10-high-water-mark)
- [Chapter 11: Paxos](#chapter-11-paxos)
- [Chapter 12: Replicated Log](#chapter-12-replicated-log)
- [Chapter 13: Singular Update Queue](#chapter-13-singular-update-queue)
- [Chapter 14: Request Waiting List](#chapter-14-request-waiting-list)
- [Chapter 15: Idempotent Receiver](#chapter-15-idempotent-receiver)
- [Chapter 16: Follower Reads](#chapter-16-follower-reads)
- [Chapter 17: Versioned Value](#chapter-17-versioned-value)
- [Chapter 18: Version Vector](#chapter-18-version-vector)

### [Part III] Patterns of Data Partitioning

- [Chapter 19: Fixed Partitions](#chapter-19-fixed-partitions)
- [Chapter 20: Key-Range Partitions](#chapter-20-key-range-partitions)
- [Chapter 21: Two-Phase Commit](#chapter-21-two-phase-commit)

### [Part IV] Patterns of Distributed Time

- [Chapter 22: Lamport Clock](#chapter-22-lamport-clock)
- [Chapter 23: Hybrid Clock](#chapter-23-hybrid-clock)
- [Chapter 24: Clock-Bound Wait](#chapter-24-clock-bound-wait)

### [Part V] Patterns of Cluster Management

- [Chapter 25: Consistent Core](#chapter-25-consistent-core)
- [Chapter 26: Lease](#chapter-26-lease)
- [Chapter 27: State Watch](#chapter-27-state-watch)
- [Chapter 28: Gossip Dissemination](#chapter-28-gossip-dissemination)
- [Chapter 29: Emergent Leader](#chapter-29-emergent-leader)

### [Part VI] Patterns of Communication between Nodes

- [Chapter 30: Single-Socket Channel](#chapter-30-single-socket-channel)
- [Chapter 31: Request Batch](#chapter-31-request-batch)
- [Chapter 32: Request Pipeline](#chapter-32-request-pipeline)

---

## Chapter 01: Promise and Perils of Distributed Systems

### 01. Fundamental Resources of Any Modern System

Every modern software system, irrespective of scale or domain, ultimately depends on four fundamental physical resources:

#### 1.1 CPU

The CPU represents computational capacity. It determines:

- How many instructions can be executed per unit time
- How much concurrent work a system can handle
- The upper bound on request processing throughput

CPU is often the first bottleneck in compute-heavy workloads.

#### 1.2 Memory

Memory is responsible for:

- Holding in-memory state
- Caching frequently accessed data
- Buffering in-flight requests and responses

Memory access is significantly faster than disk access, which makes it a critical resource for performance-sensitive systems.

#### 1.3 Network Bandwidth

Network bandwidth is the maximum volume of data that can be transmitted over a network per unit time.

Key characteristics:

- Measured in bps, Mbps, or Gbps
- Represents capacity, not latency
- Becomes a limiting factor when systems rely on inter-node communication

In distributed systems, every remote call consumes bandwidth, making it a first-class constraint.

#### 1.4 Disk Space

Disk provides durable storage:

- Persistent state
- Logs, snapshots, and backups
- Large datasets that cannot fit in memory

Disk access is slower than memory but essential for durability and recovery.

### 02. Throughput as a System Metric

Throughput is the amount of useful work completed per unit time.

Examples:

- Requests per second
- Transactions per second
- Bytes processed per second

Throughput is constrained by the slowest limiting resource among CPU, memory, network, or disk.  
Increasing throughput requires either optimizing resource usage or adding more resources.  

### 03. Limitations of Single-Server Applications

A single-server architecture centralizes all responsibilities—compute, memory, storage, and networking—into one machine.  
This design has several inherent limitations:

#### 3.1 Limited Scalability

> A single machine has hard physical limits. Once CPU, memory, disk, or network capacity is exhausted, no further scaling is possible.

#### 3.2 Single Point of Failure

> If the server crashes (hardware or software), the entire system becomes unavailable.

#### 3.3 Throughput Ceiling

> There is a strict upper bound on the number of requests that can be processed.

#### 3.4 Poor Availability

> Maintenance, upgrades, or crashes result in full downtime.

#### 3.5 Vertical Scaling Constraints

> Scaling up hardware is expensive and eventually impossible beyond a point.

#### 3.6 Lack of Fault Isolation

> A failure in one part of the system can cascade and impact everything else.

These limitations make single-server systems unsuitable for large-scale or mission-critical workloads.

### 04. First Evolution: Stateless Compute + Stateful Database

To overcome single-server constraints, systems evolve into a two-component architecture:

- Stateless compute layer
  - Contains business logic
  - Can be horizontally scaled
  - Easy to replace or replicate

- Stateful persistence layer
  - Centralized database
  - Responsible for durability and consistency

Benefits

- Compute scales independently
- Improved availability at the application layer
- Better utilization of resources

New Bottleneck Introduced (As data grows to terabytes or petabytes)

- Disk capacity becomes a bottleneck
- Disk I/O limits throughput
- The database becomes the new single point of failure

This architecture solves earlier problems but creates data scalability challenges.

### 05. Second Evolution: Compute Cluster and Data Cluster

To address database bottlenecks, systems further evolve into:

#### 5.1 Compute Cluster

- Multiple stateless servers
- Executes business logic
- Scales horizontally

#### 5.2 Data Cluster

- Large dataset split into smaller partitions
- Data distributed across multiple machines
- Removes the monolithic database

This eliminates the single massive disk bottleneck and improves fault tolerance.

### 06. Problems Introduced by Distributed Data

While distributing data solves storage limitations, it introduces new complexity:

#### 6.1 Data Consistency

> Ensuring all nodes see a correct and meaningful view of data becomes difficult.

#### 6.2 Partial Failures

> Some nodes may fail while others remain operational.

#### 6.3 Network Dependency

> Every data access requires network communication, introducing latency and uncertainty.

#### 6.4 Coordination Overhead

> Operations may require agreement across nodes.

#### 6.5 Data Placement and Routing

> The system must determine where data lives and how to access it.

#### 6.6 Rebalancing Complexity

> Adding or removing nodes requires data movement.

#### 6.7 Operational Complexity

> Monitoring, debugging, and recovery become significantly harder.

#### 6.8 Split-Brain Risk

> Network partitions can lead to divergent system state.

### 07. Data Partitioning (Sharding)

Data partitioning divides a large dataset into smaller chunks distributed across servers.  

Advantages

- Solves disk capacity limits
- Enables horizontal scaling
- Improves write throughput

Drawback

- Data is unevenly distributed
- Coordination and routing complexity increases
- Does not inherently solve availability problems

Partitioning is necessary but insufficient on its own.

### 08. Replication as the Next Step

To address failures and availability concerns, systems introduce data replication.  
Data replication is the practice of storing multiple copies of the same data on different machines.  

Goals

- Mask failures
- Improve availability
- Increase read throughput
- Enable fault tolerance

### 09. Cost of Replication at Scale

When datasets are very large:

- Storage costs multiply
- Network bandwidth is consumed during replication
- Writes become slower due to coordination
- Recovery and rebalancing are expensive

At petabyte scale, full replication is economically and operationally infeasible, requiring selective or limited replication strategies.

### 10. Failures Replication Tries to Mask

Replication exists primarily to tolerate and mask unavoidable failures, including:

#### Process crash

> A node stops executing entirely.

#### Network delay

> Messages are delayed unpredictably.

#### Process pause

> A node is alive but temporarily unresponsive (e.g., garbage collection).

#### Unsynchronized clocks

> Time-based assumptions fail across machines.

These failures are normal, not exceptional, in distributed systems.

### 11. What a Distributed System Is

A distributed system is:

```text
A software architecture composed of multiple independent nodes that communicate over a network and coordinate their actions to achieve a common goal, while presenting themselves as a unified system.
```

#### Key Properties

- Independent failure domains
- Network-based communication
- Coordination under uncertainty
- Designed for scalability and growth

#### Core Benefits

- Fault tolerance
- Resilience
- Horizontal scalability

### 12. Chapter Summary: Why Distributed Systems Are Necessary

The first chapter establishes a clear narrative:

- Single machines cannot scale indefinitely
- Centralized databases become bottlenecks
- Distributing data solves capacity issues
- Distribution introduces failures and complexity
- Replication mitigates failures but adds trade-offs
- Distributed systems are unavoidable at scale

This sets the foundation for the rest of the book, which focuses on patterns to manage these trade-offs systematically.

[Go to the Top](#content)

---

## Chapter 02: Replication, Logs, and Leadership

**A Story of Scale, Staleness, and the Need for Strong Coordination**.  

### 01. Re-establishing the Context from Chapter 01

Chapter 01 establishes **why distributed systems are unavoidable**: single machines hit hard limits on CPU, memory, disk, and network, and centralized databases become bottlenecks and single points of failure.  

Chapter 02 builds directly on that motivation and introduces a critical idea that frames everything that follows:

> **Distributing data means partitioning and replication.**

This chapter focuses primarily on **replication**—why it exists, what problems it solves, and, more importantly, what *new* problems it introduces.

### 02. What Replication Really Means (and What It Does Not)

At first glance, replication seems simple:

- Copy the same data to multiple servers
- If one server fails, others can take over

The initial mental model (which is a very common and valid starting point) was:

- The **entire dataset** is replicated across multiple nodes
- One node becomes the **leader**
- The leader accepts **all write operations**
- Other nodes follow the **Write-Ahead Log (WAL)** and serve **read requests**

This model is called **leader–follower (primary–replica) replication**, and it is absolutely correct as a baseline.

However, the book subtly pushes us to refine this model later:

- This approach works well **before sharding**
- Replication is usually applied **per shard**, not globally
- Different shards can have different leaders

For now, Chapter 02 assumes replication without yet introducing sharding in depth.

### 03. The Core Problem Replication Introduces

Replication solves **availability** and **fault tolerance**, but it introduces **coordination problems** that do not exist in single-node systems.

Key challenges include:

- **Partial writes**: a write succeeds on one node but not others
- **Mid-flight failures**: a leader crashes during processing
- **Concurrent writes**: multiple clients issuing writes at the same time
- **Divergent replicas**: different nodes having different views of state

These issues arise because:

- Networks are unreliable
- Processes can crash or pause
- Time is not synchronized

Replication, therefore, demands **ordering, durability, and agreement**.

### 04. Write-Ahead Log (WAL): The First Anchor of Safety

To deal with partial failures and crashes, systems rely on the **Write-Ahead Log (WAL)**.

#### What WAL Provides

- Every state change is **recorded before it is applied**
- If a node crashes, it can **replay the log** and recover
- WAL establishes a **total order of writes** on a node

There is an important question:

> *Is WAL something we need to implement per application, or is it part of the database?*

The answer is:

- **Most databases already implement WAL**
- Applications usually rely on database WAL
- Distributed systems often implement a **system-level WAL** when coordinating multiple nodes

WAL provides **durability**, but by itself it does **not provide availability**.

### 05. From WAL to Replicated Logs

To achieve availability, WAL entries must be **replicated**.  

Instead of replicating mutable database state:

- Systems replicate the **log of changes**
- Logs are append-only and easier to reason about

This leads to **replicated logs**, where:

- The leader appends entries
- Followers receive and replay them

Martin Fowler's explanation highlights an important realization:

- WAL alone cannot protect against leader failure
- Replication must involve **agreement**, not just copying

### 06. Leader–Follower Model as a Coordination Strategy

To avoid conflicting writes:

- A **single leader** accepts writes
- Followers **do not accept writes**
- Followers only apply what the leader commits

You explicitly asked:

> *Does the leader always perform the write operation?*

In this model:

- **Yes**, for a given shard, the leader is the only writer
- This simplifies ordering and consistency
- Followers serve reads and provide failover

This design trades **write scalability** for **correctness and simplicity**.

### 07. Majority Quorum: The Basis of Safety

Replication requires agreement on *what is safe*.  
This is where **majority quorum** comes in.  

A majority quorum means:

- More than half of the replicas (⌊N/2⌋ + 1)
- Any two majorities always overlap
- If a write is accepted by a majority, it **cannot be lost**

This guarantees that:

- A new leader elected from the majority will have committed data
- Safety is preserved across failures

### 08. Log Commitment and the Need for a Boundary

Replication introduces an uncomfortable reality:

- Some replicas may have more log entries
- Some may have fewer
- Not all log entries are equally safe

Failure scenarios include:

- Leader crashes before replication
- Leader crashes after partial replication
- Leader crashes after full replication

The system needs a way to answer:
> *Which log entries can we safely expose to clients?*

### 09. High-Water Mark: Drawing the Safety Line

This is where the **high-water mark (HWM)** comes in—a concept that initially felt unintuitive.  

From Martin Fowler's explanation:

> The high-water mark is the index of the highest log entry known to be replicated to a majority quorum.

- Entries **≤ HWM**:
  - Replicated to a majority
  - Guaranteed to survive leader failure
  - Safe to apply and serve

- Entries **> HWM**:
  - Possibly replicated to only some nodes
  - Unsafe
  - Must not be exposed

The leader:

- Advances the HWM when quorum is reached
- Shares the HWM with followers

All nodes:

- Serve client-visible data **only up to the HWM**

The HWM is both:

- A **commit boundary**
- A **visibility boundary**

### 10. Generation Clock: Handling Multiple Failures Over Time

Single failures are not the real problem—**sequences of failures are**.  
The book introduces the **generation clock** to handle this.

Scenario:

- Leader L1 writes to WAL
- L1 crashes
- Leader L2 is elected
- L1 later comes back

Without a generation clock:

- L1 might believe it is still leader
- It may attempt to accept or replicate writes

A **generation clock (term/epoch)** solves this:

- Incremented on **leader election**
- Every log entry is tagged with a generation
- Nodes reject entries from older generations

You confirmed your understanding by asking whether:

- WAL + leader failure + generation increment is the correct flow

It is — with the clarification that **generation increments at election**, not at failure.

### 11. Internals of a Leader Under Load

When many concurrent writes arrive:

- The leader cannot block on each request
- It uses **multiple internal queues**

Typical pipeline:

1. Request intake queue
2. WAL append queue
3. Replication queue
4. Commit/apply queue

This design:

- Decouples disk I/O, network I/O, and computation
- Keeps the leader responsive
- Preserves ordering and safety

### 12. Leader Failure During Concurrent Writes

If a leader fails mid-processing:

- Some writes may exist only in memory
- Some may be in WAL but not replicated
- Some may be replicated but not committed

Safety is preserved because:

- Only entries below the **high-water mark** survive
- New leader is elected from a **majority**
- Uncommitted entries are safely discarded or retried

### 13. Leader Discovery and Client Perspective

You asked a crucial client-side question:

> *How does the client know who the new leader is?*

This is handled by:

- Redirect responses (“Not leader”)
- Coordination metadata
- Smart client libraries
- Load balancers with leader awareness

Clients:

- Retry safely
- Do not need to know whether a previous write succeeded
- Rely on idempotency and ordering guarantees

### 14. Supporting Concepts Clarified Along the Way

#### What Is a Shard?

A **shard** is a subset of data.  
Replication happens **within a shard**, not across the entire dataset.

#### Partitioning vs Sharding vs Replication

- **Partitioning**: logical division of data
- **Sharding**: partitioning across machines
- **Replication**: copying data for fault tolerance

#### Replication Lag

- Time between leader commit and replica visibility
- Ranges from milliseconds to seconds
- Managed internally by databases or replication systems

#### Who Performs Leader Election?

- The cluster itself
- Using quorum-based coordination
- Not the application

### 15. Followers Handling Read Requests to Reduce Leader Load

As systems grow, the **leader quickly becomes a bottleneck**.  
Even though write traffic is naturally serialized through the leader, **read traffic often dominates** real workloads.  

To address this:

- Followers (replicas) are allowed to **serve read requests**
- The leader focuses on:
  - Accepting writes
  - Replicating logs
  - Coordinating quorum and commit

This design dramatically improves **read scalability** and reduces pressure on the leader.  

At first glance, this looks like an obvious win.  
But distributed systems rarely give something for free.

### 16. The Stale Read Problem After a Successful Write

This is also known as **read-after-write anomaly**.  

1. User **U** sends a **write request** to leader **L**
2. Leader **L**:
   - Appends the update to its **Write-Ahead Log**
   - Replicates the log entry to followers
   - Achieves a **majority quorum**
   - Marks the entry committed
   - Applies the update locally
   - Responds **success** to U
3. Followers:
   - Have received the log entry
   - Have **not yet applied** it to their local state
4. User **U** immediately sends a **read request**
5. That read is routed to a follower that is **behind**

### The Result

User **U sees stale data**, even though:

- The write succeeded
- The write is durable
- The system is safe

This is not a bug.  
This is a **designed trade-off**.

### 17. Understanding What Was (and Was Not) Guaranteed

The system **did guarantee**:

- Durability (via WAL)
- Survival across leader failure (via majority quorum)

The system **did not guarantee**:

- Immediate visibility on all replicas
- Read-your-writes consistency for follower reads

This distinction is crucial:

> **Commit means “will not be lost,” not “is visible everywhere.”**

### 18. Ways to Handle the Stale Read Trade-off

There is no universal solution—only **explicit choices**:

- **Leader-only reads**  
  Strong consistency, lower scalability

- **Quorum / barrier reads**  
  Followers wait until they catch up to a known log index

- **Session stickiness**  
  Clients read from the same node that handled their write

- **Monotonic reads**  
  Clients never observe older data than they have already seen

- **Eventual consistency**  
  Accept staleness as part of system behavior

This section teaches a recurring lesson:  
> Distributed systems trade *immediacy* for *availability and scale*.

### 19. Replication Alone Is Not Enough for Large Datasets

Replication protects against failure, but it **does not solve scale**.  

If the dataset grows very large:

- Every replica still needs to store **all data**
- Disk and I/O become bottlenecks again
- Replication multiplies storage cost

To overcome **physical limits of a single server**, we need another tool.

### 20. Partitioning (Sharding) as the Next Step

**Partitioning** divides a large dataset into smaller, independent pieces.  

Each partition:

- Contains only a subset of the data
- Can be stored and processed separately

This is often called **sharding** when partitions are placed on different machines.

The goal:

- Distribute data evenly
- Distribute load evenly
- Enable horizontal growth

### 21. Core Requirements of Any Partitioning Scheme

For partitioning to work in practice, it must satisfy three key requirements:

1. **Even data distribution**  
   Avoid hotspots and overloaded nodes

2. **Deterministic data placement**  
   The system must know *exactly* which node owns a record, without querying everyone

3. **Efficient rebalancing**  
   Adding new nodes should not require massive data reshuffling

These requirements shape every real-world sharding strategy.

### 22. Types of Partitions

Partitioning itself introduces complexity and trade-offs.

Two important types discussed are:

- **Fixed Partitions**  
  - Number of partitions decided in advance  
  - Simple routing  
  - Can cause imbalance as data grows unevenly

- **Key-Range Partitions**  
  - Data divided by ordered ranges  
  - Enables range queries  
  - Vulnerable to hotspots

No partitioning strategy is perfect—each optimizes for different access patterns.

### 22. Partitioning Must Be Combined with Replication

Partitioning solves **load and scale**, but it does **not solve failures**.  

If a node hosting a partition fails:

- That partition becomes unavailable

Therefore:
> **Each partition must be replicated.**

This leads to the combined model:

- Partitioning for scalability
- Replication for fault tolerance

Each partition:

- Has its own replicas
- Often has its own leader

### 23. Choosing a Practical Replication Factor

A key insight here is subtle but important:

> The replication factor does **not** need to grow with the number of partitions.

Why?

- Larger replication factors increase:
  - Majority quorum size
  - Write latency
  - Coordination overhead

In practice:

- **3 to 5 replicas per partition** strike a good balance
- Enough fault tolerance
- Manageable quorum sizes
- Acceptable performance

This is a pragmatic engineering choice, not a theoretical one.

### 24. Consistency Across Multiple Partitions

Single-partition operations are relatively simple.

Problems arise when:

- A single logical operation spans **multiple partitions**
- Example: transferring money between two accounts in different shards

Now the system must ensure:

- **Atomicity across partitions**
- Either all updates succeed, or none do

This is where **Two-Phase Commit (2PC)** enters the picture.

### 25. Two-Phase Commit (2PC)

2PC coordinates a distributed transaction across partitions using two phases:

1. **Prepare phase**
   - Each participant checks if it can commit
   - Writes intent to its log

2. **Commit phase**
   - Coordinator decides commit or abort
   - Participants finalize the decision

#### Critical Detail You Noted

- The **coordinator maintains its own replicated log**
- **Each participating partition maintains its own replicated log**

This ensures:

- Durability of decisions
- Crash recovery
- Correct resumption after failures

2PC preserves correctness but introduces:

- Higher latency
- Blocking in some failure scenarios

Again, correctness comes at a cost.

### 26. Why System Timestamps Cannot Be Used for Ordering

Distributed systems cannot rely on physical time because:

- Clocks are not perfectly synchronized
- Clocks drift
- Time can move backward during correction
- Network delays reorder events
- Process pauses distort time perception
- Timestamps do not capture causality

Using timestamps for ordering leads to **incorrect event sequences** and **state divergence**.

Two nodes process updates:

- Node A's clock is fast
- Node B's clock is slow

An update that **happens later** can receive an **earlier timestamp**.

If ordering depends on timestamps:

- Updates are applied in the wrong order
- Replicas diverge

This demonstrates why **time is not a reliable ordering mechanism**.

### 27. Lamport Clocks as the Fix

**Lamport Clocks** replace physical time with logical ordering.

They ensure:

- If event A happened before event B, then  
  **Lamport(A) < Lamport(B)**

They:

- Preserve causality
- Do not rely on synchronized clocks
- Provide consistent ordering across nodes

They solve ordering—not time.

### 28. The Need for a Consistent Core

As clusters grow large, managing them becomes a distributed problem itself.

The system must agree on:

- Who is alive
- Who is leader
- Who owns which partitions

This coordination **cannot tolerate inconsistency**.

Hence the need for a **Consistent Core**:

- A small, strongly consistent subsystem
- Acts as the control plane

#### Problems the Consistent Core Solves

Without a consistent core, systems suffer from:

- Split-brain leaders
- Conflicting partition ownership
- Inconsistent membership views
- Unsafe rebalancing

Even if data replication is correct, **coordination failures can break the system**.

#### Responsibilities of the Consistent Core

The consistent core:

- Tracks cluster membership
- Manages leader election
- Grants exclusive ownership via **leases**
- Allows components to react to changes via **state watch**

It provides a **single source of truth** for coordination.

### 29. Gossip Dissemination in the Data Cluster

Not all information needs strong consistency.

**Gossip dissemination** is used to:

- Spread membership information
- Share health and metadata
- Scale to very large clusters

It is:

- Decentralized
- Fault tolerant
- Eventually consistent

### 30. Gossip vs Consistent Core

These two mechanisms serve different purposes:

- **Gossip**
  - Fast
  - Scalable
  - Eventually consistent
  - Informational state

- **Consistent Core**
  - Strongly consistent
  - Authoritative
  - Coordination and control

Large distributed systems need **both**.

### Closing Thought

This extended part of Chapter 02 deepens a central lesson:

> Scaling data is not the hardest problem.  
> Coordinating safely under failure is.

Partitioning, replication, logs, ordering, and coordination are all tools to manage that reality.

[Go to the Top](#content)

---

## Chapter 03: Write-Ahead Log

Modern databases are expected to provide **reliable, consistent, and durable storage** even in the presence of failures.  
Users trust databases with critical business data, and any data loss or corruption can have catastrophic consequences.  

Strong durability is required even in case of server failures or crashes.  
Once a server agrees to perform an action, it should do so even if it fails and restarts losing all if its in-memory state.  

To achieve this, most distributed databases adopt a **Leader–Follower (Primary–Replica)** architecture.  
This pattern helps scale read traffic and centralize write control.  
However, this model introduces several challenges—especially around **fault tolerance, consistency, and durability**.  

In a typical Leader–Follower setup:

- There is **one leader node**
- There are **multiple follower nodes (replicas)**
- **All write operations** go to the leader
- **Read operations** can go to any follower

This architecture is chosen because:

- A single leader:
  - Prevents conflicting writes
  - Maintains a single source of truth
- Followers:
  - Scale read operations
  - Improve availability
  - Reduce load on the leader

How it works in practice:

- Client sends a write request
- Request reaches the leader
- Leader executes the write
- Changes are propagated to replicas
- Reads are served from replicas

This looks simple and efficient. But problems arise when **failures occur**.  

### Downsides of Leader–Follower Pattern

#### Leader Crash During Writes

- Multiple write requests arrive
- Leader starts executing them
- Some writes update memory
- Some updates reach disk
- Suddenly, the leader crashes

Now the database state becomes unclear:

- Some transactions are half-complete
- Some are fully applied
- Some never started

After restart:

- We cannot determine:
  - Which transactions committed
  - Which must be rolled back
- Database enters an **inconsistent state**

This violates the **Atomicity** property of ACID.

#### False Commit Problem

1. Leader executes a write in memory  
2. Leader replies to client:  
   > "Transaction committed successfully"
3. Data is still in RAM  
4. System crashes  
5. Memory is wiped  

After restart:

- Data is gone
- Client thinks operation succeeded

This violates **Durability**.

#### Leader as a Bottleneck

Because all writes go to one node. This leads to:

- Throughput limitation
- Resource exhaustion
- Increased latency

But still manageable.

#### Multiple Leaders – A Bad Idea

A naive solution might be:

> "Let us add more leaders."

This introduces:

- Network partitions
- Both leaders think they are primary
- Both accept writes

Now:

- Data diverges
- Two realities exist
- System enters **Split-Brain**

This is disastrous and almost impossible to recover from.

### Why We Need a Better Solution

From the above problems, we conclude:

- Memory cannot be trusted
- Disk writes are unordered
- Crashes are inevitable
- Replication alone is not enough

We need:

- A **single source of truth**
- Deterministic recovery
- A way to **replay history**

This is where **Write-Ahead Logging** comes in.

### Write-Ahead Logging (WAL)

Write-Ahead Logging is a durability and recovery mechanism based on one fundamental rule:

> **Never modify data pages before writing the change to a log.**

This log is:

- Append-only
- Sequential
- Stored on disk
- Called the **WAL file**

The log becomes the **ultimate source of truth**.

### How WAL Works – Step by Step

#### Step 1 – Client Sends Write

Client sends INSERT, UPDATE, DELETE operations.  
Request reaches leader.

#### Step 2 – Leader Writes to WAL First

Before touching real data:

- Leader creates a log record
- Log contains:
  - Transaction ID
  - Operation type
  - Old value
  - New value

This record is appended to WAL buffer.

#### Step 3 – WAL Is Flushed to Disk

On COMMIT:

- WAL buffer is flushed (`fsync`)
- COMMIT record is written
- Only then:
  - Leader replies success

At this point:

- Data pages may still be untouched
- But transaction is **durable**

#### Step 4 – WAL Is Replicated

Leader streams WAL records to followers.

Followers:

- Write WAL locally
- Replay operations

Depending on configuration:

- Leader waits for:
  - Majority quorum
- Then confirms success

This ensures:

- Replicas are consistent
- No data loss even if leader dies

#### Step 5 – Actual Data Pages Are Updated

Later:

- Background threads:
  - Apply WAL to data pages
- Happens:
  - Asynchronously
  - Out of order
  - Lazily

Safe because:

> WAL already preserves history.

### Crash Recovery Using WAL

Leader crashes again.

This time:

- WAL file is intact

On restart:

#### Phase 1 – Analysis

- Scan WAL
- Identify:
  - Committed transactions
  - Active transactions

#### Phase 2 – REDO

- Reapply committed changes

#### Phase 3 – UNDO

- Rollback incomplete ones

Database returns to:

> **Consistent and durable state**

### How WAL Solves Leader–Follower Problems

| Problem | WAL Solution |
| -------- | ------------ |
| Partial writes | Undo via logs |
| Lost commits | Redo from logs |
| Leader crash | Replay history |
| Replica lag | Log-based sync |
| Split-brain | Single log authority |

### Final Outcome

With WAL:

- Leader crash → Safe recovery
- False commits → Impossible
- Replicas → Consistent
- Writes → Durable
- System → Fault tolerant

> WAL converts random in-memory updates into a deterministic history of truth.

It ensures:

- Atomicity
- Durability
- Recoverability
- High availability

This is why:

- PostgreSQL
- MySQL (InnoDB)
- Oracle
- SQL Server

**All rely on WAL-based recovery.**  

> Leader–Follower architecture is powerful but fragile without recovery mechanisms.  
> Write-Ahead Logging completes this model by introducing:

- Order
- Determinism
- Safety
- Reliability

Together, they form the backbone of modern distributed databases.

[Go to the Top](#content)

---

## Chapter 04: Segmented Log

Before understanding segmented logs, we must establish the role of WAL.  
**Write-Ahead Logging** (WAL) is a durability mechanism where:

> Data is first written to a log before being applied to memory or data files.

Why WAL exists:

- Crash recovery
- Replication correctness
- Atomicity
- Durability guarantees

In leader–follower systems:

- Leader appends to WAL
- Followers replicate from WAL
- Guarantees strong consistency and fault tolerance

### What Is a Segmented Log?

A Segmented Log is a WAL that is split into multiple fixed-size files (segments) instead of one continuously growing file.  

```text
wal-000001.log
wal-000002.log
wal-000003.log
...
```

Each segment:

- Has a maximum size (~128MB)
- Is append-only
- Becomes read-only once full
- A new segment is created when the current one fills

This design is used in Kafka, Raft, etcd, PostgreSQL, Cassandra, HBase.

### Why Single WAL File Is a Problem

Using one ever-growing log file causes major operational issues.

#### Unbounded File Growth

- File grows forever
- Hard to manage disk usage
- Cleanup becomes complex

#### Slow Crash Recovery

- Recovery requires scanning entire file
- Startup time increases linearly with file size

#### Difficult Cleanup

- Truncating large files is risky
- Hard to delete old data safely

#### OS & Filesystem Problems

- Large file fragmentation
- Backup inefficiency
- Performance degradation

#### Replication Inefficiency

- Hard to map offsets to file positions
- Random reads become expensive

### Problems Solved by Segmented Log

| Problem              | How Segmentation Solves It    |
| -------------------- | ----------------------------- |
| Infinite growth      | Old segments deleted          |
| Slow recovery        | Replay only last segment      |
| Disk pressure        | Drop obsolete segments        |
| Compaction           | Rewrite per segment           |
| Replication          | Easy offset → segment mapping |
| Snapshot integration | Delete all old segments       |

### How Segmented Log Works

- Leader appends records to active segment
- When segment size limit is reached:
  - Close it (make read-only)
  - Create new active segment
- Maintain metadata:
  - Segment start offset
  - End offset
- Background tasks:
  - Delete old segments
  - Compact segments
  - Archive if required

#### Performance Benefits

Writes

- Sequential appends
- OS page cache friendly
- No random seeks

Reads

- Offset → segment mapping is O(1)
- Binary search over segment index

Cleanup

- Delete entire files
- Constant time operation
- No rewriting required

### WAL vs Segmented Log

| Aspect              | Single WAL  | Segmented Log |
| ------------------- | ----------- | ------------- |
| File size           | Unbounded   | Fixed chunks  |
| Cleanup             | Hard        | Easy          |
| Recovery            | Slow        | Fast          |
| Replication         | Inefficient | Efficient     |
| Compaction          | Difficult   | Simple        |
| Operational control | Poor        | Excellent     |

### Final Summary

> A Segmented Log is an append-only WAL split into fixed-size files.
> It solves:
>
> - Infinite file growth
> - Slow recovery
> - Cleanup complexity
>
> It is the backbone of:
>
> - Kafka
> - Raft
> - NoSQL databases
>
> Without segmented logs, modern distributed systems would not scale.

```text
Segmented logs are an operational optimization over WAL that enable bounded storage, fast recovery, efficient replication, and log compaction in distributed systems.
```

[Go to the Top](#content)

---

## Chapter 05: Low-Water Mark

### Where Low-Water Mark (LWM) Fits — and Why It Exists

Re-establish the Pieces (Briefly).  
We have **three distinct log-related concepts**:  

#### 1. Write-Ahead Log (WAL)

- Local durability mechanism
- Ensures crash recovery
- Append-before-apply
- Per node

#### 2. Segmented Log

- A **storage optimization** over WAL
- Splits the log into manageable segments
- Enables truncation and compaction
- Still local to a node

#### 3. Replicated Log

- A **coordination mechanism**
- Propagates committed writes from leader to followers
- Preserves ordering and consistency
- Exists logically across nodes

All of these **produce logs**, but **none of them answer one critical question**:

> *When is it safe to delete old log entries?*

This is where **Low-Water Mark (LWM)** comes in.  

### The Core Problem That Requires Low-Water Mark

Logs grow **monotonically**, and without cleanup:

- WAL grows forever
- Replicated logs grow forever
- Disk fills up
- Recovery slows down
- Replicas fall behind

But we **cannot delete log entries arbitrarily**, because:

- A replica may still need them
- A recovering node may still replay them
- A lagging follower may still depend on them

So the real problem is:

> **How do we know which part of the log is no longer needed by *anyone*?**

### What Low-Water Mark Actually Represents

The **Low-Water Mark (LWM)** is:

> **The lowest log index such that all log entries *below it* are no longer needed by any participant in the system.**

In plain terms:

- Entries **below LWM** are:
  - Applied everywhere
  - No longer required for recovery
  - No longer required for replication
  - Safe to delete

- Entries **above LWM**:
  - Still needed by at least one replica
  - Must be retained

### Contrast: High-Water Mark vs Low-Water Mark

This is crucial.

| Marker | Meaning |
| ---- | ------ |
| **High-Water Mark (HWM)** | Up to here, data is **safe to expose** |
| **Low-Water Mark (LWM)** | Below here, data is **safe to delete** |

- HWM protects **correctness**
- LWM protects **resource usage**

They solve **opposite sides of the lifecycle**.

### Where LWM Comes into the Picture (System Story)

#### Step-by-step narrative

1. Leader appends entries to the log
2. Entries are replicated to followers
3. Majority quorum commits entries (HWM moves forward)
4. Followers **apply** entries to their state machines
5. Eventually, **all replicas** confirm they have applied entries up to index `X`

At this point:

- No node will ever need entries `< X`
- No recovery requires them
- No replication requires them

Only then can:

- WAL segments be deleted
- Replicated log entries be truncated
- Disk reclaimed safely

#### Why Replication Makes LWM Necessary

In a **single-node system**:

- WAL truncation is easy
- Once applied → delete

In a **replicated system**:

- Different nodes progress at different speeds
- Some nodes may:
  - Be slow
  - Be temporarily down
  - Be catching up

So deletion must be:

- **Global**
- **Conservative**
- **Based on the slowest relevant node**

That minimum progress across nodes defines the **low-water mark**.  

### LWM's Role in Practice

#### LWM Enables

- Log truncation
- Segment deletion
- Bounded disk usage
- Faster restarts
- Controlled recovery cost

#### LWM Prevents

- Deleting data needed by lagging replicas
- Breaking recovery after failures
- Corrupting replica state

### Relationship to Segmented Logs

Segmented logs exist so that:

- Once LWM crosses a segment boundary
- The entire segment can be deleted atomically

Without LWM:

- Segmented logs still grow
- You just manage growth better—but never stop it

So:
> **Segmented logs make deletion efficient;  
> Low-water mark makes deletion correct.**

#### One-Line Mental Model

> **High-water mark tells you what is safe to *use*;  
> Low-water mark tells you what is safe to *forget*.**

### Example: Low-Water Mark in a Replicated Log System

Assume:

- 1 **Leader** (L)
- 3 **Followers** (F1, F2, F3)
- Replication factor = 4
- Log indices are monotonically increasing

#### 1. Initial State — Logs Growing

##### Leader WAL / Replicated Log

```sh
Index | Entry

100 | SET A=1
101 | SET B=2
102 | SET C=3
103 | SET D=4
104 | SET E=5
```

##### Followers' Progress

```sh
Follower | Last Applied Index

F1 | 104
F2 | 103
F3 | 101
```

Observation:

- All entries up to **101** are applied everywhere
- Entries 102–104 are still pending on at least one follower

#### 2. Computing the Low-Water Mark

The system computes: `LWM = min(lastAppliedIndex of all replicas)`  

So: `LWM = min(104, 103, 101) = 101`

Meaning:

- Entries **≤ 101** are no longer needed by *any* node
- Entries **> 101** must be retained

#### 3. Log View with LWM Annotated

```sh
Index | Entry | Status

100 | SET A=1 | SAFE TO DELETE
101 | SET B=2 | SAFE TO DELETE ← LWM
102 | SET C=3 | RETAIN
103 | SET D=4 | RETAIN
104 | SET E=5 | RETAIN
```

#### 4. Segmented Log Perspective (Very Important)

Assume segments of size 2:

```sh
segment-98-99.log
segment-100-101.log
segment-102-103.log
segment-104-105.log
```

Since: `LWM = 101`

Safe deletion:

- `segment-98-99.log`
- `segment-100-101.log`

Unsafe:

- `segment-102-103.log`
- `segment-104-105.log`

#### 5. Log Messages You'd Actually See

```sh
[INFO] ReplicaProgress: F1=104, F2=103, F3=101
[INFO] LowWaterMark advanced from 99 to 101
[INFO] Deleting log segments up to index 101
[INFO] Removed segment-98-99.log
[INFO] Removed segment-100-101.log
```

#### 6. What Happens When a Slow Replica Catches Up

Later, F3 applies more entries:

```sh
Follower | Last Applied Index
F1 | 104
F2 | 104
F3 | 104
```

Now: `LWM = min(104, 104, 104) = 104`  

Leader log:

```sh
[INFO] LowWaterMark advanced from 101 to 104
[INFO] Deleting log segments up to index 104
[INFO] Removed segment-102-103.log
```

#### 7. Why LWM Must Be Conservative

If the leader deleted entries **before** F3 applied them:

- F3 could never catch up
- Recovery would fail
- Replica would need a full snapshot or rebuild

So:
> **LWM always moves at the pace of the slowest relevant replica**

#### 8. Relationship to Failure & Recovery

If F3 crashes and restarts:

- It replays logs **from LWM onward**
- Anything below LWM is already reflected in its state
- Recovery is bounded and fast

#### 9. One-Sentence Reality Check

> Low-Water Mark is not about correctness of writes.
> It is about **correctness of deletion**.

[Go to the Top](#content)

---

## Chapter 06: Leader and Followers

### Motivation: Why Leader–Follower Is Needed

In replicated data systems, **fault tolerance** is achieved by storing data on multiple servers.  
However, replication alone does not guarantee that clients will observe **consistent data**.  

Early approaches rely on **read and write majority quorum**, but this has a fundamental limitation:

- Each server has **only local knowledge** of its own state.
- A server does **not know** the exact state of data on other servers in the quorum.
- Inconsistencies often become visible **only when comparing responses from multiple servers**.
- Even then, some failure scenarios cannot be fully resolved by quorum reads alone.

As a result, quorum-based approaches may still allow:

- Clients to see stale or conflicting data
- Ambiguous ordering of updates
- Temporary violations of consistency guarantees

This creates the need for a **stronger coordination model**.

### Core Idea of the Leader–Follower Pattern

The **Leader and Followers** pattern introduces **centralized decision-making** into an otherwise distributed system.  
The core idea is simple but powerful:

> **Select exactly one server as the leader, and let it make decisions on behalf of the entire cluster.**

The leader:

- Serializes updates
- Establishes a single global order of operations
- Propagates decisions to all other servers (followers)

Followers:

- Do not make independent decisions
- Apply updates exactly as instructed by the leader
- Serve as replicas for fault tolerance and read scalability

### Responsibilities of the Leader

In the Leader–Follower model, the leader has explicit responsibilities:

- **Accept client requests**, especially writes
- **Decide the order of updates**
- **Coordinate replication** to followers
- **Ensure consistency guarantees** before responding to clients
- **Propagate committed decisions** to all replicas

Because all authoritative decisions flow through a single leader:

- Conflicting updates are avoided
- Ordering is deterministic
- Clients receive a clear and consistent view of the system

### Role of Followers

Followers exist primarily to support:

- **Fault tolerance**
- **Read scalability**
- **Fast recovery when a leader fails**

Followers:

- Replicate state changes from the leader
- Do not independently accept client writes
- May serve read requests (depending on consistency requirements)

If a client sends a request to a follower:

- The follower **forwards or redirects** the request to the leader
- This ensures that decisions are always made by the authoritative node

### Cluster Startup and Leader Discovery

Leader election is not optional—it is a **prerequisite** for correct operation.
At startup:

1. Each server checks whether a **leader already exists**
2. If a leader is found:
   - The server joins as a follower
3. If no leader is found:
   - A **leader election process** is triggered

**Servers do not accept client requests until a leader is elected.**

This rule prevents:

- Split-brain scenarios
- Multiple nodes making conflicting decisions
- Clients interacting with an uncoordinated cluster

### Leader Election as a Critical Operation

Leader election is one of the most critical operations in the **Leader–Follower** pattern.

Key considerations:

- Exactly one leader must be elected
- All servers must agree on who the leader is
- The election must tolerate failures and message delays

To support this, leader election algorithms:

- Rely on quorum agreement
- Avoid dependence on physical clocks
- Ensure monotonic leadership changes

A failed or ambiguous election can lead to:

- Multiple leaders
- Data corruption
- Permanent inconsistency

Hence, leader election must be **carefully designed and strictly enforced**.

### Generation-Key (Term/Epoch)

To distinguish between different leaders over time, the system maintains a **Generation-Key** (also known as term or epoch).

Properties of the Generation-Key:

- It is **incremented every time a new leader is elected**
- Each leader operates under exactly one generation
- All decisions and updates are tagged with the generation

This ensures that:

- Old leaders cannot reassert authority after recovery
- Stale messages from previous leaders are rejected
- The system can unambiguously identify the *current* leader

The Generation-Key is fundamental to preventing split-brain behavior.

### Heartbeats and Leader Liveness

Once elected, the leader must continuously prove that it is still alive.  
This is done using **heartbeats**:

- Periodic signals sent by the leader to followers
- Followers monitor heartbeats to assess leader liveness

If heartbeats stop:

- Followers suspect leader failure
- A new leader election is triggered

Heartbeats provide:

- Timely failure detection
- Fast recovery
- Clear transitions between leaders

As long as heartbeats continue, the leader remains authoritative.

### Why Leader–Follower Provides Stronger Guarantees

Compared to quorum-only approaches, Leader–Follower offers:

- **Single source of truth** for decisions
- **Global ordering** of updates
- **Clear authority model**
- **Simpler reasoning** about system behavior

Even though replication still exists:

- Decisions are not made independently
- Inconsistencies are prevented proactively, not resolved after the fact

This makes the pattern especially suitable for:

- Databases
- Coordination services
- Systems requiring strong consistency guarantees

### Key Takeaways

- Quorum alone is not sufficient for strong consistency
- A single leader simplifies ordering and decision-making
- Followers replicate and support availability
- Leader election and generation keys prevent split-brain
- Heartbeats ensure timely failure detection

The Leader–Follower pattern lays the groundwork for more advanced topics, including:

- Replicated logs
- Consensus protocols
- Strongly consistent coordination services

[Go to the Top](#content)

---

## Chapter 07: Heartbeat

In a distributed database cluster, data is:

- **Partitioned** across servers for scalability
- **Replicated** across servers for fault tolerance

At any given time:

- One server acts as the **leader (primary / write DB)**
- Other servers act as **followers (read replicas)**

For the cluster to function correctly, it is **not enough** for the leader to exist.  
All followers must **continuously know** that the leader is:

- Alive
- Reachable
- Still authoritative

This creates a fundamental requirement:

> **The system must detect server failures quickly and reliably.**

### The Core Problem: Timely Failure Detection

When multiple servers form a cluster:

- Each server is responsible for **some portion of data**
- If a server fails:
  - Requests for that data must be redirected
  - Leadership may need to change
  - Replicas must take over responsibility

Delayed failure detection leads to:

- Increased unavailability
- Client-facing errors
- Stalled writes
- Inconsistent behavior

Therefore, **timely detection of failures** is critical for:

- High availability
- Correct leader election
- Fast recovery

### Heartbeat: The Basic Mechanism

The solution is the **heartbeat**. A heartbeat is:

- A **periodic signal**
- Sent by the **leader**
- Received by **followers**
- Used to indicate *liveness*, not correctness

Lets assume, we have 1 leader, 5 read replicas in the cluster.  

The leader:

- Sends a heartbeat every fixed interval (~1000 ms)

The followers:

- Receive and process heartbeats
- Track missed heartbeats
- Detect possible leader failure

### Heartbeat Processing by Followers

Each follower maintains:

- The timestamp of the **last received heartbeat**
- A counter for **consecutive missed heartbeats**

If 3–5 consecutive heartbeats are missed. Then:

- The leader is **suspected failed**
- A **failure signal** is raised inside the cluster
- Leader election is triggered

Important note:
> Heartbeat failure indicates **suspicion**, not certainty.

### Trade-offs in Heartbeat Interval Selection

#### Short Intervals

- Faster failure detection
- Lower recovery time
- Higher network and CPU overhead
- Increased risk of false positives

#### Long Intervals

- Lower overhead
- Reduced false positives
- Slower failure detection
- Longer unavailability windows

Therefore:
> Failure-detection algorithms must be **tuned to application needs**.

There is no universal “correct” interval.

### Heartbeat-Based Failure Detection Is Imperfect

Heartbeat-based systems must accept an important reality:

> **False failure detection is unavoidable.**

Common causes:

- Network delays
- Temporary partitions
- Garbage collection pauses
- CPU starvation
- Load spikes

A healthy leader may *appear* dead to followers.

Distributed systems are designed to:

- Tolerate false suspicions
- Recover safely from them
- Avoid irreversible damage

### Heartbeats and Leader Election

Heartbeats are tightly coupled with **leader election**.

#### Startup Behavior

- Each server checks for an existing leader
- If none is found, leader election is triggered
- Servers accept requests **only after a leader is elected**

#### Runtime Behavior

- As long as heartbeats are received:
  - The leader remains authoritative
- If heartbeats stop:
  - Followers initiate leader election

This ensures:

- Continuous availability
- Clear leadership transitions
- Prevention of split-brain scenarios

### Generation-Key and Heartbeats

Every leader election:

- Increments a **Generation-Key** (term / epoch)

Heartbeats:

- Carry the current generation
- Allow followers to distinguish:
  - Current leader
  - Stale or recovered leaders

This guarantees that:

- Old leaders cannot regain authority
- Messages from previous terms are rejected

Heartbeats + generation-key together enforce **temporal correctness**.

### Consensus-Based Heartbeats (Small Clusters)

Used when:

- Cluster size is small
- Strong consistency is required

Characteristics:

- Fixed heartbeat timeout
- If timeout expires:
  - Leader is considered failed
  - New leader is elected
- Strong agreement on leader state

Drawback:

- Sensitive to transient failures
- Higher risk of false positives

### Gossip-Based Heartbeats (Large Clusters)

Used when:

- Cluster size is large
- Scalability is a priority

Characteristics:

- Heartbeat information spreads gradually
- Nodes gossip liveness information
- Failure detection is probabilistic
- Eventual agreement, not immediate consensus

Trade-off:

- Slower convergence
- Better scalability
- Higher resilience to churn

### What Happens When Heartbeats Fail

A typical failure timeline:

1. Leader crashes or pauses
2. Heartbeats stop
3. Followers increment missed-heartbeat counters
4. Failure threshold is crossed
5. Leader is suspected failed
6. Leader election begins
7. A new leader is elected
8. New leader starts sending heartbeats

Throughout this process:

- Availability is restored
- Safety is preserved via generation-keys
- Old leaders are invalidated

### Why Heartbeats Are About Suspicion, Not Truth

A key insight of Chapter 07 is:

> **Failure detection is not about proving failure — it is about reacting to suspicion quickly.**

Heartbeats:

- Do not guarantee correctness
- Do not prevent mistakes
- Enable timely recovery

Distributed systems survive by:

- Detecting problems early
- Recovering fast
- Accepting occasional false alarms

### High-Level Flow (What We Are Modeling)

We assume:

- 1 Leader
- N Followers (read replicas)
- Leader sends heartbeats periodically
- Followers detect leader failure
- Followers trigger leader election on failure

#### Core Components

```java
enum Role {
    LEADER,
    FOLLOWER
}

class Heartbeat {
    final long term;
    final long timestamp;

    Heartbeat(long term) {
        this.term = term;
        this.timestamp = System.currentTimeMillis();
    }
}
```

**term** → generation/epoch  
**timestamp** → local observation (not used for ordering)

```java
class LeaderNode {

    private final long term;
    private final List<FollowerNode> followers;
    private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();

    LeaderNode(long term, List<FollowerNode> followers) {
        this.term = term;
        this.followers = followers;
    }

    public void startHeartbeats() {
        scheduler.scheduleAtFixedRate(
            this::sendHeartbeat,
            0,
            1000,
            TimeUnit.MILLISECONDS
        );
    }

    private void sendHeartbeat() {
        Heartbeat heartbeat = new Heartbeat(term);
        for (FollowerNode follower : followers) {
            follower.receiveHeartbeat(heartbeat);
        }
    }
}
```

Key Observations

- Heartbeats are periodic
- Leader does not wait for acknowledgments
- Purpose is liveness, not replication

```java
class FollowerNode {

    private static final int MISSED_HEARTBEAT_THRESHOLD = 3;

    private volatile long lastHeartbeatTime = System.currentTimeMillis();
    private volatile long currentTerm = 0;
    private int missedHeartbeats = 0;
    private Role role = Role.FOLLOWER;

    private final ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();

    public FollowerNode() {
        startFailureDetector();
    }

    public synchronized void receiveHeartbeat(Heartbeat heartbeat) {
        if (heartbeat.term >= currentTerm) {
            currentTerm = heartbeat.term;
            lastHeartbeatTime = System.currentTimeMillis();
            missedHeartbeats = 0;
        }
    }

    private void startFailureDetector() {
        scheduler.scheduleAtFixedRate(
            this::checkLeaderLiveness,
            1000,
            1000,
            TimeUnit.MILLISECONDS
        );
    }

    private synchronized void checkLeaderLiveness() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastHeartbeatTime;

        if (elapsed > 1000) {
            missedHeartbeats++;
        }

        if (missedHeartbeats >= MISSED_HEARTBEAT_THRESHOLD) {
            System.out.println("Leader suspected failed");
            triggerLeaderElection();
        }
    }
}
```

#### What This Models

- Heartbeat interval = 1000 ms
- Failure suspected after 3 consecutive misses
- Detection is local and probabilistic
- False positives are possible

#### Leader Election (Simplified)

```java
private void triggerLeaderElection() {
    role = Role.LEADER;
    currentTerm++;
    missedHeartbeats = 0;
    System.out.println("Node elected as leader for term " + currentTerm);
    // In real systems:
    // - quorum voting happens
    // - leadership must be agreed cluster-wide
}
```

### What Happens During Heartbeat Failures

- Leader crashes or pauses
- Heartbeats stop
- Followers increment missed-heartbeat counters

After threshold:

- Leader is suspected failed
- Election is triggered
- One follower becomes leader
- New leader starts heartbeats

#### Why False Failures Are Inevitable

Heartbeat failure ≠ actual failure

False positives occur due to:

- Network delay
- GC pauses
- CPU starvation
- Temporary overload

This is why:

- Heartbeat-based detection is best-effort
- Systems must tolerate mistaken suspicions
- Generation/term prevents stale leaders

### Key takeaways

- Replication requires timely failure detection
- Heartbeats communicate leader liveness
- Missed heartbeats trigger suspicion and recovery
- Shorter intervals improve responsiveness but reduce efficiency
- False failures are expected and tolerated
- Consensus-based heartbeats suit small clusters
- Gossip-based heartbeats suit large clusters
- Generation-keys prevent stale leaders from causing harm

> Heartbeats do not ensure correctness by themselves, but they enable **self-healing**, which is essential for building reliable distributed systems at scale.

- Leader sends periodic heartbeats
- Followers monitor liveness locally
- Missed heartbeats trigger suspicion
- Elections restore availability
- False failures are acceptable and expected

### Consensus-Based vs Gossip-Based Heartbeats

Heartbeats answer one question:

> **“Who is alive right now?”**

But *how* that answer is produced differs radically between **consensus-based** and **gossip-based** approaches.

In a distributed cluster:

- Nodes fail
- Networks delay or drop messages
- Processes pause without crashing

The system must:

- Detect failures **quickly**
- Avoid **split-brain**
- Scale as the cluster grows

No heartbeat mechanism can be:

- Perfect
- Instant
- Free

#### Consensus-Based Heartbeats

A **consensus-based heartbeat system** ties failure detection directly to a **strongly consistent agreement protocol**.  

Key idea:
> **All (or a majority of) nodes must agree on whether a leader is alive.**

This approach is common in:

- Small clusters
- Strongly consistent systems
- Control planes / consistent cores

##### How It Works (Consensus-Based Heartbeats)

1. A leader is elected via consensus
2. The leader sends heartbeats
3. Followers expect heartbeats within a **fixed timeout**
4. If heartbeats are missed:
   - Followers *agree* that the leader is failed
   - A new election is triggered
5. A new leader is chosen by **quorum agreement**

Failure detection and leader election are **not independent** — they are part of the same protocol.

##### Key Properties (Consensus-based Heartbeat)

- **Strong agreement**
- **Single authoritative leader**
- **Deterministic outcome**
- **Immediate cluster-wide decision**

##### What It Optimizes For (Consensus-Based Heartbeats)

- Safety
- Consistency
- Simplicity of reasoning

##### What It Sacrifices (Consensus-Based Heartbeats)

- Scalability
- Tolerance to transient issues
- Flexibility

Even short network delays or GC pauses can cause **false leader failure**.

##### Typical Use Cases (Consensus-Based Heartbeats)

- Metadata services
- Coordination systems
- Small database clusters
- Systems where correctness > availability

#### Gossip-Based Heartbeats

A **gossip-based heartbeat system** spreads liveness information **epidemically**, without requiring global agreement.

Key idea:
> **Nodes exchange what they know, and the cluster converges over time.**

There is no single instant where “the cluster agrees.”  

##### How It Works (Gossip-Based Heartbeats)

1. Each node tracks heartbeats for itself
2. Periodically, a node:
   - Selects a few random peers
   - Shares its view of who is alive/suspected
3. Peers merge that information
4. Over time:
   - Liveness information spreads
   - Failures become widely known
5. Decisions are based on **local suspicion**, not global agreement

##### Key Properties (Gossip-based Heartbeat)

- **Eventually consistent**
- **Decentralized**
- **Probabilistic**
- **Highly scalable**

No single node has a complete or authoritative view at any moment.

##### What It Optimizes For (Gossip-Based Heartbeats)

- Scalability
- Fault tolerance
- Large cluster size
- Resilience to churn

##### What It Sacrifices (Gossip-Based Heartbeats)

- Immediate consistency
- Deterministic failure detection
- Strong leadership guarantees

Different nodes may temporarily disagree about who is alive.

##### Typical Use Cases (Gossip-Based Heartbeats)

- Large data clusters
- Membership tracking
- Failure suspicion
- Load-based routing
- Systems where availability > strict consistency

### Why Consensus-Based Heartbeats Don't Scale

Consensus requires:

- Majority agreement
- Multiple round trips
- Coordination overhead

As cluster size grows:

- Latency increases
- Failure probability increases
- Leader churn becomes frequent

At large scale:
> **Consensus becomes the bottleneck, not the data.**

### Why Gossip-Based Heartbeats Can't Be Used for Leadership

Leadership requires:

- Exactly one leader
- Global agreement
- No ambiguity

Gossip provides:

- Local views
- Temporary disagreement
- Eventual convergence

So gossip is **not safe** for:

- Leader election
- Partition ownership
- Write coordination

This is why large systems separate concerns.

### The Common Hybrid Architecture (Very Important)

Most real-world systems use **both**:

#### Gossip For

- Membership
- Health
- Load
- Informational state

#### Consensus For

- Leadership
- Partition ownership
- Configuration
- Coordination decisions

This matches what we already saw as:

- **Consistent Core** (consensus)
- **Data Cluster** (gossip)

### Consensus-Based vs Gossip-Based

| Dimension | Consensus-Based | Gossip-Based |
| ------ | -------------- | ------------ |
| Agreement | Strong | Eventual |
| Scalability | Low–Medium | High |
| Failure Detection | Deterministic | Probabilistic |
| False Positives | Likely | Smoothed out |
| Leadership | Safe | Unsafe |
| Typical Cluster Size | Small | Large |
| Decision Style | Global | Local |

#### One Mental Model to Remember

- **Consensus-based heartbeats** answer:  
  *“Does the cluster agree the leader is alive?”*

- **Gossip-based heartbeats** answer:  
  *“What do I believe about the cluster right now?”*

> **Consensus-based heartbeats provide strong, immediate agreement for small, safety-critical systems,**
> **while gossip-based heartbeats provide scalable, eventually consistent failure detection for large clusters — and real systems use both where appropriate.**

[Go to the Top](#content)

---

## Chapter 08: Majority Quorum

First: Clarifying a Core Point

> **Leader–Follower is not primarily about scalability.**

Leader–Follower primarily exists to achieve:

- **Safety** (single ordering point)
- **Simplicity of reasoning**
- **Correct replication**

Scalability is a *secondary* benefit, mainly for **reads**, not writes.  

Writes in Leader–Follower systems are *inherently serialized* at the leader.

### What Majority Quorum Actually Guarantees

Majority Quorum does **one very specific thing**:

> **It guarantees that once a write is acknowledged, it cannot be lost, even if leaders change.**

Why? Because any two majorities **must intersect**.

This intersection property ensures:

- A future leader **must have seen** the committed update
- No two leaders can commit conflicting updates independently

This is a **safety guarantee**, not a performance feature.

### Why WAL + Replication + Majority Quorum Are Tied Together

1. Client sends write to leader
2. Leader:
   - Appends entry to WAL
   - Buffers it
3. WAL entry is sent to followers
4. Followers persist the entry
5. Once **majority acknowledges**, the entry is:
   - Committed
   - Applied
   - Visible (subject to read rules)

**Without majority**, WAL alone is not enough.  

WAL protects against **single-node crashes**.  
Majority protects against **leadership change**.

### Why the Story Is “Not That Simple”

#### Buffers

- Writes are batched
- Improves throughput
- Increases tail latency variance

#### Heartbeats

- Establish liveness
- Maintain leadership authority
- Carry term / generation

#### Timeouts

- Bound waiting time
- Balance liveness vs safety

All of these exist to **contain the cost of quorum**.

### The Fundamental Majority Quorum Trade-off

#### Large Number of Replicas

- Pros:
  - Higher read throughput
  - Better fault tolerance
- Cons:
  - Larger quorum
  - Higher write latency
  - More coordination overhead

#### Small Number of Replicas

- Pros:
  - Faster writes
  - Lower coordination cost
- Cons:
  - Less fault tolerance
  - Lower availability under failures

This is why **replication factor does not scale with cluster size**.

### Why “3 to 5 Replicas” Is the Industry Sweet Spot

This is not folklore — it is **math + operational reality**.

#### Replication Factor = 3

- Quorum = 2
- Tolerates 1 failure
- Fast writes
- Minimal overhead

#### Replication Factor = 5

- Quorum = 3
- Tolerates 2 failures
- More durable
- Higher cost

Beyond 5:

- Marginal gains diminish rapidly
- Coordination cost dominates

Hence:
> **3–5 replicas per shard is optimal for most systems**

### About PostgreSQL

- **Does not use quorum-based commit by default**
- Uses **primary–standby streaming replication**
- Commit acknowledgment depends on configuration:
  - Asynchronous (default)
  - Synchronous (can wait for replicas)

Postgres chooses:

- **Liveness by default**
- Allows safety tuning via synchronous replication

Distributed databases choose:

- **Safety by default**
- Allow liveness tuning via quorum configuration

Neither is “wrong”; they optimize for different use cases.

### Majority Quorum Is Not About Performance

> **Majority Quorum exists to prevent irreversible inconsistency, not to scale the system.**

It exists to answer:

- “Can this update survive failures?”
- “Can two leaders disagree?”
- “Can committed data be lost?”

Everything else (latency, throughput) is a **side effect**.

### Why Majority Quorum Is Unavoidable in Distributed Systems

If we do not use majority:

- We must accept data loss
- Or accept split brain
- Or accept manual reconciliation

There is no free alternative. We can:

- Hide Majority Quorum behind batching
- Tune timeouts
- Reduce replica count
- Restrict reads

But we cannot **remove** Majority Quorum and still claim safety.

### One Mental Model That Will Stick

Think of Majority Quorum as:

> **The minimum number of witnesses required so that history cannot be rewritten.**

If fewer witnesses exist:

- History can diverge
- Truth becomes ambiguous

### Final One-Line Summary

> **Majority Quorum is the smallest agreement mechanism that simultaneously preserves safety across failures and allows practical liveness** —
> **and every real distributed database is fundamentally shaped by this constraint.**

[Go to the Top](#content)

---

## Chapter 09: Generation Clock

> Versioning Leadership to Prevent **Split-Brain** and **Preserve Safety**.

### Motivation: Why Leadership Needs Versioning

In a **Leader–Followers** distributed system, failures are not always clean or permanent.  
Some of the most dangerous failures are **temporary**:

- Garbage collection pauses
- Short network partitions
- CPU starvation
- Transient I/O stalls

In these scenarios:

- The **leader process is still running**
- It believes it is still the leader
- But followers may stop hearing from it

If followers miss heartbeats long enough, they will:

- Suspect leader failure
- Elect a **new leader**

When the old leader recovers, it may resume sending:

- Heartbeats
- Replication requests

This creates a critical danger:
> **Two leaders may concurrently attempt to control the cluster.**

This is the classic **split-brain problem**, and it cannot be solved with heartbeats alone.

### The Core Problem Being Solved

Heartbeats answer:
> “Is this node alive?”

They do **not** answer:
> “Is this node still authorized to lead?”

A temporarily disconnected leader is:

- Alive
- Responsive
- But **no longer legitimate**

The system therefore needs a way to:

- Distinguish *current* leaders from *former* leaders
- Invalidate stale authority
- Ensure that only one leader can ever be accepted at a time

### Definition: Generation Clock

A **Generation Clock** is:

- A **monotonically increasing number**
- Associated with **leader authority**
- Also known as:
  - **Term**
  - **Epoch**
  - **Generation**

Each leadership tenure corresponds to **exactly one generation**.

> Leadership is not a boolean state — it is **versioned authority**.

#### Fundamental Rule of the Generation Clock

The rule is simple and absolute:

> **Only messages from the highest known generation are valid.**

Any message (heartbeat, replication request, vote, etc.) carrying an **older generation** must be rejected.  
This rule alone prevents split-brain.  

#### How Generation Changes

- The generation value:
  - Starts from an initial value (e.g., 0 or 1)
  - **Increments by exactly 1** on every successful leader election
- There is **no decrement**
- There is **no reuse**

This monotonicity is essential for correctness.

### Persistence of the Generation Clock

The generation must **survive crashes and restarts**.

Therefore:

- The generation is **persisted in durable storage**
- Typically stored:
  - Alongside log entries
  - In the **Write-Ahead Log (WAL)**

On startup:

- A server reads the **last known generation** from its log
- It resumes operation with that value

Without persistence:

- A rebooted node could reappear with stale authority
- Safety would be violated

### Generation Clock and Leader Election

Leader election is tightly coupled with the generation clock.

#### Election Process (Conceptual)

1. A server detects leader failure
2. It increments its **generation**
3. It sends **vote requests** containing:
   - Its proposed generation
4. Other servers:
   - Compare the received generation with their own
   - Vote only for **the highest generation**
5. Once a leader is elected:
   - All servers agree on the **same generation**

This ensures:

- A single, globally agreed leadership version
- No ambiguity about authority

#### Propagation of Generation After Election

Once elected:

- The new leader **announces itself** to followers
- Followers update:
  - Leader identity
  - Current generation

From this point onward:

- The generation becomes part of **every interaction**

#### Generation in Heartbeats and Replication

The leader includes the generation in:

- **Heartbeat messages**
- **Replication requests**
- Any control-plane communication

This allows followers to:

- Validate the leader’s authority on every message
- Detect stale leaders immediately

#### Handling Messages from a Deposed Leader

If a follower receives a message with:

- A **lower generation** than its current one

Then:

- The message is **rejected**
- No state change is applied

This rejection is intentional and necessary.

#### 11. How the Old Leader Learns It Is Deposed

When a former leader:

- Sends a heartbeat or replication request
- Receives a **rejection** due to old generation

It must:

- Recognize that its authority has expired
- **Step down immediately**
- Transition to **follower**
- Await communication from the new leader

This self-demotion is a critical safety mechanism.

### Why This Works Even Under Severe Failures

The generation clock handles:

- Network partitions
- Temporary disconnections
- Crashes and restarts
- Delayed messages
- Out-of-order delivery

Because:

- Authority is versioned
- Only the latest version is accepted
- Old versions are irrevocably invalid

No timing assumptions are required.

#### Relationship to Other Mechanisms

The generation clock works **in concert** with:

- **Heartbeats**  
  Detect liveness and trigger elections

- **Majority Quorum**  
  Ensure safe agreement on leadership and commits

- **Write-Ahead Log**  
  Persist authority and state across failures

Each solves a different aspect of the same problem:
> **Safe progress under failure**

### Key Insight (Generation Clock)

- Temporary failures are more dangerous than crashes
- Heartbeats alone cannot prevent split-brain
- Leadership must be versioned
- Generation increments on every election
- Generation is persisted durably
- Every message carries the generation
- Stale leaders are automatically rejected
- Old leaders self-demote safely

The Generation Clock ensures that **only one leader can ever be valid at a time**, even in the presence of unpredictable failures.  
This concept is central to:

- Consensus algorithms
- Replicated logs
- Strongly consistent distributed systems

[Go to the Top](#content)

---

## Chapter 10: High-Water Mark

> Defining the Boundary Between What Is Safe to Expose and What Is Not

### Context: Logs, Replication, and Divergence

In the **Leader–Followers pattern**, the leader is responsible for coordinating all state changes.

The typical flow is:

1. The leader receives a write request
2. The leader appends a new entry to its **Write-Ahead Log (WAL)**
3. The leader sends **replication requests** containing the log entry to followers
4. Followers persist the log entry
5. After sufficient acknowledgment, the leader considers the write successful

This design ensures durability and fault tolerance, but it also introduces an unavoidable reality:

> **Logs across servers are never perfectly synchronized at all times.**

Especially in distributed systems:

- Servers can fail
- Leaders can crash mid-operation
- Network delays can reorder messages
- Nodes can restart and rejoin

As a result, **log divergence is normal, not exceptional**.

### The Fundamental Problem High-Water Mark Solves

Consider two common failure scenarios:

#### Scenario 1: Leader Fails Before Replication

- Leader appends a log entry
- Leader crashes before sending it to any follower
- Only the leader has that entry

#### Scenario 2: Leader Fails After Partial Replication

- Leader appends a log entry
- Sends it to some followers, but not a majority
- Leader crashes
- Some followers have the entry, others do not

In both cases:

- Followers end up with **different log lengths**
- Some followers may have **extra entries**
- Some followers may be **missing entries**

Now comes the critical question:

> **Which part of the log is safe to expose to clients?**

Serving uncommitted or partially replicated data would violate **safety**.

### Introducing Low-Water Mark and High-Water Mark

To manage this complexity, modern distributed databases introduce two explicit markers:

#### Low-Water Mark (LWM)

- An index in the WAL
- Indicates the point **below which log entries can be safely deleted**
- Used for:
  - Disk space reclamation
  - Log compaction
- Concerned with **storage lifecycle**, not correctness

#### High-Water Mark (HWM)

- An index in the WAL
- Indicates the point **up to which log entries are safe to expose to clients**
- Concerned with **correctness and visibility**

These two markers serve **complementary but distinct purposes**.

### What High-Water Mark Means Precisely

The **High-Water Mark (HWM)** is defined as:

> **The highest log index that is known to have been successfully replicated to a Majority Quorum of followers.**

Key properties:

- HWM advances only when a **majority quorum** acknowledges a log entry
- HWM is **monotonically increasing**
- HWM never moves backward

Anything **above the HWM** is:

- Tentative
- Unsafe
- Subject to rollback or truncation

Anything **at or below the HWM** is:

- Durable across failures
- Guaranteed to survive leader changes
- Safe to serve to clients

#### Why Majority Quorum Is Essential for HWM

The HWM relies on **majority quorum** for its safety guarantees.

Because:

- Any two majorities intersect
- A new leader must be elected from a majority
- That new leader is guaranteed to have all entries ≤ HWM

This intersection property ensures:

> **Once an entry is below the HWM, it can never be lost.**

Without majority quorum:

- HWM would be meaningless
- Leaders could diverge
- Safety would be violated

#### Propagation of High-Water Mark

The leader is responsible for:

- Tracking replication progress from each follower
- Computing the current HWM
- Propagating the HWM to followers

Followers:

- Do not compute HWM independently
- Trust the leader’s HWM
- Use it to decide what data can be served

This ensures a **consistent visibility boundary** across the cluster.

#### Serving Client Requests Safely

A strict rule governs client-facing behavior:

> **All servers must only transmit to clients data derived from log entries ≤ High-Water Mark.**

This applies to:

- The leader
- All followers
- Read replicas
- Recovering nodes

Even if a follower has newer log entries:

- It must not expose them
- Until the HWM advances

This prevents:

- Dirty reads
- Inconsistent client views
- Exposure of data that might later be rolled back

### High-Water Mark During Recovery and Rejoins

When a server:

- Restarts
- Rejoins the cluster
- Recovers from a crash

It may:

- Have fewer log entries
- Have extra uncommitted entries

The HWM provides a clear rule:

- Entries above HWM are ignored or truncated
- Entries below HWM are trusted

This makes recovery:

- Deterministic
- Safe
- Bounded in complexity

### Relationship Between HWM and LWM

It is important not to confuse the two:

| Marker | Purpose | Question It Answers |
| --- | ---- | -------------- |
| Low-Water Mark | Storage management | “What can I delete?” |
| High-Water Mark | Correctness & visibility | “What can I show?” |

```sh

[ deleted | committed & visible | tentative ]
↑                    ↑
LWM                  HWM

```

- Below LWM: removed
- Between LWM and HWM: safe and visible
- Above HWM: unsafe, provisional

### Why High-Water Mark Is Critical

Without HWM:

- Followers could expose inconsistent state
- Clients could observe writes that later disappear
- Leader failover could corrupt user-visible data

With HWM:

- Visibility is globally consistent
- Failures do not violate safety
- Recovery is well-defined

### Key Insight (High-Water Mark)

- Log divergence is unavoidable in distributed systems
- Majority quorum alone is not enough; visibility must be controlled
- High-Water Mark defines the safe visibility boundary
- Only data ≤ HWM may be served to clients
- HWM is leader-computed and follower-enforced
- Together with LWM, HWM defines the full lifecycle of log entries

High-Water Mark ensures that **correctness survives crashes, restarts, and leadership changes**, which is foundational for building reliable distributed databases.

[Go to the Top](#content)

---

## Chapter 11: Paxos

[Go to the Top](#content)

---

## Chapter 12: Replicated Log

[Go to the Top](#content)

---

## Chapter 13: Singular Update Queue

[Go to the Top](#content)

---

## Chapter 14: Request Waiting List

[Go to the Top](#content)

---

## Chapter 15: Idempotent Receiver

[Go to the Top](#content)

---

## Chapter 16: Follower Reads

[Go to the Top](#content)

---

## Chapter 17: Versioned Value

[Go to the Top](#content)

---

## Chapter 18: Version Vector

[Go to the Top](#content)

---

## Chapter 19: Fixed Partitions

[Go to the Top](#content)

---

## Chapter 20: Key-Range Partitions

[Go to the Top](#content)

---

## Chapter 21: Two-Phase Commit

[Go to the Top](#content)

---

## Chapter 22: Lamport Clock

[Go to the Top](#content)

---

## Chapter 23: Hybrid Clock

[Go to the Top](#content)

---

## Chapter 24: Clock-Bound Wait

[Go to the Top](#content)

---

## Chapter 25: Consistent Core

[Go to the Top](#content)

---

## Chapter 26: Lease

[Go to the Top](#content)

---

## Chapter 27: State Watch

[Go to the Top](#content)

---

## Chapter 28: Gossip Dissemination

[Go to the Top](#content)

---

## Chapter 29: Emergent Leader

[Go to the Top](#content)

---

## Chapter 30: Single-Socket Channel

[Go to the Top](#content)

---

## Chapter 31: Request Batch

[Go to the Top](#content)

---

## Chapter 32: Request Pipeline

[Go to the Top](#content)

---

### THE END

---
