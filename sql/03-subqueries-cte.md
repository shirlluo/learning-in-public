# CASE WHEN, Subqueries & CTEs
Three related tools for handling conditional logic and multi-step queries: `CASE WHEN` for inline conditionals, subqueries for nesting one query inside another, and CTEs for naming a subquery so it reads cleanly.

---

## 1. CASE WHEN

SQL's equivalent of the if/else statement: contains `WHEN`, `THEN`, `ELSE` clauses, finished with `END` and an alias, and returns a new column.

```sql
SELECT
  ...,
  CASE WHEN x = 1 THEN 'a'
       WHEN x = 2 THEN 'b'
       ELSE 'c' END AS new_column
FROM table;
```

- If `ELSE` is omitted, non-matching rows return `NULL`.
- To filter on a `CASE` result, repeat the full `CASE` expression (without the alias) inside `WHERE`. *Note: You can't reference the alias there.*

**Aggregating with CASE**: put the column to aggregate inside `THEN`, and wrap the whole thing in an aggregate function:

```sql
SELECT
  season,
  SUM(CASE WHEN hometeam_id = 8560 THEN home_goal END) AS home_goals,
  SUM(CASE WHEN awayteam_id = 8560 THEN away_goal END) AS away_goals
FROM match
GROUP BY season;
```

**Percentages with CASE + `AVG()`**: score each row 1 or 0, then average. Any row that hits neither condition becomes `NULL` and is automatically **excluded** from the average:

```sql
SELECT
  AVG(CASE WHEN condition_is_met THEN 1
           WHEN condition_is_not_met THEN 0 END) AS pct
FROM table;
```

---

## 2. Subqueries

A subquery can appear inside `SELECT`, `FROM`, `WHERE`, or `GROUP BY`. The table it queries can be the same table or a different one.

### Subquery inside WHERE: filtering against a computed value

Compare with an aggregating result:

```sql
SELECT *
FROM populations
WHERE year = 2015
  AND life_expectancy > 1.15 *
    (SELECT AVG(life_expectancy)
     FROM populations
     WHERE year = 2015);
```

Generate a filter list with `IN`:

```sql
SELECT name, country_code, urbanarea_pop
FROM cities
WHERE name IN
  (SELECT capital FROM countries)
ORDER BY urbanarea_pop DESC;
```

### Subquery inside SELECT: comparing a row to an aggregate

Must return a single value, and requires an **alias**.

```sql
SELECT
  countries.name AS country,
  -- Subquery that provides the count of cities:
  (SELECT COUNT(name)
   FROM cities
   WHERE cities.country_code = countries.code) AS cities_num
FROM countries
ORDER BY cities_num DESC, country
LIMIT 9;
```

### Subquery inside FROM: treating a query result as a table

Useful for restructuring data or calculating an aggregate-of-an-aggregate. Requires an **alias**, and a **shared column** to join on.

*Eg, generate a subquery, and then join it to the country table to calculate information about matches with 10 or more goals in total*

```sql
SELECT
  c.name AS country_name,
  COUNT(sub.id) AS matches
FROM country AS c
INNER JOIN (
  SELECT country_id, id
  FROM match
  WHERE (home_goal + away_goal) >= 10) AS sub
ON c.id = sub.country_id
GROUP BY country_name;
```

---

## 3. Correlated subqueries

A correlated subquery references a column from the _outer_ query, which means it can't run on its own, and gets re-evaluated once per row of the final result set.

|Simple subquery|Correlated subquery|
|---|---|
|Runs independently of the main query|Depends on values from the main query|
|Evaluated once, total|Evaluated in a loop — once per outer row (slower)|

```sql
SELECT
  main.country_id,
  main.date,
  main.home_goal,
  main.away_goal
FROM match AS main
-- Filter for matches with the maximum number of total goals scored:
WHERE 
	(home_goal + away_goal) =
	  (SELECT MAX(sub.home_goal + sub.away_goal)
	   FROM match AS sub
	   WHERE main.country_id = sub.country_id
	     AND main.season = sub.season);
```

---

## 4. Nested subqueries

A subquery inside another subquery: each level can be correlated, uncorrelated, or a mix. Built in three logical steps:
1. **Inner subquery**: pulls the base value
2. **Outer subquery**: aggregates the inner result into a scalar value that can be used in the final query
3. **Final query**: compares the scalar value to each row of the main query

```sql
SELECT
  season,
  MAX(home_goal + away_goal) AS max_goals,
  -- Outer subquery:
  (SELECT MAX(home_goal + away_goal)
   FROM match
   WHERE season = main.season
     AND
     country_id IN (
     -- Inner subquery to get the max goals in an 'England Premier League' match for the same season:
       SELECT country_id
       FROM league
       WHERE name = 'England Premier League')
  ) AS pl_max_goals
FROM match AS main
GROUP BY season;
```

