# Data Sources & File Formats

## Data Sources API v1

- __Data Sources API v1 (DSv1)__ - the original Spark data source interface; based on Hadoop `InputFormat`/`OutputFormat`; still used by many connectors; being phased out in favor of DSv2
- Entry point - `DataFrameReader.format("source").load(path)` triggers `DataSource.resolveRelation()`

### DSv1 Interfaces

- __`RelationProvider`__ - creates a `BaseRelation` from options; read-only sources
- __`SchemaRelationProvider`__ - like `RelationProvider` but accepts user-provided schema
- __`CreatableRelationProvider`__ - creates a `BaseRelation` and writes data; read-write sources
- __`BaseRelation`__ - represents a table; has a `schema: StructType`; delegates to scan traits
- Scan traits mixed into `BaseRelation` -
    - `TableScan` - `buildScan(): RDD[Row]` - full table scan; no pushdown
    - `PrunedScan` - `buildScan(requiredColumns: Array[String]): RDD[Row]` - column pruning
    - `PrunedFilteredScan` - `buildScan(requiredColumns, filters: Array[Filter]): RDD[Row]` - column pruning + filter pushdown
    - `InsertableRelation` - `insert(data: DataFrame, overwrite: Boolean)` - write support

### DSv1 Limitations

- `RDD[Row]` output - returns JVM `Row` objects; incompatible with Tungsten binary format; cannot participate in WSCG
- Filter pushdown limited - only a fixed set of `Filter` subtypes (`EqualTo`, `GreaterThan`, `In`, etc.); cannot express arbitrary predicates
- No partition awareness at the source level - Spark cannot ask the source for its partitioning scheme
- No statistics reporting - source cannot tell Catalyst how many rows it will return; CBO cannot use source-level statistics
- No columnar output - cannot return Arrow `ColumnarBatch`; always row-by-row

---

## Data Sources API v2

- __Data Sources API v2 (DSv2)__ - the modern data source interface introduced in Spark 2.3, stabilized in Spark 3.0; enables sources to participate deeply in Catalyst optimization
- Key design principles -
    - Sources return `InternalRow` (Tungsten binary) not `Row` (JVM objects)
    - Sources report statistics to Catalyst for CBO
    - Sources negotiate filter and column pruning at planning time
    - Sources can produce `ColumnarBatch` (Arrow) for vectorized reads
    - Unified batch and streaming interface

### DSv2 Core Interfaces

- __`TableProvider`__ - entry point; `getTable(options)` returns a `Table`
- __`Table`__ - represents a data source; has `name()`, `schema()`, `capabilities()`
- __`SupportsRead`__ - `Table` mixin; `newScanBuilder(options)` returns a `ScanBuilder`
- __`SupportsWrite`__ - `Table` mixin; `newWriteBuilder(options)` returns a `WriteBuilder`
- __`ScanBuilder`__ - builds a `Scan`; applies pushdowns before `build()` finalizes the scan
- __`Scan`__ - describes what to read; `toBatchScan()` returns a `Batch` for batch execution
- __`Batch`__ - `planInputPartitions()` returns `InputPartition[]`; `createReaderFactory()` returns factory for `PartitionReader`
- __`PartitionReader[T]`__ - reads one `InputPartition`; `next()/get()` iterator interface

### DSv2 Physical Plan

- `DataSourceV2Relation` (logical) → `BatchScanExec` (physical)
- `BatchScanExec` calls `Scan.toBatchScan().planInputPartitions()` to get partitions
- Each partition processed by one task using `PartitionReader` from `createReaderFactory()`
- If source implements `SupportsReportPartitioning` - reports its output partitioning; `EnsureRequirements` can avoid inserting shuffles

### DSv2 Capabilities

- `TableCapability` enum - sources declare what they support -
    - `BATCH_READ`, `BATCH_WRITE`, `STREAMING_READ`, `STREAMING_WRITE`
    - `OVERWRITE_BY_FILTER`, `OVERWRITE_DYNAMIC`, `TRUNCATE`
    - `ACCEPT_ANY_SCHEMA` - source accepts any schema on write
    - `MICRO_BATCH_READ`, `CONTINUOUS_READ` - streaming modes

---

## ScanBuilder & Scan Internals

- __`ScanBuilder`__ - the builder object that accumulates pushdown decisions before finalizing a `Scan`; Catalyst calls pushdown methods on `ScanBuilder` during physical planning

### ScanBuilder Pushdown Interfaces

- __`SupportsPushDownFilters`__ -
    - `pushFilters(filters: Filter[]): Filter[]` - Catalyst passes ALL filters; source returns the ones it could NOT handle
    - Unhandled filters returned by source remain as `FilterExec` above the scan in Spark plan
    - Handled filters evaluated inside the source (row group skipping, index lookups)
