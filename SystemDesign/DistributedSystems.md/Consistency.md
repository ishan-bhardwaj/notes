# Consistency in Distributed Systems

- Consistency ensures that all replicas of data agree, or reads return the most recent writes.
- Helps decide storage systems like S3 or Cassandra based on required guarantees.
- Consistency Spectrum - Eventual (Weakest) → Causal → Sequential → Strict/Linearizable (Strongest)
- ACID vs CAP Consistency -
    - ACID - Database-level rules (e.g., uniqueness, foreign keys).
    - CAP - Logical consistency across replicas in distributed systems.

## Eventual Consistency

- Weakest consistency - replicas eventually converge if no new writes occur.
- Different replicas may temporarily return different values.
- Use Case - High availability systems where latest reads are not critical.
- Example - DNS, Cassandra

## Causal Consistency

- Preserves ordering of causally-related operations - independent operations may appear in any order.
- Prevents non-intuitive behaviors in dependent operations.
- Use Case - Systems requiring cause-effect ordering (e.g., comments and replies)
- Example - Facebook comment threads

## Sequential Consistency

- Preserves program order per client - writes are seen in order of submission, not necessarily globally or instantaneously.
- Ensures each client sees operations in their own sequence.
- Use Case - Social networking, messaging apps
- Example - Posts or comments appear in submission order per user

## Strict Consistency / Linearizability

- All reads return the latest write.
- Limits performance for strong guarantees.
- May reduce availability.
- Often uses quorum-based replication.
- Challenges - Network delays, failures - requires synchronous replication + consensus (Paxos/Raft).
- Use Case - Applications requiring immediate correctness (e.g., password updates, banking)
- Example - Google Spanner claims linearizability for many operations.

