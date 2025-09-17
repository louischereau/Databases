# Triggers

A trigger is a special type of stored procedure that automatically executes when certain events occur in the database (e.g. data modification). Triggers are often used to enforce business rules, maintain data integrity, and audit changes to data.

| Triggers                            | Stored Procedures                       |
|-------------------------------------|-----------------------------------------|
| Fired automatically by an event.    | Run only when explicitly called.        |
| Don't allow parameters or transactions | Accept input parameters and transactions. |
| statements (e.g; BEGIN TRANSACTION or | Can return values as output.          |
| COMMIT).                            | Used for:                               |
| Cannot return values as output.     |   - general tasks                       |
| Used for:                           |   - user specific needs                 |
| - auditing (e.g. fincnacial inspection)|                                      |
| - intergrity enforcement            |                                         |



Types of triggers (based on T-SQL commands):
- Data Manipulation Language (DML) triggers (INSERT, UPDATE or DELETE statements)
- Data Definition Language (DDL) triggers (CREATE, ALTER or DROP statements)
- Logon triggers (fire in response to LOGON events)

| DML triggers                        | DDL triggers                            |
|-------------------------------------|-----------------------------------------|
| Events associated with DML statements | Events associated with DDL statements (CREATE, ALTER, DROP)        |
| Used with AFTER or INSTEAD OF       | Only used with AFTER                    |
| Attached to tables or views         | Attached to databases or servers        |
| Inserted and deleted special tables | No special tables                       |

## Trigger Definition

### With AFTER

AFTER triggers are executed after the triggering SQL statement is executed. They can be used to enforce referential integrity or to perform additional actions after data modification. Can be use for DML and DDL statements.

1. Initial event fires the trigger.
2. Initital event is executed.
3. Trigger actions execute.


```sql
CREATE TRIGGER TableTrigger --  Create trigger
ON Table -- Attach trigger to table
AFTER INSERT -- trigger behavior type
AS
PRINT('...'); -- Action executed by the trigger
```

#### Usage example

Cleanse data after insert then generate table report with procedure results and notify database admin.

```sql
CREATE TRIGGER SalesTrigger
ON Sales
AFTER INSERT
AS
EXEC sp_cleansing @Table = 'Sales';
EXEC p_generateReport;
EXEC sp_sendNotiication;
```

### With INSTEAD OF

INSTEAD OF triggers are executed in place of the triggering SQL statement. They can be used to override the default behavior of data modification statements. Can only be used for DML statements.

1. Initial event fires the trigger.
2. Initital event is not executed anymore.
3. Trigger actions execute.

```sql
CREATE TRIGGER TableTrigger --  Create trigger
ON Table -- Attach trigger to table
INSTEAD OF UPDATE -- trigger behavior type
AS
PRINT('...'); -- Action executed by the trigger
```

### Finding triggers

```sql
SELECT * FROM sys.triggers;
```

### Finding Server Level triggers
```sql
SELECT * FROM sys.server_triggers;
```
### Identify events that will fire triggers
```sql
SELECT * FROM sys.trigger_events;
```

```sql
SELECT * FROM sys.server_trigger_events
```

```sql
SELECT * FROM sys.trigger_event_types; -- Show all existing events that can be used to fire events
```
### List of triggers along with their fire events and objects they're attached to

```sql
SELECT * 
FROM sys.triggers AS t
INNER JOIN sys.trigger_events AS te ON t.object_id = te.object_id
LEFT OUTER JOIN sys.objects AS o ON o.object_id = t.parent_id;
```

### Info on execution of triggers that are currently stored in memory
```sql
SELECT * FROM sys.dm_exec_trigger_stats;
```


### Viewing a trigger definition

```sql
EXECUTE sp_helptext @objname = 'trgAfterInsert';
```

```sql
SELECT definition
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('trgAfterInsert');
```

```sql
SELECT OBJECT_DEFINITION(OBJECT_ID('trgAfterInsert'));
```

### Identifying triggers attached to a table

```sql
-- Find table id of products and use it to determine triggers attached
SELECT name AS tableName, object_id AS tableId
FROM sys.objects
WHERE name = '...';
```