- __`SupportsPushDownRequiredColumns`__ -
    - `pruneColumns(requiredSchema: StructType)` - Catalyst sends only the columns actually needed
    - Source reads only required columns from storage
- __`SupportsPushDownLimit`__ -
    - `pushLimit(limit: Int): Boolean` - source can short-circuit after `limit` rows
- __`SupportsPushDownTopN`__ -
    - `pushTopN(order: SortOrder[], limit: Int): Boolean` - sort + limit pushed to source
- __`SupportsPushDownAggregates`__ -
    - `pushAggregation(aggregation: Aggregation): Boolean` - aggregates pushed to source (eg - COUNT pushed to database)
- __`SupportsReportStatistics`__ -
    - `estimateStatistics(): Statistics` - source reports estimated row count and size; used by CBO

### Scan Object

- `ScanBuilder.build()` finalizes the scan with all accepted pushdowns
- `Scan` contains -
    - `readSchema(): StructType` - the pruned schema (after column pruning)
    - `toBatchScan(): Batch` - batch execution interface
    - `supportedCustomMetrics()` - custom metrics reported in Spark UI

### PartitionReader

- One per `InputPartition`; lives on executor
- `next(): Boolean` - advances to next record
- `get(): T` - returns current record (`T` is `InternalRow` or `ColumnarBatch`)
- `close()` - releases resources; called via task completion listener

---

## Predicate Pushdown at Source

- __Predicate pushdown__ - moving filter conditions into the data source so data is filtered before being passed to Spark; reduces deserialization, network transfer, and memory pressure

### DSv2 Filter Types

- Spark passes `Filter` objects (not `Expression` trees) to `SupportsPushDownFilters` -
    - `EqualTo(attribute, value)`, `EqualNullSafe(attribute, value)`
    - `GreaterThan`, `GreaterThanOrEqual`, `LessThan`, `LessThanOrEqual`
    - `In(attribute, values[])`, `IsNull(attribute)`, `IsNotNull(attribute)`
    - `And(left, right)`, `Or(left, right)`, `Not(filter)`
    - `StringStartsWith`, `StringEndsWith`, `StringContains`
- Sources inspect each filter and accept/reject based on their capability
- Complex expressions (UDFs, arbitrary SQL) - cannot be pushed; stay as `FilterExec`

### Verifying Pushdown

- Python -
```python
    df = spark.read.parquet("hdfs://path/").filter(col("date") == "2024-01-01")
    df.explain("extended")
    # Pushed filter appears INSIDE BatchScanExec or FileSourceScanExec
    # Non-pushed filter appears as FilterExec ABOVE the scan node
```

---

## Column Pruning at Source

- __Column pruning__ - reading only the columns needed by the query; critical for columnar formats (Parquet, ORC) where unneeded columns are stored separately and can be completely skipped

### Mechanism

- Catalyst's `ColumnPruning` rule computes which columns are referenced by the full query
- Passes pruned `StructType` to `SupportsPushDownRequiredColumns.pruneColumns(schema)`
- Source's `readSchema()` returns the pruned schema
- `PartitionReader` reads only those columns from storage
- For Parquet - unneeded column chunks not read at all (column-level I/O)
- For CSV/JSON - all bytes must be read and parsed; column pruning only eliminates deserialization overhead

---

## Partition Pruning at Source

- __Partition pruning__ - eliminating entire data partitions (in the storage partitioning sense, not Spark task partitions) based on filter predicates; reduces files scanned

### File-Based Partition Pruning

- For Hive-style partitioned tables (`year=2024/month=01/`) -
    - Partition predicates extracted from query filters
    - `FileIndex.listFiles(partitionFilters, dataFilters)` called with partition predicates
    - Only matching directories listed; non-matching directories never opened
    - `spark.sql.hive.metastorePartitionPruning=true` - uses metastore partition list instead of directory listing (faster for tables with many partitions)

### Dynamic Partition Pruning (DPP)

- __DPP__ - at runtime, uses the result of a smaller dimension table scan to prune partitions of a larger fact table; only available when AQE enabled or for specific join patterns
- Enabled via `spark.sql.optimizer.dynamicPartitionPruning.enabled=true` (default `true`)
- Mechanism -
    1. Fact table joined to dimension table on partition column
    2. Dimension table filter produces a small set of values
    3. Those values used to prune fact table partitions before scan
    4. In-broadcast path - if dimension side is broadcast, the broadcast relation reused as a runtime filter
