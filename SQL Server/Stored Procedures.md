# Stored Procedures 

A stored procedure is a precompiled collection of one or more SQL statements that can be executed as a single unit. Stored procedures are used to encapsulate logic, improve performance, and enhance security in database applications.

Store procedure is a routine that:
- accepts input parameters
- performs actions (EXECUTE, SELECT, INSERT, UPDATE, DELETE ...)
- returns status (success or failure)
- returns output parameters 

Output parameters can be of any data type (except table) and can declare multiple per SP.

Stored procedures return value is often used to indicate success (0) or failure (> 0) and integer data type only.

Stored Procedures used to:
- reduce execution time
- reduce network traffic
- allow for modular programming
- improved security

Use Stored Procedures for CRUD operations as it:
- decouples SQL code from other application layers in N-tier architecture.
- improves security
- improves performance (reuse of query optimizer execution plan)

## Stored Procedures vs UDFs

| UDFs             | Stored Procedures   |
|------------------|---------------------|
| - Must return value     | - Return value is oprional   |
| - Allow table value     | - No table value   |
| - Embedded SELECT execute allowed  | - Cannot embed in SELECT to execute   |
| - No output paramters     | - Return output parameters and status   |
| - No INSERT, UPDATE, DELETE    | - INSERT, UPDATE, DELETE allowed   |
| - Cannot execute Store Procedure    | - Can execute functions and Stored Procedures   |
| - No error handling   | - Error handling with TRY ... CATCH   |

## Syntax

```sql
CREATE PROCEDURE procedureName  -- Cannot have same name as UDF
    @parameter1 datatype [= default_value], -- input parameter
    @parameter2 datatype [= default_value] OUTPUT, -- output parameter(should be returned)
    ...
AS
SET NOCOUNT ON -- Prevents SQL from returning the number of rows affected by the stored procedure to the caller.
BEGIN 
    SELECT ... FROM ... WHERE ...
    RETURN -- keyword is optional
END
```

## CREATE (create records in table)

```sql
CREATE PROCEDURE dbo.cusp_Create (
    @TripDate AS DATE,
    @TripHours AS NUMERIC(18, 0)
)
AS BEGIN
    INSERT INTO dbo.Table(date, TripHours)
    VALUES (@TripDate, @TripHours)
    SELECT date, TripHours
    FROM dbo.Table
    WHERE date = @TripDate
END
```

## READ (return records with matching value)

```sql
CREATE PROCEDURE dbo.cusp_Read (@TripDate AS DATE)
AS BEGIN
    SELECT Date, TripHours
    FROM Table
    WHERE date = @TripDate
END;
```

## UPDATE (update existing records)

```sql
CREATE PROCEDURE dbo.cusp_update(
    @TripDate AS date,
    @TripHours AS NUMERIC(18, 0)
)
AS BEGIN
    UPDATE dbo.Table
    SET Date = @TripDate,
        TripHours = @TripHours
    WHERE Date = @TripDate
END;
```

## DELETE (delete matching records in table)

```sql
CREATE PROCEDURE dbo.cusp_delete (
    @TripDate AS DATE,
    @RowCountOut INT OUTPUT
)
AS BEGIN
    DELETE 
    FROM Table
    WHERE Date = @TripDate
    SET @RowCountOut = @@ROWCOUNT
END
```

## Ways to execute stored procedures

Ways to execute stored procedures:
- No output parameter nor return values
- Store return value
- With ouput parameter 
- With otuput parameter abd store return value
- Store result set

### No output parameter or return value

```sql
EXEC dbo.cusp_Update @TripDate = '1/5/2017', @TripHours='300'
```

### With output parameter

```sql
DECLARE @RideHrs AS NUMERIC(18, 0); -- A local variable should be declared to store value.

EXEC dbo.cusp_Sum
    @DateParam = '1/5/2017',
    @RideHrsOut = @RideHrs OUTPUT; -- OUTPUT keyword is required.

SELECT @RideHrs AS TotalRideHrs;
```