```sql
-- Find existing triggers
SELECT o.name AS tableName,
    o.object_id AS tableId,
    t.name AS triggerName,
    t.object_id AS triggerId,
    t.is_disabled AS isDisabled,
    t.is_instead_of_trigger AS isInsteadOf
FROM sys.objects AS o
INNER JOIN sys.triggers AS t ON t.pare,t_id = o.object_id
WHERE o.name = 'products';
```
```sql
-- Viewing definition of triggers
SELECT o.name,
    o.object_id,
    t.name,
    t.object_id,
    t.is_disabled,
    t.is_instead_of_trigger,
    te.type_desc AS firingEvent,
    OBJECT_DEFINITION(t.object_id) AS triggerDefinition
FROM sys.objects AS o
INNER JOIN sys.triggers AS t ON t.parent_id = o.object_id
INNER JOIN sys.triggers AS te ON t.object_id = te.object_id
WHERE o.name = 'products';
```

### Deleting triggers

```sql
DROP TRIGGER triggerName; -- triggers atatched to tables or views
```

```sql
DROP TRIGGER triggerName ON DATABASE; -- database level triggers
```

```sql
DROP TRIGGER triggerName ON ALL SERVER;
```

### Diabling triggers

```sql
DISABLE TRIGGER triggerName ON tableName; -- disable trigger on table or view
```

```sql
DISABLE TRIGGER triggerName ON DATABASE; -- disable database level trigger
```

```sql
DISABLE TRIGGER triggerName ON ALL SERVER; -- disable server level trigger
```
### Enabling triggers

```sql
ENABLE TRIGGER triggerName ON tableName; -- enable trigger on table or view
```

```sql
ENABLE TRIGGER triggerName ON DATABASE; -- enable database level trigger
```

```sql
ENABLE TRIGGER triggerName ON ALL SERVER
; -- enable server level trigger
```

### Altering triggers

```sql
ALTER TRIGGER triggerName
ON tableName
INSTEAD OF / AFTER ...
AS
PRINT('...')
```


## DML

DML triggers are associated with tables or views and are executed in response to data modification events (INSERT, UPDATE, DELETE).

Used for:
- inititate action when manipulating data (inserting, modifying, deleting)
- prevent data manipulation
- tracking data or database object changes
- user auditing and database security (track user action)

"inserted" and "deleted" tables are special tables used by DML triggers and are automatically created by SQL server.

| Special Tables | INSERT trigger | UPDATE trigger | DELETE trigger |
|----------------|--------|--------|--------|
| inserted       |   ✅   |   ✅   |        |
| deleted        |        |   ✅   |   ✅   |

✅ = special table is avialable for trigger

#### Table auditing using triggers

```sql
CREATE TRIGGER Audit
ON Tables
AFTER INSERT, UPDATE, DELETE
AS BEGIN
    DECLARE @Insert BIT=0, @Delete BIT=0;
    IF EXISTS (SELECT * FROM inserted) SET @Insert=1;
    IF EXISTS (SELECT * FROM deleted) SET @Delete=1;
END;

INSERT INTO Tables (TableName, EventType, UserAccount, EventDate);

SELECT 'Tables' AS TableName,
    CASE WHEN @Insert=1 AND @Delete=0 THEN 'INSERT'
        WHEN @Insert=1 AND @Delete=1 THEN 'UPDATE'
        WHEN @Insert=0 AND @Delete=1 THEN 'DELETE'
    END AS Event,
    ORIGINAL_LOGIN(),
    GETDATE();
```

#### Notifying users

```sql
CREATE TRIGGER Notification
ON Tables
AFTER INSERT
AS BEGIN
    EXECUTE SendNotification @RecipientEmail='email', @EmailSubject='subject', @EmailBody='body';
END;
```

#### Triggers with conditional logic

```sql
CREATE TRIGGER ConfirmStock
ON Orders
INSTEAD OF INSERT
AS BEGIN
    -- Check whether there is usfficient stock for an order
    IF EXISTS (
        SELECT * FROM Products AS p
        INNER JOIN inserted AS i ON i.Product=p.Product
        WHERE p.Quantity < i.Quantity
    ) RAISERROR('');
    ELSE
        INSERT INTO dbo.Orders(...)
        SELECT * FROM inserted;
END;
```

#### Triggers that prevent and notify

```sql
CREATE TRIGGER PreventCustomerRemoval
ON Customers
INSTEAD OF DELETE
AS BEGIN
    DECLARE @EmailBodyText NVARCHAR(50)=(
        SELECT 'User' + ORIGINAL_LOGIN() + 'tried to remove from db.'
    );
    RAISERROR(...);
    EXECUTE SendNotification @RecipientEmail='email', @EmailSubject='subject', @EmailBody='body';
END;
```
## DDL