- Example -
```sql
    SELECT * FROM orders o JOIN dates d ON o.date_id = d.id
    WHERE d.year = 2024
    -- DPP: scans only orders partitions where date_id is in 2024 dates
```

---

## FileScan & FilePartition

- __`FileScan`__ - the DSv2 `Scan` implementation for file-based sources (Parquet, ORC, CSV, JSON, Avro)
- __`FilePartition`__ - a group of `PartitionedFile` objects assigned to one task

### FilePartition Creation

- `FilePartition.getFilePartitions(sparkSession, partitionedFiles, maxSplitBytes)` -
    - Groups files into partitions targeting `maxSplitBytes` per partition
    - `maxSplitBytes = min(maxPartitionBytes, max(openCostInBytes, bytesPerCore))`
    - `maxPartitionBytes = spark.sql.files.maxPartitionBytes` (default $128 MB$)
    - `openCostInBytes = spark.sql.files.openCostInBytes` (default $4 MB$) - virtual cost added per file to discourage too many small files in one partition
    - Result - each `FilePartition` contains one or more `PartitionedFile` objects summing to ~`maxSplitBytes`

### PartitionedFile

- Represents a slice of a file to read -
    - `filePath` - HDFS/S3/local path
    - `start` - byte offset within file (for splittable formats)
    - `length` - bytes to read
    - `partitionValues` - Hive partition column values inferred from directory path
    - `locations` - HDFS block replica node addresses (preferred locations for scheduling)

### FileSourceScanExec (DSv1)

- Physical operator for DSv1 file sources
- `output` filtered by column pruning
- `partitionFilters` and `dataFilters` applied at scan level
- `inputRDD()` creates `FileScanRDD` - one partition per `FilePartition`

---

## DataSourceV2Relation

- __`DataSourceV2Relation`__ - the logical plan node representing a DSv2 table scan; holds reference to `Table`, `Scan`, and output attributes
- Created by `DataFrameReader.load()` when source implements DSv2
- Catalyst transforms this logical node through the optimization pipeline -
    - Analysis - schema resolved from `Table.schema()`
    - Optimization - `SupportsPushDownFilters` / `SupportsPushDownRequiredColumns` called; filters and columns pushed into `Scan`
    - Physical planning - `DataSourceV2Strategy` converts `DataSourceV2Relation` → `BatchScanExec`

### BatchScanExec

- Physical operator for DSv2 batch reads
- `planInputPartitions()` called once at planning time to get `InputPartition[]`
- `createReaderFactory()` returns factory; each task uses it to create `PartitionReader`
- If source returns `ColumnarBatch` - `BatchScanExec` operates in columnar mode; compatible with columnar operators downstream

---

## JDBC Data Source Internals

- __JDBC data source__ - reads from and writes to relational databases via JDBC; available in both DSv1 (primary) and partial DSv2

### Read Internals

- `JDBCRelation` (DSv1 `BaseRelation`) - holds connection URL, table name, partition info, connection properties
- Single-partition by default - all data through one JDBC connection; usable only for small tables
- Multi-partition requires specifying `partitionColumn`, `lowerBound`, `upperBound`, `numPartitions` -
    - Creates `numPartitions` range predicates on `partitionColumn`
    - Each Spark task opens its own JDBC connection and executes its range query
    - `lowerBound` / `upperBound` only affect partition boundary calculation; rows outside range NOT excluded

### Predicate Pushdown

- `JDBCRDD` constructs SQL `WHERE` clause from pushed filters -
    - Converts Spark `Filter` objects to SQL fragments
    - Sends composite `WHERE` clause to database; database executes predicate
    - Database-side indexes leveraged automatically
- Unpushed filters evaluated by `FilterExec` after rows returned to Spark

### Column Pruning

- `JDBCRDD` sends `SELECT col1, col2 FROM table WHERE ...` - only required columns selected
- Reduces bytes transferred over JDBC connection

### Write Internals

- `JdbcUtils.saveTable` - each task writes its partition via JDBC batch inserts
- `batchsize` option (default $1000$) - rows per JDBC batch; increase for bulk writes
- `isolationLevel` - transaction isolation; default `READ_UNCOMMITTED` for performance
- `truncate=true` - uses `TRUNCATE TABLE` instead of `DROP TABLE` + `CREATE TABLE` for overwrite; faster; preserves table structure

### Connection Management

- Each task creates its own JDBC connection; connection closed when task completes
- `numPartitions` = number of concurrent JDBC connections to database; tune carefully to avoid overwhelming DB connection pool
- `connectionProvider` SPI (Spark 3.1+) - custom connection pool integration

---

## Kafka Data Source Internals

- __Kafka data source__ - Structured Streaming and batch source for Apache Kafka; maintains offset tracking for exactly-once processing

