---
layout: post
title: "Oracle SQL performance"
date: 2025-12-25
categories: [SQL, Oracle, performance]
---
Lately I spent some time with performance optimization for a data warehouse using Oracle database and here I would like to share my key takeaways. Most suggestions do not involve data modelling and can be implemented with minimum effort. As always, the better the code, the more difficult are improvements.

### The problem

Given is a SELECT statement (possibly using common table expressions, i.e. involving the WITH clause) that should be replaced by another statement, which is logically equivalent, but has better performance. This immediately leads to two questions, namely: *What is performance?* and *What is logical equivalence?*

#### Logical equivalence
Two SELECTS are logically equivalent if they return the same result, issued against all possible data instantiations of the database. In real life there is no time to test against all possible data. Instead, we are forced to live with necessarily incomplete test data. Hence, the observed accordance of the results of both SELECTs is a necessary, but not sufficient requirement for logical equivalence. Hence, it is not possible to assert the logical equivalence of two SELECTs. This is not a problem in practice, as the current state should never be regarded as a holy grail, and the ability to adapt to changing business requirements is welcome. Rather, emphasis should be placed on test automation and the ability to compare result sets quickly, possibly using clients like SQLPLUS or SQLcl.
#### Performance
The observed runtime of the same query can vary considerably, depending on database load. For the performance, different metrics can be employed, e.g. observed runtime, memory, storage or network footprint. Often the different metrics go hand in hand, i.e. the faster query tends to have lower storage footprint, but there are corner cases. A slow query with low storage footprint might be preferred over a fast query with large storage footprint. I found the metric TEMP_SPACE_ALLOCATED useful, in order to assess the query plan performance, see the Oracle table <a href="https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DBA_HIST_ACTIVE_SESS_HISTORY.html">DBA_HIST_ACTIVE_SESS_HISTORY</a> for further details. Beware of <a href="https://docs.oracle.com/en/database/oracle/oracle-database/19/vldbg/using-parallel.html">parallel execution</a> in Oracle which can increase the storage footprint.

Here comes a list of suggestions to consider for performance optimization:

#### Filter early, add late

This is the most important principle. The entire SELECT statement can be imagined as a tree. The top-level SELECT statement, i.e. the final result, is the root of the tree, while the row sources or the underlying base tables are the leaves. Within this tree, filters should be applied as early as possible. Also, data (e.g. setting literal values or joining additional data) should be added as late as possible.

```sql 
-- Bad
WITH t1 as (SELECT id, amount, '1' AS code FROM t0)
SELECT t1.id, t1.amount, t1.code, t2.b FROM t1 INNER JOIN t2 on t1.id = t2.id WHERE t1.amount > 0

-- Good
WITH t1 as (SELECT id, amount FROM t0 WHERE amount > 0)
SELECT t1.id, t1.amount, '1' AS code, t2.b FROM t1 INNER JOIN t2 on t1.id = t2.id
```

#### Project early
Columns which are not needed downstream should not be selected in the first place.

```sql 
-- Bad
WITH t1 as (SELECT a, b, c, d FROM t0)
SELECT a, b, c FROM t1

-- Good
WITH t1 as (SELECT a, b, c FROM t0)
SELECT a, b, c FROM t1
```

#### Avoid SELECT DISTINCT and UNION

Both SELECT DISTINCT (instead of SELECT) and UNION (instead of UNION ALL) involve deduplication. The problem with deduplication is twofold. First, energy is spent in order to retrieve and process surplus data, that is later thrown away. Secondly, energy is spent to keep rows in memory, match rows and throw the surplus data away. The presence of SELECT DISTINCT and UNION is a manifestation of the **inability** or **failure** to retrieve the desired rows and renders the query less comprehensible and less maintanable. If the avoidance of SELECT DISTINCT or UNION is out of reach, moving them closer to the row sources can be advantageous. One problem of using deduplication is that it **hides the necessity** to apply early filters.

#### Use CROSS JOINs
CROSS JOINs should be used whenever they make sense and are preferred to JOINs with trivial predicates, which always evaluate to TRUE. The advantage of CROSS JOINs is that they incentivize the use of early filters.
```sql 
-- Bad
WITH grid AS (SELECT 1 AS prefix FROM dual UNION ALL SELECT 2 AS prefix FROM dual)
SELECT grid.prefix || t0.id AS id FROM grid LEFT JOIN t0 ON 1=1

-- Good
WITH grid AS (SELECT 1 AS prefix FROM dual UNION ALL SELECT 2 AS prefix FROM dual)
SELECT grid.prefix || t0.id AS id FROM grid CROSS JOIN t0
```

#### Avoid functions in predicates
Functions in JOIN predicates should be avoided. A possible exception is the presence of function-based indexes. Beware of implicit type conversions.

```sql 
-- Bad
WITH t1 as (SELECT id, a FROM t0)
SELECT t1.a, t2.b FROM t1 INNER JOIN t2 ON TO_NUMBER(t1.id) = t2.id

-- Good
WITH t1 as (SELECT TO_NUMBER(id) AS id, a FROM t0)
SELECT t1.a, t2.b FROM t1 INNER JOIN t2 ON t1.id = t2.id
```

#### EXISTS
Subqueries within **EXISTS** or **NOT EXISTS** may or may not improve performance. Often it is possible to rewrite them as JOINs. Beware of not introducing duplicates when doing so.

