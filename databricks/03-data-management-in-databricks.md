# Delta Lake & Data Management

How Databricks stores and manages tables: what Delta Lake adds over a traditional data lake, how managed and unmanaged tables differ, and how views and access control fit in.

---

## 1. What Delta Lake is

Delta Lake was developed by Databricks to manage large, dynamic datasets. It addresses several limitations of traditional data lakes:

- **ACID transactions** (Atomicity, Consistency, Isolation, Durability): protect against partial updates and inconsistent reads, so processing stays reliable.
	
	To confirm all changes are committed under ACID properties:
```sql
SELECT COUNT(*) FROM table_name;
```

- **Schema enforcement and evolution**: enforces schemas to keep data valid, while allowing the schema to evolve without breaking workflows or corrupting data.
- **Time travel** (data versioning): query previous versions of the data to track and verify what it looked like in the past.
- **Unified batch and streaming**: the same table supports both streaming (real-time) and batch reads and writes, which reduces redundancy and simplifies pipeline architecture.

---

## 2. Managing data in Delta Lake

- Update records:
```sql
UPDATE table
	SET field = new_value
	WHERE <condition>;
```

- Delete records:
```sql
DELETE FROM table_path
WHERE <condition>;
```

- Delete the whole table:
```sql
DROP table_name;
```

- Compact small files to improve query performance:
```sql
OPTIMIZE table_path;
```

- Check the numFiles property:
```sql
DESCRIBE DETAIL table;
```

---

## 3. Table persistence: managed vs. unmanaged

**Table persistence** defines how data is stored and retained across sessions, which affects storage, access, and maintenance.

||Managed table|Unmanaged (external) table|
|---|---|---|
||Fully handled by Databricks, including data location and lifecycle|A decentralized approach, providing greater flexibility and control|
|On `DROP TABLE`|underlying data is removed too|underlying data is **not** removed|
|Storage location|default path, handled automatically|user-defined location like S3, ADLS, or Azure Blob|
|Suitable for|simple, centralized data management|custom storage needs or compliance requirements|
|Data sharing|sharing across systems requires exporting|easier to share via accessible storage|

When creating an **unmanaged table**, we need to specify the data's *storage location* and manage its *lifecycle* independently

```sql
-- Managed tables:
-- Create a managed table from an existing table
CREATE TABLE patients_managed AS
SELECT * FROM patients;

-- Dropping it removes the data as well
DROP TABLE patients_managed;
```

### The LOCATION keyword

`LOCATION` defines the exact storage location, essential for unmanaged tables.

- It affects storage costs, retrieval times, and retention policies.
- It overrides default storage, so data can live in external cloud locations (eg, AWS S3 or Azure Blob Storage).
- Data can be relocated without disrupting the table's structure or the workflow around it.

### Creating databases and tables

1. Create a database (as a logical container to organize related tables & views):  
    ```sql
    CREATE DATABASE my_custom_databse;
    ```
    
2. Create a table within this database:
    
    ```sql
    USE my_custom_databse;
    
    CREATE TABLE default_table AS
    SELECT 1 AS id, 'Sample Name' AS name;
    ```
    
3. Customize storage location for a table:
    
    ```sql
    USE my_custom_databse;
    
    CREATE TABLE custom_table 
    USING DELTA
    LOCATION 'path'
    AS
    SELECT '2' AS id, 'Another Sample Name' AS name;
    ```
    
4. Check the locations: <img width="1132" height="720" alt="image" src="https://github.com/user-attachments/assets/93af4b64-e0c5-4843-8101-1055e731c49b" />



---

## 4. Views and temp views

- Create/Update views:
	```sql
	CREATE VIEW view_name AS ...;
	CREATE OR REPLACE VIEW view_name AS ...;
	```

- Temp views: last only for the current session and will be automatically erased when the session ends
	```sql
	CREATE OR REPLACE TEMP VIEW view_name AS ...;
	```

- Check the list of persistent views
	```sql
	SHOW VIEWS IN default;
	```

Temp views are dropped automatically when the session ends, so they won't appear as persistent views afterwards.

---

## 5. Data exploration and security

**Personally Identifiable Information (PII)** is any data that identifies individuals. 
A common safeguard is **role-based access control (RBAC)**: limit who can view or modify PII to the roles that actually need it.

---

## Quick reference: which do I need?

- **Need a table that's safe against partial writes and inconsistent reads**: Delta Lake (ACID)
- **Need to see what the data looked like before**: time travel
- **Same table must serve streaming and batch** : Delta Lake unified processing
- **Simple setup, Databricks handles storage**: managed table
- **Data must stay in a specific cloud location, or survive a `DROP TABLE`**: unmanaged table with `LOCATION`
- **A query result that only matters for this session**: temp view
- **Restrict access to sensitive columns or tables**: role-based access control

---

_Part of my [Databricks notes](https://claude.ai/chat/README.md), written while upskilling in data analytics._