### Offset Management

- `KafkaSourceProvider` - DSv2 `TableProvider` for Kafka
- `KafkaTable` reports available offset ranges per partition
- Each Spark partition corresponds to one Kafka topic partition + offset range
- `KafkaMicroBatchStream` - for Structured Streaming; commits offsets to checkpoint after each batch

### Partition Mapping

- By default - one Spark partition per Kafka topic partition
- `minPartitions` option - if total Kafka partitions < `minPartitions`, splits Kafka partitions into sub-ranges
    - Increases Spark parallelism for slow Kafka partitions
- Kafka partition → Spark partition mapping computed by `KafkaOffsetRangeCalculator`

### Batch Read

- Python -
```python
    df = spark.read \
        .format("kafka") \
        .option("kafka.bootstrap.servers", "broker:9092") \
        .option("subscribe", "events") \
        .option("startingOffsets", "earliest") \
        .option("endingOffsets", "latest") \
        .load()
    # Schema: key (binary), value (binary), topic, partition, offset, timestamp, timestampType
```

### Schema

- Kafka source always returns fixed schema - `key: BinaryType`, `value: BinaryType`, `topic: StringType`, `partition: IntegerType`, `offset: LongType`, `timestamp: TimestampType`
- Deserialization of `key` and `value` bytes is user's responsibility (cast, from_json, Avro deserializer)

### Offset Tracking for Streaming

- `KafkaSourceOffset` - wraps `Map[TopicPartition, Long]`; stored in checkpoint directory
- On restart - reads committed offsets from checkpoint; resumes from last committed position
- Exactly-once guarantee combined with `ForeachBatchWriter` to idempotent sink

---

## Custom Data Source Implementation

- Implementing a DSv2 data source end-to-end

### Minimal DSv2 Read Source

- Scala -
```scala
    import org.apache.spark.sql.connector.catalog._
    import org.apache.spark.sql.connector.read._
    import org.apache.spark.sql.types._
    import org.apache.spark.sql.util.CaseInsensitiveStringMap

    // 1. TableProvider - entry point registered via META-INF/services
    class MyTableProvider extends TableProvider {
        override def getTable(schema: StructType, transforms: Array[Transform],
                              properties: java.util.Map[String, String]): Table =
            new MyTable(properties)

        override def inferSchema(options: CaseInsensitiveStringMap): StructType =
            StructType(Array(StructField("id", LongType), StructField("value", StringType)))

        override def supportsExternalMetadata(): Boolean = true
    }

    // 2. Table - represents the data source
    class MyTable(properties: java.util.Map[String, String])
            extends Table with SupportsRead {

        override def name(): String = "MyCustomTable"
        override def schema(): StructType =
            StructType(Array(StructField("id", LongType), StructField("value", StringType)))
        override def capabilities(): java.util.Set[TableCapability] =
            java.util.EnumSet.of(TableCapability.BATCH_READ)
        override def newScanBuilder(options: CaseInsensitiveStringMap): ScanBuilder =
            new MyScanBuilder(options)
    }

    // 3. ScanBuilder - applies pushdowns
    class MyScanBuilder(options: CaseInsensitiveStringMap)
            extends ScanBuilder with SupportsPushDownFilters {

        private var pushedFilters: Array[Filter] = Array.empty

        override def pushFilters(filters: Array[Filter]): Array[Filter] = {
            val (supported, unsupported) = filters.partition(canPush)
            pushedFilters = supported
            unsupported    // return unhandled filters to Spark
        }

        private def canPush(f: Filter): Boolean = f match {
            case EqualTo(_, _) => true
            case _ => false
        }

        override def pushedFilters(): Array[Filter] = pushedFilters

        override def build(): Scan = new MyScan(pushedFilters)
    }

    // 4. Scan - describes what to read
    class MyScan(filters: Array[Filter]) extends Scan with Batch {
        override def readSchema(): StructType =
            StructType(Array(StructField("id", LongType), StructField("value", StringType)))

        override def toBatch(): Batch = this

        override def planInputPartitions(): Array[InputPartition] =
            Array(new MyInputPartition(0, filters), new MyInputPartition(1, filters))

        override def createReaderFactory(): PartitionReaderFactory = new MyReaderFactory(filters)
    }

    // 5. InputPartition - serializable; sent to executor
    class MyInputPartition(val partitionId: Int, val filters: Array[Filter])
            extends InputPartition

    // 6. PartitionReaderFactory - creates readers on executor
    class MyReaderFactory(filters: Array[Filter]) extends PartitionReaderFactory {
        override def createReader(partition: InputPartition): PartitionReader[InternalRow] =
            new MyPartitionReader(partition.asInstanceOf[MyInputPartition])
    }

    // 7. PartitionReader - runs on executor; produces InternalRow
    class MyPartitionReader(partition: MyInputPartition)
            extends PartitionReader[InternalRow] {

        private val data = fetchData(partition.partitionId)  // your data fetch logic
        private var idx = 0
        private val row = new GenericInternalRow(2)

        override def next(): Boolean = idx < data.length
        override def get(): InternalRow = {
            row.setLong(0, data(idx).id)
            row.setString(1, data(idx).value)  // actually UTF8String
            idx += 1
            row
        }
        override def close(): Unit = {}
    }
```

