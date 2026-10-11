# Exploratory Data Analysis in SQL

What to do with a dataset you haven't seen before: check its keys and types, summarize the numbers, clean up categorical text, and read the shape of a distribution. 

---

## 1. First checks: keys, NULLs, and types

- **Primary key**: unique and never `NULL`.
- **Foreign key**: a value that exists in the referenced column, or `NULL`. It must reference a column with unique values, so the referenced row can be identified. (More on keys in [07-database-design](./07-database-design.md)) 
- `COALESCE(value1, value2, ...)` works row by row and returns the first non-null value, left to right.

**Casting to a different type**

```sql
SELECT CAST(profit AS INTEGER) FROM fortune500;

-- Same thing with :: notation
SELECT 10::NUMERIC / 3;   -- 3.333333...
```

Casting matters in the second example: dividing two integers returns an integer, so one side needs to be cast first.

---

## 2. Summarizing and aggregating numeric data

### Number types

| Type                       | Notes                                                                                                    |
| -------------------------- | -------------------------------------------------------------------------------------------------------- |
| `serial`                   | auto-incrementing integer, used to generate ID columns when the data has no unique identifier of its own |
| `real`, `double precision` | decimal numbers stored with less precision                                                               |

### Reducing precision: TRUNC()

`TRUNC()` replaces the smallest numeric places with zeros.

```sql
SELECT trunc(4.1246, 2);    -- return 4.12
SELECT trunc(12735, -3);    -- return 12000
```

> **Note:** Truncating is not the same as rounding. The result's absolute value is never larger than the original number.

### Grouping data:

- `TRUNC()` with a **negative** second argument: indicate places before the decimal to replace with zero, *eg*, `trunc(12735, -3)` ⇒ 12000
	```sql
	SELECT trunc(age, -1) AS age_group, count(*)   -- 20, 30, 40, ...
	FROM people
	GROUP BY age_group;
	```

- `generate_series(start, end, step)` builds a series of values, and the end value is **inclusive**. It's handy for creating bins or filling gaps.

### Summary statistics

|Measure|Function|
|---|---|
|Population variance (divides by n)|`var_pop()`|
|Sample variance (divides by n - 1)|`var_samp()`|
|Sample standard deviation|`stddev()` or `stddev_samp()`|
|Population standard deviation|`stddev_pop()`|
|Correlation between two columns|`corr(a, b)`|

### Median and percentiles

Percentile functions use ordered-set aggregate syntax, so the data has to be ordered, and the percentile is a number between 0 and 1.

```sql
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY col);  -- smooth distributions
SELECT percentile_disc(0.5) WITHIN GROUP (ORDER BY col);  -- when results must be a real data point
```

| Function            | Behavior                                                                                                                  | Use when                                             |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `percentile_cont()` | **continuous**: interpolates between the values around the percentile, so the result might not exist in the original data | describing a distribution                            |
| `percentile_disc()` | **discrete**: always returns a value that exists in the column                                                            | the answer must be an actual item (an ID, a ranking) |

>**Median in MySQL** (no ready-made function): number the rows in both directions and average the middle one or two.

```sql
WITH cte AS (
  SELECT *,
    ROW_NUMBER() OVER (ORDER BY col ASC)  AS row_asc,
    ROW_NUMBER() OVER (ORDER BY col DESC) AS row_desc
  FROM table_name
)
SELECT AVG(col)
FROM cte
WHERE row_asc IN (row_desc, row_desc - 1, row_desc + 1);
```

### Temporary tables

`CREATE TEMP TABLE` copies the result of a query into a new table that has **no connection** to the original table. It's deleted automatically when you disconnect from the database.

```sql
CREATE TEMP TABLE new_table AS
SELECT col_1, col_2
FROM table_name;
```

*Alternative*: with `SELECT INTO`

```sql
SELECT col_1, col_2
INTO TEMP TABLE new_table
FROM table_name;
```

**Inserting data**

```sql
INSERT INTO table_name VALUES (1, 'Cheese', 9.99);

-- Naming the columns is safer
INSERT INTO table_name (product_no, name, price) VALUES (1, 'Cheese', 9.99);
```

