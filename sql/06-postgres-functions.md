# PostgreSQL Functions

PostgreSQL-specific tools for exploring a database's structure, working with arrays, handling dates and time series, parsing text, and running full-text search. Less about core query syntax, more about the functions that make PostgreSQL practical to work in day-to-day.

---

## 1. Exploring database structure

- To check tables in my database: `pg_catalog.pg_tables` (query the system table) ⇒ returns all tables for all schemas

```sql
-- All tables across all schemas (PostgreSQL-specific)
SELECT * FROM pg_catalog.pg_tables;

-- Same idea, ANSI-standard
SELECT * FROM information_schema.tables;
```

- `INFORMATION_SCHEMA` is where PostgreSQL stores metadata about every database object

```sql
-- Public tables only
SELECT * FROM INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'public';

-- Columns of a specific table
SELECT * FROM INFORMATION_SCHEMA.COLUMNS
WHERE table_name = 'orders';
```

- `PG_TYPEOF(column_name)` returns a column's data type (PostgreSQL-only)

```sql
SELECT pg_typeof(column_name) FROM table_name;
```

---

## 2. Arrays

To create a table with an array column:
1. Create an ARRAY type: append `[]` to the data type

```sql
CREATE TABLE grades (
	student_id int,
	email text[][], -- nested array
	scores int[] -- an array of integ
);
```

2. Insert array values with `'{...}'` syntax

```sql
INSERT INTO grades
	VALUES (1, -- id
	'{{"main","mainemail@abc.com"},{"other","other@abc.com"}}', -- email
	'{92, 81, 88, 97}'); -- scores
```

To access ARRAYs: `SELECT array[index]` (***Note:*** indexes start at 1, not 0)
To search ARRAYs with an index: `WHERE array_col[i]=value` 

```sql
SELECT title, special_features
FROM film
WHERE special_features[1] = 'Trailers';
```

`ANY()` searches an array for a value (regardless of position):

```sql
SELECT title, special_features
FROM film
WHERE 'Trailers' = ANY (special_features);
```

-Or use the contains operator `@>`: `WHERE array_col @> ARRAY[...]`

---

## 3. Dates and times

### 3.1 Main types
- `TIMESTAMP`: a DATE (`yyyy-mm-dd`) & a TIME value at microsecond precision
	With timezone: `YYYY-MM-DD HH:MM:SS+HH` (offset from UTC)
- `INTERVAL`: a span of time (years, months, days, hours...), written as `'n day(s)'` or `'n year(s)'`, etc.

### 3.2 Basic arithmetic operations
- `DATE` - `DATE` ⇒ returns an `INTEGER`
  `DATE + INTEGER` ⇒ returns another `DATE`
- `TIMESTAMP - TIMESTAMP`⇒ returns an `INTERVAL`
  Or, use `AGE()` to calculate the difference between two timestamp values

- Convert `INTEGER` or `TIMESTAMP` to `INTERVAL`: multiply an integer by a '1-day' interval

```sql
SELECT 
	INTERVAL '1 day' * timestamp '2019-04-10 12:34:56', 
	INTERVAL '1 day' * INTEGER 7
```
- Convert an `INTERVAL` to `INT`: `EXTRACT(field FROM date/time)` or `DATE_PART('field', date/time)`

### 3.3 Functions

**Current timestamp**
- `NOW()` or `CURRENT_TIMESTAMP`: returns the current timestamp at microsecond precision with timezone
>**Difference**: `CURRENT_TIMESTAMP(n)` lets us specify precision, *eg*, `CURRENT_STAMP(2)`
- `CURRENT_DATE`, `CURRENT_TIME` : just the date, or just the time
- Drop the timezone: `CAST(NOW() AS timestamp)` or `NOW()::timestamp`

**Extracting fields**

- `SPLIT_PART('string','seperator','partNum')`: split a string (representing date) and return part of it. *Eg*, to return the year: `SPLIT_PART('30.11.1989', '.', 3)`
- `EXTRACT(field FROM datetime)`, or `DATE_PART('field', datetime)`

`datetime` needs to be a TIMESTAMP, DATE, TIME, or INTERVAL data type
`field` can be `year`, `month`, `quarter`, `dow` (day of week, PostgreSQL only), etc. 
>*`dow`: starts from Sunday=0, ends on Saturday=6*


