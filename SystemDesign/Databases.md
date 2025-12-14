# Databases

- An organized collection of data.
- Supports store, retrieve, update, delete (CRUD) operations.
- Types - Relational and Non-Relational databases.

## Relational Databases (SQL)

- Predefined schema (tables, rows, columns).
- Data stored as relations (tables).
- Each row (tuple) has a primary key.
- Foreign keys link tables together.
- Uses SQL for querying and manipulation.

### ACID Properties -

- Relational databases ensure data integrity using ACID -
    - Atomicity - All operations in a transaction succeed or fail together.
    - Consistency - Database moves from one valid state to another.
    - Isolation - Concurrent transactions don’t interfere.
    - Durability - Committed data survives system failures.

> [!NOTE]
> ACID simplifies application logic but may trade performance for strong guarantees.

- Advantages -
    - Flexibility - Schema changes via DDL while DB is running.
    - Reduced redundancy - Normalization avoids data duplication.
    - Concurrency control - Transactions prevent conflicts (e.g., double booking).
    - Integration - Multiple applications can share a single database.
    - Backup & recovery - Strong guarantees, replication, mirroring.

- Drawbacks - Impedance Mismatch
    - Mismatch between -
        - Relational model (tables, rows)
        - In-memory objects (lists, nested structures)
        - Requires translation (ORMs, joins, mapping)
        - Complex objects must be split across tables


## Non-Relational Databases (NoSQL)

- Simple data models (avoid impedance mismatch).
- Horizontal scalability (scale-out on clusters).
- High availability via replication.
- Schema-less or dynamic schema.
- Cost-effective (open source, commodity hardware).
- Often relax strong consistency (eventual consistency).

- Designed for -
    - Large-scale systems
    - Low latency
    - Semi-structured / unstructured data
    - Flexible schemas

- Drawbacks -
    - Lack of standardization across systems.
    - Harder to migrate between NoSQL databases.
    - Weaker consistency guarantees.
    - No built-in referential integrity.

### Types of NoSQL Databases

- Key-Value Databases -
    - Store data as (key → value) pairs.
    - Values can be simple or complex objects.
    - Easy partitioning and scaling.
    - Examples - Amazon DynamoDB, Redis, Memcached
    - Use cases - Session management, caching, User session data (recommendations, preferences) etc.

- Document Databases -
    - Store data as documents (JSON, XML, BSON).
    - Each document can have a different structure.
    - Hierarchical and nested data supported.
    - Examples - MongoDB, Google Firestore
    - Use cases - Product catalogs (e-commerce), Content management systems, Blogs & media platforms etc.

- Graph Databases -
    - Data stored as nodes (entities) and edges (relationships).
    - Optimized for relationship-heavy queries.
    - Examples - Neo4j, OrientDB, InfiniteGraph
    - Use cases - Social networks, Recommendation engines, Fraud detection, Knowledge graphs etc.

- Columnar Databases -
    - Store data column-wise instead of row-wise.
    - Optimized for read-heavy analytical workloads.
    - Examples - Amazon Redshift, Google BigQuery
    - Use cases - Analytics, Data warehousing, Aggregations and trend analysis etc.

> [!WARNING]
> Columnar Databases - Not to be confused with wide-column stores (e.g., Cassandra, HBase).


## Replication

- Maintaining multiple copies of data across different nodes (often geo-distributed).
- Benefits -
    - Lower latency (data closer to users)
    - Higher availability (tolerates node failures)
    - Higher read throughput (multiple replicas serve reads)

> [!WARNING]
> Complexity arises when data changes frequently.

- Key Replication Challenges -
    - Keeping replicas consistent
    - Handling node failures
    - Synchronous vs asynchronous replication
    - Replication lag
    - Concurrent writes
    - Exposed consistency model

### Synchronous vs Asynchronous Replication

- Trade-off - Consistency vs Availability.
- Synchronous Replication -
    - Primary waits for ack from replicas.
    - Strong consistency
    - High latency
    - Reduced availability if replicas fail
- Asynchronous Replication
    - Primary responds without waiting for replicas
    - Low latency, high availability
    - Risk of data loss on primary failure
    - Eventual consistency

## Replication Models

### Single-Leader (Primary–Secondary) -

- Best for - Read-heavy workloads.
- One primary handles all writes.
- Secondaries replicate data and serve reads.
- Pros -
    - Simple
    - Read scalability
    - Read resilience
- Cons -
    - Primary is bottleneck
    - No write scalability
    - Data loss possible with async replication

- Primary–Secondary Replication Methods -
    - Statement-Based Replication (SBR) -
        - Replicates SQL statements.
        - Used in older MySQL.
        - Pros - simple.
        - Cons - Nondeterministic functions (e.g., NOW()) cause inconsistency.
    - Write-Ahead Log (WAL) Shipping -
        - Replicates transaction logs.
        - Used in PostgreSQL, Oracle.
        - Pros -
            - Crash recovery
            - Deterministic
        - Cons -
            - Tight coupling with DB internals
            - Harder upgrades
    - Logical (Row-Based) Replication -
        - Replicates row-level changes.
        - Schema-aware.
        - Pros -
            - Flexible
            - Safer schema evolution

- Problems with Async Primary–Secondary -
    - Lost writes if primary fails.
    - Read-after-write inconsistency.
    - Mitigation -
        - Read user-modified data from leader.
        - Other reads from followers.
    - ⚠️ Increases load on primary

### Multi-Leader Replication

