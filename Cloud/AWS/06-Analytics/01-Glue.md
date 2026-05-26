# Glue

- Fully managed, serverless data integration service designed to make it easy to discover, prepare, and combine data for analytics, machine learning, and application development.
- Removes the need to manage infrastructure while providing deep ETL capabilities.
- __Features__ - 
  - Serverless - No servers to manage. AWS provisions, scales, and manages compute resources automatically. Users pay only for resources consumed.
  - Automatic schema discovery - Glue can automatically detect table definitions and schemas from structured, semi-structured, or unstructured data.
  - Central metadata repository - The Glue Data Catalog serves as a single source of truth for all metadata, tables, partitions, and schema versions.
  - ETL capabilities - Glue can extract data from sources, transform it, and load it into targets using Apache Spark under the hood.
  - Integration with AWS ecosystem - Works seamlessly with S3, Athena, Redshift, EMR, RDS, Kinesis, Lake Formation, and SageMaker.

- __Solves two main problems__ -
  - Schema discovery - Automatically understands the structure of incoming data without manual intervention.
  - Data transformation - Provides a scalable ETL engine capable of transforming data via Python/Scala code or through a visual interface.

## Glue Data Catalog

- Backbone of Glue.
- Acts as a central metadata repository, storing only metadata, not actual data.
- This allows unstructured data to be treated as structured for analytics and machine learning workflows.
- __Features__ -
  - Schema storage - Maintains table name, column definitions, data types, and partition keys.
  - Partition tracking - Tracks S3 partitions for improved query performance.
  - Statistics - Stores metadata statistics such as row counts, file sizes, last modified timestamps, etc.
  - Hive Metastore compatibility - Can act as a Hive Metastore, allowing Athena or EMR to query tables with standard Hive SQL. Conversely, Hive metastores can be imported into Glue.
  - Automatic discovery - Glue Crawlers automatically populate and update the catalog.
  - Cross-service integration - Catalog is accessible by Athena, Redshift Spectrum, EMR, SageMaker, and more.

## Glue Crawlers

- Scan data sources, infer schemas, and populate the Data Catalog.
- Can handle structured, semi-structured (JSON, XML), or unstructured data.
- __Behavior__ -
  - Reads data from S3 or JDBC sources.
  - Automatically infers column data types (string, int, float, boolean, timestamp, etc.).
  - Detects partitions based on folder structures in S3.
  - Can be scheduled periodically or triggered programmatically.
- __Crawler output__ -
  - Creates new tables in the Glue Data Catalog.
  - Updates schemas for existing tables.
  - Adds new partitions without modifying existing ones.
- __Best Practices__ -
  - Partition S3 Data Strategically - Organize buckets based on query patterns.
    - Example -
      - Query primarily by time ranges → `yyyy/mm/dd/deviceId`
      - Query primarily by device → `deviceId/yyyy/mm/dd`
  - Provide hints for schema inference when data is ambiguous (e.g., JSON fields with mixed types).
  - Avoid unnecessary frequent crawler runs on very large datasets to reduce cost.

## Glue ETL

- Glue’s ETL engine is serverless and powered by Apache Spark.
- Supports Python (PySpark) and Scala.
- Automatic job generation via Glue Studio visual interface.
- Jobs can be triggered on-demand, on schedule, or via events.
- Handles encryption -
  - Server-side encryption for data at rest.
  - SSL/TLS for data in transit.
- Can provision additional DPUs (Data Processing Units) to increase performance of underlying Spark jobs.
  - Metrics help monitor maximum DPU usage, plot allocated vs needed executors, and analyze ETL movement.
- Error reporting is integrated with CloudWatch and can be sent to SNS for notifications.
- Can read from S3, JDBC, Kinesis, Kafka and write to S3, JDBC targets, Redshift, or Glue Data Catalog.

### `DynamicFrame`

- Glue’s core abstraction for ETL.
- Extend Spark DataFrames with schema-aware, self-describing records optimized for ETL.
- A `DynamicFrame` is a collection of `DynamicRecords` -
  - Each record carries its schema.
  - Supports nested structures and schema evolution.
