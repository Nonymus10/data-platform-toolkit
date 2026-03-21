---
name: trino-optimizer
description: Optimizes Trino query performance against Apache Iceberg tables for large-scale data platforms.
---

# Trino Query Optimization Rules

When writing or reviewing Trino queries against Apache Iceberg tables, you MUST follow these three strict rules:

## Rule 1: Always Filter by Partition Keys

Every query MUST include a filter on the table's partition columns (e.g., `dt`, `region`, `event_date`). Full table scans on Iceberg tables are never acceptable - they blow up compute costs and can destabilize the cluster. If no partition filter exists, reject the query and ask the author to add one.

```sql
-- BAD: full table scan
SELECT * FROM iceberg.tax.transactions WHERE amount > 1000;

-- GOOD: partition-pruned
SELECT * FROM iceberg.tax.transactions WHERE dt = '2026-03-21' AND amount > 1000;
```

## Rule 2: Never Use Cross Joins

Cross joins on large Iceberg tables will produce catastrophic data explosions. Always use explicit `JOIN ... ON` conditions. If a cross join is detected in a query, flag it immediately as a blocker - no exceptions.

```sql
-- BAD: cross join
SELECT * FROM table_a, table_b;

-- GOOD: explicit join
SELECT * FROM table_a a JOIN table_b b ON a.id = b.ref_id;
```

## Rule 3: Prefer Predicate Pushdown-Friendly Patterns

Structure `WHERE` clauses so that Iceberg's metadata filtering and Trino's predicate pushdown can eliminate files early. Avoid wrapping partition columns in functions (e.g., `CAST`, `DATE_FORMAT`, `SUBSTR`) - this defeats file pruning and forces full partition scans.

```sql
-- BAD: function on partition column defeats pushdown
SELECT * FROM iceberg.tax.invoices WHERE YEAR(dt) = 2026;

-- GOOD: direct comparison enables file pruning
SELECT * FROM iceberg.tax.invoices WHERE dt BETWEEN DATE '2026-01-01' AND DATE '2026-12-31';
```
