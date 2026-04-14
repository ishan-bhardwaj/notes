# Segmented Log

- A __segmented log__ splits a large append-only log into smaller files (segments) that are rolled by size/time.
- Used in WALs, Kafka partitions, Raft logs, LSM engines.

## Problem with a Single Huge Log

- A single append-only log works early on but fails at scale.
- __Startup / Recovery Bottleneck__ -
  - At startup systems must -
    - Scan WAL
    - Rebuild indexes
    - Replay unflushed state
  - If WAL = 500GB → startup = minutes/hours.

- __Deletion & Retention is Hard__ -
  - You cannot easily -
    - Delete first 200GB of file
    - Shrink a file from the front
    - Compact partial ranges
  - Filesystems are optimized for append, not delete from front.

- __Compaction is Impossible__ -
  - Compaction requires -
    - Rewrite selected records
    - Drop obsolete entries
  - Rewriting a single giant file = catastrophic.

- __Replication Inefficiency__ -
  - Followers need incremental replication - “Send me logs after offset X”
  - With a single file -
    - Finding offsets becomes expensive.
    - Sending chunks is messy.

## Segment Lifecycle

| State             | Description           |
| ----------------- | --------------------- |
| Active segment    | Current append target |
| Closed segment    | Immutable, read-only  |
| Compacted segment | Cleaned & rewritten   |
| Deleted segment   | Removed by retention  |

## Rolling Trigger Strategies

| Trigger               | Why                        |
| --------------------- | -------------------------- |
| Max size (e.g. 1GB)   | Bound recovery time        |
| Max age (e.g. 1 hour) | Enable timely retention    |
| Low disk space        | Backpressure               |
| Compaction scheduling | Prepare immutable segments |

> [!NOTE]
> Kafka uses both time + size.

## Rolling Code Concept

```
public Long writeEntry(WALEntry entry) {
    maybeRoll();
    return openSegment.writeEntry(entry);
}
```

- Rolling happens before append, not after.
- Why?
  - Prevent segment overflow
  - Maintain invariant - `segment size ≤ max size`.

## Naming & Addressing Segments

- We must map - `Global Log Offset` → `Segment File` → `Offset inside segment`
- __Two Addressing Strategies__ -
  - __Base Offset in File Name__ -

    ```
    wal_00000000000000000000.log
    wal_00000000000001000000.log
    wal_00000000000002000000.log
    ```

    - File name = first offset inside segment.
    - Kafka, Raft, Pulsar all use this.
    - Benefits -
      - O(1) lookup via binary search
      - Human readable
      - Easy replication

  - __Split Logical Offset__ -
    - Instead of using big numbers as offsets, eg - `LSN = 148,392,991,221 bytes` 
    - We split Logical LSN (Logical Sequence Number) into - 
      ```
      [ segmentId | intra-segment offset ]

      // Example
      Segment 14, offset 812 bytes
      ```

      - Used by databases with page-based WAL.
      - More complex but enables -
        - Page alignment -
          - Ensuring WAL records are written so they never straddle disk pages (`4–16KB` blocks).
          - Disks, filesystems and OS page cache all operate in page-sized I/O, not bytes.
          - If a WAL record crosses a page boundary -
            ```
            | Page N | Page N+1 |
                 ^ WAL record spans here
            ```
          - You force a read-modify-write of two pages for a single append -
            - `read page N` > `modify tail` > `write page N` > `read page N+1` > `modify head` > `write page N+1`
          - That kills sequential write throughput and doubles write amplification.
          - Aligned WAL instead becomes -
            ```
            | Page N | Page N+1 |
            |records |records   |
            ```
          - Why this is critical -
            - __Sequential I/O path__ -
              - WAL becomes pure append-only sequential writes
              - Maximizes disk throughput and OS readahead/writeback efficiency
            - __Atomicity guarantees__ -
              - Storage often guarantees atomic page writes, not byte writes.
              - Alignment ensures crash never leaves “half a record”.
            - __Fast crash recovery__ -
              - Recovery scans WAL page-by-page without stitching split records.
              - Huge startup time win on large logs.
        - Efficient preallocation -
          - Pre-creating fixed-size WAL segments (e.g., 1GB) before they are filled.
          - Instead of - 
            - `append` → `extend file` → `allocate blocks` → `update metadata` → `fsync metadata`
          - We do - `create 1GB WAL segment once` → `append forever`
          - Growing a file repeatedly is expensive because the filesystem must -
            - allocate new disk blocks
            - update inode + metadata
            - potentially fragment the file
            - sync metadata to disk (slow!)
          - That introduces latency spikes in the commit path — exactly where DBs cannot tolerate jitter.

