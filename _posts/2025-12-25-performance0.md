---
layout: post
title: "Oracle SQL performance"
date: 2025-12-25
categories: [SQL, Oracle, performance]
---
Lately I spent some time with performance optimization in Oracle SQL and I would like to share my key preliminary takeaways.


### Filter early, add late

The entire SELECT statement can be imagined as a tree. The top-level SELECT statement is the root of the tree, while the row sources are the leaves. Within this tree, filters should be applied as early as possible. Also, data should be added as late as possible.
