# Partitioning

- Promotes scalability.
- Splitting a dataset into smaller datasets and assigning them to different nodes.
- Enables adding nodes to scale storage and processing capacity.
- Types -
  - Horizontal Partitioning / Sharding -
    - Split a table into multiple tables, each containing a subset of rows.
    - Example - Partition a student table alphabetically by surname - each shard stored on a different node.
  - Vertical Partitioning -
    - Split a table into multiple tables with fewer columns - related columns stored in separate tables.
    - Often uses normalization - can split even normalized columns.

## Horizontal Partitioning Algorithms

- Range Partitioning -

  - Split dataset into ranges based on a specific attribute - each range stored on a different node.
  - Node Mapping - System maintains a map of ranges → nodes to route requests.
  - Advantages -
    - Simple and easy to implement.
    - Efficient range queries for small ranges within a single node.
    - Easy to adjust ranges (repartition) by moving data between two nodes.
  - Disadvantages -
    - Range queries on non-partitioning attributes not supported.
    - Poor performance for large ranges spanning multiple nodes.
    - Uneven data/traffic distribution can overload some nodes.
  - Examples - Google BigTable, Apache HBase.

- Hash Partitioning -

  - Apply a hash function to an attribute - hash value determines the partition/node.
  - Example - `node = hash(surname) mod n`
  - Advantages -
    - No need to store a mapping - can compute node at runtime.
    - Uniform data distribution reduces node overload.
  - Disadvantages -
    - Range queries not possible without extra data or querying all nodes.
    - Adding/removing nodes requires repartitioning, moving significant data.

- Consistent Hashing -
  - Hash nodes to a ring `[0, L]` - each key maps to the node after hash(key) mod L.
  - Node Changes -
    - Adding a node → only affects next node on the ring.
    - Removing a node → its data transferred to next node on the ring.
  - Advantages -
    - Reduced data movement compared to standard hash partitioning.
  - Disadvantages -
    - Potential for nonuniform data distribution due to random node placement.
    - Imbalance when nodes are added/removed.
  - Mitigation - Use virtual nodes (multiple positions per physical node).
  - Examples - Apache Cassandra, Dynamo.

> [!NOTE]
> Hybrid approach - Combining range & hash partitioning can balance query efficiency and uniform data distribution.
