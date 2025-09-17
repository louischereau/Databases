# Database Views

In a Database, a view (non-materialized views) is a "virtual table" taht is not part of the physical schema.

A view is defined by a SQL query that selects data from one or more tables. When you query a view, the database executes the underlying SQL query and returns the result set as if it were a table.

The query is stored in memoery, not the data.

View data is aggregated from data in tables.

Can be queried like a relational database table.

No need to retype common queries or alter schemas.

Benefits of views:
- Does not take up storage (except for query)
- A form of access control (hide sensitive columns and restrict what user can see)
- Masks complexity of queries (useful for highly normalised schemas)

## Create Table

```sql
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

## View all views

```sql
SELECT * FROM information_schema.views;
```

## Granting and revoking access

```sql
GRANT SELECT, INSERT, DELETE ON view_name TO role_name, user_name
```

```sql
REVOKE SELECT, INSERT, DELETE ON view_name FROM role_name, user_name
```

## Updating a view

```sql
UPDATE view_name 
SET column1 = value1, column2 = value2, ...
WHERE condition;
```
Not all views are updatable: 
- View needs to be made of 1 table
- Cannot use a indow or aggregte function

## Inserting into a view

```sql
INSERT INTO view (col1, col2, col3 ...)
VALUES (val1, val2, val3 ...)
```

## Droping a view

```sql
DROP VIEW view_name CASCADE | RESTRICT;
```

- RESTRICT (default): returns an error if there are objects that depend on the view.

- CASCADE: drop views and any object that depends on the view.

## Redefining a view

```sql
CREATE OR REPLACE VIEW view_name AS new_query
```

- If a view with view_name exists, it is replaced.
- new_query must egenrate the same column names, order and data types as the old query.
- the column output may be different.
- new columns may be added at the end.

If these criterias can't be met, drop existing view and create a new one.

## Altering a view

```sql
ALTER VIEW [IF EXISTS] view_name ALTER [COLUMN] col_name SET DEFAULT;

ALTER VIEW [IF EXISTS] view_name ALTER [COLUMN] col_name DROP DEFAULT;

ALTER VIEW [IF EXISTS] view_name OWNER TO new_owner;

ALTER VIEW [IF EXISTS] view_name RENAME TO new_view_name;

ALTER VIEW [IF EXISTS] view_name SET SCHEMA new_schema;

ALTER VIEW [IF EXISTS] view_name SET ( );

ALTER VIEW [IF EXISTS] view_name RESET ( );
```

## Materialized views

Materialized views are physical copies of the data that are stored on disk. They are used to improve query performance by precomputing and storing the results of complex queries.

Materialized views can be refreshed periodically to ensure that they reflect the latest data from the underlying tables.

Summary:
- stores query results, not the query.
- querying a materialized view means accessing the stored query results (not running the query like a non-materialized view).
- refreshed or rematerialized when prompted or scheduled.

Use when:
- using queries with long execution times.
- when underlying results don't change often (views not frequently refreshed).
- useful in data warehouses because OLAP not write-intensive hence save of computational cost of frequent queries.

```sql
CREATE MATERIALIZED VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition
WITH [NO] DATA; -- NO DATA means the view is created but not populated with data
```

```sql
REFRESH MATERIALIZED VIEW view_name;
```

Materilized views often depend on other materialized views. Creates a dependency chain when refreshing views. Not the most efficient to refresh all views at the same time.

Tools for managing dependencies when refreshing include:
- Directed acyclic graphs (DAGs) to keep track of views.
- Pipeline scheduler tools (Apache Airflow, Luigi)




