## [Stages of a Query](./stages-of-a-query.md)

## File System Layout
1. [Database Clusters]
2. [Databases]
3. [Relations]
4. [Pages]
5. [Tuples]

## PostgreSQL Auxiliary Processes

As per the official PostgreSQL documentation: 

> **Auxiliary Process** is a process within an instance that is in charge of some specific background task for the instance. The auxiliary processes consist of the autovacuum launcher (but not the autovacuum workers), the background writer, the checkpointer, the logger, the startup process, the WAL archiver, the WAL receiver (but not the WAL senders), the WAL summarizer, and the WAL writer.

1. [Checkpointer](./auxiliary-processes/checkpointer.md)
2. [Background Writer](./auxiliary-processes/background-writer.md)
3. [WAL Writer](./auxiliary-processes/wal-writer.md)
4. [WAL Archiver](./auxiliary-processes/wal-archiver.md)
5. [WAL Receiver](./auxiliary-processes/wal-receiver.md)
6. [WAL Summarizer](./auxiliary-processes/wal-summarizer.md)
7. [Autovaccum Launcher](./auxiliary-processes/autovacuum-launcher.md)
8. [Startup Process](./auxiliary-processes/startup-process.md)
9. [Logger](./auxiliary-processes/logger.md)

2.1. [Checkpointer, BgWriter & Backend](./auxiliary-processes/checkpointer-bgwriter-backend.md)

### Additional Processes
1. [Autovacuum Workers](./auxiliary-processes/additional/autovacuum-worker.md)
2. [WAL Sender](./auxiliary-processes/additional/wal-sender.md)
3. [Logical Replication Launcher](./auxiliary-processes/additional/logical-replication-launcher.md)

## PostgreSQL Memory Components
### Shared Memory 
1. [Shared Buffers](./memory-components.md#1-shared-buffers)
2. [WAL Buffers](./memory-components#2-wal-buffers)
3. [CLOG Buffers (Commit Log Buffers)](./memory-components#3-clog-buffers-commit-log-buffers)
4. [PROC Array](./memory-components#4-proc-array)
5. [Lock Space](./memory-components#5-lock-space)
6. [Buffer Descriptors](./memory-components#6-buffer-descriptors)

### Per-Backend Memory
1. [Work Memory](./memory-components#1-work-memory)
2. [Maintenance Work Memory](./memory-components#2-maintenance-work-memory)
3. [Temp Buffers](./memory-components#3-temp-buffers)
4. [Vacuum Buffers](./memory-components#4-vacuum-buffers)
5. [Relcache](./memory-components#5-relcache)
6. [Syscache](./memory-components#6-syscache)