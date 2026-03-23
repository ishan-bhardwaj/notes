# Distributed Systems

- A **distributed system** consists of multiple independent computers (nodes) connected over a network that coordinate by passing messages.
- **Goal** - act as a single coherent system despite network delays, failures, and concurrency.

- **Core challenges** -

  - No global clock - difficult to order events.
  - Partial failures - some components fail while others continue.
  - Independent execution - no centralized control.

## System Models

- Distributed systems can only make progress under certain assumptions about timing and communication.
- Two key models define those assumptions - _Synchronous_ & _Asynchronous_.

- **Synchronous system** -

  - Upper bounds exist for all critical operations -
    - Message delay `≤ Δ`
    - Process execution `≤ Φ`
    - Clock drift negligible (or global time available)
  - Enables algorithms to use _timeouts_ and _rounds_ deterministically -

    - Each round = send → deliver all → compute
    - Messages from round `r` can’t arrive in round `r+2`.

- **Asynchronous system** -
  - No time bounds at all -
    - Message may be arbitrarily delayed.
    - Processes can pause unpredictably.
    - No global time or synchronized clocks.
  - Therefore, time cannot be used for correctness reasoning.
  - Most internet-scale systems are _asynchronous_ in practice, though they often approximate synchrony using practical SLAs and timeouts.

## Failure Models

- **Fail-stop** -

  - Node halts permanently.
  - Failure is detectable by others (e.g., explicit signal).
  - Simplified model - rarely perfect but good approximation.

- **Crash (Crash-Stop / Crash-Recovery)** -

  - Node halts silently (no direct signal).
  - Others only detect failure through missing responses.
  - Crash-recovery variant allows restart with persistent state.
  - Basis for most consensus protocols (Raft, Paxos).

- **Omission Failures** -

  - Node intermittently fails to send or receive messages.
  - Possible causes - transient network partitions, queue overflow, misuse of I/O.
  - Harder to detect than crash-stop - often leads to inconsistent replicas.
  - Types -
    - Send omission - node does not send a message.
    - Receive omission - node does not process an incoming message.

- **Temporal Failures** -

  - Node produces correct results but too late to be useful.
  - Possible causes - poor algorithms, bad design, clock synchronization issues etc.

- **Byzantine Failures** -
  - Node behaves arbitrarily or maliciously, for instance, sends conflicting messages, violates protocol.
  - Requires _BFT protocols_ - tolerates up to `f` faults with `3f+1` replicas.
  - Used in blockchain, multi-organization coordination.

> [!TIP]
> Choosing the right fault model guides your algorithm and recovery design.

## Failure Detection

- **Timeouts** -

  - Simple, heuristic-based detection - if node doesn’t respond within threshold → suspect failure.
  - Trade-offs depend on timeout length -
    - Short → quicker detection, more false positives.
    - Long → stable detection, slower reaction.
  - Strategies -
    - _Adaptive timeouts_ based on observed latency.
    - _Exponential backoff + jitter_ to avoid bursts.

- **Failure Detectors** -
  - Abstract component providing “suspect/failure” info.
  - Properties -
    - _Completeness_ - Detects all actual failures eventually.
    - _Accuracy_ - Avoids falsely suspecting correct nodes.
  - Practical implementations -
    - Gossip-based (SWIM).
    - φ-accrual detectors (gradual suspicion score).

> [!NOTE]
> Perfect failure detection is impossible in asynchronous systems.

## Message Delivery Semantics

- Once you can detect or suspect failures, message correctness becomes the next concern.

- Issues -

  - Drop messages.
  - Delayed messages.
  - Deliver messages out of order.
  - Deliver duplicate messages - retrying for reliability adds risk of duplicated processing.