### Registration

- `META-INF/services/org.apache.spark.sql.sources.DataSourceRegister` - register `MyTableProvider` short name
- Or use full class name - `spark.read.format("com.example.MyTableProvider").load()`

---

## Parquet Internals — Row Groups, Column Chunks, Pages

- __Parquet__ - columnar binary file format; the dominant storage format for analytical workloads on Spark; designed for efficient columnar I/O and predicate pushdown

### File Structure

```
Parquet File
├── Magic bytes (PAR1)
├── Row Group 0
│   ├── Column Chunk (col 0) → Pages → data + statistics
│   ├── Column Chunk (col 1) → Pages → data + statistics
│   └── Column Chunk (col N) → Pages → data + statistics
├── Row Group 1
│   └── ...
├── Footer (File Metadata)
│   ├── Schema
│   ├── Row group metadata (offsets, sizes, row counts)
│   ├── Column chunk metadata (encoding, compression, statistics)
│   └── Key-value metadata (arbitrary properties)
└── Magic bytes (PAR1)
```

### Row Group

- A horizontal partition of rows; all columns for a range of rows stored together in column chunks
- Default size - $128 MB$ (matching HDFS block size for locality); configurable via `parquet.block.size`
- Spark reads one row group per task when possible (aligned with HDFS blocks)
- Row group is the unit of row-level statistics (min/max per column chunk) for row group skipping
- Too large - less granular skipping; more memory needed for reading
- Too small - more metadata; more footer overhead; less compression efficiency

### Column Chunk

- All values for one column within one row group
- Stored contiguously; Parquet reads only the column chunks for required columns (column pruning at I/O level)
- Contains column-level statistics - `min`, `max`, `null_count`, `distinct_count`
- Each column chunk is one or more pages

### Pages

- The smallest unit of I/O and compression in Parquet
- Three types -
    - __Data page__ - actual encoded column values; typically $1 MB$ default (`parquet.page.size`)
    - __Dictionary page__ - dictionary for dictionary-encoded columns; one per column chunk; stored first
    - __Index page__ - page-level statistics (Parquet 2.x); enables fine-grained page skipping
- Compression applied per page (`snappy`, `gzip`, `zstd`, `lz4`, `brotli`)
- Each page individually decompressible - random access within a column chunk

### Footer

- Stored at end of file; Parquet readers read footer first (seek to end) to get metadata
- Contains -
    - File schema (`MessageType`)
    - Row group metadata (file offsets, byte sizes, row counts per row group)
    - Column chunk metadata (encoding type, compression codec, statistics, dictionary offset)
    - Key-value metadata (user-defined properties; Spark stores schema, partitioning info, etc.)
- Small footer → fast metadata reads; large footer (many columns × many row groups) → slow `DESCRIBE`

---

## Parquet Dictionary Encoding

- __Dictionary encoding__ - replaces repeated column values with integer indices into a dictionary; dramatically reduces storage for low-cardinality columns
- Applied per column chunk; Parquet decides per-chunk whether to use dictionary or plain encoding

### How Dictionary Encoding Works

- Parquet builds a dictionary of distinct values seen in the column chunk
- Each value replaced by its index (integer) in the dictionary
- Dictionary stored in `DictionaryPage` at start of column chunk
- Data pages store integer indices (bit-packed)
- If dictionary grows too large (`parquet.dictionary.page.size` default $1 MB$) → falls back to plain encoding for remaining values

### Impact on Parquet

- `status` column with values `"active"`, `"inactive"`, `"pending"` - $3$ dictionary entries; each row stores $2$-bit index instead of $6$-$8$ byte string
- Compression ratio - often $10-100×$ for low-cardinality string columns
- Read performance - dictionary decoded once; comparisons done on integer indices (Parquet vectorized reader)

### Parquet Vectorized Reader in Spark

