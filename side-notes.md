// clock sweep vs lru
// object id vs relfilenode ; why 2; what do they signify; usecases
// system catalogs vs system views vs modules/extensions  vs server applications
// catalogs/views for monitoring, stats
// relfilenode vs reltablespace vs reldatabase vs relforknumber vs relblocknumber
// ring vs normal buffer (diff, where and why in postgres)

// LSN format : WAL write location as a 64-bit LSN (Log Sequence Number), formatted as A/BBBBBBBB (e.g., C/CE7BAD70).  The first part is the upper 32 bits (segment number), the second is the lower 32 bits (offset within the segment).

// building index => B-Tree

### SQL-LEVEL JOINs & JOIN Execution Strategies 

<u>SQL-Level Joins</u>:

- Inner Join : only matching rows from both sides
- Left Join : All rows from left + matching right (NULLs fill missing right)
- Right Join : All rows from right + matching left (NULLs fill missing left)
- Full Join : All rows from both sides, NULLs fill gaps on either side
- Cross Join : every row x every row

<u>Physical Execution Strategies</u>:

- Nested Loop : For each outer row, scan/index the inner. Best when outer is small or inner has an index. The only option for inequality predicates.

- Hash Join : Builds a hash table for smaller side, probes with larger. Best when both sides are large, equality join and unsorted. 

- Merge Join : Sort both sides on join key, walk in parallel. Best when both sides are large and are already sorted (via index or ORDER BY)

The planner can swap which table is inner vs outer for efficiency. 


### Reading WAL LSN and WAL Filename

A single hex digit consists of 4 bits

1 byte = 2 hex digits = 2^8 = 8 bits

0 - 9, A = 10, ..., F = 15

1 byte = FF = 256

Each WAL segment is 16 MB.
```sql
SHOW wal_segment_size;
```

Each WAL segment consists of WAL records (lowest level description of change applied to the database)
The size of each WAL record is variable. It has to be atleast 24 bytes (header). The totatl size of the record is `xl_tot_len`

```c
typedef struct XLogRecord
{
    uint32      xl_tot_len;     /* total len of entire record (header + body) */
    TransactionId xl_xid;
    XLogRecPtr  xl_prev;        /* LSN of previous record */
    .....
} XLogRecord;   
```

Each segment is 16 MB size.

The `Log Sequence Number (LSN)` is a 64-bit number that acts as a byte address into one continuous, ever-growing logical stream of WAL data.

The 64-bit Layout: 

```
XXXXXXXX / YY ZZZZZZ
  │         │   │
  │         │   |- 6 hex chars = 3 bytes = 24 bits => byte offset WITHIN the
  |         |                                          16 MB segment
  │         |----- 2 hex chars = 1 byte  =  8 bits => which segment (0–255) 
  |                                                    in the group
  |--------------- 8 hex chars = 4 bytes = 32 bits => group number   
```

Segment groups are a numbering convention (not a physical unit). It's not a physical directory, partition, or allocation. It's just how the 64-bit LSN is sliced. Out of 16 MB (16,777,216 bytes) of WAL, ZZZZZZ indicates at what byte the WAL record starts from.
Max of YY is FF which is equal to 256 consecutive segment files. 

Think of it as a chapter (group) in a book. Each chapter (group) contains 256 pages (segments). Each page has 16 MB (WAL) of content. So each chapter contains 256 x 16 MB worth of content = 4 GB of content (WAL). The chapter number is just a counter.

The group number (XXXXXXXX) counter is incremented by everytime 256 segments or 4GB WAL content is crossed. 

Total number of groups = FFFFFFFF = = 2³² = 4,294,967,296 (about 4.3 billion).
Total addressable bytes: 4.3 billion × 4 GB ≈ 16 exabytes.
This is purely a numbering ceiling - the maximum value the 32-bit group counter can hold before wrapping.

```
TTTTTTTTXXXXXXXXYYYYYYYY
│       │        │       │
│       │        │       |--- "000000" (always 6 zeros - structural padding)
│       │        |----------- YY (2 hex chars: segment index 0–255)
│       |-------------------- XXXXXXX (8 hex chars: group number, zero-padded)
|---------------------------- TTTTTTTT (8 hex chars: timeline ID, usually 1)  
```

The physical pg_wal/ directory holds a small sliding window of 16 MB segment files, sized by max_wal_size + min_wal_size + replication lag. Old segments are recycled (renamed to future positions) or removed (deleted) at each checkpoint to keep disk usage bounded. 

```sql
-- LSN: segment filename
SELECT pg_walfile_name('5D/257E19B0');
-- 000000010000005D00000025
-- timeline | segment group | segment file
-- 1        | 5D = 93       | 25 = 37

-- LSN: filename + byte offset within that segment
SELECT * FROM pg_walfile_name_offset('5D/257E19B0');
--  file_name                | file_offset
--  000000010000005D00000025 | 8264112

-- filename: segment number + timeline
SELECT * FROM pg_split_walfile_name('000000010000005D00000025');
--  segment number | timeline_id
--  23845          | 1
```

> 8 byte alignment

Every WAL record starts at an LSN that is a multiple of 8, so if a record's actual length isn't a multiple of 8, Postgres leaves a few unused padding bytes after it before the next record begins.

The next record's start = current record's start + `xl_tot_len`, rounded up to the next multiple of 8. The rounding on 64-bit platforms MAXALIGN is 8.

`pg_waldump` prints `xl_tot_len` (example: 30, 50, 114), which excludes padding.
LSN differences (0x20, 0x38, 0x78) include padding, because the next record's LSN is the aligned position.
That is why the LSNs in your output all end in hex digits that are multiples of 8 (...900, ...958, ...A08), i.e. the last hex digit is always 0 or 8.