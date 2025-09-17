# Temporal Exploratory Data Analysis

## Functiosn for EDA

- CONVERT()
- DATEPART()
- DATENAME()
- DATEDIFF()

## Variables for datetime data

1. 
```sql 
DECLARE @MyDate AS TIME = '08:00 AM'
```

2.
```sql
DECLARE @MyDate AS TIME
SET @MyDate = '08:00 AM'
```

3. 
```sql
DECLARE @BeginDate AS DATE
SET @BeginDate = (
    SELECT TOP 1 PickupDate
    FROM Table
    ORDER BY PickupDate ASC
)
```

## Combine date and time variable into one datetime variable

```sql
CAST(expression AS data_type[(length)]) -- Returns expression based on data type
```
```sql
DECLARE @StartDateTime AS DATETIME
SET @StartDateTime = CAST(@BeginDate AS DATETIME) + CAST(@StartTime AS DATETIME)
```

## Table variable

```sql
DECLARE @MyTableVar TABLE(
    StartDate DATE,
    EndDate DATE
)
INSERT INTO @Table (StartDate, EndDate)
SELECT '3/1/2018', '3/2/2018'
```
## Date manipulation

### Find yesterdays's date

```sql
DATEADD(d, -1, GETDATE())
```

### Find first day of this week

```sql
DATEADD(WEEK, DATEDIFF(WEEK, 0, GETDATE()), 0)
```
### Find first day of this month

```sql
DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()), 0)
```


