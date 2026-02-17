# Execution Plan Reading Guide

## Table of Contents

- [Key Operators](#key-operators)
- [Warning Signs](#warning-signs)
- [Reading XML Plans](#reading-xml-plans)
- [Cardinality Estimates](#cardinality-estimates)
- [Cost Analysis](#cost-analysis)

---

## Key Operators

### Data Access

| Operator | Meaning | Good/Bad |
|----------|---------|----------|
| **Index Seek** | Targeted lookup using index B-tree | Good |
| **Index Scan** | Full scan of an index | Bad on large tables |
| **Table Scan** | Full scan of a heap (no clustered index) | Bad on large tables |
| **Clustered Index Scan** | Full scan of clustered index (= table scan) | Bad on large tables |
| **Key Lookup** | Bookmark lookup for columns not in the index | Bad if high count |
| **RID Lookup** | Bookmark lookup on a heap | Bad if high count |

### Joins

| Operator | When Used | Notes |
|----------|-----------|-------|
| **Nested Loops** | Small outer input, indexed inner | Best for small datasets with good indexes |
| **Hash Match** | Large unsorted inputs | Watch for memory grants and spills |
| **Merge Join** | Both inputs sorted on join key | Efficient for pre-sorted data |
| **Adaptive Join** (2017+) | Optimizer defers choice to runtime | Normal, indicates uncertainty |

### Aggregation & Sorting

| Operator | Notes |
|----------|-------|
| **Sort** | Expensive if large; watch for spills. Can be eliminated with matching index |
| **Hash Match (Aggregate)** | Used for GROUP BY on unsorted data |
| **Stream Aggregate** | Used when data is already sorted on GROUP BY columns |
| **Distinct Sort** | Combined sort + distinct elimination |

### Other Important Operators

| Operator | Notes |
|----------|-------|
| **Compute Scalar** | Usually harmless. Watch for UDF calls or implicit conversions |
| **Filter** | Residual predicate not pushed to access operator. Often indicates non-SARGable condition |
| **Parallelism (Gather/Repartition/Distribute)** | Exchange operators for parallel plans |
| **Spool (Lazy/Eager)** | Caches results for reuse. Eager spools can indicate Halloween Protection |
| **Table Spool** | Often appears with spills or suboptimal plans |

## Warning Signs

Look for these in execution plans (yellow triangle warnings):

| Warning | Meaning | Action |
|---------|---------|--------|
| **Missing Index** | Optimizer suggests an index | Evaluate the suggestion (often overly broad) |
| **No Join Predicate** | Accidental cross join | Add proper JOIN condition |
| **Implicit Conversion** | Type mismatch on predicate | Fix parameter or column types |
| **Sort Warning** | Sort spilled to tempdb | Add ORDER BY-matching index or increase memory grant |
| **Hash Warning** | Hash spilled to tempdb | Check cardinality estimates, increase memory |
| **Columns With No Statistics** | Missing stats on computed/filtered columns | CREATE STATISTICS |
| **Unmatched Indexes** | Filtered index not used | Check predicate compatibility |
| **ExcessiveGrant** (2017+) | Way more memory granted than needed | Indicates overestimation |
| **InsufficientGrant** (2017+) | Not enough memory granted | Indicates underestimation, causes spills |

## Reading XML Plans

When given an XML execution plan, look for these key elements:

### StmtSimple or StmtCond
The statement-level node. Check `StatementEstRows` vs actual rows.

### RelOp
Each operator. Key attributes:
- `PhysicalOp` - the actual operator (Index Seek, Hash Match, etc.)
- `EstimateRows` - optimizer's row estimate
- `ActualRows` (actual plans only) - real rows processed
- `EstimatedTotalSubtreeCost` - cost of this subtree
- `Parallel` - whether operator runs in parallel

### IndexScan / IndexSeek
- `Index Name` - which index is used
- `Lookup` attribute - if true, it's a key lookup
- `Ordered` - whether results come back in order

### Warnings element
Contains any warnings (implicit conversions, spills, no stats).

### ParameterList
Shows sniffed parameter values. Compare `ParameterCompiledValue` vs `ParameterRuntimeValue` for parameter sniffing.

### MemoryGrantInfo
- `SerialRequiredMemory` / `SerialDesiredMemory` - how much memory the query wanted
- `GrantedMemory` - how much it got
- `MaxUsedMemory` - how much it actually used (compare to granted for waste)

## Cardinality Estimates

The #1 source of bad plans. Check:

1. **Estimated vs Actual rows** on every operator
   - Ratio > 10x = significant issue
   - Overestimate: excessive memory grant, unnecessary hash/sort
   - Underestimate: spills, nested loops on large sets

2. **Common causes of bad estimates:**
   - Stale statistics
   - Ascending key columns (new data beyond histogram)
   - Correlated predicates (optimizer assumes independence)
   - Table variables (always estimated at 1 row, use temp tables instead)
   - Multi-statement TVFs (fixed estimate)
   - Complex expressions in predicates

3. **Fixes:**
   - UPDATE STATISTICS with FULLSCAN
   - Add filtered statistics for skewed data
   - Use OPTION(RECOMPILE) for variable-sensitive queries
   - Replace table variables with temp tables
   - Use inline TVFs instead of multi-statement

## Cost Analysis

- Cost numbers are **relative estimates**, not actual time
- Useful for comparing operators within the same plan
- **Fat pipes** (thick arrows) = high row estimates between operators
- Focus on operators with highest actual row counts and I/O
- A 99% cost operator is not necessarily slow - it's relative to the rest of the plan

### Useful metrics from actual plans
- `ActualRows` * `ActualExecutions` = total rows processed
- `ActualLogicalReads` on data access operators = I/O impact
- `ActualElapsedms` per operator (2016+) = wall clock time