- Handling duplicates -
  - **Idempotent Operations** -
    - Operation can be applied multiple times with same final effect.
    - Examples
      - Idempotent - adding a value to a set - re-adding has no effect if already present.
      - Non-idempotent - incrementing a counter - repeating increases the count each time.
    - Pros - Guarantees correctness even if a message is processed multiple times.
    - Cons - not all operations can be made idempotent.
  - **De-duplication** -
    - Messages labeled with unique IDs.
    - Receivers track processed IDs and skip repeats.
    - Requires -
      - Shared ID scheme.
      - Bounded storage (TTL cleanup).
      - Trade-off between memory and correctness.

### Delivery Guarantees

| Semantics     | Meaning                         | Implementation                                               | Example Use cases                    |
| ------------- | ------------------------------- | ------------------------------------------------------------ | ------------------------------------ |
| At-most-once  | ≤1 delivery, possible loss      | Send once - ignore retries                                   | Logs, metrics                        |
| At-least-once | ≥1 delivery, duplicates allowed | Retry until acknowledged                                     | Kafka, SQS                           |
| Exactly-once  | Processed once end-to-end       | Hard in practice - requires idempotent ops or de-duplication | Financial systems, payment pipelines |

## Stateless vs Stateful Components

- **Stateless Systems** -

  - No memory of past operations.
  - Output depends only on current request (and possibly external lookups).
  - Advantages -
    - Horizontal scalability.
    - Easy deployment, no data migration.
    - Simple failure recovery.
  - Examples - API gateways, load balancers, compute microservices.

- **Stateful Systems** -
  - Persist internal or external state across requests.
  - Need careful guarantees around consistency, replication, recovery.
  - Examples - databases, caches, stream processors, consensus nodes.
  - Complications -
    - Failover and replication lag.
    - Sharding and rebalancing.

### Design Strategy

- Separate -
  - Stateless layers (logic → scalable).
  - Stateful layers (storage → durable, consistent).
- Patterns -
  - _CQRS_ - read/write separation.
  - _Event sourcing_ - log-based state reconstruction.
  - _Stateful stream processing_ - partitioned local state with checkpointing.

## ACID Transactions

- **ACID** - 
  - Atomicity, Consistency, Isolation, Durability.
  - Guarantees that transactions behave predictably under failures and concurrency.
- **Distributed** - a single logical transaction spans multiple shards/services and still must satisfy ACID end-to-end, not just within one DB node.

### Atomicity

- All sub-operations succeed or none - system never ends up “half-committed”.

- **Local mechanism** -
  - _Write-Ahead Log (WAL)_ -
    - Log changes first, then applied.
    - On crash - either rollback or redo from the log.

- **Distributed mechanism** -
  - _2PC (Two-Phase Commit), 3PC variants_.
  - _Paxos/Raft-based commit_ inside systems like Spanner/Cockroach (consensus-based atomic commit).

- **Challenges** -
  - Coordinator failure in 2PC → blocking - participants cannot decide alone safely
  - Network partitions → may prevent forming a global decision while keeping safety.

- **Workarounds** -
  - _Saga pattern_ -
    - For long-running distributed workflows.
    - Break into sequence of local transactions plus compensations.
    - Favors availability and eventual consistency.
  - _Outbox pattern_ -
    - For cross-service message consistency.
    - DB transaction writes business data + “outbox record”.
    - A reliable worker publishes messages, ensuring DB + messages are atomic locally.

### Consistency

- Each committed transaction transitions the system from one valid state to another, preserving invariants (schema + business rules).
- Examples - foreign keys, uniqueness, non-negative balances, “total debits = total credits” etc.

> [!TIP]
> ACID consistency ≠ CAP/replication consistency -
>   - ACID - “no constraints violated”.
>   - CAP/linearizability- “how operations are ordered and observed across replicas”.

### Isolation

- Concurrent transactions appear as if executed one at a time.
- Avoids interference and anomalies between transactions.

- **Techniques** -
  - _2PL (locks)_ - acquire locks, hold until commit, prevent conflicting concurrent writes.
  - _MVCC_ - multiple versions; readers see a snapshot; writers create new versions.
  - _Optimistic CC_ - transactions run without locks, validated at commit.
  - _Snapshot isolation_ - readers see a consistent snapshot; a common practical compromise.

