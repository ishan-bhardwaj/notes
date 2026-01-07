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

## Serializability

- **Goal** -
  - Concurrent execution behaves as if transactions ran one-by-one, with no overlap.
  - The actual interleaving is allowed, but the final outcome and all observable effects match some serial schedule.

- **Schedules** -
  - A schedule is an interleaving of operations (reads/writes/commits) from multiple transactions.
  - A schedule is serial if it runs all operations of one transaction before another.
  - A schedule is serializable if its effect is equivalent to a serial schedule (possibly in a different order).

- **Types** -
  - View Serializability - 
    - Schedules read and write the same data values as a serial schedule.
    - More general but expensive to check (NP-hard), mostly theoretical.
  - Conflict Serializability - 
    - Conflicting operations between transactions are ordered the same as in a serial schedule.
    - Cheap to check; used in practice.

### Conflict serializability

- **Two operations conflict if** -
  - They belong to different transactions.
  - They access the same data item.
  - At least one is a write.

- **Types** -
  - Read–write - `T1`: `R(x)` and `T2`: `W(x)`
  - Write–read - `T1`: `W(x)` and `T2`: `R(x)`
  - Write–write - `T1`: `W(x)` and `T2`: `W(x)`

> [!TIP]
> If operations don’t conflict (e.g., read–read, or different keys), their order can be swapped without changing the result.

- **Precedence graph** -
  - To test conflict serializability -
    - Build a precedence graph (serialization graph) -
      - Nodes = transactions.
      - Edge `Ti` → `Tj` if there exists a conflicting pair where -
        - `Ti`’s operation comes before `Tj`’s operation in the schedule.
    - If the graph has no cycles, the schedule is conflict-serializable.
    - If there is a cycle, no equivalent serial order exists respecting those conflicts.

- **How systems use this** -
  - Two styles -
    - Prevent cycles (pessimistic) -
      - Concurrency control (locks / timestamp ordering) ensures that generated schedules never form a cycle.
    - Allow then check (optimistic) -
      - Let transactions run, then at commit -
        - Validate read/write sets and abort transactions that would form cycles.

### View serializability

- Two schedules are view-equivalent if -
  - They read the same initial values.
  - Every read sees the same source write.
  - Final writes produce the same final values.

- A schedule is view-serializable if view-equivalent to some serial schedule.
- Includes more schedules than conflict-serializability, but -
  - Checking view serializability is expensive (NP-complete).
  - Systems usually target conflict-serializability or a stronger condition.

## Pessimistic concurrency control (2PL family)

- Pessimistic CC assumes conflicts are likely; it blocks them up front using locks.
- Pessimistic CC is usually chosen for high-contention workloads where aborts are expensive.

### 2-Phase Locking (2PL)

- **Locks** -
  - Shared(`S`) lock for reads -
    - Multiple transactions can hold S on the same item.
  - Exclusive(`X`) lock for writes -
    - Only one transaction can hold `X`.
    - `X` conflicts with both `S` and `X`.

- **Lock compatibility** -
  - `S` + `S` → OK
  - `S + X` or `X + S` or `X + X` → block.

- **Two phases** -
  - Expanding (growing) phase -
    - Transaction acquires locks; no locks are released.
  - Shrinking phase -
    - Transaction releases locks; cannot acquire new locks.

- **Guarantee** -
  - Schedules produced by (basic) 2PL are conflict-serializable.
  - Serialization order corresponds to the order in which transactions finish acquiring locks (end of growing phase).

- **Predicate locking** -
  - Lock not just single rows but ranges / predicates (“all rows with salary > 100K”) to prevent phantom anomalies.
  - Without predicate locks, 2PL may still see phantoms when new rows appear that satisfy the predicate.

- **Variants** -
  - Strict 2PL -
    - All write locks are held until commit/abort.
    - Prevents dirty reads and simplifies recovery (no one sees uncommitted writes).
  - Strong Strict 2PL (SS2PL) -
    - Often means both read and write locks are held until commit:
    - Very strong guarantees but more blocking.

- **Deadlocks and handling** -
  - Locks introduce deadlocks -
    - Deadlock - `T1` waits on lock held by `T2`, and `T2` waits on lock held by `T1` (or more complex cycles).
    - Handling strategies -
      - Prevention -
        - Enforce global ordering on lock acquisition (e.g., lock by key order).
        - Use protocols like wait-die / wound-wait based on timestamps.
      - Detection -
        - Maintain waits-for graph; periodically detect cycles and abort one transaction.
      - Timeout -
        - Abort if a transaction waits too long (simpler but less precise).

## Optimistic concurrency control (OCC)

- OCC assumes conflicts are rare; transactions run without locks then validate at commit.
- OCC is especially used in MVCC-based systems and high-concurrency OLTP/HTAP systems.

