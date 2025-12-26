# Iceberg Architecture

- Iceberg maintains a tree of metadata that -
    - tracks a table’s partitioning, sorting, schema over time etc.
    - an engine can use to plan queries at a fraction of the time.

- Metadata tree consists of four components of metadata of the table -
    - **Manifest file** -
        - A list of datafiles, containing each datafile’s location/path and key metadata about those datafiles.
        - Allows creating more efficient execution plans.
    - **Manifest list** -
        - Files that define a single snapshot of the table as a list of manifest files along with stats on those manifests.
        - Allows creating more efficient execution plans.
    - **Metadata file** -
        - Files that define a table’s structure, including its schema, partitioning scheme, and a listing of snapshots.
    - **Catalog** -
        - Contains a mapping of table name to location of the table’s most recent metadata file. - - Several tools, including a Hive Metastore, can be used as a catalog.

## The Data Layer

- Stores the actual data of the table - primarily made up of the datafiles themselves, but also includes delete files.
- The files in the data layer make up the leaves of the tree structure of an Apache Iceberg table.
- The data layer is backed by a distributed file system like HDFS, S3, Azure Data Lake Storage, Google Cloud Storage.

### Datafiles

- Stores the data itself.
- Iceberg provides the flexibility to choose different formats depending on what is best suited for a given workload, eg -
    - Parquet - for large-scale online analytical processing (OLAP) analytics.
    - Avro - for low-latency streaming analytics tables.
- Parquet is the most common - 
    - columnar structure lays the foundation for performance features such as the ability for a single file to be split multiple ways for increased parallelism, statistics for each of these split points, and increased compression, which provides lower storage volume and higher read throughput.
    - Parquet file has a set of rows that are broken down so that all the rows’ values for a given column are stored together.
    - All the rows’ values for a given column are further broken down into subsets of the rows’ values for this column, which are called pages.
    - Each of these levels can be read independently by engines and tools, and therefore each can be read in parallel by a given engine or tool.
    - Also, Parquet stores statistics (e.g., minimum and maximum values for a given column for a given row group) that enable engines and tools to decide whether it needs to read all the data or whether it can prune row groups that don’t fit the query.

### Delete Files

- Track which records in the dataset have been deleted.
- Data lake storage is treated as immutable, so we can’t update rows in a file in place. Instead, we need to write a new file.
- The new file - 
    - can be a copy of the old file with the changes reflected in a new copy of it (called copy-on-write [COW])
    - or, it only has the changes written, which engines reading the data then coalesce (called merge-on-read [MOR]).
- Delete files enable the MOR strategy for performing updates and deletes to Iceberg tables.
- Two types of delete files -
    - Positional delete files - identify the row to be deleted by its exact position in the dataset.
    - Equality delete files - identify the row to be deleted by the values of one or more fields of the row. Note that in contrast to positional delete files, there is no reference to where these rows are located within the table.

> [!NOTE]
> Situation - an equality delete file deletes a record via column values, and then in a subsequent commit, a record is added back to the dataset that matches the delete file’s column values.
> Problem - query engines may remove the newly added record from the logical table when reading it.
> Solution - sequence numbers i.e. higher sequence number is allocated to the newly added rows, so the engines won't discard them.

## The Metadata Layer

- Tree structure that tracks the datafiles and metadata about them as well as the operations that resulted in their creation.
- This tree structure is made up of three file types - manifest files, manifest lists, and metadata files.

### Manifest files

- Keep track of files in the data layer (i.e., datafiles and delete files) as well as additional details and statistics about each file, such as the minimum and maximum values for a datafile’s columns.
- Each manifest file keeps track of a subset of the datafiles - contain information such as details about partition membership, record counts, and lower and upper bounds of columns.
- While some of these statistics are also stored in the datafiles themselves, a single manifest file stores these statistics for multiple datafiles, meaning the pruning done from the stats in a single manifest file greatly reduces the need to open many datafiles.
- These statistics are written during the write operation by the engine/tool for each subset of datafiles a manifest file tracks.

> [!TIP]
> While manifest files track datafiles as well as delete files, a separate set of manifest files are used for each of them (i.e., a single manifest file will contain only datafiles or delete files), though the manifest file schemas are identical.

### Manifest Lists

- A manifest list is a snapshot of an Iceberg table at a given point in time.
- For the table at that point in time, it contains a list of all the manifest files, including the location, the partitions it belongs to, and the upper and lower bounds for partition columns for the datafiles it tracks.
- A manifest list contains an array of structs, with each struct keeping track of a single
manifest file.
- Schema of an Iceberg manifest file -

| Always Present? | Field Name             | Data Type            | Description                                                                                                                      |
| --------------- | ---------------------- | -------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Yes             | `manifest_path`        | string               | Location of the manifest file                                                                                                    |
| Yes             | `manifest_length`      | long                 | Length of the manifest file in bytes                                                                                             |
| Yes             | `partition_spec_id`    | int                  | ID of the partition spec used to write the manifest; refers to an entry listed in `partition-specs` in the table’s metadata file |
| Yes             | `content`              | int                  | Type of files tracked by the manifest: `0` = data files, `1` = delete files                                                      |
| Yes             | `sequence_number`      | long                 | Sequence number when the manifest was added to the table                                                                         |
| Yes             | `min_sequence_number`  | long                 | Minimum data sequence number of all live data files or delete files in the manifest                                              |
| Yes             | `added_snapshot_id`    | long                 | ID of the snapshot where the manifest file was added                                                                             |
| Yes             | `added_files_count`    | int                  | Number of entries in the manifest file with status `ADDED (1)`                                                                   |
| Yes             | `existing_files_count` | int                  | Number of entries in the manifest file with status `EXISTING (0)`                                                                |
| Yes             | `deleted_files_count`  | int                  | Number of entries in the manifest file with status `DELETED (2)`                                                                 |
| Yes             | `added_rows_count`     | long                 | Sum of rows in all files marked as `ADDED` in the manifest                                                                       |
| Yes             | `existing_rows_count`  | long                 | Sum of rows in all files marked as `EXISTING` in the manifest                                                                    |
| Yes             | `deleted_rows_count`   | long                 | Sum of rows in all files marked as `DELETED` in the manifest                                                                     |
| No              | `partitions`           | array<field_summary> | List of field summaries for each partition field in the spec; each entry corresponds to a field in the manifest’s partition spec |
| No              | `key_metadata`         | binary               | Implementation-specific key metadata for encryption                                                                              |

