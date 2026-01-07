# Consistency in Distributed Systems

- Consistency ensures that all replicas of data agree, or reads return the most recent writes.
- Stronger consistency → more constraints on what reads can return; weaker consistency → more freedom but better availability/latency.
- Consistency Spectrum - 
    - Eventual (Weakest) → Causal → Sequential → Strict/Linearizable (Strongest)
- ACID vs CAP Consistency -
    - ACID - Intra-database / intra-transaction - preserve invariants like uniqueness, foreign keys, balance ≥ 0.
    - CAP - Inter-replica - how writes become visible across nodes; e.g., linearizable vs eventual.

## Eventual Consistency

- If no new writes happen, all replicas will eventually converge to the same value.
​- In the meantime, different replicas may return different values for the same key.

- **Properties** -
    - Writes can be accepted locally and propagated asynchronously via gossip, anti-entropy, or background replication.
    - Reads are cheap and fast but may be stale or inconsistent.
    - Conflicts (concurrent writes to different replicas) are resolved via -
        - Last-write-wins (timestamps).
        - Version vectors / vector clocks + application merge.
        - CRDTs for automatically convergent data types.

- **Pros** -
    - High availability - replicas can continue serving reads/writes during partitions.
    - Low latency - no need to coordinate synchronously with other replicas for every write.
​
- **Cons** -
    - Clients can see -
        - Stale data.
        - Non-monotonic reads (see new then old).
        - Missing writes that have not propagated yet.
    - Must push complexity to the application to handle anomalies and conflict resolution.

- **Typical use cases** -
    - DNS - updates gradually propagate; slight staleness is acceptable.
    - Object stores / key-value systems when used as “mostly append” or “read-mostly” (e.g., S3-style metadata scenarios, shopping carts, counters).
    - Logs, metrics, analytics aggregations, recommendation features.

## Causal Consistency

- If operation `B` causally depends on `A` (e.g., write `B` after reading `A`, or reply to a post), then every process must see `A` before `B`.
- Operations that are not causally related (concurrent) may be seen in different orders by different clients.
- Causal consistency guarantees - if `A` → `B`, no one will see `B` without seeing `A` first.

- **Properties** -
    - Prevents “nonsense” views like -
        - Seeing a comment that replies to a post that “doesn’t exist” yet.
        - Seeing a “fix” before seeing the bug it fixes.
    - Does not impose a single total order on all operations -
        - Concurrent writes can be seen in different orders, which reduces coordination cost.
​
- **Implementation** -
    - Track causal dependencies using -
        - Version vectors or vector clocks attached to updates.
        - Per-client session metadata that encodes all updates the client has seen.
    - A replica delays making `B` visible until it has applied all of `B`’s causal predecessors.
​
- **Pros** -
    - More intuitive behavior for users - cause always before effect.
    - Weaker than linearizability but much stronger than raw eventual consistency.

- **Cons** -
    - Metadata overhead - vectors grow with number of replicas/clients (mitigated via compression techniques).
    - More complex storage and replication logic than simple last-write-wins.

- **Typical use cases** -
    - Social features -
        - Posts and comments, replies, likes.
        - Collaborative editing where you preserve causality but accept that concurrent edits may be merged.
    - Messaging / chat where you want at least causality to hold for threads.

## Sequential Consistency

- All operations appear to execute in some global sequence that is consistent with each individual client’s program order.
​- It does not require that this order matches real-time order.

- **Example** -
    - Client `A` - write `x=1`; then write `x=2`.
    - Client `B` - write `x=3`.
    - A sequentially consistent system must pick some total order like -
        - `A1` (`x=1`) → `A2` (`x=2`) → `B1` (`x=3`), or
        - `B1` → `A1` → `A2`, etc.
        - But `A1` must appear before `A2` everywhere (respect `A`’s program order).

- **Properties** -
    - Stronger than _causal/even eventual_ - there is a total order that all processes agree on.
    - Weaker than _linearizability_ - the chosen order may violate real-time constraints (a later operation may appear before an earlier one if they are not causally linked).
​
- **Implementation** -
    - Typically uses some form of serialization -
        - Single sequencer that orders all operations.
        - Token or primary that dictates global ordering.
    - Can be implemented without precise physical clocks; logical ordering is enough.

- **Pros** -
    - Simple mental model - “there is one global log of operations; each client’s actions appear in it in program order.”
    - Easier to reason about than weaker models; still sometimes cheaper than linearizability.

- **Cons** -
    - Still requires global coordination or a central sequencer, especially at high write volumes.
    - Does not respect real-time, so it is not suitable when “return time” semantics matter.

- **Typical use cases** -
    - Conceptually important in memory models and some replicated systems.
    - In practice, many systems either go all the way to linearizable or drop down to causal/eventual.

## Strict Consistency / Linearizability

- Strict consistency (theoretical) assumes a single global clock; any read returns the value of the most recent write according to that clock.
- Linearizability is the practical version -
    - Each operation appears to take effect at some instant between its invocation and response.
    - Respect real-time - if `op1` finishes before `op2` starts, every node must see `op1` before `op2`.
​
- **Properties** -
    - Appears like a single, correct copy of the data:
        - If a write returns success, all subsequent reads (from any node) see that write or a later one.
    - Compositional - if each object is linearizable, the entire system behaves in an intuitive way.

- **Implementation** - 
    - Common approaches -
        - **Leader + quorum replication** -
            - Leader serializes writes.
            - Commit only when written to a quorum of followers (e.g., majority).
            - Reads go to leader or quorum to avoid stale data.
    - **Consensus protocols (Paxos/Raft)** -
        - Replicas act as a replicated state machine; log entries agreed via consensus.
    - **Time-based (TrueTime)** -
        - Globally synchronized clocks with bounded uncertainty; transactions commit at timestamps that reflect a consistent global order (Spanner).
​
- **Challenges** -
    - Network delays and partitions -
        - To maintain linearizability, node must often wait for quorum acknowledgments; under partitions, it may have to reject or block writes.
    - Latency -
        - Each write incurs at least one extra RTT to other replicas; across regions this becomes user-visible.
    - Availability (CAP) -
        - Under partitions, cannot keep both linearizability and full availability: must choose CP behavior.

- **Typical use cases** -
    - Banking / financial ledgers - correctness of balances and transfers.
    - Authentication, authorization, password and token updates.
    - Strongly consistent configuration / feature flags where stale configs would cause correctness issues.
    - Systems like Google Spanner, TiDB, CockroachDB provide linearizable or externally consistent transactions for many operations.
