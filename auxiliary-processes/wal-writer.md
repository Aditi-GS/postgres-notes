# 3. WAL Writer:
<details>
    <summary>An auxiliary process that writes WAL records from shared memory to WAL files.
    <br><br>
    <u>WAL records</u> : A low-level description of an individual data change. 
    <ul>
    <li>It contains sufficient information for the data change to be re-executed (replayed) in case a system failure causes the change to be lost.
    <li>WAL records use a non-printable binary format.
    </ul>
    <br><u>WAL files</u> : Also known as WAL segment or WAL segment file.
    <ul>
    <li>Each of the sequentially-numbered files that provide storage space for WAL. 
    <li>The files are all of the same predefined size and are written in sequential order, interspersing changes as they occur in multiple simultaneous sessions. 
    <li>If the system crashes, the files are read in order, and each of the changes is replayed to restore the system to the state it was in before the crash.
    <li>Each WAL file can be released after a checkpoint writes all the changes in it to the corresponding data files. 
    <li>Releasing the file can be done either by deleting it, or by changing its name so that it will be used in the future, which is called recycling.</ul></br>
    </br></br>
    </summary>
<br>

### Asynchronous and Synchronous Commits

> Asynchronous

Asynchronous commit is an option that allows transactions to complete more quickly, at the cost that the most recent transactions may be lost if the database should crash. In many scenarios - like event logging - this is an acceptable trade-off. The mode of commit is controlled by the user settable parameter `synchronous_commit` which can be toggled between `on and off`.

Transaction commit is normally synchronous: the server waits for the transaction's WAL records to be flushed to permanent storage `pg_wal` before returning a success to the client. The client is therefore guaranteed that a transaction reported to be committed will be preserved, even in the event of a server crash immediately after. 

However, for short transactions this delay is a major component of the total transaction time. Selecting asynchronous commit mode means that the server returns success as soon as the transaction is logically completed, before the WAL records it generated have actually made their way to disk. This can provide a significant boost in throughput for small transactions.

Asynchronous commit introduces the risk of data loss (not data corruption). There is a short time window between the report of transaction completion to the client and the time that the transaction is truly committed. If the database should creash - it will recover by replaying WAL up to the last record that was flushed. The database will therefore be restored to a self-consistent state, but any transactions that were not yet flushed to disk will not be reflected in that state. The net effect is therefore loss of the last few transactions. Because the transactions are replayed in commit order, no inconsistency can be introduced. 

The duration of the risk window is limited because a background process (the `WAL writer`) flushes unwritten WAL records to disk every `wal_writer_delay` milliseconds. The actual maximum duration of the risk window is `3 x wal_writer_delay` because the WAL writer is designed to favor writing whole pages at a time during busy periods.

The user can also select the commit mode of each transaction, so that it is possible to have both synchronous and asynchronous commit transactions running concurrently. But certain utility commands, like `DROP TABLE` are forced to commit synchronously regardless of the set mode to ensure consistency between server's file system and logical state of the database. 

> Why 3 x wal_writer_delay ?

When the `WAL writer` wakes, it doesn't flush all the records in the buffer blindly - it looks for the last complete page (8KB of WAL records/page) and flushes up until there - since it favors whole pages writes during heavy write load - because writing a full page is more I/O efficient that writing a partial one. If the current WAL page in the buffer is incomplete, the WAL writer skips it until next cycle until it is full. 

Let's say the current async commit record falls in the middle of the current WAL page. In the worst case scenario (heavy load), in the first cycle the WAL writer skips this page and only flushes all the full pages before the current incomplete WAL page. In the second cycle, more WAL is written and let's assume the page containing the async commit record which signaled the WAL writer earlier is now full. The WAL writer still might decide to batch multiple whole pages together rather than flushing a single one - it might wait to coalesce multiple whole WAL pages into a single write. The third cycle is when the the WAL page containing the record is flushed in this worst case scenario. Hence `3 x wal_writer_delay`.

> Synchronous

When `synchronous_commit=on` the backend calls `XlogFlush` itself upon commit to flush the WAL records in the WAL buffers to the `pg_wal` directory - only after persisting it on the disk does it send a success indication back to the client. `commit_delay` is a synchronous commit method that causes a delay just before the transation flushed WAL to the disk - in the hopes that a single flush invoked by such a transaction will batch other transactions commiting at the same time - within the interval introduced by it. 

### When triggered

