# Docker Compose Setup

To run this workshop, you can use the following Docker Compose setup to easily spin up a PostgreSQL and PgAdmin instance.

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:17
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: sa
      POSTGRES_PASSWORD: sa
      POSTGRES_DB: testdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    restart: unless-stopped
    environment:
      PGADMIN_DEFAULT_EMAIL: sa@ai.com
      PGADMIN_DEFAULT_PASSWORD: sa     
      PGADMIN_SERVER_JSON_FILE: /pgadmin4/servers.json
    ports:
      - "8081:80"
    depends_on:
      - postgres
    volumes:
      - ./servers.json:/pgadmin4/servers.json:ro
volumes:
  postgres_data:
```

Save the above as `docker-compose.yml` and create a `servers.json` file in the same directory with the following content to automatically configure the server connection in PgAdmin:

```json
{
    "Servers": {
        "1": {
            "Name": "Postgres Workshop",
            "Group": "Servers",
            "Host": "postgres",
            "Port": 5432,
            "MaintenanceDB": "testdb",
            "Username": "sa",
            "SSLMode": "prefer"
        }
    }
}
```

Then run `docker-compose up -d` to start the services. You can access PgAdmin at `http://localhost:8081`.

# PostgreSQL Workshop: Understanding Scan Costs with EXPLAIN ANALYZE

This exercise demonstrates how PostgreSQL's query planner chooses different scan methods based on data distribution. We will use `EXPLAIN ANALYZE` to inspect the query plans.

This workshop is inspired by Bruce Momjian's presentation on the Postgres Query Optimizer.

### Updated Workshop Exercise: Dynamic Plan Analysis

This updated exercise uses a PL/pgSQL function to automatically run `EXPLAIN` for each department, showing a direct correlation between data frequency and the chosen query plan.

#### **Step 1: Setup the Temporary Table and Data**

This step creates a temporary table, inserts skewed data, and creates an index.

```sql
-- Using a temporary table ensures it's automatically dropped at the end of the session.
CREATE TEMPORARY TABLE employee_simple (
  id SERIAL PRIMARY KEY,
  name TEXT,
  department TEXT
);

-- Insert skewed data
INSERT INTO employee_simple (name, department)
SELECT 'Employee ' || i, 'Sales' FROM generate_series(1, 10000) i;

INSERT INTO employee_simple (name, department)
SELECT 'Employee ' || i, 'Engineering' FROM generate_series(10001, 10100) i;

INSERT INTO employee_simple (name, department)
SELECT 'Employee ' || i, 'HR' FROM generate_series(10101, 10110) i;

INSERT INTO employee_simple (name, department)
VALUES ('The Boss', 'Management');

-- Create the index
CREATE INDEX ON employee_simple(department);
```

#### **Step 2: View the Data Distribution**

Before we analyze query plans, let's confirm the data skew we created. This query calculates the number of employees in each department and their percentage of the total.

```sql
WITH department_counts (department, count) AS (
  SELECT
    department,
    COUNT(*)
  FROM employee_simple
  GROUP BY 1
)
SELECT
  department,
  count,
  (count * 100.0 / (SUM(count) OVER ()))::numeric(4,1) AS "%"
FROM department_counts
ORDER BY 2 DESC;
```

You will see an output like this, clearly showing that 'Sales' accounts for over 98% of the rows. This is the key information the planner will use after we run `ANALYZE`.

| department  | count | %      |
| :---------- | :---- | :----- |
| Sales       | 10000 | 98.9   |
| Engineering | 100   | 1.0    |
| HR          | 10    | 0.1    |
| Management  | 1     | 0.0    |


#### **Step 3: Manually Running EXPLAIN**

Before using the function, you can manually run `EXPLAIN` on each department to see the initial query plans (before `ANALYZE` has been run).

**For Sales (High Frequency):**
```sql
EXPLAIN SELECT * FROM employee_simple WHERE department = 'Sales';
```

**For Engineering (Medium Frequency):**
```sql
EXPLAIN SELECT * FROM employee_simple WHERE department = 'Engineering';
```

**For HR (Low Frequency):**
```sql
EXPLAIN SELECT * FROM employee_simple WHERE department = 'HR';
```

**For Management (Single Row):**
```sql
EXPLAIN SELECT * FROM employee_simple WHERE department = 'Management';
```

#### **Step 4: Create the `EXPLAIN` Function**

This function takes a department name as input and returns the query plan for selecting employees from that department. This makes it easy to compare the plans for all departments at once.

*Note: We use `EXECUTE ... USING` to safely pass `dept_name` as a parameter. This is the standard, secure way to execute dynamic queries in PL/pgSQL and prevents SQL injection.*

```sql
CREATE OR REPLACE FUNCTION get_department_plan(dept_name text)
RETURNS SETOF text AS $$
BEGIN
  RETURN QUERY EXECUTE
    'EXPLAIN SELECT * FROM employee_simple WHERE department = $1'
    USING dept_name;
END;
$$ LANGUAGE plpgsql;
```

#### **Step 5: Run the Combined Analysis Query Before and After `ANALYZE`**

This is the key step where we observe the planner's behavior.

**A. Run the Analysis Without Statistics**

