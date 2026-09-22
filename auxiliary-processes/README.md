# PostgreSQL Auxiliary Processes

As per the official PostgreSQL documentation: 

> **Auxiliary Process** is a process within an instance that is in charge of some specific background task for the instance. The auxiliary processes consist of the autovacuum launcher (but not the autovacuum workers), the background writer, the checkpointer, the logger, the startup process, the WAL archiver, the WAL receiver (but not the WAL senders), the WAL summarizer, and the WAL writer.

Click on the links below, to dive into more details for each auxiliary process:

1. [Checkpointer](./checkpointer.md)
2. [Background Writer](./background-writer.md)

    a. [Checkpointer, BgWriter & Backend](./checkpointer-bgwriter-backend.md)

3. [WAL Writer](./wal-writer.md)
4. [WAL Archiver](./wal-archiver.md)
5. [WAL Receiver](./wal-receiver.md)
6. [WAL Summarizer](./wal-summarizer.md)
7. [Autovaccum Launcher](./autovacuum-launcher.md)
8. [Startup Process](./startup-process.md)
9. [Logger](./logger.md)

### Other mentioned processes
1. [Autovacuum Workers](./autovacuum-worker.md)
2. [WAL Sender](./wal-sender.md)
3. [Logical Replication Launcher](./logical-replication-launcher.md)