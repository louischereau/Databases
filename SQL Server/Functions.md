# User defined functions

User defined functions are routines that accept parameters, perform an action (such as a complex calculation), and return the result of that action as a value. Functions can be used in SQL statements anywhere that expressions are allowed.

User defined functions are routines that:
- can accept input parameters
- perform an action (such as a complex calculation)
- return result (single scalar value or table)
- can reduce execution time
- can reduce network traffic
- allows for modular programming

Modular Programming = softwrae design technique thaat emphasizes separating functionality of a program into independent, interchangeable modules. These modules can be reused in various queries.

## Scalar user defined functions

### Scalar UDF with no onput parameter

```sql
CREATE FUNCTION GetTomorrow()
RETURNS DATE AS BEGIN -- Describes scalar data type that will be returned
RETURN (SELECT DATEADD(DAY, 1, GETDATE())) -- Function logic
END
```
BEGIN and END keywords are control flow statements.

### Scalar UDF with one input parameter

```sql
CREATE FUNCTION GetRideHrsOneDay(@DateParam date)
RETURNS NUMERIC AS BEGIN
RETURN (
    SELECT SUM(DATEDIFF(second, PickupDate, DropOffDate)) / 3600
    FROM Table
    WHERE CONVERT(date, PickupDate) = @DateParam
)
END
```

### Scalar UDF with 2 input parameters

```sql
CREATE FUNCTION GetRideHrsDateRange(
    @StartDate DATETIME,
    @EndDate DATETIME
)
RETURNS NUMERIC AS BEGIN
RETURN (
    SELECT SUM(DATEDIFF(second, PickupDate, DropOffDate)) / 3600
    FROM Table
    WHERE PickupDate > @StartDate
        AND DropOffDate < @EndDate
)
END;
```

### Inline Table Valued Functions (ITVF)

Inline table valued functions return a table data type and contain a single `SELECT` statement. They are similar to views, but can accept parameters.

```sql
CREATE FUNCTION SumLocationStats(@StartDate AS DATETIME = '1/1/2017') -- Parameter @StartDate has a optional default value of 1/1/2017.
RETURNS TABLE AS RETURN (
    SELECT PULocationID AS PickupLocation
            COUNT(ID) AS RideCount,
            SUM(TripDistance) AS TotalTripDistance
    FROM Table
    WHERE CAST(PickupDate AS DATE) = @StartDate
    GROUP BY PULocationID;
)
```
Table valued function does not require BEGIN nor END if the function body is a single sttement.

Returns result of SELECT statement along with assigned column nales . No table variable, BEGIN/END block nor INSERT. Faster performance than MSTVF.

### Multi-statement Table Valued Functions (MSTVF)

Multi-statement table valued functions return a table data type and can contain multiple statements to populate the table variable.

Only use when cannot write query in a single statement.

```sql
CREATE FUNCTION CountTripAvgFareDay(
    @Month CHAR(2), @Year CHAR(4)
)
RETURNS @TripCountAvgFare TABLE ( -- Need to declare table variable to be returned with column names and data types.
    DropOffDate DATE,
    TripCount INT,
    AvgFare NUMERIC
) AS BEGIN -- Need to use BEGIN/END block to contain mutliple SQL statements.
INSERT INTO @TripCountAvgFare -- INSERT data into table variable.
SELECT CAST(DropOffDate AS DATE),
        COUNT(ID),
        AVG(FareAmount) AS AvgFareAmt
FROM Table
WHERE DATEPART(month, DropOffDate) = @Month
    AND DATEPART(year, DropOffDate) = @Year
GROUP BY CAST(DropOffDate AS DATE)
RETURN -- RETURN should be last statement within BEGIN/END block.
END;
```

## User defined functions in action

### Execute scalar with SELECT

```sql
SELECT dbo.GetTomorrow();
```
dbo is the schema where function exists. Must be specified when executing a UDF otherwise SQL will automatically assign it to user's default schema.

### Execute scalar UDF with EXEC and store result

```sql
DECLARE @TotalRideHrs AS NUMERIC -- Declare a local numeric variable
EXEC @TotalRideHrs = dbo.GetRideHrsOneDay @DateParam = '1/15/2017' -- Execute GetRideHrsOneDay() function and assign result to tthe @TotalRideHrs variable.
SELECT 'Total Ride Hours for 1/15/2017:', @TotalRideHrs
```

### SELECT parameter value and scalar UDF

```sql
DECLARE @DateParam AS DATE = (
SELECT TOP 1 CONVERT(date, PickupDate)
FROM Table
ORDER BY PickupDate DESC) -- Set variable to oldest date in table
SELECT @DateParam AS DateParam, dbo.GetRideHrsOneDay(@DateParam) --Pass to function 
```

### Execute table valued functions using the SELECT and FROM keywords

```sql
SELECT TOP 10 *
FROM dbo.SumLocationStats('1/09/2017')
ORDER BY RideCount DESC
```
### Execute table valued function and store results in table variable

```sql
DECLARE @CountTripAvgFareDay TABLE(
    DropOffDate DATE,
    TripCount INT,
    AvgFare NUMERIC
)
INSERT INTO @CountTripAvgFareDay
SELECT TOP 10 *
FROM dbo.CountTripAvgFareDay('01', '2017')
ORDER BY DropOffDate ASC

SELECT * FROM @CountTripAvgFareDay
```

## Maintaining User defined functions

### ALTER function
```sql
ALTER FUNCTION SumLocationStats(@EndDate AS DATETIME = '1/01/2017')
RETURNS TABLE AS RETURN (
    SELECT ...
    FROM Table
    WHERE CAST(DropOffDate AS DATE) = @EndDate
    GROUP BY PULocationID;
    ) 
```
### Create or ALTER

```sql
CREATE OR ALTER FUNCTION SumLocationStats(...)
```
Cannot use ALTER to convert table valued function from multi statement to inline or vis-versa. Must delete then recreate function.

### Delete function

```sql
DROP FUNCTION dbo.CountTripAvgFareDay
```

## Determinism

A function is deterministic if it always returns the same result anytime it is called with a specific set of input values. For example, a function that adds two integers is deterministic because it will always return the same result when given the same two integers.

A function is non-deterministic if it can return different results each time it is called with a specific set of input values. For example, a function that returns the current date and time is non-deterministic because it will return different results each time it is called.

Determinism is a characteristic of a function which improves performance and is True when a function will return the same result given:
- same input parameters
- same database state (data, schema, etc)

A function's schemabinding option can be enabled to verify that teh database state won't change in a way that would affect its result.

Schema is a collection of database objects associated with one particular database owner like dbo.

Database objects are tables, columns, data types or other functions.

UDFs can reference database objects so if SCHEMABINDING option is enabled within the functio, SQL server will prevent changes to the database objects it references.

### Example

```sql
CREATE OR ALTER FUNCTION ...
RETURNS ... WITH SCHEMABINDING
AS BEGIN RETURN (
    SELECT ...
    FROM dbo.Table -- Must reference schema name when using SCHEMABINDING
    WHERE ...
)
END
```



