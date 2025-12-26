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