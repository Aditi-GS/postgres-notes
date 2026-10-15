# PostgreSQL Memory Components

### Shared Memory Area
1. [Shared Buffers](#1-shared-buffers)
2. [WAL Buffers](#2-wal-buffers)
3. [CLOG Buffers (Commit Log Buffers)](#3-clog-buffers-commit-log-buffers)
4. [PROC Array](#4-proc-array)
5. [Lock Space](#5-lock-space)
6. [Buffer Descriptors](#6-buffer-descriptors)

### Per-Backend Memory Components
1. [Work Memory](#1-work-memory)
2. [Maintenance Work Memory](#2-maintenance-work-memory)
3. [Temp Buffers](#3-temp-buffers)
4. [Vacuum Buffers](#4-vacuum-buffers)
5. [Relcache](#5-relcache)
6. [Syscache](#6-syscache)

## Shared Memory Area

Shared Memory Area is the entire memory segment (RAM) that the postmaster allocates at startup and which maps into every backend process's address space. It belongs to the entire instance (database cluster) and not a single database. Shared Memory is protected by light weight locks (LWLocks) for concurrent access. It is composed of multiple sub-areas like:

---

### 1. Shared Buffers

Is the largest area used to cache data/index pages. It's purpose is to minimize disk I/O. The unit of operation is blocks/pages and modified blocks are called "dirty pages". The "dirty data" is later written/persisted onto the disk containing the physical "data files". A specific dirty bit within the buffer descriptor is used to track the status of a dirty page internally.

Capacity of these buffers is controlled by the parameter `shared_buffers`.

```
Total memory = buffers slots x fundamental unit of storage (8KB block/page)
Default: 128MB => shared_buffers
# of buffer slots = 128 MB / 8 KB => 16 x 2^10 
```

Suggested starting value: 25% of the system's RAM

Do not exceed: ~40% of RAM - starves the OS cache that the rest of the system needs

---

### 2. WAL Buffers

Ring buffer that stores WAL data that has not yet been written to the disk. The memory allocated to them is controlled by `wal_buffers` parameter. The default setting of `-1` means about 3% of the `shared_buffers` capacity is allocated for staging WAL data - but not less than 64KB and not more than the size of WAL segment (16MB).

For each transaction commit, the contents of the WAL buffers are flushed to the disk (`/pg_wal` directory).

- why a ring buffer ?
WAL records are written are strictly sequential - the producers of WAL records (backends) append their records at one end and the consumers (WAL writer) always drains from the other. Once a record is flushed to a segment on a disk - the space in the ring is immediately reused. 

- The WAL buffer is small by design - capped at 16MB (one WAL segment). If a transaction generates more WAL than fits in the buffer before it commits, PostgreSQL doesn’t wait or error. It flushes the full buffer to disk mid-transaction to make room and keeps going. This is why very large bulk operations (massive inserts, bulk updates) generate more disk writes - they’re flushing WAL incrementally, not just once at the end.

---

### 3. CLOG Buffers (Commit Log Buffers)

Keep a track of transaction status for every transaction id (xact id or xid) as to whether it is in-progress, committed or aborted for all active transactions. Located in the `pg_xact` subdirectory of the data directory, these buffers allow the database engine to quickly check transaction states without repeatedly accessing disk I/O. There is no specific config parameter to set the size of this buffer. Based on `shared_buffers` setting and system concurrency - memory is managed automatically.

---

### 4. PROC Array

Is a critical internal shared data structure which stores the process ID (pid), transaction ID (xid), virtual transaction id (vxid), etc of all active transactions of all backend processes. Every single query that builds a snapshot must grab the shared lock on ProcArray and scan the whole list. This becomes a bottleneck when hundreds of connections are established. When a transaction ends, it's xid is removed from the list.

- Not every transaction gets a real xid. A transaction only gets one assigned when it actually modifies data. 
- Read-only transactions never get a real xid at all. But they still need an identity in the PROC array so other transactions can see them. That’s what the virtual transaction id (vxid) is for. It’s a lightweight local identifier composed of the backend’s ID and a sequence number, allocated immediately when a transaction starts, and never written to any on-disk structure. 
- Real xids are a finite resource (32-bit counter, ~4 billion values before wraparound).

> Snapshot

A snapshot is a consistent view of the database at a point in time. 
- When a query starts, instead of locking all rows it might read - it captures the `xmin`, `xmax` and `xip` list from the PROC array and for every tuple it encounters it decides whether the tuple should be visible to the query or not. 

- visible => if the tuple was created by a transaction commited before it's snapshot
- invisible => if the tuple was created by a transaction still in progress 
- invisible => if the tuple was deleted by a committed transaction before it's snapshot

- When a backend builds a snapshot, it grabs a shared lock on the PROC array and scans every entry. From that scan it derives:

`xmin` - lowest xid across all active transactions
`xmax` - next xid not yet assigned
`xip` - list of all in-progress transactions at the time of snapshot

In tuple, the fields `xmin and xmax` have different meanings: 

`xmin` - xid of the transaction that inserted the tuple
`xmax` - xid of the transaction that deleted the tuple

For update it's requires both xmin and xmax - as the old tuple version will be marked dead and the new tuple version will have the updating xact's xid for xmin.

- For inserions: 
```
tuple.xmin < snapshot.xmin  -> all committed    -> visible
in xip  -> in-progress  -> invisible  
between xmin and xmax, not xip  -> already committed    -> visible
tuple.xmin >= snapshot.xmax -> didn't exist yet -> invisible
```

- For deletions: 
```
tuple.xmax = 0  -> not deleted  -> visible
tuple.xmax < snapshot.xmin -> deletion before snapshot -> invisible (dead row)
tuple.xmax >= snapshot.xmax -> deletion after snapshot -> visible (deletion invisible)
tuple.xmax in xip -> deletion by in-progress xact -> visible (deletion invisible)
```
This is why the PROC array scan is a bottleneck at high connection counts - every new query locks and walks the entire array to build these three values.

---

### 5. Lock Space

Is a hash table where all types of heavy weight locks are stored - whose size is decided by parameters `max_locks_per_transaction` (by default 64) and `max_connections` (by default 100). Server will not able to grant any locks at all if this space is exhausted.

Heavy weight (regular) locks manage concurrent access to high-level resources such as tables, indexes and rows. 
Light weight locks are internal interprocess sychronization mechanisms that protect shared memory structures within the database engine - like buffer descriptors, WAL buffers, and the lock manager hash table itself. They have only two modes (shared - for read-only access and exclusive - for modifications).

Regular lock types include relation and row level locks (regular) and advisory locks. Row level locks do not occupy any slots in the lock space - they are unlimited. Row level locks are stored in the tuple headers on the disk. Locks are held until transaction ends. There is no UNLOCK command.

**Lock-related timeout parameters:**

| Parameter | Behavior |
|---|---|
| `lock_timeout` | Wait until the acquirer releases the lock or abort the waiter after timeout |
| `statement_timeout` | If a single statement in a transaction runs for too long, it is killed, transaction aborts and lock is released |
| `idle_in_transaction_session_timeout` | A session is idle inside a open transaction for too long, it is terminated - rolling back the transaction |
| `transaction_timeout` | Times out based on the total age of a transaction (active or idle) - kills it |

Relation-level (object-level) locks are acquired when a transaction needs access to referenced tables/indexes/sequences/views/etc.

Advisory locks are generic user/application defined locks that lock application defined resource (not a relation/row). PostgreSQL guarantees that only one session at a time holds a given advisory lock key.

- `pg_advisory_xact_lock(key)` - released automatically when a transaction commits or rolls back
- `pg_advisory_lock(key)` - at session scope, released when `pg_advisory_unlock(key)` is explicitly called

The key is just a number - it's meaning is completely upto the application to decide.

---

### 6. Buffer Descriptors

Are present for each shared buffer slot and stores metadata of the cached data pages. It tracks:

| Field | Description |
|---|---|
| `tag` | Which page is in the slot (relation OID, block number) |
| `buf_id` | It's own buffer slot index |
| `state` (32-bit) | Packed into one word it indicate how many backends are asking for it (refcount), how "hot" is the data page (usage_count) and whether the page is `BM_DIRTY`, `BM_VALID`, `BM_IO_IN_PROGRESS`, `BM_CHECKPOINT_NEEDED`, etc (flag) |

Buffer Hash Table is another structure present along with shared buffers and buffer descriptor. It tracks which object/relation is in which slot index in the shared buffers - it is keyed on (relation OID, fork number, block number).

Each slot in shared buffer stores a pair - a descriptor and the actual 8KB page. They are stored as 2 parallel arrays.

---

### Notes: 
**Light Weight Locks**

LWLocks are transient - lasting only few micro-nanoseconds while internal processes manipulate data in the memory. They aren't visible in `pg_locks` but can be monitored via `pg_stat_activity -> wait_event_type='lwlock'`. They are fine-grained - the scope of each individual LWLock is narrow - protecting only a small unit of shared structure like - per page, per buffer slot, etc. This allows multiple backend processes operate in parallel on different parts of the shared structure at the same time. Their fine grained design helps acheive maximum concurrency but when a single resource - like a page - is hammered by multiple backends in short amount of time - the page remains "hot" causing reduction in performance or bottlenecks. Backends wait until the LWLock released for few microseconds and if not released, it goes to sleep until the OS informs the release of the lock.

## Per-Backend Memory Components

Unlike shared memory area which is shared by all backend and auxiliary processes, there are memory components assigned per backend process:

### 1. Work Memory

Is the amount of memory allocated for each sort or hash operation within a query. It is `not per-query or per-backend` - it is `per operation node in the execution plan`. A single query with multiple sort nodes or hash joins can multiply this allocation several times over. 

`work_mem` indicates how much memory is allocated for each sort or hash operation before it spills to disk - writing temporary files in `pgsql_tmp/` under the data directory. Spills are expensive: what was a pure memory sort becomes disk I/O. `hash_mem_multiplier` is a multiplier applied to `work_mem` specifically for hash-based operations while keeping allocation for sort the same (`work_mem`). Hash spills onto disk is expensive and sort spill degrades gradually. So when spills increases for hash operations, increase the `hash_mem_multiplier` instead of `work_mem` as it avoids the unecessary bloat due to increased allocation for sort as well and reaises the ceiling so that the batches fit.  

If build side exceeds `work_mem x hash_mem_multiplier`, then both sides (outer/smaller and inner/larger) are partitioned into batches using a second hash function. Each partition pair is loaded into memory, hashed, and joined independently. This repeats for every batch. 

Used by : 
- ORDER BY, DISTINCT sorts
- Hash Joins
- Hash Aggregations
- Window functions with sorting

The worst case memory usage is: 

```max_connections × max_sort_nodes_per_query × work_mem```

---

### 2. Maintenance Work Memory

Is the amount of memory allocated for maintenance operations. Unlike work_mem which can multiply across many nodes in a query plan, only one maintenance operation typically runs per backend at a time - so it's possible to safely set this much higher than `work_mem` without the multiplication risk. It is controlled by the parameter `maintenance_work_mem` and is 64 MB by default. 

Used by:
- VACUUM - for storing dead tuple IDs before they are cleaned up. The more dead tuples it can hold in memory, the fewer heap passes it needs to make.
- CREATE INDEX / REINDEX - for sorting index entries during index build. More memory = fewer merge passes = faster index build.
- CLUSTER - for sorting tuples when rewriting the table in index order.
- ALTER TABLE ADD FOREIGN KEY - for the constraint validation scan.

Since autovacuum workers also use this parameter, `autovacuum_work_mem` separates the two concerns - cap what autovacuum workers use without limiting what a manual VACUUM or index build gets.

---

### 3. Temp Buffers

Is a per-backend buffer pool exclusively for temporary tables. Temporary tables are session-scoped tables that exist only for the duration of a session and are not shared with other backends.

Common uses of temporary tables:
- Staging intermediate results within a complex multi-step operation
- Breaking up a large query into steps - compute into a temp table, then query that
- Avoiding repeated expensive subquery computation

They behave exactly like a regular table within the session. 

Unlike shared buffers which cache permanent relation pages for all backends, temp buffers are private to each backend and only allocated on demand - a backend that never touches a temporary table never allocates them.

The pages cached here follow the same 8KB block size as shared buffers, but are managed entirely within the backend’s own memory without any LWLock contention - since no other backend can ever access the same temporary table, there is no need for shared synchronization.

Default is 8MB. Must be set before the first temporary table is created in a session.

By default temporary tables are unlogged, so they produce no WAL records and their data does not survive a crash or session end. The pages in temp buffers are simply discarded when the session ends.

---

### 4. Vacuum Buffers

Is a private buffer pool allocated per VACUUM operation for scanning heap and index pages during cleanup. It exists separately from shared buffers specifically to prevent VACUUM from polluting the shared buffer pool - a full table scan by VACUUM would otherwise evict hot working data pages that backends are actively using. Vacuum buffers are per VACUUM invocation, which happens to be tied to whichever backend (or autovacuum worker) is running it at that moment. A backend can run multiple VACUUM operations sequentially - each one gets its own fresh allocation of vacuum buffers. When one VACUUM finishes, the buffer is released, and the next VACUUM starts with a clean pool.

When VACUUM reads a page, it first checks this private pool. If not found there, it reads from disk directly into this pool rather than going through shared buffers. This keeps the churn of a large VACUUM scan contained and isolated.

Default is 256KB - VACUUM’s access pattern is sequential and non-repeating - it scans each page once, cleans it, and moves on.

Can be set per-session or per-VACUUM call:
```sql
VACUUM (BUFFER_USAGE_LIMIT '4MB') <table_name>;
```

`vacuum_buffer_usage_limit` - controls the page buffer pool for scanning. How many 8KB pages VACUUM can hold in its private cache at once. Vacuum buffers are for the page scanning side, maintenance/autovacuum_work_mem for the dead tuple collection side.

---

### Catalog Caches

Both Relcache and Syscache cache system catalogs metadata to avoid repeated disk reads from catalog tables - which are consulted in almost every query.

---

### 5. Relcache

Stores the relation descriptor of each relation (table, index, view, sequence, etc). It aggregates multiple system catalog tables to collect metadata related to the relation - it includes `pg_class`, `pg_attributes`, index info, constraints info, partition metadata, etc in a single aggregate relation descriptor struct. It is keyed by OID and is used every time a backend needs to access or inspect a relation - during query planning, execution, permission checks, etc. Upon a cache miss, it reads the relevant catalogs and constructs a new descriptor for that relation.

When DDL change modifies a relation, an invalidation message is sent to all backends - each backend marks it as invalid in their local entry. It is rebuilt in the next access.

---

### 6. Syscache

Stores individual catalog tuples. It is used whenever a backend requires a specific row by it's key. Upon a cache miss, it performs an index scan over the underlying catalog table and adds the tuple to the cache. If the tuple doesn't exist, it makes negative cache entry to mark it's absence to avoid repeated scans.

When a catalog row is inserted/modified/deleted, an invalidation message is sent to all backends which mark it as a dead row and removes it from their local entry. The tuple is re-fetched in the next lookup.