# Transactions

A transaction is a sequence of one or more SQL operations treated as a single unit. Transactions ensure data integrity and consistency, especially in multi-user environments.

```sql
BEGIN {TRAN | TRANSACTION}; -- Start a new transaction
-- SQL operations (INSERT, UPDATE, DELETE, etc.)
COMMIT; -- Save changes if all operations succeed
ROLLBACK; -- Undo changes if any operation fails. More precisely reverts a transaction to the begining of it or a savepoint inside the transaction.
```

```sql
BEGIN {TRAN | TRANSACTION} [transaction_name |@tran_name_variable]
[WITH MARK ['decription']]
```

```sql
COMMIT {TRAN | TRANSACTION} [transaction_name |@tran_name_variable]
[WITH (DELAYED_DURABILITY = {OFF | ON})]
```

```sql
ROLLBACK {TRAN | TRANSACTION} [transaction_name |@tran_name_variable | savepoint_name | @savepoint_variable]
```

@@TRANCOUNT = Number of BEGIN TRAN statements that are active in current connection. If > 0 then open transactions present else no open trnsactions.

Modified by:
- BEGIN TRAN => @@TRANCOUNT + 1
- COMMIT => @@TRANCOUNT - 1
- ROLLBACK => @@TRANCOUNT = 0 (except with save point name)

## Savepoints

A savepoint allows you to set a point within a transaction that you can roll back to without affecting the entire transaction.

Summary:
- Markers within transaction.
- Allow to rollback to savepoint.
- SAVE { TRAN | TRANSACTION } { save_point_name | @save_point_variable }

```sql
BEGIN TRAN;

SAVE TRAN savepoint1;
INSERT INTO Customers VALUES (...)

SAVE TRAN savepoint2;
INSERT INTO Customers VALUES (...);

ROLLBACK TRAN savepoint2; -- Rolls back to savepoint2, undoing the second INSERT

ROLLBACK TRAN savepoint1;

SAVE TRAN savepoint3;
INSERT INTO Customers VALUES (...);

COMMIT TRAN; -- Commits the transaction, saving changes made before savepoint1
```
## XACT_ABORT and XACT_STATE

XACT_ABORT is a setting that controls the behavior of transactions when an error occurs. When XACT_ABORT is ON, if a runtime error occurs, the entire transaction is automatically rolled back and terminated. When OFF, only the statement that caused the error is rolled back, and the transaction remains active. Hence specifies whether the current transsaction will be automatically rolled back when an error occurs.

```sql
SET XACT_ABORT { ON | OFF };
```
XACT_STATE() function returns the current state of the transaction in the current session. It can return three possible values:
- 1: open and committable transaction exists.
- 0: No open transactions exists.
- -1: open but uncommittable transaction (doomed transaction). Can't commit, can't rollback to savepoint, can't make any changes. Can rollback full transaction and can read data.

### Open and committable

```sql
SET XACT_ABORT OFF;
BEGIN TRY
    BEGIN TRAN;
        INSERT INTO customers VALUES(...);
        INSERT INTO customers VALUES(...);
    COMMIT TRAN; -- Commit if all operations succeed
END TRY
BEGIN CATCH
    IF XACT_STATE() = -1
        ROLLBACK TRAN;
    IF XACT_STATE() = 1
        COMMIT TRAN;
    SELECT ERROR_MESSAGE() AS error_message;
END CATCH
```

### Open and uncommittable

```sql
SET XACT_ABORT ON;
BEGIN TRY
    BEGIN TRAN;
        INSERT INTO customers VALUES(...);
        INSERT INTO customers VALUES(...); -- Assume this causes a runtime error
    COMMIT TRAN; -- This line will not be reached if an error occurs
END TRY
BEGIN CATCH
    IF XACT_STATE() = -1
        ROLLBACK TRAN; -- Rollback the entire transaction
    SELECT ERROR_MESSAGE() AS error_message;
END CATCH
```

## Transaction Isolation Levels

Transaction isolation levels define the degree to which the operations in one transaction are isolated from those in other transactions. They help manage concurrency and ensure data integrity.

Concurrency = two or more transactions that read / write shared data at the same time.

Isolate transactiosn from other transactions:;
- READ COMMITTED (default)
- READ UNCOMMITTED
- REPEATABLE READ
- SERIALIZABLE
- SNAPSHOT

```sql
SET TRANSACTION ISOLATION LEVEL {
    READ UNCOMMITTED | 
    READ COMMITTED | 
    REPEATABLE READ | 
    SNAPSHOT | 
    SERIALIZABLE
};
```

### Knowing the current isolation level:

