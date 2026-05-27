# Serialization Deep Dive

## Java Serialization in Spark

- __Java serialization__ - the JVM's built-in object serialization mechanism; objects implementing `java.io.Serializable` converted to byte streams via `ObjectOutputStream`; deserialized via `ObjectInputStream`
- Spark's default serializer before Kryo is configured; still used for certain internal paths regardless of `spark.serializer` setting

### Where Java Serialization is Used in Spark

- __Task results__ - `ResultTask` results smaller than `spark.task.maxDirectResultSize` serialized for return to Driver; uses configured serializer (Kryo if set)
- __RPC messages__ - ALL RPC messages between Driver and Executors use Java serialization regardless of `spark.serializer` - `AccumulatorV2` updates, `TaskDescription`, `MapStatus`, RPC endpoint messages
- __Broadcast variables__ (fallback) - if broadcast object is not Kryo-serializable, falls back to Java serialization
- __Python UDF closure objects__ - Python-side objects use Python's pickle; JVM side uses Java serialization for wrapper
- __Streaming state__ - legacy `HDFSBackedStateStore` uses Java serialization for some state entries

### Java Serialization Mechanics

- `ObjectOutputStream.writeObject(obj)` -
    1. Writes class descriptor (fully qualified class name, serialVersionUID, field descriptions)
    2. Recursively writes each field's value
    3. For object graphs - tracks already-serialized objects to handle cycles; stores back-references
- Output - verbose; includes full class metadata per object; large size
- Performance - slow; reflective field access; no type-specific optimization; creates many intermediate objects

### Java Serialization Problems in Spark

- __Size__ - a `HashMap[String, Int]` with $1000$ entries serializes to ~$50 KB$ in Java; ~$8 KB$ in Kryo
- __Speed__ - $10-20×$ slower than Kryo for typical Spark workloads
- __Closure serialization__ - task closures must be Java-serializable; capturing non-serializable objects causes `NotSerializableException` at task dispatch time
- __`serialVersionUID` issues__ - if class definition changes between serialization and deserialization (eg - different JAR versions on Driver vs Executor) → `InvalidClassException`

### NotSerializableException Debugging

- Common trigger - a Scala/Java class captured in a task closure that does not implement `Serializable`
- Python -
```python
    # This pattern causes NotSerializableException
    class MyProcessor:
        def __init__(self):
            self.conn = DatabaseConnection()   # not serializable

        def process(self, rdd):
            # 'self' captured → tries to serialize DatabaseConnection
            return rdd.map(lambda x: self.conn.lookup(x))   # FAILS

    # Fix - don't capture non-serializable objects in closure
    def process(rdd):
        def lookup(x):
            conn = DatabaseConnection()   # created inside task; not serialized
            return conn.lookup(x)
        return rdd.map(lookup)
```

- Scala -
```scala
    // FAILS - Connection not Serializable
    class MyTransformer {
        val conn = new DatabaseConnection()
        def transform(rdd: RDD[String]): RDD[String] =
            rdd.map(x => conn.lookup(x))   // captures 'this' including conn
    }

    // Fix option 1 - make class Serializable with @transient
    class MyTransformer extends Serializable {
        @transient lazy val conn = new DatabaseConnection()   // not serialized; recreated on executor
        def transform(rdd: RDD[String]): RDD[String] =
            rdd.map(x => conn.lookup(x))
    }

    // Fix option 2 - extract to local val (ClosureCleaner handles this)
    class MyTransformer {
        def transform(rdd: RDD[String]): RDD[String] = {
            val localConn = conn    // if conn is serializable, extract to local val
            rdd.map(x => localConn.lookup(x))
        }
    }
```

### @transient Annotation

- `@transient` in Scala/Java - marks a field as NOT serialized by Java serialization
- Field is `null` after deserialization on executor; must be lazily re-initialized
- `@transient lazy val` pattern - field not serialized; re-created on first access on executor

---

## Kryo Serialization Deep Dive

- __Kryo__ - a fast, compact Java binary serialization library; $3-10×$ faster and $3-5×$ smaller than Java serialization for typical Spark workloads
- Enabled via `spark.serializer=org.apache.spark.serializer.KryoSerializer`
- Used for RDD shuffles, RDD persistence (serialized storage levels), task results, and broadcast when configured

### Why Kryo is Faster