- Provides ETL-specific transformations like `applyMapping`, `resolveChoice`, and `dropNullFields`.
- Example -
```
val pushdownEvents = glueContext.getCatalogSource(
  database = "githubarchive_month", tableName = "data"
)

val projectedEvents = pushdownEvents.applyMapping(Seq(
  ("id", "string", "id", "long"),                 // converts the ids from strings to long
  ("type", "string", "type", "string"),
  ("repo.name", "string", "repo", "string"),      // renames repo.name to repo
  ("org.login", "string", "org", "string")
))
```

- __Common Transformations__ -

| Transformation              | Purpose                                                                          |
| --------------------------- | -------------------------------------------------------------------------------- |
| DropFields / DropNullFields | Remove unnecessary columns or null values                                        |
| Filter                      | Filter rows based on a function or condition                                     |
| Join                        | Combine data from multiple sources                                               |
| Map                         | Add, modify, or remove fields, perform lookups                                   |
| ML Transformations          | **FindMatches ML**: Detect duplicates even without unique identifiers            |
| Format Conversion           | Convert between CSV, JSON, Avro, Parquet, ORC, XML                               |
| Advanced Spark Operations   | Supports all Spark capabilities, including clustering, grouping, and aggregation |

## Handling Schema Ambiguities — `resolveChoice`

- Used when columns have multiple types (e.g., `price` as both `string` and `double`).
- Resolutions -
  - `make_cols` - creates separate columns for each type, eg - `price_double`, `price_string`.
  - `cast` - converts all values to a specified type.
  - `make_struct` - packs multiple types into a nested struct.
  - `project` - forces all values to a single type.

- __Usage__ -
```
"myList": [{"price": 100.00, "price": "$100.00"}]
df1 = df.resolveChoice(choice = "make_cols")
df2 = df.resolveChoice(specs = [("myList[].price", "make_struct"), ("columnA", "cast:double")])
```

## Data Catalog Updates via ETL

- ETL scripts can create, update, or extend table definitions and partitions.
- Adding new partitions - use `enableUpdateCatalog` with `partitionKeys`.
- Updating table schema - `enableUpdateCatalog` + `updateBehavior`.
- Creating new tables - `enableUpdateCatalog` + `updateBehavior` + `setCatalogInfo`.
- Restrictions -
  - Only works with S3 sources.
  - Supported formats - JSON, CSV, Avro, Parquet (Parquet requires special handling).
  - Nested schemas are not supported for automatic updates.

## Job Scheduling and Orchestration

- __Time-based schedules__ using cron expressions.
- __Event-driven triggers__ -
  - S3 object creation
  - Lambda function invocations
  - Kinesis or Kafka events
- __Workflows__ - orchestrate multiple jobs and crawlers with conditional logic.
- __Job bookmarks__ -
  - Persist job state to avoid reprocessing old data.
  - Track new rows in JDBC or S3 sources.

## Glue Studio

- Visual ETL editor with drag-and-drop interface.
- Generates Python/Scala code automatically.
- Job dashboard for monitoring status, duration, and performance metrics.
- Supports partitioning, sampling, and DataBrew integration for data prep.

## Glue DataBrew

- No-code, visual data preparation tool.
- Handles cleaning, normalization, masking, and PII management.
- Supports over 250 transformations.
- Can read/write from S3, Redshift, Snowflake, RDS.
- __Recipes__ are reusable, can be executed as Glue jobs.
- Supports data quality rules integrated with Glue ETL.
- May create datasets with custom SQL from Redshit and Snowflake.
- Security -
  - Can integrate with KMS (with customer master keys only).
  - SSL in transit.
  - IAM permissions enforce access control.
  - CloudWatch & CloudTrail for auditing.

```
{
  "RecipeAction": {
    "Operation": "NEST_TO_MAP",
    "Parameters": {
      "sourceColumns": "[\"age\", \"weight_kg\", \"height_cm\"]",
      "targetColumn": "columnName",
      "removeSourceColumns": "true"
    }
  }
}
```

