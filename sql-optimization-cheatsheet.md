# SQL Server Optimization Cheat Sheet

## Part 1: Identify Slow Queries (Finding the Problem)

### 1.1 Query Store (SQL Server 2016+)

```sql
-- Top 10 queries by total CPU time (recent)
SELECT TOP 10
    qt.query_sql_text,
    rs.avg_duration / 1000 AS avg_duration_ms,
    rs.avg_cpu_time / 1000 AS avg_cpu_ms,
    rs.avg_logical_io_reads,
    rs.count_executions,
    rs.last_execution_time
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
WHERE rs.last_execution_time > DATEADD(HOUR, -24, GETUTCDATE())
ORDER BY rs.avg_cpu_time DESC;
```

### 1.2 DMVs (Dynamic Management Views)

```sql
-- Top queries by total elapsed time (since last restart)
SELECT TOP 20
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(qt.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2)+1) AS query_text,
    qs.execution_count,
    qs.total_elapsed_time / 1000 AS total_elapsed_ms,
    qs.total_elapsed_time / qs.execution_count / 1000 AS avg_elapsed_ms,
    qs.total_logical_reads,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.creation_time,
    qs.last_execution_time
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_elapsed_time DESC;
```

### 1.3 Currently Running Long Queries

```sql
-- Queries running right now, sorted by duration
SELECT
    r.session_id,
    r.start_time,
    DATEDIFF(SECOND, r.start_time, GETDATE()) AS running_seconds,
    r.status,
    r.wait_type,
    r.blocking_session_id,
    t.text AS query_text,
    qp.query_plan
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) qp
WHERE r.session_id > 50
ORDER BY r.start_time ASC;
```

### 1.4 Missing Index Suggestions

```sql
-- Indexes SQL Server recommends based on workload
SELECT TOP 20
    ROUND(s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans), 0) AS improvement_measure,
    d.statement AS table_name,
    d.equality_columns,
    d.inequality_columns,
    d.included_columns,
    s.user_seeks,
    s.user_scans
FROM sys.dm_db_missing_index_groups g
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
JOIN sys.dm_db_missing_index_details d ON g.index_handle = d.index_handle
ORDER BY improvement_measure DESC;
```

### 1.5 Extended Events (Lightweight Tracing)

```sql
-- Create session to capture queries > 5 seconds
CREATE EVENT SESSION [SlowQueries] ON SERVER
ADD EVENT sqlserver.sql_statement_completed (
    ACTION (sqlserver.sql_text, sqlserver.database_name, sqlserver.username)
    WHERE duration > 5000000  -- microseconds = 5 seconds
)
ADD TARGET package0.event_file (SET filename = N'SlowQueries.xel')
WITH (MAX_MEMORY = 4096 KB, STARTUP_STATE = ON);

ALTER EVENT SESSION [SlowQueries] ON SERVER STATE = START;
```

### 1.6 AWS RDS/Aurora Specific

```sql
-- Performance Insights equivalent: enable Query Store
ALTER DATABASE [YourDB] SET QUERY_STORE = ON;
ALTER DATABASE [YourDB] SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    DATA_FLUSH_INTERVAL_SECONDS = 60,
    INTERVAL_LENGTH_MINUTES = 5,
    MAX_STORAGE_SIZE_MB = 1024
);
```

For AWS RDS, also check:
- **Performance Insights** in AWS Console (visual top SQL)
- **CloudWatch** metrics: `ReadLatency`, `CPUUtilization`, `DatabaseConnections`

---

## Part 2: Diagnose & Fix (Optimization Techniques)

### 2.1 Read the Execution Plan

```sql
-- Always check the ACTUAL execution plan, not estimated
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- Then run your query and read:
-- * Actual vs Estimated rows (big mismatch = bad stats or parameter sniffing)
-- * Scan vs Seek (scan on large table = missing index or non-SARGable)
-- * Key Lookup (means index doesn't cover all needed columns)
-- * Sort / Hash Match spills (tempdb pressure)
```

### 2.2 SARGability (Search ARGument Able)