- **Distributed complications** -
  - Replica lag - followers can return stale data; must choose which operations can read from followers.
  - Sharding - transactions span partitions; need distributed CC (locks/timestamps across shards).
  - Multi-region - higher latency, more clock skew.

### Durability
  - Once a transaction is acknowledged as committed, its effects must survive crashes.
  - **Local** - 
    - _WAL + fsync before commit_ → after crash, replay log to restore committed state.
  - **Distributed** - 
    - Quorum writes - commit only after a majority of replicas persist the log entry.
    - Synchronous replication - leader waits for replicas before ACK.
    - Checkpoints + log replay to rebuild state across nodes.

# CAP Theorem

- Impossible for a distributed system to provide all three simultaneously: Consistency, Availability, Partition Tolerance.
- Consistency (C) -
  - Every successful read returns the most recent write.
  - Different from ACID consistency.
- Availability (A) -
  - Every request receives a non-error response.
  - No guarantee it reflects the latest write.
- Partition Tolerance (P) -
  - System operates despite dropped messages or network partitions.
  - Cannot be abandoned in distributed systems.
- Forces explicit trade-offs between consistency and availability.
- Categories Based on CAP -
  - CP systems - consistent under partition, may sacrifice availability.
  - AP systems - available under partition, may sacrifice consistency.
- Trade-off During Normal Operation -
  - Latency vs Consistency trade-off exists even without partitions.
  - Synchronous replication → favors consistency → higher latency.
  - Asynchronous replication → favors lower latency → eventual consistency.

## PACELC Theorem

- Extension of CAP theorem.
- Statement -

  - P: During partition → choose between Availability (A) and Consistency (C).
  - Else (E): No partition → choose between Latency (L) and Consistency (C).

- Categories Based on PACELC -

  - AP/EL → favors Availability during partition, Latency when normal.
  - CP/EL → favors Consistency during partition, Latency when normal.
  - AP/EC → favors Availability during partition, Consistency when normal.
  - CP/EC → favors Consistency during partition, Consistency when normal.

- Design Principle -
  - Most systems prioritize either performance & availability or strict consistency.
  - Common categories → AP/EL or CP/EC.
- Implications for Distributed Databases -
  - Eventual consistency may be adopted to improve responsiveness across data centers.
  - PACELC helps understand trade-offs between Consistency, Availability, Partition Tolerance, and Latency.

# Replication

- Promotes availability.
- Storing the same piece of data in multiple nodes (replicas) to increase availability.
- Purpose -
  - Ensures system remains functional despite failures.
  - Requests can be served from other replicas if one node crashes.
- Benefits -
  - Increased availability.
  - Provides the illusion of a single copy to simplify software design.
- Complications -
  - Multiple copies must be kept in sync on every update.
  - May require significant hardware or trade-offs in other properties (e.g., performance vs consistency).

## Replication Strategies

- Pessimistic Replication -

  - Guarantees all replicas are identical from the start.
  - Behaves as if there is only one copy of the data.

- Optimistic Replication / Lazy Replication -
  - Allows replicas to diverge temporarily.
  - Guarantees eventual convergence when the system is idle or quiesced.

## Primary-Backup Replication

- A replication technique where one node is designated as primary/leader to handle all updates - other nodes are followers/secondaries handling read requests only.
- Update Propagation Techniques -
  - Synchronous Replication -
    - Leader waits for acknowledgments from all replicas before replying to the client.
    - Guarantees consistency and durability.
    - Slower write performance due to waiting on all replicas.
  - Asynchronous Replication -
    - Leader replies to client immediately after local update.
    - Increases write performance.
    - Reduces consistency and durability - stale reads possible - updates may be lost if leader crashes.
- Scalable for read-heavy workloads by adding more followers, but not very scalable for write-heavy workloads (leader is bottleneck).
  Disadvantages -