With `synchronous_commit=off` the backends don't write() + fsync() the WAL themselves. WAL writer guarantess the data durability by periodically flushing the WAL records from the WAL buffers in the shared memory to the disk. It acquires the `WALWriteLock LWLock` and calls `XlogWrite` which triggers synchronization via `issue_xlog_fsync` [ apart from the WAL writer, this is also triggered by `XlogInsertRecord` when the buffers are full and by `XlogFlush` during commits ].

WAL writer perdiocally flushes WAL records to the disk every `wal_writer_delay` ms or immediately after `wal_writer_flush_after` bytes of WAL accumulates in the buffers. It write() [flush to the OS/Kernel cache] and then fsync()s` the records. 

Even with `synchronous_commit=on` the WAL writer is still triggered in situtations like: 

1. WAL records generated by non-backend processes: 
The checkpointer writes checkpoint records to the WAL and the startup process writes WAL during recovery/replay.

2. WAL from ongoing (uncommited) transactions and avoid buffer exhaustion:
Long running transactions might have generated a large amount of WAL and not have committed yet. Backends trying to write new blocks of WAL are blocked until space frees if the buffers get full. WAL writer drains the buffers to the disk periodically in the background - preventing the buffers from filling up and also reducing the I/O burst that might happen at commit time.

The methodn of fsync depends on the parameter `wal_sync_method`.

### Relevant Server Configured Parameters: 
<u>For detailed definitions</u>:
[Server Configurations - Write Ahead Log](https://www.postgresql.org/docs/18/runtime-config-wal.html)

> ***wal_level (enum)***

wal_level determines how much information is written to the WAL.
- The default value is `replica`, which writes enough data to support WAL archiving and replication, including running read-only queries on a standby server.
- `minimal` removes all logging except the information required to recover from a crash or immediate shutdown. It generates the least WAL volume but doesn't contain enough information for PITR.
- `logical` adds information necessary to support logical decoding.

> ***fsync (boolean)***

If this parameter is on, the PostgreSQL server will try to make sure that updates are physically written to disk, by issuing fsync() system calls or various equivalent methods (see wal_sync_method). This ensures that the database cluster can recover to a consistent state after an operating system or hardware crash.

While turning off fsync is often a performance benefit, this can result in unrecoverable data corruption in the event of a power failure or system crash. Thus it is only advisable to turn off fsync if you can easily recreate your entire database from external data.

> ***synchronous_commit (enum)***

Specifies how much WAL processing must complete before the database server returns a “success” indication to the client. 

When set to on, commits wait until replies from the current synchronous standby(s) indicate they have received the commit record of the transaction and flushed it to durable storage. This ensures the transaction will not be lost unless both the primary and all synchronous standbys suffer corruption of their database storage. 

> ***wal_sync_method (enum)***

Method used for forcing WAL updates out to disk. Possible values are:

- open_datasync
- fdatasync
- fsync (call fsync() at each commit)
- fsync_writethrough
- open_sync

> ***wal_writer_delay (integer)***

Specifies how often the WAL writer flushes WAL, in time terms. After flushing WAL the writer sleeps for the length of time given by wal_writer_delay, unless woken up sooner by an asynchronously committing transaction. 

If the last flush happened less than wal_writer_delay ago and less than wal_writer_flush_after worth of WAL has been produced since, then WAL is only written to the operating system, not flushed to disk. 

> ***wal_writer_flush_after (integer)***

Specifies how often the WAL writer flushes WAL, in volume terms. If the last flush happened less than wal_writer_delay ago and less than wal_writer_flush_after worth of WAL has been produced since, then WAL is only written to the operating system, not flushed to disk. If wal_writer_flush_after is set to 0 then WAL data is always flushed immediately. 

> ***commit_delay (integer)***

Setting commit_delay adds a time delay before a WAL flush is initiated. This can improve group commit throughput by allowing a larger number of transactions to commit via a single WAL flush, if system load is high enough that additional transactions become ready to commit within the given interval. However, it also increases latency by up to the commit_delay for each WAL flush.

Because the delay is just wasted if no other transactions become ready to commit, a delay is only performed if at least commit_siblings other transactions are active when a flush is about to be initiated.

> ***commit_siblings (integer)***

Minimum number of concurrent open transactions to require before performing the commit_delay delay. A larger value makes it more probable that at least one other transaction will become ready to commit during the delay interval.

</details>

---