- Insert rows from a query: the selected columns must match the table's columns

```sql
INSERT INTO created_table
SELECT col_1, col_2
FROM table_name;
```


**Dropping**: add `IF EXISTS` to avoid an error when the table isn't there.

```sql
DROP TABLE IF EXISTS profit80;

CREATE TEMP TABLE profit80 AS
  SELECT sector,
         percentile_disc(.8) WITHIN GROUP (ORDER BY profits) AS pct80
  FROM fortune500
  GROUP BY sector;

SELECT * FROM profit80;
```

Temporary tables will also be deleted automatically when you disconnect from the database.

---

## 3. Categorical data and unstructured text

### Text data types

| Type                                 | Behavior                                                                                          |
| ------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `CHARACTER(n)`, `CHAR(n)`            | fixed length n; shorter values are padded with trailing spaces (which are ignored in comparisons) |
| `CHARACTER VARYING(n)`, `VARCHAR(n)` | variable length, up to a maximum of n                                                             |
| `TEXT`, `VARCHAR`                    | no specified length, unlimited                                                                    |

### Checking a categorical column

```sql
SELECT category, count(*)
FROM table_name
GROUP BY category
ORDER BY count DESC;
```

Sorting order for text: space < `'A'` < `'Z'` < `'a'` < `'z'`.

The usual inconsistencies are **case** and **spaces**:
- Case-insensitive search: `ILIKE` ("I" for insensitive).
- Removing spaces:

| Function              | Removes spaces from                            |
| --------------------- | ---------------------------------------------- |
| `trim()` or `btrim()` | both ends                                      |
| `ltrim()`             | the left end, eg, `ltrim(' abc ')` ⇒ `'abc '`  |
| `rtrim()`             | the right end, eg, `rtrim(' abc ')` ⇒ `' abc'` |

`trim` can also remove other characters: `trim('Wow!', '!wW')` gives `'o'`.

### Multiple transformations: a recode table

When several values need standardizing, build a lookup instead of nesting functions.

1. `CREATE TEMP TABLE` with 2 columns that start with the same values: original and standardized.
	```sql
	CREATE TEMP TABLE reocde AS
		SELECT DISTINCT fav_fruit AS original,
			fav_fruit AS standardized
		FROM fruit;
	```
2. `UPDATE` the standardized column.
	```sql
	UPDATE temp_table
	SET new_column = standardized_value
	WHERE <condition>;
	```

	For example:
	```sql
	-- All rows: lower case, remove white space on ends
	UPDATE recode
	SET standardized = trim(lower(original));
	
	-- Specific rows: correct misspelling
	UPDATE recode
	SET standardized = 'banana'
	WHERE standardized LIKE '%nn%';
	
	-- All rows: remove any 's'
	UPDATE recode
	SET standardized = rtrim(standardized, 's');
	```
3. `JOIN` the original table to the recode table by matching the original values.
	```sql
	SELECT standardized, COUNT(*)
	FROM fruit
	LEFT JOIN recode
	ON fav_fruit = original
	GROUP BY standardized;
	```

---

## 4. Reading a distribution (histograms)

- **Modality**: unimodal (one peak), bimodal (two), trimodal (three).
- **Skewness**: a left-skewed distribution has its outliers (extreme values) on the left.
- **Kurtosis**: how many extreme values there are.
    - _Leptokurtic_: a narrow peak and lots of extreme values.
    - _Platykurtic_: a broad peak and few extreme values.

---

## Quick reference: which do I need?

- **Bucket numbers into groups (decades, hundreds)**: `trunc()` with a negative second argument
- **Typical value of a distribution**: `percentile_cont(0.5)`
- **Median that must be a real row**: `percentile_disc(0.5)`
- **Spread of the data**: `var_samp()` / `stddev()`
- **Relationship between two columns**: `corr(a, b)`
- **Keep an intermediate result without touching the source table**: `CREATE TEMP TABLE`
- **Messy categories (case, stray spaces)**: `ILIKE`, `trim()`, or a recode table
- **Understand the shape of a column**: histogram (modality, skew, kurtosis)

---

_Part of my [SQL notes](./README.md), written during upskilling in data analytics._
