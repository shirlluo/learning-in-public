# Window Functions
A class of functions that perform calculations on a result set that has already been generated (_ie_, a _**window**_); can be used to perform aggregate calculations without grouping, and calculate running totals, rankings, and moving averages.

```sql
SELECT
  function_name() OVER (...) AS new_column
FROM table;
```

`OVER()` can be empty, or contain `ORDER BY`, `PARTITION BY`, and frame clauses (`ROWS BETWEEN ... AND ...`).

> **Note:** window functions are evaluated _after_ the rest of the query (except the final `ORDER BY`). That means window functions use the result set to calculate, instead of the database.

---

## 1. Aggregation over a window

**`OVER()`:** pass the aggregate value over the existing result set (like a subquery in SELECT), useful for comparing an individual row to an overall total (_e.g._ percent of total, performance vs. benchmark).

```sql
SELECT
	AVG(value) OVER () AS overall_avg
FROM table;
```

```sql
-- Calculate percent of total:
SELECT team_id,
	SUM(points) / SUM(points) OVER() AS percent_of_ttl
FROM match
GROUP BY team_id
```

## 2. Rankings

- **`RANK()`**: generates a column numbering the dataset in ascending order (default) based on a specified column

```sql
SELECT ...
	RANK() OVER(ORDER BY col ASC/DESC) AS col_rank
```

```sql
SELECT
  league.name,
  AVG(m.home_goal + m.away_goal) AS avg_goals,
  -- Rank leagues in descending order by average goals:
  RANK() OVER (ORDER BY AVG(m.home_goal + m.away_goal) DESC) AS league_rank
FROM league
LEFT JOIN match AS m ON league.id = m.country_id
WHERE m.season = '2011/2012'
GROUP BY league.name
ORDER BY league_rank;
```

`RANK()` ties identical values together, then skips the next rank number.

- **`DENSE_RANK()`**: assigns the same number to rows with identical values, but doesn’t skip over the next number

## 3. Row numbers: `ROW_NUMBER()`

Assigns each row a unique sequential number

```sql
SELECT ROW_NUMBER() OVER (ORDER BY col) AS rn
FROM table;
```

If `ORDER BY` is omitted inside `OVER()`, row order is undefined.

Ordering **inside** `OVER()` determines the numbering itself; ordering **outside** `OVER()` just sorts the final result set.

1. Inside first: **`ROW_NUMBER()` will assign numbers **based on the order within** **`OVER`** (so the row numbers are given after sorting the table by _year_ and _event_)
2. Then outside: the `ORDER BY` outside of OVER sorts the results of the table by _country_ and _row number_.

```sql
SELECT year, event, country,
	ROW_NUMBER() OVER(ORDER BY year DESC, event ASC) AS rn
FROM events
ORDER BY country ASC, rn ASC;
```

**🌟 Ranking variants compared:**

| Function       | Duplicate values            | Skips next number? |
| -------------- | --------------------------- | ------------------ |
| `ROW_NUMBER()` | always unique, even if tied | n/a                |
| `RANK()`       | same number for ties        | yes                |
| `DENSE_RANK()` | same number for ties        | no                 |

## 4. Partitioning: `PARTITION BY`

Calculates separate values for each category/group, without collapsing rows.

```sql
SELECT
  Aggfunc OVER (PARTITION BY column) AS ...
```

Can partition by multiple columns.

With `PARTITION BY`:

- `ROW_NUMBER()` resets at the start of each partition
- `LAG()` only pulls a value if the previous row is in the _same_ partition

## 5. Sliding windows (running totals, moving averages)

A sliding window calculates relative to the current row, using a frame **within the `OVER` clause** to specify the data we want to use:

```sql
OVER (ROWS BETWEEN <start> AND <finish>)
```

**Frame keywords:**

|Keyword|Meaning|
|---|---|
|`n PRECEDING` / `n FOLLOWING`|n rows before / after the current row|
|`UNBOUNDED PRECEDING` / `UNBOUNDED FOLLOWING`|include every row since the beginning / end of the dataset|
|`CURRENT ROW`|stop the calculation at the current row|

```sql
-- Running total (cumulative sum) since the beginning
SELECT
  Country, Year, Medals,
  MAX(Medals) OVER (PARTITION BY Country ORDER BY Year ASC) AS Max_Medals
FROM Country_Medals
ORDER BY Country, Year;

-- 3-year moving average
SELECT
  Year, Medals,
  AVG(Medals) OVER (PARTITION BY Country ORDER BY Year ASC
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS Medals_MA
FROM Country_Medals
ORDER BY Country, Year;
```

**`ROWS BETWEEN` _vs_. `RANGE BETWEEN`:**

- `ROWS BETWEEN` counts a fixed number of _rows_ relative to the current one.
- `RANGE BETWEEN` looks at the _value_ in the `ORDER BY` column; duplicate values are treated as one group and calculated together.

## 6. Relative & absolute value lookups

| Function           | Returns                                           |
| ------------------ | ------------------------------------------------- |
| `LAG(col, n)`      | `col`'s value `n` rows **before** the current row |
| `LEAD(col, n)`     | `col`'s value `n` rows **after** the current row  |
| `FIRST_VALUE(col)` | the first value in the table/partition            |
| `LAST_VALUE(col)`  | the last value in the table/partition             |

```sql
SELECT
  year, Champion,
  LAG(Champion, 1) OVER (ORDER BY year ASC) AS Last_Champion
FROM Weightlifting_Gold
ORDER BY year ASC;
```

> By default a window starts at the beginning of the table/partition and **ends at the current row,** so without `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, `LAST_VALUE` just returns the current row's value.

## 7. Paging: `NTILE(n)`

Splits data into `n` approximately equal chunks (_e.g_. top/middle/bottom third, quartiles for percentile analysis), returns the number of the group to which the row belongs.

```sql
WITH thirds AS (
  SELECT
    Athlete, Medals,
    NTILE(3) OVER (ORDER BY Medals DESC) AS third -- return 1, 2, or 3
  FROM Athlete_Medals
)
SELECT third, 
	AVG(Medals) AS avg_medals -- get the average medals earned in each third
FROM thirds
GROUP BY third
ORDER BY third ASC;
```

---

## Quick reference: which function do I need?

- **Compare a row to the group total** → aggregate function + `OVER()`
- **Rank rows, ties should share a rank** → `RANK()` or `DENSE_RANK()`
- **Give every row a unique index** → `ROW_NUMBER()`
- **Calculate per-category values without collapsing rows** → `PARTITION BY`
- **Running total / moving average** → sliding window with `ROWS BETWEEN`
- **Compare a row to the previous/next row** → `LAG()` / `LEAD()`
- **Split into equal-sized buckets (e.g. quartiles)** → `NTILE(n)`

---
Part of my [SQL notes](README.md)