```sql
SELECT CASE transaction_isolation_level
    WHEN 0 THEN 'Unspecified'
    WHEN 1 THEN 'Read Uncommitted'
    WHEN 2 THEN 'Read Committed'
    WHEN 3 THEN 'Repeatable Read'
    WHEN 4 THEN 'Serializable'
    WHEN 5 THEN 'Snapshot'
END AS isolation_level
FROM sys.dm_exec_sessions
WHERE session_id = @@SPID;
```

### READ UNCOMMITED

- Least restrictive isolation level 
- Read rows modified by other transaction whih hasn't been committed / rolled back.
- Can encounter dirty reads, non-repeatble reads, phantom reads.
- Pros: Can be faster, does not block other transactions.
- Cons: Allows dirty reads, non-repeatble reads, phantom reads.
- Use when:
    - Don't want to be blocked by other transactions but don't mind concurrency phenomena.
    - Explicitly want to watch uncommitted data.

Concurrency phenomena:
- Dirty reads: Transaction reads data written by another uncommitted transaction. If the other transaction rolls back, the data read is invalid. In other words, dirty reads occur when a transaction reads data that has been modified by another transaction which has not yet been committed.

- Non-repeatable reads: Transaction reads the same row twice and gets different data because another committed transaction modified the row in between the two reads. In other words, non-repeatable reads occur whan a transaction reads a record twice but the first result is different from the second result as a consequence of another committed transaction altering this data.

- Phantom reads: Transaction re-executes a query returning a set of rows that satisfy a search condition and finds that the set of rows satisfying the condition has changed due to another recently committed transaction (new rows added or existing rows deleted). In other words, phantom reads occur when a transaction reads a record twice but the first record it gets is different from the second result as a consequence of another committed transaction having inserted a row.

### READ COMMITED

- Default isolation level
- Can't read data modified by other transaction taht hasn't committed or rolled back.
- Can encounter non-repeatable reads, phantom reads.
- Pros: Prevents dirty reads, balances data integrity and concurrency.
- Cons: Allows non-repeatable reads, phantom reads, can be blocked by another transaction.
- Use when:
    - Want to ensure that only read committed data
    - General purpose applications where a balance between data integrity and concurrency is needed.


### REPEATABLE READ

- Can't read uncommitted data from other transactions.
- If some data is read, other transactions cannot modify that data until REPEATABLE READ transation finishes.
- Prevents dirty reads and non-repeatable reads.
- Can encounter phantom reads only.
- Pros: Prevents other transaction from modifying the data you are reading (non-repeatable reads and dirty reads).
- Cons: Allows phantom reads. Can be blocked by a REPEATABLE READ transaction.

### SERIALIZABLE

- Most restrictive isolation level.
- Transactions are completely isolated from each other.
- Prevents dirty reads, non-repeatable reads, phantom reads.
- Can lock records under SERIALIZABLE using a query with a WHERE clause based on an index range.
- If query not based on index rangethen will lock complete table.
- Pros: Ensures highest level of data integrity / consistency.
- Cons: Can lead to significant blocking and reduced concurrency. Can be slower. Can be blocked by a SERIALIZABLE transaction.
- Use when data consistency is a must.

### SNAPSHOT

- Each transaction works with a snapshot of the database as it existed at the start of the transaction.
- Every modification is stored in the tempDB table.
- Only see committed changes that ocured before the start of the SNAPSHOT transaction and own changes.
- Can't see any changes mady by other transactions after the start of the SNAPSHOT transaction.
- Reading don't block writings and writings don't block readings. Hence SNAPSHOT does not block transactions while SERIALIZABLE does.
- Can have update conflicts. If two transactions try to update the same data at the same time.
- Prevents dirty reads, non-repeatable reads, phantom reads.
- Pros: High level of data integrity without locking resources. Readers do not block writers and writers
    do not block readers.
- Cons: Requires additional tempdb resources to maintain row versions. Can lead to increased storage and performance overhead.
- Use when:
    - Want high level of data integrity without locking resources.
    - Readers should not block writers and writers should not block readers.

Syntax:
```sql
ALTER DATABASE database_name
SET ALLOW_SNAPSHOT_ISOLATION { ON | OFF };
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
```

### READ COMMITED SNAPSHOT

- Changes behavior of READ COMMITTED.
- makes every READ COMMITTED statement only able to see committed changes that occured before teh start of that statement.
- Can't have update conflicts.

Syntax:
```sql
ALTER DATABASE database_name
SET READ_COMMITTED_SNAPSHOT { ON | OFF };
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### WITH (NO LOCK)

- Used to read uncommitted data
- Equivalent to READ UNCOMMITTED isolation level but only for the specific query.
- READ UNCOMMITTED applies to the entire connection whereas WITH (NO LOCK) applies to a specific table.
- Use under any isolation level when you just want to read uncommitted data from specific tables. Or you want to stop being blocked while maintaining isolation level.

```sql
SELECT * FROM table_name WITH (NOLOCK) WHERE condition;
```
