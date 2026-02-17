# SQL Server Query Anti-Patterns

## Table of Contents

- [Non-SARGable Predicates](#non-sargable-predicates)
- [Implicit Conversions](#implicit-conversions)
- [Scalar UDFs in Queries](#scalar-udfs-in-queries)
- [Correlated Subqueries](#correlated-subqueries)
- [OR in WHERE Clause](#or-in-where-clause)
- [SELECT *](#select-)
- [Cursors and RBAR](#cursors-and-rbar)
- [NOLOCK Everywhere](#nolock-everywhere)
- [Missing JOIN Predicates](#missing-join-predicates)
- [Row-Goal Issues (TOP with JOINs)](#row-goal-issues)
- [Multi-Statement TVFs](#multi-statement-tvfs)
- [LIKE with Leading Wildcard](#like-with-leading-wildcard)
- [EXISTS vs COUNT for Existence Checks](#exists-vs-count)
- [Unnecessary DISTINCT](#unnecessary-distinct)
- [Data Type Mismatches in JOINs](#data-type-mismatches-in-joins)

---

## Non-SARGable Predicates

Functions on indexed columns prevent index seeks.

**Bad:**
```sql
WHERE YEAR(OrderDate) = 2024
WHERE ISNULL(Status, 'Active') = 'Active'
WHERE CONVERT(VARCHAR, CreatedDate, 112) = '20240101'
WHERE LEFT(CustomerName, 3) = 'ABC'
WHERE Amount + Tax > 1000
```

**Good:**
```sql
WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'
WHERE (Status = 'Active' OR Status IS NULL)
WHERE CreatedDate >= '2024-01-01' AND CreatedDate < '2024-01-02'
WHERE CustomerName LIKE 'ABC%'
WHERE Amount > 1000 - Tax  -- or redesign schema
```

## Implicit Conversions

Type mismatches force SQL Server to convert every row, preventing index seeks.

**Detect:** Look for `CONVERT_IMPLICIT` in execution plans or warnings.

**Common cases:**
```sql
-- Column is VARCHAR, parameter is NVARCHAR (very common with ORMs)
WHERE VarcharColumn = N'some value'  -- causes scan

-- Column is INT, compared to VARCHAR
WHERE IntColumn = '123'

-- Date column compared to string
WHERE DateColumn = '2024-01-01'
```

**Fix:** Match the parameter type to the column type exactly. In application code, set `SqlParameter.SqlDbType` explicitly.

## Scalar UDFs in Queries

Scalar UDFs before SQL Server 2019 execute row-by-row, preventing parallelism.

**Bad:**
```sql
SELECT dbo.fn_GetCustomerName(CustomerID), Amount
FROM Orders
WHERE dbo.fn_IsActive(StatusCode) = 1
```

**Good:**
```sql
-- Inline the logic
SELECT c.CustomerName, o.Amount
FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE o.StatusCode IN ('A', 'P')

-- Or use inline TVF instead of scalar UDF
```

**Note:** SQL Server 2019+ has Scalar UDF Inlining for eligible functions (`SELECT OBJECTPROPERTYEX(OBJECT_ID('fn_name'), 'IsInlineable')`).

## Correlated Subqueries

Execute once per outer row. Replace with JOINs or window functions.

**Bad:**
```sql
SELECT o.OrderID, o.Amount,
    (SELECT MAX(OrderDate) FROM Orders o2 WHERE o2.CustomerID = o.CustomerID) AS LastOrder
FROM Orders o
```

**Good:**
```sql
SELECT o.OrderID, o.Amount,
    MAX(o.OrderDate) OVER (PARTITION BY o.CustomerID) AS LastOrder
FROM Orders o
```

## OR in WHERE Clause

OR can prevent index seeks and force scans.

**Bad:**
```sql
WHERE CustomerID = @CustID OR @CustID IS NULL  -- "catch-all" query
WHERE Status = 'A' OR Priority > 5              -- different columns
```

**Good:**
```sql
-- Catch-all: use OPTION(RECOMPILE) or dynamic SQL
WHERE CustomerID = @CustID OPTION(RECOMPILE)

-- Different columns: UNION ALL
SELECT * FROM Orders WHERE Status = 'A'
UNION ALL
SELECT * FROM Orders WHERE Priority > 5 AND Status <> 'A'
```

## SELECT *

Fetches all columns, preventing covering index usage and wasting I/O.

**Bad:**
```sql
SELECT * FROM Orders WHERE CustomerID = 123
```

**Good:**
```sql
SELECT OrderID, OrderDate, Amount FROM Orders WHERE CustomerID = 123
-- Now a covering index on (CustomerID) INCLUDE (OrderID, OrderDate, Amount) works
```

## Cursors and RBAR

Row-By-Agonizing-Row processing. Almost always replaceable with set-based operations.

**Bad:**
```sql
DECLARE cur CURSOR FOR SELECT OrderID FROM Orders
OPEN cur
FETCH NEXT FROM cur INTO @OrderID
WHILE @@FETCH_STATUS = 0
BEGIN
    UPDATE Orders SET Status = 'Processed' WHERE OrderID = @OrderID
    FETCH NEXT FROM cur INTO @OrderID
END
```

**Good:**
```sql
UPDATE Orders SET Status = 'Processed'
-- Add WHERE clause as needed
```

For genuinely iterative logic, consider `WHILE` loops with batch sizes or use window functions.

## NOLOCK Everywhere

`WITH (NOLOCK)` / `READ UNCOMMITTED` is not a tuning fix. It can read:
- Uncommitted data (dirty reads)
- Duplicate rows or skip rows (during page splits)
- Data from rolled-back transactions

Only acceptable for rough estimates on reporting tables that never get updated.

## Missing JOIN Predicates

Accidental cross joins when a JOIN condition is missing.

**Detect:** Row count in execution plan is product of two table sizes.

**Fix:** Ensure every JOIN has a proper ON clause. Review multi-table queries carefully.

## Row-Goal Issues

`TOP`, `EXISTS`, or `FAST n` hint can cause optimizer to underestimate work.

**Detect:** Estimated rows much lower than actual in nested loop joins under a TOP.

**Fix:**
```sql
-- Disable row goal for specific query
SELECT TOP 10 ...
OPTION (USE HINT('DISABLE_OPTIMIZER_ROWGOAL'))
```

## Multi-Statement TVFs

Multi-statement table-valued functions are estimated at 1 row (pre-2017) or 100 rows (2017+), causing terrible plans.

**Fix:** Replace with inline TVFs (single SELECT statement) which get inlined into the query plan.

## LIKE with Leading Wildcard

```sql
WHERE Name LIKE '%smith'  -- cannot use index
```

**Fix options:**
- Reverse the column and search: `WHERE ReverseName LIKE 'htims%'` with a computed column
- Full-text search
- Redesign the data model if this is a frequent search pattern

## EXISTS vs COUNT

**Bad:**
```sql
IF (SELECT COUNT(*) FROM Orders WHERE CustomerID = 123) > 0
```

**Good:**
```sql
IF EXISTS (SELECT 1 FROM Orders WHERE CustomerID = 123)
-- Stops at first match instead of counting all rows
```

## Unnecessary DISTINCT

Often masks a bad JOIN that produces duplicates. Fix the JOIN instead.

**Bad:**
```sql
SELECT DISTINCT c.CustomerName
FROM Customers c
JOIN Orders o ON c.CustomerID = o.CustomerID
-- DISTINCT hides the 1-to-many duplication
```

**Good:**
```sql
SELECT c.CustomerName
FROM Customers c
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerID = c.CustomerID)
```

## Data Type Mismatches in JOINs

Joining columns with different types causes implicit conversions on every row.

**Detect:** Look for `CONVERT_IMPLICIT` on JOIN operators in execution plan.

**Fix:** Ensure JOIN columns have identical data types, or explicitly CAST the smaller side.
