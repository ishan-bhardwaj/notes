# ACID Transactions

- **ACID** - 
  - Atomicity, Consistency, Isolation, Durability.
  - Guarantees that transactions behave predictably under failures and concurrency.
- **Distributed** - a single logical transaction spans multiple shards/services and still must satisfy ACID end-to-end, not just within one DB node.

## Atomicity

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

## Consistency

- Each committed transaction transitions the system from one valid state to another, preserving invariants (schema + business rules).
- Examples - foreign keys, uniqueness, non-negative balances, “total debits = total credits” etc.

> [!TIP]
> ACID consistency ≠ CAP/replication consistency -
>   - ACID - “no constraints violated”.
>   - CAP/linearizability- “how operations are ordered and observed across replicas”.

## Isolation

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

## Durability
  - Once a transaction is acknowledged as committed, its effects must survive crashes.
  - **Local** - 
    - _WAL + fsync before commit_ → after crash, replay log to restore committed state.
  - **Distributed** - 
    - Quorum writes - commit only after a majority of replicas persist the log entry.
    - Synchronous replication - leader waits for replicas before ACK.
    - Checkpoints + log replay to rebuild state across nodes.

## 2-Phase Commit (2PC)

- Achieve distributed atomicity - either every participant commits or all abort, despite unreliable networks.
- Roles -
  - **Coordinator** - Drives the protocol, knows all participants, logs global decisions.
  - **Participants** - Own local data, can prepare/commit/abort, log local decisions.

> [!NOTE]
> One participant can also act as coordinator in small systems.

- Key idea -
  - One round to ask “can you commit?” & one round to broadcast “commit/abort”.
  - Local WAL at each node ensures crash recovery of prepared/committed state.

- **Blocking nature** -
  - Once participants are prepared (`YES`), they cannot unilaterally change their mind without risking inconsistent outcomes.
  - If the coordinator is down or lost, prepared participants must wait indefinitely → 2PC is blocking.

- **Safety vs liveness** -
  - Safety - 2PC guarantees atomicity (no split-brain commit/abort) as long as logs are durable.
  - Liveness - not guaranteed; the system can stall if the coordinator or network is in a bad state.

- **Usage** -
  - Widely used in -
    - XA transactions (resource managers + transaction manager).
    - Some JDBC/JTA stacks, RDBMS integration.
  - For internet-scale microservices, cross-service 2PC is usually avoided due to latency, coupling, and blocking behavior.

### 2PC Phases

- Phase 1 - **Voting (Prepare)** -
  - Coordinator sends transaction or `PREPARE(Tx)` to participants.
  - Participants -
    - Execute operations up to _commit point_ (evaluate constraints, acquire locks, write tentative changes to WAL).
    - Write `prepared(Tx)` to disk, guaranteeing they can commit later even after a crash.
    - Respond -
      - `YES` if ready to commit.
      - `NO` on error or constraint violation.
  - Can combine with 2PL so that once prepared, participants hold locks until commit/abort.

- Phase 2 - **Commit / Abort** -
  - Coordinator collects votes -
    - If all `YES` - logs `global-commit(Tx)`, sends `COMMIT` to all.
    - If any `NO` or timeout - logs `global-abort(Tx)`, sends `ABORT`.
  - Participants -
    - On `COMMIT` - Flip state from prepared → committed (ideally minimal work, e.g., flip a bit), make changes visible, release locks.
    - On `ABORT` - Rollback tentative state, release locks.
    - Send ACKs to coordinator.

### Failure Handling in 2PC

- Participant fails during voting -
  - Coordinator times out → assumes `NO` → aborts transaction.
  - Safe - no participant has committed.

- Participant fails after `YES`, before commit -
  - After recovery, participant sees `prepared(Tx)` in WAL but no final decision.
  - It must contact coordinator and block until it gets commit/abort.
  - Atomicity preserved - it never guesses on its own.

- Network failures -
  - Treated like node failures - coordinator uses timeouts; participants don’t infer decisions just from silence.