- `spark.sql.parquet.enableVectorizedReader=true` (default `true`) -
    - Reads data in column-oriented `ColumnarBatch` (Arrow-style)
    - Dictionary-encoded columns decoded lazily - indices decoded only for accessed rows
    - Filter evaluation on indices (not decoded strings) → avoids string materialization for non-matching rows
    - `spark.sql.parquet.columnarReaderBatchSize` (default $4096$) - rows per `ColumnarBatch`

---

## Parquet RLE & Bit-Packing

- __Run-Length Encoding (RLE)__ - encodes runs of repeated values as `(value, count)` pairs; efficient for sorted or low-cardinality data
- __Bit-Packing__ - packs integer values using only the bits required for their range; eg - values $0-7$ packed into $3$ bits each

### Hybrid RLE/Bit-Packing

- Parquet uses a hybrid scheme for integer data (including dictionary indices and definition/repetition levels) -
    - Switches between RLE and bit-packing dynamically per run
    - RLE used for long runs of the same value; bit-packing for varied values
    - Header byte indicates mode and run length

### Null Encoding

- Parquet uses `definition levels` to encode nulls -
    - Definition level = number of non-null ancestors in the nested schema path
    - For flat schema - definition level $0$ = null, $1$ = non-null
    - Definition levels stored via RLE/bit-packing; very compact for sparse nulls or sparse non-nulls

### Repetition Levels (for nested types)

- Parquet uses `repetition levels` to encode nested repeated fields (arrays, maps)
- Repetition level indicates at which level in the schema path the value repeats
- Enables Parquet's Dremel-style encoding of nested data into flat column storage

---

## Parquet Min/Max Statistics & Row Group Skipping

- __Row group skipping__ - using column-level min/max statistics to skip entire row groups without reading their data; the primary predicate pushdown mechanism in Parquet

### Statistics Storage

- Each column chunk stores in footer -
    - `min_value` / `max_value` - typed min and max of all values in that column chunk
    - `null_count` - number of null values
    - `distinct_count` (optional; expensive to compute)
- Spark writes these statistics during `df.write.parquet()`
- For string columns - statistics stored as bytes; comparisons are byte-wise; only `UNSIGNED_LEXICOGRAPHICAL` sort order supported

### Row Group Pruning in Spark

- `ParquetFileFormat` reads footer metadata before opening data
- For each row group, checks column chunk statistics against query predicates -
    - `col > 100` → skip row group if `max(col) <= 100`
    - `col = "Alice"` → skip row group if `min > "Alice"` OR `max < "Alice"`
    - `col BETWEEN 10 AND 20` → skip if `max < 10` OR `min > 20`
- Skipped row groups = no I/O for those rows; highly effective for sorted/clustered data

### Statistics Effectiveness

- Highly effective when data is sorted or clustered on filter columns -
    - `ORDER BY date` → each row group covers a small date range → tight statistics → most row groups pruned
- Ineffective for random/unsorted data -
    - Random data → each row group covers full value range → no pruning
- Z-ordering improves statistics effectiveness for multi-column filters by clustering similar values

### Page-Level Statistics (Parquet 2.x)

- Parquet 2.x adds page-level statistics (`column_index`, `offset_index`) for finer-grained skipping within a row group
- `spark.sql.parquet.recordLevelFilter.enabled` (default `false`) - enable page-level filtering
- Page-level skipping - skip individual pages within a column chunk; more granular than row group skipping

---

## Parquet Bloom Filters

- __Parquet Bloom Filter__ - a probabilistic data structure stored per column chunk; answers "is this value definitely NOT in this column chunk?" for equality queries
- Complements min/max statistics - min/max good for range queries; bloom filters good for equality on high-cardinality columns

### How Bloom Filters Work in Parquet

- Parquet Bloom Filters use Split Block Bloom Filter (SBBF) - a cache-friendly variant
- Each column chunk stores one bloom filter
- On write, each distinct value hashed and bits set in bloom filter
- On read, for equality predicate `col = value` -
    - Hash `value`; check bloom filter bits
    - If bits NOT set → value definitely not in chunk → skip chunk
    - If bits set → value MIGHT be in chunk → read chunk (possible false positive)
- False positive rate controlled by bloom filter size

### Spark Parquet Bloom Filter Config

- `spark.sql.parquet.bloomFilter.enabled=true` (default `false`) - enable bloom filter writes
- `spark.sql.parquet.bloomFilter.maxBytes` (default $1 MB$) - max bloom filter size per column
- Column-level enable -
```python
    df.write \
        .option("parquet.bloom.filter.enabled#user_id", "true") \
        .option("parquet.bloom.filter.expected.ndv#user_id", "1000000") \
        .parquet(path)
```
- Read-side filtering - `spark.sql.parquet.filterPushdown=true` (default `true`) and bloom filter in footer → used for chunk skipping