```sql
SELECT
  EXTRACT(dow FROM rental_date) AS dayofweek,
  AGE(return_date, rental_date) AS rental_days
FROM rental
WHERE rental_date BETWEEN CAST('2005-05-01' AS timestamp)
  AND CAST('2005-05-01' AS timestamp) + INTERVAL '90 days';
```

**Truncate dates/timestamps**
- `DATE_TRUNC('field', timestamp/interval)`: returns a TIMESTAMP/INTERVAL at a specified precision
***Note***: `DATE_TRUNC()` returns an interval or timestamp but NOT a numeric value

**Formatting with `TO_CHAR`**
`to_char(timestamp, format)`: converts a number or a timestamp into a string in a specified format

| Format | Meaning | Example |
|---|---|---|
| `YYYY` / `YY` | full / 2-digit year | 2023 / 23 |
| `Q` | quarter | `'"Q"Q'` → Q2 |
| `MM` | 2-digit month | 01 |
| `MON` / `Month` | abbreviated / full month name | Jan / January |
| `DD` | day of month | 01-31 |
| `DY` / `Day` | abbreviated / full day name | Mon / Monday |

```sql
TO_CHAR(timestamp, 'day')                    -- sunday, monday, ...
TO_CHAR(DATE_TRUNC('month', date), 'Month')  -- July, August, ...
```

**Aggregation with series** (including periods of time with no observations):

1. Generate DateTime series: `generate_series(start, end, interval)`, useful for filling in gaps where no rows exist for a given date
Returns the last value ≤ the end date. If `start > end`, it returns an empty result. 

```sql
generate_series('2016-01-01', '2018-06-30', '1 day'::interval)
```


2. Join the generated series to the original data and `COALESCE` the missing counts to 0

*Eg*, Monthly average with missing dates: find the average requests created per day for each month of the data.

```sql
-- Generate series with all days from 2016-01-01 to 2018-06-30
WITH all_days AS (
  SELECT generate_series('2016-01-01', '2018-06-30', '1 day'::interval) AS date
),
daily_count AS (
  SELECT date_trunc('day', date_created) AS day, count(*) AS count
  FROM orders
  GROUP BY day
)

-- Aggregate daily counts by month using date_trunc
SELECT date_trunc('month', date) AS month,
       avg(coalesce(count, 0)) AS average
FROM all_days
LEFT JOIN daily_count ON all_days.date = daily_count.day
GROUP BY month
ORDER BY month;
```

Similarly, aggregation with bins:
1. Generate two series: lower bound and upper bound of each bin (*upper bound exclusive)
2. Left join the original table

**Time gaps between events:**
`LAG()` pushes all of the values down one row; `LEAD()` pulls all values up one row.

>Window functions can't be nested inside an aggregate function directly, so computing an *average* gap needs a subquery: compute the per-row gap first, then aggregate the result

```sql
SELECT AVG(gap)
FROM (
	SELECT date - lag(date) OVER(ORDER BY date) AS gap
	FROM sales
	) AS gaps
```

---

## 4. Parsing and manipulating text

| Task        | Syntax                            | Notes                                                                                                                                   |
| ----------- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Concatenate | \|\| or `CONCAT(a, b, c)`         | Able to concatenate mixed data types (string & non-string)<br>Difference: `CONCAT()` skips NULLs; \|\| returns NULL if any part is NULL |
| Change case | `UPPER()`, `LOWER()`, `INITCAP()` | `INITCAP()` capitalizes the first letter (title case)                                                                                   |
| Replace     | `REPLACE(field, old, new)`        |                                                                                                                                         |
| Reverse     | `REVERSE()`                       | returns the same string in reverse order                                                                                                |

**Length and position**
- `CHAR_LENGTH()` or `LENGTH()`: returns character count, including spaces
- `POSITION('char' IN field)`: returns 1-indexed position of the *first* match
- `SRTPOS(field, 'character')`

**Substrings**
- `LEFT(field, n)`, `RIGHT(field, n)`: first/last n characters
- `SUBSTRING(field, index_start, length)`, or `SUBSTRING(field FROM start FOR length)`: extract a substring, 1-indexed
  If the `length` parameter is omitted, it'll read all the way to the end.
- `SUBSTR(field, start, length)`: same idea, no `FROM`/`FOR` variant.

```sql
SELECT
  LEFT(email, POSITION('@' IN email) - 1) AS username,
  SUBSTRING(email FROM POSITION('@' IN email) + 1 FOR LENGTH(email)) AS domain
FROM customer;
```

