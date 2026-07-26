# SQL Interview Mastery — CTEs, Joins, Advanced Queries

> **Goal:** Walk into HackerRank SQL (Easy → Hard) and big-tech SQL rounds confident.
> **Method:** One memorable data model, **see the data before and after every query**.

---

## How to Read This Guide

**Complete beginner?** Read **SQL Keywords for Complete Beginners** (right after the data snapshot) before the examples. It explains every keyword used in this document.

Every example follows the same pattern:

1. **Input** — what the tables look like (sample rows)
2. **Query** — the SQL
3. **Output** — exact result set you should expect
4. **Why** — one-line interview takeaway

Memorize the **Acme Shop** data model below. Once you can picture these 6 tables, every pattern clicks.

---

## The Story Data Model

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ departments │       │  employees  │       │   customers │
├─────────────┤       ├─────────────┤       ├─────────────┤
│ dept_id PK  │◄──┐   │ emp_id PK   │       │ cust_id PK  │
│ dept_name   │   └───│ dept_id FK  │       │ name        │
└─────────────┘       │ manager_id  │──┐    │ city        │
                      │ name        │  │    │ signup_date │
                      │ salary      │  │    └──────┬──────┘
                      └─────────────┘  │           │
                            ▲          │           │
                            └──────────┘           │
                      (self-ref: manager)          │
                                                   │
┌─────────────┐       ┌─────────────┐       ┌──────▼──────┐
│  products   │       │    orders   │       │ order_items │
├─────────────┤       ├─────────────┤       ├─────────────┤
│ prod_id PK  │◄──┐   │ order_id PK │◄──────│ order_id FK │
│ name        │   │   │ cust_id FK  │       │ prod_id FK  │
│ category    │   └───│ order_date  │       │ quantity    │
│ price       │       │ status      │       │ unit_price  │
└─────────────┘       └─────────────┘       └─────────────┘
```

### DDL + Seed Data (run once to practice)

```sql
CREATE TABLE departments (
    dept_id   INT PRIMARY KEY,
    dept_name VARCHAR(50)
);

CREATE TABLE employees (
    emp_id     INT PRIMARY KEY,
    name       VARCHAR(50),
    dept_id    INT REFERENCES departments(dept_id),
    manager_id INT REFERENCES employees(emp_id),
    salary     DECIMAL(10,2),
    hire_date  DATE
);

CREATE TABLE customers (
    cust_id     INT PRIMARY KEY,
    name        VARCHAR(50),
    city        VARCHAR(50),
    signup_date DATE
);

CREATE TABLE products (
    prod_id  INT PRIMARY KEY,
    name     VARCHAR(50),
    category VARCHAR(30),
    price    DECIMAL(10,2)
);

CREATE TABLE orders (
    order_id   INT PRIMARY KEY,
    cust_id    INT REFERENCES customers(cust_id),
    order_date DATE,
    status     VARCHAR(20)
);

CREATE TABLE order_items (
    order_id   INT REFERENCES orders(order_id),
    prod_id    INT REFERENCES products(prod_id),
    quantity   INT,
    unit_price DECIMAL(10,2),
    PRIMARY KEY (order_id, prod_id)
);

INSERT INTO departments VALUES
(1,'Engineering'), (2,'Sales'), (3,'HR');

INSERT INTO employees VALUES
(1,'Alice',1,NULL,150000,'2018-01-15'),
(2,'Bob',1,1,120000,'2019-03-01'),
(3,'Carol',1,1,110000,'2020-06-10'),
(4,'Dave',2,1,95000,'2019-08-20'),
(5,'Eve',2,4,85000,'2021-02-14'),
(6,'Frank',3,1,80000,'2020-11-01');

INSERT INTO customers VALUES
(101,'Acme Corp','NYC','2022-01-10'),
(102,'Beta LLC','SF','2022-03-15'),
(103,'Gamma Inc','NYC','2023-06-01'),
(104,'Delta Co','LA',NULL);

INSERT INTO products VALUES
(201,'Laptop','Electronics',999.99),
(202,'Mouse','Electronics',29.99),
(203,'Desk Chair','Furniture',299.00),
(204,'Notebook','Stationery',4.99);

INSERT INTO orders VALUES
(301,101,'2024-01-05','completed'),
(302,101,'2024-02-10','completed'),
(303,102,'2024-01-20','cancelled'),
(304,103,'2024-03-01','completed'),
(305,102,'2024-03-15','pending');

INSERT INTO order_items VALUES
(301,201,1,999.99),
(301,202,2,29.99),
(302,203,1,299.00),
(304,201,2,999.99),
(304,204,10,4.99),
(305,202,5,29.99);
```

---

## Visual Data Snapshot (Memorize These Tables)

### `departments` (3 rows)

| dept_id | dept_name   |
|--------:|-------------|
| 1       | Engineering |
| 2       | Sales       |
| 3       | HR          |

### `employees` (6 rows)

| emp_id | name  | dept_id | manager_id | salary  | hire_date  |
|-------:|-------|--------:|-----------:|--------:|------------|
| 1      | Alice | 1       | NULL       | 150000  | 2018-01-15 |
| 2      | Bob   | 1       | 1          | 120000  | 2019-03-01 |
| 3      | Carol | 1       | 1          | 110000  | 2020-06-10 |
| 4      | Dave  | 2       | 1          | 95000   | 2019-08-20 |
| 5      | Eve   | 2       | 4          | 85000   | 2021-02-14 |
| 6      | Frank | 3       | 1          | 80000   | 2020-11-01 |

**Org tree (visual):**

```
Alice (CEO, Engineering)
├── Bob (Engineering)
├── Carol (Engineering)
├── Dave (Sales)
│   └── Eve (Sales)
└── Frank (HR)
```

### `customers` (4 rows)

| cust_id | name      | city | signup_date |
|--------:|-----------|------|-------------|
| 101     | Acme Corp | NYC  | 2022-01-10  |
| 102     | Beta LLC  | SF   | 2022-03-15  |
| 103     | Gamma Inc | NYC  | 2023-06-01  |
| 104     | Delta Co  | LA   | NULL        |

### `products` (4 rows)

| prod_id | name       | category    | price  |
|--------:|------------|-------------|-------:|
| 201     | Laptop     | Electronics | 999.99 |
| 202     | Mouse      | Electronics |  29.99 |
| 203     | Desk Chair | Furniture   | 299.00 |
| 204     | Notebook   | Stationery  |   4.99 |

### `orders` (5 rows)

| order_id | cust_id | order_date | status    |
|--------:|--------:|------------|-----------|
| 301     | 101     | 2024-01-05 | completed |
| 302     | 101     | 2024-02-10 | completed |
| 303     | 102     | 2024-01-20 | cancelled |
| 304     | 103     | 2024-03-01 | completed |
| 305     | 102     | 2024-03-15 | pending   |

### `order_items` (6 rows)

| order_id | prod_id | quantity | unit_price | line_total |
|--------:|--------:|---------:|-----------:|-----------:|
| 301     | 201     | 1        | 999.99     | 999.99     |
| 301     | 202     | 2        | 29.99      | 59.98      |
| 302     | 203     | 1        | 299.00     | 299.00     |
| 304     | 201     | 2        | 999.99     | 1999.98    |
| 304     | 204     | 10       | 4.99       | 49.90      |
| 305     | 202     | 5        | 29.99      | 149.95     |

**Completed order totals:**

| order_id | customer  | order_total |
|--------:|-----------|------------:|
| 301     | Acme Corp | 1059.97     |
| 302     | Acme Corp | 299.00      |
| 304     | Gamma Inc | 2049.88     |

---

# SQL Keywords for Complete Beginners

Read this section once before the examples. Every keyword used in this guide is explained here in plain language.

**Mental model:** A database is a folder of **Excel sheets** (tables). Each sheet has **rows** (records) and **columns** (fields). A **query** is a question you ask; the **result** is a new temporary sheet the database builds for you.

---

## A. Building blocks (schema words)

These appear in `CREATE TABLE` and `INSERT` — they define and fill tables.

| Keyword | What it means | Beginner explanation |
|---------|---------------|----------------------|
| **CREATE TABLE** | Make a new empty table | Like adding a new sheet named `employees` with fixed column headers |
| **INSERT INTO** | Add rows | Like pasting new rows into the sheet: `INSERT INTO employees VALUES (...)` |
| **PRIMARY KEY (PK)** | Unique ID for each row | No two rows can share the same PK — like a student ID number |
| **FOREIGN KEY (FK)** | Points to another table's PK | `dept_id` in employees must match a real `dept_id` in departments — a link between sheets |
| **REFERENCES** | Defines the FK link | `dept_id INT REFERENCES departments(dept_id)` = "this value must exist in departments" |
| **INT, VARCHAR, DATE, DECIMAL** | Data types | INT = whole number, VARCHAR = text, DATE = calendar date, DECIMAL = money-style number |
| **NULL** | No value / unknown | Empty cell — not zero, not blank string — literally "we don't know" |

**Example in plain English:**

```sql
INSERT INTO employees VALUES (2,'Bob',1,1,120000,'2019-03-01');
```

Means: "Add one row to `employees`: emp_id=2, name=Bob, dept_id=1, manager_id=1, salary=120000, hired 2019-03-01."

---

## B. The core SELECT query (read data)

Almost every example starts with these. Think of them as a pipeline:

```
FROM (get the sheet)
  -> WHERE (filter rows)
  -> GROUP BY (bucket rows)
  -> HAVING (filter buckets)
  -> SELECT (pick columns)
  -> ORDER BY (sort)
  -> LIMIT (keep first N rows)
