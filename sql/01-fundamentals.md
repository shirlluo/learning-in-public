# SQL Fundamentals

The core syntax every other SQL topic builds on: selecting, filtering, aggregating, sorting, and grouping. Examples use PostgreSQL, with SQL Server differences noted where they matter.

---

## 1. Selecting data

`SELECT field FROM table;`

```sql
SELECT *
FROM books;
```

`*` indicates all fields, `;` indicates the query is complete. 
If a field name contains spaces, wrap it in double quotes: `SELECT title, "release year" FROM films;

- **Alias a column or table:** `SELECT field AS abc FROM table AS t`
- **Unique rows:** `SELECT DISTINCT field`

```sql
-- Unique combinations across multiple columns
SELECT DISTINCT department_id, year_hired
FROM employees;
```

**Views**: a saved query that behaves like a virtual table. It stores the *query*, not the data, so results always reflect the latest table state:

```sql
CREATE VIEW employee_hire_years AS
SELECT id, name, year_hired
FROM employees;
```

**Limiting results**: *(the one place PostgreSQL and SQL Server diverge)*

- PostgreSQL:
```sql
SELECT id, name
FROM employees
LIMIT 2;
```

- SQL Server:
```sql
SELECT TOP(2) id, name
FROM employees;
```

**COUNT():**
- `COUNT(*)` includes rows with missing values
- `COUNT(field)` counts only non-missing values in that field
- `COUNT(DISTINCT field)` counts unique non-missing values

```sql
SELECT COUNT(name) AS count_names, COUNT(birthdate) AS count_bd
FROM people;

SELECT COUNT(DISTINCT birthdate) AS count_distinct_bd
FROM people;
```

---

## 2. Filtering (WHERE)

```sql
SELECT ... FROM ... WHERE condition LIMIT n;
```

Execution order: `FROM` → `WHERE` → `SELECT` → `LIMIT`.

**Comparison operators:** `>`, `<`, `>=`, `<=`, `=` (***Note:*** single `=`, not `==`), `<>`

```sql
SELECT COUNT(*) AS films_over_100K_votes
FROM reviews
WHERE num_votes >= 100000;
```

**Combining conditions:** `OR`, `AND`, `BETWEEN`
The field must be specified on both sides of an `OR`/`AND`. `BETWEEN` is inclusive of both endpoints.

```sql
SELECT *
FROM clothes
WHERE color = 'yellow' AND length = 'long';
```

**Pattern matching:** `LIKE`, `NOT LIKE`, `IN`

- `LIKE` & `NOT LIKE`: to find records that match (or do not match) a specified pattern

| Wildcard | Matches                              |
| -------- | ------------------------------------ |
| `%`      | 0, 1, or many characters             |
| `_`      | exactly one character                |
| `[0-9]`  | any character in the set/range       |
| `[^0-9]` | any character *not* in the set/range |

```sql
-- eg, third character is 't'
SELECT title FORM movies LIKE '__t%'
```

In PostgreSQL, `~~` is synonymous with `LIKE`, and `!~~` with `NOT LIKE`.

For anything more complex than wildcards, `REGEXP_LIKE(field, 'pattern', match_type)` supports full regex: 

| pattern       | matches                   |
| ------------- | ------------------------- |
| `^`           | anchor: starting with     |
| `$`           | anchor: ending with       |
| `.`           | any character             |
| `*`           | from 0 to many characters |
| `+`           | 1 or many characters      |
| `?`           | either 0 or 1 character   |
| `{n}`,`{n,m}` | repeat counts             |

| match-type flags |                           |
| ---------------- | ------------------------- |
| `c`              | case-sensitive matching   |
| `i`              | case-insensitive matching |
| `m`              | multi-line mode           |
| `e`              | extract sub-matches       |


- **Multiple values:** `IN()`

```sql
SELECT title, certification, language
FROM films
WHERE certification IN ('NC-17', 'R')
  AND language IN ('English', 'Italian', 'Greek');
```

- **Missing values:** `IS NULL`, `IS NOT NULL`

```sql
-- Count missing birthdates
SELECT COUNT(name) AS non_birthdates
FROM people
WHERE birthdate IS NULL;
```

---

## 3. Aggregating

**Aggregate functions** operate vertically, down a column:
- Numeric only: `AVG()`, `SUM()`
- Any type: `MIN()`, `MAX()`, `COUNT()`

`ROUND(number, decimal_places)`: `decimal_places`= 0 by default
When passing a negative value to `decimal_places`, the function rounds to the left of the decimal point (eg, to round to the hundred thousand)

**Arithmetic** (`+ - * /`) operates horizontally within a record
>Dividing two integers returns an *integer* in SQL ⇒ cast to `float` or multiply by `1.0` for a precise result:

```sql
-- Calculate the title and duration_hours from films
SELECT title, ROUND(duration / 60.0, 2) AS duration_hours
FROM films;

-- Find the number of decades in the films table
SELECT (MAX(release_year) - MIN(release_year)) / 10.0 AS number_of_decades
FROM films;
```

 Execution order: `FROM` → `WHERE` → `SELECT` (aliases created here) → `LIMIT`
 ***Note***: an alias defined in `SELECT` **cannot** be referenced in `WHERE`, it doesn't exist yet at that point in execution.

---

## 4. Sorting (ORDER BY)

```sql
SELECT ... FROM ... WHERE ... ORDER BY field;
```

Ascending by default (symbols → numbers → letters). Be explicit with `ASC`/ `DESC`
`ORDER BY` runs last, just before `LIMIT`

```sql
SELECT certification, release_year, title
FROM films
ORDER BY certification ASC, release_year DESC;
```

---

## 5. Grouping (GROUP BY & HAVING)

#### 5.1 GROUP BY

```sql
SELECT ... FROM ... WHERE ... GROUP BY field;
```

When grouping by one field but selecting others, every non-grouped column needs an aggregate function.

Combine `GROUP BY` with `ORDER BY`: group our results, make a calculation ( in `SELECT`), and then order our results

```sql
SELECT release_year, country, MAX(budget) AS max_budget
FROM films
GROUP BY release_year, country
ORDER BY release_year, country;
```

Execution order: `FROM` →`WHERE` → `GROUP BY` → `SELECT` data (create alias) → `ORDER BY` →`LIMIT`

#### 5.2 Filtering grouped data: HAVING

**HAVING vs. WHERE**: 
`WHERE` filters individual rows *before* grouping; `HAVING` filters *already-grouped* results

```sql
SELECT country, ROUND(AVG(budget), 2) AS average_budget
FROM films
GROUP BY country
HAVING ROUND(AVG(budget), 2) > 1000000000
ORDER BY average_budget DESC;
```

**Full execution order:**

```
FROM → WHERE → GROUP BY → HAVING → SELECT (aliases created) → ORDER BY → LIMIT
```

Aliases created in `SELECT` *can* be reused in `ORDER BY`, since `ORDER BY`
runs after `SELECT`, unlike `WHERE`, which runs before it.

---

## Quick reference: which do I need?

- **Row-level condition** → `WHERE`
- **Group-level condition (after aggregating)** → `HAVING`
- **Text pattern match** → `LIKE` / `REGEXP_LIKE`
- **Check for missing data** → `IS NULL` / `IS NOT NULL`
- **Total, average, or count across rows** → aggregate function + `GROUP BY`
- **Sort final output** → `ORDER BY` (runs last)

---
*Part of my [SQL notes](./README.md), written during upskilling in data analytics.*
