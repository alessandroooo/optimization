---
layout: post
title: "Oracle SQL performance"
date: 2025-12-25
categories: [SQL, Oracle, performance]
---
Lately I spent some time with performance optimization in Oracle SQL and here I would like to share my key preliminary takeaways.

### The problem

Given is a SELECT statement (possibly using common table expressions, i.e. the WITH clause) that should be replaced by another statement, which is logically equivalent, but has better performance. This leads to two questions: *What is performance?* and *What is logical equivalence?*

#### Logical equivalence
Two SELECTS are logically equivalent if they return the same result, issued against all possible data instantiations of the database. In real life there is no time to test against all possible data. Instead, we are forced to live with necessarily incomplete test data. Hence, the observed accordance of the result of both SELECTs is a necessary, but not sufficient requirement for logical equivalence.
#### Performance
The performance can be quantified using different metrics, e.g. observed runtime, memory, storage or network footprint. Often the different metrics go hand in hand, i.e. the faster query tends to have lower storage footprint, but there are corner cases. A slow query with low storage footprint might be preferred over a fast query with large storage footprint. In practice I found the metric TEMP_SPACE_ALLOCATED useful, in order to assess a query's performance. 

Here comes the list of points to consider for performance optimization:

### Filter early, add late

The entire SELECT statement can be imagined as a tree. The top-level SELECT statement is the root of the tree, while the row sources are the leaves. Within this tree, filters should be applied as early as possible. Also, data should be added as late as possible.

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

Both SELECT DISTINCT (instead of SELECT) and UNION (instead of UNION ALL) involve deduplication. The problem with deduplication is twofold. First, energy is spent in order to retrieve and process data, that is later thrown away. Secondly, energy is spent to throw the surplus data away.

### Use CROSS JOINS
CROSS JOINS should be used whenever they make sense. CROSS JOINS incentivize the use of early filters.

### Avoid functions in predicates

### Avoid joins to tables that are not needed.

### Avoid table repetitions

### Use bind variables

### Beware of hints