```

| Keyword | What it does | When it runs | Beginner analogy |
|---------|--------------|--------------|------------------|
| **SELECT** | Choose which columns to show | Late (after filtering) | "Show me these columns in the answer" |
| **FROM** | Which table(s) to read | First | "Look at this sheet" |
| **WHERE** | Filter **individual rows** | Before grouping | "Only rows where salary > 90000" |
| **GROUP BY** | Collapse rows into groups (buckets) | After WHERE | "Put all Engineering people in one bucket" — see **What does bucket mean?** above |
| **HAVING** | Filter **groups** | After GROUP BY | "Only buckets where average salary > 100k" |
| **ORDER BY** | Sort the result | Near the end | "Sort by salary, highest first" |
| **ASC / DESC** | Sort direction | With ORDER BY | ASC = ascending (A-Z, 1-9), DESC = descending (9-1) |
| **LIMIT** | Return only first N rows | Last | "Give me the top 2 rows" |
| **OFFSET** | Skip first N rows | With LIMIT | "Skip the top 1, then give me 1" — used for 2nd highest salary |
| **DISTINCT** | Remove duplicate rows | In SELECT | `SELECT DISTINCT city` = list each city once |
| **AS** | Rename a column (alias) | In SELECT | `salary AS pay` — answer column header says "pay" |

**Tiny example:**

```sql
SELECT name, salary          -- columns to display
FROM employees               -- which table
WHERE dept_id = 1            -- only Engineering rows
ORDER BY salary DESC         -- highest salary first
LIMIT 2;                     -- top 2 only
```

**WHERE vs HAVING (most common beginner mistake):**

- **WHERE** = filter rows **before** counting/summing (e.g. exclude Frank before averaging)
- **HAVING** = filter **after** you built groups (e.g. "departments whose average > 100k")

### What does "bucket" mean?

In this guide (and most SQL tutorials), **bucket** is plain English for **group** — a pile of rows that share the same value in one or more columns.

**Analogy:** Imagine sorting laundry into baskets labeled by color. All white shirts go in the **white bucket**, all blue in the **blue bucket**. You do not mix them until you are done sorting. `GROUP BY dept_id` does the same: every employee with `dept_id = 1` lands in the **Engineering bucket**, `dept_id = 2` in the **Sales bucket**, and so on.

**With our `employees` table:**

| Row   | name  | dept_id | Which bucket?   |
|-------|-------|--------:|-----------------|
| Alice | Alice | 1       | Engineering     |
| Bob   | Bob   | 1       | Engineering     |
| Carol | Carol | 1       | Engineering     |
| Dave  | Dave  | 2       | Sales           |
| Eve   | Eve   | 2       | Sales           |
| Frank | Frank | 3       | HR              |

`GROUP BY dept_id` creates **3 buckets**. Then `AVG(salary)` runs **inside each bucket** separately:

| bucket (dept_id) | rows inside        | AVG(salary) |
|------------------|--------------------|------------:|
| 1 Engineering    | Alice, Bob, Carol  | 126666.67   |
| 2 Sales          | Dave, Eve          | 90000.00    |
| 3 HR             | Frank              | 80000.00    |

**Same word, two related uses:**

| Term in guide              | Meaning |
|----------------------------|---------|
| **GROUP BY bucket**        | Rows grouped by matching column(s) — e.g. all rows with same `dept_id` |
| **HAVING filters buckets** | Drop whole groups that fail a test — e.g. "only buckets where avg > 100k" keeps Engineering, drops Sales and HR |
| **NTILE(n) bucket**        | Rank-based split — e.g. NTILE(3) divides sorted salaries into low / mid / high thirds (not the same as GROUP BY, but still called buckets) |

**Bucket vs row:** A **row** is one record (one person). A **bucket** is many rows collected together so you can compute one summary (average, count, sum) for that pile.

---

## C. Comparison and logic operators

Used inside `WHERE`, `HAVING`, and `CASE`.

| Keyword / symbol | Meaning | Example |
|------------------|---------|---------|
| **=** | Equals | `status = 'completed'` |
| **<>** or **!=** | Not equal | `city <> 'NYC'` |
| **>, <, >=, <=** | Greater / less | `salary > 100000` |
| **AND** | Both conditions true | `dept_id = 1 AND salary > 110000` |
| **OR** | Either condition true | `status = 'pending' OR status = 'cancelled'` |
| **NOT** | Flip true/false | `NOT EXISTS (...)` |
| **IN (...)** | Value in a list | `cust_id IN (101, 103)` |
| **NOT IN (...)** | Value not in list | Careful with NULLs — prefer NOT EXISTS |
| **BETWEEN a AND b** | Inclusive range | `order_date BETWEEN '2024-01-01' AND '2024-03-31'` |
| **LIKE** | Pattern match | `name LIKE 'M%'` = starts with M (`%` = anything) |
| **IS NULL** | Cell is empty | `signup_date IS NULL` |
| **IS NOT NULL** | Cell has a value | `manager_id IS NOT NULL` |

**NULL rule:** `NULL = NULL` is not TRUE. Always use `IS NULL` / `IS NOT NULL`, never `= NULL`.

---

## D. JOIN — combine two tables

**Problem:** Employee sheet has `dept_id` (a number). Department sheet has `dept_name`. JOIN glues them together.

| Keyword | What it keeps | Plain English |
|---------|---------------|---------------|
| **JOIN** / **INNER JOIN** | Only rows that match on **both** sides | "Show employees who have a valid department" |
| **LEFT JOIN** | All rows from left table + matching right (or NULL) | "All customers, even if they never ordered" |
| **RIGHT JOIN** | All rows from right + matching left | Mirror of LEFT — rare in interviews |
| **FULL OUTER JOIN** | Everything from both sides | Unmatched rows on either side still appear |
| **ON** | The match condition | `ON e.dept_id = d.dept_id` — "glue where IDs match" |
| **CROSS JOIN** | Every left row × every right row | No ON clause — Cartesian product |

**Table alias:** `FROM employees e` — `e` is a short nickname so you write `e.name` instead of `employees.name`.

**SELF JOIN:** Same table twice with two aliases — `employees e` joined to `employees m` to get employee + manager names.

**Visual (INNER JOIN):**

```
employees          departments
emp_id dept_id      dept_id dept_name
   2      1    +         1  Engineering   -> Bob, Engineering
   7      9    +         1  Engineering   -> row 7 DROPPED (no dept 9)
