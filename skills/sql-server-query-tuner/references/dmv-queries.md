# DMV Diagnostic Queries

Run these against a live SQL Server instance. Always prefix with:

```sql
SET NOCOUNT ON;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

## Table of Contents

- [Top Expensive Queries](#top-expensive-queries)
- [Missing Indexes](#missing-indexes)
- [Index Usage Stats](#index-usage-stats)
- [Redundant/Duplicate Indexes](#redundantduplicate-indexes)
- [Wait Statistics](#wait-statistics)
- [Blocking and Deadlocks](#blocking-and-deadlocks)
- [Statistics Freshness](#statistics-freshness)
- [TempDB Spills](#tempdb-spills)
- [Parameter Sniffing Detection](#parameter-sniffing-detection)
- [Get Execution Plan for a Query](#get-execution-plan-for-a-query)

---

## Top Expensive Queries

### By CPU

```sql
SELECT TOP 20
    qs.total_worker_time / qs.execution_count AS avg_cpu_us,
    qs.total_worker_time AS total_cpu_us,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text,
    qp.query_plan,
    DB_NAME(st.dbid) AS database_name
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_worker_time DESC;
```

### By Logical Reads

```sql
SELECT TOP 20
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    qs.total_logical_reads,
    qs.execution_count,
    qs.total_worker_time / qs.execution_count AS avg_cpu_us,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_logical_reads DESC;
```

### By Duration

```sql
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count AS avg_duration_us,
    qs.total_elapsed_time,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_elapsed_time DESC;
```

## Missing Indexes

```sql
SELECT TOP 30
    CONVERT(DECIMAL(18,2), migs.avg_total_user_cost * migs.avg_user_impact *
        (migs.user_seeks + migs.user_scans)) AS improvement_measure,
    mid.statement AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.user_seeks,
    migs.user_scans,
    migs.avg_total_user_cost,
    migs.avg_user_impact
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON mig.index_group_handle = migs.group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID()
ORDER BY improvement_measure DESC;
```

## Index Usage Stats

```sql
SELECT
    OBJECT_SCHEMA_NAME(i.object_id) + '.' + OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,
    ius.last_user_seek,
    ius.last_user_scan
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON i.object_id = ius.object_id AND i.index_id = ius.index_id
    AND ius.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
    AND i.name IS NOT NULL
ORDER BY ius.user_seeks + ius.user_scans + ISNULL(ius.user_lookups, 0) DESC;
```

### Unused Indexes (candidates for removal)

```sql
SELECT
    OBJECT_SCHEMA_NAME(i.object_id) + '.' + OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc,
    ius.user_updates,
    ius.system_seeks + ius.system_scans AS system_usage
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON i.object_id = ius.object_id AND i.index_id = ius.index_id
    AND ius.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
    AND i.type_desc = 'NONCLUSTERED'
    AND i.is_primary_key = 0
    AND i.is_unique_constraint = 0
    AND ISNULL(ius.user_seeks, 0) = 0
    AND ISNULL(ius.user_scans, 0) = 0
    AND ISNULL(ius.user_lookups, 0) = 0
ORDER BY ius.user_updates DESC;
```

## Redundant/Duplicate Indexes

```sql
;WITH IndexColumns AS (
    SELECT
        i.object_id, i.index_id, i.name AS index_name, i.type_desc,
        STUFF((
            SELECT ',' + COL_NAME(ic2.object_id, ic2.column_id)
            FROM sys.index_columns ic2
            WHERE ic2.object_id = i.object_id AND ic2.index_id = i.index_id
                AND ic2.is_included_column = 0
            ORDER BY ic2.key_ordinal
            FOR XML PATH('')
        ), 1, 1, '') AS key_columns,
        STUFF((
            SELECT ',' + COL_NAME(ic2.object_id, ic2.column_id)
            FROM sys.index_columns ic2
            WHERE ic2.object_id = i.object_id AND ic2.index_id = i.index_id
                AND ic2.is_included_column = 1
            ORDER BY ic2.column_id
            FOR XML PATH('')
        ), 1, 1, '') AS include_columns
    FROM sys.indexes i
    WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
        AND i.type_desc = 'NONCLUSTERED'
)
SELECT
    OBJECT_SCHEMA_NAME(a.object_id) + '.' + OBJECT_NAME(a.object_id) AS table_name,
    a.index_name AS index_a,
    a.key_columns AS keys_a,
    a.include_columns AS includes_a,
    b.index_name AS index_b,
    b.key_columns AS keys_b,
    b.include_columns AS includes_b
FROM IndexColumns a
JOIN IndexColumns b ON a.object_id = b.object_id
    AND a.index_id < b.index_id
    AND (a.key_columns = b.key_columns
         OR a.key_columns LIKE b.key_columns + ',%'
         OR b.key_columns LIKE a.key_columns + ',%');
