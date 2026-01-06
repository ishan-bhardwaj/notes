# Distributed Systems

- A distributed system is a system whose components are located on different networked computers, which communicate and coordinate their actions by passing messages to one another.

- Categories -

  - Synchronous system -

    - A synchronous distributed system is one in which upper bounds exist on message transmission delay, process execution time, and clock drift, allowing algorithms to reason using time.
    - A distributed system is synchronous if all of the following are guaranteed -
      - Bounded message delay - Every message sent is delivered within a known, fixed maximum time ≤ Δ.
      - Bounded process execution time - Each process executes steps within a known time bound ≤ Φ.
      - Perfectly synchronized clocks - All nodes have clocks that are synchronized with negligible drift (or exact global time).
    - Because of these guarantees, time itself can be used as a correctness tool.
    - In a synchronous system -
      - The system progresses in rounds.
      - Each round consists of -
        - Send messages
        - Deliver all messages
        - Perform computation
      - No message from round `r` arrives in round `r+2`.

  - Asynchronous system -
    - An asynchronous distributed system is a model where no timing guarantees exist at all.
    - A distributed system is asynchronous if -
      - No bound on message delay -
        - Messages can take arbitrarily long to arrive.
        - They may be delayed indefinitely (but not lost, in theory).
      - No bound on process execution time -
        - A process can pause for any amount of time (GC, OS scheduling, crashes).
      - No clock synchronization -
        - Clocks may drift arbitrarily.
        - There is no global notion of time.
      - In short - time is meaningless for correctness.

## Types of Failures

- Fail-stop -
  - A node halts permanently.
  - Other nodes can reliably detect that it has failed (e.g., explicit failure signal).
- Crash -
  - A node halts, but silently.
  - Other nodes may not be able to detect this state. They can only assume its failure when they are unable to communicate with it.
- Omission -
  - A node fails to send or receive messages.
  - Types -
    - Send omission - node does not send a message.
    - Receive omission - node does not process an incoming message.
- Byzantine -
  - A node behaves arbitrarily.
  - A node may -
    - Send incorrect or conflicting messages
    - Lie
    - Violate the protocol
    - Act maliciously

## Multiple Deliveries of a Message

- Problem -

  - Distributed nodes communicate via messages.
  - Networks are unreliable → messages may get lost.
  - To cope, nodes retry sending messages, which may lead to duplicate deliveries.
  - Risk - Duplicate messages can cause serious side effects (e.g., double charging in a bank transaction).

- Approaches to Handle Duplicates -
  - Idempotent Operations -
    - Definition - An operation that can be applied multiple times without changing the result beyond the first application.
    - Example (Idempotent) - Adding a value to a set (re-adding has no effect if already present).
    - Example (Non-idempotent) - Incrementing a counter (repeating increases the count each time).
    - Pros - Guarantees correctness even if a message is processed multiple times.
    - Cons - Imposes tight constraints; not all operations can be made idempotent.
  - De-duplication -
    - Definition - Each message has a unique identifier. Recipient tracks processed IDs and ignores duplicates.
    - Requirement - Both sender and receiver must support the mechanism.

| Semantics         | Meaning                        | Implementation                                              |
| ----------------- | ------------------------------ | ----------------------------------------------------------- |
| **At-most-once**  | Message delivered ≤ 1 time     | Send once; ignore retries                                   |
| **At-least-once** | Message delivered ≥ 1 time     | Retry until acknowledged                                    |
| **Exactly-once**  | Message processed exactly once | Hard in practice; requires idempotent ops or de-duplication |

## Detecting Failures

- Timeouts -

  - Impose an artificial upper bound on response times.
  - If a node does not respond within the timeout, it is considered failed.
  - Trade-offs -
    - Small timeout -
      - Pros - Quickly detects crashed nodes.
      - Cons - May mistakenly mark slow nodes as failed.
    - Large timeout -
      - Pros - Fewer false positives for slow nodes.
      - Cons - Slower detection of crashed nodes, wasting time waiting.

- Failure Detector -
  - Component of a node that identifies which nodes have failed.
  - Properties of Failure Detectors -
    - Completeness - Fraction of actual crashed nodes correctly detected.
    - Accuracy - Fraction of non-faulty nodes incorrectly suspected as failed.

> [!NOTE]
> Perfect failure detection is impossible in asynchronous systems.

## Types of Systems

- Stateless Systems -

  - Maintains no state of past interactions.
  - Performs actions purely based on current inputs.
  - Inputs can be -
    - Direct inputs - included in the request.
    - Indirect inputs - received from other systems to fulfill the request.
  - Examples -
    - Service receives numbers and returns the maximum.
    - Service calculates product price using current price and discounts from other services.

- Stateful Systems -
  - Maintains and mutates state over time.
  - Results depend on stored state.
  - Examples -
    - System storing employee ages and returning the maximum age.

> [!TIP]
> Often wise to separate stateless components (business logic) and stateful components (data handling).

## ACID Transactions

- ACID Transactions - Set of properties that guarantee expected behavior of database transactions during errors or failures.
- Atomicity (A) -
  - Transaction with multiple operations is treated as a single unit.
  - Either all operations execute or none execute.
  - In distributed systems, operation executes on all nodes or none.
- Consistency (C) -
  - Transaction only transitions database from one valid state to another.
  - Maintains application-specific invariants (e.g., foreign key constraints).
  - Note - this is different from distributed systems consistency (CAP theorem).
- Isolation (I) -
  - Concurrent transactions appear as if executed one at a time.
  - Prevents interference and anomalies between transactions.
- Durability (D) -
  - Committed transactions remain committed even in case of failure.
  - In single-node systems - recorded in non-volatile storage.
  - In distributed systems - durably stored in multiple nodes for recovery after failures.