```

**Visual (LEFT JOIN customers + orders):**

```
customers (ALL kept)     orders (matched if exists)
Delta Co        +   (no order)  -> Delta Co, NULL, NULL
Acme Corp       +   order 301   -> Acme Corp, 301, completed
```

---

## E. Aggregate functions — summarize many rows into one number

| Function | What it computes | NULL behavior |
|----------|------------------|---------------|
| **COUNT(*)** | Number of rows | Counts rows even if some cells NULL |
| **COUNT(col)** | Rows where col is not NULL | Ignores NULL in that column |
| **COUNT(DISTINCT col)** | Unique non-NULL values | `COUNT(DISTINCT city)` = how many different cities |
| **SUM(col)** | Add up numbers | Skips NULL |
| **AVG(col)** | Average of numbers | Skips NULL |
| **MIN(col)** | Smallest value | Skips NULL |
| **MAX(col)** | Largest value | Skips NULL |

**Rule with GROUP BY:** Every column in SELECT must either:
- Be in the `GROUP BY` list, OR
- Be inside an aggregate (`SUM`, `COUNT`, etc.)

```sql
SELECT dept_id, AVG(salary)   -- OK: dept_id grouped, salary averaged
FROM employees
GROUP BY dept_id;
```

You cannot `SELECT name, AVG(salary)` without grouping by `name` — which name would you show for 3 people in one bucket?

---

## F. Subqueries — a query inside another query

A **subquery** is `(SELECT ...)` nested inside a bigger query.

| Pattern | How it works | Used for |
|---------|--------------|----------|
| **Scalar subquery** | Returns **one value** | `WHERE salary > (SELECT AVG(salary) FROM employees)` |
| **IN subquery** | Returns a **list** of values | `WHERE cust_id IN (SELECT cust_id FROM orders ...)` |
| **EXISTS subquery** | Returns true if inner query finds **any row** | `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.cust_id = c.cust_id)` |
| **NOT EXISTS** | True if inner finds **no row** | Customers with zero orders — safe with NULLs |
| **Correlated subquery** | Inner query references **outer row** | `WHERE e2.dept_id = e1.dept_id` — recomputed per employee |

**EXISTS vs IN:** For "is there at least one match?", EXISTS often stops at first match and handles NULLs better than NOT IN.

---

## G. CTE — WITH (name your intermediate step)

**CTE = Common Table Expression.** A temporary named result you use in the same query.

```sql
WITH customer_revenue AS (
    SELECT cust_id, SUM(...) AS revenue
    FROM ...
    GROUP BY cust_id
)
SELECT * FROM customer_revenue WHERE revenue > 500;
```

| Keyword | Role |
|---------|------|
| **WITH** | Start a named subquery block |
| **CTE name** | `customer_revenue` — use like a table in the next line |
| **Chained CTEs** | `WITH a AS (...), b AS (...), c AS (...)` — step 1, step 2, step 3 |
| **RECURSIVE** | CTE calls itself — org trees, date series |
| **UNION ALL** (in recursive) | Anchor rows + recursive rows stitched together |

**Why use CTEs?** Same as writing "Step 1" and "Step 2" on a whiteboard — interviewers read it faster than deeply nested parentheses.

---

## H. Window functions — add a column without collapsing rows

**Aggregate + GROUP BY** collapses 100 rows into 5 buckets. **Window functions** keep all 100 rows and add a 6th computed column.

```sql
SUM(salary) OVER (PARTITION BY dept_id ORDER BY hire_date) AS running_total
```

| Keyword | Role |
|---------|------|
| **OVER (...)** | "Compute this function over a window of rows" |
| **PARTITION BY** | Split data into groups (like GROUP BY, but rows stay separate) |
| **ORDER BY** (inside OVER) | Order rows inside each partition — matters for LAG, running sums |
| **ROWS BETWEEN ... AND ...** | Frame: which nearby rows to include (moving average) |

**Ranking functions:**

| Function | Ties (same value) | Next rank after tie |
|----------|-------------------|---------------------|
| **ROW_NUMBER()** | Breaks ties arbitrarily | always n+1 |
| **RANK()** | Same rank for ties | skips (1, 2, 2, 4) |
| **DENSE_RANK()** | Same rank for ties | no skip (1, 2, 2, 3) |
| **NTILE(n)** | Split into n buckets | quartiles, deciles |

**Navigation functions:**

| Function | Meaning |
|----------|---------|
| **LAG(col)** | Value from **previous** row (in ORDER BY order) |
| **LEAD(col)** | Value from **next** row |

**Pattern:** `ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) = 1` means "top earner per department."

---

## I. CASE, COALESCE, NULLIF — shape values

### CASE WHEN (if/else for cells)

```sql
CASE
    WHEN salary >= 120000 THEN 'Senior'
    WHEN salary >= 90000  THEN 'Mid'
    ELSE 'Junior'
