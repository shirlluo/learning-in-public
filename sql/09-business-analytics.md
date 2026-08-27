# Analyzing Business Data in SQL (KPIs)

Common patterns for turning raw transaction/order data into the metrics a business team actually asks for: growth, retention, revenue per user, and pivot tables.

---

## 1. Running totals: cumulative registrations

Combine a CTE with a window function to turn monthly counts into a cumulative total, without a self-join.

```sql
WITH reg_dates AS (
  SELECT user_id,
    MIN(order_date) AS reg_date
  FROM orders
  GROUP BY user_id
),
-- Registrations by month
regs AS (
  SELECT DATE_TRUNC('month', reg_date) :: DATE AS delivr_month,
    COUNT(DISTINCT user_id) AS regs
  FROM reg_dates
  GROUP BY delivr_month
)

-- Calculate the registrations running total by month
SELECT
  delivr_month,
  SUM(regs) OVER (ORDER BY delivr_month) AS regs_rt
FROM regs
ORDER BY delivr_month ASC;
```

---

## 2. Retention rate

**Retention rate = Uc / Up**, where *Uc* is the count of distinct users active in BOTH the current and previous months, and *Up* is the count of users active in the previous period. Self-join the same activity table, offset by one month:

```sql
WITH user_monthly_activity AS (
  SELECT DISTINCT
    DATE_TRUNC('month', order_date) :: DATE AS delivr_month,
    user_id
  FROM orders
)

SELECT
  previous.delivr_month,
  ROUND(
    COUNT(DISTINCT current.user_id)::NUMERIC /
    GREATEST(COUNT(DISTINCT previous.user_id), 1),
  2) AS retention_rate
-- Join user_monthly_activity to itself on the user ID and the month, pushed forward one month:
FROM user_monthly_activity AS previous
LEFT JOIN user_monthly_activity AS current
  ON previous.user_id = current.user_id
  AND previous.delivr_month = current.delivr_month - INTERVAL '1 month'
GROUP BY previous.delivr_month
ORDER BY previous.delivr_month ASC;
```

> Wrap the denominator in `GREATEST(..., 1)` to avoid a divide-by-zero error on any period with no prior users.

---

## 3. Unit economics: ARPU

ARPU (Average Revenue Per User): total revenue divided by distinct users for a given time window.

```sql
WITH kpi AS (
  SELECT
    DATE_TRUNC('week', order_date) :: DATE AS delivr_week,
    SUM(order_quantity * meal_price) AS revenue,
    COUNT(DISTINCT user_id) AS users
  FROM meals AS m
  JOIN orders AS o ON m.meal_id = o.meal_id
  GROUP BY delivr_week
)

SELECT
  delivr_week,
  ROUND(revenue :: NUMERIC / users, 2) AS arpu
FROM kpi
ORDER BY delivr_week ASC;
```

---

## 4. Frequency tables & bucketing

**Frequency table**: round revenue into $100 bins to see the distribution of users across spending ranges:

```sql
WITH user_revenues AS (
  SELECT
    user_id,
    SUM(order_quantity * meal_price) AS revenue
  FROM meals AS m
  JOIN orders AS o ON m.meal_id = o.meal_id
  GROUP BY user_id
)

SELECT
  ROUND(revenue::NUMERIC, -2) AS revenue_100,
  COUNT(user_id) AS users
FROM user_revenues
GROUP BY revenue_100
ORDER BY revenue_100 ASC;
```

**Bucketing with CASE WHEN**: group users into named segments instead of raw bins:

```sql
SELECT
  CASE
    WHEN revenue < 150 THEN 'Low-revenue users'
    WHEN revenue < 300 THEN 'Mid-revenue users'
    ELSE 'High-revenue users'
  END AS revenue_group,
  COUNT(user_id) AS users
FROM user_revenues
GROUP BY revenue_group;
```

**Quartiles** with `PERCENTILE_CONT`:

```sql
SELECT
  ROUND(
    PERCENTILE_CONT(.25) WITHIN GROUP (ORDER BY revenue) :: NUMERIC,
  2) AS revenue_p25
FROM user_revenues;
```

---

## 5. Executive reports: pivoting with CROSSTAB

PostgreSQL's `tablefunc` extension → `CROSSTAB()`

*Eg*, rank ordering users by eatery and by quarter:

```sql
-- Import tablefunc
CREATE EXTENSION IF NOT EXISTS tablefunc;

-- Pivot the previous query by quarter
SELECT * FROM CROSSTAB($$
  WITH eatery_users AS (
    SELECT
      eatery,
      TO_CHAR(order_date, '"Q"Q YYYY') AS delivr_quarter,
      COUNT(DISTINCT user_id) AS users
    FROM meals
    JOIN orders ON meals.meal_id = orders.meal_id
    GROUP BY eatery, delivr_quarter
    ORDER BY delivr_quarter, users
  )
  SELECT
    eatery,
    delivr_quarter,
    RANK() OVER (
      PARTITION BY delivr_quarter
      ORDER BY users DESC) :: INT AS users_rank
  FROM eatery_users
  ORDER BY eatery, delivr_quarter;
$$)
-- Select the columns of the pivoted table
AS ct (
  eatery TEXT,
  "Q2 2018" INT,
  "Q3 2018" INT,
  "Q4 2018" INT
)
ORDER BY "Q4 2018";
```

>`CROSSTAB` needs `tablefunc` enabled, a fixed inner query wrapped in `$$`, and an explicit list of output columns — one per pivoted value.

---

## 6. Bonus: compressing consecutive dates into ranges

A common variant of business KPI work: turning a list of individual active dates into date *ranges*, *eg*, "this user ordered every day from the 1st to the 5th" instead of five separate rows
***Trick:*** subtracting a row number from the date ← consecutive dates produce the same difference, so that difference becomes a grouping key

```sql
WITH c AS (
  SELECT
    consumer_id,
    order_date,
    (DATE_PART('day', AGE(order_date, (SELECT MIN(order_date) FROM orders))) + 1)
      - (ROW_NUMBER() OVER (PARTITION BY consumer_id ORDER BY order_date)) AS gap
  FROM (SELECT DISTINCT consumer_id, order_date FROM orders) AS d
)

SELECT
  consumer_id,
  MIN(order_date) AS start_date,
  MAX(order_date) AS end_date
FROM c
GROUP BY consumer_id, gap
ORDER BY consumer_id, start_date;
```

This "gaps and islands" pattern shows up anywhere you need to detect streaks: *active-user runs, consecutive login days*, or matching up project start/end dates where adjoining tasks count as the same project.

---

## Quick reference: which metric do I need?

- **Is growth accelerating or flattening?** → running total with a window function
- **Are users coming back?** → retention rate (self-join offset by one period)
- **How much is each user worth?** → ARPU
- **How is revenue distributed across users?** → frequency table or bucketed CASE WHEN
- **Need a report by group *and* time period, side by side?** → CROSSTAB pivot
- **Need to detect streaks or active-date ranges?** → gaps-and-islands with ROW_NUMBER

---
*Part of my [SQL notes](./README.md), written during upskilling in data analytics.*