First, run the combined analysis query *before* running `ANALYZE`. Without up-to-date statistics, the planner doesn't know about the skewed data distribution and will likely default to using an `Index Scan` or `Bitmap Heap Scan` for all departments, assuming it will be faster.

```sql
WITH department_counts (department, count) AS (
  SELECT
    department,
    COUNT(*)
  FROM employee_simple
  GROUP BY 1
)
SELECT
  department,
  count,
  (SELECT * FROM get_department_plan(department) LIMIT 1) AS query_plan
FROM department_counts
ORDER BY 2 DESC;
```

**B. Update Table Statistics**

Now, run `ANALYZE`. This command collects statistics about the data distribution in the table and stores them in the system catalogs. The query planner will use these statistics to make more informed decisions.

```sql
-- IMPORTANT: Manually run ANALYZE on temporary tables.
-- The autovacuum daemon does not process temporary tables,
-- so statistics must be updated manually for the planner to have accurate information.
ANALYZE employee_simple;
```

**C. Run the Analysis Again With Statistics**

Run the same analysis query again.

```sql
WITH department_counts (department, count) AS (
  SELECT
    department,
    COUNT(*)
  FROM employee_simple
  GROUP BY 1
)
SELECT
  department,
  count,
  (SELECT * FROM get_department_plan(department) LIMIT 1) AS query_plan
FROM department_counts
ORDER BY 2 DESC;
```

### Expected Output and Interpretation

**Before `ANALYZE`**

Without statistics, the planner defaults to its assumptions. It will likely choose an `Index Scan` or `Bitmap Heap Scan` for all queries because it doesn't know that the 'Sales' department is so large. The output will look something like this, with the planner making the same guess for every department:

| department  | count | query_plan                                                              |
| :---------- | :---- | :---------------------------------------------------------------------- |
| Sales       | 10000 | `Bitmap Heap Scan on employee_simple  (cost=4.68..69.57 rows=51 width=68)` |
| Engineering | 100   | `Bitmap Heap Scan on employee_simple  (cost=4.68..69.57 rows=51 width=68)` |
| HR          | 10    | `Bitmap Heap Scan on employee_simple  (cost=4.68..69.57 rows=51 width=68)` |
| Management  | 1     | `Bitmap Heap Scan on employee_simple  (cost=4.68..69.57 rows=51 width=68)` |

**After `ANALYZE`**

With statistics, the planner makes an informed choice. The exact plan for the smaller departments may vary slightly between `Index Scan` and `Bitmap Heap Scan` depending on your PostgreSQL version and configuration, but the principle remains the same.

| department  | count | query_plan                                                              |
| :---------- | :---- | :---------------------------------------------------------------------- |
| Sales       | 10000 | `Seq Scan on employee_simple  (cost=0.00..191.39 rows=10000 width=23)`   |
| Engineering | 100   | `Index Scan using employee_simple_department_idx on employee_simple...` |
| HR          | 10    | `Index Scan using employee_simple_department_idx on employee_simple...` |
| Management  | 1     | `Index Scan using employee_simple_department_idx on employee_simple...` |

This single query perfectly summarizes the behavior of the PostgreSQL optimizer:

*   **High Frequency (`Sales`):** The planner sees that 10,000 rows match. It calculates that reading the whole table (`Seq Scan`) is cheapest.
*   **Low to Medium Frequency (`Engineering`, `HR`, `Management`):** For a smaller number of rows, the planner determines it's much faster to use the index to find the few matching rows directly (`Index Scan`) rather than reading the entire table.

### Step 6: Forcing an Index Scan for High-Frequency Data

We saw that the planner correctly chose a `Seq Scan` for the 'Sales' department after `ANALYZE` because it was cheaper. But what would the cost be if it were forced to use an index? We can simulate this by disabling sequential and bitmap scans.

**A. Disable `seqscan` and `bitmapscan`**

These settings tell the planner to avoid these scan types if at all possible.

```sql
SET enable_seqscan = false;
SET enable_bitmapscan = false;
```

**B. Run `EXPLAIN` for the 'Sales' Department Again**

Now, run the `EXPLAIN` command for the 'Sales' department.

```sql
EXPLAIN SELECT * FROM employee_simple WHERE department = 'Sales';
```

**C. Compare the Costs**

You will see an output like this:

```
                                                          QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------
 ndex Scan using employee_simple_department_idx on employee_simple  (cost=0.29..311.49 rows=10000 width=23)
   Index Cond: (department = 'Sales'::text)
```

Compare this cost to the `Seq Scan` cost from before: `(cost=0.00..191.39 rows=10000 width=23)`.

*   **`Seq Scan` cost:** up to **191.39**
*   **Forced `Index Scan` cost:** up to **311.49**

This demonstrates *why* the planner chose the `Seq Scan`. Forcing an `Index Scan` on a large number of rows is significantly more expensive because it requires two steps:
1.  Scanning the index to find the location of all 10,000 rows.
2.  Visiting the table heap to fetch each of those 10,000 rows individually, which can be very inefficient if the rows are scattered across the table.

**D. Reset the settings**

It's good practice to reset the settings to their defaults.

```sql
RESET enable_seqscan;
RESET enable_bitmapscan;
```
