# DP-800 Complete Study Notes: SQL AI Developer Associate 

These revised notes cover all three domains of [Exam DP-800: Developing AI-Enabled Database Solutions](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800) for the [Microsoft Certified: SQL AI Developer Associate](https://learn.microsoft.com/en-us/credentials/certifications/developing-ai-enabled-database-solutions/) certification, starting with a practical guide on how to master T-SQL query writing.

## Part 0: How to Think Like a T-SQL Pro & Write Great Queries

Before diving into advanced DP-800 features, the single biggest skill you need is writing clean, accurate, and fast [Transact-SQL (T-SQL) statements](https://learn.microsoft.com/en-us/sql/t-sql/tutorial-writing-transact-sql-statements?view=sql-server-ver17). Once you understand the mental model below, you can read or write almost any SQL query with confidence.

### 1. The Secret to SQL: How You Write It vs. How SQL Actually Runs It

Most beginners struggle with SQL because they think the database reads a query from top to bottom starting with `SELECT`. **It does not!** According to Microsoft's [Logical Processing Order of the SELECT Statement](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-transact-sql?view=sql-server-ver17), SQL Server runs your query in this exact order:

| Execution Step | Clause | What SQL Server Does at This Step | Common Beginner Mistake Fixed by Knowing This |
| :--- | :--- | :--- | :--- |
| **Step 1** | **[FROM & JOIN](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-select-transact-sql?view=fabric)** | Grabs the source tables and matches rows together based on your `ON` condition. | If you filter a `LEFT JOIN` table in `WHERE` instead of `ON`, you accidentally turn it into an `INNER JOIN`! |
| **Step 2** | **[WHERE](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-transact-sql?view=sql-server-ver17)** | Filters individual rows **before** any grouping or totals are calculated. | You **cannot** use `SUM()`/`COUNT()` or a `SELECT` column alias inside `WHERE` because `SELECT` hasn't happened yet! |
| **Step 3** | **[GROUP BY](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-group-by-transact-sql?view=sql-server-ver17)** | Buckets the remaining rows into summary groups (e.g., one bucket per `CustomerID`). | Once you use `GROUP BY`, every column in `SELECT` must either be in `GROUP BY` or inside an aggregate like `SUM()` or `COUNT()`. |
| **Step 4** | **[HAVING](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-group-by-transact-sql?view=sql-server-ver17)** | Filters the **grouped buckets** after totals are calculated (e.g., `HAVING SUM(Total) > 1000`). | Don't put simple row filters in `HAVING`—filter raw rows early in `WHERE` so SQL groups fewer rows. |
| **Step 5** | **[SELECT](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-transact-sql?view=sql-server-ver17)** | Computes expressions, runs window functions (`OVER`), and creates column aliases (`AS TotalSpent`). | Because `SELECT` runs at Step 5, column aliases created here can **only** be used in Step 6 (`ORDER BY`). |
| **Step 6** | **`ORDER BY` & `TOP`** | Sorts the final output rows and limits how many rows are returned to you ([SELECT (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-transact-sql?view=sql-server-ver17)). | Never put `ORDER BY` inside a view or subquery unless paired with `TOP` or `OFFSET-FETCH`. |

#### Example: Reading a Query in Execution Order (Steps 1 to 6)
```sql
SELECT TOP (5)                                    -- Step 5 & 6: Pick columns, alias them, and take top 5
    c.CustomerID,
    c.CustomerName,
    COUNT(o.OrderID) AS TotalOrders,
    SUM(o.OrderAmount) AS TotalSpent
FROM dbo.Customers AS c                           -- Step 1a: Start with the Customers table
INNER JOIN dbo.Orders AS o                        -- Step 1b: Connect matching rows from Orders
    ON c.CustomerID = o.CustomerID
WHERE o.OrderDate >= '2026-01-01'                 -- Step 2: Keep only 2026 orders (raw row filter)
GROUP BY c.CustomerID, c.CustomerName             -- Step 3: Bucket rows by customer
HAVING SUM(o.OrderAmount) >= 500.00               -- Step 4: Keep only customers who spent at least $500
ORDER BY TotalSpent DESC;                         -- Step 6: Sort highest spenders first (can use alias!)
```

### 2. The 5-Step Mental Recipe to Write Any Query from Scratch

Whenever you face a blank screen or a tough exam question, **do not type `SELECT` first**. Follow this 5-step recipe ([Tutorial: Write Transact-SQL statements](https://learn.microsoft.com/en-us/sql/t-sql/tutorial-writing-transact-sql-statements?view=sql-server-ver17)):

1. **Ask: "Which tables hold the data?" (`FROM` & `JOIN`)**:
   - Write `FROM dbo.MainTable AS m` first. Giving short aliases (`c` for `Customers`, `o` for `Orders`) prevents ambiguous column errors and gives you instant IntelliSense in SSMS/VS Code.
   - Choose the right [JOIN type](https://learn.microsoft.com/en-us/sql/t-sql/queries/from-select-transact-sql?view=fabric):
     - **`INNER JOIN`**: Keep only rows that match in **both** tables (e.g., customers who actually placed orders).
     - **`LEFT JOIN`**: Keep **every row** from the left table, even if the right table has no match (missing right-side columns become `NULL`).
2. **Ask: "Which raw rows do I want to throw away immediately?" (`WHERE`)**:
   - Add your `WHERE` conditions to shrink the dataset as early as possible.
3. **Ask: "Do I want one summary row per group, or do I want to keep every row?" (`GROUP BY` vs. `OVER`)**:
   - If the question asks for *"one row per category with total sales"*, use [`GROUP BY Category`](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-group-by-transact-sql?view=sql-server-ver17).
   - If the question asks to *"show every order alongside the customer's running total or rank"*, keep the detail rows and use a **Window Function** (`SUM(Amount) OVER (PARTITION BY CustomerID)`).
4. **Ask: "Do I need to filter the summary totals?" (`HAVING`)**:
   - Use `HAVING` only when filtering on an aggregate like `COUNT(*) > 3` or `AVG(Price) > 50`.
5. **Fill in the `SELECT` columns and `ORDER BY` last**:
   - Now that your tables and aliases are in place, go back to the top and list the exact columns you want to return.

#### Example: Classic `INNER JOIN` vs. `LEFT JOIN` Pattern
```sql
-- Pattern A: Find customers AND their orders (only customers who bought something)
SELECT c.CustomerID, c.CustomerName, o.OrderID
FROM dbo.Customers AS c
INNER JOIN dbo.Orders AS o ON c.CustomerID = o.CustomerID;

-- Pattern B: Find customers who have NEVER placed an order (Classic Exam Pattern!)
SELECT c.CustomerID, c.CustomerName
FROM dbo.Customers AS c
LEFT JOIN dbo.Orders AS o ON c.CustomerID = o.CustomerID
WHERE o.OrderID IS NULL; -- Only rows where no matching order existed!
```

### 3. Seven Golden Habits of High-Performance T-SQL Developers

1. **Write SARGable Queries (Never wrap an indexed column in a function in `WHERE`)**:
   - **SARGable** stands for *Search Argument Able*. If you put a function around a table column in your `WHERE` clause (like `YEAR(OrderDate) = 2026` or `UPPER(LastName) = 'SMITH'`), SQL Server **cannot seek on your index**—it is forced to scan every single row in the table ([SELECT (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-transact-sql?view=sql-server-ver17)).
   - Always keep the column clean on the left side of the operator and put the calculation on the right side!
2. **Think in Sets, Not Row-by-Row Loops**:
   - Avoid `WHILE` loops and cursors when updating or inserting data. A single `UPDATE` with a `CASE` expression or a window function processes 100,000 rows in milliseconds.
3. **Break Big Problems into Small Steps with CTEs (`WITH`)**:
   - Instead of writing one giant, confusing query with nested subqueries, use [Common Table Expressions (`WITH`)](https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql?view=sql-server-ver17) to build your query like Lego blocks—step 1 prepares the data, and step 2 queries step 1.
4. **Handle `NULL` Properly (`IS NULL` and `COALESCE`)**:
   - In SQL, `NULL` means "unknown." `WHERE Price = NULL` will **never** return any rows! Always write `WHERE Price IS NULL` or `WHERE Price IS NOT NULL`, and use `COALESCE(Discount, 0)` to turn `NULL` into a safe default value during math (`100 + NULL` equals `NULL`!).
5. **Prefer `EXISTS` over `IN` (and Never Use `NOT IN` on Nullable Columns)**:
   - `WHERE EXISTS (SELECT 1 FROM ...)` stops scanning as soon as it finds a single match. Even more importantly, `WHERE ID NOT IN (SELECT ID FROM ...)` returns **zero rows** if the subquery contains even one `NULL` value—always use `WHERE NOT EXISTS (...)` instead ([SELECT (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/queries/select-transact-sql?view=sql-server-ver17)).
6. **Never Use `SELECT *` in Production Code**:
   - Returning unneeded columns wastes memory, breaks covering indexes, and pulls massive `JSON` or `VECTOR(1536)` columns across the network unnecessarily.
7. **Match Data Types in Comparisons**:
   - Comparing an `NVARCHAR` parameter to a `VARCHAR` column causes an **implicit conversion** that can ruin index performance. Match your parameter types (`@Code VARCHAR(20)`) to the exact table column type.

#### Example: Bad (Slow) vs. Good (Fast) Query Writing Side-by-Side
```sql
-- ❌ BAD QUERY: Non-SARGable functions on columns, SELECT *, and unsafe NULL math
SELECT *
FROM dbo.Orders
WHERE YEAR(OrderDate) = 2026                -- Scans the whole table! Cannot use index on OrderDate
  AND LEFT(CustomerCode, 3) = 'VIP'         -- Scans the whole table! Cannot use index on CustomerCode
  AND SalesRepID NOT IN (SELECT RepID FROM dbo.InactiveReps); -- Returns 0 rows if any RepID is NULL!

-- ✅ GOOD QUERY: SARGable index seeks, explicit columns, safe COALESCE, and NOT EXISTS
SELECT 
    o.OrderID,
    o.CustomerCode,
    o.OrderDate,
    o.SubTotal + COALESCE(o.TaxAmount, 0.00) AS FinalTotal -- Safe math even if TaxAmount is NULL
FROM dbo.Orders AS o
WHERE o.OrderDate >= '2026-01-01'           -- Fast Index Seek! Column is untouched
  AND o.OrderDate <  '2027-01-01'
  AND o.CustomerCode LIKE 'VIP%'            -- Fast Index Seek! Trailing wildcard uses the index
  AND NOT EXISTS (                          -- Safe with NULLs and stops at first match
      SELECT 1 
      FROM dbo.InactiveReps AS r 
      WHERE r.RepID = o.SalesRepID
  );
```

## Part 1: Database Design, Advanced T-SQL, and JSON (35–40% of Exam)

### 1. Choosing the Right Table Type (The Foundation)

Think of table types as tools in a toolbox. On the [DP-800 exam](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800), look for the keyword in the question scenario and match it to the right table:

| Table Type | When to Choose It (Exam Keywords) | Simple Explanation |
| :--- | :--- | :--- |
| **[Temporal Table](https://learn.microsoft.com/sql/relational-databases/tables/temporal-tables)** | Point-in-time history, audit trail, "what was the price last Tuesday?", trend analysis | Automatically saves old versions of every updated or deleted row into a hidden history table with timestamps—no triggers required. |
| **[Ledger Table](https://learn.microsoft.com/sql/relational-databases/security/ledger/ledger-overview)** | Tamper-evident, cryptographically verifiable, prove to auditors nobody secretly altered records, financial transactions | Uses blockchain-style cryptography so if even a database administrator tampers with a row, verification will catch it. |
| **Graph Table** | Many-to-many relationships, social networks, fraud rings, supply chains, traversing connections | Uses **Node** tables (things, like People or Products) and **Edge** tables (relationships, like Bought or Knows) queried with `MATCH` ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)). |
| **In-Memory OLTP Table** | Extreme low latency, high throughput, thousands of fast concurrent inserts per second, lock-free | Lives completely in RAM (`MEMORY_OPTIMIZED = ON`) so transactions never wait on disk locks ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)). |
| **External Table** | Query Parquet/CSV files sitting in Azure Data Lake / Blob Storage without moving or copying them | Lets SQL read files directly from cloud storage as if they were a local table (`CREATE EXTERNAL TABLE`) ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)). |

#### Example: Temporal Table (Automatic History + `FOR SYSTEM_TIME`)
Every time someone updates or deletes a row in a [Temporal Table](https://learn.microsoft.com/sql/relational-databases/tables/temporal-tables), SQL Server moves the old version into the history table automatically.

```sql
-- 1. Create a Temporal Table with automatic history
CREATE TABLE dbo.Products (
    ProductID INT PRIMARY KEY,
    ProductName NVARCHAR(100) NOT NULL,
    Price DECIMAL(10,2) NOT NULL,
    ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL,
    ValidTo   DATETIME2 GENERATED ALWAYS AS ROW END HIDDEN NOT NULL,
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
)
WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.ProductsHistory));

-- 2. Query what the price was at an exact moment in the past
SELECT ProductID, ProductName, Price
FROM dbo.Products
FOR SYSTEM_TIME AS OF '2026-10-01T12:00:00';
```

**Know the 5 `FOR SYSTEM_TIME` Time-Window Clauses:**
- `AS OF <date_time>`: Shows what the row looked like at one exact point in time ([Temporal Tables](https://learn.microsoft.com/sql/relational-databases/tables/temporal-tables)).
- `FROM <start> TO <end>`: Returns rows that were active during the window (excludes rows that started exactly at `<end>`) ([Temporal Tables](https://learn.microsoft.com/sql/relational-databases/tables/temporal-tables)).
- `BETWEEN <start> AND <end>`: Same as `FROM...TO`, except it **includes** rows that became active at `<end>` ([Temporal Tables](https://learn.microsoft.com/sql/relational-databases/tables/temporal-tables)).
- `CONTAINED IN (<start>, <end>)`: Only returns rows that **both started and ended** completely inside that window ([Temporal Tables](https://learn.microsoft.com/sql/relational-databases/tables/temporal-tables)).
- `ALL`: Returns every row from both the current table and the history table combined ([Temporal Tables](https://learn.microsoft.com/sql/relational-databases/tables/temporal-tables)).

#### Example: Ledger Table (`WITH (LEDGER = ON)`)
Use [Ledger Tables](https://learn.microsoft.com/sql/relational-databases/security/ledger/ledger-overview) when you need cryptographic proof that data hasn't been tampered with.

```sql
-- Create an updatable ledger table for bank balances
CREATE TABLE dbo.AccountBalances (
    AccountID INT PRIMARY KEY,
    CustomerName NVARCHAR(100) NOT NULL,
    Balance DECIMAL(18,2) NOT NULL
)
WITH (SYSTEM_VERSIONING = ON, LEDGER = ON);

-- Verify that no one has secretly tampered with the ledger
EXECUTE sp_verify_database_ledger_from_digest_storage;
```

#### Example: Graph Tables (`AS NODE`, `AS EDGE`, and `MATCH`)
Use Graph tables when your queries traverse relationships (e.g., "Who is friends with Alice?").

```sql
-- 1. Create Node and Edge tables
CREATE TABLE dbo.Person (PersonID INT PRIMARY KEY, Name NVARCHAR(50)) AS NODE;
CREATE TABLE dbo.Friends (SinceDate DATE) AS EDGE;

-- 2. Find all friends of Alice using MATCH
SELECT Friend.Name AS FriendName
FROM dbo.Person AS Person1, dbo.Friends, dbo.Person AS Friend
WHERE MATCH(Person1-(Friends)->Friend)
  AND Person1.Name = 'Alice';
```

### 2. Data Integrity: Constraints and Sequences vs. Identity

> **Exam Tip:** Don't confuse `CHECK` and `DEFAULT` constraints! A **`DEFAULT` constraint** fills in a fallback value when an `INSERT` leaves a column out. A **`CHECK` constraint** validates whether a value obeys a rule (like `Price > 0`) and rejects the row if it fails ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).

- **`PRIMARY KEY`**: Guarantees every row has a unique identifier, never allows `NULL`, and automatically creates a unique index so you can find any row fast ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **`FOREIGN KEY`**: Connects a child table (`Orders`) to a parent table (`Customers`). It prevents "orphan" rows by ensuring every `CustomerID` in `Orders` actually exists in `Customers` ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **`UNIQUE`**: Ensures no two rows have the same value in a column (such as `Email`), even though it is not the primary key ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **`CHECK`**: Enforces a rule on values entering a column (for example, `Quantity > 0`). If the rule is false, the insert or update fails ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **`DEFAULT`**: Automatically fills in a value when an `INSERT` statement leaves that column out (for example, setting `Status` to `'Pending'`) ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).

#### Example: All 5 Constraints Together
```sql
CREATE TABLE dbo.Orders (
    OrderID INT IDENTITY(1,1) PRIMARY KEY,                      -- Unique row ID, auto-generated
    OrderCode NVARCHAR(20) UNIQUE,                              -- No duplicate order codes allowed
    CustomerID INT NOT NULL REFERENCES dbo.Customers(CustomerID), -- Foreign Key to parent table
    Quantity INT NOT NULL CHECK (Quantity > 0),                 -- CHECK: rejects 0 or negative numbers
    OrderStatus NVARCHAR(20) NOT NULL DEFAULT 'Pending'         -- DEFAULT: fills in 'Pending' if omitted
);
```

#### `IDENTITY` vs. `SEQUENCE`: How to Tell Them Apart
- **`IDENTITY(seed, increment)`**: Tied to **one single table column**. You cannot share it across multiple tables, and you cannot get the number before inserting the row ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **`SEQUENCE`**: An **independent database object** (`CREATE SEQUENCE`). You can share one sequence across multiple tables (e.g., `Invoices` and `Receipts` sharing one numbering stream), restart or cycle it, and grab the next number **before** inserting using `NEXT VALUE FOR` ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).

```sql
-- Create a Sequence starting at 1000 and going up by 1
CREATE SEQUENCE dbo.InvoiceSeq AS INT START WITH 1000 INCREMENT BY 1;

-- Use the sequence in an INSERT statement
INSERT INTO dbo.Invoices (InvoiceNumber, Amount)
VALUES (NEXT VALUE FOR dbo.InvoiceSeq, 250.00);
```

### 3. The T-SQL Engine: Views, Procedures, Triggers, CTEs, Window Functions & Error Handling

- **View (`CREATE VIEW`)**: A saved `SELECT` query that acts like a virtual table so you don't have to rewrite complex joins repeatedly ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **Indexed View**: A regular view doesn't store data on disk, but if you add `WITH SCHEMABINDING` and create a unique clustered index on the view, SQL materializes and stores the aggregated results on disk. Great for heavy dashboards and `SUM`/`COUNT` aggregations ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **Stored Procedure (`CREATE PROCEDURE`)**: Pre-compiled, parameter-driven T-SQL code that can accept inputs, modify tables, handle errors, and return results ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **Triggers (`CREATE TRIGGER`)**: Code that runs automatically when an `INSERT`, `UPDATE`, or `DELETE` happens on a table. **Exam warning:** Triggers run **inside** the write transaction! If you put slow logic (like calling an AI model or heavy queries) inside a trigger, every insert slows down and locks the table longer ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **CTE (Common Table Expression)**: A temporary named result set ([`WITH SalesCTE AS (...)`](https://learn.microsoft.com/en-us/sql/t-sql/queries/with-common-table-expression-transact-sql?view=sql-server-ver17)) that exists only for one query. It makes complex queries easy to read from top to bottom and supports recursion.
- **Window Functions (`OVER (PARTITION BY ... ORDER BY ...)`)**: Let you calculate rankings (`ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`), running totals (`SUM() OVER(...)`), or previous/next row values (`LAG()`, `LEAD()`) **without collapsing the rows** like `GROUP BY` does ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **Correlated Subquery**: An inner query that depends on a column from the outer query and runs logically per outer row ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).

#### Example: CTE + Window Function
```sql
WITH RankedOrders AS (
    SELECT 
        CustomerID,
        OrderID,
        OrderTotal,
        -- Rank each customer's orders from highest to lowest without grouping rows away
        ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY OrderTotal DESC) AS OrderRank
    FROM dbo.Orders
)
SELECT CustomerID, OrderID, OrderTotal
FROM RankedOrders
WHERE OrderRank = 1; -- Get each customer's single biggest order
```

#### Example: Safe Error Handling (`TRY...CATCH`, `SET XACT_ABORT ON`, and `THROW`)
Always pair `BEGIN TRY...BEGIN CATCH` with `SET XACT_ABORT ON` so that any runtime error automatically aborts an open transaction cleanly, and use `THROW;` to re-raise the exact original error ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).

```sql
CREATE PROCEDURE dbo.TransferFunds
    @FromAccount INT,
    @ToAccount INT,
    @Amount DECIMAL(18,2)
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON; -- Automatically roll back on severe errors

    BEGIN TRY
        BEGIN TRANSACTION;
            UPDATE dbo.Accounts SET Balance = Balance - @Amount WHERE AccountID = @FromAccount;
            UPDATE dbo.Accounts SET Balance = Balance + @Amount WHERE AccountID = @ToAccount;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW; -- Re-raises the original error number and message to the caller
    END CATCH
END;
```

### 4. Native JSON in SQL (JSON IN vs. JSON OUT)

> **Exam Tip:** Keep the direction straight! **`JSON_OBJECT` builds/creates JSON** from SQL columns (JSON OUT). To **read** values out of existing JSON (JSON IN), use **`JSON_VALUE`** (for a single text/number value), **`JSON_QUERY`** (for a nested JSON object or array), or **`OPENJSON`** (to turn JSON into table rows and columns) ([Work with JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)).

- **Native `JSON` Data Type**: In Azure SQL, SQL in Fabric, and SQL Server 2025, store JSON in the native `JSON` column type (not plain `NVARCHAR(MAX)`). It automatically validates the JSON and stores it in an optimized binary format for fast reads ([Work with JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)).
- **JSON IN (Reading JSON into SQL)**:
  - `JSON_VALUE(column, '$.path')`: Extracts one scalar value (like a string or number) ([Work with JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)).
  - `JSON_QUERY(column, '$.path')`: Extracts a nested JSON object `{...}` or array `[...]` ([Work with JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)).
  - [OPENJSON](https://learn.microsoft.com/en-us/sql/t-sql/functions/openjson-transact-sql?view=sql-server-ver17): Shreds a JSON document into relational rows and columns so you can `INSERT` or `JOIN` it.
- **JSON OUT (Turning SQL Rows/Columns into JSON)**:
  - [JSON_OBJECT('key': value, ...)](https://learn.microsoft.com/en-us/sql/t-sql/functions/json-object-transact-sql?view=sql-server-ver17): Builds a clean `{ "key": "value" }` JSON object without messy string concatenation.
  - `JSON_ARRAY(val1, val2)` and `JSON_ARRAYAGG(column)`: Builds a JSON array `[...]` from values or across multiple rows ([Work with JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)).
  - `FOR JSON AUTO`: Automatically formats a `SELECT` result as JSON based on table structure (fast, less control) ([Work with JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)).
  - `FOR JSON PATH`: Gives you full control over JSON nesting using dot-separated column aliases like `AS [Customer.Name]`. Add `WITHOUT_ARRAY_WRAPPER` if you want a single JSON object `{...}` without outer `[...]` brackets ([Work with JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)).

#### Example: JSON IN and JSON OUT
```sql
-- 1. Create a table using the native JSON data type
CREATE TABLE dbo.CustomerProfiles (
    CustomerID INT PRIMARY KEY,
    ProfileData JSON NOT NULL
);

-- 2. JSON IN: Read a single field with JSON_VALUE and shred an array with OPENJSON
SELECT 
    c.CustomerID,
    JSON_VALUE(c.ProfileData, '$.city') AS City,
    j.Tag
FROM dbo.CustomerProfiles AS c
CROSS APPLY OPENJSON(c.ProfileData, '$.interests') 
    WITH (Tag NVARCHAR(50) '$') AS j
WHERE JSON_CONTAINS(c.ProfileData, 'Running', '$.interests') = 1;

-- 3. JSON OUT: Build a JSON object and array from regular SQL columns
SELECT 
    JSON_OBJECT(
        'customerId': c.CustomerID,
        'orders': JSON_ARRAYAGG(o.OrderID)
    ) AS CustomerPayload
FROM dbo.Customers AS c
JOIN dbo.Orders AS o ON c.CustomerID = o.CustomerID
GROUP BY c.CustomerID;
```

#### Indexing JSON Columns (Classic vs. Native JSON Index)
- **Classic Index (Computed Column)**: When JSON is stored in `NVARCHAR(MAX)`, add a computed column `AS JSON_VALUE(Info, '$.Email')` and create a standard B-tree index on that computed column ([Work with JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server?view=sql-server-ver17)).
- **Native JSON Index + `JSON_CONTAINS`**: On a native `JSON` column, create a `JSON INDEX` to index paths inside the document and pair it with `JSON_CONTAINS` (which returns `1` when a value exists at a given JSON path) ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

### 5. Pattern Matching (Regex) and Fuzzy String Matching

**Exam Rule of Thumb:** Always pick the **lightest search tool** that solves the problem:
1. **Exact or simple wildcard**: `=` or `LIKE`
2. **Keywords in documents**: [Full-Text Search (`CONTAINS`, `FREETEXT`)](https://learn.microsoft.com/sql/relational-databases/search/full-text-search)
3. **Structured text pattern (emails, phone numbers, codes)**: Regular Expressions (`REGEXP_*`)
4. **Typos and misspelled names**: Fuzzy Matching (`EDIT_DISTANCE`, `JARO_WINKLER_SIMILARITY`)
5. **Searching by meaning/concept**: Vector Search (`VECTOR_SEARCH`)

> **Exam Tips:**
> 1. **`REGEXP_INSTR` and `REGEXP_COUNT` are two separate functions**: `REGEXP_INSTR` returns the position where a match appears, while `REGEXP_COUNT` returns how many times it matches ([REGEXP_LIKE & Regex Functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-like-transact-sql?view=sql-server-ver17)).
> 2. **[EDIT_DISTANCE](https://learn.microsoft.com/en-us/sql/t-sql/functions/edit-distance-transact-sql?view=sql-server-ver17) returns the exact integer count of character edits** (insertions, deletions, substitutions, transpositions) needed to turn string A into string B (`0` means identical). If you want a **0 to 100 similarity score**, call **[EDIT_DISTANCE_SIMILARITY](https://learn.microsoft.com/en-us/sql/t-sql/functions/edit-distance-similarity-transact-sql?view=sql-server-ver17)**!

| Function | What It Returns | Example Use Case |
| :--- | :--- | :--- |
| **[REGEXP_LIKE(text, pattern)](https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-like-transact-sql?view=sql-server-ver17)** | Boolean (`true`/`false`) | Use in `WHERE` or `CHECK` constraints to validate an email or postal code format. |
| **`REGEXP_SUBSTR(text, pattern)`** | Matching substring | Pull the domain name or order number out of a raw log string ([REGEXP_LIKE & Regex Functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-like-transact-sql?view=sql-server-ver17)). |
| **`REGEXP_REPLACE(text, pattern, replacement)`** | Modified string | Mask or strip out phone numbers from free-text comments ([REGEXP_LIKE & Regex Functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-like-transact-sql?view=sql-server-ver17)). |
| **`REGEXP_INSTR(text, pattern)`** | Integer position | Find the character index where a pattern appears ([REGEXP_LIKE & Regex Functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-like-transact-sql?view=sql-server-ver17)). |
| **`REGEXP_COUNT(text, pattern)`** | Integer count | Count how many times a hashtag or pattern appears in a string ([REGEXP_LIKE & Regex Functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/regexp-like-transact-sql?view=sql-server-ver17)). |
| **[EDIT_DISTANCE(s1, s2)](https://learn.microsoft.com/en-us/sql/t-sql/functions/edit-distance-transact-sql?view=sql-server-ver17)** | Integer count of edits (lower = closer) | `'Jon'` vs `'John'` returns `1` (one character insertion). |
| **[EDIT_DISTANCE_SIMILARITY(s1, s2)](https://learn.microsoft.com/en-us/sql/t-sql/functions/edit-distance-similarity-transact-sql?view=sql-server-ver17)** | `0` to `100` score (higher = closer) | Normalized edit similarity where `100` is an exact match. |
| **[JARO_WINKLER_SIMILARITY(s1, s2)](https://learn.microsoft.com/en-us/sql/t-sql/functions/jaro-winkler-similarity-transact-sql?view=sql-server-ver17)** | `0.0` to `1.0` similarity score | Best for matching short strings like person names or city names where the beginning of the word matches. |

#### Example: Regex and Fuzzy Matching in T-SQL
```sql
-- 1. Validate email format with REGEXP_LIKE and extract domain with REGEXP_SUBSTR
SELECT 
    Email,
    REGEXP_SUBSTR(Email, '@(.+)$') AS DomainPart,
    REGEXP_COUNT(Notes, '#[A-Za-z0-9_]+') AS HashtagCount
FROM dbo.Contacts
WHERE REGEXP_LIKE(Email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

-- 2. Find misspelled customer names ("Smyth" vs "Smith")
SELECT 
    CustomerName,
    EDIT_DISTANCE(CustomerName, 'Smith') AS EditsNeeded,
    EDIT_DISTANCE_SIMILARITY(CustomerName, 'Smith') AS EditSimilarityScore,
    JARO_WINKLER_SIMILARITY(CustomerName, 'Smith') AS JaroWinklerScore
FROM dbo.Customers
WHERE EDIT_DISTANCE(CustomerName, 'Smith') <= 2;
```

### 6. AI-Assisted Authoring (GitHub Copilot & MCP Setup)

- **[GitHub Copilot in SSMS](https://learn.microsoft.com/ssms/github-copilot/overview) & VS Code**: Uses your active database connection's schema metadata (tables, columns, types) to write context-aware T-SQL, explain execution plans, and fix syntax errors.
- **Custom Instructions (`.github/copilot-instructions.md`)**: Store your team's T-SQL naming conventions, formatting rules, and security standards in `.github/copilot-instructions.md` so Copilot automatically follows them ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **MCP Configuration (`.vscode/mcp.json`)**: Add a server entry pointing to your SQL MCP endpoint so AI assistants in VS Code can inspect schemas and query allowed tables safely ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **Golden Rule**: Copilot respects your database permissions, but **always review AI-generated T-SQL** in a test environment before running it in production ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

## Part 2: Securing, Optimizing, and Deploying Database Solutions (35–40% of Exam)

### 1. Data Protection: Always Encrypted vs. Dynamic Data Masking vs. Row-Level Security

> **Exam Tip:** Keep these three security features strictly separate on the exam—know **who** each feature restricts and **where** the data is protected:

| Security Feature | Who Is Restricted? | How It Works |
| :--- | :--- | :--- |
| **[Always Encrypted](https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-database-engine)** | **Database Administrators (DBAs)** and cloud admins | The client driver encrypts data **before** it leaves the app. SQL Server only ever sees ciphertext (`in rest, in transit, and in memory`). Uses two keys: a **Column Master Key (CMK)** stored outside SQL (e.g., in Azure Key Vault) which protects the **Column Encryption Key (CEK)** inside SQL. |
| **[Dynamic Data Masking (DDM)](https://learn.microsoft.com/en-us/sql/relational-databases/security/dynamic-data-masking?view=sql-server-ver17)** | **Regular app users / support agents** (NOT admins) | Data is stored in plain text in the database, but query results are disguised on the fly (e.g., `xxxx-xxxx-xxxx-1234`) unless the user has the `UNMASK` permission. |
| **[Row-Level Security (RLS)](https://learn.microsoft.com/sql/relational-databases/security/row-level-security)** | **Multi-tenant users / regional managers** (horizontal filtering) | Uses an inline table-valued function (predicate) and a `SECURITY POLICY` so each user or tenant only sees or modifies **their own rows** (`FILTER PREDICATE` and `BLOCK PREDICATE`). |

#### Example: Dynamic Data Masking
```sql
-- 1. Create a table with masked columns
CREATE TABLE dbo.Employees (
    EmpID INT IDENTITY(1,1) PRIMARY KEY,
    FullName NVARCHAR(100) NOT NULL,
    Salary DECIMAL(10,2) MASKED WITH (FUNCTION = 'random(500, 1500)') NULL,
    Email NVARCHAR(100) MASKED WITH (FUNCTION = 'email()') NULL,
    CardNumber VARCHAR(19) MASKED WITH (FUNCTION = 'partial(0, "XXXX-XXXX-XXXX-", 4)') NULL
);

-- 2. Create a low-privilege test user and grant SELECT
CREATE USER MaskedViewer WITHOUT LOGIN;
GRANT SELECT ON dbo.Employees TO MaskedViewer;

-- 3. Test what MaskedViewer sees (masked values)
EXECUTE AS USER = 'MaskedViewer';
SELECT FullName, Salary, Email, CardNumber FROM dbo.Employees;
REVERT;

-- 4. Let a Finance user see the real unmasked values
GRANT UNMASK TO FinanceAnalystRole;
```

#### Example: Row-Level Security (RLS)
```sql
-- 1. Create a filter function that checks if the row's TenantID matches the logged-in session
CREATE FUNCTION Security.fn_TenantFilter(@TenantID AS INT)
    RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_TenantFilter_Result
    WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS INT);
GO

-- 2. Attach the function to the Orders table using a Security Policy
CREATE SECURITY POLICY Security.TenantPolicy
    ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID) ON dbo.Orders,
    ADD BLOCK PREDICATE Security.fn_TenantFilter(TenantID) ON dbo.Orders AFTER INSERT;
```

### 2. Permissions, Passwordless Identity, and Auditing

- **Permissions Hierarchy**:
  - `GRANT`: Gives permission (`SELECT`, `INSERT`, `EXECUTE`) on a schema, table, or specific columns (`GRANT SELECT (Name, Department) ON dbo.Employees TO Analyst;`).
  - `REVOKE`: Removes a previously granted or denied permission.
  - `DENY`: Explicitly blocks access. **Exam rule: `DENY` always overrides `GRANT`!** ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **Cell-Level Encryption (`ENCRYPTBYKEY`)**: Encrypts specific cells inside T-SQL using a symmetric key protected by a certificate (`OPEN SYMMETRIC KEY ... DECRYPTION BY CERTIFICATE ...`) ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **Passwordless Identity**: Never hardcode passwords in connection strings or scripts. Use **Microsoft Entra Managed Identity** for apps, **Azure Key Vault** for secrets/keys, and **`CREATE DATABASE SCOPED CREDENTIAL`** when SQL Server needs to authenticate to external endpoints or storage ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **Azure SQL Auditing**: Sends audit logs of who accessed or changed what to **Azure Monitor Log Analytics**, **Azure Storage**, or **Azure Event Hubs** ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).

### 3. Diagnosing Performance: Query Store, DMVs, Plans & Parameter Sniffing

Memorize this 3-part rule from the [Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk):
- **History ("A query got slower after Tuesday's deployment")** → Use **[Query Store](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)**. It records query history and execution plans over time, powers **Query Performance Insight** (top queries by CPU/duration), and lets you **force the last known good plan (`sp_query_store_force_plan`) without changing any application code**.
- **Right Now ("What is blocking or running at this exact second?")** → Use **Dynamic Management Views (DMVs)** like `sys.dm_exec_requests`, `sys.dm_exec_sessions`, and `sys.dm_os_wait_stats` ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **One Specific Slow Query ("Why is this single query scanning the whole table?")** → Inspect the **Execution Plan** to spot missing indexes, table scans, or expensive key lookups ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

#### Parameter Sniffing: What It Is and How to Fix It
When a stored procedure runs for the very first time, SQL Server compiles and caches an execution plan tailored to that **first parameter value**. If the first run had 2 rows, SQL caches a nested-loops plan; if the next run has 2 million rows, that cached plan runs terribly slowly.
- **Fix 1 (No code change)**: Force the good plan in [Query Store](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store).
- **Fix 2 (Compile a fresh plan every run)**: Add `OPTION (RECOMPILE)` to the query.
- **Fix 3 (Optimize for a typical value)**: Add `OPTION (OPTIMIZE FOR (@CustomerID = 100))` or `OPTION (OPTIMIZE FOR UNKNOWN)`.

### 4. Isolation Levels, Concurrency, Columnstore, and Partitioning

> **Exam Tip:** Understand the difference between **Read Committed Snapshot Isolation (RCSI)** and **Snapshot Isolation**:
> - **Read Committed Snapshot Isolation (RCSI)**: The **default in Azure SQL Database and SQL in Fabric**. When someone updates a row, SQL keeps the old committed version in `tempdb`. Readers automatically read the last committed version **without waiting or blocking writers**, and writers don't block readers ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
> - **Snapshot Isolation (`SET TRANSACTION ISOLATION LEVEL SNAPSHOT`)**: Gives the entire transaction a frozen snapshot of data as of the moment the transaction started. If **two transactions try to update the exact same row** at the same time, the second one fails with an **update conflict error** and rolls back to prevent lost updates ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

- **Blocking vs. Deadlocks**:
  - **Blocking**: Transaction A holds a lock on a row, so Transaction B has to wait in line until Transaction A commits or rolls back.
  - **Deadlock**: Transaction A locks Table 1 and waits for Table 2, while Transaction B locks Table 2 and waits for Table 1. SQL Server detects the cycle and kills one transaction as the "deadlock victim" (error 1205).
- **Columnstore Index (`CREATE CLUSTERED COLUMNSTORE INDEX`)**: Stores data **by column** instead of by row using high compression. Best for analytics dashboards and scanning millions of rows to calculate sums or averages ([DP-800 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800)).
- **Table Partitioning**: Splits a huge table into horizontal slices (e.g., by `OrderYear` or `OrderMonth`). Queries that filter on the partition column jump straight to the needed partition (**partition elimination**), and you can archive old months in milliseconds using **partition switching (`ALTER TABLE ... SWITCH PARTITION`)** ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

### 5. CI/CD Deployment: SQL Projects, DACPACs, Testing, and Passwordless GitHub Actions

- **[SQL Database Projects (`.sqlproj`) & DACPAC](https://learn.microsoft.com/sql/tools/sql-database-projects/sql-database-projects)**:
  - **State-based (Declarative — DACPAC)**: You define what the database *should* look like in an SDK-style SQL project (`Microsoft.Build.Sql`). Building the project creates a `.dacpac` artifact. At deploy time, `SqlPackage` compares the `.dacpac` to the target database and automatically generates the `ALTER`/`CREATE` script to match.
  - **Migration-based**: You write ordered upgrade scripts (`v1.sql`, `v2.sql`, `v3.sql`) that run one after another.
  - **Reference Data**: Keep static lookup rows (like country codes or status types) in a **post-deployment script** (`Script.PostDeployment.sql` using `MERGE`) inside the SQL project.
- **Quality Gates & Testing**:
  - **Unit Tests (`tSQLt`)**: Test one stored procedure or function in isolation using fake/mocked tables (`tSQLt.FakeTable`) so tests don't depend on production data ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
  - **Integration Tests**: Deploy the `.dacpac` to a temporary container or test database in the CI pipeline and run the full test suite before merging ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
  - **`SqlPackage` Reports before Publish**:
    - `DeployReport`: Shows exactly which tables/objects will be created, altered, or dropped before you actually change production.
    - `DriftReport`: Shows any manual hotfixes or schema edits someone made directly in production outside the CI/CD pipeline ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **Passwordless CI/CD with OIDC (Workload Identity Federation)**:
  - Never store database passwords or long-lived client secrets in GitHub Secrets!
  - Instead, configure **OpenID Connect (OIDC)** between GitHub Actions and Microsoft Entra ID. GitHub requests a short-lived token, Entra verifies the repository and branch identity, and the pipeline deploys the `.dacpac` with a temporary token—nothing to rotate or leak ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

### 6. Tracking Data Changes: Change Tracking vs. CDC vs. Change Event Streaming

This comparison is heavily tested on [Exam DP-800](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800):

| Feature | What It Records | Best Exam Scenario |
| :--- | :--- | :--- |
| **Change Tracking (CT)** | **Only the Primary Key** of rows that changed and the operation (`I`, `U`, `D`)—**no old/intermediate values**. | Lightweight sync! E.g., "Find which product rows changed since the last batch job so you can re-generate their AI embeddings." |
| **Change Data Capture (CDC)** | **Full before-and-after column values** of every single change by reading the transaction log asynchronously. | Historical data warehouse ETL pipelines or audit trails that need to see both the old value and the new value. |
| **Change Event Streaming (CES)** | Streams row changes **as JSON events directly to Azure Event Hubs** in near real time. | Event-driven real-time downstream apps or real-time AI pipelines reacting immediately to database changes. |

## Part 3: Vector Search, Embeddings, and Indexing (25–30% of Exam)

### 1. Registering and Managing External AI Models in T-SQL

Instead of pulling data out of SQL Server just to generate embeddings, you register the AI model endpoint inside the database using [CREATE EXTERNAL MODEL](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql?view=sql-server-ver17) and generate vectors directly in T-SQL with [AI_GENERATE_EMBEDDINGS](https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql?view=sql-server-ver17).

> **Exam Tip:** Watch your spelling on fill-in-the-code exam questions:
> - Function name is **`AI_GENERATE_EMBEDDINGS`** (plural, with an `S` at the end and an underscore after `AI`).
> - Data type is **`VECTOR(1536)`**.

#### Example: Register, Use, Alter, and Drop an External Model
```sql
-- 1. Store the API key or Managed Identity in a Database Scoped Credential
CREATE DATABASE SCOPED CREDENTIAL [https://my-ai-foundry.openai.azure.com/]
WITH IDENTITY = 'Managed Identity',
     SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';

-- 2. Register the External Model in the database
CREATE EXTERNAL MODEL EmbedModel
WITH (
    LOCATION = 'https://my-ai-foundry.openai.azure.com/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-06-01',
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL = 'text-embedding-3-small',
    CREDENTIAL = [https://my-ai-foundry.openai.azure.com/]
);

-- 3. Add a native VECTOR(1536) column and populate embeddings for all products
ALTER TABLE dbo.Products ADD Embedding VECTOR(1536) NULL;

UPDATE dbo.Products
SET Embedding = AI_GENERATE_EMBEDDINGS(Description USE MODEL EmbedModel);

-- 4. Update the endpoint URL later if you move to a new deployment
ALTER EXTERNAL MODEL EmbedModel
SET (LOCATION = 'https://new-endpoint.openai.azure.com/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-06-01');

-- 5. Remove the model definition if no longer needed
DROP EXTERNAL MODEL EmbedModel;
```

**Important Rules About Embeddings:**
- **Never mix models**: Vectors from two different models (or different dimensions) live in completely different mathematical spaces and **cannot be compared**. Always embed the user's search question using the **exact same model** that embedded the table rows, and if you switch models, you must re-embed every row ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **What to embed vs. what to filter**: Only embed descriptive text (`ProductName`, `Description`, `CategoryName`). Keep exact numbers, IDs, prices, `InStock` flags, and dates in normal SQL columns and filter them with a `WHERE` clause ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

### 2. The `VECTOR` Data Type and Distance Metrics

A [VECTOR(N)](https://learn.microsoft.com/en-us/sql/sql-server/ai/vectors?view=sql-server-ver17) column stores an ordered array of `N` floating-point numbers representing meaning. You pass it in as a JSON-style array string (like `'[0.12, -0.45, 0.88]'`), and SQL stores it in a compact binary format.

- **[VECTOR_DISTANCE('metric', v1, v2)](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql?view=sql-server-ver17)**: Calculates how far apart two vectors are (**smaller distance = more similar**):
  - `'cosine'`: Measures the **angle (direction)** between two vectors, ignoring length. Best default for text embeddings ([Vector Search & Vector Index](https://learn.microsoft.com/en-us/sql/sql-server/ai/vectors?view=sql-server-ver17)).
  - `'euclidean'`: Measures the **straight-line distance** between two points in space. Common for image or spatial embeddings ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
  - `'dot'`: Negative dot product (multiplies matching components and sums them). Fastest calculation when vectors are already normalized to length `1` (like OpenAI embeddings) ([Vector Search & Vector Index](https://learn.microsoft.com/en-us/sql/sql-server/ai/vectors?view=sql-server-ver17)).
- **`VECTOR_NORM(v, 'norm2')`**: Returns the length (magnitude) of a vector ([Vectors in SQL](https://learn.microsoft.com/sql/relational-databases/vectors/vectors-sql-server)).
- **`VECTOR_NORMALIZE(v, 'norm2')`**: Scales a vector so its length becomes exactly `1` ([Vectors in SQL](https://learn.microsoft.com/sql/relational-databases/vectors/vectors-sql-server)).

### 3. Exact Search (`VECTOR_DISTANCE`) vs. Approximate Search (`VECTOR_SEARCH` + DiskANN)

> **Exam Tip:** In `CREATE VECTOR INDEX`, the index `TYPE` is always **`'DiskANN'`** (Microsoft's graph-based Approximate Nearest Neighbor algorithm), **not** `'distance'`: `WITH (METRIC = 'cosine', TYPE = 'DiskANN')` ([CREATE VECTOR INDEX](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-vector-index-transact-sql?view=sql-server-ver17)).

| Search Type | T-SQL Function | How It Works & When to Use |
| :--- | :--- | :--- |
| **Exact Nearest Neighbor (ENN)** | [VECTOR_DISTANCE](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-distance-transact-sql?view=sql-server-ver17) | Compares `@queryVector` against **every single row** (100% recall). Does **not** use a vector index. Use for small tables (under ~50k rows) or when a `WHERE` filter narrows the rows down to a tiny subset first. |
| **Approximate Nearest Neighbor (ANN)** | [VECTOR_SEARCH](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-search-transact-sql?view=sql-server-ver17) + [CREATE VECTOR INDEX](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-vector-index-transact-sql?view=sql-server-ver17) | Uses a **DiskANN** graph index to find the closest vectors in milliseconds across **millions of rows**, trading a tiny fraction of recall for massive speed. |

#### Example: Exact vs. Approximate Vector Search
```sql
-- Embed the customer's question once into a variable
DECLARE @q VECTOR(1536) = AI_GENERATE_EMBEDDINGS(N'quiet kitchen dishwasher' USE MODEL EmbedModel);

-- OPTION A: Exact Search (small table or heavily filtered subset)
SELECT TOP (10) ProductID, ProductName,
       VECTOR_DISTANCE('cosine', @q, Embedding) AS Distance
FROM dbo.Products
WHERE Category = 'Appliances'
ORDER BY Distance ASC;

-- OPTION B: Approximate Search at Scale with DiskANN Vector Index
-- 1. Create the DiskANN vector index
CREATE VECTOR INDEX vix_Products_Embedding
ON dbo.Products(Embedding)
WITH (METRIC = 'cosine', TYPE = 'DiskANN');

-- 2. Query the vector index with VECTOR_SEARCH
SELECT TOP (10) WITH APPROXIMATE
    p.ProductID,
    p.ProductName,
    s.distance
FROM VECTOR_SEARCH(
    TABLE = dbo.Products AS p,
    COLUMN = Embedding,
    SIMILAR_TO = @q,
    METRIC = 'cosine',
    TOP_N = 10
) AS s
WHERE p.InStock = 1
ORDER BY s.distance ASC;
```

### 4. Post-Filtering vs. Iterative Filtering & Hybrid Search (RRF)

- **The Post-Filtering Problem**: Suppose a vector index grabs the top 10 closest products first, and **then** applies `WHERE p.InStock = 1`. If 8 of those 10 products are out of stock, your query only returns 2 rows instead of 10 (or even 0 rows!) ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **Iterative Filtering (Modern Engine Behavior)**: Applies the `WHERE` filter **during** the DiskANN traversal so the engine keeps walking the graph until it collects the full `N` qualifying in-stock rows ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **Hybrid Search with Reciprocal Rank Fusion (RRF)**:
  - **Vector search** is great for **meaning** (e.g., `"waterproof hiking boots"`), while **[Full-Text Search](https://learn.microsoft.com/sql/relational-databases/search/full-text-search)** (`CONTAINS` / `FREETEXTTable`) is great for **exact part numbers or SKUs** (e.g., `"SKU-9942-X"`).
  - **Reciprocal Rank Fusion (RRF)** runs both searches, ranks each list (`1 / (60 + rank)`), and sums the scores so items that rank well in **both** keyword and meaning rise to the very top ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

### 5. Keeping Vectors Fresh When Data Changes

How should you update embeddings when rows are inserted or updated?
1. **Trigger (`AFTER INSERT, UPDATE`)**: Simplest to write, **but slowest for writes** because the database transaction has to wait for the external AI model HTTP call before committing. Only use when writes are rare and embeddings must exist the exact millisecond the row is saved ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
2. **Change Tracking + Scheduled Job**: Turn on Change Tracking, and run a background job every few minutes that re-embeds only the changed rows (`CHANGETABLE(CHANGES ...)`). Keeps `INSERT`/`UPDATE` transactions super fast!
3. **Azure Functions SQL Trigger**: Event-driven serverless function that wakes up automatically when rows change, generates the embedding, and writes the vector back—no polling code required ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
4. **Azure Logic Apps**: Low-code workflow triggered on a schedule or row change (useful when human approvals or notifications are involved).

## Part 4: RAG (Retrieval-Augmented Generation) & External REST Endpoints

### 1. Why RAG? Retrieve, Augment, Generate

Large Language Models (LLMs) don't know your private database and can hallucinate (make up answers). **RAG** solves this in three steps right inside T-SQL ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)):
1. **Retrieve**: Find the most relevant rows or document chunks using `VECTOR_SEARCH`.
2. **Augment**: Pack those retrieved rows as JSON context into a grounded prompt with a clear system rule ("Answer only using the provided context").
3. **Generate**: Send the prompt to Azure OpenAI / Microsoft Foundry using [sp_invoke_external_rest_endpoint](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql) and extract the answer.

### 2. Chunking Long Documents with `AI_GENERATE_CHUNKS`

Long manuals or PDFs are too big to embed as a single vector (meaning gets blurry, and it may exceed token limits). Use [AI_GENERATE_CHUNKS](https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-chunks-transact-sql?view=sql-server-ver17) with `CROSS APPLY` to split one long document into smaller overlapping chunks:
- `CHUNK_SIZE`: Number of **characters** per chunk (too small loses context; too large blurs meaning).
- `OVERLAP`: **Percentage (`0–100`)** of characters shared between adjacent chunks so sentences cut at the boundary don't lose their meaning.

#### Example: Chunk and Embed Documents in One Statement
```sql
INSERT INTO dbo.DocChunks (DocID, ChunkText, Embedding)
SELECT 
    d.DocID,
    c.chunk,
    AI_GENERATE_EMBEDDINGS(c.chunk USE MODEL EmbedModel)
FROM dbo.Documents AS d
CROSS APPLY AI_GENERATE_CHUNKS(
    SOURCE = d.BodyText,
    CHUNK_TYPE = FIXED,
    CHUNK_SIZE = 500, -- 500 characters per chunk
    OVERLAP = 10      -- 10% character overlap between chunks
) AS c;
```

### 3. Calling the LLM with `sp_invoke_external_rest_endpoint`

> **Exam Tip:** Remember the exact procedure name **`sp_invoke_external_rest_endpoint`**, the retry parameter **`@retry_count`**, and the JSON path **`'$.result.choices[0].message.content'`** (with a dot `.` after `$`) ([sp_invoke_external_rest_endpoint](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql)).

#### Complete End-to-End T-SQL RAG Pipeline Example
```sql
DECLARE @question NVARCHAR(500) = N'Which pedals fit the Mountain-500 bike?';
DECLARE @q VECTOR(1536) = AI_GENERATE_EMBEDDINGS(@question USE MODEL EmbedModel);

-- Step 1: Retrieve top 5 matching chunks as a JSON string
DECLARE @context NVARCHAR(MAX) = (
    SELECT TOP (5) WITH APPROXIMATE
        c.ChunkText
    FROM VECTOR_SEARCH(
        TABLE = dbo.DocChunks AS c,
        COLUMN = Embedding,
        SIMILAR_TO = @q,
        METRIC = 'cosine',
        TOP_N = 5
    ) AS s
    ORDER BY s.distance ASC
    FOR JSON PATH
);

-- Step 2: Build the grounded JSON request payload safely with JSON_OBJECT and JSON_ARRAY
DECLARE @systemRule NVARCHAR(500) = N'You are a helpful bike specialist. Answer ONLY using the provided JSON context. If the answer is not in the context, say you do not know.';

DECLARE @payload NVARCHAR(MAX) = JSON_OBJECT(
    'messages': JSON_ARRAY(
        JSON_OBJECT('role': 'system', 'content': @systemRule),
        JSON_OBJECT('role': 'user',   'content': N'Context: ' + @context + N' Question: ' + @question)
    ),
    'max_tokens': 500,
    'temperature': 0.2
);

-- Step 3: Call Azure OpenAI via sp_invoke_external_rest_endpoint
DECLARE @response NVARCHAR(MAX), @returnValue INT;

EXEC @returnValue = sp_invoke_external_rest_endpoint
    @url = N'https://my-ai-foundry.openai.azure.com/openai/deployments/gpt-4o/chat/completions?api-version=2024-06-01',
    @method = 'POST',
    @payload = @payload,
    @credential = [https://my-ai-foundry.openai.azure.com/],
    @retry_count = 3,
    @timeout = 30,
    @response = @response OUTPUT;

-- Step 4: Check return value and parse $.result.choices[0].message.content
IF @returnValue = 0
BEGIN
    SELECT JSON_VALUE(@response, '$.result.choices[0].message.content') AS GroundedAnswer,
           JSON_VALUE(@response, '$.result.usage.total_tokens') AS TokensUsed;
END
ELSE
BEGIN
    SELECT @returnValue AS HttpStatusCode,
           JSON_VALUE(@response, '$.response.status.http.description') AS ErrorReason;
END;
```

**Exam Facts to Memorize for [sp_invoke_external_rest_endpoint](https://learn.microsoft.com/sql/relational-databases/system-stored-procedures/sp-invoke-external-rest-endpoint-transact-sql):**
- **Return Value (`@returnValue`)**:
  - `0` = Success (`2xx` HTTP status code).
  - Non-zero = The actual HTTP status code: **`429`** means **throttled / too many requests**, **`401` / `403`** means **invalid or unauthorized credential**.
- **Response Envelope Structure**:
  - `$.response.status.http.code` → HTTP status code and headers.
  - `$.result` → The body returned by the external API (so for Azure OpenAI chat completions, the text answer is at **`$.result.choices[0].message.content`** and token usage is at **`$.result.usage`**).
- **Required Permission**: `EXECUTE ANY EXTERNAL ENDPOINT` (or `ALTER ANY EXTERNAL ENDPOINT`), and the target URL must be on Microsoft's allow-listed Azure endpoints ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **Production Tip**: Never call `sp_invoke_external_rest_endpoint` row-by-row inside a huge `SELECT` or write trigger—each call holds a SQL worker thread while waiting on the network. Pre-compute embeddings in batches and send **one batched payload per question** ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

## Part 5: Data API Builder (DAB) & SQL MCP Server

Here is the complete, simple breakdown of Data API Builder and SQL MCP Server for [Exam DP-800](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800):

### 1. What Data API Builder (DAB) Does
[Data API builder](https://learn.microsoft.com/azure/data-api-builder/) turns **one single JSON configuration file (`dab-config.json`)** into **REST**, **GraphQL**, and **SQL MCP Server** endpoints in front of your database without writing boilerplate backend controller code:
- **CLI Commands**:
  - `dab init`: Creates `dab-config.json` with your database connection string.
  - `dab add`: Adds an entity (table, view, or stored procedure) and sets role permissions.
  - `dab start`: Runs the API server locally.
- **Entity Rules**:
  - **Tables**: Automatic CRUD (`create`, `read`, `update`, `delete`).
  - **Views**: Must specify `--source.key-fields` so DAB knows how to uniquely identify each row.
  - **Stored Procedures**: Exposed with the `execute` action.
- **REST vs. GraphQL on the Exam**:
  - **REST (`/api/Order`)**: Supports query parameters `$select` (choose columns), `$filter` (WHERE condition), `$orderby` (sorting), and `$first` / `$after` (cursor pagination).
  - **GraphQL (`/graphql`)**: Best when an app needs **related data from multiple tables in a single network call** (for example, fetching an `Order`, its `OrderLines`, and the `Customer` in one request by configuring relationships in `dab-config.json`) ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).
- **Serverless Hosting**: Deploy DAB in a container to **[Azure Container Apps](https://learn.microsoft.com/azure/data-api-builder/deployment/azure-container-apps)** for serverless scale-to-zero hosting and monitor it with **Application Insights**.

#### Example: `dab` CLI Setup
```bash
# 1. Initialize dab-config.json for Azure SQL
dab init --database-type mssql --connection-string "@env('SQL_CONNECTION_STRING')" --host-mode Production

# 2. Expose the Orders table: allow 'anonymous' to read, and 'agent-role' to read & update (but NOT delete!)
dab add Order --source dbo.Orders --permissions "anonymous:read" --permissions "agent-role:read,update"

# 3. Expose a View (requires --source.key-fields)
dab add SalesSummary --source dbo.vw_SalesSummary --source.type "view" --source.key-fields "CategoryID" --permissions "authenticated:read"

# 4. Start the Data API Builder server
dab start
```

### 2. SQL MCP Server (Model Context Protocol for AI Agents)
When you enable MCP in [Data API builder](https://learn.microsoft.com/azure/data-api-builder/) (exposed at the `/mcp` endpoint), AI agents can safely interact with your SQL data using **six built-in structured tools** (`describe_entities`, `read_records`, `create_record`, `update_record`, `delete_record`, `execute_entity`) ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)):
- **Why SQL MCP Server is safer than raw NL-to-SQL**: Agents cannot run arbitrary `DROP TABLE` (no DDL) or raw natural-language SQL strings. They must go through DAB's configured entities, role permissions, field rules, and database policies.
- **Exam Scenario Tip**: If an exam question says *"An AI agent using SQL MCP Server must read and update orders, but must never delete orders,"* the answer is: **Do not grant the `delete` action to the agent's role in `dab-config.json` (or disable the `delete_record` MCP tool)** ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)).

## Quick Exam-Day Cheat Sheet: "If the Question Says X, Pick Y"

| If the Exam Scenario Mentions... | Pick This Answer |
| :--- | :--- |
| "Query past states / prices as of any point in time with minimal code" | [Temporal Table (`SYSTEM_VERSIONING = ON` + `FOR SYSTEM_TIME AS OF`)](https://learn.microsoft.com/sql/relational-databases/tables/temporal-tables) |
| "Cryptographically prove to auditors that records were not secretly altered" | [Ledger Table (`WITH (LEDGER = ON)`)](https://learn.microsoft.com/sql/relational-databases/security/ledger/ledger-overview) |
| "Ensure DBAs and cloud admins can never read sensitive plaintext data" | [Always Encrypted (with Column Master Key in Azure Key Vault)](https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-database-engine) |
| "Hide full credit card/salary values from support agents, let Finance see all" | [Dynamic Data Masking (`MASKED WITH`) + `GRANT UNMASK`](https://learn.microsoft.com/en-us/sql/relational-databases/security/dynamic-data-masking?view=sql-server-ver17) |
| "Ensure each tenant or regional manager only sees their own rows" | [Row-Level Security (`CREATE SECURITY POLICY` + inline TVF)](https://learn.microsoft.com/sql/relational-databases/security/row-level-security) |
| "A query got slower after a deployment; fix it immediately with no code change" | [Query Store (force the last good plan)](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store) |
| "Find which rows changed (primary keys only) to re-embed vectors in batches" | Change Tracking (`CHANGETABLE`) ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)) |
| "Capture both old (before) and new (after) values for an audit/ETL pipeline" | Change Data Capture (CDC) ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)) |
| "Stream row changes as real-time JSON events to Azure Event Hubs" | Change Event Streaming (CES) ([Let's Get Certified: DP-800 Guide](https://drive.google.com/file/d/1wq6Lf_XH1u-YWCx_rATyH0sOjBZtNFAH/view?usp=drivesdk)) |
| "Semantic search over millions of rows with millisecond latency" | [CREATE VECTOR INDEX (`TYPE = 'DiskANN'`)](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-vector-index-transact-sql?view=sql-server-ver17) + [VECTOR_SEARCH](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-search-transact-sql?view=sql-server-ver17) |
| "Combine semantic meaning search with exact keyword matching" | Hybrid Search ([VECTOR_SEARCH](https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-search-transact-sql?view=sql-server-ver17) + [Full-Text Search](https://learn.microsoft.com/sql/relational-databases/search/full-text-search) + Reciprocal Rank Fusion) |
| "Fetch an order, its line items, and customer details in a single API call" | GraphQL relationships in [Data API builder](https://learn.microsoft.com/azure/data-api-builder/) |
| "Let an AI agent safely read/update SQL data without raw NL-to-SQL or DDL" | SQL MCP Server (`/mcp` in [Data API builder](https://learn.microsoft.com/azure/data-api-builder/)) |