DDL triggers are associated with database or server events and are executed in response to DDL events (CREATE, ALTER, DROP).

Used for:
- prevent changes to database schema
- track changes to database schema
- enforce business rules related to database schema changes
- user auditing and database security (track user action)

### DDL triggers capabilities 

| Database Level                       | Server Level                            |
|-------------------------------------|-----------------------------------------|
| CREATE_TABLE, ALTER_TABLE, DROP_TABLE | CREATE_DATABASE, ALTER_DATABSE, DROP_DATABASE    |
| CREATE_VIEW, ALTER_VIEW, DROP_VIEW       | GRANT_SERVER, DENY_SERVER, REVOKE_SERVER                   |
| CREATE_INDEX, ALTER_INDEX, DROP_INDEX         | CREATE_CREDENTIAL, ALTER_CREDENTIAL, DROP_CREDENTIAL        |
| ADD_ROLE_MEMEBER, DROP_ROLE_MEMBER |                         |
| CREATE_STATISTICS, DROP_STATISTICS |                        |

Database level responds to statements related to tables or views.

### Syntax
```sql
CREATE TRIGGER triggerName
FOR CREATE_TABLE -- FOR = AFTER in SQL Server triggers. FOR often used in DDL triggers and AFTER in DML.
```

### Example
```sql
CREATE TRIGGER TrackTableChanges
ON DATABASE -- or ON ALL SERVER
FOR CREATE_TABLE,
    ALTER_TABLE,
    DROP_TABLE
AS
INSERT INTO TablesChangeLog (EventData, ChangedBy)
VALUES (EVENTDATA(), USER);
```
#### Preventing triggering events for DDL triggers

```sql
CREATE TRIGGER PreventTableDeletion
ON DATABASE
FOR DROP_TABLE
AS
RAISERROR(...)
ROLLBACK;
```

#### Database auditing
```sql
CREATE TRIGGER DatabaseAudit
ON DATABASE
FOR DDL_TABLE_VIEW_EVENTS -- Group event (specify a single event to cover all the cases that should fire the trigger)
AS BEGIN
    INSERT INTO DatabaseAudit (EventType, Database, Object, UserAccount, Query, EventTime);

    SELECT EVENTDATA().value('(/EVENT_INSTANCE/EventType)[1]', 'NVARCHAR(50)'), -- EVENTDATA() extracts details of operations and returns info in XML format
          EVENTDATA().value('(/EVENT_INSTANCE/DatabaseName)[1]', 'NVARCHAR(50)'),
          EVENTDATA().value('(/EVENT_INSTANCE/ObjectName)[1]', 'NVARCHAR(50)'),
          EVENTDATA().value('(/EVENT_INSTANCE/LoginName)[1]', 'NVARCHAR(50)'),
          EVENTDATA().value('(/EVENT_INSTANCE/TSQLCommand/CommandText)[1]', 'NVARCHAR(50)'),
          EVENTDATA().value('(/EVENT_INSTANCE/PostTime)[1]', 'Datetime'),
END;
```

#### Preventing server changes
```sql
CREATE TRIGGER PreventDatabaseDelete
ON ALL SERVER
FOR DROP_DATABASE
AS
    PRINT '...';
    ROLLBACK; 
```

## Logon

Logon triggers are executed in response to LOGON events. They can be used to enforce security policies or to audit logon activity.

Logon triggers fired after authentication phase (username / password check) but before user session established (when SQL Server info becomes available for queries).

Used for:
- restrict logon based on time of day or other criteria
- track logon activity for auditing purposes
- enforce security policies related to logon events
- prevent logon for specific users or roles

Syntax:
```sql
CREATE TRIGGER LoginAudit
ON ALL SERVER WITH EXECUTE AS 'sa' -- Logon triggers attached at server level so use ALL SERVER syntax. To avoid permission issues, trigger will be executed as teh 'sa' account with admin priviliges.
FOR LOGON
AS
INSERT INTO ServerLogonLog (LoginName, LoginDate, SessionId)
SELECT ORIGINAL_LOGIN(), GETDATE(), @@SPID
FROM sys.dm_exec_connections 
WHERE session_id = @@SPID;
```
## Disadvantages of triggers

- Difficult to view and detect existing triggers in database abd their behaviors.
- Invisible to client applications or when debugging code.
- Hard to follow logic of complex code when troubleshooting (search for source of problem)
- Can effect server performance when they are overused or poorly designed.





