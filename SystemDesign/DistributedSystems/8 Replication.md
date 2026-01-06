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