- __No class metadata in stream__ - registered classes stored by integer ID (2 bytes) not full class name
- __Type-specific serializers__ - Kryo has hand-optimized serializers for common types (`String`, `ArrayList`, `HashMap`, arrays, primitives); avoids reflection
- __Field-level access__ - uses `sun.misc.Unsafe` for direct field access in generated serializers; avoids reflection overhead
- __Reusable buffers__ - `Kryo` instance and `Output` buffer reused across calls; no per-call allocation
- __Object pooling__ - `Kryo` instances pooled per thread; no synchronization on common path

### Kryo Serialization Mechanics

- `KryoSerializer.newInstance()` → `KryoSerializerInstance` - wraps a `Kryo` instance + `Output` buffer
- `serializerInstance.serialize(value: T): ByteBuffer` -
    1. Gets `Kryo` from pool
    2. Writes class registration ID (2 bytes if registered; variable-length class name if not)
    3. Calls type-specific serializer's `write(kryo, output, object)` method
    4. Returns byte buffer
- `serializerInstance.deserialize(bytes: ByteBuffer): T` -
    1. Reads class ID → looks up registered class
    2. Calls type-specific serializer's `read(kryo, input, type)` method
    3. Returns reconstructed object

### Kryo Buffer Configuration

- `spark.kryoserializer.buffer` (default $64 KB$) - initial size of Kryo serialization buffer per serializer instance
- `spark.kryoserializer.buffer.max` (default $64 MB$) - maximum buffer size; if object exceeds this → `KryoException: Buffer overflow`
- Increase `buffer.max` for large objects in shuffle/broadcast:
```python
    spark.conf.set("spark.kryoserializer.buffer.max", "512m")
```

### Kryo vs Java for Different Types

| Type | Java size | Kryo registered | Kryo unregistered |
| --- | --- | --- | --- |
| `String("hello")` | $57$ bytes | $7$ bytes | $28$ bytes |
| `List[Int](1,2,3)` | $~100$ bytes | $~15$ bytes | $~40$ bytes |
| `Map[String,Int](size=100)` | $~5000$ bytes | $~800$ bytes | $~1500$ bytes |
| Custom case class | $~200$ bytes | $~30$ bytes | $~80$ bytes |

---

## Kryo Registration & Custom Serializers

### Why Registration Matters

- __Registered class__ - stored in Kryo stream as integer ID (2 bytes); fast lookup; compact
- __Unregistered class__ - stored as full class name string; verbose; slower; requires class resolution on read
- `spark.kryo.registrationRequired=true` - throw exception if unregistered class encountered; forces registration discipline

### Registration Methods

- Python -
```python
    spark = SparkSession.builder \
        .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
        .config("spark.kryo.registrationRequired", "false") \
        .config("spark.kryo.classesToRegister",
                "com.example.MyClass,com.example.OtherClass") \
        .getOrCreate()
```

- Scala - via `KryoRegistrator` (recommended for many classes) -
```scala
    import com.esotericsoftware.kryo.Kryo
    import org.apache.spark.serializer.KryoRegistrator

    class MyKryoRegistrator extends KryoRegistrator {
        override def registerClasses(kryo: Kryo): Unit = {
            // Register without custom serializer - Kryo uses field-based default
            kryo.register(classOf[MyEvent])
            kryo.register(classOf[UserProfile])
            kryo.register(classOf[Array[MyEvent]])

            // Register with custom serializer
            kryo.register(classOf[MyComplexClass], new MyComplexClassSerializer)

            // Register with explicit ID (ensures stable IDs across restarts)
            kryo.register(classOf[MyEvent], 100)
            kryo.register(classOf[UserProfile], 101)
        }
    }

    // In SparkConf
    spark = SparkSession.builder()
        .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
        .config("spark.kryo.registrator", "com.example.MyKryoRegistrator")
        .getOrCreate()
```

### Custom Kryo Serializer

- Implement `com.esotericsoftware.kryo.Serializer[T]` for classes with complex state or external dependencies -
```scala
    import com.esotericsoftware.kryo.{Kryo, Serializer}
    import com.esotericsoftware.kryo.io.{Input, Output}

    class BloomFilterSerializer extends Serializer[BloomFilter] {
        override def write(kryo: Kryo, output: Output, bf: BloomFilter): Unit = {
            val bytes = bf.toByteArray
            output.writeInt(bytes.length)
            output.writeBytes(bytes)
        }

        override def read(kryo: Kryo, input: Input, `type`: Class[BloomFilter]): BloomFilter = {
            val length = input.readInt()
            val bytes = input.readBytes(length)
            BloomFilter.fromByteArray(bytes)
        }
    }

    // Register
    kryo.register(classOf[BloomFilter], new BloomFilterSerializer)
```

### Registration Stability