END AS band
```

| Part | Role |
|------|------|
| **CASE** | Start conditional logic |
| **WHEN ... THEN ...** | If condition true, use this value |
| **ELSE** | Default if nothing matched |
| **END** | Close the CASE |

Also used inside aggregates: `SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END)` = count completed orders.

### COALESCE(a, b, c)

Returns the **first non-NULL** value. `COALESCE(signup_date, 'Unknown')` — show date or "Unknown".

### NULLIF(a, b)

Returns NULL if `a = b`, else returns `a`. Used to avoid divide-by-zero: `x / NULLIF(y, 0)`.

### ROUND(x, n)

Round number to `n` decimal places. `ROUND(126666.666, 2)` -> 126666.67.

---

## J. Set operations — stack two query results

Both queries must return the **same number of columns** with compatible types.

| Keyword | What it does |
|---------|--------------|
| **UNION** | Stack results, **remove duplicates** |
| **UNION ALL** | Stack results, **keep duplicates** |
| **INTERSECT** | Rows in **both** results |
| **EXCEPT** | Rows in first result **but not** second (PostgreSQL / SQL Server) |

---

## K. Date and time keywords

| Keyword | Role |
|---------|------|
| **DATE '2024-01-01'** | Literal date value |
| **INTERVAL '1 day'** | Add/subtract time length |
| **DATE_TRUNC('month', date)** | Chop to month start (PostgreSQL) — cohort analysis |
| **::text** | Cast to text (PostgreSQL) — `signup_date::text` |

Example: `event_time - prev_time > INTERVAL '30 minutes'` — gap detection for sessions.

---

## L. Quick keyword lookup by example number

When you see a query in this guide, find unfamiliar words here:

| You see in query | Section |
|------------------|---------|
| bucket, GROUP BY, group | B — **What does bucket mean?** |
| SELECT, FROM, WHERE, ORDER BY, LIMIT | B |
| JOIN, LEFT JOIN, ON | D |
| GROUP BY, HAVING, SUM, COUNT, AVG | E |
| (SELECT ...) inside WHERE | F |
| WITH, RECURSIVE | G |
| OVER, PARTITION BY, LAG, LEAD, RANK | H |
| CASE WHEN, COALESCE | I |
| UNION, INTERSECT, EXCEPT | J |
| BETWEEN, LIKE, IN, EXISTS | C |
| DISTINCT, AS, OFFSET | B |

---

## M. One query, fully annotated (read this slowly)

```sql
WITH completed AS (                          -- G: name step 1
    SELECT * FROM orders
    WHERE status = 'completed'               -- B: filter rows
),
totals AS (                                  -- G: name step 2
    SELECT
        o.cust_id,                           -- E: group key
        SUM(oi.quantity * oi.unit_price)     -- E: add line totals
            AS revenue                       -- B: alias
    FROM completed o
    JOIN order_items oi                      -- D: glue orders to items
        ON o.order_id = oi.order_id          -- D: match key
    GROUP BY o.cust_id                       -- E: one row per customer
)
SELECT c.name, t.revenue                     -- B: final columns
FROM totals t
JOIN customers c ON t.cust_id = c.cust_id   -- D: get customer names
WHERE t.revenue > 500                        -- B: filter final rows
ORDER BY t.revenue DESC;                     -- B: biggest spenders first
```

**Execution story:**
1. `completed` = only 3 orders (301, 302, 304)
2. `totals` = sum line items per customer (101 -> 1358.97, 103 -> 2049.88)
3. Final SELECT joins names, drops nothing (both > 500), sorts Gamma first

---

# Part 1 — Foundations

## Example 1.1 — Basic SELECT + WHERE + ORDER BY

**Goal:** All Engineering employees, highest salary first.

**Input:** `employees` (see snapshot)

```sql
SELECT name, salary, hire_date
FROM employees
WHERE dept_id = 1
ORDER BY salary DESC;
```

**Output:**

| name  | salary | hire_date  |
|-------|-------:|------------|
| Alice | 150000 | 2018-01-15 |
| Bob   | 120000 | 2019-03-01 |
| Carol | 110000 | 2020-06-10 |

**Takeaway:** `WHERE` filters rows before sort. `ORDER BY DESC` = highest first.

---

## Example 1.2 — LIMIT (Top 2 earners)

```sql
SELECT name, salary
FROM employees
ORDER BY salary DESC
LIMIT 2;
```

**Output:**

| name  | salary |
|-------|-------:|
| Alice | 150000 |
| Bob   | 120000 |

---

## Example 1.3 — SQL execution order (interview favorite)

```sql
SELECT dept_id, AVG(salary) AS avg_sal
FROM employees
WHERE salary > 80000
GROUP BY dept_id
HAVING AVG(salary) > 100000
ORDER BY avg_sal DESC;
```

**Step-by-step (what the engine does):**

| Step | Action | Rows left |
|------|--------|-----------|
| FROM | Read employees | 6 |
| WHERE | salary > 80000 | 5 (Frank excluded) |
| GROUP BY | Bucket by dept_id | 3 groups |
| HAVING | avg > 100000 | 1 group (Engineering) |
| SELECT | Show columns | 1 row |
| ORDER BY | Sort | 1 row |

**Output:**

| dept_id | avg_sal   |
|--------:|----------:|
| 1       | 126666.67 |

**Takeaway:** You cannot use alias `avg_sal` in `HAVING` on all dialects — `HAVING AVG(salary) > 100000` is safest.

---

# Part 2 — JOINs

## Example 2.1 — INNER JOIN (employees + departments)

**Goal:** Show each employee with department name.

**Input:** `employees` JOIN `departments` on `dept_id`

```sql
SELECT e.name, d.dept_name, e.salary
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

**Output:**

| name  | dept_name   | salary |
|-------|-------------|-------:|
| Alice | Engineering | 150000 |
| Bob   | Engineering | 120000 |
| Carol | Engineering | 110000 |
| Dave  | Sales       | 95000  |
| Eve   | Sales       | 85000  |
| Frank | HR          | 80000  |

**Takeaway:** INNER = only rows with a match on **both** sides.

---

## Example 2.2 — LEFT JOIN (all customers, with or without orders)

```sql
SELECT c.name, o.order_id, o.status
FROM customers c
LEFT JOIN orders o ON c.cust_id = o.cust_id
ORDER BY c.name, o.order_id;
```

**Output:**

| name      | order_id | status    |
|-----------|--------:|-----------|
| Acme Corp | 301     | completed |
| Acme Corp | 302     | completed |
| Beta LLC  | 303     | cancelled |
| Beta LLC  | 305     | pending   |
| Delta Co  | NULL    | NULL      |
| Gamma Inc | 304     | completed |

**Takeaway:** Delta Co has **no orders** — `order_id` is NULL. This is the setup for anti-joins.

---

## Example 2.3 — Anti-join: customers who NEVER ordered

```sql
SELECT c.name
FROM customers c
LEFT JOIN orders o ON c.cust_id = o.cust_id
WHERE o.order_id IS NULL;
```

**Output:**

| name     |
|----------|
| Delta Co |

**Same with NOT EXISTS:**

```sql
SELECT c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.cust_id = c.cust_id
);
```

**Output:** same — `Delta Co`

**Takeaway:** HackerRank "find A not in B" = `LEFT JOIN ... WHERE B.id IS NULL` or `NOT EXISTS`.

---

## Example 2.4 — Multiple JOINs: revenue per customer (completed only)

```sql
SELECT
    c.name AS customer,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM customers c
JOIN orders o ON c.cust_id = o.cust_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status = 'completed'
GROUP BY c.cust_id, c.name
ORDER BY total_revenue DESC;
```

**How rows multiply (before GROUP BY):**

| customer  | order_id | prod_id | line_total |
|-----------|--------:|--------:|-----------:|
| Acme Corp | 301     | 201     | 999.99     |
| Acme Corp | 301     | 202     | 59.98      |
| Acme Corp | 302     | 203     | 299.00     |
| Gamma Inc | 304     | 201     | 1999.98    |
| Gamma Inc | 304     | 204     | 49.90      |

Beta LLC excluded (cancelled + pending only). Delta Co excluded (no orders).

**Output:**

| customer  | total_revenue |
|-----------|-------------:|
| Gamma Inc | 2049.88      |
| Acme Corp | 1358.97      |

---

## Example 2.5 — SELF JOIN (employee + manager)

```sql
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id
ORDER BY e.emp_id;
```

**Output:**

| employee | manager |
|----------|---------|
| Alice    | NULL    |
| Bob      | Alice   |
| Carol    | Alice   |
| Dave     | Alice   |
| Eve      | Dave    |
| Frank    | Alice   |

