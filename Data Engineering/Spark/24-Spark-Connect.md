# Spark Connect

## Spark Connect Overview

- __Spark Connect__ - a decoupled client-server architecture for Spark introduced in Spark 3.4 and stabilized in Spark 4.x; separates the client-side DataFrame API from the server-side Spark execution engine via a well-defined gRPC protocol
- Before Spark Connect - every Spark client (Python, Scala, Java, R) required a direct JVM connection to the Driver; the client process WAS part of the Driver or tightly coupled to it
- With Spark Connect - the client is a thin process that builds a logical plan locally, serializes it as a Protobuf message, and sends it to a remote Spark Connect server over gRPC; the server executes and streams results back
- Spark 3.4 - preview; Spark 3.5 - production-ready; Spark 4.0 - default connection mode

### Why Spark Connect Exists

- __Stability__ - client bugs cannot crash the Driver; client and server have independent lifecycles
- __Upgradability__ - Spark server upgraded independently of client libraries; old clients continue working against new servers (protocol versioned)
- __Remote clients__ - any process anywhere on the network can connect to Spark without being co-located with the cluster; enables notebook servers, microservices, CI pipelines to use Spark without a local JVM
- __Multi-language parity__ - all languages get identical API behavior because they all go through the same server-side plan execution; no language-specific execution paths
- __Resource isolation__ - client process memory issues (large Python objects, memory leaks) cannot affect Spark Driver heap

---

## Spark Connect Architecture (client/server)

### Components

Client Process (Python/Scala/Java/R)
├── DataFrame API (same surface as classic)
├── LogicalPlan builder (local; no execution)
├── Protobuf serializer (plan → bytes)
└── gRPC stub (sends to server; receives results)
↓ gRPC over HTTP/2 (TLS optional)
Spark Connect Server (JVM; runs on or near Driver)
├── gRPC endpoint (receives plan bytes)
├── Protobuf deserializer (bytes → LogicalPlan)
├── SparkSession per client connection
├── Catalyst pipeline (analysis → optimization → physical plan)
├── Spark execution engine (RDD DAG; tasks on executors)
└── Arrow result stream (executes → streams ArrowBatch back to client)
↓ Arrow IPC stream
Client Process
└── receives ArrowBatch → reconstructs DataFrame/local result

### Client Side

- __`RemoteSparkSession`__ - the client-side `SparkSession`; surface API identical to classic `SparkSession`; internally builds `proto.Plan` objects instead of executing locally
- __`LogicalPlan` builder__ - every DataFrame transformation (`.filter()`, `.groupBy()`, `.join()`) builds a `proto.Relation` (Protobuf logical plan node) in the client process; zero execution at this stage
- __`SparkConnectClient`__ - wraps the gRPC stub; manages connection lifecycle, retries, and session ID
- __`ExecutePlanResponseReattachableIterator`__ - handles streaming response from server; supports reattaching if connection drops mid-stream

### Server Side

- __`SparkConnectService`__ - gRPC service implementation; one service instance per server
- __`SessionHolder`__ - holds a `SparkSession` per client session ID; sessions isolated per client
- __`SparkConnectPlanner`__ - deserializes `proto.Plan` → Spark `LogicalPlan`; translates every Protobuf relation type to its Catalyst equivalent
- __`ExecuteHolder`__ - manages one execution (one `execute_plan` RPC call); tracks state, caches results for reattachment
- __Result streaming__ - physical plan executed; results encoded as Arrow `RecordBatch`es via `ArrowConverters`; streamed back to client via gRPC server-streaming RPC

### Session Management

- Each client connection has a UUID session ID
- Server maintains `SessionHolder` map - `sessionId → SparkSession`
- Sessions are long-lived - persist across multiple RPC calls; temp views, UDFs, configs registered in one call visible in subsequent calls on the same session
- Session timeout - idle sessions cleaned up after configurable timeout (`spark.connect.session.ttl`, default $1h$)
- Session resurrection - if client reconnects with same session ID, server checks if session still exists; if expired, new session created (client gets `SessionNotFoundException`)

---

## gRPC Protocol

- __gRPC__ - Google's Remote Procedure Call framework; uses HTTP/2 for transport and Protocol Buffers (Protobuf) for serialization; enables bidirectional streaming and efficient binary encoding

### Why gRPC for Spark Connect

- __Streaming__ - gRPC server-streaming RPC allows server to push multiple result batches back to client without client polling; natural fit for streaming query results
- __Binary efficiency__ - Protobuf encoding is compact; much smaller than JSON for plan trees and result metadata
- __Language-agnostic__ - gRPC stubs generated for Python, Scala, Java, Go, Rust, etc. from same `.proto` files; all languages get identical wire protocol
- __HTTP/2__ - multiplexed streams; multiple concurrent RPCs over one connection; flow control; header compression