## Handling Personally Indentifiable Information (PII) in DataBrew Transformations

- Substitution (`REPLACE_WITH_RANDOM`)
- Shuffling (`SHUFFLE_ROWS`) - shuffle PII values in a row and shuffle them.
- Deterministic Encryption (`DETERMINISTIC_ENCRYPT`) - given value always results in the same encrypted value.
- Probabilistic Encryption (`ENCRYPT`) - more than one result from an encryption of a given field.
- Decryption (`DECRYPT`)
- Nulling out or deletion (`DELETE`)
- Masking out (`MASK_CUSTOM`, `_DATE`, `_DELIMITER`, `_RANGE`)
- Hashing (`CRYPTOGRAPHIC_HASH`) - multiple values can hash to the same result.

## Glue Flex Jobs

- Execution class ("FLEX") for non-urgent ETL workloads -
  - Use spare compute capacity instead of dedicated workloads.
  - Might have to wait for capacity to become available.
- Appropriate for testing, one-time workloads, or time-insensitive ETL.
- Lower cost (~30–35% savings).
- Not supported -
  - Python shell jobs
  - Streaming jobs
  - AWS Glue ML jobs
- When to use -
  - Standard - Predictable start time and performance, hard SLAs.
  - Flex - Variable start/performance, cheaper.

## Glue Data Quality

- Evaluate datasets against custom or recommended rules using Data Quality Definition Language (DQDL).
- Can be created manually or recommended automatically.
- Integrates with Glue jobs using transformer - "Evaluate Data Quality".
- Uses Data Quality Definition Language (DQDL).
- Results can be used to fail the job, or just be reported to CloudWatch.
- A DQDL document consists of a _ruleset_, for eg -
```
Rules = [
  -- Dataset must have at least 100 rows
  RowCount >= 100,

  -- Every customer_id must be present (no NULLs)
  IsComplete, "customer_id", 

  -- Every customer_id must be unique
  IsUnique, "customer_id",

  -- At least 99% of emails must be non-NULL
  Completeness "email" > 0.99

  -- Age must be between 0 and 120
  ColumnValues "age" between 0 and 120

  -- Singup date must be within last 3 years
  ColumnValues "signup_date" > (now() - 3 years)
]
```

- Lots of built-in rule types like Mean, Sum, CustomSQL, Uniqueness, DetectAnomalies etc.
- Complete list - https://docs.aws.amazon.com/glue/latest/dg/dqdl.html
- Composite rules (and, or) -
  - `(ColumnValues "age" > 30) AND (ColumnValues "salary" > 1000)`
  - Evaluates left to right and will fail early, unlike SQL.
    - Unless you specify otherwise with `additionalOptions` for `compositeRuleEvaluation`.
- Supports -
  - Expressions (`=`, `!=`, `<`, `>` etc)
  - Filters (where clauses)
  - Constants (storing common SQL snippets)
  - Labels (apply all rules with a given label, for a given team)

## Cost Model

- Billed by the second for crawler and ETL jobs.
- First million objects stored and accesses are free for the Glue Data Catalog.
- Development endpoints for developing ETL code is charged by the minute.

## Anti-Patterns

- Avoid using multiple ETL engines for the same dataset (Hive, Pig, etc.).
- Streaming ETL is no longer a limitation -
  - Glue now supports serverless Spark Structured Streaming from Kinesis/Kafka.

## Glue Workflows

- Orchestrates complex ETL operations within Glue.
- Design multi-job, multi-crawler ETL processes run together.
- Create form AWS Glue blueprint, from the console or API.
- Triggers within workflows start jobs or crawlers, or can be fired when jobs or crawlers complete.
- Schedule based on cron expression.
- On demand.
- EventBridge events -
  - Start on single event or batch of events.
  - For eg - arrival of a new object in S3.
  - Optional batch conditions -
    - Batch size (number of events).
    - Batch window (within X seconds, default is 15 mins).