- Coordinator failure -
  - Before logging decision - Prepared participants block; they cannot safely choose commit/abort.
  - After logging decision, before sending to all - On recovery, coordinator replays log and re-sends decision.
  - Disk failure of coordinator may require manual recovery, because its log is the authoritative record of the global decision.


## 3-Phase Commit (3PC)

- Problem with 2PC - Coordinator failure can leave participants in “prepared” state with no way to decide, causing indefinite blocking.
- Key idea of 3PC - Split 2PC’s prepare into two sub-phases so participants can infer safe decisions in more cases.

- Phases -
  - **CanCommit?** - coordinator asks if participants could commit.
  - **PreCommit** - if all are willing, coordinator sends “prepare-to-commit” and waits for ACKs.
  - **Commit** - coordinator sends final commit.

- **Effect** -
  - If participants reach pre-commit, they know all others are also ready to commit.
  - If coordinator fails after pre-commit, participants can independently decide to commit.
  - If they never receive pre-commit, they can safely abort.

- **Benefits** - 
  - Non-blocking under some failure patterns where 2PC blocks.
  - Better liveness.

- **Drawbacks** - safety under partitions -
  - Network partition example -
    - Coordinator sends pre-commit to some participants, not others, then fails.
    - Some commit (saw pre-commit), others abort (didn’t).
    - Atomicity is violated; system ends up inconsistent.
  - 3PC assumes partial synchrony; under real-world partitions it can break atomicity.

> [!WARNING]
> 3PC improves liveness but sacrifices safety under certain failures.

> [!NOTE]
> Rarely used in practice; instead, real systems move to quorum/consensus-based commit.

## Quorum-Baesd Commit Protocol

- Use _voting_ + _overlapping quorums_ to decide commit/abort in a way that preserves atomicity even under failures and partitions.
- Roles -
  - **Participants** - 
    - Each node that executes part of a distributed transaction is assigned some number of votes $V_i$.
    - Total votes - $V = ∑_i V_i$
  - **Quorums for commit/abort** -
    - Commit quorum $V_c$ - minimum votes required to safely decide _commit_.
    - Abort quorum $V_a$ - minimum votes required to safely decide _abort_.

- **Key safety rule** -
  - $V_a + V_c > V$
  - Ensures a _commit_ quorum and an _abort_ quorum can never be _disjoint_ i.e. any two quorums overlap in at least one site.

- **Intuition** -
  - A transaction can only commit if enough sites agree (commit quorum).
  - It can only abort if enough sites agree (abort quorum).
  - Because quorums overlap, you cannot have one partition commit while another aborts the same transaction.

- **Benefits over 2PC/3PC** -
  - Moves the decision authority from a single coordinator to quorum agreements.
  - Even if the original coordinator fails or the network partitions, any node with enough information and quorum support can drive termination.
  - Preserve atomicity (safety) while reducing blocking probability and handling partitions more gracefully than 2PC/3PC.

### Commit and abort rules

- Given total votes $V$, commit quorum $V_c$, abort quorum $V_a$, and rule $V_a + V_c > V$ -
  - **Before commit** -
    - A transaction must collect at least $V_c$ votes from sites that are compatible with committing (prepared/ready/commit states).
    - Those votes represent the subset of the system that has agreed to commit.

  - **Before abort** -
    - A transaction must collect at least $V_a$ votes from sites that are compatible with aborting (not prepared, explicitly aborting, or waiting).
​
  - **Safety property** -
    - Because any commit quorum and abort quorum overlap, it is impossible for one partition to gather a full commit quorum while another gathers a full abort quorum for the same transaction.
    - Therefore, commit and abort decisions cannot diverge across the system.

  - **Liveness (high-level)** -
    - As long as there is a partition with enough votes to form one of the quorums (commit or abort), that partition can make forward progress and terminate the transaction.
    - Very small or fragmented partitions without sufficient votes will block, but they also cannot contradict decisions made by larger partitions.

### Normal commit protocol (no failures)

