# Understanding the Cost of Queries in PostgreSQL

This document provides an overview of how PostgreSQL calculates and uses costs to optimize query execution plans.

## Introduction to EXPLAIN

PostgreSQL's `EXPLAIN` command is a powerful tool for understanding query performance. It shows the execution plan that the query planner generates for a given SQL statement.

The execution plan details:
- How tables will be scanned (sequentially, via index, etc.)
- What join algorithms will be used
- The estimated cost of each operation

Understanding these plans is crucial for optimizing database performance.

## Cost Calculation and Units

### What Are Costs?

The costs shown in `EXPLAIN` output are measured in an **arbitrary unit**, not in time (milliseconds). These costs represent the planner's estimate of how much work a query will require.

### Cost Components

PostgreSQL calculates costs based on several factors:

1. **Sequential Page Cost (`seq_page_cost`)**: The cost of reading a single page sequentially, which defaults to 1.0 units.
2. **Random Page Cost (`random_page_cost`)**: The cost of reading a single page randomly, which defaults to 4.0 units (reflecting slower random access on traditional HDDs).
3. **CPU Tuple Cost (`cpu_tuple_cost`)**: The CPU cost of processing each row, which defaults to 0.01 units.
4. **CPU Operator Cost (`cpu_operator_cost`)**: The CPU cost of processing each operator or function call, which defaults to 0.0025 units.

### Startup vs Total Cost

Each operation in the plan shows two cost values:
- `cost=0.00..17.00` means:
  - **Startup Cost**: 0.00 (estimated cost to fetch the first row)
  - **Total Cost**: 17.00 (estimated cost to complete the operation)

### How Costs Are Calculated

For a sequential scan:
```
Total cost = (estimated sequential page reads * seq_page_cost) + 
             (estimated rows returned * cpu_tuple_cost)
```

For example, if a table has 7 pages and 1000 rows:
```
Total cost = (7 * 1) + (1000 * 0.01) = 7 + 10 = 17
```

## Scan Methods and When They Are Chosen

PostgreSQL has several scan methods for retrieving data from tables:

### Sequential Scan (Seq Scan)

- Reads every row in a table
- Chosen for small tables or when a large percentage of rows need to be accessed
- Most efficient when retrieving many rows because it minimizes random I/O

### Index Scan

- Uses an index to find specific rows
- Efficient for retrieving a small number of rows based on indexed columns
- Involves additional I/O to fetch the actual row data from the table heap
- Best when the query is highly selective

### Bitmap Heap Scan

- Hybrid approach between Seq Scan and Index Scan
- First, an index is scanned to create a bitmap of matching row locations
- Then, the table heap is scanned in physical storage order, visiting only the pages indicated by the bitmap
- Reduces random I/O compared to a regular Index Scan when more than a few rows match
- Useful for medium selectivity queries

### When Each Scan Is Chosen

The planner chooses a scan method based on:
1. Table size
2. Available indexes
3. Estimated selectivity of the query (how many rows match the conditions)
4. Statistics about the data distribution

As demonstrated in the workshop exercise:
- For high-frequency values (like 'Sales' with 10,000 rows), a Seq Scan is often chosen
- For low-frequency values (like 'Management' with 1 row), an Index Scan is more efficient

## Join Decision Making

PostgreSQL uses several join algorithms, and the choice depends on various factors:

### Nested Loop Join

- For each row in the outer table, scan the inner table
- Works well when the outer table is small
- Can use indexes for the inner table scan
- Simple but can be expensive if both tables are large

### Hash Join

- Creates a hash table from one table, then scans the other table and probes the hash table
- Efficient for large equi-joins (joins with equality conditions)
- Requires enough memory to hold the hash table
- Often the best choice for large tables with suitable join conditions

### Merge Join

- Both tables are sorted on the join key, then merged together
- Efficient when the data is already sorted or when sorting is cheaper than other options
- Works well with ORDER BY clauses
- Good for range conditions

### Join Selection Factors

The decision tree for join selection considers:
1. Whether the join condition is an equi-join or range on sorted columns
2. Size of the tables after WHERE clause filtering
3. Availability of suitable indexes
4. Available memory for hash operations
5. Whether the data is already sorted
6. Whether it's an outer join (which requires special handling)

## Importance of Statistics

### What Are Statistics?

Statistics are metadata about the contents of the database that help the planner make informed decisions. They include:
- Number of rows in each table
- Number of distinct values in each column
- Most common values in each column
- Histogram of value distributions

### How Statistics Are Gathered

Statistics are collected by the `ANALYZE` command:
- Can be run manually
- Automatically run by the autovacuum daemon
- Stored in the `pg_statistic` system catalog

### Why Statistics Matter

Accurate statistics are crucial for query optimization:
- Without them, the planner makes guesses that may be wrong
- Outdated statistics can lead to suboptimal plan choices
- Skewed data distributions (like in our workshop exercise) require accurate statistics for the planner to make the right choice

As shown in the workshop:
- Before `ANALYZE`, all departments used the same plan type
- After `ANALYZE`, the planner correctly chose different plans based on row counts

## Using EXPLAIN and EXPLAIN ANALYZE for Optimization

### EXPLAIN vs EXPLAIN ANALYZE

- `EXPLAIN`: Shows the planner's estimated costs and chosen plan without executing the query
- `EXPLAIN ANALYZE`: Executes the query and shows actual runtimes and row counts along with the plan

### Reading EXPLAIN Output

Key information in EXPLAIN output:
- Operation type (Seq Scan, Index Scan, etc.)
- Cost estimates (startup and total)
- Estimated number of rows
- Width of rows (average size in bytes)
- Actual execution time and row counts (in EXPLAIN ANALYZE)

### Identifying Performance Issues

Compare estimated vs actual values:
- If estimated rows are very different from actual rows, statistics may be outdated
- If actual execution time is much higher than estimated cost, there may be I/O or locking issues
- Look for unexpectedly high-cost operations

### Forcing Plan Choices

Parameters like `enable_seqscan`, `enable_bitmapscan`, etc., can be used to disable specific scan types:
- Useful for testing alternative plans
- Should not be used in production as a permanent solution
- Shows the cost of alternative approaches

## Conclusion and Best Practices

Understanding PostgreSQL's cost model is key to optimizing query performance:

1. **Keep Statistics Current**: Run `ANALYZE` regularly, especially after large data changes
2. **Tune Cost Constants**: Adjust `random_page_cost` and others based on your hardware (SSDs vs HDDs)
3. **Use EXPLAIN ANALYZE**: Regularly check actual performance vs estimates
4. **Understand Your Data**: Know the distribution of values in your tables
5. **Monitor Plan Changes**: Be aware of when the planner might choose different plans

By mastering these concepts, you can write more efficient queries and better understand PostgreSQL's behavior.