# Iceberg

- Table format created in 2017 by Netflix’s Ryan Blue and Daniel Weeks. A table format is a method of structuring a dataset’s files to present them as a unified “table”.
- File format agnostic and currently supports Apache Parquet, Apache ORC, and Apache Avro. Parquet is the most commonly used format.
- Goals -
    - Consistency - Updates to a table across multiple partitions is quickly and atomically. Users see the data either before or after the update, but not in between.
    - Performance - The table provides metadata and avoid excessive file listing, enabling fast query planning, and the resultant plan executes more quickly since they scan only the files necessary to satisfy the query
    - Easy to use - End users do not need to understand physical table structure. The table is able to give users the benefits of partitioning based on naturally intuitive queries.
    - Evolvability - A table is able to evolve its schema and partitioning scheme safely and without the need for rewriting.
    - Scalability - designed to operate at petabyte scale.

## Key Features

- ACID Transactions -
    - Uses optimistic concurrency control to enable ACID guarantees.
    - Concurrency guarantees are handled by the catalog.

> [!TIP]
> **Optimistic concurrency** assumes transactions won’t conflict and checks for conflicts only when necessary - to minimize locking and improve performance.
> **Pessimistic concurrency** model uses locks to prevent conflicts between transactions, assuming conflicts are likely to occur - not available in iceberg currently.

- Partial evolution -
    - Can update how the table is partitioned at any time without the need to rewrite the table and all its data.
    - Only need to update the metadata.

- Hidden partitioning -
    - Partitioning occurs in two parts -
        - the column - which physical partitioning should be based on.
        - an optional transform to that value including functions such as bucket, truncate, year, month, day, and hour.
    - No need to create extra partition columns.
    - For eg - if a table is partitioned on a `timestamp` with a `day` transform, filtering by `day` (e.g., on the `timestamp` column) uses the transformed value of the `timestamp` for partition pruning, so only the relevant `day` partitions are scanned.
    - In Hive, filtering on the timestamp column would still result in a full table scan, because Hive partitions on separate derived columns (`year`/`month`/`day`) and does not automatically map `timestamp` filters to those partitions.

- Row-level table operations -
    - Supports two row-level update modes -
        - Copy-on-Write (COW) -
            - For any row updates, entire data file is rewritten.
            - Even a single record change rewrites the full file.
            - Slower writes, faster reads.
        - Merge-on-Read (MOR) -
            - Row updates written to separate delta files.
            - Changes are merged at read time.
            - Slower reads, faster writes.

- Time Travel -
    - Iceberg provides immutable snapshots for table’s historical state. 
    - Allows running queries on the state of the table at a given point in time in the past - time travel.

- Version Rollback -
    - Iceberg snapshots also reverts the table’s current state to any of those previous snapshots.

- Schema Evolution -
    - Iceberg provides robust schema evolution, such as - adding/removing a column, renaming a column, or changing a column’s data type etc.
