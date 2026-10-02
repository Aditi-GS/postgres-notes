# 4. WAL Archiver

<details>
    <summary>An auxiliary process which, if enabled, saves copies of WAL files for the purpose of creating backups or keeping replicas current.
    <br>
    <u>Replica (server)</u> : A database that is paired with a primary database and is maintaining a copy of some or all of the primary database's data. The foremost reasons for doing this are to allow for greater access to that data, and to maintain availability of the data in the event that the primary becomes unavailable.
    </br>
    </summary>

- When `archive_mode=on`, the WAL archiver's job is to copy each completed 16 MB WAL segment file to an external archival location before the server recycles it. 

- `archive_timeout` forces a wal segment switch - in turn archiving - even if the segment isn't full - only if there has been some database activity generating WAL. Useful in low-traffic systems.

- `wal_level` must be `replica` (or `logical`) for archiving to work; `minimal` is insufficient for PITR.

- A base backup plus a continuous sequence of archived WAL files extending back to the backup's start time is required for recovery. 

- The archiver invokes the configured `archive_command`, dynamically substituting:
    `%p` : with full path to the segment file on disk
    `%f` : with just the filename
The file passed to the command must be treated as read-only - the archiver doesn't modify the original.

- The command runs sequentially. If it exits with 0, it indicates success. 

- If it exits non-zero, the archiver retries periodically (up to 3 immediate retries at 1-second intervals, then waits ~60 s between subsequent attempts) until it succeeds.  

- In such cases, a permanently failing archive command blocks the old completed WAL segment files and with new incoming WAL simultaneously, the `pg_wal` fills up. If the filesystem fills up, PostgreSQL performs a PANIC shutdown - no committed data is lost but DB is offline until space is freed.

> `Base Backup`: `pg_basebackup` is the standard PostgreSQL utility for taking online, consistent, file-system-level binary copies of an entire database cluster. It copies all data files, configuration files, and WAL (Write-Ahead Log) data - everything in the data directory simultaneously. It ensures consistency by automatically putting the server into backup mode, performing a checkpoint, and streaming WAL files in parallel. Though it backs up the entire database cluster, it cannot restore individual databases or objects.

### Continuous Archiving and Point-in-Time Recovery (PITR)

- The primary use of WAL (Write Ahead Log) maintained in `pg_wal` is crash safety (replayed from last checkpoint). As it is a sequential indefinitely long stream of every change applied to data files - it can be replayed to any point - enabling Point-In-Time-Recovery (PITR) i.e., restore the DB to any moment after the base backup.

- Continuously backing up 16 MB WAL segment files to another server loaded with the same base backup results in a `warm-standby` (near-real-time read only replica).

- Without archived WAL, restoration can only be performed till the last base backup. With it, replay and restore can be done till any point of time.

### SQL dump - another method to backup & restore

A text file is generated with SQL commands, such that, when fed back to the server, the database is recreated in the same state as it was at the time of taking the dump. 

`pg_dump` can also create files in other formats that allow for parallelism and more fine-grained control of object restoration. The dump represents a snapshot of the database at the time `pg_dump` began running - hence it is internally consistent.

`pg_dump` is a regular PostgreSQL client application. Meaning backup procedure can be performed by any remote host that has access to the database. But it doesn't have special permissions. In order to backup the entire database, it must be executed as a database superuser - having read access to all tables.

</details>