### With return value

```sql
DECLARE @ReturnValue AS INT; -- Value to indicate stored procedure was susccessful or not

EXEC @ReturnValue = dbo.cusp_UPDATE 
    @TripDate = '1/5/2017',
    @TripHours = '350';

SELECT @ReturnValue AS ReturnValue;
```

### With output parameter and return value

```sql
DECLARE @ReturnValue AS INT;
DECLARE @RowCount AS INT;

EXEC @ReturnValue = dbo.cusp_DELETE,
    @TripDate = '1/5/2017',
    @RowCountOut = @RowCount OUTPUT;

SELECT @ReturnValue AS ReturnValue, 
       @RowCount AS rowCount;
```
### Store result set

```sql
DECLARE @ResultSet AS Table (TripDate AS DATE, TripHours AS NUMERIC(18,0))

INSERT INTO @ResultSet
EXEC dbo.cusp_Read @TripDate = '1/5/2017'

SELECT * FROM @ResultSet
```

## Error handling in Stored Procedures

Anticipation, detection and resolution of errors. Maintains normal flow of execution. Intergrated into initial query design.

### Example

```sql
ALTER PROCEDURE dbo.cusp_Create
    @TripDate AS NVARCHAR(30),
    @RideHrs AS NUMERIC,
    @ErrorMsg AS NVARCHAR(MAX) = NULL OUTPUT 
AS BEGIN
    BEGIN TRY
        INSERT INTO Table (Date, TripHours)
        VALUES (@TripDate, @RideHrs)
    END TRY
    BEGIN CATCH
        SET @ErrorMsg = 'Error_Num: ' + CAST(ERROR_NUMBER() AS VARCHAR) + 'Error_Sev: ' + CAST(ERROR_SEVERITY() AS VARCHAR) + 'Error_Msg: ' + ERROR_MESSAGE()
    END CATCH
END

-- Show Error

DECLARE @ErrorMsgOut NVARCHAR(MAX)

EXEC dbo.cusp_Create 
    @TripDate = '1/32/2018', 
    @RideHrs = 100, 
    @ErrorMsg = @ErrorMsgOut OUTPUT

SELECT @ErrorMsgOut AS ErrorMessage
```

To switch control from TRY block to CATCH block use:
- THROW() => statements following NOT executed.
- RAISERROR() => statements following can be executed. Generates new error and cannot access details of original error (e.g. line number)

## Data Imputation

Data imputation methods:
- Mean -> replace missing values with mean
- Hot deck -> replace missing value with randomly selected value
- Omission -> exclude missing records

Note: all techniques create bias

### Mean

Doesn't change the mean value. Increase correlations with other columns.

```sql
CREATE PROCEDURE dbo.Mean
AS BEGIN
    DECLARE @Avg AS FLOAT
    SELECT @Avg = AVG(Duration)
    FROM Table
    WHERE Duration > 0
    UPDATE Table
    SET Duration = @Avg 
    WHERE Duration = 0
END;
```

### Hot Deck

```sql
-- Selects 1 record from table of 1000 random records.

CREATE PROCEDURE dbo.GetHotDeck()
RETURNS DECIMAL(18, 4)
AS BEGIN
    RETURN(
        SELECT TOP 1 Duration
        FROM Table
        TABLESAMPLE(1000 rows)
        WHERE Duration > 0
    )
END

-- Implement Hot Deck impoutation by clling function each time we encounter a duration value of 0 via CASE statement.
SELECT StartDate, 
        "TripDuration" = CASE WHEN Duration > 0 THEN Duration ELSE dbo.GetHotDeck() END
FROM Table;
```

### Omission

```sql
SELECT DATENAME(weekday, StartDate) AS DayOfWeek,
        AVG(Duration) AS 'AvgDuration'
FROM Table
WHERE Duration > 0
GROUP BY DATENAME(weekday, StartDate)
ORDER BY AVG(Duration) DESC;
```
