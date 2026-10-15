# Stages of a Query

> add details
- protocol + per connection per query network streaming
- parser 
- analyzer
- rewriter
- planner + executor

1. A connection from an application program to the PostgreSQL server has to be established. The application program transmits a query to the server and waits to receive the results sent back by the server.  
    - A supervisor process called `postmaster` listens at a specified TCP/IP port for incoming connections.  Whenever it detects a request for a connection, it spawns a new backend process. 
    - Each client process connects to exactly one backend process. The backend processes communicate with each other and with other processes of the instance using semaphores and shared memory to ensure data integrity throughout concurrent data access.
    - Once a connection is established, the client process can send a query to the backend process it's connected to. The query is transmitted using plain text, i.e., there is no parsing done in the client. The backend process parses the query, creates an execution plan, executes the plan, and returns the retrieved rows to the client by transmitting them over the established connection.
2. The parser stage checks the query transmitted by the application program for correct syntax and creates a query tree.  
    - ***Lexer*** is defined in `scan.l` and is responsible for recognizing identifiers, the SQL key words etc. For every key word or identifier that is found, a token is generated and handed to the parser.  
    - ***Parser*** is defined in `gram.y` and consists of a set of grammar rules and actions that are executed whenever a rule is fired. The code of the actions (which is actually C code) is used to build up the parse tree.
    - The parser stage creates a parse tree using only fixed rules about the syntactic structure of SQL. It does not make any lookups in the system catalogs.  
3. The rewrite system takes the query tree created by the parser stage and looks for any rules (stored in the system catalogs) to apply to the query tree. It performs the transformations given in the rule bodies.
    - One application of the rewrite system is in the realization of views. Whenever a query against a view (i.e., a virtual table) is made, the rewrite system rewrites the user's query to a query that accesses the base tables given in the view definition instead.  
4. The planner/optimizer takes the (rewritten) query tree and creates a query plan that will be the input to the executor. It does so by first creating all possible paths leading to the same result. Next the cost for the execution of each path is estimated and the cheapest path is chosen. The cheapest path is expanded into a complete plan that the executor can use.
5. The executor takes the plan from the planner and recursively processes it to extract the required rows. This works as a demand-pull pipeline: each time a plan node is called, it delivers one more row, or signals that it's done. 

#### Glossary #### 
- **identifiers and keywords**: Tokens in a query that identify tables, columns or other db objects are called `identifiers`. `Keywords` are tokens/words that have fixed meaning in SQL language (like SELECT, UPDATE, INSERT, etc.)
- **postmaster**: The very first process of an instance. It starts and manages the auxiliary processes and creates backend processes on demand.
- **instance**: A group of backend and auxiliary processes that communicate using a common shared memory area. One postmaster process manages the instance. one instance manages exactly one database cluster (a collection of databases stored in a common file system location) and occupies a unique IP/port combination. Many instances can share the same server as long as they use different data areas and TCP/IP ports.
- **backend process**: Process of an instance which acts on behalf of a client session and handles its requests.
- **auxillary process**:  A process within an instance that is in charge of some specific background task for the instance. The auxiliary processes consist of the `autovacuum launcher` (but not the autovacuum workers), the `background writer`, the `checkpointer`, the `logger`, the `startup process`, the `WAL archiver`, the `WAL receiver` (but not the WAL senders), the `WAL summarizer`, and the `WAL writer`.
- **client process**: Any process, possibly remote, that establishes a session by connecting to an instance to interact with a database.
- **WAL record**: A low-level description of an individual data change. It contains sufficient information for the data change to be re-executed (replayed) in case a system failure causes the change to be lost. WAL records use a non-printable binary format.

> How does raw parsing handle dynamic SQL?

#### References ####
Overview of PostgreSQL Internals https://www.postgresql.org/docs/current/overview.html

Frontend/Backend Protocol
https://www.postgresql.org/docs/current/protocol.html

The Rule System and Query Rewriter
https://www.postgresql.org/docs/current/rules.html

System Catalogs
https://www.postgresql.org/docs/current/catalogs-overview.html

Glossary
https://www.postgresql.org/docs/current/glossary.html

> If you wish to check the actual parse tree, query tree and plan tree for a query then, in the PSQL CLI first execute these lines before running you query: 
```sql
SET client_min_messages=log;
SET debug_pretty_print=on;

--- for Parse Tree
SET debug_print_parse=on;

--- for Query Tree
SET debug_print_rewritten=on;

--- for Plan Tree
SET debug_print_plan=on;
```

---
***`TRANSACTION LIFECYCLE`***  
1. **Active State**
    - Modifications made in memory (buffers)
2. **Partially Commited state**
    - Last statement executed successfully -> preparing to make changes permanent
3. **Commited State**
    - Transaction completed successfully -> changes made DURABLE
4. **Failed State**
5. **Aborted/Roll Back State**
6. **Terminated State**
    - Final consistent state post commit or rollback => All resources released (locks, memory,...)  

Every transaction is identified by a unique VirtualTransactionId (also called virtualXID or vxid), which is comprised of a backend's process number (or procNumber) and a sequentially-assigned number local to each backend, known as localXID. For example, the virtual transaction ID 4/12532 has a procNumber of 4 and a localXID of 12532.

Non-virtual TransactionIds (or xid) are assigned sequentially to transactions from a global counter used by all databases within the PostgreSQL cluster. This assignment happens when a transaction first writes to the database. The internal transaction ID type xid is 32 bits wide and wraps around every 4 billion transactions. 

The transaction IDs of currently executing transactions are shown in pg_locks in columns virtualxid and transactionid. Read-only transactions will have virtualxids but NULL transactionids, while both columns will be set in read-write traThe transaction IDs of currently executing transactions are shown in pg_locks in columns virtualxid and transactionid. Read-only transactions will have virtualxids but NULL transactionids, while both columns will be set in read-write transactions.

When it comes to transactions, `BEGIN` AND `COMMIT` are the default keywords within which steps of a transaction are placed in a.k.a transaction blocks. They are the markers indication start and end of a single transaction. If not specified explicitly for SQL commands (like SELECT, INSERT,...) - they are executed as a transaction by default.

While the transaction block controls the results - all or nothing - `SAVEPOINT` act as named recovery markers that help regain a little control within a transaction block.  
This marker records the position after the statements that already ran. Later statements can still be undone back to that marker, while earlier statements stay pending in the open transaction.
By including `SAVEPOINTS`, a transaction has a place to return or  `ROLLBACK TO` without ending the transaction completely. Several savepoints can exist in the same transaction, which lets us mark more than one recovery point during a longer write flow.  

Subtransactions are started inside transactions, allowing large transactions to be broken into smaller units. Subtransactions can commit or abort without affecting their parent transactions, allowing parent transactions to continue. They can be started explicitly using the SAVEPOINT command.

https://www.postgresql.org/docs/current/transactions.html

https://medium.com/@AlexanderObregon/savepoints-in-sql-transactions-e22b13cfba16

---