- Kryo IDs assigned in registration order if not explicitly set
- Registration order must be IDENTICAL on Driver and Executor
- If using `KryoRegistrator` - class loaded and executed on both Driver and all Executors
- Stable explicit IDs (via `kryo.register(cls, id)`) - recommended for production; prevents ID shift if class registration order changes
- IDs $0$-$9$ reserved by Kryo internals; start custom IDs at $10$+

---

## Serialization in Shuffle

- __Shuffle serialization__ - data written to shuffle files must be serialized; deserialized on the reduce side when fetched
- Shuffle is the highest-volume serialization path in Spark; serialization overhead here dominates overall job time for data-intensive workloads

### RDD Shuffle Serialization

- Configured via `spark.serializer` (default Java; should be Kryo)
- `SortShuffleWriter` → `ExternalSorter.insertAll` → records serialized using `SerializerInstance`
- Serialized bytes written directly to shuffle data file
- On reduce side - `BlockStoreShuffleReader` deserializes fetched bytes using same `SerializerInstance`

### DataFrame/SQL Shuffle Serialization

- DataFrame shuffles use Tungsten's `UnsafeRow` binary format DIRECTLY - no additional serialization
- `UnsafeRow` is already in binary wire format; shuffle write = `memcpy` of binary row bytes
- `UnsafeRowSerializer` - the serializer used for DataFrame shuffles; essentially a no-op wrapper that copies raw `UnsafeRow` bytes
- Zero serialization overhead for DataFrame shuffles - this is why DataFrames are faster than RDDs for shuffles

### SerializerManager Compression

- `SerializerManager` wraps the serializer with optional compression -
    - `spark.shuffle.compress=true` (default) - compresses shuffle data with configured codec
    - `spark.shuffle.spill.compress=true` (default) - compresses spill files
    - Compression applied AFTER serialization; compressed bytes written to disk/network
- For already-compressed data (Parquet files in shuffle) - compression adds CPU cost with minimal size benefit; consider disabling

### Serialization and SortShuffleWriter Write Paths

- `UnsafeShuffleWriter` (used for DataFrame operations) -
    - Records already serialized (`UnsafeRow` bytes) BEFORE insertion into sort buffer
    - Sort operates on serialized bytes - no deserialization during sort
    - Zero-copy path from `UnsafeRow` → shuffle file bytes

- `SortShuffleWriter` (used for RDD operations with aggregation) -
    - Records deserialized (JVM objects) in `PartitionedAppendOnlyMap`
    - Serialized only when spilling or writing final output
    - Higher memory usage; GC pressure from JVM objects

---

## Serialization in Broadcast

- __Broadcast serialization__ - the broadcast value must be serialized on the Driver for chunked distribution to executors via `TorrentBroadcast`

### Serialization Path

1. `SparkContext.broadcast(value)` called on Driver
2. `TorrentBroadcast` serializes `value` using configured serializer (`spark.serializer`) -
    - Kryo if configured; Java serialization otherwise
    - Result is a byte array
3. Byte array compressed using `spark.broadcast.compress` codec (default `lz4`)
4. Compressed bytes split into chunks of `spark.broadcast.blockSize` (default $4 MB$)
5. Chunks stored in Driver's `BlockManager` as `BroadcastBlockId("piece0")`, `("piece1")`, ...

### Deserialization on Executor

1. Executor fetches all chunks via `TorrentBroadcast` protocol
2. Chunks decompressed and concatenated back into serialized byte array
3. Byte array deserialized using same serializer as step 2 → reconstructed Java object
4. Deserialized object cached in executor `BlockManager` as `BroadcastBlockId(id)` (not chunk IDs)
5. `bc.value` returns cached deserialized object; subsequent calls return same object (no re-deserialization)

### Broadcast and Kryo

- Large broadcast variables (ML models, large lookup tables) - Kryo produces $3-5×$ smaller byte arrays
- Smaller chunks = faster `TorrentBroadcast` distribution; less network traffic
- Always configure Kryo when broadcasting objects > $10 MB$

### Broadcast Serialization Limits

- `spark.kryoserializer.buffer.max` must be large enough to serialize the broadcast object in one shot
- A $500 MB$ broadcast object requires `buffer.max >= 500m`
- Java serialization has no explicit limit; Kryo has the buffer limit
- For very large objects - consider splitting into smaller broadcasts or using a distributed lookup service

---

## Serialization in Task Closure

- __Task closure__ - the serialized state that a task carries to the executor; includes the RDD operation function and all captured variables

### Closure Serialization Path