### Spark Connect Proto Schema

- Defined in `connector/connect/common/src/main/protobuf/spark/connect/` -
    - `base.proto` - `Plan`, `Command`, `Relation` (logical plan node)
    - `relations.proto` - all relation types (`Read`, `Filter`, `Project`, `Join`, `Aggregate`, `Sort`, `Limit`, `Union`, `WithColumns`, etc.)
    - `expressions.proto` - all expression types (`Literal`, `AttributeReference`, `Alias`, `Cast`, `UnresolvedFunction`, `ScalarSubquery`, etc.)
    - `commands.proto` - non-query operations (`WriteOperation`, `CreateDataFrameViewCommand`, `RegisterFunction`, etc.)
    - `types.proto` - `DataType` hierarchy matching Spark's type system

### Key RPC Methods

```protobuf
service SparkConnectService {
    // Execute a query plan; server streams result batches back
    rpc ExecutePlan(ExecutePlanRequest) returns (stream ExecutePlanResponse);

    // Analyze a plan without executing (schema, explain, etc.)
    rpc AnalyzePlan(AnalyzePlanRequest) returns (AnalyzePlanResponse);

    // Reattach to an existing execution (for fault tolerance)
    rpc ReattachExecute(ReattachExecuteRequest) returns (stream ExecutePlanResponse);

    // Release resources for an execution
    rpc ReleaseExecute(ReleaseExecuteRequest) returns (ReleaseExecuteResponse);

    // Interrupt/cancel a running execution
    rpc Interrupt(InterruptRequest) returns (InterruptResponse);

    // Fetch configuration
    rpc Config(ConfigRequest) returns (ConfigResponse);

    // Add artifact (JAR, Python file, etc.) to server session
    rpc AddArtifacts(stream AddArtifactsRequest) returns (AddArtifactsResponse);
}
```

### ExecutePlan Request/Response

- `ExecutePlanRequest` contains -
    - `session_id` - UUID identifying client session
    - `operation_id` - UUID for this specific execution; used for reattachment
    - `plan` - `Plan` Protobuf message (the serialized logical plan)
    - `request_options` - execution options (reattachable flag, etc.)

- `ExecutePlanResponse` (streamed) contains one of -
    - `arrow_batch` - `ArrowBatch { data: bytes, row_count: int64 }` - actual result data
    - `schema` - `DataType` of result
    - `sql_command_result` - result of a `CREATE TABLE`, `INSERT`, etc.
    - `result_complete` - sentinel; indicates stream finished
    - `metrics` - `ExecutionMetrics` per plan node (rows, bytes, time)
    - `observed_metrics` - `ObservedMetrics` for `df.observe()` calls

### Protobuf Plan Example

- `df.filter(col("age") > 18).select("name", "age")` serialized as -
```protobuf
    Plan {
        root: Relation {
            project: Project {
                input: Relation {
                    filter: Filter {
                        input: Relation {
                            read: Read {
                                named_table: NamedTable { unparsed_identifier: "users" }
                            }
                        }
                        condition: Expression {
                            unresolved_function: UnresolvedFunction {
                                function_name: ">"
                                arguments: [
                                    Expression { unresolved_attribute: { unparsed_identifier: "age" } }
                                    Expression { literal: { integer: 18 } }
                                ]
                            }
                        }
                    }
                }
                expressions: [
                    Expression { unresolved_attribute: { unparsed_identifier: "name" } }
                    Expression { unresolved_attribute: { unparsed_identifier: "age" } }
                ]
            }
        }
    }
```

### Reattachable Execution

- Long-running queries risk gRPC connection timeouts or client network interruptions
- Reattachable execution - server caches result batches for a short time after sending
- If client disconnects mid-stream - calls `ReattachExecute(session_id, operation_id, last_received_response_id)` to resume from last received batch
- `last_received_response_id` - each `ExecutePlanResponse` has a monotonic ID; client tracks last received; server resumes from next
- `spark.connect.execute.reattachable.enabled=true` (default) - enable reattachable execution

---

## Spark Connect vs Classic SparkSession

### API Surface

- Spark Connect client exposes the SAME `SparkSession` API as classic Spark -
    - `spark.read`, `spark.sql`, `spark.createDataFrame` - identical
    - DataFrame/Dataset transformations - identical
    - `spark.udf.register` - UDF serialized and sent to server via `AddArtifacts` RPC
- Differences and limitations -
    - `spark.sparkContext` - NOT available in Spark Connect client; `SparkContext` lives on server only
    - `sc.parallelize()`, `sc.textFile()` - NOT available; RDD API not exposed via Spark Connect
    - `spark.streams` streaming queries - supported; streaming execution runs on server; client gets progress updates
    - `df.rdd` - NOT available; RDD API inaccessible from thin client
    - Direct `BlockManager` or `AccumulatorContext` access - server-side only

