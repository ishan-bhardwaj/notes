# Isolation

- Defines how each transaction appears relative to other concurrent transactions.
- In distributed systems, isolation is harder because -
  - Data is sharded and replicated.
  - There is no global clock, and replicas may lag.
  - Cross-shard transactions need distributed concurrency control and commit.

## Anomalies

- **Dirty write** -
  - A transaction overwrites data that another uncommitted transaction has written.
  - This can corrupt invariants because both are updating based on uncommitted state.

- **Dirty read** -
  - A transaction reads uncommitted data written by another transaction.
  - If the writer later aborts, the reader has acted on values that never really existed.

- **Lost update** -
  - Two transactions read the same value and both write based on that value; the second write overwrites the first.
  - Example - both read `inventory=10`, subtract `2` → both write `8`; one decrement is lost.

- **Non-repeatable (fuzzy) read** -
  - A transaction reads a row twice and sees different values because another transaction updated it in between.
  - Decisions based on the first read may become inconsistent.

- **Phantom read** -
  - A transaction runs a predicate-based query (e.g., “WHERE region = 'US'”) twice and sees different sets of rows because another transaction inserted/deleted matching rows.
  - Aggregate queries (count/avg/max) can change mid-transaction.

- **Read skew** -
  - A transaction reads multiple related items that should be consistent together but are updated by another transaction in between the reads.
  - Example - reading account `A` and account `B` separately and seeing one updated, the other not, violating a logical invariant at read time.

- **Write skew** -
  - Two transactions read the same constraint-relevant data and then update disjoint rows based on that read.
  - Each individually preserves constraints, but together they violate them.
  - Classic example - two doctors, rule “at least one doctor on-call”; both see “other is on-call”, both go off-call → no one on-call.

## Isolation levels

- **Serializability** -
  - Strongest.
  - Concurrent execution is equivalent to some serial execution order of the transactions.
  - Conceptually - as if transactions ran one after another with no overlap.
  - Implementation -
    - Two-Phase Locking (2PL) with predicate locks.
    - Serializable Snapshot Isolation (SSI) on top of MVCC.
    - In distributed systems - distributed locking or timestamp ordering + validation across shards.
  - Trade-offs -
    - Strongest guarantees; safest default for correctness.
    - Highest contention; more blocking or aborts under high concurrency.
    - In distributed context, can be quite expensive (more coordination, more aborts).

- **Repeatable Read** -
  - Once a transaction reads a row, subsequent reads of that row within the same transaction see the same value.
  - Guarantees stability of individual row reads, but not necessarily the set of rows that match a predicate.
  - Implementation -
    - Lock-based - long-lived read locks on rows read.
    - MVCC-based - snapshot for each row that is stable for the transaction.
  - Distributed angle -
    - Guaranteeing repeatable read for cross-shard predicates may still allow phantoms unless predicate locking or global index locking is used.
  - Use -
    - Good when you need stable row views but can tolerate changes in the set of matching rows.

- **Snapshot Isolation (SI)** -
  - Each transaction reads from a consistent snapshot of the database as of a particular timestamp.
  - At commit, the transaction succeeds only if no concurrent transaction has modified the same rows it updated (“first-committer-wins”).
  - Implementation -
    - MVCC -
      - Multiple versions per row tagged with timestamps.
      - Transaction gets a snapshot timestamp; reads versions <= that timestamp.
      - Commit validates that no conflicting concurrent write has occurred on rows it writes.
  - Distributed angle -
    - Need consistent snapshot across shards -
      - Global timestamps (e.g., Spanner’s TrueTime).
      - Commit protocol that enforces snapshot-read + conflict-check across partitions.
  - Use -
    - Popular default (e.g., many Postgres workloads) because it gives strong “read stability” and good performance, but you must be aware of write skew.

- **Read Committed** -
  - A transaction only sees data that has been committed by other transactions at the time of each read.
  - Each read may see a different snapshot (no guarantee of repeatability).
  - Implementation -
    - Lock-based - Writers hold row locks until commit; readers only see committed rows.
    - MVCC - Each read uses the latest committed version at the moment of read.
  - Distributed angle -
    - Followers with lag can still give “less than committed” from leader’s perspective.
    - “Read committed” is a property at the transaction level; you still need replication-level consistency to avoid reading stale replicas if that matters.
  - Use -
    - Common default in many OLTP systems; good compromise for many workloads when you don’t need strong invariants across multiple reads.

- **Read Uncommitted** -
  - Allows reading data written by transactions that have not yet committed (and may abort).
  - Implementation
    - Typically - readers ignore locks or read uncommitted versions.
  - Use
    - Rarely appropriate except for -
      - Diagnostic queries.
      - Non-critical analytics where slightly “nonsense” data is acceptable for performance.

### Anomalies prevention by isolation levels

| Level              | Dirty Write | Dirty Read | Non-Repeatable | Phantom | Lost Update | Read Skew | Write Skew  |
| ------------------ | ----------- | ---------- | -------------- | ------- | ----------- | --------- | ----------- |
| Read Uncommitted   | ✗           | ✗          | ✗              | ✗       | ✗           | ✗         | ✗           |
| Read Committed     | ✓           | ✓          | ✗              | ✗       | ✗*          | ✗         | ✗           |
| Repeatable Read    | ✓           | ✓          | ✓              | ✗       | ✓*          | partial   | partial     |
| Snapshot Isolation | ✓           | ✓          | ✓              | ✓       | ✓           | partial   | ✗ (allowed) |
| Serializable       | ✓           | ✓          | ✓              | ✓       | ✓           | ✓         | ✓           |

_(*Lost update handling at Read Committed / Repeatable Read can vary; some systems add extra checks.)_
