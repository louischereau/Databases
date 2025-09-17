# Query Optimization

## Logical Processing Order

Below is order database processes syntax in a query:

1. FROM       |
2. ON         |
3. JOIN       | Concerned with finding, merging, aggregating, and filtering data.
4. WHERE      |
5. GROUP BY   |
6. HAVING     |

7. SELECT

8. DISTINCT   |
9. ORDER BY   | Concerned with actions on the final data exxtrcated. Quite expensive on database processing resources.
10. TOP       |

## Filtering

### Filtering with WHERE

```sql
SELECT PlayerName, Team, (DRebound, ORebound) AS TotalRebounds 
FROM PlayerStats
WHERE (DRebound + ORebound) >= 1000 -- WHERE will run calculation on every row to cheeck if row meets condition or not. Increase query times.
ORDER BY TotalRebounds DESC;
```

```sql
SELECT PlayerName, Team, TotalRebounds
-- Use subquery to reduce query time
FROM (
    SELECT PlayerName, Team, (DRebound + ORebound) AS TotalRebounds 
    FROM PlayerStats
    WHERE (DRebound + ORebound) >= 1000
    ORDER BY TotalRebounds DESC;
) 
```

```sql
SELECT PlayerName, College, DraftYear
FROM PLayers
WHERE UPPER(LEFT(College, 7)) = 'GEORGIA'; -- Applying complex / multiple functions to a column in the WHERE filter condition increase query time.
```

### Filtering with HAVING

WHERE => filter individual rows
HAVING => filter aggregated / grouped rows

HAVING generaly used to apply a numeric aggregate filter on grouped rows.

```sql
SELECT Team, 
        SUM(DRebound + ORebound) AS TotRebounds,
        SUM(DRebound) AS TotDef,
        SUM(ORebound) AS TotOff
FROM PlayerStats
WHERE ORebound >= 1000 -- Query will not return all rebound stats by Team where team offensive rebounds are 1000 àr more because the WHERE clause filters for individual players not team.
GROUP BY Team;
```

```sql
SELECT Team, 
        SUM(DRebound + ORebound) AS TotRebounds,
        SUM(DRebound) AS TotDef,
        SUM(ORebound) AS TotOff
FROM PlayerStats
GROUP BY Team
HAVING ORebound > 1000; -- Will raise an error because column not xontained in an agrregate function. Need to apply HAVING filter to a numeric column using an aggregate function.
```

```sql
SELECT Team, 
        SUM(DRebound + ORebound) AS TotRebounds,
        SUM(DRebound) AS TotDef,
        SUM(ORebound) AS TotOff
FROM PlayerStats
GROUP BY Team
HAVING SUM(Points) >= 1000 -- HAVING will run calculation only on grouped rows. SUM was misssing aggregate function required.
```

## Interrogation after SELECT

SELECT * can be bad for performance.

A query should select only the rows it needs.

SELECT * IN JOINs returns duplicates from joining columns.

| Row Limiter   | Database         |
|---------------|------------------|
| TOP           | SQL Server       |
| ROWNUM        | Oracle           |
| LIMIT         | PostgreSQL       |

ORDER BY slows performance.

## Managing duplicates

Following funtions are used to remove duplicates:
- DISTINCT()
- GROUP BY
- UNION (appends rows from one or more tables and removes duplicates)

DISTINCT and UNION may use an internal sort mechanism to order and check for duplicates, potentially increasing the time it takes for a query to run.

Before using DISTINCT, we should ask:
- is there an alternative method? (using a table with a unique key instead) 
- is the query using an aggregate function? (then use GROUP BY)

Before using UNION, we should ask:
- are duplicate rows ok?                             | Use UNION ALL which does not make use of internal sort.
- will appending queries produce duplicate rows?     |

## Subqueries

### Subquery with FROM

```sql
SELECT OrderId, CustomerId, NumDays
-- subquery acts as virtual table
FROM (
    SELECT *, DATEDIFF(DAY, OrderDate, ShippedDate) AS NumDays
    FROM Orders
) AS o
WHERE NumDays >= 35;
```
### Subquery with WHERE

```sql
SELECT CustomerId, CompanyName
FROM Customers
-- Filter based on data of another table
WHERE CustomerID IN (
    SELECT CustomerId
    FROM Orders
    WHERE Freight > 800
);
```

### Subquery with SELECT

```sql
SELECT CustomerId, CompanyName,
    -- to derive a new column
    (SELECT AVG(Freight) 
     FROM Orders o
     WHERE c.CustomerID = o.CustomerID) AS AvgFreight
FROM Customers c;
```
### Types of subqueries

#### Uncorrelated

Subquery does not contain a reference to the outer query and therefore can run independently from outer query. Subquery executes only once and returns results to outer query. Used with WHERE and FROM.

```sql
SELECT CustomerId, CustomerName, (
    SELECT AVG(Freight)
    FROM Orders o
    WHERE c.CustomerID = o.CustomerID
) AS AvgFreight
FROM Customers c;
```

#### Correlated

Subquery contains a reference to the outer query. Subquery cannot run independently of teh outer query. Subquery executes for each row in the outer query. Used with WHERE and SELECT. Involves high execution times / low performance. Can replicate results with INNER JOIN.

