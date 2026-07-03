# SQL Notes

Notes from hands-on SQL practice (PostgreSQL first, with Snowflake syntax noted separately), covering everything from basic filtering to database design and business analytics queries.

## Files

1. [Fundamentals]() — SELECT, WHERE, aggregating, sorting/grouping, LIKE/regex
2. [Joins & Set Operations]() — INNER/OUTER/CROSS/SELF joins, UNION/INTERSECT/EXCEPT, semi/anti joins
3. [Subqueries, CASE WHEN, CTEs]() — nested queries, correlated subqueries, WITH clauses
4. [Window Functions](04-window-functions.md) — OVER(), RANK/ROW_NUMBER, PARTITION BY, frames, LAG/LEAD
5. [Pivoting & Group Totals]() — CROSSTAB, ROLLUP, CUBE, GROUPING SETS
6. [PostgreSQL Functions]() — data types, date/time functions, text parsing, full-text search
7. [Database Design]() — OLTP vs OLAP, dimensional modeling, normalization (1NF–3NF)
8. [Views, Roles & Partitioning]() — views vs. materialized views, access control, table partitioning
9. [Business Analytics in SQL]() — retention rate, ARPU, cohort bucketing, pivoted executive reports
10. [Snowflake SQL]() — syntax differences, JSON/VARIANT handling, NATURAL/LATERAL joins

## Reference tables

Comparison tables I keep coming back to are collected in each relevant file rather than duplicated here — e.g. `RANK()` vs `DENSE_RANK()` vs `ROW_NUMBER()` lives in [04](https://claude.ai/chat/04-window-functions.md), `ROLLUP` vs `CUBE` lives in [05](https://claude.ai/chat/05-pivoting-and-totals.md).

---

[← Back to main index](../README.md)
