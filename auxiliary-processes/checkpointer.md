# 1. Checkpointer:

<details>
    <summary>An auxiliary process that is responsible for executing checkpoints.
    <br><br>
    <u>Checkpoint</u> : A point in the WAL sequence at which it is guaranteed that the heap and index data files have been updated with all information from shared memory modified before that checkpoint; a checkpoint record is written and flushed to WAL to mark that point.
    </br></br>
    </summary>
<br>

> **What it does:**

Flushes all dirty (modified) pages from shared_buffers to their data files on disk, then records the checkpoint's WAL position in `pg_control`. 

> **What it stores:**

The checkpoint record (WAL position, time, etc.) in `pg_control`. It doesn't "store" data itself - it ensures data already in shared_buffers is persisted. 

> **When triggered:**

1. *Time:* `checkpoint_timeout` (default 5 min)    
    ``` sql
    SHOW checkpoint_timeout;
    ```

2. *WAL volume:* `max_wal_size` (default 1 GB)
    ``` sql
    SHOW max_wal_size;
    ```

3. **Manual:** `CHECKPOINT`; command

---

### Internal workings:

- The checkpointer records the current WAL position as the REDO point (recovery start point). 
- `REDO LSN` -> the LSN of XLOG (WAL)- the next record to be inserted at the moment checkpointer started. 
- The checkpointer acquires all of the 8 `WALInsertLock LWLock` (***exclusive*** - blocks other processes from inserting wal records) briefly to capture the REDO LSN into a variable stored in shared memory and then the checkpointer also stores the value of the variable in it's own local memory (to avoid re-reads from the shmem) and inserts the `XLOG_CHECKPOINT_REDO` record (has empty data portion - only to mark the start of checkpointing).
- It then releases the locks for other transactions to continue while it starts flushing (write() to OS/Kernel Cache and fsync() to disk)
to the disk. [flushing to disk != deleting from buffers]
- The checkpointer performs a single complete linear scan of the `BufferDescriptors[]` array in shared memory and checks the `state` field for `BM_DIRTY` label,
indicating if the corresponding page the descriptor refers to is dirty (modified) or not.
- During it's linear scan, it builds a local array where it stores the `buf_id, relfilenode, forknum, blocknum` of every dirty buffer slot it comes across.
- This local array is later sorted by it's physical location - to turn random I/O into roughly sequential I/O.
- The checkpointer later walks the `sorted local array` again. It pins it (increment the `pin count`) to prevent `clock-sweep eviction` mid-write - a pinned buffer slot can't be chosen as a candidate by clock-sweep eviction.
- To recheck the `state` - so that it can skip buffers which are now clean because of bgwriter or other backends (due eviction) - it acquires the `buffer header spinlock` and checks if it's still `BM_DIRTY`. If it is `BM_IO_IN_PROGRESS` then it skips again because the bgwriter or any backends can be flushing the page instead. Otherwise, it set's the state to `BM_IO_IN_PROGRESS` (to claim the buffer for write()) and releases the spinlock.
- It later acquires the buffer's `content_lock LWLock` in `shared mode (read-only)` that even backends use to modify pages.
- The checkpointer gradually flushes the buffers to the disk - spread writes over `checkpoint_completion_target` `×` `checkpoint_timeout`. Regardless of `checkpoint_timeout`, all dirty pages (since checkpointing began) are flushed onto stable storage - even if it's past the `checkpoint_timeout`.
- The exact `relfilenode`, the `block number` and `fork number` is already known from the descriptor.
- Once flushed onto the disk it unpins and releases the lock. This guarantees that all dirty pages are fsynced and have persisted on disk.

---

- The checkpointer now inserts (`WALInsertLock`) the checkpoint record `XLOG_CHECKPOINT_NORMAL` to WAL. Though part of the WAL stream, we can differentiate checkpoint record from a WAL record because:

| | WAL Record | Checkpoint Record |
|---|---|---|
| resource manager (rmgr) | HEAP, BTREE, TRANSACTION, etc| XLOG |
| contains | Modification applied to a page | Redo LSN, transaction ID state, timestamp, next OID |
| purpose | Replay a specific page change | Mark a safe recovery starting point |
| page reference | specific relation, block number | - |

- The checkpoint record contains: 
    - REDO LSN
    - Xact ID state at checkpoint time
    - OID state
    - timestamp
    - WAL and replication state

- Then it updates (fsyncs) `pg_control` immediately (after the record is flushed to `pg_wal` - WALWriteLock) with:
    - WAL location of the checkpoint record written
    - the REDO LSN embedded in the record
    - the checkpoint's timestamp and transaction state

---

- After checkpointing, the old WAL files are no longer required for crash recovery and hence are eligible for deletion (unlinking from the disk) or recycling/renamed (if `wal_recycle=on`).
- Internally though, WAL files older than ***previous checkpoint*** are deleted/recycled. The number of these WAL segments 
recycled is controlled by `min_wal_size` parameter.

```sql
SHOW min_wal_size;
[Default=80MB]
```

- If any of the WAL segments contain fraction of WAL records that are older than redo point, the XLOG_CHECKPOINT_REDO itself and/or WAL records after the checkpoint in the same segment file - then it won't be removed/recycled - since WAL operates on whole segment files only.
- If `archive_mode=on` or if the `archive_command` is executed, then WAL files can't be removed or recycled until the WAL files are persisted/archived.
- This might create a block if archiving is slow and when none are archived and no freed up space is available for new incoming WAL segments.

---

#### Result: 

1. Crash recovery now only needs to replay WAL from the checkpoint forward. 
2. WAL segments before the REDO point can be recycled/deleted. 

---

### Relevant Server Configured Parameters: 
<u>For detailed definitions</u>:
[Server Configurations - Write Ahead Log](https://www.postgresql.org/docs/18/runtime-config-wal.html)

> ***checkpoint_timeout (integer)*** 

Maximum time between automatic WAL checkpoints. 
Increasing this parameter can increase the amount of time needed for crash recovery (accumulation of WAL)

> ***checkpoint_completion_target (floating point)*** 

Specifies the target of checkpoint completion, as a fraction of total time between checkpoints.
The default is 0.9, which spreads the checkpoint across almost all of the available interval, providing fairly consistent I/O load while also leaving some time for checkpoint completion overhead.
Reducing this parameter is not recommended because it causes the checkpoint to complete faster.
This results in a higher rate of I/O during the checkpoint followed by a period of less I/O between the 
checkpoint completion and the next scheduled checkpoint.

> ***max_wal_size (integer)*** 

Maximum size to let the WAL grow during automatic checkpoints.
This is a soft limit; WAL size can exceed max_wal_size under special circumstances, such as heavy load, 
a failing archive_command or archive_library, or a high wal_keep_size setting. 
Increasing this parameter can increase the amount of time needed for crash recovery.

> ***min_wal_size (integer)*** 

As long as WAL disk usage stays below this setting, old WAL files are always recycled for future use 
at a checkpoint, rather than removed. This can be used to ensure that enough WAL space is reserved to 
handle spikes in WAL usage, for example when running large batch jobs. 

> ***checkpoint_flush_after (integer)*** 

After every checkpoint_flush_after pages are written directly from shared buffers into the OS page cache, it issues an asynchronous `sync_file_range(SYNC_FILE_RANGE_WRITE)` hint to the kernel to start pushing those pages to the block device. This prevents a huge backlog from accumulating in the OS page cache so that the final fsync() at checkpoint end stays cheap. `sync_file_range(SYNC_FILE_RANGE_WRITE)` does not guarantee durability like fsync() does - it is a fire-and-forget hint that merely initiates write-out and returns immediately, without waiting for I/O completion or flushing the drive's volatile write cache.

</details>

---