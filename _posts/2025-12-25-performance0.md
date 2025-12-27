---
layout: post
title: "Oracle SQL performance"
date: 2025-12-25
categories: [SQL, Oracle, performance]
---
Lately I spent some time with performance optimization in an Oracle data warehouse and here I would like to share my key takeaways.

### The problem

Given is a SELECT statement (possibly using common table expressions, i.e. involving the WITH clause) that should be replaced by another statement, which is logically equivalent, but has better performance. This immediately leads to two questions, namely: *What is performance?* and *What is logical equivalence?*

#### Logical equivalence
Two SELECTS are logically equivalent if they return the same result, issued against all possible data instantiations of the database. In real life there is no time to test against all possible data. Instead, we are forced to live with necessarily incomplete test data. Hence, the observed accordance of the results of both SELECTs is a necessary, but not sufficient requirement for logical equivalence. Hence, it is not possible to assert the logical equivalence of two SELECTs.
#### Performance
The observed runtime of the same query can vary considerably, depending on database load. For the performance different metrics can be employed, e.g. observed runtime, memory, storage or network footprint. Often the different metrics go hand in hand, i.e. the faster query tends to have lower storage footprint, but there are corner cases. A slow query with low storage footprint might be preferred over a fast query with large storage footprint. I found the metric TEMP_SPACE_ALLOCATED useful, in order to assess the query plan performance, see the Oracle table <a href="https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_ACTIVE_SESS_HISTORY.html">DBA_HIST_ACTIVE_SESS_HISTORY</a> for further details. Beware of <a href="https://docs.oracle.com/en/database/oracle/oracle-database/19/vldbg/using-parallel.html">parallel execution</a> in Oracle which can increase the storage footprint.


Here comes the list of points to consider for performance optimization:

### Filter early, add late

The entire SELECT statement can be imagined as a tree. The top-level SELECT statement is the root of the tree, while the row sources or the underlying base tables are the leaves. Within this tree, filters should be applied as early as possible. Also, data (e.g. literal values) should be added as late as possible.

```sql 
--Bad
WITH t0 as (SELECT id, amount, '1' AS code FROM t0)
SELECT t0.id, t0.amount, t0.code FROM t0 INNER JOIN t1 on t0.id = t1.id WHERE t0.amount > 0

--Good
WITH t0 as (SELECT id, amount FROM t0 WHERE amount > 0)
SELECT t0.id, t0.amount, '1' AS code FROM t0 INNER JOIN t1 on t0.id = t1.id
```

### Project early

### Avoid SELECT DISTINCT and UNION

Both SELECT DISTINCT (instead of SELECT) and UNION (instead of UNION ALL) involve deduplication. The problem with deduplication is at least twofold. First, energy is spent in order to retrieve and process surplus data, that is later thrown away. Secondly, energy is spent to keep rows in memory, match rows and throw the surplus data away. The presence of SELECT DISTINCT and UNION is a manifestation of the **inability** to order or address exactly the rows which are needed and renders the query less comprehensible. If the avoidance of SELECT DISTINCT or UNION is out of reach, moving them closer to the row sources can be advantageous. The fundamental problem of using deduplication is that it **hides the necessity** to apply filters early.

### Use CROSS JOINS
CROSS JOINS should be used whenever they make sense. CROSS JOINS incentivize the use of early filters.

### Avoid functions in predicates

### Avoid joins to tables that are not needed.

### Avoid table repetitions

### Use bind variables

### Beware of hints

