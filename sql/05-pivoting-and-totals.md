# Pivoting & Group Totals

Turning long-format results into wide, spreadsheet-style tables, and generating subtotal/grand-total rows without writing several separate queries.

---

## 1. Pivoting with CROSSTAB

`CREATE EXTENSION`: makes extra functions in an extension available for use (like `import` in Python)
To create pivot table: `CROSSTAB`

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM CROSSTAB($$
  <source_sql>
$$) AS ct (col_unpivot datatype,
           col_pivot_1 datatype_1,
           ...)
```

1. `tablefunc` contains the `CROSSTAB` function
2. Place the source query between `$$...$$`
3. `ct(...)` lists the output column names and data types for the pivoted result (VARCHAR, INTEGER, …)

> Use double quotes `""` for column names that are *numbers* or contain *special characters* (NOT single quotes) to ensure SQL treats them as identifiers (Single quotes are used for string literals, not for identifiers like column names)

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM CROSSTAB($$
  WITH Country_Awards AS (
    SELECT Country, Year, COUNT(*) AS Awards
    FROM Summer_Medals
    WHERE
      Country IN ('FRA', 'GBR', 'GER')
      AND Year IN (2004, 2008, 2012)
      AND Medal = 'Gold'
    GROUP BY Country, Year)
  SELECT
    Country, Year,
    RANK() OVER (PARTITION BY Year ORDER BY Awards DESC) :: INTEGER AS rank
  FROM Country_Awards
  ORDER BY Country ASC, Year ASC;
$$) AS ct (country VARCHAR,
           "2004" INTEGER,
           "2008" INTEGER,
           "2012" INTEGER)
ORDER BY Country ASC;
```

In PostgreSQL, `::dtype` casts to a different data type (equivalent to `CAST(... AS dtype)`). This matters here because `RANK()` and `COUNT()` return `BIGINT` by default, which won't always match the type declared in `ct()`.

---

## 2. Group-level and grand totals

### 2.1 ROLLUP

`ROLLUP()`: a `GROUP BY` sub-clause that adds extra subtotal rows for group-level aggregations (similar to a running "Total" row at the bottom of a report)

```sql
SELECT country, gender, COUNT(*) AS Gold_Awards
FROM Summer_Medals
WHERE
  Year = 2004 AND Medal = 'Gold' AND Country IN ('DEN', 'NOR', 'SWE')
-- Generate Country-level subtotals
GROUP BY country, ROLLUP(gender)
ORDER BY Country ASC, Gender ASC;
```

Returns *country-level* totals across gender subgroups, with gender as `NULL` on the subtotal row


**Grand Total**: if we ROLLUP all GROUP BY columns (`GROUP BY ROLLUP(col1, col2)`), we'll have a grant total row

`ROLLUP` is **hierarchical**: the order of the columns inside `ROLLUP()` will affect the output
- `ROLLUP(country, medal)` produces country-level subtotals
- `ROLLUP(medal, country)` produces medal-level subtotals
- Either way, both include a grand total row

### 2.2 CUBE

`CUBE()`: a non-hierarchical version of `ROLLUP`, generates **every possible** group-level aggregation

**ROLLUP vs. CUBE:** 
- Use `ROLLUP` for hierarchical data (eg, date parts, where year > quarter > month naturally nests) when we don't need every possible combination
- Use `CUBE` when we want all combinations

### 2.3 GROUPING SETS

Equivalent to a `UNION` across several separate `GROUP BY` queries, and equivalent to `GROUP BY CUBE(a, b)`.

```sql
GROUP BY GROUPING SETS ((a, b), (a), (b), ())
```

Each of the groups in () represents one GROUP BY statement; the final empty `()` produces the grand total row
-This combines everything a pivot table would show into a single query
-We can specify which aggregation levels to include (instead of getting every combination with `CUBE`)

---

## 3. Replace nulls from total row: COALESCE

`ROLLUP`, `CUBE`, pivoting, and window functions like `LAG`/`LEAD` all tend to leave `NULL`s behind

`COALESCE(col, 'text')`: takes a list of values and returns the first non-null value (left to right), which makes it the fix for turning those `NULL`s into a readable label

```sql
SELECT
	COALESCE(country, 'Both countries') AS country,
	COALESCE(medal, 'All medals') AS medal,
	COUNT(*) AS awards
FROM medals_2008
WHERE COUNTRY IN ('CHN', 'RUS')
GROUP BY ROLLUP(country, medal)
ORDER BY country, medal
```
⇒The Country column will display 'Both countries' on the grand total line, the Medal column will display 'All medals' on the total medals lines


Passing the original column as the first argument keeps all non-null values unchanged, only actual `NULL`s get replaced by the fallback
```sql
COALESCE(
  CASE WHEN status IN ('Open', 'In Progress', 'Resolved')
  THEN status END, -- replace any other values with NULL
  'Resolved') AS status -- return 'Resolved' for null values
```


---

## 4. Compressing rows into a list: STRING_AGG

`STRING_AGG(column, 'separator')` concatenates every value in a column into a single delimited string, eg, a comma-separated list

```sql
WITH Country_Medals AS (
  SELECT
    Country,
    COUNT(*) AS Medals
  FROM Summer_Medals
  WHERE Year = 2000
    AND Medal = 'Gold'
  GROUP BY Country),
Country_Ranks AS (
  SELECT
    Country,
    RANK() OVER (ORDER BY Medals DESC) AS Rank
  FROM Country_Medals
  ORDER BY Rank ASC)
  
-- Compress the countries column
SELECT STRING_AGG(Country, ', ')
FROM Country_Ranks
WHERE Rank <= 3;

-- Result: USA, RUS, AUS
```

---

## Quick reference: which do I need?

- **Turn long-format results into a wide table** : `CROSSTAB`
- **Add subtotal rows for hierarchical groups** : `ROLLUP`
- **Every possible group-level combination** : `CUBE`
- **Pick exactly which aggregation levels to show** : `GROUPING SETS`
- **Clean up a NULL total-row label** : `COALESCE`
- **Turn a column of values into one delimited string** : `STRING_AGG`

---
*Part of my [SQL notes](./README.md), written during upskilling in data analytics.*