Indexed columns must be "naked" on the left side of the comparison.

| Bad (causes scan) | Good (allows seek) |
|---|---|
| `WHERE YEAR(OrderDate) = 2024` | `WHERE OrderDate >= '2024-01-01' AND OrderDate < '2025-01-01'` |
| `WHERE CAST(ProductID AS VARCHAR) = @input` | `WHERE ProductID = CAST(@input AS INT)` |
| `WHERE FirstName + LastName = @name` | `WHERE FirstName = @first AND LastName = @last` |
| `WHERE Amount * 1.1 > 100` | `WHERE Amount > 100 / 1.1` |
| `WHERE ISNULL(Status, 'X') = 'Active'` | `WHERE Status = 'Active'` |

### 2.3 Covering Indexes

```sql
-- Problem: Index Seek + Key Lookup (expensive on many rows)
-- Query: SELECT OrderId, CustomerId, OrderDate, TotalAmount
--        FROM Orders WHERE CustomerId = @id

-- Fix: INCLUDE the columns the query selects
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId
ON Orders (CustomerId)
INCLUDE (OrderDate, TotalAmount);
```

### 2.4 Parameter Sniffing

```sql
-- Detect: same query, wildly different performance per parameter value
-- Check if plan was compiled for an atypical value:

SELECT
    qs.plan_handle,
    qs.execution_count,
    qs.min_elapsed_time / 1000 AS min_ms,
    qs.max_elapsed_time / 1000 AS max_ms,
    TRY_CAST(qp.query_plan AS XML).value(
        '(//ParameterList/ColumnReference/@ParameterCompiledValue)[1]', 'NVARCHAR(100)'
    ) AS sniffed_value
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE qp.query_plan LIKE '%YourTableOrProc%';

-- Fixes (use only when confirmed):
-- Option 1: OPTIMIZE FOR UNKNOWN (generic plan)
SELECT ... FROM Orders WHERE CustomerId = @id
OPTION (OPTIMIZE FOR UNKNOWN);

-- Option 2: RECOMPILE (fresh plan each time, use for infrequent queries)
OPTION (RECOMPILE);

-- Option 3: Plan Guide or Query Store forced plan
```

### 2.5 Statistics & Maintenance

```sql
-- Check stale statistics
SELECT
    OBJECT_NAME(s.object_id) AS table_name,
    s.name AS stat_name,
    sp.last_updated,
    sp.rows,
    sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE sp.modification_counter > 0.2 * sp.rows  -- >20% modified
ORDER BY sp.modification_counter DESC;

-- Update stats for a specific table
UPDATE STATISTICS Orders WITH FULLSCAN;
```

### 2.6 Common Anti-Patterns

| Anti-Pattern | Impact | Fix |
|---|---|---|
| `SELECT *` | Extra I/O, blocks covering index | List only needed columns |
| Scalar UDF in WHERE/SELECT | Row-by-row execution | Inline the logic or use inline TVF |
| Cursor / WHILE loop | RBAR (row by agonizing row) | Rewrite as set-based operation |
| `NOLOCK` everywhere | Dirty reads, incorrect results | Use READ COMMITTED SNAPSHOT |
| Implicit conversion | Index scan instead of seek | Match data types explicitly |
| `OR` on different columns | Optimizer can't seek | Split into UNION ALL |

---

## Quick Decision Flow

```
Query is slow
  |
  +--> Step 1: Find it (Part 1 above - DMVs, Query Store, Extended Events)
  |
  +--> Step 2: Get ACTUAL execution plan + SET STATISTICS IO ON
  |
  +--> Step 3: Check for these (in order):
         |
         +-- Table/Index Scan on large table? --> Missing index or non-SARGable filter
         |
         +-- Key Lookup with many rows? --> Add INCLUDE columns to index
         |
         +-- Huge row estimate mismatch? --> Stale stats or parameter sniffing
         |
         +-- Sort spilling to tempdb? --> Add index matching ORDER BY, or increase memory grant
         |
         +-- Parallelism issues? --> Check MAXDOP, cost threshold for parallelism
```