---

## Example 2.6 — Revenue by product category

```sql
SELECT
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.prod_id = p.prod_id
WHERE o.status = 'completed'
GROUP BY p.category
ORDER BY revenue DESC;
```

**Output:**

| category    | revenue |
|-------------|--------:|
| Electronics | 3059.95 |
| Furniture   | 299.00  |
| Stationery  | 49.90   |

---

## Example 2.7 — Products never sold

```sql
SELECT p.name
FROM products p
LEFT JOIN order_items oi ON p.prod_id = oi.prod_id
WHERE oi.order_id IS NULL;
```

**Output:**

| name |
|------|
| *(empty — all products were ordered at least once)* |

**Takeaway:** Empty result is valid. Desk Chair was only on completed order 302; all products appear in `order_items`.

---

## Example 2.8 — Order count per customer

```sql
SELECT c.name, COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.cust_id = o.cust_id
GROUP BY c.cust_id, c.name
ORDER BY order_count DESC;
```

**Output:**

| name      | order_count |
|-----------|------------:|
| Beta LLC  | 2           |
| Acme Corp | 2           |
| Gamma Inc | 1           |
| Delta Co  | 0           |

---

# Part 3 — Aggregations & GROUP BY

## Example 3.1 — Dept average vs company average

```sql
SELECT
    d.dept_name,
    ROUND(AVG(e.salary), 2) AS dept_avg,
    ROUND((SELECT AVG(salary) FROM employees), 2) AS company_avg
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_id, d.dept_name;
```

**Output:**

| dept_name   | dept_avg  | company_avg |
|-------------|----------:|------------:|
| Engineering | 126666.67 | 106666.67   |
| Sales       | 90000.00  | 106666.67   |
| HR          | 80000.00  | 106666.67   |

---

## Example 3.2 — HAVING (filter groups, not rows)

```sql
SELECT d.dept_name, ROUND(AVG(e.salary), 2) AS avg_sal
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_id, d.dept_name
HAVING AVG(e.salary) > 100000;
```

**Output:**

| dept_name   | avg_sal   |
|-------------|----------:|
| Engineering | 126666.67 |

Sales (90000) and HR (80000) filtered out by HAVING.

---

## Example 3.3 — Conditional aggregation (pivot without PIVOT)

```sql
SELECT
    d.dept_name,
    COUNT(*) AS headcount,
    SUM(CASE WHEN e.salary >= 100000 THEN 1 ELSE 0 END) AS high_earners,
    SUM(CASE WHEN e.salary < 100000 THEN 1 ELSE 0 END) AS others
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_id, d.dept_name;
```

**Output:**

| dept_name   | headcount | high_earners | others |
|-------------|----------:|-------------:|-------:|
| Engineering | 3         | 3            | 0      |
| Sales       | 2         | 0            | 2      |
| HR          | 1         | 0            | 1      |

---

## Example 3.4 — COUNT DISTINCT cities with customers

```sql
SELECT COUNT(DISTINCT city) AS city_count FROM customers;
```

**Output:**

| city_count |
|----------:|
| 3          |

NYC, SF, LA — NULL city does not count in `COUNT(DISTINCT city)`.

---

# Part 4 — Subqueries

## Example 4.1 — Scalar subquery: above company average

```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees)
ORDER BY salary DESC;
```

Company avg = 106666.67

**Output:**

| name  | salary |
|-------|-------:|
| Alice | 150000 |
| Bob   | 120000 |
| Carol | 110000 |

---

## Example 4.2 — IN: customers with a completed order

```sql
SELECT name
FROM customers
WHERE cust_id IN (
    SELECT cust_id FROM orders WHERE status = 'completed'
)
ORDER BY name;
```

**Output:**

| name      |
|-----------|
| Acme Corp |
| Gamma Inc |

Beta LLC had only cancelled/pending. Delta Co never ordered.

---

## Example 4.3 — Correlated subquery: earn more than dept average

```sql
SELECT e1.name, e1.salary, d.dept_name
FROM employees e1
JOIN departments d ON e1.dept_id = d.dept_id
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e1.dept_id
);
```

**Per-dept averages:** Eng 126666.67, Sales 90000, HR 80000

**Output:**

| name  | salary | dept_name   |
|-------|-------:|-------------|
| Alice | 150000 | Engineering |
| Dave  | 95000  | Sales       |

Alice beats Eng avg; Dave beats Sales avg. Bob/Carol are below Eng avg.

---

## Example 4.4 — 2nd highest salary (3 ways)

**Input:** salaries 150k, 120k, 110k, 95k, 85k, 80k

**Way 1 — OFFSET:**

```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
OFFSET 1 LIMIT 1;
```

**Way 2 — DENSE_RANK:**

```sql
WITH r AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
    FROM employees
)
SELECT salary FROM r WHERE dr = 2;
```

**Way 3 — MAX trick:**

```sql
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

**All three output:**

| salary |
|-------:|
| 120000 |

---

# Part 5 — CTEs

## Example 5.1 — Simple CTE: customers with revenue > 500

```sql
WITH customer_revenue AS (
    SELECT
        c.cust_id,
        c.name,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM customers c
    JOIN orders o ON c.cust_id = o.cust_id
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY c.cust_id, c.name
)
SELECT name, revenue
FROM customer_revenue
WHERE revenue > 500
ORDER BY revenue DESC;
```

**CTE `customer_revenue` (intermediate):**

| name      | revenue |
|-----------|--------:|
| Gamma Inc | 2049.88 |
| Acme Corp | 1358.97 |

**Final output:** same (both > 500)

---

## Example 5.2 — Chained CTEs (step-by-step)

```sql
WITH completed_orders AS (
    SELECT * FROM orders WHERE status = 'completed'
),
line_totals AS (
    SELECT
        o.cust_id,
        o.order_id,
        SUM(oi.quantity * oi.unit_price) AS order_total
    FROM completed_orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.cust_id, o.order_id
),
customer_totals AS (
    SELECT cust_id, SUM(order_total) AS total_spent
    FROM line_totals
    GROUP BY cust_id
)
SELECT c.name, ct.total_spent
FROM customer_totals ct
JOIN customers c ON ct.cust_id = c.cust_id
ORDER BY ct.total_spent DESC;
```

**Step 1 — `completed_orders`:**

| order_id | cust_id | order_date | status    |
|--------:|--------:|------------|-----------|
| 301     | 101     | 2024-01-05 | completed |
| 302     | 101     | 2024-02-10 | completed |
| 304     | 103     | 2024-03-01 | completed |

**Step 2 — `line_totals`:**

| cust_id | order_id | order_total |
|--------:|---------:|------------:|
| 101     | 301      | 1059.97     |
| 101     | 302      | 299.00      |
| 103     | 304      | 2049.88     |

**Step 3 — `customer_totals`:**

| cust_id | total_spent |
|--------:|------------:|
| 103     | 2049.88     |
| 101     | 1358.97     |

**Final output:**

| name      | total_spent |
|-----------|------------:|
| Gamma Inc | 2049.88     |
| Acme Corp | 1358.97     |

---

## Example 5.3 — Recursive CTE: all reports under Alice

```sql
WITH RECURSIVE report_chain AS (
    SELECT emp_id, name, manager_id, 1 AS depth
    FROM employees
    WHERE manager_id = 1

    UNION ALL

    SELECT e.emp_id, e.name, e.manager_id, rc.depth + 1
    FROM employees e
    JOIN report_chain rc ON e.manager_id = rc.emp_id
)
SELECT * FROM report_chain ORDER BY depth, name;
```

**Output:**

| emp_id | name  | manager_id | depth |
|-------:|-------|----------:|------:|
| 2      | Bob   | 1         | 1     |
| 3      | Carol | 1         | 1     |
| 4      | Dave  | 1         | 1     |
| 6      | Frank | 1         | 1     |
| 5      | Eve   | 4         | 2     |

**Takeaway:** Eve appears at depth 2 (report of Dave, who reports to Alice).

---

## Example 5.4 — Generate date series (Jan 2024)

```sql
WITH RECURSIVE dates AS (
    SELECT DATE '2024-01-01' AS d
    UNION ALL
    SELECT d + INTERVAL '1 day' FROM dates WHERE d < DATE '2024-01-07'
)
SELECT d FROM dates;
```

**Output (first 7 days):**

| d          |
|------------|
| 2024-01-01 |
| 2024-01-02 |
| 2024-01-03 |
| 2024-01-04 |
| 2024-01-05 |
| 2024-01-06 |
| 2024-01-07 |

---

# Part 6 — Window Functions

## Example 6.1 — ROW_NUMBER vs RANK vs DENSE_RANK (ties demo)

**Mini-table `scores` (for tie behavior):**

| player | points |
|--------|-------:|
| Ann    | 100    |
| Bob    | 90     |
| Cal    | 90     |
| Dee    | 80     |

```sql
SELECT
    player,
    points,
    ROW_NUMBER() OVER (ORDER BY points DESC) AS row_num,
    RANK()       OVER (ORDER BY points DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY points DESC) AS dense_rank
