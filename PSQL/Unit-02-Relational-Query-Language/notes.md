# Unit 2 — Relational Query Languages

> This unit is the **hands-on heart of the course**: how we *talk* to a database using SQL — defining its structure, filling it with data, protecting it with constraints, and querying it.

## ✅ What you'll learn

- The 5 families of SQL commands: **DDL, DML, DQL, DCL, TCL**
- How to create/modify/delete tables and other schema objects
- All 6 **integrity constraints** and what each one prevents
- Every **database key** type (super, candidate, primary, alternate, foreign, composite, surrogate)
- Everyday **SQL operations**: `WHERE`, `DISTINCT`, `ORDER BY`, `LIKE`, `IN`, `BETWEEN`, `LIMIT`

---

## 📑 Table of Contents

1. [Introduction — Classification of SQL Commands](#1-introduction--classification-of-sql-commands)
2. [Sample Database Used in This Unit](#2-sample-database-used-in-this-unit)
3. [DDL — Data Definition Language](#3-ddl--data-definition-language)
4. [DML — Data Manipulation Language](#4-dml--data-manipulation-language)
5. [DQL — Data Query Language (SELECT)](#5-dql--data-query-language-select)
6. [DCL — Data Control Language](#6-dcl--data-control-language)
7. [TCL — Transaction Control Language](#7-tcl--transaction-control-language)
8. [Integrity Constraints](#8-integrity-constraints)
9. [Database Keys](#9-database-keys)
10. [SQL Basic Operations (Operators & Clauses)](#10-sql-basic-operations-operators--clauses)
11. [⚡ Quick Revision](#-quick-revision)
12. [📝 Practice Set](#-practice-set)

---

## 1. Introduction — Classification of SQL Commands

📌 SQL (Structured Query Language) is the standard language of relational databases. Its commands are grouped into **5 families** depending on *what* they do:

| Family | Full Form | Purpose | Commands |
|--------|-----------|---------|----------|
| **DDL** | Data Definition Language | Define / change **structure** (schema) | `CREATE`, `ALTER`, `DROP`, `TRUNCATE`, `RENAME` |
| **DML** | Data Manipulation Language | Change **data inside** tables | `INSERT`, `UPDATE`, `DELETE` |
| **DQL** | Data Query Language | **Read** data | `SELECT` |
| **DCL** | Data Control Language | **Rights & permissions** | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control Language | Group changes into **all-or-nothing** units | `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `BEGIN` |

### 🔹 Memory trick

- **DDL = shape** of the building (walls, rooms).
- **DML = furniture** you move in and out.
- **DQL = looking** at the furniture.
- **DCL = keys** — who may enter which room.
- **TCL = the save button** — save the game or go back.

### 🎯 Questions

**Q1. Why are DDL commands said to be "auto-committed"?**
**Answer:** DDL changes the *data dictionary* (metadata), not just rows. Most DBMSs (including PostgreSQL) commit the current transaction automatically when a DDL statement runs, so `ALTER TABLE` effects cannot be rolled back with a plain `ROLLBACK` afterwards.

**Q2. Which family does `TRUNCATE` belong to, and why is it special?**
**Answer:** `TRUNCATE` is DDL (it deallocates the table's storage pages rather than scanning and deleting rows one by one). It removes **all rows instantly**, cannot have a `WHERE` clause, and (unlike `DELETE`) is not row-by-row.

---

## 2. Sample Database Used in This Unit

📌 Every example below runs on this tiny **college** database. Run this block once in `psql` first.

```sql
CREATE DATABASE college;                          -- create an empty database named 'college'
\c college                                        -- psql meta-command: connect to the 'college' database

CREATE TABLE departments (                        -- parent table: one row per department
  dept_id      INT PRIMARY KEY,                   -- unique department number (identity of the row)
  dept_name    VARCHAR(30) NOT NULL UNIQUE        -- department name; mandatory and no duplicates
);                                                -- end of the departments table definition

INSERT INTO departments VALUES (1, 'CSE');        -- add department 1: Computer Science
INSERT INTO departments VALUES (2, 'ECE');        -- add department 2: Electronics
INSERT INTO departments VALUES (3, 'ME');         -- add department 3: Mechanical

CREATE TABLE students (                           -- child table: one row per student
  roll_no      INT PRIMARY KEY,                   -- roll number uniquely identifies each student
  sname        VARCHAR(50) NOT NULL,              -- student name; can never be NULL/empty
  email        VARCHAR(80) UNIQUE,                -- optional but must be unique if provided
  age          INT CHECK (age >= 17),             -- only students aged 17 or above are valid
  fee_paid     NUMERIC(8,2) DEFAULT 0,            -- if no fee given, store 0.00 automatically
  dept_id      INT REFERENCES departments(dept_id) -- foreign key: must exist in departments
);                                                -- end of the students table definition

INSERT INTO students VALUES (101, 'Aarav', 'aarav@x.com', 20, 45000, 1);  -- row 1 → CSE
INSERT INTO students VALUES (102, 'Diya',  NULL,          19, 52000, 2);  -- row 2 → ECE, no email
INSERT INTO students VALUES (103, 'Kabir', 'kabir@x.com', 22, 0,     1);  -- row 3 → CSE
INSERT INTO students VALUES (104, 'Meera', NULL,          17, 48000, 3);  -- row 4 → ME, no email
```

**Line-by-line walkthrough:**
- `CREATE DATABASE` makes the database itself; `\c` (a psql shortcut, not SQL) moves your session into it.
- `departments` is created **first** because `students` points at it — a foreign key needs its target to exist.
- `INSERT INTO ... VALUES` lists values in the **exact column order** of the table.
- `NULL` in an INSERT means "no value" — allowed here for `email` because it is only `UNIQUE`, not `NOT NULL`.

---

## 3. DDL — Data Definition Language

📌 DDL defines and changes the **structure**: tables, columns, types, constraints.

### 🔹 3.1 CREATE — build new objects

```sql
CREATE TABLE books (                    -- create a new table called 'books'
  book_id   SERIAL PRIMARY KEY,         -- SERIAL = auto-numbering integer (1,2,3...) + primary key
  title     VARCHAR(100) NOT NULL,      -- book title; required field
  price     NUMERIC(8,2) CHECK (price > 0)  -- price must be a positive amount
);                                      -- end of table definition
```

**Line-by-line walkthrough:**
- `SERIAL` is PostgreSQL's convenience type: it silently creates a sequence and fills `book_id` for you.
- `PRIMARY KEY` = `UNIQUE + NOT NULL` + an index built automatically for fast lookups.

### 🔹 3.2 ALTER — change structure of an existing object

```sql
ALTER TABLE books ADD COLUMN author VARCHAR(60);        -- add a new column (existing rows get NULL)
ALTER TABLE books ALTER COLUMN title TYPE VARCHAR(150); -- widen the title column to 150 chars
ALTER TABLE books RENAME COLUMN title TO book_title;    -- rename a column safely
ALTER TABLE books RENAME TO textbooks;                  -- rename the whole table
ALTER TABLE books DROP COLUMN author;                   -- remove a column (data in it is lost!)
```

**Line-by-line walkthrough:**
- `ADD COLUMN` is instant — PostgreSQL only updates the catalog, existing rows just read NULL for the new column.
- `ALTER COLUMN ... TYPE` rewrites the table, so on huge tables it can take time.
- `RENAME` is metadata-only: no data moves, all permissions/views referencing it are updated.
- `DROP COLUMN` is destructive — there is no "undo" without a backup.

### 🔹 3.3 DROP vs TRUNCATE vs DELETE — the classic exam question

```sql
DELETE FROM books;              -- DML: removes row by row, logs each one, fires triggers, can be rolled back
TRUNCATE TABLE books;           -- DDL: deallocates all pages at once — instant, no WHERE allowed
DROP TABLE books;               -- DDL: deletes the table itself (structure + data + indexes gone)
```

| | `DELETE` | `TRUNCATE` | `DROP` |
|---|---|---|---|
| Family | DML | DDL | DDL |
| Removes | selected/all **rows** | **all rows** | whole **table** |
| `WHERE` allowed? | ✅ | ❌ | ❌ |
| Speed on big tables | slow (row-by-row) | very fast | very fast |
| Rollback-able? | ✅ (in a transaction) | ❌ usually | ❌ |
| Triggers fire? | ✅ | ❌ (in PG: `TRUNCATE` triggers only) | ❌ |

### 🎯 Questions

**Q1. You dropped a column by mistake. Can `ROLLBACK` bring it back?**
**Answer:** No. `ALTER TABLE ... DROP COLUMN` is DDL and auto-commits. Recovery is only possible from a backup (or if the DDL ran inside an explicit transaction block and you roll back *before* it commits).

**Q2. Why is `TRUNCATE` faster than `DELETE FROM t;`?**
**Answer:** `DELETE` must find every row, write a log record per row, and maintain indexes/triggers. `TRUNCATE` simply unlinks the storage pages — one metadata operation — so time is independent of row count.

**Q3. Difference between dropping a table and truncating it?**
**Answer:** `TRUNCATE` leaves an **empty table with the same structure**; `DROP` removes the **structure itself**, so any later `SELECT` fails with *relation does not exist*.

---

## 4. DML — Data Manipulation Language

📌 DML works on the **rows** (the instance), not the structure.

### 🔹 4.1 INSERT — add rows (3 forms)

```sql
INSERT INTO students (roll_no, sname, dept_id)        -- name only the columns you have values for
VALUES (105, 'Ishaan', 2);                            -- fee_paid defaults to 0, email/age become NULL

INSERT INTO students VALUES                           -- full form: values for EVERY column, in order
(106, 'Riya', 'riya@x.com', 21, 50000, 1),
(107, 'Vivaan', 'viv@x.com', 18, 47000, 2);           -- multiple rows inserted in ONE statement

INSERT INTO students_backup                           -- copy form: insert the result of a SELECT
SELECT roll_no, sname FROM students WHERE dept_id = 1 -- only CSE students are copied
ON CONFLICT DO NOTHING;                               -- skip rows that would violate a constraint
```

**Line-by-line walkthrough:**
- **Named-column form** is safest: column order changes in the future won't silently corrupt your inserts.
- The **multi-row form** is one network round trip — much faster than N separate INSERTs.
- The **INSERT ... SELECT** form copies rows between tables; column lists on both sides must match in count and type.

### 🔹 4.2 UPDATE — change existing rows

```sql
UPDATE students                          -- which table to modify
SET fee_paid = fee_paid * 1.10           -- increase every fee by 10% (arithmetic on old value)
WHERE dept_id = 1;                       -- ⚠️ only CSE rows; without WHERE, ALL rows change!
```

**Line-by-line walkthrough:**
- `SET` may reference the old value (`fee_paid * 1.10`) — the right side is evaluated per row.
- **Forgetting `WHERE` is the most expensive typo in SQL** — the update silently hits every row.
- Rows whose new value violates a constraint (e.g., `CHECK (age >= 17)`) make the whole statement fail atomically.

### 🔹 4.3 DELETE — remove rows

```sql
DELETE FROM students                     -- remove rows...
WHERE age < 18;                          -- ...only those matching the condition (Meera, age 17)

DELETE FROM students;                    -- ⚠️ no WHERE → deletes EVERY row (table still exists)
```

**Line-by-line walkthrough:**
- DELETE is row-by-row: every removed row is logged, triggers fire, and it can be rolled back inside a transaction.
- If other tables reference a deleted row via foreign key, the FK rule (`CASCADE`/`SET NULL`/`RESTRICT`) decides what happens — see Section 8.

### 🎯 Questions

**Q1. Predict the effect:** `UPDATE students SET age = 16;` given `CHECK (age >= 17)`.
**Answer:** The statement fails for the first row that violates the check, and the **entire update is rolled back** (statement atomicity). No row keeps the old... no row is changed at all.

**Q2. Difference between `DELETE FROM students WHERE dept_id = 1;` and `TRUNCATE students;`?**
**Answer:** Both end with an empty table, but DELETE scans/filters/logs each row (can target a subset, fires row triggers, is rollback-able), while TRUNCATE instantly clears everything, cannot filter, and is treated as DDL.

**Q3. How do you insert a row supplying values only for some columns?**
**Answer:** Use the named-column form: `INSERT INTO t (col1, col3) VALUES (v1, v3);` — omitted columns receive their `DEFAULT` if one exists, otherwise `NULL` (and error if the column is `NOT NULL` without default).

---

## 5. DQL — Data Query Language (SELECT)

📌 `SELECT` never changes data — it **reads** it. Full clause order matters:

```text
SELECT   → which columns           (5th: project)
FROM     → which tables            (1st: source rows)
WHERE    → filter rows             (2nd: row filter)
GROUP BY → form groups             (3rd: aggregate)
HAVING   → filter groups           (4th: group filter)
ORDER BY → sort result             (6th: sort)
LIMIT    → keep only N rows        (7th: page)
```

```sql
SELECT sname, fee_paid                    -- project only these two columns
FROM students                             -- read from the students table
WHERE dept_id = 1                         -- keep only CSE students (row filter)
ORDER BY fee_paid DESC                    -- highest fee first (DESC = descending)
LIMIT 2;                                  -- stop after 2 rows (top-2 query)
```

**Line-by-line walkthrough:**
- Even though `SELECT` is written first, the DBMS logically evaluates `FROM → WHERE → SELECT → ORDER BY → LIMIT`.
- `DESC` is needed because `ORDER BY` defaults to ascending (`ASC`).
- `LIMIT 2` after `ORDER BY` = "top 2 by fee" — order of the two clauses matters.

### 🎯 Questions

**Q1. Can you use a column alias created in SELECT inside the WHERE clause?**
**Answer:** No. `WHERE` is evaluated *before* `SELECT`, so the alias doesn't exist yet there. Repeat the expression, or filter in an outer query / use `HAVING`-style ordering only in `ORDER BY` (where aliases *are* allowed).

**Q2. What does `SELECT *` cost you?**
**Answer:** It reads and ships **every column** — more I/O and memory than needed, breaks when the table gains columns, and defeats covering indexes. In production, name the columns.

---

## 6. DCL — Data Control Language

📌 DCL answers **"who is allowed to do what?"** — users and privileges.

```sql
CREATE USER exam_user WITH PASSWORD 'secret123';      -- create a login role with a password

GRANT SELECT, INSERT ON students TO exam_user;        -- allow reading and inserting...
GRANT SELECT ON departments TO exam_user;             -- ...but on departments only reading
GRANT ALL PRIVILEGES ON DATABASE college TO exam_user;-- broad grant at database level (be careful!)

REVOKE INSERT ON students FROM exam_user;             -- take the INSERT right back
REVOKE ALL ON students FROM exam_user;                -- strip every privilege on the table
```

**Line-by-line walkthrough:**
- Privileges cascade downward: `SELECT ON students` permits `SELECT *`, column lists, joins — everything read-only on that table.
- `GRANT ALL` is convenient for admins and dangerous for apps — grant the **minimum needed** (principle of least privilege).
- `REVOKE` removes exactly what was granted; other privileges the user has remain untouched.

### 🎯 Questions

**Q1. A reporting app must only read data. Which commands set this up?**
**Answer:** `CREATE USER app_read ...;` then `GRANT SELECT ON students, departments TO app_read;` — nothing else, so any write attempt by the app fails with *permission denied*.

**Q2. Who can grant privileges?**
**Answer:** The object's **owner**, a **superuser**, or anyone who was granted that privilege **WITH GRANT OPTION** (allowing them to pass it on).

---

## 7. TCL — Transaction Control Language

📌 A **transaction** is a group of statements that must succeed or fail **as one unit** (all-or-nothing).

```sql
BEGIN;                                       -- start a transaction (nothing is final yet)

UPDATE accounts SET balance = balance - 1000 -- withdraw ₹1000 from account 1
WHERE acc_no = 1;                            -- (the sender)

UPDATE accounts SET balance = balance + 1000 -- deposit ₹1000 into account 2
WHERE acc_no = 2;                            -- (the receiver)

COMMIT;                                      -- 👍 make BOTH changes permanent
```

```sql
BEGIN;                                       -- another transfer attempt
UPDATE accounts SET balance = balance - 1000 WHERE acc_no = 1;  -- money left account 1 (in-tx only)
SAVEPOINT before_deposit;                    -- mark a checkpoint inside the transaction
UPDATE accounts SET balance = balance + 1000 WHERE acc_no = 99; -- ❌ wrong account number!
ROLLBACK TO before_deposit;                  -- undo ONLY the deposit, keep the withdrawal pending
ROLLBACK;                                    -- abandon everything: withdrawal also undone
```

**Line-by-line walkthrough:**
- Between `BEGIN` and `COMMIT`, other sessions **cannot see** your changes (isolation).
- `COMMIT` = permanent (durable); `ROLLBACK` = as if the transaction never ran.
- `SAVEPOINT` creates a **partial undo marker**: `ROLLBACK TO savepoint` undoes only what came after it.
- The classic example: money transfer must be **atomic** — debit and credit succeed together or fail together.

### 🎯 Questions

**Q1. Why must a bank transfer be one transaction?**
**Answer:** If the debit committed but the credit failed, money would vanish. Wrapping both updates in one transaction guarantees **atomicity**: both are applied or neither is.

**Q2. Order these by scope:** `COMMIT`, `ROLLBACK TO sp`, `ROLLBACK`.
**Answer:** `ROLLBACK` undoes the whole transaction; `ROLLBACK TO sp` undoes only work after the savepoint (transaction stays open); `COMMIT` undoes nothing — it finalizes everything.

---

## 8. Integrity Constraints

📌 Constraints are **rules the DBMS enforces automatically** on every INSERT/UPDATE/DELETE. They protect correctness even against buggy applications.

| Constraint | Guarantees | Example |
|------------|-----------|---------|
| `NOT NULL` | a value must exist | `sname VARCHAR(50) NOT NULL` |
| `UNIQUE` | no duplicate values (NULLs allowed) | `email VARCHAR(80) UNIQUE` |
| `PRIMARY KEY` | `UNIQUE + NOT NULL` — row identity | `roll_no INT PRIMARY KEY` |
| `FOREIGN KEY` | value must exist in parent table | `dept_id REFERENCES departments` |
| `CHECK` | value satisfies a condition | `age >= 17` |
| `DEFAULT` | value used when none supplied | `fee_paid DEFAULT 0` |

### 🔹 8.1 Named constraints + violation demo

```sql
CREATE TABLE employees (                          -- create table with NAMED constraints
  emp_id     INT CONSTRAINT emp_pk PRIMARY KEY,   -- name 'emp_pk' appears in error messages
  ename      VARCHAR(40) CONSTRAINT emp_name_nn NOT NULL,  -- name can never be NULL
  salary     NUMERIC(9,2) CONSTRAINT emp_sal_chk CHECK (salary > 10000),  -- sane salary floor
  dept_no    INT CONSTRAINT emp_dept_fk REFERENCES departments(dept_id)   -- FK to departments
);

INSERT INTO employees VALUES (1, 'Ravi', 5000, 1);    -- ❌ fails: violates emp_sal_chk (5000 ≤ 10000)
-- ERROR:  new row for relation "employees" violates check constraint "emp_sal_chk"
```

**Line-by-line walkthrough:**
- `CONSTRAINT name` is optional but **recommended**: the name is what you see in errors and use in `ALTER TABLE ... DROP CONSTRAINT`.
- The INSERT fails **atomically** — nothing partial is stored.

### 🔹 8.2 FOREIGN KEY behaviour on parent deletes

```sql
ALTER TABLE students                              -- modify the existing students table
DROP CONSTRAINT students_dept_id_fkey,            -- remove the default-named FK constraint
ADD CONSTRAINT students_dept_fk                   -- re-add it with explicit name and rule...
FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
ON DELETE CASCADE;                                -- ...deleting a department deletes its students too

DELETE FROM departments WHERE dept_id = 3;        -- remove ME → students 104 vanishes automatically
```

| Rule | Meaning when parent row is deleted |
|------|------------------------------------|
| `RESTRICT` / `NO ACTION` (default) | ❌ refuse the delete while children exist |
| `CASCADE` | delete all matching child rows too |
| `SET NULL` | set child's FK column to NULL |
| `SET DEFAULT` | set child's FK column to its default |

**Line-by-line walkthrough:**
- PostgreSQL auto-names constraints like `students_dept_id_fkey` when you don't name them — check names with `\d students`.
- `CASCADE` is powerful and dangerous: deleting one department silently removes many students.
- Choose `RESTRICT` when children must never disappear implicitly (e.g., invoices → customer).

### 🎯 Questions

**Q1. Can a PRIMARY KEY column contain NULL? Can a UNIQUE column?**
**Answer:** PK: never (`UNIQUE + NOT NULL`). UNIQUE: yes — SQL treats each NULL as "unknown", so PostgreSQL allows multiple NULLs in a UNIQUE column (use `UNIQUE NULLS NOT DISTINCT` to forbid that too).

**Q2. Which constraint prevents a student from enrolling in department number 99?**
**Answer:** The **foreign key** (`REFERENCES departments`) — the DBMS checks dept 99 exists before accepting the row; otherwise: *insert or update on table "students" violates foreign key constraint*.

**Q3. Difference between entity integrity and referential integrity?**
**Answer:** **Entity integrity**: every table has a PK and PK values are unique/non-null (each row is identifiable). **Referential integrity**: every FK value either matches an existing parent key or is NULL (no orphan rows).

---

## 9. Database Keys

📌 Keys let us **uniquely identify rows** and **link tables**. One worked example, then definitions:

```text
STUDENT (roll_no, aadhaar, name, email, (dept_id, year))
Candidate keys (minimal unique identifiers): {roll_no}, {aadhaar}, {email}
```

| Key | Meaning | In the example |
|-----|---------|----------------|
| **Super key** | ANY attribute set that uniquely identifies a row (may contain extras) | `{roll_no, name}`, `{aadhaar, email}` |
| **Candidate key** | **minimal** super key — no extra attributes | `{roll_no}`, `{aadhaar}`, `{email}` |
| **Primary key** | the candidate key the designer **chooses** | `{roll_no}` |
| **Alternate key** | candidate keys **not chosen** as PK | `{aadhaar}`, `{email}` |
| **Composite key** | key made of **≥ 2 columns** | `{(dept_id, year)}` if unique per pair |
| **Foreign key** | column referencing a key of **another** table | `students.dept_id → departments.dept_id` |
| **Surrogate key** | artificial auto-generated key with no business meaning | `SERIAL id` |

```sql
CREATE TABLE enrollment (                     -- classic composite-key table
  dept_id  INT,                               -- part 1 of the composite key
  year     INT,                               -- part 2 of the composite key
  strength INT NOT NULL,                      -- some measured value
  PRIMARY KEY (dept_id, year)                 -- composite PK: the PAIR must be unique
);
```

**Line-by-line walkthrough:**
- `{roll_no, name}` is a super key but not candidate — remove `name` and it still identifies rows, so it wasn't minimal.
- Uniqueness of a composite key applies to the **combination**, not each column: `(1,2024)` and `(2,2024)` coexist fine.
- Surrogate keys (SERIAL/IDENTITY/UUID) are stable when business keys change (e.g., someone's email changes).

### 🔹 Primary key vs Unique key (frequent 5-marker)

| | PRIMARY KEY | UNIQUE |
|---|---|---|
| NULLs allowed | ❌ never | ✅ (by default) |
| How many per table | exactly one | many |
| Implies index | ✅ | ✅ |
| Purpose | row **identity** | avoid duplicates in a secondary field |

### 🎯 Questions

**Q1. A table has candidate keys {emp_id} and {pan_no}. emp_id is the PK. What is {pan_no}?**
**Answer:** An **alternate key** — it qualified as a candidate key but was not chosen as primary.

**Q2. Can a foreign key be NULL? Can it duplicate?**
**Answer:** Yes and yes — NULL means "not linked yet" (if the column isn't NOT NULL), and many child rows may reference the **same** parent (many-to-one). FK uniqueness is *not* required.

**Q3. Why prefer a surrogate key over Aadhaar/email as PK?**
**Answer:** Business identifiers change, are sensitive, or may be reformatted; a surrogate never changes, keeps FK joins cheap (small fixed-size), and avoids exposing sensitive data everywhere.

---

## 10. SQL Basic Operations (Operators & Clauses)

📌 The everyday toolbox for filtering and shaping results.

### 🔹 10.1 DISTINCT — remove duplicate output rows

```sql
SELECT DISTINCT dept_id                    -- one output row per distinct value
FROM students;                             -- → 1, 2, 3 (not every student's dept repeated)
```

### 🔹 10.2 WHERE operators — the complete starter kit

```sql
SELECT sname FROM students WHERE age = 20;             -- equality
SELECT sname FROM students WHERE age <> 20;            -- not equal (also !=)
SELECT sname FROM students WHERE age > 19 AND fee_paid < 50000;  -- both conditions must hold
SELECT sname FROM students WHERE dept_id = 1 OR dept_id = 3;     -- either condition
SELECT sname FROM students WHERE dept_id IN (1, 3);              -- membership in a list (= OR list, shorter)
SELECT sname FROM students WHERE age BETWEEN 18 AND 21;          -- inclusive range [18, 21]
SELECT sname FROM students WHERE sname LIKE 'A%';                -- starts with 'A' (% = any string)
SELECT sname FROM students WHERE sname LIKE '_i%';               -- 2nd letter 'i' (_ = exactly one char)
SELECT sname FROM students WHERE email IS NULL;                  -- find missing emails (⚠️ = NULL never works)
SELECT sname, fee_paid + 500 AS new_fee FROM students;           -- arithmetic + column alias
```

**Line-by-line walkthrough:**
- `IN (1, 3)` is exactly `dept_id = 1 OR dept_id = 3`, just cleaner and optimizable.
- `BETWEEN a AND b` **includes both** endpoints.
- LIKE wildcards: `%` = zero or more characters, `_` = exactly one. `'A%'` → Aarav, not Kabir.
- ⚠️ `WHERE email = NULL` is **always unknown** and matches nothing — NULL tests must use `IS NULL` / `IS NOT NULL`.
- `AS new_fee` renames the output column only — the table data is untouched.

### 🔹 10.3 LIMIT / OFFSET — pagination

```sql
SELECT sname FROM students      -- simple list of names
ORDER BY roll_no                -- deterministic order (always order when paging!)
LIMIT 2 OFFSET 1;               -- skip 1 row, then return 2 → rows #2 and #3
```

**Line-by-line walkthrough:**
- Without `ORDER BY`, row order is unspecified — page 2 might repeat page 1's rows.
- `OFFSET 2 LIMIT 2` on page 1, `OFFSET 0 LIMIT 2`... pages read: page n = `LIMIT size OFFSET (n-1)*size`.

### 🎯 Questions

**Q1. Why does `WHERE email = NULL` return no rows?**
**Answer:** Comparisons with NULL yield **UNKNOWN**, not TRUE, and WHERE keeps only TRUE rows. NULL must be tested with `IS NULL`.

**Q2. Write a query for students whose name ends with 'a' and are in dept 1 or 2, youngest first.**
**Answer:**
```sql
SELECT * FROM students                                  -- all columns
WHERE sname LIKE '%a'                                   -- name ends with letter 'a'
  AND dept_id IN (1, 2)                                 -- department 1 or 2
ORDER BY age ASC;                                       -- youngest first
```

**Q3. What's the difference between `IN` and `BETWEEN`?**
**Answer:** `IN` matches a **discrete list** of values; `BETWEEN` matches a **continuous inclusive range**. `age IN (18, 21)` hits exactly two values; `age BETWEEN 18 AND 21` hits 18, 19, 20, 21.

---

## ⚡ Quick Revision

| Concept | One-liner / Syntax |
|---------|-------------------|
| DDL | structure: `CREATE / ALTER / DROP / TRUNCATE / RENAME` |
| DML | rows: `INSERT / UPDATE / DELETE` |
| DQL | read: `SELECT ... FROM ... WHERE` |
| DCL | rights: `GRANT / REVOKE` |
| TCL | unit-of-work: `BEGIN / COMMIT / ROLLBACK / SAVEPOINT` |
| `DELETE` vs `TRUNCATE` vs `DROP` | rows (rollback-able) / all rows fast (DDL) / table gone |
| NOT NULL | value must be supplied |
| UNIQUE | no duplicates, NULLs allowed |
| PRIMARY KEY | UNIQUE + NOT NULL, one per table, row identity |
| FOREIGN KEY | child value must exist in parent (`REFERENCES`) |
| `ON DELETE CASCADE` | parent gone → children gone |
| CHECK | row must satisfy a condition |
| DEFAULT | auto value when none given |
| Super key | unique, may have extra columns |
| Candidate key | minimal super key |
| Primary / Alternate key | chosen candidate / the other candidates |
| Composite key | key of ≥ 2 columns together |
| Surrogate key | artificial auto number (SERIAL/IDENTITY) |
| `DISTINCT` | deduplicate output rows |
| `LIKE` | pattern: `%` many chars, `_` one char |
| `IN` | membership in a value list |
| `BETWEEN` | inclusive range |
| `IS NULL` | the only correct NULL test |
| `LIMIT n OFFSET m` | return n rows after skipping m |

---

## 📝 Practice Set

### 2-mark questions

**P1. Name the SQL family: `GRANT`, `SAVEPOINT`, `ALTER TABLE`, `INSERT`.**
**Answer:** `GRANT` → DCL · `SAVEPOINT` → TCL · `ALTER TABLE` → DDL · `INSERT` → DML.

**P2. What does `SELECT DISTINCT dept_id FROM students;` return for our sample data?**
**Answer:** Three rows — 1, 2, 3 — duplicates collapsed to one row each.

**P3. Which constraint auto-creates an index in PostgreSQL?**
**Answer:** `PRIMARY KEY` (and `UNIQUE`) — both need an index to enforce uniqueness efficiently.

**P4. Write the SQL to find students with no email address.**
**Answer:** `SELECT sname FROM students WHERE email IS NULL;` (rows: Diya, Meera).

### 5-mark questions

**P5. Compare DELETE, TRUNCATE, DROP on: family, WHERE, speed, rollback.**
**Answer:** DELETE — DML, WHERE ✅, slow row-by-row, rollback-able in a transaction. TRUNCATE — DDL, WHERE ❌, instant page deallocation, not rollback-able (in PG outside a tx block). DROP — DDL, removes the table itself; later queries error with *relation does not exist*.

**P6. Explain all six integrity constraints with one example each.**
**Answer:** See the table in Section 8 — NOT NULL (sname), UNIQUE (email), PRIMARY KEY (roll_no), FOREIGN KEY (dept_id → departments), CHECK (age ≥ 17), DEFAULT (fee_paid = 0); each is enforced by the DBMS on every write.

**P7. Given FDs {roll_no → aadhaar, email} — classify: {roll_no, name} super or candidate? What are the alternate keys if roll_no is PK?**
**Answer:** {roll_no, name} is a **super key** (contains extra attribute `name`, not minimal). Candidate keys are {roll_no}, {aadhaar}, {email}; with {roll_no} as PK, the **alternate keys** are {aadhaar} and {email}.

**P8. A transaction debits account A then credits account B, but the system crashes after the debit. What saves you, and which property is this?**
**Answer:** The log/undo records kept since `BEGIN`; on restart the uncommitted transaction is rolled back. This is **atomicity** (all-or-nothing), supported by TCL's `COMMIT`/`ROLLBACK`.

### Lab / hands-on questions

**P9. Create a `courses` table: course_id auto-number PK, title required unique, fee positive with default 10000, and link it to departments.**
**Answer:**
```sql
CREATE TABLE courses (                                  -- new table
  course_id  SERIAL PRIMARY KEY,                        -- auto-increment identity
  title      VARCHAR(60) NOT NULL UNIQUE,               -- mandatory, no duplicates
  fee        NUMERIC(8,2) DEFAULT 10000 CHECK (fee > 0),-- default + positivity rule
  dept_id    INT REFERENCES departments(dept_id)        -- FK to departments
);                                                      -- done
```

**P10. Raise every CSE student's fee by ₹5,000, but only if the result stays under 60000.**
**Answer:**
```sql
UPDATE students                                         -- modify rows
SET fee_paid = fee_paid + 5000                          -- arithmetic on current value
WHERE dept_id = 1                                       -- CSE students only
  AND fee_paid + 5000 < 60000;                          -- guard: keep the result under 60000
```

**P11. Show page 2 (2 students per page) of students ordered by roll_no.**
**Answer:**
```sql
SELECT roll_no, sname FROM students                     -- choose columns
ORDER BY roll_no                                        -- stable order
LIMIT 2 OFFSET 2;                                       -- skip first 2, take next 2 → rows 3 & 4
```

**P12. Demonstrate (in psql) what error appears when inserting a student with dept_id = 99.**
**Answer:**
```sql
INSERT INTO students VALUES (108, 'Test', NULL, 20, 0, 99);  -- dept 99 does not exist
-- ERROR: insert or update on table "students" violates foreign key constraint
-- DETAIL: Key (dept_id)=(99) is not present in table "departments".
```
This is **referential integrity** working — the FK check rejects orphan rows.

**P13. Safe way to test TRUNCATE without losing data forever?**
**Answer:** Wrap it in a transaction and inspect before committing:
```sql
BEGIN;                          -- open a transaction
TRUNCATE students;              -- clear it (inside the tx only)
SELECT count(*) FROM students;  -- → 0 (visible only to your session)
ROLLBACK;                       -- undo: all rows are back
```
(In PostgreSQL DDL like TRUNCATE *is* transactional — a luxury most other DBMSs don't offer.)

---

*INT306 · Unit 2 notes · Session 2025-26*