- Large number of followers can create leader network bandwidth bottleneck.
- Failover introduces downtime and potential errors.

### Failover

- When the leader node crashes, a follower takes over as the new leader.
- Approaches -
  - Manual - Operator selects new leader - safest but causes downtime.
  - Automated - Followers detect leader crash (e.g., heartbeats) and elect new leader - faster but risky.
- Examples - PostgreSQL, MySQL (support both sync and async replication).

## Multi-Primary Replication / Multi-Master Replication

- All replicas are equal and can accept write requests - updates are propagated to all replicas.
- Favors higher availability and performance over strict data consistency.
- Suitable for applications where access and speed matter more than immediate consistency (e.g., shopping carts).
- Key Differences from Primary-Backup -
  - No single leader node → requests handled concurrently by all nodes.
  - Concurrent writes can lead to conflicts due to differing request order across nodes.

### Conflict Resolution -

- Process of reconciling divergent replicas to maintain system correctness.
- Timing -
  - Eager Resolution - Resolve conflict during the write operation.
  - Lazy Resolution - Maintain multiple versions - resolve later, e.g., during a read.
- Common Approaches -
  - Client-Driven Resolution -
    - System returns multiple versions to client - client selects correct one.
    - Example - Shopping cart application.
  - Last-Write-Wins -
    - Each write tagged with timestamp - latest timestamp chosen.
    - Limitation - No global clock → can override causally dependent writes incorrectly.
  - Causality Tracking Algorithms -
    - Track causal relationships between writes.
    - Retain write that is the cause of another.
    - Concurrent writes (no causal relation) remain unresolved automatically.

# Partitioning

- Promotes scalability.
- Splitting a dataset into smaller datasets and assigning them to different nodes.
- Enables adding nodes to scale storage and processing capacity.
- Types -
  - Horizontal Partitioning / Sharding -
    - Split a table into multiple tables, each containing a subset of rows.
    - Example - Partition a student table alphabetically by surname - each shard stored on a different node.
  - Vertical Partitioning -
    - Split a table into multiple tables with fewer columns - related columns stored in separate tables.
    - Often uses normalization - can split even normalized columns.

## Horizontal Partitioning Algorithms

- Range Partitioning -

  - Split dataset into ranges based on a specific attribute - each range stored on a different node.
  - Node Mapping - System maintains a map of ranges → nodes to route requests.
  - Advantages -
    - Simple and easy to implement.
    - Efficient range queries for small ranges within a single node.
    - Easy to adjust ranges (repartition) by moving data between two nodes.
  - Disadvantages -
    - Range queries on non-partitioning attributes not supported.
    - Poor performance for large ranges spanning multiple nodes.
    - Uneven data/traffic distribution can overload some nodes.
  - Examples - Google BigTable, Apache HBase.

- Hash Partitioning -

  - Apply a hash function to an attribute - hash value determines the partition/node.
  - Example - `node = hash(surname) mod n`
  - Advantages -
    - No need to store a mapping - can compute node at runtime.
    - Uniform data distribution reduces node overload.
  - Disadvantages -
    - Range queries not possible without extra data or querying all nodes.
    - Adding/removing nodes requires repartitioning, moving significant data.

- Consistent Hashing -
  - Hash nodes to a ring `[0, L]` - each key maps to the node after hash(key) mod L.
  - Node Changes -
    - Adding a node → only affects next node on the ring.
    - Removing a node → its data transferred to next node on the ring.
  - Advantages -
    - Reduced data movement compared to standard hash partitioning.
  - Disadvantages -
    - Potential for nonuniform data distribution due to random node placement.
    - Imbalance when nodes are added/removed.
  - Mitigation - Use virtual nodes (multiple positions per physical node).
  - Examples - Apache Cassandra, Dynamo.

> [!NOTE]
> Hybrid approach - Combining range & hash partitioning can balance query efficiency and uniform data distribution.