```

## Wait Statistics

```sql
;WITH Waits AS (
    SELECT
        wait_type,
        wait_time_ms / 1000.0 AS wait_time_s,
        100.0 * wait_time_ms / SUM(wait_time_ms) OVER() AS pct,
        signal_wait_time_ms / 1000.0 AS signal_wait_time_s,
        waiting_tasks_count
    FROM sys.dm_os_wait_stats
    WHERE wait_type NOT IN (
        'CLR_SEMAPHORE','LAZYWRITER_SLEEP','RESOURCE_QUEUE','SQLTRACE_BUFFER_FLUSH',
        'SLEEP_TASK','SLEEP_SYSTEMTASK','WAITFOR','HADR_FILESTREAM_IOMGR_IOCOMPLETION',
        'CHECKPOINT_QUEUE','REQUEST_FOR_DEADLOCK_SEARCH','XE_TIMER_EVENT','XE_DISPATCH_WAIT',
        'BROKER_TO_FLUSH','BROKER_TASK_STOP','CLR_MANUAL_EVENT','CLR_AUTO_EVENT',
        'DISPATCHER_QUEUE_SEMAPHORE','FT_IFTS_SCHEDULER_IDLE_WAIT','LOGMGR_QUEUE',
        'ONDEMAND_TASK_QUEUE','SLEEP_BPOOL_FLUSH','DIRTY_PAGE_POLL','BROKER_EVENTHANDLER',
        'DBMIRROR_DBM_MUTEX','DBMIRROR_EVENTS_QUEUE','DBMIRRORING_CMD',
        'SQLTRACE_INCREMENTAL_FLUSH_SLEEP','SP_SERVER_DIAGNOSTICS_SLEEP',
        'QDS_PERSIST_TASK_MAIN_LOOP_SLEEP','QDS_ASYNC_QUEUE',
        'QDS_CLEANUP_STALE_QUERIES_TASK_MAIN_LOOP_SLEEP','WAIT_XTP_CKPT_CLOSE'
    )
    AND waiting_tasks_count > 0
)
SELECT TOP 15
    wait_type,
    CAST(wait_time_s AS DECIMAL(16,2)) AS wait_time_s,
    CAST(pct AS DECIMAL(5,2)) AS pct,
    CAST(signal_wait_time_s AS DECIMAL(16,2)) AS signal_wait_s,
    waiting_tasks_count
FROM Waits
ORDER BY wait_time_s DESC;
```

## Blocking and Deadlocks

### Current Blocking

```sql
SELECT
    r.session_id AS blocked_spid,
    r.blocking_session_id AS blocking_spid,
    r.wait_type,
    r.wait_time / 1000.0 AS wait_time_s,
    DB_NAME(r.database_id) AS database_name,
    st.text AS blocked_query,
    st2.text AS blocking_query
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) st
OUTER APPLY (
    SELECT sql_handle FROM sys.dm_exec_requests r2
    WHERE r2.session_id = r.blocking_session_id
) br
OUTER APPLY sys.dm_exec_sql_text(br.sql_handle) st2
WHERE r.blocking_session_id > 0;
```

## Statistics Freshness

```sql
SELECT
    OBJECT_SCHEMA_NAME(s.object_id) + '.' + OBJECT_NAME(s.object_id) AS table_name,
    s.name AS stat_name,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter,
    CAST(100.0 * sp.modification_counter / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS pct_modified
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
    AND sp.modification_counter > 0
ORDER BY sp.modification_counter DESC;
```

## TempDB Spills

Check recent queries that spilled to tempdb (SQL Server 2016+):

```sql
SELECT TOP 20
    qs.total_spills,
    qs.total_spills / qs.execution_count AS avg_spills,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE qs.total_spills > 0
ORDER BY qs.total_spills DESC;
```

## Parameter Sniffing Detection

Find queries with high variance in execution times:

```sql
SELECT
    qs.query_hash,
    COUNT(*) AS plan_count,
    MIN(qs.total_elapsed_time / qs.execution_count) AS min_avg_duration_us,
    MAX(qs.total_elapsed_time / qs.execution_count) AS max_avg_duration_us,
    MAX(qs.total_elapsed_time / qs.execution_count) * 1.0 /
        NULLIF(MIN(qs.total_elapsed_time / qs.execution_count), 0) AS variance_ratio,
    MIN(SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1)) AS sample_query
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
GROUP BY qs.query_hash
HAVING COUNT(*) > 1
ORDER BY variance_ratio DESC;
```

## Get Execution Plan for a Query

To capture the actual execution plan for a specific query:

```sql
SET STATISTICS XML ON;
-- paste query here
SET STATISTICS XML OFF;
```

Or for estimated plan only:

```sql
SET SHOWPLAN_XML ON;
GO
-- paste query here
GO
SET SHOWPLAN_XML OFF;
GO
```
