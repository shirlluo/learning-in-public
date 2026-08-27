# Joins & Set Operations

Two different ways to combine tables: JOIN combines columns side by side (horizontally), while Set operations stack rows on top of each other (vertically). Semi and anti joins are a hybrid: they filter one table based on another without actually pulling in its columns.

---
## 1. JOINs (horizontally)
### 1.1 Inner join

`INNER JOIN` keeps only the records that match on the joining field in **both** tables.

```sql
SELECT ...
FROM left_table AS t1
INNER JOIN right_table AS t2
ON left_table.key = right_table.key;
```

Execution order: `FROM` → `INNER JOIN` → `ON` → `SELECT`

- When a column name exists in both tables, use `table.column` to avoid ambiguity errors (also in the `SELECT` clause) 
- Alias tables with `AS` in `FROM` and `JOIN`: the alias can then be used in both `SELECT` and `ON`
- When joining on two identically named columns, use `USING(key_col)` instead of `ON`

**Joining on multiple keys**: use `AND` in the `ON` clause so the result only includes records matching on *both* keys at once.

```sql
SELECT *
FROM t1
INNER JOIN t2
ON t1.id = t2.id
	AND t1.date = t2.date;
```

**Chaining multiple joins:**

```sql
SELECT name, e.year, fertility_rate, unemployment_rate
FROM countries AS c
INNER JOIN populations AS p
ON c.code = p.country_code
INNER JOIN economies AS e
ON c.code = e.code AND e.year = p.year;
```

---

### 1.2 Outer joins

| Join       | Keyword                              | Keeps                                                     |
| ---------- | ------------------------------------ | --------------------------------------------------------- |
| Left join  | `LEFT JOIN` (or `LEFT OUTER JOIN`)   | all rows from the left table, matched rows from the right |
| Right join | `RIGHT JOIN` (or `RIGHT OUTER JOIN`) | all rows from the right table, matched rows from the left |
| Full join  | `FULL JOIN`                          | all rows from both tables, matched where possible         |

The table order in a `RIGHT JOIN` mirrors how you'd write the same query as a `LEFT JOIN`, just with the tables swapped.

---

### 1.3 Cross join

`CROSS JOIN` returns every possible combination of rows between two tables, with no `ON` or `USING` clause at all

```sql
SELECT col_1, col_2
FROM table1
CROSS JOIN table_2;
```

---

### 1.4 Self join

Used to compare rows within the same table for the same entity (eg, this year's value against last year's). Requires two *different* aliases for the same table (mandatory)

```sql
-- Compare the population of different countries in 2010 and 2015
SELECT
  p1.country_code, p1.size AS size2010, p2.size AS size2015
FROM populations AS p1
INNER JOIN populations AS p2
ON p1.country_code = p2.country_code
WHERE p1.year = 2010
  AND p1.year = p2.year - 5;
```

To exclude a row matching itself, add `AND t1.key <> t2.key` to `ON`

---

## 2. Set operations (vertically)

Set operations stack rows vertically instead of joining columns, so they need no shared key at all. 
-Only requirement: both queries must select the **same number of columns**, with **matching data types** in the same order. 
-The output uses the column names from the **left (first)** query, regardless of aliasing.

- `UNION`: takes two tables as input, and returns all records **(without duplicates**) from both tables
- `UNION ALL`: takes two tables and returns **all** records from both tables, including duplicates
- `INTERSECT`: takes two tables as input, and returns only the records that exist in **both** tables
- `EXCEPT`: retains only records from the left table that are **NOT** present in the right table.

```sql
-- UNION: all pairs of code and year from both tables, no duplicates
SELECT code, year
FROM economies
UNION
SELECT country_code, year
FROM populations
ORDER BY code, year;

-- EXCEPT: cities whose name doesn't match any country name
SELECT name
FROM cities
EXCEPT
SELECT name
FROM countries
ORDER BY name;
```

---

## 3. Semi join & anti join (Subqueries)

Both filter rows in one table based on values found (or not found) in another, without ever pulling in that second table's columns. They're built with a `WHERE ... IN` subquery rather than a `JOIN` keyword.

**Semi join**: keep rows where a column's value *is* in a list produced by a subquery on another table

```sql
-- Languages spoken only in countries within the 'Middle East' region
SELECT DISTINCT name
FROM languages
WHERE code IN
  (SELECT code
   FROM countries
   WHERE region = 'Middle East')
ORDER BY name;
```

**Anti join**: keep rows where a column's value is **not** in that list. Same structure, with `NOT` added before `IN`

```sql
SELECT ...
FROM languages
WHERE code NOT IN
  (SELECT code
   FROM countries
   WHERE region = 'Middle East');
```

---

## Quick reference: which do I need?

- **Only rows that match in both tables** : `INNER JOIN`
- **All rows from one table, matches from the other** : `LEFT`/`RIGHT JOIN`
- **All rows from both, matched where possible** : `FULL JOIN`
- **Every possible row combination** : `CROSS JOIN`
- **Compare rows within the same table** : self join (`INNER JOIN` with two aliases)
- **Stack two result sets with the same columns** : `UNION` / `UNION ALL`
- **Rows common to both result sets** : `INTERSECT`
- **Rows in one result set but not the other** : `EXCEPT`
- **Filter one table by matches in another, without pulling in its columns** : semi join (`WHERE IN`) or anti join (`WHERE NOT IN`)

---
*Part of my [SQL notes](./README.md), written during upskilling in data analytics.*