1. `TaskSchedulerImpl.resourceOffers` assigns task to executor slot
2. Task object created - `ShuffleMapTask` or `ResultTask`
3. `Task.serialize(task)` called - serializes the task object using configured `SparkEnv.serializer`
4. Serialized task stored in `TaskDescription.serializedTask: ByteBuffer`
5. `TaskDescription` sent to executor via RPC (`LaunchTask` message)
6. On executor - `TaskRunner.run()` deserializes task from `serializedTask`

### What the Closure Contains

- The RDD compute function (the lambda) - for `map`, `filter`, `reduce` etc.
- ALL variables referenced by the lambda that are captured from enclosing scope
- For `ShuffleMapTask` - also the `ShuffleDependency` and partition info
- For `ResultTask` - also the result function

### ClosureCleaner

- `SparkContext.clean(closure)` - called before task serialization
- Uses ASM bytecode analysis to null out fields in captured outer classes that are NOT referenced by the closure
- Reduces closure size; prevents accidental large object serialization

### Closure Size Limit

- `spark.rpc.message.maxSize` (default $128 MB$) - maximum serialized RPC message size
- A task with a closure exceeding this → `SparkException: Serialized task ... was ... bytes, which exceeds max allowed: ...`
- Common cause - accidentally capturing a large `HashMap`, `Array`, or DataFrame/RDD in closure
- Fix - use `sc.broadcast(large_obj)` and reference `broadcastVar.value` inside closure

### Closure Serialization Debugging

- Python -
```python
    import pickle

    # Check what's being serialized in closure
    large_dict = {i: str(i) for i in range(1000000)}

    # BAD - large_dict captured in closure
    def bad_map(x):
        return large_dict.get(x, "unknown")

    # Check size
    serialized = pickle.dumps(bad_map)
    print(f"Closure size: {len(serialized)} bytes")  # will be large

    # GOOD - broadcast instead
    bc = sc.broadcast(large_dict)
    def good_map(x):
        return bc.value.get(x, "unknown")

    serialized = pickle.dumps(good_map)
    print(f"Closure size: {len(serialized)} bytes")  # tiny - just bc reference
```

---

## Arrow & Columnar Serialization

- __Apache Arrow__ - a language-independent columnar memory format; zero-copy reads between different processes and languages; the serialization format for Pandas UDFs and columnar batch transfers

### Arrow Format in Spark

- Arrow used in two contexts in Spark -
    1. __Pandas UDF data transfer__ - JVM → Python process communication for vectorized UDFs
    2. __`spark.sql.execution.arrow.pyspark.enabled=true`__ - `df.toPandas()` and `spark.createDataFrame(pandas_df)` use Arrow instead of row-by-row pickle

### Arrow Memory Format

- Columnar - values for each column stored contiguously (not row-by-row)
- `RecordBatch` - a table fragment; N rows across M columns; each column stored as `Array`
- `Array` - contiguous buffer of typed values + validity (null) bitmap
- Schema stored separately from data - shared once; not repeated per batch

### JVM → Python Transfer (Pandas UDF)

1. JVM accumulates `ColumnarBatch` (Arrow format) of `spark.sql.execution.arrow.maxRecordsPerBatch` rows
2. Arrow `RecordBatch` written to `ArrowStreamWriter` → byte stream on socket
3. Python worker reads from socket via `pyarrow.ipc.RecordBatchStreamReader`
4. Python reads Arrow buffers as `pd.DataFrame` or `pd.Series` -
    - For numeric types (int, double, float) - ZERO COPY; `pd.Series` directly wraps Arrow buffer
    - For strings - copy required (Python string objects)
5. User function executed on `pd.Series`/`pd.DataFrame`
6. Result written back as Arrow `RecordBatch` via `ArrowStreamWriter`
7. JVM reads result → converts to `InternalRow`s

### Arrow IPC Format

- Two Arrow IPC formats -
    - __Stream format__ (`ArrowStreamWriter`) - sequential; no random access; used for Pandas UDF pipes
    - __File format__ (`ArrowFileWriter`) - has footer with record batch offsets; supports random access; used for `df.toPandas()` with `spark.sql.execution.arrow.pyspark.fallback.enabled`

### Arrow Columnar Batch in Spark SQL

- `ColumnarBatch` - Spark's wrapper for Arrow-style columnar data in the SQL engine
- `OnHeapColumnVector` / `OffHeapColumnVector` - column arrays for each type
- Used by vectorized Parquet and ORC readers - `VectorizedParquetRecordReader` returns `ColumnarBatch`
- `BatchScanExec` with columnar sources - operators downstream can receive `ColumnarBatch` and process column-at-a-time (faster than row-at-a-time for numeric operations)

