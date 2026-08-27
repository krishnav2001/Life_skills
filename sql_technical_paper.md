# Database Concepts — Quick Technical Notes

## 1. ACID Properties

ACID describes the properties that make database transactions reliable.

- **Atomicity** — A transaction is all-or-nothing.
- **Consistency** — A transaction moves the database from one valid state to another.
- **Isolation** — Concurrent transactions do not improperly interfere with each other.
- **Durability** — Committed changes survive failures.

### Example

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 10000
WHERE account_id = 101;

UPDATE accounts
SET balance = balance + 10000
WHERE account_id = 102;

COMMIT;
```

If an operation fails:

```sql
ROLLBACK;
```

---

## 2. CAP Theorem

CAP basically means:

- **Consistency** — Reads return the latest successful write or an error.
- **Availability** — Every request receives a response.
- **Partition Tolerance** — The system continues operating despite network failures.

During a network partition, a distributed system generally chooses between:

- **CP** — Consistency + Partition Tolerance
- **AP** — Availability + Partition Tolerance

---

## 3. SQL Joins

A join combines rows from related tables.

### INNER JOIN

Returns only matching rows.

```sql
SELECT e.name, d.department_name
FROM employees e
INNER JOIN departments d
    ON e.department_id = d.department_id;
```

### LEFT JOIN

Returns all rows from the left table and matching rows from the right.

### RIGHT JOIN

Returns all rows from the right table and matching rows from the left.

### FULL OUTER JOIN

Returns matching and unmatched rows from both tables.

### CROSS JOIN

Creates a Cartesian product.

If one table has 3 rows and another has 3 rows:

```text
3 × 3 = 9 rows
```

### SELF JOIN

A table is joined with itself, commonly used for employee-manager relationships.

---

## 4. Aggregations and Filters

Important SQL clauses:

- `WHERE`
- `GROUP BY`
- `HAVING`
- `ORDER BY`
- `LIMIT`

Common aggregate functions:

- `COUNT()`
- `SUM()`
- `AVG()`
- `MIN()`
- `MAX()`

### WHERE

Filters rows before grouping.

```sql
SELECT *
FROM employees
WHERE salary > 50000;
```

### GROUP BY

Groups rows for aggregation.

```sql
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id;
```

### HAVING

Filters groups after aggregation.

```sql
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5;
```

### WHERE vs HAVING

| WHERE | HAVING |
|---|---|
| Filters rows | Filters groups |
| Applied before grouping | Applied after grouping |

### Logical Processing Order

```text
FROM
↓
JOIN
↓
WHERE
↓
GROUP BY
↓
HAVING
↓
SELECT
↓
ORDER BY
↓
LIMIT
```

---

## 5. Normalization

Normalization organizes data to reduce redundancy and prevent data anomalies.

### 1NF — First Normal Form

Each column contains atomic values.

**Bad:**

```text
products = "Laptop, Mouse"
```

**Better:**

```text
Laptop
Mouse
```

### 2NF — Second Normal Form

Must be in 1NF and have no partial dependency on part of a composite key.

Example:

```text
(order_id, product_id) → quantity
product_id → product_name
```

Move product information into a separate `products` table.

### 3NF — Third Normal Form

Must be in 2NF and have no transitive dependency between non-key attributes.

Example:

```text
employee_id → department_id
department_id → department_name
```

Separate department information into a `departments` table.

### BCNF

A stronger form of 3NF where every determinant is a candidate key.



Excessive normalization can increase the number of joins, so practical systems balance normalization and performance.

---

## 6. Indexes

An index is a database structure that helps retrieve rows faster.

### Example

```sql
CREATE INDEX idx_employees_email
ON employees(email);
```

Useful for frequently searched columns:

- `WHERE`
- `JOIN`
- `ORDER BY`

### Composite Index

```sql
CREATE INDEX idx_employee_dept_salary
ON employees(department_id, salary);
```

Column order matters. An index beginning with `department_id` is generally most useful when queries constrain that column.

### Index Costs

Indexes:

- Improve read performance.
- Consume storage.
- Add work to `INSERT`, `UPDATE`, and `DELETE`.

Therefore, do not create indexes on every column.

Use query plans to investigate performance:

```sql
EXPLAIN ANALYZE
SELECT *
FROM employees
WHERE department_id = 10;
```

---

## 7. Transactions

A transaction is a sequence of one or more operations performed on a single database.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 5000
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 5000
WHERE account_id = 2;

COMMIT;
```

### Important Commands

```sql
BEGIN;       -- Start transaction
COMMIT;      -- Save changes
ROLLBACK;    -- Undo transaction
```

### SAVEPOINT

Allows partial rollback.

```sql
BEGIN;

UPDATE employees
SET salary = salary + 5000
WHERE department_id = 10;

SAVEPOINT salary_update;

UPDATE employees
SET salary = salary + 10000
WHERE department_id = 20;

ROLLBACK TO SAVEPOINT salary_update;

COMMIT;
```

---

## 8. Locking

Locks control concurrent access to database resources.

### Row-Level Lock

In PostgreSQL:

```sql
SELECT *
FROM accounts
WHERE account_id = 1
FOR UPDATE;
```

This locks the selected row for conflicting updates.

### Shared Lock

Allows compatible reads but prevents conflicting operations.

### Exclusive Lock

Used for modifications and prevents conflicting operations.

### Deadlock

A deadlock occurs when transactions wait for resources held by each other.

```text
Transaction A → waits for B
Transaction B → waits for A
```

Databases normally detect deadlocks and abort one transaction.

### Preventing Deadlocks

- Lock resources in a consistent order.
- Keep transactions short.
- Avoid unnecessary locks.
- Avoid external API calls while holding locks.
- Retry transactions after transient failures.

---

## 9. Isolation Levels

Isolation levels control how concurrent transactions interact.

The standard levels are:

1. Read Uncommitted
2. Read Committed
3. Repeatable Read
4. Serializable

### Dirty Read

Reading uncommitted data from another transaction.

### Non-Repeatable Read

Reading the same row twice and getting different committed values.

### Phantom Read

Repeating a range query and getting a different set of matching rows.

### Isolation Comparison

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | Possible in theory | Possible | Possible |
| Read Committed | No | Possible | Possible |
| Repeatable Read | No | No | Depends on database |
| Serializable | No | No | No |

### PostgreSQL

PostgreSQL does not allow true dirty reads; `READ UNCOMMITTED` behaves like `READ COMMITTED`.

PostgreSQL's `REPEATABLE READ` uses snapshots and prevents ordinary phantom changes within the transaction.

### Serializable

Provides the strongest standard isolation and makes concurrent transactions behave as if they were executed serially.

It may cause serialization failures, so applications should be prepared to retry transactions.

### Example

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

SELECT *
FROM accounts
WHERE account_id = 1;

COMMIT;
```

---

## 10. Triggers

A trigger automatically executes database logic when an event occurs.

### Common Events

- `INSERT`
- `UPDATE`
- `DELETE`

### Common Timing

- `BEFORE`
- `AFTER`

### Example

Automatically update a timestamp:

```sql
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employees_update_timestamp
BEFORE UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();
```

### Uses

- Audit logging
- Automatic timestamps
- Data validation
- Maintaining derived information

---