>`REVERSE()` combined with `POSITION()` to find the *last* whitespace in a string, e.g. to truncate a description without cutting a word in half:

```sql
-- Get the first 50 words of description
SELECT UPPER(category.name) || ': ' || title AS film_category, 
	LEFT(description, 50 - POSITION(' ' IN REVERSE(LEFT(description, 50))))
```

- `SPLIT_PART('string', 'seperator', 'partNum')`: split string on delimiter and return the given field (1-indexed). eg, `SPLIT_PART('a,bc,d', ',', 2)` ⇒ `'bc'`

**Remove whitespace:**
- `TRIM([LEADING | TRAILING | BOTH] [characters] FROM string)` removes characters from one or both ends (defaults: `BOTH`, whitespace)
  - `LEADING/TRAILING/BOTH`: Optional, to specify where to trim (from the beginning ,or the end of the string, or both). If omitted, default to BOTH
  - `characters`: Optional, defaults to space
- `LTRIM()`/`RTRIM()`: only remove characters from either the beginning or the end of the string

**Pad with characters:**
- `LPAD(string, length, pad_char)` / `RPAD()`: pad a string on the left to a fixed length (default pad character: space)
- `RPAD()`: pads the string on the right

---

## 5. Full-text search

Full-text search uses stemming, fuzzy string matching to handle spelling mistakes, and a mechanism to rank results by similarity to the search string.

LIKE *v.s*. Full-text search:
- `LIKE` matches literal wildcards (`_` = one character, `%` = zero or more)
- Full-text search goes further: it handles variations of strings, and is case-insensitive

```sql
-- Perform a full-text search on the title column for the word elf
SELECT title, description
FROM film
-- Convert the title to a tsvector and match it against the tsquery
WHERE to_tsvector(title) @@ to_tsquery('elf');
```

The match operator `@@` compares the values returned by two built-in functions: `to_tsvector`, `to_tsquery`
- `tsvector`: data type, a normalized, sorted list of word "lexemes")
- `tsquery`: data type, stores lexemes that are to be searched for

```sql
-- Transform the film description to a tsvector
SELECT to_tsvector(description) FROM film;
```

**Fuzzy matching extensions**

To discover the available extensions to be installed: query the `pg_available_extensions` system view
To check the installed extensions (currently available to use): `pg_extension`

```sql
-- Available extensions:
SELECT name FROM pg_available_extensions;

-- Installed extensions:
SELECT exname FROM pg_extension;
```

- `fuzzystrmatch` extension → `levenshtein()`: calculates the *levenshtein* distance between two strings (*ie*, the number of edits required for the strings to be a perfect match)
- `pg_trgm` extension → `similarity()`: returns a trigram-based similarity score from 0 (no match) to 1 (perfect match).

```sql
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;

SELECT
  title, description,
  similarity(description, 'Astounding Drama')
FROM film
-- Match "Astounding Drama" in the description
WHERE to_tsvector(description) @@ to_tsquery('Astounding & Drama')
ORDER BY similarity(description, 'Astounding Drama') DESC;
```

>`IF NOT EXISTS` avoids an error if the extension is already installed

---

## 6. User-defined types

Use `CREATE TYPE` to register the type in a system table and make it available to be used anywhere PostgreSQL expects a type name

1. Create enumerated data types (`ENUM`) to define a custom list of values that are never going to change, like the days of the week

```sql
CREATE TYPE weekday AS ENUM (
  'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'
);
```

2. Confirm it's registered / Access: query the system table `pg_type`, which lists every data type (built-in and user-defined) available in the database

```sql
SELECT typname, typcategory
FROM pg_type
WHERE typname = 'weekday';
```

---

## Quick reference: which do I need?

- **See what's in the database**: `pg_catalog.pg_tables` or `INFORMATION_SCHEMA`
- **Fill in missing dates in a time series**: `generate_series` + `LEFT JOIN` + `COALESCE`
- **Format a date for display**: `TO_CHAR(date, format)`
- **Round a date down to the month/week/day**: `DATE_TRUNC`
- **Pull part of a string out**: `LEFT`/`RIGHT`/`SUBSTRING`/`SPLIT_PART`
- **Match text approximately, not exactly**: full-text search (`tsvector`/`tsquery`) or `similarity()`

---
*Part of my [SQL notes](./README.md), written during upskilling in data analytics*