## Reading From Segmented Log

- __Algorithm__ -
  - Identify segment containing start offset
  - Read that segment
  - Read all later segments sequentially

> [!TIP]
> Production systems never linear scan segments list.
>
> They maintain - `TreeMap<BaseOffset, Segment>`
> 
> Lookup becomes - `segment = floorEntry(startOffset)`
>
> Time complexity - `O(log n)`

## Indexes per Segment

- Real systems NEVER scan segment linearly.
- Each segment has indexes -

| Index             | Purpose                |
| ----------------- | ---------------------- |
| Offset index      | offset → file position |
| Time index        | timestamp → offset     |
| Transaction index | txn boundaries         |

> [!TIP]
> Kafka has `.index` and `.timeindex` files.

## Retention & Deletion

- Segmentation enables simple retention.
- Delete whole files, instead of rewriting 500GB file -
  - Filesystems do not support deleting bytes from the beginning.
  - So the database would have to -
    - Create a new file
    - Copy the last 100 GB into it
    - fsync it
    - Replace the old file
    - Delete the 500 GB file
  - With segmentation - `delete wal_001 → wal_400`
  - Just unlink files — which is a metadata operation and extremely fast.
- __Retention policies__ -
  - Time based (7 days)
  - Size based (keep 1TB)
  - Compaction based (keep latest per key)

## Compaction & Cleaning

- Segmentation enables log compaction.
- Append-only logs accumulate redundant history (multiple updates per key). 
- Example - `SET A=1` → `SET B=2` → `SET A=3` → only latest value per key is needed for rebuilding state.
- Log compaction rewrites old log data to keep the latest value per key while preserving append-only semantics.
- How compaction works operationally -
  - Active segment fills → becomes read-only.
  - Background compactor scans closed segments.
  - Build latest-value-per-key index.
  - Rewrite compacted segment.
  - Atomically replace old segment with compacted one.
  
```
Before:  SET A=1 | SET B=2 | SET A=3
After:   SET A=3 | SET B=2
```

- This changes compaction from a global rewrite problem → incremental background maintenance.

- Kafka log compaction relies on this.

## Recovery Improvements

- Without Segmentation - Replay entire WAL.
- With Segmentation -
  - Replay only -
    - Last segment
    - Maybe last few segments
  - Recovery time becomes bounded.
- Typical segment sizes -
  - Kafka - `1GB`
  - Databases - `16–256MB`
  - Raft - `64–128MB`

## Replication Benefits (Raft/Kafka Insight)

- Followers replicate segment ranges.
- Leader sends - `Segment 120 → bytes 4MB–8MB`
- This enables: -
  - Efficient catch-up
  - Resume after network failure
  - Parallel replication.

## Crash Safety Subtleties

- Segment rolling must be crash safe.
- Correct order -
  - Flush segment
  - Fsync segment
  - Update metadata
  - Create new segment
- If order wrong → data loss or corruption.
- Kafka uses -
  - checkpoint files
  - atomic renames
  - fsync barriers

## Segment Size Tradeoffs

| Small Segments         | Large Segments  |
| ---------------------- | --------------- |
| Fast recovery          | Slow recovery   |
| More file handles      | Fewer files     |
| Faster deletion        | Slower deletion |
| More metadata overhead | Less overhead   |

## File System Considerations

- Segmented logs rely heavily on FS behavior.
- Important tuning -
  - Preallocate segment files (avoid fragmentation)
  - Use sequential IO
  - Avoid random writes
  - Use O_DIRECT or buffered IO depending on workload

## Monitoring Metrics (Production)

- Monitor -
  - Segment count per partition
  - Segment roll frequency
  - Recovery time
  - Segment compaction lag
  - Oldest segment age
  - Open file descriptors

## Real World Systems Using Segmented Logs

- Messaging / Streaming -
  - Apache Kafka
  - Pulsar
  - Redpanda
- Consensus -
  - Raft
  - etcd
  - Consul
- Databases -
  - Cassandra commit log
  - RocksDB WAL
  - PostgreSQL WAL (segments!)