- **Phases** -
  - Begin -
    - Assign a unique start timestamp to the transaction.
  - **Read & Modify (working phase)** -
    - Reads from the database and a local write-set (private workspace).
    - Writes are buffered locally, not visible to others.
  - **Validate & Commit/Rollback** -
    - At commit, transaction -
      - Gets a finish (commit) timestamp.
      - Checks whether its read/write sets conflict with other transactions’ writes that committed in the meantime.
    - If conflict -
      - Abort and retry.
    - Else -
      - Apply buffered writes atomically to the database.

- **Validation strategies** -
  - Version checking -
    - Each data item has a version or timestamp.
    - Transaction records versions it read.
    - At commit, commit only if no version changed since read; otherwise abort.
  - Timestamp ordering / read-write set validation -
    - Maintain read-set and write-set for each transaction.
    - At commit, ensure -
      - No committed transaction with overlapping writes has a timestamp between this transaction’s start and finish and conflicts with its reads.
    - If conflict found, abort and retry.

- **Pros** -
  - No blocking during read phase; great for mostly-read workloads with few conflicts.
  - Simpler reasoning for readers; they never block on locks.

- **Cons** -
  - Wasted work when conflicts are frequent (transactions do full work then abort).
  - Validation and commit section is a critical point that must be atomic; can become a bottleneck.

## Snapshot Isolation (SI) via MVCC

- Snapshot Isolation is not full serializability but is widely used because it gives strong guarantees with good performance.

### MVCC

- Each logical row has multiple versions, each tagged with -
  - The transaction (or timestamp) that created it.
- Updates -
  - Create a new version instead of overwriting in place.
- Reads -
  - Choose a version appropriate for the transaction’s snapshot (e.g., last committed version <= snapshot timestamp).

- **Snapshot Isolation guarantee** -
  - At transaction start -
    - Assign a snapshot timestamp; record the set of active transactions.
  - Reads -
    - See a consistent snapshot -
      - Only versions committed before the snapshot timestamp.
      - Ignore versions from transactions active at snapshot start.
      - If the transaction has updated the item, read its own version.
    - This prevents -
      - Dirty reads.
      - Non-repeatable reads (all reads see the same snapshot).
  - Writes -
    - At commit, check if any concurrent transaction has updated the same items since the snapshot:
    - If yes → conflict → abort (prevents lost updates).
    - If no → install new versions and commit.

- **Anomalies under SI** -
  - SI prevents many anomalies -
    - No dirty reads / writes
    - No non-repeatable reads.
    - No phantoms within the snapshot (new rows after snapshot are invisible to the reader).
    - No lost updates on rows you update.
  - But it is not serializable -
    - Write skew is still possible -
      - Two transactions read overlapping data at the same snapshot.
      - Each updates disjoint rows based on the assumption the invariant holds.
      - Both commit, breaking the invariant, even though no single row appears to conflict.
  - Example - on-call doctors (two rows, each doctor’s on-call flag) -
    - Both `T1` and `T2` see “at least one doctor on-call.”
    - Both set their own row to off-call and commit (disjoint rows).
    - Invariant “at least one doctor on-call” is violated; no serial order could produce this outcome.

### Serializable Snapshot Isolation (SSI)

- SSI is an optimistic algorithm that upgrades SI to full serializability by detecting and aborting non-serializable patterns.

- **Key insight** -
  - All non-serializable executions under SI share a characteristic:
    - There exists a cycle in the precedence graph with at least two consecutive read–write (rw) dependency edges between concurrently active transactions.
  - RW-dependency -
    - `T0` reads a version of `x`.
    - `T1` later writes a new version of `x` that comes after the version read by `T0`.
    - This gives an edge `T0` → `T1` (“`T0` read stale relative to `T1`’s write”).

- **SSI algorithm** (high level) -
  - For each transaction `T`, track -
    - `T.inConflict` – whether there is an incoming rw-dependency (someone wrote after `T`’s read).
    - `T.outConflict` – whether `T` wrote after someone else’s read (outgoing `rw`-dependency).
  - Mechanism -
    - Reads -
      - When `T` reads `x`, mark “`T` has read version `v`”.
      - If later a writer `U` writes a newer version of `x`, create `rw`-dependency `T` → `U`, set -
        - `T.outConflict` = `true`, `U.inConflict` = `true`.
    - Writes -
      - Use non-blocking `SIREAD` locks or similar markers on data items read by others.
      - When a transaction writes an item, it checks `SIREAD` info to detect if it is creating an rw-dependency edge with a concurrent reader.

  - Abort rule -
    - If a transaction has both `inConflict` and `outConflict` set, it is a potential middle of a dangerous cycle.
    - Abort one transaction in such patterns to break the cycle.

- **Properties** -
  - Achieves full serializability on top of snapshot isolation.
  - Still optimistic -
    - Reads do not block writes; SIREAD locks signal conflicts but are not blocking locks.
  - May cause false positives -
    - It may abort some transactions that would have been serializable, but this keeps the algorithm simple and avoids full cycle detection.
