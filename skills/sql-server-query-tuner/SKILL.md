---
name: sql-server-query-tuner
description: >
  SQL Server query performance tuning and optimization. Analyzes execution plans,
  identifies anti-patterns, recommends indexes, rewrites slow queries, and diagnoses
  performance bottlenecks. Use when the user wants to: (1) tune or optimize a slow
  SQL Server query, (2) analyze an execution plan (XML or pasted), (3) find missing
  indexes or redundant indexes, (4) rewrite a query for better performance,
  (5) run DMV diagnostic queries against a live SQL Server instance, (6) understand
  why a query is slow, (7) review query statistics or wait stats, or any SQL Server
  performance-related request. Triggers on: "tune this query", "why is this slow",
  "optimize this SQL", "check execution plan", "missing indexes", "query performance",
  "SQL Server slow", "index recommendations", "rewrite this query".
---

# SQL Server Query Tuner

## Tuning Workflow

Query tuning follows this process:

1. **Collect** - Gather the query, execution plan, and context
2. **Diagnose** - Identify root causes (scans, spills, bad estimates, anti-patterns)
3. **Recommend** - Suggest indexes, rewrites, or configuration changes
4. **Deliver** - Present findings with tuned query and explanation

## Step 1: Collect

Determine input type:

**User pastes a query?**
- Ask for the execution plan if not provided (XML preferred, graphical description accepted)
- Ask for table schemas/row counts if not obvious from plan
- Ask which SQL Server version they're on if relevant

**User has a live SQL Server connection?**
- Use `sqlcmd` or the connection tool available to run DMV queries
- See [references/dmv-queries.md](references/dmv-queries.md) for ready-to-use diagnostic queries
- Collect: actual execution plan, table/index metadata, wait stats

**User pastes an XML execution plan?**
- Parse it directly - look for `RelOp`, `IndexScan`, `Sort`, `Hash Match`, estimated vs actual rows

## Step 2: Diagnose

Check for these issues in priority order:

1. **Table/index scans** on large tables (should be seeks)
2. **Estimated vs actual row** mismatches (bad cardinality estimates)
3. **Key lookups** with high execution count (missing covering index)
4. **Sort/Hash spills** to tempdb (memory grant issues)
5. **Implicit conversions** in predicates (type mismatches killing index usage)
6. **Parameter sniffing** indicators (plan reuse with bad estimates)
7. **Scalar UDF calls** in SELECT/WHERE (row-by-row execution)
8. **Cursor/RBAR patterns** (row-by-agonizing-row)

See [references/anti-patterns.md](references/anti-patterns.md) for the full anti-pattern catalog with examples and fixes.

See [references/execution-plan-guide.md](references/execution-plan-guide.md) for reading execution plan operators and warnings.

## Step 3: Recommend

For each issue found, provide:

- **What**: The specific problem identified
- **Why**: How it impacts performance
- **Fix**: The concrete change (index DDL, rewritten query, or config change)
- **Impact**: Expected improvement (high/medium/low)

### Index Recommendations

When suggesting indexes:
- Include the full `CREATE INDEX` statement with `INCLUDE` columns
- Check for existing indexes that overlap (consolidate, don't duplicate)
- Flag if a new index would be a superset of an existing one
- Consider write overhead - note if the table is write-heavy
- Suggest `DROP INDEX` for redundant indexes being replaced

### Query Rewrites

When rewriting queries:
- Show the original and rewritten side by side
- Explain what changed and why
- Preserve exact semantic equivalence (same results, same order if ORDER BY present)
- Common rewrites: correlated subquery to JOIN, scalar UDF to inline expression, CURSOR to set-based, OR to UNION ALL, functions on columns to SARGable form

## Step 4: Deliver

Present findings using this structure:

```
## Query Tuning Report

### Summary
[One-line: what was wrong and what to do about it]

### Issues Found
1. [Issue] - [Impact: High/Medium/Low]
   - Problem: [what's happening]
   - Fix: [what to change]

### Recommended Indexes
[CREATE INDEX statements, if any]

### Optimized Query
[Rewritten SQL, if applicable]

### Additional Notes
[Parameter sniffing concerns, statistics update suggestions, etc.]
```

## Live Server Diagnostics

When connected to a live SQL Server, use DMV queries from [references/dmv-queries.md](references/dmv-queries.md) to:

- Pull top expensive queries by CPU/reads/duration
- Check missing index DMVs
- Review wait statistics
- Examine index usage stats (seeks vs scans vs unused)
- Check for blocking and deadlocks

Always use `SET NOCOUNT ON` and `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED` for diagnostic queries to minimize impact.

## Key Principles

- Never recommend `NOLOCK` hints as a tuning fix (masks problems, risks dirty reads)
- Prefer SARGable predicates - no functions wrapping indexed columns
- Favor covering indexes over wide table scans
- Consider the full workload, not just one query in isolation
- UPDATE STATISTICS or REBUILD INDEX are maintenance, not query tuning - suggest them only when stale stats are the proven root cause
