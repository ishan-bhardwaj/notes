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


### 3-Phase Commit (3PC)

- 3PC addresses the main bottleneck of 2PC, which is coordinator failures leading to blocked participants.
- In 2PC, participants cannot safely take the lead because they do not know the state of other participants.
- Coordinator failure during the commit phase can leave participants unable to decide without waiting for the failed participant or coordinator to recover.
- Tackling 2PC with 3PC -
  - Split the first round (voting phase) into 2 sub-rounds.
  - Coordinator first communicates the vote results to participants and waits for acknowledgment.
  - Then, coordinator sends commit or abort instructions.
  - Participants can complete the protocol independently if the coordinator fails.
  - This improves availability and reduces the coordinator as a single point of failure.
- Benefit of 3PC -
  - Participants can commit if they receive a prepare-to-commit message, knowing all participants voted Yes.
  - Participants can abort if they do not receive a prepare-to-commit message, knowing no participant has committed.
  - Coordinator failures do not block progress.
  - Overall availability increases.
- Cost of 3PC -
  - Correctness can be compromised in certain failure scenarios.
  - Network partitions are a key vulnerability.
- Network partition failure example -
  - Coordinator sends prepare-to-commit to some participants and then fails.
  - Participants that received the prepare-to-commit may commit.
  - Participants that did not receive the prepare-to-commit abort.
  - When the network partition heals, the system can be left in an inconsistent state.
  - Atomicity of the transaction is violated.
- Conclusion -
  - 3PC ensures liveness, meaning the protocol always makes progress.
  - 3PC sacrifices safety, as atomicity can be violated under certain failures.

### Quorum-Based Commit Protocol

- Quorum-based commit protocol addresses the main issue of 3PC: network partitions that can lead to inconsistent states.
- In 3PC, participants might take the lead during a partition without full knowledge, potentially causing a split-brain scenario.
- Coping with network partitions -
  - Use a quorum to ensure safety.
  - Define a commit quorum (`VC`) and an abort quorum (`VA`).
  - A node can commit only if a commit quorum is formed.
  - A node can abort only if an abort quorum is formed.
  - Values of `VA` and `VC` must satisfy: `VA + VC > V`, where `V` is the total number of participants.
  - This ensures that two conflicting decisions cannot be made on separate sides of a partition.
- Sub-protocols in quorum-based commit protocol -
  - Commit protocol - for starting a new transaction.
  - Termination protocol - for handling network partitions.
  - Merge protocol - for recovering after a network partition.
- Commit protocol -
  - Similar to 3PC.
  - Coordinator waits for `VC` acknowledgments at the end of phase 3 to commit.
  - If a network partition prevents the coordinator from completing the transaction, participants follow the termination protocol.
- Termination protocol -
  - A surrogate coordinator is elected from the participants.
  - The coordinator queries participants for their transaction status.
  - If at least one participant has committed or aborted, the coordinator enforces the same decision.
  - If participants are in prepare-to-commit state and at least VC participants are waiting, the coordinator sends prepare-to-commit messages.
  - If no participant is in prepare-to-commit and at least VA participants are waiting, the coordinator sends prepare-to-abort messages.
  - Waits for acknowledgments and completes the transaction similar to the commit protocol.
- Merge protocol -
  - Leader election occurs among leaders of the partitions being merged.
  - Executes the termination protocol to reconcile the transaction state across partitions.
- Example -
  - System with `V=3` participants.
  - Quorum sizes - `VA=2`, `VC=2`
    - During a network partition:
    - Left partition cannot form commit quorum.
    - Right partition can form abort quorum and aborts the transaction.
  - When the partition heals, the merge protocol ensures consistency by aborting the transaction on the left side as well.
  - Quorum sizes can be tuned to bias toward commit or abort in the presence of partitions.
- Conclusion -
  - Quorum-based commit protocol ensures safety (atomicity).
  - Liveness is not guaranteed in extreme scenarios (e.g., continuous small partitions).
  - More resilient than 2PC and 3PC.
  - Can make progress in most common failure scenarios.

### Long-Lived Transactions and Sagas

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