- Schema of field_summary -

| Always Present? | Field Name      | Data Type | Description                                                                                                                 |
| --------------- | --------------- | --------- | --------------------------------------------------------------------------------------------------------------------------- |
| Yes             | `contains_null` | boolean   | Whether the manifest contains at least one partition with a null value for the field                                        |
| No              | `contains_nan`  | boolean   | Whether the manifest contains at least one partition with a NaN value for the field                                         |
| No              | `lower_bound`   | bytes     | Lower bound for non-null, non-NaN values in the partition field, or null if all values are null or NaN; serialized to bytes |
| No              | `upper_bound`   | bytes     | Upper bound for non-null, non-NaN values in the partition field, or null if all values are null or NaN; serialized to bytes |

> [!TIP]
> Manifest files and manifest lists are stored in Avro format.

### Metadata Files

- Track manifest lists.
- Metadata files store metadata about an Iceberg table at a certain point in time - includes information about the table’s schema, partition information, snapshots, and which snapshot is the current one.
- Each time a change is made to an Iceberg table, a new metadata file is created and is registered as the latest version of the metadata file atomically via the catalog.
- Metadata le schema -

| Always Present? | Field Name              | Data Type             | Description                                                                                                                                                              |
| --------------- | ----------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Yes             | `format-version`        | integer               | Version number for the table format. Currently `1` or `2`. Implementations must throw an exception if the table’s version is higher than supported. Default is `2`.      |
| Yes             | `table-uuid`            | string                | UUID identifying the table, generated at table creation. Implementations must throw an exception if the UUID does not match the expected UUID after refreshing metadata. |
| Yes             | `location`              | string                | Table’s base location used by writers to store data files, manifest files, and metadata files.                                                                           |
| Yes             | `last-sequence-number`  | 64-bit signed integer | Highest assigned sequence number for the table. A monotonically increasing value tracking snapshot order.                                                                |
| Yes             | `last-updated-ms`       | 64-bit signed integer | Timestamp (milliseconds since Unix epoch) when the table was last updated. Updated just before writing each metadata file.                                               |
| Yes             | `last-column-id`        | integer               | Highest assigned column ID in the table. Ensures new columns receive unused IDs during schema evolution.                                                                 |
| Yes             | `schemas`               | array                 | List of schemas, stored as objects with `schema-id`.                                                                                                                     |
| Yes             | `current-schema-id`     | integer               | ID of the table’s current schema.                                                                                                                                        |
| Yes             | `partition-specs`       | array                 | List of partition specs, stored as full partition spec objects.                                                                                                          |
| Yes             | `default-spec-id`       | integer               | ID of the default partition spec writers should use.                                                                                                                     |
| Yes             | `last-partition-id`     | integer               | Highest assigned partition field ID across all partition specs, ensuring new partition fields receive unused IDs.                                                        |
| No              | `properties`            | map<string, string>   | Table properties controlling read/write behavior (e.g., `commit.retry.num-retries`). Not intended for arbitrary metadata.                                                |
| No              | `current-snapshot-id`   | 64-bit signed integer | ID of the current table snapshot. Must match the current ID of the `main` branch in `refs`.                                                                              |
| No              | `snapshots`             | array                 | List of valid snapshots whose data files exist in the filesystem. Data files must not be deleted until the last referencing snapshot is garbage collected.               |
| No              | `snapshot-log`          | array                 | List of timestamp and snapshot ID pairs tracking changes to the current snapshot. Entries before expired snapshots should be removed.                                    |
| No              | `metadata-log`          | array                 | List of timestamp and metadata file location pairs tracking previous metadata files. Tables may retain a fixed-size recent history.                                      |
| Yes             | `sort-orders`           | array                 | List of sort orders, stored as full sort order objects.                                                                                                                  |
| Yes             | `default-sort-order-id` | integer               | Default sort order ID for writers. Not used during reads, as reads rely on specs stored in manifest files.                                                               |
| No              | `refs`                  | map                   | Map of snapshot references. Keys are reference names; values are snapshot reference objects. A `main` branch always exists pointing to `current-snapshot-id`.            |
| No              | `statistics`            | array                 | Optional list of table statistics.                                                                                                                                       |

### Pun Files

- A puffin file stores statistics and indexes about the data in the table that improve the performance of an even broader range of queries than the statistics stored in the datafiles and metadata files.
- Such query example - how many unique people placed an order with you in the past 30 days - we can prune out only the data for the last 30 days, but we would still have to read every order in those 30 days and do aggregations in the engine.
- The file contains sets of arbitrary byte sequences called blobs, along with the associated metadata required to analyze the blobs.

> [!NOTE]
> While this structure enables statistics and index structures of any type (e.g., bloom filters), currently the only type supported is the Theta sketch from the Apache DataSketches library.

- Valuable when the use case allows for an approximation, eg - approximate number of distinct values of a column for a given set of rows.