```sql 
-- Bad?
SELECT t1.id FROM t1
WHERE NOT EXISTS (SELECT 1 FROM t0 where t0.id = t1.id)

-- Good?
SELECT t1.id FROM t1 LEFT JOIN t0 ON t1.id = t0.id WHERE t0.id IS NULL
```

#### Avoid joins to tables that are not needed
This might seem obvious and it should be, but deviations from this rule are occasionally observed.

```sql 
-- Bad
SELECT t0.a, t2.b FROM t0
LEFT JOIN t1 ON t0.id = t1.id
LEFT JOIN t2 ON t0.id = t2.id

-- Good
SELECT t0.a, t2.b FROM t0
LEFT JOIN t2 ON t0.id = t2.id
```

#### NULL handling
The handling of three-valued logic is not entirely consistent within SQL. Anyway, if in a specific context (such as predicates used after **CASE WHEN**, **JOIN ON**, or **WHERE**), the distinction between the truth values **UNKNOWN** and **FALSE** is irrelevant, then it does not make sense to capture **UNKNOWN** using functions like **COALESCE** or **NVL**. In these contexts, as long as complicated predicates don't involve negations (**NOT**), but only conjunctions (**AND**) or disjunctions (**OR**), it is safe to ignore the distinction between the truth values **UNKNOWN** and **FALSE**. 

```sql 
-- Bad
SELECT t1.id FROM t0 WHERE COALESCE(amount, 0) > 0

-- Good
SELECT t1.id FROM t0 WHERE amount > 0
```

#### Avoid table repetitions
Multiple occurrences of the same base tables within a single SELECT statement should be treated with suspicion. If there is redundance which can be avoided, one should aspire to do so. 

#### Use bind variables
The queries below might look identical, if the bind variable **:product_cd** is set to the value **'1201'**, but they are not. 
```sql 
-- Bad
SELECT * FROM PRODUCTS WHERE product_cd = '1201'

-- Good
SELECT * FROM PRODUCTS WHERE product_cd = :product_cd
```
From the perspective of the database these queries are entirely different. Oracle encourages the use of <a href="https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/improving-rwp-cursor-sharing.html">bind variables</a>, instead of hard coded literals. In practice, the runtime can vary considerably, depending on the use of bind variables or literals.

#### CASE WHEN
If the **WHERE** clause precludes a condition it should not be tested within **CASE WHEN**.

```sql 
-- Bad
SELECT CASE
    WHEN product_cd = '1202' THEN 'a'
    ELSE 'b' AS description FROM PRODUCTS
WHERE product_cd = '1201'

-- Good
SELECT 'b' AS description FROM PRODUCTS
WHERE product_cd = '1201'
```
Multiple levels of nested CASE statements are suspicious. If a CASE follows after ELSE, then it should be merged with the higher CASE statement. 

```sql 
-- Bad
SELECT CASE
    WHEN a = 'x' THEN 1 ELSE CASE
    WHEN b = 'y' THEN 2 ELSE 3 END
    END AS code FROM PRODUCTS

-- Good
SELECT CASE
    WHEN a = 'x' THEN 1
    WHEN b = 'y' THEN 2
    ELSE 3 END
    AS code FROM PRODUCTS
```

#### Views
Beware of views that reference other views, especially when they involve multiple levels of recursion. Joins involving views might lead to suboptimal plans.

#### Beware of hints
<a href="https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Comments.html">Hints</a>
should be treated with suspicion. The general rule is to not use them. There are even undocumented hints in Oracle SQL, such as **materialize**, which can be used to persist intermediate results of a complicated query.

### Data modelling

Among other things, data modelling concerns the choice of data structures used to persist the data, before it is retrieved. There should be a feedback loop from the observed patterns of data retrieval to data modelling. In Oracle, the column usage patterns are stored in the undocumented table **SYS.COL_USAGE$**. This table can be used to gain insights on which columns are popular for filtering. The partitioning and indexing strategy should reflect these empirical column usage patterns.

#### Small data is good data
Data redundancies can be present at the column or row level. 
At the column level small data types should be preferred. For character data, **NVARCHAR2** requires more space than **VARCHAR2** and should only be used, if needed. Numeric data should not be stored using character types.

At the row level, table normalization reduces redundancies and overall storage consumption present in the table data. The idea is to split a table into multiple tables, each having fewer columns and fewer rows.

#### Partitioning 
<a href="https://docs.oracle.com/en/database/oracle/oracle-database/19/vldbg/partition-concepts.html">Partitioning</a> divides a table into smaller pieces and involves at maximum two levels.
The hierarchy of <a href="https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/logical-storage-structures.html">logical storage structures</a> in Oracle involves rows, blocks, extents, and segments. Every extent resides in a single data file on disk. Every unpartitioned table, table partition or table subpartition is its own segment. Partitioning enables pruning and can greatly improve performance, because the database might know in advance, which data files to retrieve from the storage tier, thus avoiding search. Oracle recommends partitioning for tables exceeding 2GB of size.

#### Indexes
The standard type of index is the B-tree (balanced tree). If the key is composite, then the more selective or frequently filtered columns should be placed on the left side, with less important columns to the right.
For columns with low cardinality and infrequent updates bitmap indexes may be used.
