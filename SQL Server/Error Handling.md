# Error Handling

## Try / catch

```sql
BEGIN TRY
    -- SQL STatement to try
END TRY
BEGIN CATCH
    -- Statement to run if error in try block
END CATCH
```

## Nested Try / Catch

```sql

BEGIN TRY
    -- SQL Statement to try
END TRY
BEGIN CATCH
    BEGIN TRY
        -- SQL Statement to try if error in outer try block
    END TRY
    BEGIN CATCH
        -- Statement to run if error in inner try block
    END CATCH
END CATCH
```

## View error messages

```sql
SELECT * FROM sys.messages
```

### Functions

- ERROR_NUMBER(): returns number of te error
- ERROR_SEVERITY(): returns error severity (11 - 19)
- ERROR_STATE(): returns state of the error
- ERROR_LINE(): returns number of the line of the error
- ERROR_PROCEDURE(): returns name of stored procedure / trigger. NULL if there is no stored procedure / trigger
- ERROR_MESSAGE(): returns text of error message

## Raise error statements

### RAISERROR

```sql
RAISERROR (
    {'message' | message_id | @local_variable_message}, 
    severity, 
    state,
    [argument [,...n]],
    [WITH OPTION [,...n]]
);
```
#### Examples

```sql
IF NOT EXISTS (SELECT * FROM table WHERE id = 15)
RAISERROR ('', state, severity);
```

```sql
RAISERROR('No %s with id %d', severity, state, 'person', 15)
```

```sql
RAISERROR('%d%% discount', severity, state, 50);
```

### THROW

```sql
THROW [ { error_number | @local_variable }, { message | @local_variable }, {
    state | @local_variable } ] [ ; ]
```
#### Examples

```sql
DECLARE @staff_id AS INT=500;
DECLARE @my_message NVARCHAR(500)=CONCAT('There is no staff member for id', @staff_id);

IF NOT EXISTS (SELECT * FROM staff WHERE staff_id=@staff_id)
THROW 50000, @my_message, 1;
```

```sql
FORMATMESSAGE(
    {'message_string' | message_number},
    [partam_value [,...n]]
)

DECLARE @staff_id AS INT=500;
DECLARE @my_message NVARCHAR(500)=FORMATMESSAGE('There is no staff member for id %d', @staff_id);

IF NOT EXISTS (SELECT * FROM staff WHERE staff_id=@staff_id)
THROW 50000, @my_message, 1;
```

#### Possible message numbers

```sql
SELECT * FROM sys.messages;
```

#### Add a new message

```sql
EXEC sp_addmessage
    @msgnum = 50001,
    @severity = 16,
    @msgtext = 'There is no staff member for id %d',
    @lang = 'us_english',
    WITH LOG {'TRUE' | 'FALSE'},
    @replace = 'TRUE';
```



