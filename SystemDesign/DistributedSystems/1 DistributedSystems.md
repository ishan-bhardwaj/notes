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
> Perfect failure detection impossible in asynchronous systems.

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

- ACID Transactions - Set of properties that guarantee expected behavior of database transactions during errors or failures.

- **Atomicity (A)** -
  - Either all sub-operations succeed or none.
  - Mechanisms -
    - Local DB - _Write-ahead log (WAL)_
    - Distributed - _Two-Phase Commit (2PC), Paxos Commit_
  - Challenges -
    - Coordinator failure can block.
    - Network partition invalidates global atomic commit.
  - Alternatives -
    - _Saga pattern_ for long-running distributed workflows.
    - _Outbox pattern_ for cross-service message consistency.
- **Consistency (C)** -

  - Transaction only transitions database from one valid state to another.
  - Maintains application-specific invariants (e.g., foreign key constraints).
  - Note - this is different from distributed systems consistency (CAP theorem).

- **Isolation (I)** -

  - Concurrent transactions appear as if executed one at a time.
  - Prevents interference and anomalies between transactions.
  - Techniques - _MVCC, locks, optimistic concurrency, snapshot isolation_.
  - Distributed complication -
    - Replica lag can expose stale reads.
    - Sharded systems require distributed concurrency control.

- Durability (D) -
  - Ensures committed data persists across failures.
  - Local - _WAL, fsync before commit_
  - Distributed - _quorum writes, synchronous replication_