---

## Parquet Nested Types (arrays, maps, structs)

- Parquet stores nested data using Dremel encoding - flat columnar storage with definition and repetition levels encoding the nesting structure

### Struct Storage

- `StructType` stored as multiple leaf columns in Parquet - one column per leaf field
- `address.city` stored as column `address.city` with full nesting path
- Column pruning applies at leaf level - `SELECT address.city` reads only the `address.city` column

### Array Storage

- `ArrayType(IntegerType)` stored in two columns -
    - `list.element` - the actual values
    - Definition and repetition levels encode which row each element belongs to and whether element is null
- Spark reads arrays as sequences of values with associated levels; reconstructs `ArrayType` per row

### Map Storage

- `MapType` stored as two array columns - `key_value.key` and `key_value.value`
- Same Dremel encoding; keys and values aligned via repetition levels

### Nested Type Pushdown

- Spark can push filters on struct leaf fields - `filter(col("address.city") == "NYC")` → pushed as `EqualTo("address.city", "NYC")` to Parquet reader
- Cannot push filters inside arrays or maps - `filter(array_contains(col("tags"), "spark"))` not pushed to Parquet

---

## ORC Internals

- __ORC (Optimized Row Columnar)__ - columnar binary format developed by Hortonworks; alternative to Parquet; tighter integration with Hive; similar capabilities but different internal structure
- Spark fully supports ORC; DSv2-based vectorized reader available

### ORC vs Parquet

| Aspect | ORC | Parquet |
| --- | --- | --- |
| Origin | Hortonworks / Hive ecosystem | Cloudera / general |
| Footer location | End of file | End of file |
| Row grouping | Stripes | Row groups |
| Bloom filter | Built-in (default) | Optional (Parquet 2.x) |
| ACID support | Yes (with Hive ORC writer) | No |
| Vectorized reader | Yes (OrcColumnarBatchReader) | Yes (VectorizedParquetRecordReader) |
| Schema evolution | Good | Good |

---

## ORC Stripe Layout