Nested + Correlated together: an inner subquery grouped and aggregated, then joined back to the outer query:

```sql
SELECT
  c.name AS country,
  AVG(outer_s.matches) AS avg_seasonal_high_scores
FROM country AS c
LEFT JOIN (
  SELECT country_id, season, COUNT(id) AS matches
  FROM (
    SELECT country_id, season, id
    FROM match
    WHERE home_goal >= 5 OR away_goal >= 5) AS inner_s
  GROUP BY country_id, season
) AS outer_s
ON c.id = outer_s.country_id
GROUP BY country;
```

---

## 5. Subqueries inside WHERE EXISTS

`WHERE EXISTS`: checks if the result of a correlated nested query is NOT empty (*ie*, the query returns _any_ rows), returns a boolean rather than a value. 
`NOT EXISTS`: returns `TRUE` when it's empty.

```sql
SELECT col
FROM table_1 AS t1
WHERE EXISTS
	(SELECT * FROM table_2 AS t2 WHERE t2.col2 = t1.col2);
```

*Eg, select all actors who play in a Comedy:*

```sql
SELECT *
FROM actors AS a
WHERE EXISTS (
  SELECT *
  FROM actsin AS ai
  LEFT JOIN movies AS m ON m.movie_id = ai.movie_id
  WHERE m.genre = 'Comedy'
    AND ai.actor_id = a.actor_id
);
```

**`EXISTS` vs. `IN` vs. `JOIN`**

- `EXISTS` only ever returns `TRUE`/`FALSE`. `IN` can also return `UNKNOWN`/`NULL` (we can't compare `UNKNOWN`/`NULL`, so `EXISTS` is generally preferred for existence checks.
- Use **JOIN** when we need to retrieve columns in `SELECT` from multiple tables.
- Use **EXISTS** when we only need to check whether matching data exists in another table.

---

## 6. Common Table Expressions (CTEs)

A CTE is a subquery declared before the main query: we name it using `WITH`, then reference it by name like a table (like giving a temporary table a label instead of nesting it inline).

```sql
WITH cte AS (
  SELECT col1, col2
  FROM table
)
SELECT ...
FROM cte;
```

 *Eg, return the median of the Northern Latitudes (LAT_N) and round it to 4 decimal places.* 
```sql
WITH ranked AS (
    SELECT LAT_N,
        ROW_NUMBER() OVER(ORDER BY LAT_N ASC) AS n_asc,
        ROW_NUMBER() OVER(ORDER BY LAT_N DESC) AS n_desc
    FROM STATION
    ORDER BY LAT_N ASC
)
SELECT ROUND(AVG(LAT_N), 4) AS median
FROM ranked
WHERE n_asc IN (n_desc, n_desc-1, n_desc+1);
```

Multiple CTEs: separated by `,` in the same `WITH` statement.

*Eg: [Project Planning](https://www.hackerrank.com/challenges/sql-projects/problem): return the start and end dates of different projects. If the `End_Date` of the tasks are consecutive, then they are part of the same project.* 
```sql
-- output: start and end dates of projects
-- order: the NUM of days it took to complete ASC, Start_Date
WITH p_start AS (
    SELECT Start_Date, 
        ROW_NUMBER() OVER(ORDER BY Start_Date) AS r_start
    FROM Projects
    WHERE Start_Date NOT IN (SELECT End_Date FROM Projects)
),
p_end AS (
    SELECT End_Date,
        ROW_NUMBER() OVER(ORDER BY End_Date) AS r_end
    FROM Projects
    WHERE End_Date NOT IN (SELECT Start_Date FROM Projects)
)
SELECT s.Start_Date, e.End_Date
FROM p_start AS s
    LEFT JOIN p_end AS e
    ON r_start = r_end
ORDER BY DATEDIFF(e.End_Date, s.Start_Date), s.Start_Date;
```

**Why use a CTE over a plain subquery in FROM?** 
It's readability: the main query stays flat and easy to follow instead of nesting several levels deep, and a CTE can be referenced more than once in the same query.

---

## Quick reference: which do I need?

- **Inline conditional logic in a column** → `CASE WHEN`
- **Filter rows against a computed/aggregate value** → subquery in `WHERE`
- **Compare one row to an overall aggregate** → subquery in `SELECT`
- **Treat a query result as a joinable table** → subquery in `FROM`
- **Subquery depends on the outer query's current row** → correlated subquery
- **Just need to check if matching rows exist** → `WHERE EXISTS`
- **Query is deeply nested and hard to read** → pull it out into a CTE

---

_Part of my [SQL notes](sql/README) — written while upskilling in data analytics._