```sql
SELECT CustomerId, CustomerId, AVG(Freight)
FROM Customers c
INNER JOIN Orders o
ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerId, c.CustomerName;
```

## Presence and Absence

#### INTERSECT / EXCEPT

##### Check if data in one table is also present in another table

```sql
SELECT CustomerId
FROM Customers

INTERSECT

SELECT CustomerId
FROM Orders;
```

##### Check for absence of data in one table that are present in another table

```sql
SELECT CustomerId
FROM Customers

EXCEPT

SELECT CustomerId
FROM Orders;
```

INTERSECT and EXCEPT remove duplicates.

The number and order of columns in the SELECT statement must be the same between queries.

#### EXISTS / NOT EXISTS

EXISTS filters the outer query when there is a watch of data between teh outer query and asubquery.

NOT EXISTS does the contrary.

```sql
SELECT CustomerId, CompanyName, ContactName
FROM Customers c
WHERE EXISTS (
    SELECT 1 -- A 1 (or True) is returned if CustomerId from c matches CustomerId from o and teh c table is filtered on the match.
    FROM Orders o
    WHERE c.CustomerID = o.CustomerID
);
```

```sql
SELECT CustomerId, CompanyName, ContactName
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM Orders o
    WHERE c.CustomerID = o.CustomerID
);
```

#### IN / NOT IN

```sql
SELECT CustomerId, CompanyName, ContactName
FROM Customers
WHERE CustomerID IN (
    SELECT CustomerID
    FROM Orders
);
```

NOT IN returns no results if the columns in the subquery contain NULL values.

##### EXISTS vs IN

- EXISTS will stop searching the subquery when the condition is TRUE.
- IN collects all teh resukts from a subquery before passing to the other query.
- Consider using EXISTS instead of IN with a subquery.

Advantage of EXISTS / NOT EXISTS and IN / NOT IN: Reuslts can contain any column from the outer query and in any order contrary to INTERSECT / EXCEPT.

#### JOINs

- INNER JOIN => Check for presence
- LEFT OUTER JOIN => Check for absence

##### Exclusive LEFT OUTER JOIN

Exclusive LEFT OUTER JOIN to check the presence of data in one table that is absent in another table. Returns only rows in hte left query (outer query) that are not present in the right query.

```sql
SELECT c.CustomerId, c.CompanyName, o.OrderID
FROM Customers c
LEFT OUTER JOIN Orders o
ON c.CustomerID = o.CustomerID
WHERE o.CustomerID IS NULL; -- Allows restriction of filter condition to rows that do not match
```

##### Inclusive LEFT OUTER JOIN

Inclusive LEFT OUTER JOIN returns all the rows in the left (outer) query.

- Advantage: Results can contain any column from all joined queries in any order, contrary to EXISTS / NOT EXISTS and IN / NOT IN whose results are restricted to columns from outer query only.

- Disadvantage: Requirement to add the IS NULL WHERE filter condition with the exclusive LEFT OUTER JOIN.

## Time Statistics

Measure running times of queries.

- CPU time: time taken by server processors to process the query.
- Elapsed time: Total duration of the query. Includes time taken to process the query and waiting time for resources.

To get time statistics, we first need to run SET STATISTICS TIME ON before executing our query:

```sql
SET STATISTICS TIME ON;
SET STATISTICS TIME OFF;
```

| Elapsed Time                                | CPU time                                                  |
|---------------------------------------------|-----------------------------------------------------------|
|  May be variable when analysing query time  | Should vary little when analyzing query time statisctis   |
|  statistics (sensitive to several factors)  |                                                           |
|                                             | May not be a useful measure server if server processors   |
|  Best time statistics measure for the       | are running in parallel                                   |
|  fastest runnign query                      |                                                           | 


## Page Read Statistics

Measure amount of disk / memory activity generated by queries.

- All data in either memory or disk is stored in 8 KB size "pages".
- One page can store many rows or one value could span multiple pages.
- A page can only belong to one table.
- SQL Server works with pages coached.
- If a page is not coached in memory, it is read from disk and coached in memory.

To get page read statistics, we first need to run SET STATISTICS IO ON before executing our query:

```sql
SET STATISTICS IO ON;
SET STATISTICS IO OFF;
```

Logical reads measure of the number of 8 KB pages read from memory to process and return the results of the query.

## Indexes

An index:
- is a structure to improve speed of accessing data from a table.
- used to locate data quickly without having to scan entire table.
- useful for improving performance of queries with filter conditions.
- applied to table columns.
- typically added by a databse administrator.

### Clustered Index

- Analogy: dictionary (words stored alphabetically)
- Table data pages are ordered by the columns with the index.
- Only one clustered index per table (data ordered one way).
- Speeds up search operation by reducing number of data page reads by query.

### Non-clustered index

- Analogy: index at the back of a book (index entries point to page numbers)
- Structure contains an ordered layer of index pointers to unordered table data pages.
- A table can contain more than one non-clustered index.
- Improves INSERT and UPDATE operations.

## Execution Plans

Provide information on:
- whether indexes were used.
- types of joins used.
- location and relative costs of filter conditions, sorting, aggregations.

Each icon in an execution plan represents an operator that is used to perform a specific task.

Hovering over an operator provides detailed statistics of the task used in the query.

Execution plans are read from right to left.

The width of each arrow reflects how much data was passed from one operator to the next.