- __Stripe__ - ORC's equivalent of Parquet's row group; a horizontal partition of rows
- Default stripe size - $64 MB$ (smaller than Parquet's $128 MB$ default)
- Each stripe contains -
    - __Index data__ - row index entries every $10000$ rows (configurable); stores min/max/bloom filter per index group
    - __Data__ - column data streams for all columns in the stripe
    - __Stripe footer__ - column stream metadata (offsets, lengths, encoding types)

### ORC Data Streams per Column

- ORC stores each column as multiple streams depending on type -
    - `PRESENT` stream - null bitmap (RLE encoded); optional if no nulls
    - `DATA` stream - primary column data
    - `LENGTH` stream - for strings/binary; lengths of variable-length values
    - `DICTIONARY_DATA` stream - for dictionary-encoded strings
    - `SECONDARY` stream - for complex types (map keys, struct fields)
- Each stream independently compressed; Spark reads only required streams (column pruning at stream level)

### ORC Row Index

- Every $10000$ rows within a stripe - an index entry stored
- Index entry contains -
    - Column statistics (min, max, sum, count) for that $10000$-row group
    - Byte offsets to resume reading at that position
- Used for fine-grained row-level skipping within a stripe

### File Footer

- ORC file footer contains -
    - File schema (type tree)
    - Per-stripe metadata (row count, byte offsets)
    - Per-column file statistics (min, max across entire file)
    - User metadata (key-value properties)
- Stripe statistics in footer enable file-level row group skipping (skip entire file if file-level stats don't match)

---

## ORC Bloom Filters & Predicate Pushdown

### ORC Bloom Filters

- ORC includes bloom filters by default (unlike Parquet where they're optional)
- One bloom filter per column per stripe (not per row index group)
- Stored in stripe index section
- `orc.bloom.filter.columns` - comma-separated list of columns to bloom filter; or `*` for all
- `orc.bloom.filter.fpp` (default $0.05$) - false positive probability; lower = larger filter

### ORC Predicate Pushdown in Spark

- `spark.sql.orc.filterPushdown=true` (default `true`) - enable ORC predicate pushdown
- `spark.sql.orc.splits.include.file.footer=true` - include file footer in split metadata; enables file-level pruning without separate footer read
- Pushdown uses -
    1. File-level statistics (from file footer) - skip entire files
    2. Stripe-level statistics - skip entire stripes
    3. Row index statistics - skip row groups within stripe
    4. Bloom filter lookup - skip stripe for equality predicates

### ORC Vectorized Reader

- `spark.sql.orc.impl=native` (default `native` in Spark 3+) - use Spark's native ORC reader
- `spark.sql.orc.enableVectorizedReader=true` (default `true`) - vectorized `ColumnarBatch` reads
- Native reader significantly faster than Hive ORC reader for analytical queries

---

## Avro in Spark

- __Avro__ - row-oriented binary serialization format with schema stored in the file (or schema registry); widely used for Kafka messages and Hadoop interchange
- Spark Avro support via `spark-avro` library (included in Spark distribution since 2.4)

### Avro in Spark Context

- Primary use - Kafka message deserialization and serialization in Structured Streaming
- `from_avro(col, jsonSchema)` / `to_avro(col)` - SQL functions for Avro conversion
- `spark.read.format("avro").load(path)` - read Avro files

### Avro Schema

- Schema stored in each Avro file header (`avro.schema.url` or inline JSON schema)
- Schema evolution - reader and writer schemas may differ; Avro resolution rules handle field addition/removal
- Schema registry integration (Confluent) - schema stored externally; messages contain schema ID -
```python
    from pyspark.sql.avro.functions import from_avro

    # Read Kafka value bytes as Avro
    df = spark.readStream.format("kafka").load()
    schema_str = '{"type": "record", "name": "Event", "fields": [{"name": "id", "type": "long"}]}'
    df.select(from_avro(col("value"), schema_str).alias("event"))
```

### Avro vs Parquet in Spark

- Avro - row-oriented; good for write-heavy (Kafka); poor for analytical reads (reads entire row for any column)
- Parquet - columnar; poor for streaming writes; excellent for analytical reads (skip columns, row groups)
- Typical pipeline - ingest via Avro (Kafka) → convert to Parquet at landing zone → query Parquet

### Avro Compression

- `spark.sql.avro.compression.codec` - `snappy` (default), `deflate`, `bzip2`, `xz`, `zstandard`
- `spark.sql.avro.deflate.level` - deflate compression level ($1-9$)

---

## CSV & JSON Parsing Internals

### CSV Parsing

- `spark.read.csv()` - file read via `TextInputRDD` then parsed per row using `UnivocityParser`
- Univocity - a high-performance CSV parser library; handles quoting, escaping, multi-line records
- __Non-splittable with compression__ - gzip-compressed CSV cannot be split; one partition per file; use bzip2 or uncompressed for parallelism

### CSV Schema Inference

- `inferSchema=true` - triggers TWO passes over data -
    1. Pass 1 - reads all values as strings; infers type per column by trying to parse as Long, Double, Boolean, Date, Timestamp, String in order
    2. Pass 2 - reads again with inferred schema
- `inferSchema=false` - all columns read as `StringType`; single pass; much faster
- Always use explicit schema in production - faster and type-safe

### CSV Corrupt Record Handling

- `mode` option -
    - `PERMISSIVE` (default) - corrupt rows stored in `_corrupt_record` column; processing continues
    - `DROPMALFORMED` - corrupt rows silently dropped
    - `FAILFAST` - throws exception on first corrupt row
- `columnNameOfCorruptRecord` (default `_corrupt_record`) - column name for corrupt row content

### JSON Parsing

- `spark.read.json()` - uses Jackson for single-line JSON; custom multi-line parser for `multiLine=true`
- Each line treated as one JSON object (default); schema inferred by scanning all records
- `samplingRatio` (default $1.0$) - fraction of records used for schema inference; set lower for faster inference on large files

### JSON Schema Inference

- `inferSchema=true` (default for JSON) - infers by scanning `samplingRatio` of records
- Column types inferred as widest type seen - if one record has `"age": 25` (int) and another has `"age": 25.5` (double), inferred as `DoubleType`
- Nested objects inferred as `StructType`; arrays as `ArrayType`
- `timestampFormat` option - pattern for parsing timestamp strings; if not specified, Spark tries multiple formats

### Performance Characteristics

- CSV and JSON are row-oriented text formats - Spark must read and parse ALL bytes even for single-column queries (no column pruning at I/O level)
- No statistics available - no row group skipping
- Predicate pushdown limited to partition directory pruning
- Always convert CSV/JSON landing data to Parquet/ORC for analytical workloads

> [!TIP]
> Format selection guidelines for Spark workloads -
> - __Parquet__ - default choice for analytical data; columnar; statistics; predicate pushdown; dictionary encoding; excellent compression
> - __ORC__ - Hive-native workloads; built-in bloom filters; ACID support
> - __Avro__ - Kafka source/sink; schema evolution; row-oriented streaming
> - __CSV/JSON__ - ingestion/landing zone only; convert to Parquet immediately
> - __Delta/Iceberg__ - when ACID, schema evolution, time travel, or incremental reads needed; layered on Parquet