### Execution Model Differences

| Aspect | Classic SparkSession | Spark Connect |
| --- | --- | --- |
| Client location | Same JVM as Driver or co-process | Any network-accessible machine |
| Plan building | Local Catalyst `LogicalPlan` objects | Local Protobuf `proto.Relation` objects |
| Plan optimization | Catalyst runs in client JVM (driver) | Catalyst runs on server |
| Result transport | Java object return or RDD | Arrow IPC stream over gRPC |
| Fault tolerance | Client crash = job dies | Client crash = server continues; reconnect possible |
| RDD API | Full access | Not available |
| SparkContext | Direct access | Not available |
| UDF execution | Client JVM calls UDF | UDF serialized; executed on server |

### Python Client Implementation

```python
# Classic SparkSession (co-located with driver)
from pyspark.sql import SparkSession
spark = SparkSession.builder \
    .master("yarn") \
    .appName("my-app") \
    .getOrCreate()

# Spark Connect client (remote; no local JVM needed)
from pyspark.sql import SparkSession
spark = SparkSession.builder \
    .remote("sc://spark-connect-server:15002") \   # Spark Connect endpoint
    .getOrCreate()

# API identical from this point
df = spark.read.parquet("hdfs://path/data")
result = df.filter(col("age") > 18).groupBy("country").count()
result.show()   # triggers ExecutePlan RPC; results streamed back as Arrow; printed locally
```

### Scala Client Implementation

```scala
// Spark Connect Scala client (thin; no local Spark execution)
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder()
    .remote("sc://spark-connect-server:15002")
    .getOrCreate()

// Identical DataFrame API
val df = spark.read.parquet("hdfs://path/data")
df.filter($"age" > 18).groupBy("country").count().show()
```

### Server Setup

```bash
# Start Spark Connect server
$SPARK_HOME/sbin/start-connect-server.sh \
  --master yarn \
  --conf spark.connect.grpc.binding.port=15002 \
  --conf spark.connect.grpc.maxInboundMessageSize=134217728 \
  --conf spark.connect.session.ttl=3600s
```

```python
# Or programmatically
spark = SparkSession.builder \
    .master("local[*]") \
    .config("spark.connect.grpc.binding.port", "15002") \
    .getOrCreate()
# spark.startConnectServer() in Spark 4.x
```

### Plan Translation on Server

- `SparkConnectPlanner` handles every `proto.Relation` type -
```scala
    def transformRelation(rel: proto.Relation): LogicalPlan = rel.getRelTypeCase match {
        case FILTER   => transformFilter(rel.getFilter)
        case PROJECT  => transformProject(rel.getProject)
        case JOIN     => transformJoin(rel.getJoin)
        case AGGREGATE => transformAggregate(rel.getAggregate)
        case READ     => transformRead(rel.getRead)
        case SORT     => transformSort(rel.getSort)
        case LIMIT    => transformLimit(rel.getLimit)
        // ... all relation types
    }
```
- Translated `LogicalPlan` submitted to `SparkSession.sessionState.executePlan()` - normal Catalyst pipeline from here

---

## Spark Connect vs Classic SparkSession

### Configuration

- `spark.conf.set()` works on Spark Connect client -
    - Serialized as `ConfigRequest` RPC to server
    - Applied to server-side `SparkSession` for this client
    - Affects subsequent query executions on that session
- `spark.conf.get()` - `ConfigRequest` RPC to server; returns server-side value

### Artifacts (JARs, Python Files, UDFs)

- `sc.addJar()` / `sc.addFile()` equivalent via `spark.addArtifact()` -
```python
    # Add Python file to server session
    spark.addArtifacts("my_module.py")

    # Add JAR
    spark.addArtifacts("my-library.jar")
```
- `AddArtifacts` RPC - client streams artifact bytes to server; server adds to session classpath
- UDF registration - `spark.udf.register("name", func)` serializes Python function via cloudpickle; sends as artifact; registers on server

---

## Thin Client Use Cases

- __Thin client__ - a Spark client that runs without a local Spark JVM, local Hadoop libraries, or cluster credentials; just the language SDK and a network connection to the Spark Connect server

### Notebook Servers

- Traditional Jupyter/Zeppelin - each notebook required a local SparkContext or a kernel that included Spark JARs; resource-heavy; kernel crash = SparkContext death
- With Spark Connect -
    - Notebook server is a thin process; no Spark JARs needed locally
    - Multiple notebooks connect to the same Spark Connect server; each gets an isolated session
    - Notebook restart does not affect the Spark server; reconnect and resume
    - Server can be a long-running shared Spark instance with pre-cached datasets