- Concrete behavior is similar to a multi-phase commit (often 3PC-like), but with quorum thresholds -
  - **Execution / can-commit** -
    - Transaction’s sub-operations run at each site.
    - Sites decide if they could commit based on local checks.

  - **Prepare / pre-commit phase** -
    - Coordinator sends a _prepare_ or _pre-commit_ message.
    - Sites that can commit move to a prepared / pre-commit state and log that state durably.
    - They send acknowledgments (votes) back to the coordinator.

  - **Commit decision** -
    - Coordinator counts votes -
      - If it gathers at least $V_c$ votes from sites in pre-commit/ready-to-commit state - 
        - It can safely decide commit and broadcast that decision.
      - If it cannot reach $V_c$, but can reach $V_a$ consistent with abort -
        - It can decide abort and broadcast that decision.
​    - Sites apply the decision and release resources.

> [!TIP]
> The coordinator does not need every site; only quorum thresholds matter for safety, unlike 2PC where any single non-responding prepared participant can block.
​
### Termination protocols

- **Trigger** -
  - Original coordinator is unreachable or suspected failed.
  - Some sites are stuck in prepared/waiting states and want to complete the transaction.

- **Surrogate coordinator election** -
  - One of the sites (e.g., any that notices the stall) is elected as a surrogate coordinator.

- **State collection** -
  - Surrogate queries other sites for their local state for this transaction -
    - States typically include - _initial, waiting, prepared/pre-commit, committing, committed, aborting, aborted._
​
- **Decision rules (simplified)** -
  - If any site reports committed - Surrogate adopts commit as final decision and orders others to commit.
  - Else if any site reports aborted - Surrogate adopts abort as final decision and orders others to abort.
  - Else if a set of sites in prepared/pre-commit state collectively holds at least $V_c$ votes - Surrogate decides commit and sends commit to all.
  - Else if a set of sites in non-prepared / waiting states collectively holds at least $V_a$ votes - Surrogate decides abort and sends abort to all.
  - Else - There is not enough information/votes to decide safely → those sites must block until more information or connectivity is available.

### Merge protocols

- **Scenario** -
  - Network partitions have produced multiple groups of sites with partial states.
  - After the partition heals, these groups must reconcile to a single decision.

- **Process** -
  - Each partition selects a representative (local leader).
  - Leaders exchange their transaction views and run logic equivalent to the termination protocol -
    - If any partition has already committed/aborted → global decision is that outcome.
    - Otherwise, leaders check whether combined votes across partitions can form a commit or abort quorum, using the same quorum rules.
  - Final decision is propagated to all sites so state converges.

- **Result** - despite partitions and arbitrary failure interleavings, the system converges to one decision per transaction, thanks to quorum overlap.
​

### Example with 3 sites

- Assume 3 participants with one vote each - $V = 3$, commit quorum $V_c = 2$, abort quorum $V_a = 2$.
​ - Safety condition - $V_a + V_c = 4 > V = 3$
​
- **Case - network partition** -
  - Partition A - sites S1, S2 (2 votes).
  - Partition B - site S3 (1 vote).
  - Partition B (1 vote) - Cannot reach $V_c = 2$ or $V_a = 2$ → cannot commit or abort; must block.
  - Partition A (2 votes) -
    - Can form commit or abort quorum (depending on local states).
    - If A decides abort (has $V_a = 2$) → global abort is safe, because no other partition can form a commit quorum disjointly.

- After partition heals, merge protocol communicates A’s abort decision to S3; S3 aborts too, keeping atomicity -
  - No execution path allows one side to commit with quorum 2 while the other side simultaneously aborts with quorum 2, because those quorums would overlap.

### Relationship to quorum-based replication and consensus

- **Quorum-based replication** -
  - Similar math - define read quorum $V_r$, and write quorum $V_w$ such that -
    - $V_r + V_w > V$ and $V_w > V / 2$
  - Ensures - 
    - Reads intersect writes → see at least one up-to-date replica.
    - Writes intersect writes → no conflicting concurrent writes.
  
