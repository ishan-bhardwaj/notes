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