```python
# Notebook cell
from pyspark.sql import SparkSession
spark = SparkSession.builder.remote("sc://shared-spark:15002").getOrCreate()

# Each notebook user gets their own session; isolated temp views, configs
spark.sql("CREATE TEMP VIEW my_data AS SELECT * FROM events WHERE user_id = 'alice'")
```

### CI/CD and Testing

- Classic Spark testing required starting a local SparkContext (`local[*]`) in every test process -
    - Slow startup ($5-15s$ per test suite)
    - High memory per test process
    - Cannot test against real cluster behavior
- With Spark Connect -
    - CI test process connects to a remote Spark Connect server (dev/test cluster)
    - No local JVM initialization; fast test startup
    - Tests run against real cluster; catches issues invisible in local mode

```python
# pytest fixture with Spark Connect
import pytest
from pyspark.sql import SparkSession

@pytest.fixture(scope="session")
def spark():
    return SparkSession.builder \
        .remote("sc://test-spark-connect:15002") \
        .getOrCreate()

def test_filter(spark):
    df = spark.createDataFrame([(1, "alice"), (2, "bob")], ["id", "name"])
    result = df.filter(col("id") > 1).collect()
    assert len(result) == 1
    assert result[0]["name"] == "bob"
```

### Microservices and APIs

- Spark logic embedded in web services or microservices without shipping the full Spark runtime
- HTTP API calls Spark queries on-demand via Spark Connect client -
```python
    # FastAPI microservice using Spark Connect
    from fastapi import FastAPI
    from pyspark.sql import SparkSession
    from pyspark.sql.functions import col

    app = FastAPI()
    spark = SparkSession.builder.remote("sc://spark-connect:15002").getOrCreate()

    @app.get("/users/{user_id}/summary")
    def get_user_summary(user_id: str):
        result = spark.table("user_events") \
            .filter(col("user_id") == user_id) \
            .groupBy("event_type") \
            .count() \
            .collect()
        return {"user_id": user_id, "events": [r.asDict() for r in result]}
```

### Multi-Language Clients

- Spark Connect enables first-class clients in languages beyond Python/Scala/Java -
    - __Go client__ - `github.com/apache/spark-connect-go`; gRPC stub generated from same `.proto` files
    - __Rust client__ - community-maintained; same protocol
    - All languages get identical behavior because server-side execution is shared

### IDE Integration

- Spark Connect enables proper IDE support for PySpark -
    - Thin client library installable via `pip install pyspark` without Hadoop/Java
    - IDE auto-complete works on client-side API stubs
    - No need to configure `JAVA_HOME`, `SPARK_HOME`, `HADOOP_CONF_DIR` in development environment

### Edge Deployments

- Thin clients on edge nodes or IoT gateways -
    - Edge device runs lightweight Python client
    - Sends data to central Spark Connect server for processing
    - Results streamed back to edge device
    - No Spark runtime on edge; just network connectivity and Python SDK

### Limitations of Thin Clients

- __No RDD API__ - `sc.parallelize()`, `sc.textFile()`, custom `InputFormat`; must use DataFrame API
- __No SparkContext direct access__ - cannot call `sc.broadcast()`, `sc.accumulator()` directly; use DataFrame equivalents
- __No driver-side accumulators__ - accumulators not supported in Spark Connect (server-side execution; client cannot register accumulators)
- __Latency__ - gRPC adds network round-trip for every action; for local development, classic `local[*]` mode still faster
- __Streaming limitations__ - Structured Streaming supported but some sink types (e.g. `foreach` with local objects) require serialization to server

> [!NOTE]
> Spark Connect does NOT replace classic SparkSession for all use cases. For local development, unit testing with simple data, or jobs that need RDD-level control, classic mode remains appropriate. Spark Connect's primary value is in multi-tenant server deployments, remote clients, and long-running shared Spark instances where client isolation and independent lifecycle management are critical requirements.

> [!TIP]
> Spark Connect adoption pattern -
> ```
> Phase 1 - Start Spark Connect server alongside existing cluster
>   spark/sbin/start-connect-server.sh --master yarn
>
> Phase 2 - Migrate notebook server to thin clients
>   Replace SparkContext-per-notebook with SparkSession.builder.remote(...)
>   Verify session isolation (temp views, configs don't leak between users)
>
> Phase 3 - Migrate CI/CD to remote execution
>   Point test fixtures at dev Spark Connect server
>   Decouple test container from Spark JVM dependency
>
> Phase 4 - Expose as internal API
>   Wrap Spark Connect client in internal SDK
>   Microservices use SDK to query data without knowing Spark internals
> ```