FROM scores;
```

**Output:**

| player | points | row_num | rank | dense_rank |
|--------|-------:|--------:|-----:|-----------:|
| Ann    | 100    | 1       | 1    | 1          |
| Bob    | 90     | 2       | 2    | 2          |
| Cal    | 90     | 3       | 2    | 2          |
| Dee    | 80     | 4       | 4    | 3          |

**Takeaway:** RANK skips to 4 after tie at 2. DENSE_RANK goes 1,2,2,3.

---

## Example 6.2 — Top earner per department

```sql
WITH ranked AS (
    SELECT
        name,
        dept_id,
        salary,
        DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT name, dept_id, salary
FROM ranked
WHERE rnk = 1
ORDER BY dept_id;
```

**Output:**

| name  | dept_id | salary |
|-------|--------:|-------:|
| Alice | 1       | 150000 |
| Dave  | 2       | 95000  |
| Frank | 3       | 80000  |

---

## Example 6.3 — Running total of completed revenue

```sql
SELECT
    order_date,
    order_id,
    order_total,
    SUM(order_total) OVER (ORDER BY order_date, order_id) AS running_revenue
FROM (
    SELECT o.order_date, o.order_id,
           SUM(oi.quantity * oi.unit_price) AS order_total
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_date, o.order_id
) t
ORDER BY order_date, order_id;
```

**Output:**

| order_date | order_id | order_total | running_revenue |
|------------|--------:|------------:|----------------:|
| 2024-01-05 | 301     | 1059.97     | 1059.97         |
| 2024-02-10 | 302     | 299.00      | 1358.97         |
| 2024-03-01 | 304     | 2049.88     | 3408.85         |

---

## Example 6.4 — LAG: salary vs previous hire in dept

```sql
SELECT
    name,
    dept_id,
    salary,
    LAG(salary) OVER (PARTITION BY dept_id ORDER BY hire_date) AS prev_salary,
    salary - LAG(salary) OVER (PARTITION BY dept_id ORDER BY hire_date) AS delta
FROM employees
ORDER BY dept_id, hire_date;
```

**Output:**

| name  | dept_id | salary | prev_salary | delta  |
|-------|--------:|-------:|------------:|-------:|
| Alice | 1       | 150000 | NULL        | NULL   |
| Bob   | 1       | 120000 | 150000      | -30000 |
| Carol | 1       | 110000 | 120000      | -10000 |
| Dave  | 2       | 95000  | NULL        | NULL   |
| Eve   | 2       | 85000  | 95000       | -10000 |
| Frank | 3       | 80000  | NULL        | NULL   |

---

## Example 6.5 — NTILE: split employees into 3 salary buckets

```sql
SELECT name, salary,
       NTILE(3) OVER (ORDER BY salary DESC) AS bucket
FROM employees
ORDER BY salary DESC;
```

**Output:**

| name  | salary | bucket |
|-------|-------:|-------:|
| Alice | 150000 | 1      |
| Bob   | 120000 | 1      |
| Carol | 110000 | 2      |
| Dave  | 95000  | 2      |
| Eve   | 85000  | 3      |
| Frank | 80000  | 3      |

---

# Part 7 — CASE, NULL, COALESCE

## Example 7.1 — Salary bands

```sql
SELECT name, salary,
    CASE
        WHEN salary >= 120000 THEN 'Senior'
        WHEN salary >= 90000  THEN 'Mid'
        ELSE 'Junior'
    END AS band
FROM employees
ORDER BY salary DESC;
```

**Output:**

| name  | salary | band   |
|-------|-------:|--------|
| Alice | 150000 | Senior |
| Bob   | 120000 | Senior |
| Carol | 110000 | Mid    |
| Dave  | 95000  | Mid    |
| Eve   | 85000  | Junior |
| Frank | 80000  | Junior |

---

## Example 7.2 — COALESCE for NULL signup

```sql
SELECT name,
       COALESCE(signup_date::text, 'Unknown') AS signup
FROM customers;
```

**Output:**

| name      | signup     |
|-----------|------------|
| Acme Corp | 2022-01-10 |
| Beta LLC  | 2022-03-15 |
| Gamma Inc | 2023-06-01 |
| Delta Co  | Unknown    |

---

## Example 7.3 — Order status summary per customer

```sql
SELECT
    c.name,
    SUM(CASE WHEN o.status = 'completed' THEN 1 ELSE 0 END) AS completed,
    SUM(CASE WHEN o.status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled,
    SUM(CASE WHEN o.status = 'pending' THEN 1 ELSE 0 END) AS pending
FROM customers c
LEFT JOIN orders o ON c.cust_id = o.cust_id
GROUP BY c.name
ORDER BY c.name;
```

**Output:**

| name      | completed | cancelled | pending |
|-----------|----------:|----------:|--------:|
| Acme Corp | 2         | 0         | 0       |
| Beta LLC  | 0         | 1         | 1       |
| Delta Co  | 0         | 0         | 0       |
| Gamma Inc | 1         | 0         | 0       |

---

# Part 8 — HackerRank Classics (with data + output)

## Example 8.1 — Consecutive available seats

**Input `seats`:**

| seat_id | free  |
|--------:|:------|
| 1       | true  |
| 2       | true  |
| 3       | false |
| 4       | true  |
| 5       | true  |
| 6       | true  |
| 7       | false |
| 8       | true  |

```sql
WITH t AS (
    SELECT seat_id,
           seat_id - ROW_NUMBER() OVER (ORDER BY seat_id) AS grp
    FROM seats
    WHERE free = true
)
SELECT MIN(seat_id) AS start_seat, MAX(seat_id) AS end_seat, COUNT(*) AS streak_len
FROM t
GROUP BY grp
HAVING COUNT(*) >= 2
ORDER BY start_seat;
```

**Intermediate (free seats + grp):**

| seat_id | grp |
|--------:|----:|
| 1       | 1   |
| 2       | 1   |
| 4       | 3   |
| 5       | 3   |
| 6       | 3   |
| 8       | 6   |

Seats 1-2 share grp=1; 4-6 share grp=3.

**Output:**

| start_seat | end_seat | streak_len |
|----------:|---------:|-----------:|
| 1         | 2        | 2          |
| 4         | 6        | 3          |

---

## Example 8.2 — Type of triangle

**Input `triangles`:**

| id | A | B | C |
|---:|--:|--:|--:|
| 1  | 3 | 4 | 5 |
| 2  | 5 | 5 | 5 |
| 3  | 4 | 4 | 7 |
| 4  | 1 | 2 | 3 |

```sql
SELECT
    id,
    CASE
        WHEN A + B <= C OR A + C <= B OR B + C <= A THEN 'Not A Triangle'
        WHEN A = B AND B = C THEN 'Equilateral'
        WHEN A = B OR B = C OR A = C THEN 'Isosceles'
        ELSE 'Scalene'
    END AS triangle_type
FROM triangles;
```

**Output:**

| id | triangle_type  |
|---:|----------------|
| 1  | Scalene        |
| 2  | Equilateral    |
| 3  | Isosceles      |
| 4  | Not A Triangle |

(1+2=3, not > 3, so id=4 fails triangle inequality)

---

## Example 8.3 — Duplicate emails

**Input `users`:**

| user_id | email           |
|--------:|-----------------|
| 1       | alice@co.com    |
| 2       | bob@co.com      |
| 3       | alice@co.com    |
| 4       | carol@co.com    |

```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

**Output:**

| email        | cnt |
|--------------|----:|
| alice@co.com | 2   |

**Dedupe — keep newest:**

```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY email ORDER BY user_id DESC) AS rn
    FROM users
)
SELECT user_id, email FROM ranked WHERE rn = 1 ORDER BY user_id;
```

**Output:**

| user_id | email        |
|--------:|--------------|
| 2       | bob@co.com   |
| 3       | alice@co.com |
| 4       | carol@co.com |

---

## Example 8.4 — The Report (COALESCE + LEFT JOIN)

**Input `students`:**

| id | name  |
|---:|-------|
| 1  | Alice |
| 2  | Bob   |

**Input `grades`:**

| student_id | grade |
|----------:|-------|
| 1         | 10    |

```sql
SELECT s.name, COALESCE(g.grade, 0) AS grade
FROM students s
LEFT JOIN grades g ON s.id = g.student_id
ORDER BY s.id;
```

**Output:**

| name  | grade |
|-------|------:|
| Alice | 10    |
| Bob   | 0     |

---

# Part 9 — Big Tech Scenarios (full mini datasets)

## Example 9.1 — Daily Active Users (DAU)

**Input `events`:**

| user_id | event_date | event_type |
|--------:|------------|------------|
| 1       | 2024-01-01 | login      |
| 2       | 2024-01-01 | login      |
| 1       | 2024-01-01 | click      |
| 1       | 2024-01-02 | login      |
| 3       | 2024-01-02 | login      |
| 2       | 2024-01-03 | login      |

```sql
SELECT event_date, COUNT(DISTINCT user_id) AS dau
FROM events
WHERE event_type = 'login'
GROUP BY event_date
ORDER BY event_date;
```

**Output:**

| event_date | dau |
|------------|----:|
| 2024-01-01 | 2   |
| 2024-01-02 | 2   |
| 2024-01-03 | 1   |

---

## Example 9.2 — Funnel conversion

**Input `user_events`:**

| user_id | step     |
|--------:|----------|
| 1       | visit    |
| 1       | signup   |
| 1       | purchase |
| 2       | visit    |
| 2       | signup   |
| 3       | visit    |

```sql
WITH funnel AS (
    SELECT
        user_id,
        MAX(CASE WHEN step = 'visit' THEN 1 ELSE 0 END) AS visited,
        MAX(CASE WHEN step = 'signup' THEN 1 ELSE 0 END) AS signed_up,
        MAX(CASE WHEN step = 'purchase' THEN 1 ELSE 0 END) AS purchased
    FROM user_events
    GROUP BY user_id
)
SELECT
    SUM(visited) AS visitors,
    SUM(signed_up) AS signups,
    SUM(purchased) AS buyers,
    ROUND(100.0 * SUM(signed_up) / NULLIF(SUM(visited), 0), 2) AS visit_to_signup_pct,
    ROUND(100.0 * SUM(purchased) / NULLIF(SUM(signed_up), 0), 2) AS signup_to_buy_pct
FROM funnel;
```

**Per-user funnel:**

| user_id | visited | signed_up | purchased |
|--------:|--------:|----------:|----------:|
| 1       | 1       | 1         | 1         |
| 2       | 1       | 1         | 0         |
| 3       | 1       | 0         | 0         |

**Output:**

| visitors | signups | buyers | visit_to_signup_pct | signup_to_buy_pct |
|--------:|--------:|-------:|--------------------:|------------------:|
| 3       | 2       | 1      | 66.67               | 50.00             |

---

## Example 9.3 — Sessionization (30-min gap)

**Input `events` (one user):**

| user_id | event_time          |
|--------:|---------------------|
| 1       | 2024-01-01 10:00:00 |
| 1       | 2024-01-01 10:15:00 |
| 1       | 2024-01-01 11:00:00 |
| 1       | 2024-01-01 11:05:00 |

Gap 10:15 -> 11:00 = 45 min > 30 min = new session.

```sql
WITH ordered AS (
    SELECT user_id, event_time,
           LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_time
    FROM events
),
flagged AS (
    SELECT *,
        CASE WHEN prev_time IS NULL
                  OR event_time - prev_time > INTERVAL '30 minutes'
             THEN 1 ELSE 0 END AS new_session
    FROM ordered
),
sessions AS (
    SELECT *,
        SUM(new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
    FROM flagged
)
SELECT user_id, session_id,
       MIN(event_time) AS session_start,
       COUNT(*) AS events
FROM sessions
GROUP BY user_id, session_id;
```

**Output:**

| user_id | session_id | session_start       | events |
|--------:|-----------:|---------------------|-------:|
| 1       | 1          | 2024-01-01 10:00:00 | 2      |
| 1       | 2          | 2024-01-01 11:00:00 | 2      |

---

## Example 9.4 — Month-1 retention cohort

**Input `events`:**

| user_id | event_date |
|--------:|------------|
| 1       | 2024-01-05 |
| 1       | 2024-02-10 |
| 2       | 2024-01-20 |
| 3       | 2024-02-01 |

```sql
WITH first_month AS (
    SELECT user_id, DATE_TRUNC('month', MIN(event_date)) AS cohort_month
    FROM events
    GROUP BY user_id
),
activity AS (
    SELECT user_id, DATE_TRUNC('month', event_date) AS activity_month
    FROM events
    GROUP BY user_id, DATE_TRUNC('month', event_date)
)
SELECT
    f.cohort_month,
    COUNT(DISTINCT f.user_id) AS cohort_size,
    COUNT(DISTINCT CASE
        WHEN a.activity_month = f.cohort_month + INTERVAL '1 month'
        THEN f.user_id END) AS month_1_retained
FROM first_month f
LEFT JOIN activity a ON f.user_id = a.user_id
GROUP BY f.cohort_month;
```

**Cohorts:** user1 Jan, user2 Jan, user3 Feb

**Output:**

| cohort_month | cohort_size | month_1_retained |
|--------------|------------:|-----------------:|
| 2024-01-01   | 2           | 1                |
| 2024-02-01   | 1           | 0                |

User 1 returned in Feb (retained). User 2 only in Jan. User 3 cohort has no month+1 data yet.

---

# Part 10 — More Practice Examples

## Example 10.1 — Cities with multiple customers

```sql
SELECT city, COUNT(*) AS cust_count
FROM customers
WHERE city IS NOT NULL
GROUP BY city
HAVING COUNT(*) > 1;
```

**Output:**

| city | cust_count |
|------|----------:|
| NYC  | 2         |

---

## Example 10.2 — Employees hired after 2020

```sql
SELECT name, hire_date FROM employees
WHERE hire_date > '2020-01-01'
ORDER BY hire_date;
```

**Output:**

| name  | hire_date  |
|-------|------------|
| Carol | 2020-06-10 |
| Frank | 2020-11-01 |
| Eve   | 2021-02-14 |

---

## Example 10.3 — Average order value (completed)

```sql
SELECT ROUND(AVG(order_total), 2) AS avg_order_value
FROM (
    SELECT o.order_id, SUM(oi.quantity * oi.unit_price) AS order_total
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_id
) t;
```

**Output:**

| avg_order_value |
|----------------:|
| 1136.28         |

(1059.97 + 299 + 2049.88) / 3

---

## Example 10.4 — Customers in NYC who ordered

```sql
SELECT DISTINCT c.name
FROM customers c
JOIN orders o ON c.cust_id = o.cust_id
WHERE c.city = 'NYC';
```

**Output:**

| name      |
|-----------|
| Acme Corp |
| Gamma Inc |

---

## Example 10.5 — UNION: all cities (customers + hypothetical shippers)

```sql
SELECT city FROM customers WHERE city IS NOT NULL
UNION
SELECT 'NYC' UNION SELECT 'Chicago';
```

**Output:**

| city    |
|---------|
| NYC     |
| SF      |
| LA      |
| Chicago |

---

## Example 10.6 — INTERSECT: cities in both customers and suppliers

**Input `suppliers`:**

| city    |
|---------|
| NYC     |
| SF      |
| Boston  |

```sql
SELECT city FROM customers WHERE city IS NOT NULL
INTERSECT
SELECT city FROM suppliers;
```

**Output:**

| city |
|------|
| NYC  |
| SF   |

---

## Example 10.7 — EXCEPT: customer cities not served by suppliers

Uses same `suppliers` table as 10.6.

```sql
SELECT city FROM customers WHERE city IS NOT NULL
EXCEPT
SELECT city FROM suppliers;
```

**Output:**

| city |
|------|
| LA   |

---

## Example 10.8 — LIKE: products starting with 'M'

```sql
SELECT name FROM products WHERE name LIKE 'M%';
```

**Output:**

| name  |
|-------|
| Mouse |

---

## Example 10.9 — BETWEEN: orders in Q1 2024

```sql
SELECT order_id, order_date, status
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'
ORDER BY order_date;
```

**Output:**

| order_id | order_date | status    |
|--------:|------------|-----------|
| 301     | 2024-01-05 | completed |
| 303     | 2024-01-20 | cancelled |
| 302     | 2024-02-10 | completed |
| 304     | 2024-03-01 | completed |
| 305     | 2024-03-15 | pending   |

---

## Example 10.10 — LEAD: next order date per customer

```sql
SELECT
    cust_id,
    order_date,
    LEAD(order_date) OVER (PARTITION BY cust_id ORDER BY order_date) AS next_order
FROM orders
ORDER BY cust_id, order_date;
```

**Output:**

| cust_id | order_date | next_order |
|--------:|------------|------------|
| 101     | 2024-01-05 | 2024-02-10 |
| 101     | 2024-02-10 | NULL       |
| 102     | 2024-01-20 | 2024-03-15 |
| 102     | 2024-03-15 | NULL       |
| 103     | 2024-03-01 | NULL       |

---

# Quick Reference Card

```
JOIN match:        ON a.id = b.id
Anti-join:         LEFT JOIN ... WHERE b.id IS NULL  OR  NOT EXISTS
Top per group:     DENSE_RANK() OVER (PARTITION BY g ORDER BY x DESC) = 1
Running sum:       SUM(x) OVER (ORDER BY t)
Previous row:      LAG(x) OVER (ORDER BY t)
Dedupe keep one:   ROW_NUMBER() ... rn = 1
Consecutive IDs:   id - ROW_NUMBER() constant per streak
2nd highest:       DENSE_RANK = 2  OR  MAX WHERE < (SELECT MAX...)
Readable steps:    WITH cte1 AS (...), cte2 AS (...) SELECT ...
NULL safe:         COALESCE, IS NULL, NOT EXISTS not NOT IN
```

---

# SQL Execution Order (memorize)

```
FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT
```

---

# Dialect Cheat Sheet

| Feature | PostgreSQL | MySQL 8+ | SQL Server |
|---------|------------|----------|------------|
| `LIMIT/OFFSET` | Yes | Yes | `TOP` / `OFFSET-FETCH` |
| `FULL OUTER JOIN` | Yes | No (workaround) | Yes |
| Recursive CTE | Yes | Yes | Yes |
| `ILIKE` | Yes | No — use LIKE | No |
| `DATE_TRUNC` | Yes | `DATE_FORMAT` | `DATEADD` |
| Window functions | Yes | Yes | Yes |
| `BOOLEAN` | Yes | TINYINT(1) | BIT |

---

# 2-Week Practice Plan

| Week | Day | Focus | Example # |
|------|-----|-------|-----------|
| 1 | 1 | SELECT, WHERE | 1.1, 1.2, 10.2 |
| 1 | 2 | JOINs | 2.1–2.8 |
| 1 | 3 | GROUP BY, HAVING | 3.1–3.4 |
| 1 | 4 | Subqueries | 4.1–4.4 |
| 1 | 5 | CTEs | 5.1–5.4 |
| 1 | 6 | Recursive CTE | 5.3, 5.4 |
| 1 | 7 | Review | All Part 1–5 |
| 2 | 8 | Window rank | 6.1, 6.2 |
| 2 | 9 | LAG/LEAD | 6.4, 10.10 |
| 2 | 10 | Running totals | 6.3 |
| 2 | 11 | HackerRank | 8.1–8.4 |
| 2 | 12 | Big tech | 9.1–9.4 |
| 2 | 13 | CASE, NULL | 7.1–7.3 |
| 2 | 14 | Mock interview | Explain + optimize |

---

# Final Interview Mindset

1. **Sketch the tables** — draw or recall sample rows before writing SQL
2. **State expected output** — "I expect 2 rows: Gamma and Acme"
3. **Use CTEs** — interviewers read steps easier than nested subqueries
4. **Call out NULLs** — Delta Co, cancelled orders, tie ranks
5. **Optimize if asked** — index FKs, filter early, explain plan

You now have **50+ worked examples** with input tables and result sets — rehearse until you can predict the output before running the query.