- **Quorum-based commit vs replication** -
  - Commit protocol uses commit/abort quorums for decision; replication uses read/write quorums for data visibility.
  - Both rely on overlapping quorums to enforce one-copy serializability: the system behaves as if there is a single copy of data despite replication.

- **Consensus** -
  - Quorum-based commit can be seen as a transaction-level application of consensus ideas -
    - The commit/abort decision is a value that must be agreed on by a majority/quorum.
  - Modern systems often rephrase this as “run consensus (Paxos/Raft) on the commit record”, which is conceptually similar.

### Properties and trade-offs

- **Safety (atomicity)** -
  - Achieved by overlapping commit and abort quorums $V_a + V_c > V$.
  - Guarantees no split-brain commit vs abort decisions.

- **Liveness** - 
  - Better than 2PC - Large partitions that can form a quorum can make progress without the original coordinator.
  - But still can block in extreme scenarios -
    - Very small partitions.
    - Constantly changing failures/partitions.

- **Complexity** -
  - More complex than vanilla 2PC -
    - Weighted voting.
    - Termination and merge protocols.
    - State tracking across partitions.
  - Conceptually close to using a consensus algorithm directly.

- **Tuning bias** -
  - Choice of $V_c$ and $V_a$ can bias toward commit or abort under partial failures - 
    - Larger $V_c$, smaller $V_a$ → easier to abort than commit.
    - Symmetric quorums → neutral behavior.

## Long-Lived Transactions and Sagas

- Achieving full isolation between transactions is expensive -

  - Systems must hold locks for long durations or abort transactions to maintain safety.
  - Long transactions increase the impact of these mechanisms on throughput.

- **Long-Lived Transactions (LLT)** -

  - Transactions with a duration of hours or days.
  - Can occur due to -
    - Processing large datasets.
    - Requiring human input.
    - Interacting with slow third-party systems.
  - Examples -
    - Batch jobs generating large reports.
    - Insurance claims requiring multi-stage approvals.
    - E-commerce orders spanning multiple days.
  - Running LLTs using traditional concurrency mechanisms is inefficient because resources must be held for long periods.

- **Sagas** -
  - A saga is a sequence of transactions `T1`, `T2`, …, `TN`.
  - Transactions can interleave with other transactions.
  - Guarantees atomicity: either all transactions succeed, or none do.
  - Each transaction `Ti` has a compensating transaction `Ci` executed if rollback is needed.
  - Benefits of Sagas -
    - Useful in distributed systems where traditional distributed transactions are expensive or unavailable.
    - Sagas provide atomicity while improving availability and performance.
    - Systems remain loosely coupled and resilient.
  - Example Scenario - E-commerce Order -
    - Steps - credit card authorization, inventory check, item shipping, invoice creation.
    - Using a distributed transaction: failure of one component (e.g., payment system) could halt the whole process.
    - Using a saga - model each step as a transaction with a compensating transaction -
      - Example - debiting a bank account has a refund as the compensating transaction.
      - If a step fails, previously executed transactions are rolled back via compensating transactions.
  - Isolation Considerations in Sagas -
    - Some scenarios require isolation to prevent interference between concurrent transactions.
    - Example - two orders `A` and `B` for the same product -
      - `A` reserves the last item.
      - `B` fails due to zero inventory.
      - `A` later fails due to insufficient funds.
      - Compensating transactions return the reserved item.
    - Outcome - order `B` was rejected unnecessarily.
  - Providing Isolation at the Application Layer -
    - Semantic locks - signal that data is in process and should not be accessed. Locks are released by the final transaction.
    - Commutative updates - updates that yield the same result regardless of execution order.
    - Re-ordering the saga structure -
      - Introduce a pivot transaction that separates critical transactions from others.
      - Transactions after the pivot cannot be rolled back, ensuring serious operations (like increasing account balance) remain safe.
  - Trade-offs -
    - Techniques to provide isolation in sagas introduce complexity.
    - Developers must consider all possible failure scenarios.
    - Choosing between saga transactions and datastore-provided transactions requires weighing performance, availability, and complexity.
