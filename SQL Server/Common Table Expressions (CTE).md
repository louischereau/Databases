# Common Table Expressions (CTE)

A Common Table Expression (CTE) is a temporary result set that you can reference within a `SELECT`, `INSERT`, `UPDATE`, or `DELETE` statement. CTEs are defined using the `WITH` clause and can help improve the readability and organization of complex queries. 

Hence create a table that can later be used for querying.

```sql
WITH cteName (col1, col2)
AS
(
    SELECT col1, col2
    FROM someTable
    WHERE condition
)
```

More than 1 CTE can be defined in one WITH statement.

Used to:
- manage complicated queries
- can be used within SELECT, INSERT, UPDATE or DELETE statements
- combine several CTEs with UNION or JOIN
- substitute for a view (e.g when don't have to store definition in metadata)
- recursion query

## Guide to use recursive CTE

A recursive CTE is a CTE that references itself. It consists of two parts: the anchor member and the recursive member. The anchor member is executed first, and then the recursive member is executed repeatedly until it returns no more rows.

```sql
WITH cteName (col1, col2)
AS
(
    -- Anchor member (intitial query)
    SELECT col1, col2
    FROM someTable
    WHERE condition

    UNION ALL

    -- Recursive member (recursive query)
    SELECT col1, col2
    FROM someTable
    INNER JOIN cteName ON someTable.col = cteName.col
    WHERE condition
)
```

For more than 200 recursion setps (limit by default), increase the number of recursion steps using: OPTION(MAXRECURSION...)

Following statemenst are not allowed:
- GROUP BY
- HAVING
- LEFT JOIN
- RIGHT JOIN
- OUTER JOIN
- SELECT DISTINCT
- TOP
- Subqueries

Number of columns and data types for anchor member (before UNION ALL) and recursive member (after UNION ALL) must be the same.

### Get the hierarchy

```sql
WITH hierarchy AS (
  SELECT id, Supervisor
  FROM employee
  WHERE Supervisor = 0

  UNION ALL

  SELECT emp.id, emp.Supervisor
  FROM employee emp
  INNER JOIN hierarchy h ON emp.Supervisor = h.id
)
```

### Get the level of hierarchy
```sql
WITH hierarchy AS (
    SELECT id, Supervisor, 1 AS Level
    FROM employee
    WHERE Supervisor = 0
    
    UNION ALL
    
    SELECT emp.id, emp.Supervisor, h.Level + 1
    FROM employee emp
    INNER JOIN hierarchy h ON emp.Supervisor = h.id
)
```

### Combine results into 1 field

```sql
WITH hierarchy AS (
    SELECT id, Supervisor, CAST('0' AS VARCHAR(MAX)) AS HierarchyPath
    FROM employee
    WHERE Supervisor = 0
    
    UNION ALL
    
    SELECT emp.id, emp.Supervisor, CAST(h.HierarchyPath + ' -> ' + CAST(emp.Supervisor AS VARCHAR(MAX)) AS VARCHAR(MAX))
    FROM employee emp
    INNER JOIN hierarchy h ON emp.Supervisor = h.id
)
```

### Querying for all possible flights with cost limits

```sql
WITH flightRoute (Departure, Arrival, Stops, totalCost, route)
AS (
    SELECT f.Departure, f.Arrival, 0, Cost, CAST(Departure + '->' + Arrival AS NVARCHAR(MAX))
    FROM flightPlan f
    WHERE Origin = 'NYC' AND Cost <= 500

    UNION ALL

    SELECT p.Departure, f.Arrival, p.stops + 1, p.totalCOst + f.Cost, CAST(p.route + '->' + f.Arrival AS NVARCHAR(MAX)) 
    FROM flightPlan f, flightRoute p
    WHERE p.Arrival = f.Departure AND p.stops < 5
)
SELECT p.route
FROM flightRoute p
WHERE totalCost < 500
```