- Multiple nodes accept writes.
- Replicate changes to each other.
- Pros -
    - Better write scalability
    - Offline support
- Cons - Write conflicts
- Use case - Offline-first apps (e.g., calendars)

- Conflict Handling in Multi-Leader -
    - Conflict Avoidance -
        - Route writes for same record to one leader.
        - Breaks with geo-mobility.
    - Last-Write-Wins (LWW) -
        - Latest timestamp wins.
        - Clock skew can cause data loss.
    - Custom Logic -
        - App-defined conflict resolution.
        -Most flexible.

- Multi-Leader Topologies -
    - Circular
    - Star
    - All-to-all (most common) - avoids single-node failure issues

### Peer-to-Peer (Leaderless) Replication

- No primary node.
- All nodes accept reads and writes.
- Used in - Apache Cassandra.
- Pros -
    - High availability
    - Write scalability
    - No single point of failure
- Cons -
    - Inconsistency from concurrent writes

- Quorums (Leaderless Replication)
    - Let -
        - `n` = total replicas
        - `w` = replicas required for write
        - `r` = replicas required for read
    - Condition for consistency - `w + r > n`
    - Ensures at least one replica has latest data
    - Configurable in - Dynamo-style databases

## Data Partitioning

- Partitioning (sharding) distributes data across nodes to -
    - Increase throughput
    - Reduce latency
    - Scale storage & traffic
    - Balance read/write load

### Sharding
- Splitting a large dataset into smaller chunks (shards).
- Each shard is managed by a different node.
- Goal - balanced partitions + balanced query load.
- ⚠️ Uneven sharding → hotspots → bottlenecks

- Types of Sharding -
    - Vertical Sharding -
        - Split data by columns or tables.
        - Different tables/columns stored on different nodes.
        - Use cases -
            - Separate wide columns (BLOBs, large text)
            - Reduce row size → faster reads
        - Example -
            - Employee table → Employee + EmployeePicture
            - Primary key replicated in both tables.
        - Notes -
            - Joins across shards are expensive.
            - Mostly manual and schema-driven.
    - Horizontal Sharding -
        - Split data row-wise.
        - Each shard stores a subset of rows.
        - Used when -
            - Tables grow too large.
            - Read/write latency increases.
    - Types of Sharding -
        - Key-Range Based Sharding -
            - Each shard owns a continuous range of keys.
            - Pros -
                - Easy to locate data.
                - Efficient range queries.
                - Data can stay sorted.
            - Cons -
                - Range queries only on partition key.
                - Poor key choice → hotspots.
            - Multi-table sharding -
                - Same partition key across related tables.
                - Tables with same key colocated on same shard.
            - Design rules -
                - Partition key replicated in all tables.
                - Global uniqueness of primary keys.
                - Creation_date used for global merges.

        -  Hash-Based Sharding -
            - Hash(key) → partition
            - Typically - `hash(key) mod N`
            - Pros - Uniform data distribution
            - Cons -
                - No range queries
                - Poor support for rebalancing

    - How Many Shards?
        - Based on max data/node with acceptable performance.
        - Example -
            - DB size = `10 TB`
            - Node capacity = `50 GB`
            - Shards = `200`

- Consistent Hashing -
    - Nodes & keys placed on a hash ring.
    - Minimal data movement on node changes.
    - Advantages -
        - Easy horizontal scaling
        - Better latency & throughput
    - Disadvantages -
        - Possible uneven distribution without tuning

### Rebalancing Partitions

- Why Rebalancing Is Needed -
    - Uneven data distribution
    - Hot partitions
    - Increased traffic
    - Node addition/removal

- Rebalancing Strategies -
    - Avoid `hash mod n`.
    - Adding/removing nodes changes mappings.
    - Causes massive data movement.

- Fixed Number of Partitions -
    - Create many partitions upfront.
    - Assign partitions to nodes.
    - Pros -
        - Simple rebalancing
    - Cons -
        - Too many small partitions → overhead
        - Too few large partitions → slow recovery
    - Used by - Elasticsearch, Riak

- Dynamic Partitioning -
    - Split partitions when size threshold reached.
    - Partitions grow/shrink with data.
    - Pros -
        - Adapts to data size automatically.
    - Cons -
        - Complex during live reads/writes.
        - Consistency & latency challenges.
    - Used by - HBase, MongoDB

- Partition Proportional to Nodes -
    - Fixed partitions per node.
    - Partitions resize with data.
    - Behavior - New node splits random partitions.
    - Cons - Can cause unfair splits.
    - Used by - Cassandra, Ketama

### Partitioning & Secondary Indexes

- Local (Document-based) Indexes -
    - Each shard maintains its own indexes.
    - Pros -
        - Simple writes
    - Cons -
        - Reads must query all shards.
        - High tail latency.

- Global (Term-based) Indexes -
    - Index partitioned by indexed term.
    - Pros -
        - Efficient reads
    - Cons -
        - Writes touch multiple shards
        - Higher complexity

### Request Routing (Service Discovery)

- How does a client find the right shard? Approaches -
    - Any-node routing (forward if needed)
    - Dedicated routing tier
    - Client-side shard awareness
- Challenge - Keeping mappings updated after rebalancing.

- ZooKeeper -
    - Centralized metadata & coordination service.
    - Tracks -
        - Partition ↔ node mapping
        - Node membership
    - Used by - HBase, Kafka, SolrCloud

> [!TIP]
> Sharding vs Replication -
>   - Sharding → scalability & load distribution
>   - Replication → availability & fault tolerance
