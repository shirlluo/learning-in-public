# Databricks SQL

Running SQL in Databricks: getting a warehouse running, bringing data into the lakehouse, transforming it in layers, and keeping pipelines efficient as new data arrives.

---

## 1. Getting started

1. Create a **SQL Warehouse** (a compute cluster designed for SQL process): Choose *Serverless* for simplicity and performance.
2. Use the **SQL Editor** to query data.
3. Create visuals with the **+** next to the results section (*Visualization*), or build a *Dashboard*.

---

## 2. Ingesting data

**GUI-based options**

| Option | What it does | Best for |
|---|---|---|
| Lakeflow Connect | fully managed, built-in connectors for databases and SaaS apps; auto-creates pipelines for existing and new data | large-scale, production datasets |
| Data upload | manually upload files (CSV, Parquet, ...) to quickly create Delta tables | smaller, ad hoc analysis |

**Code-based options**

| Option | What it does | Best for |
|---|---|---|
| `COPY INTO` | copies data from cloud storage directly into Delta tables | relatively static datasets that don't change often |
| Auto Loader | automatically ingests new files from cloud storage (`FROM cloud_files('path', 'csv')`) | larger, dynamic data, and (near) real-time updates |

### Using COPY INTO

1. Create a new volume: <img width="1616" height="750" alt="image" src="https://github.com/user-attachments/assets/c19a606d-d1a9-4e2f-a05e-eebb7f3091e1" />
   **Managed volume**: a folder in cloud storage where you can upload files directly, with access managed through Unity Catalog.

2. Upload the CSV file and copy its file path (the Catalog browser on the left can copy it for you).<img width="1614" height="602" alt="image" src="https://github.com/user-attachments/assets/793f00ac-9b07-4562-b76f-1d25defac8ae" />

3. Run `COPY INTO` in the SQL Editor:

  ```sql
  COPY INTO vendor_data
  FROM '/Volumes/<catalog>/default/<volume_name>/vendor_data_extra.csv'
  FILEFORMAT = CSV
  COPY_OPTIONS ('mergeSchema' = 'true');
  ```

---

## 3. Transforming data

1. Clean data in the **raw layer → analytics-ready layer**
2. Aggregate & simplify data from the **raw layer → BI-ready layer**

Pipelines can be automated with **Workflows**.

---

## 4. Optimizing data pipelines

### Handling incoming data

Both strategies work whether the table structure stays the same or the schema changes.

- **Incrementally append** with `INSERT INTO`: adds all new data to the **end** of the existing table. Assumes every row is new and existing data isn't changing.
- **Change Data Capture (CDC)** with `MERGE INTO`: integrates new data into an existing table, either appending new rows or updating existing ones.

  ```sql
  MERGE INTO existing_table AS a
  USING new_data AS b
  ON a.key = b.key
  WHEN MATCHED THEN UPDATE SET *      -- update all columns of existing rows
  WHEN NOT MATCHED THEN INSERT *;     -- append rows that don't exist yet
  ```

To update only certain columns, replace `UPDATE SET *` with
`UPDATE SET a.col = b.col`.

**Use case: updating sales data**
1. Set the notebook to run from centralized data schema:
  ```sql
  USE CATALOG ...;
  USE SCHEMA default;
  ```

2. Insert new data:
  ```sql
  INSERT INTO existing_table TABLE new_data;
  ```

3. Merge new data:
  ```sql
  MERGE INTO existing_table USING new_data ON ...;
  ```

### Data layout optimization

| Command | What it does | Works best for |
|---|---|---|
| `OPTIMIZE` | compacts several small files or subsets together, fixing the "small file problem" | any table with many small files |
| `Z-ORDER` | co-locates related data in the same files (similar to indexing), reducing read time | high-cardinality columns (many unique values) |
| `PARTITION BY` | splits the table by column value | low-cardinality columns (few unique values) |

```sql
OPTIMIZE table_name; -- on a table

OPTIMIZE table_name
  WHERE date >= '2024-01-01'; -- subset of data

OPTIMIZE table-name
  WHERE date >= current_timestamp() - INTERVAL 1 day
  ZORDER BY (eventType);
```

---

## Quick reference: which do I need?

- **Quick, one-off file load** : data upload
- **Production-scale loading from databases or SaaS tools** : Lakeflow Connect
- **Static dataset loaded from cloud storage** : `COPY INTO`
- **Files keep arriving and I want them picked up automatically** : Auto Loader
- **All incoming rows are new** : `INSERT INTO`
- **Incoming rows might update existing ones** : `MERGE INTO`
- **Too many small files** : `OPTIMIZE`
- **Filtering often on a column with many unique values** : `Z-ORDER`
- **Filtering often on a column with few unique values** : `PARTITION BY`

---
*Part of my [Databricks notes](learn-in-public/README.md), written while upskilling in data analytics.*