### df.toPandas() with Arrow

- `spark.conf.set("spark.sql.execution.arrow.pyspark.enabled", "true")` -
    - Collects DataFrame to Driver as Arrow `RecordBatch`es (not serialized rows)
    - Converts to `pd.DataFrame` with Arrow zero-copy for numeric columns
    - $5-10×$ faster than default row-based collect for wide DataFrames

---

## Tungsten Serialization

- __Tungsten serialization__ - operating directly on binary `UnsafeRow` data; not a traditional serialization library but rather a zero-overhead binary layout that IS the wire format

### UnsafeRow as Wire Format

- `UnsafeRow` binary layout designed to be BOTH the in-memory execution format AND the serialized format
- No serialization step for DataFrame shuffle writes -
    - `ShuffleExchangeExec` writes `UnsafeRow.getBaseObject()` bytes directly to shuffle file
    - No `serialize(row)` call; `memcpy` of the binary region is sufficient
    - On read - `UnsafeRow` object pointed at the deserialized bytes; `baseOffset` set; `sizeInBytes` set; ready to use

### UnsafeRowSerializer

- `UnsafeRowSerializer` - the serializer used for DataFrame shuffles; wraps raw byte copy operations
- `serialize(row: UnsafeRow): ByteBuffer` -
    - Writes 4-byte row length prefix
    - Copies `row.sizeInBytes` bytes from `row.getBaseObject()` at `row.getBaseOffset()`
- `deserialize(bytes: ByteBuffer): UnsafeRow` -
    - Reads 4-byte length
    - Creates `UnsafeRow` pointing at the byte buffer region; no copy, no decode

### Benefits Over Java/Kryo for DataFrames

| Aspect | Kryo (RDD) | Tungsten (DataFrame) |
| --- | --- | --- |
| Serialization cost | CPU: field traversal + write | CPU: memcpy only |
| Deserialization cost | CPU: read + object creation | CPU: pointer assignment only |
| Object creation | One per record | Zero (reuse UnsafeRow wrapper) |
| GC pressure | Per-record objects | None (off-heap or array-backed) |
| Wire size | Compact but has Kryo metadata | Exact binary size (no metadata) |

### Tungsten and Shuffle Compression

- After `UnsafeRowSerializer` writes raw bytes → `SerializerManager` applies compression
- Compression applied to the raw `UnsafeRow` byte stream
- `UnsafeRow` fixed sections (integers, longs, doubles) - poor compression ratio (already binary; few patterns)
- Variable sections (strings, arrays) - better compression if data has repetition
- For integer-heavy DataFrames - compression may not be worth the CPU cost; consider `spark.shuffle.compress=false`

### Tungsten Off-Heap Serialization

- With `spark.memory.offHeap.enabled=true` -
    - `UnsafeRow` points to off-heap memory address instead of JVM heap array
    - Shuffle write - `Platform.copyMemory(null, baseOffset, outputBytes, ...)` copies from off-heap to output stream
    - Network transfer - Netty `DirectByteBuf` → OS `sendfile()` → zero JVM heap involvement
    - Completely bypasses JVM GC for the data path

> [!NOTE]
> The serialization hierarchy in Spark by performance (fastest to slowest) -
> 1. __Tungsten `UnsafeRow`__ - memcpy of binary region; zero object creation; zero GC
> 2. __Arrow columnar__ - zero-copy for numerics; batch transfer; one Python call per batch
> 3. __Kryo (registered)__ - $3-10×$ faster than Java; registered types stored as 2-byte ID
> 4. __Kryo (unregistered)__ - class name in stream; still faster than Java
> 5. __Java serialization__ - reflective; verbose; slowest; avoid for shuffle and broadcast

> [!TIP]
> Serialization configuration checklist for production -
> ```python
> spark = SparkSession.builder \
>     .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
>     .config("spark.kryo.registrator", "com.example.MyKryoRegistrator") \
>     .config("spark.kryo.registrationRequired", "false") \   # set True after all classes registered
>     .config("spark.kryoserializer.buffer", "128k") \
>     .config("spark.kryoserializer.buffer.max", "512m") \
>     .config("spark.sql.execution.arrow.pyspark.enabled", "true") \
>     .getOrCreate()
> # DataFrame operations already use Tungsten - no config needed
> # RDD operations benefit from Kryo
> # Pandas UDFs already use Arrow - no config needed
> ```