# Write-Ahead Log (WAL) / Commit Log

## Definition

- Provides a durability guarantee without requiring primary storage data structures to be flushed to disk on every update.
- Achieves this by -
  - Persisting every state change as a command.
  - Writing that command to an append-only log.
  - Ensuring the log is safely stored before modifying in-memory state.

## Problem

- Distributed systems and storage engines must provide strong durability guarantees, even in the presence of Process crashes, Machine failures, Power loss, Kernel panic or Out-of-memory termination.

## Requirement

- Once a server agrees to perform an action (e.g., returns success to a client) - that action must survive crashes and restarts — even if all in-memory state is lost.

- Without WAL -
  - In-memory changes disappear
  - Partial updates corrupt state
  - Acknowledged writes may be lost
  - System consistency is violated

## Core Idea

- Persist the intent to change state before actually changing the state.
- Write Order -
  1. Append command to WAL (append-only)
  2. Flush WAL to physical media
  3. Apply change to in-memory structures

- This ensures -
  - If crash occurs after (2) → system can recover.
  - If crash occurs before (2) → operation is treated as never committed.

## Log Structure

- A typical WAL entry -
```
class WALEntry {

  private final Long entryIndex;   // Monotonic ID
  private final byte[] data;       // Serialized command
  private final EntryType entryType;
  private final long timeStamp;
}
```

- Key Properties -
  - Monotonically increasing identifier
  - Append-only
  - Sequential writes
  - Single log per server process

## Why Sequential Append?

- Sequential writes 
  - Are faster than random writes
  - Align with disk & SSD characteristics
  - Simplify crash recovery
  - Avoid complex page-level flush logic

- Sequential logs are easier to -
  - Replay
  - Segment
  - Compact
  - Replicate

## Recovery Model

- On startup -
  1. Open WAL
  2. Read all entries
  3. Replay entries in order
  4. Reconstruct in-memory state

## Example: In-Memory KV Store with WAL

### Write Path

```
class KVStore {

  private Map<String, String> kv = new HashMap<>();

  public void put(String key, String value) {
      appendLog(key, value);
      kv.put(key, value);
  }

  private Long appendLog(String key, String value) {
      return wal.writeEntry(new SetValueCommand(key, value).serialize());
  }
}
```

- The `put()` operation -
  - Serializes a command
  - Writes it to the WAL
  - Applies it to the in-memory HashMap

### Command Serialization

```
class SetValueCommand {

  final String key;
  final String value;

  public void serialize(DataOutputStream os) throws IOException {
      os.writeInt(Command.SetValueType);
      os.writeUTF(key);
      os.writeUTF(value);
  }

  public static SetValueCommand deserialize(InputStream is) {
      var in = new DataInputStream(is);
      return new SetValueCommand(in.readUTF(), in.readUTF());
  }
}
```

### Recovery Logic

```
public void applyLog() {
    List<WALEntry> walEntries = wal.readAll();
    applyEntries(walEntries);
}
```

```
private void applyEntries(List<WALEntry> walEntries) {
    for (WALEntry walEntry : walEntries) {
        Command command = deserialize(walEntry);
        if (command instanceof SetValueCommand cmd) {
            kv.put(cmd.key, cmd.value);
        }
    }
}
```

## Implementation Considerations

- __Forcing Data to Physical Media__ -
  - It is critical that log entries are actually persisted, not just buffered in OS cache.
  - Most languages provide -
    - `fsync()`
    - `FileChannel.force(true)`
    - `fdatasync()`

## Tradeoff: Durability vs Performance

| Strategy          | Durability | Performance     |
| ----------------- | ---------- | --------------- |
| Flush every write | Strong     | Slow            |
| Async flush       | Weaker     | Faster          |
| Batched flush     | Balanced   | Common practice |

## Production Approach

- Use -
  - Batching
  - Group commit
  - Configurable flush interval

## Corruption Detection

- Logs may be partially written due to crashes.
- Solutions -
  - CRC checksum per entry
  - Length prefix + integrity check
  - End-of-entry markers
- If corrupted -
  - Discard incomplete entry
  - Replay up to last valid record

## Log Growth Problem

- Single log file grows unbounded.
- Techniques -
  - Segmented Log
  - Low-Water Mark
  - Log compaction
  - Snapshotting

## Low-Water Mark

- Entries below this mark -
  - Have been applied
  - Acknowledged by replicas
  - Safe to delete

## Duplicate Entries

- Because WAL is append-only -
  - Client retries
  - Network timeouts
  - Idempotent retries
- May produce duplicate entries.

## Handling Duplicates

- If state is idempotent (e.g., `Map.put()`) -
  - No issue
- Otherwise -
  - Attach unique request IDs
  - Maintain deduplication table

## Stable Storage Assumption

- WAL assumes disk is reliable.
- If disk fails - WAL is lost.
- For disk failure tolerance - use Replicated Log pattern, eg - Raft, Zab, Paxos-based systems.

## WAL in Transactional Storage

- Transactions provide ACID.
- WAL directly guarantees atomic durable updates.
- Isolation & consistency require -
  - Locking
  - Concurrency control
  - Two-phase commit mechanisms

## Example: RocksDB

- RocksDB uses WAL to ensure atomic batch writes.
```
WriteBatch batch = new WriteBatch();
batch.put("title", "Microservices");
batch.put("author", "Martin");

kv.put(batch);
```

- Internally -
```
appendLog(batch);
kv.putAll(batch.kv);
```

- If crash occurs -
  - Either whole batch is replayed
  - Or none of it is
- Never partial.

## Compared to Event Sourcing

| WAL                     | Event Sourcing              |
| ----------------------- | --------------------------- |
| For recovery            | Source of truth             |
| Can discard old entries | Keep indefinitely           |
| Focused on durability   | Focused on business history |
| Internal mechanism      | Architectural pattern       |

- Event sourcing uses its log for -
  - Time travel
  - Rebuilding historical state
  - Synchronizing systems
- WAL is usually -
  - Internal
  - Garbage collected after safe checkpoint

## Real-World Systems Using WAL

- Raft (Ongaro 2014)
- Zab (ZooKeeper protocol)
- Apache Kafka
- Cassandra
- PostgreSQL
- RocksDB
- All major relational databases

## Advanced Topics

- __Group Commit__ -
  - Multiple client writes -
    - Aggregated
    - Flushed together
    - Improve throughput drastically
- __Sync vs Async Replication__ -
  - WAL + replication = Replicated Log
  - Commit only after quorum acknowledgment
  - Foundation of consensus algorithms
- __Interaction with Modern Storage__ -
  - WAL exists because -
    - RAM is volatile
    - Disk is persistent
  - With persistent memory (e.g., Intel Optane) -
    - Write-behind logging may replace WAL
    - Direct persistent memory writes reduce need for redo logging
- __Performance Bottlenecks__ -
  - WAL can become -
    - Write bottleneck
    - IO-bound
    - Contention-heavy
  - Mitigation -
    - Single writer thread (Singular Update Queue)
    - Lock-free ring buffers
    - Async IO
    - Segmented logs