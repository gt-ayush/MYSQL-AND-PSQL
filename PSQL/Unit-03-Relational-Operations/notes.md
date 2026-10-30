# Unit 3 — Relational Operations

📌 **INT306 · Database Management Systems · PostgreSQL (psql) edition**

This unit is the *workshop* of DBMS: after learning how to model data, you now learn how to **slice, combine, group, rank and speed up** that data. Everything below runs as-is in `psql` — every line is commented, and every query shows its expected output, so you can follow along by pasting into a terminal.

**✅ What you'll learn**

- 🔹 Aggregate functions — `COUNT, SUM, AVG, MIN, MAX`, `GROUP BY`, `HAVING` vs `WHERE`, and NULL behaviour
- 🔹 All SQL joins — INNER, LEFT/RIGHT/FULL OUTER, CROSS, SELF, NATURAL, non-equijoin, and 3-table joins
- 🔹 Set operators — `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT` (Oracle's `MINUS`)
- 🔹 Views — normal, updatable, `WITH CHECK OPTION`, materialized, ALTER/DROP
- 🔹 Subqueries — single-row, multi-row, correlated, `EXISTS`, derived tables, subqueries in DML
- 🔹 Relational algebra — σ, π, ∪, ∩, −, ×, ⋈, ρ, ÷ with exam-style worked examples
- 🔹 Window functions — `OVER`, `PARTITION BY`, `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`, `LAG`, `LEAD`, running totals
- 🔹 Hashing — static, extendible, and PostgreSQL hash indexes
- 🔹 Indexing — B-tree, unique/partial/multi-column, when NOT to index
- 🔹 Query optimization — parse→plan→execute, cost & cardinality, `EXPLAIN` / `EXPLAIN ANALYZE`

## 📑 Table of Contents

- [0. Sample Database (Used in Every Section)](#0-sample-database-used-in-every-section)
- [1. Aggregate Functions](#1-aggregate-functions)
- [2. SQL Joins](#2-sql-joins)
- [3. Set Operators](#3-set-operators)
- [4. Views](#4-views)
- [5. Subqueries](#5-subqueries)
- [6. Relational Algebra](#6-relational-algebra)
- [7. Window Functions](#7-window-functions)
- [8. Hashing](#8-hashing)
- [9. Indexing](#9-indexing)
- [10. Query Optimization](#10-query-optimization)
- [11. Quick Revision Cheat Sheet](#11-quick-revision-cheat-sheet)
- [12. Practice Set](#12-practice-set)

---

## 0. Sample Database (Used in Every Section)

📌 Run this **one** block first. It builds three small tables with *deliberate quirks* (a NULL salary, an orphan department id, a department with no employees, a salary tie) so every concept later has a visible, predictable effect.

```sql
DROP TABLE IF EXISTS employees;      -- remove leftover copies so this script is re-runnable
DROP TABLE IF EXISTS departments;    -- remove leftover copies so this script is re-runnable
DROP TABLE IF EXISTS salary_grades;  -- remove leftover copies so this script is re-runnable

CREATE TABLE departments (           -- lookup table: one row per department
  dept_id    INT PRIMARY KEY,       -- unique department number
  dept_name  VARCHAR(30) NOT NULL,  -- department title (never empty)
  location   VARCHAR(30)            -- city of the department (nullable)
);

CREATE TABLE employees (             -- main table: one row per employee
  emp_id     INT PRIMARY KEY,       -- unique employee number
  emp_name   VARCHAR(30) NOT NULL,  -- employee name
  salary     INT,                   -- salary; NULL allowed on purpose (Neha)
  dept_id    INT,                   -- department number; NO foreign key on purpose (see Vijay)
  manager_id INT,                   -- emp_id of the boss; NULL means top-level
  hire_date  DATE                   -- joining date
);

CREATE TABLE salary_grades (         -- band table used for the non-equijoin demo
  grade    INT PRIMARY KEY,         -- grade number 1..4
  low_sal  INT NOT NULL,            -- lowest salary in the band
  high_sal INT NOT NULL             -- highest salary in the band
);

INSERT INTO departments VALUES       -- four departments; Marketing will have NO employees
  (1, 'HR',        'Delhi'),        -- dept 1: only Sara
  (2, 'IT',        'Bengaluru'),    -- dept 2: the biggest team (4 people)
  (3, 'Finance',   'Mumbai'),       -- dept 3: two people
  (4, 'Marketing', 'Pune');         -- dept 4: EMPTY — perfect for outer-join demos

INSERT INTO employees VALUES         -- eight employees with deliberate quirks
  (101, 'Aarav', 90000,  2, NULL, '2021-01-15'),  -- IT head, no manager, top salary
  (102, 'Diya',  75000,  2, 101,  '2021-03-10'),  -- IT — salary TIES with Kabir (for RANK demos)
  (103, 'Kabir', 75000,  2, 101,  '2022-06-01'),  -- IT — the other half of the tie
  (104, 'Meera', 85000,  3, NULL, '2020-11-20'),  -- Finance head, no manager
  (105, 'Rohan', 55000,  3, 104,  '2023-02-14'),  -- Finance, reports to Meera
  (106, 'Sara',  48000,  1, NULL, '2022-09-05'),  -- HR, smallest real salary
  (107, 'Vijay', 30000, 99, NULL, '2023-07-19'),  -- ORPHAN: dept 99 does not exist (outer joins)
  (108, 'Neha',  NULL,   2, 101,  '2024-01-08');  -- NULL salary — shows how aggregates skip NULLs

INSERT INTO salary_grades VALUES     -- four salary bands for the non-equijoin section
  (1,     0,  40000),               -- grade 1: 0..40000
  (2, 40001,  60000),               -- grade 2: 40001..60000
  (3, 60001,  80000),               -- grade 3: 60001..80000
  (4, 80001, 100000);               -- grade 4: 80001..100000
```

**Line-by-line walkthrough**

- The three `DROP TABLE IF EXISTS` lines let you paste the whole block again and again without errors.
- `employees.dept_id` has **no FOREIGN KEY on purpose**, so Vijay's row (`dept_id = 99`) can exist — he is our "orphan" for outer-join demos.
- `salary` is nullable: Neha has `NULL`, which exposes how each aggregate treats missing values.
- Diya and Kabir share salary `75000` — a perfect tie for `RANK` vs `DENSE_RANK`.
- Marketing has employees? **No** — that gap is what outer joins are for.

📋 **Data snapshot (memorize this!)**

**employees** (8 rows)

| emp_id | emp_name | salary | dept_id | manager_id | hire_date  |
|--------|----------|--------|---------|------------|------------|
| 101 | Aarav | 90000 | 2  | NULL | 2021-01-15 |
| 102 | Diya  | 75000 | 2  | 101  | 2021-03-10 |
| 103 | Kabir | 75000 | 2  | 101  | 2022-06-01 |
| 104 | Meera | 85000 | 3  | NULL | 2020-11-20 |
| 105 | Rohan | 55000 | 3  | 104  | 2023-02-14 |
| 106 | Sara  | 48000 | 1  | NULL | 2022-09-05 |
| 107 | Vijay | 30000 | 99 | NULL | 2023-07-19 |
| 108 | Neha  | NULL  | 2  | 101  | 2024-01-08 |

**departments** (4 rows)

| dept_id | dept_name | location  |
|---------|-----------|-----------|
| 1 | HR        | Delhi     |
| 2 | IT        | Bengaluru |
| 3 | Finance   | Mumbai    |
| 4 | Marketing | Pune      |

**salary_grades** (4 rows)

| grade | low_sal | high_sal |
|-------|---------|----------|
| 1 | 0     | 40000  |
| 2 | 40001 | 60000  |
| 3 | 60001 | 80000  |
| 4 | 80001 | 100000 |

---

## 1. Aggregate Functions

📌 An **aggregate function** eats *many rows* and returns **one value**. The famous five: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

### 🔹 How each one handles NULL (exam favourite!)

| Function | Meets NULL | Our data (salaries incl. one NULL) |
|---|---|---|
| `COUNT(*)` | Counts **rows**, ignores contents | **8** |
| `COUNT(col)` | Skips NULLs | `COUNT(salary)` → **7** |
| `SUM(col)` | Skips NULLs (all-NULL → NULL, not 0!) | **458000** |
| `AVG(col)` | SUM of non-NULL ÷ COUNT of non-NULL | 458000/7 → **65428.57** |
| `MIN` / `MAX` | Skips NULLs | **30000 / 90000** |
| `COUNT(DISTINCT col)` | Dedupes first, then skips NULLs | `COUNT(DISTINCT dept_id)` → **4** (1,2,3,99) |

### 🔹 The five in one shot

```sql
SELECT COUNT(*)                AS total_rows,     -- counts EVERY row, even Neha's NULL-salary row
       COUNT(salary)           AS salaried,       -- counts only NON-NULL salaries → 7
       COUNT(DISTINCT dept_id) AS distinct_depts, -- duplicate dept numbers counted once → 4
       SUM(salary)             AS total_salary,   -- NULLs are skipped: 458000, NOT NULL
       AVG(salary)             AS avg_salary,     -- 458000 / 7 = 65428.57 (divided by 7, not 8!)
       MIN(salary)             AS lowest,         -- smallest NON-NULL salary
       MAX(salary)             AS highest         -- largest NON-NULL salary
FROM employees;                                  -- no GROUP BY → whole table is ONE group → 1 row
```

**Line-by-line walkthrough**

- `COUNT(*)` vs `COUNT(salary)` differ by exactly one row — Neha's. This 1-row gap is a classic 2-mark question.
- `AVG` divides by the **number of non-NULL values (7)**, never by 8 — that's why `AVG ≠ SUM/COUNT(*)` when NULLs exist.
- Without `GROUP BY`, aggregates collapse the entire table into a **single result row**.
- ✅ Expected output (1 row):

| total_rows | salaried | distinct_depts | total_salary | avg_salary       | lowest | highest |
|------------|----------|----------------|--------------|------------------|--------|---------|
| 8 | 7 | 4 | 458000 | 65428.571428571429 | 30000 | 90000 |

### 🔹 GROUP BY — one row per group

```sql
SELECT dept_id,                    -- the grouping column: one output row per distinct dept_id
       COUNT(*)      AS heads,     -- rows in this group (includes Neha's NULL-salary row)
       COUNT(salary) AS paid,      -- only NON-NULL salaries inside this group
       SUM(salary)   AS payroll,   -- group total (NULL contributions silently skipped)
       AVG(salary)   AS avg_sal,   -- payroll ÷ paid  (NOT payroll ÷ heads)
       MAX(salary)   AS top_sal    -- biggest salary inside the group
FROM employees                     -- source rows
GROUP BY dept_id                   -- collapse rows sharing the same dept_id
ORDER BY dept_id;                  -- tidy display order
```

**Line-by-line walkthrough**

- Every column in the SELECT must be either **in GROUP BY** or **inside an aggregate** — PostgreSQL enforces this with an error otherwise.
- Dept 2 shows the NULL effect perfectly: `heads=4` but `paid=3` (Neha counted as a head, not as a salary).
- Dept 99 appears even though it's not a real department — `GROUP BY` groups *values*, not foreign keys.
- ✅ Expected output:

| dept_id | heads | paid | payroll | avg_sal | top_sal |
|---------|-------|------|---------|---------|---------|
| 1  | 1 | 1 | 48000  | 48000 | 48000 |
| 2  | 4 | 3 | 240000 | 80000 | 90000 |
| 3  | 2 | 2 | 140000 | 70000 | 85000 |
| 99 | 1 | 1 | 30000  | 30000 | 30000 |

### 🔹 HAVING vs WHERE (5-mark favourite)

📌 **WHERE filters rows *before* grouping; HAVING filters groups *after* aggregation.** WHERE cannot see aggregates; HAVING usually only makes sense with aggregates.

```sql
SELECT dept_id,                  -- group identifier shown in the result
       SUM(salary) AS payroll    -- total salary of the rows that SURVIVED the WHERE filter
FROM employees
WHERE salary > 50000             -- ROW filter, runs BEFORE grouping: drops Sara(48000), Vijay(30000)
                                 --   and Neha (NULL > 50000 is unknown → row removed too)
GROUP BY dept_id                 -- groups whatever survived: only depts 2 and 3 remain
HAVING SUM(salary) > 150000      -- GROUP filter, runs AFTER aggregation: dept 3 (140000) fails
ORDER BY dept_id;                -- display order of surviving groups
```

**Line-by-line walkthrough**

- Execution order: **FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY**. Memorize this pipeline!
- Neha's row dies at WHERE because any comparison with NULL yields *unknown*, and unknown is treated as false.
- Dept 3's group *exists* after WHERE (140000) but is *deleted* by HAVING — a group filter, not a row filter.
- ✅ Expected output:

| dept_id | payroll |
|---------|---------|
| 2 | 240000 |

### 🔹 Displaying NULLs safely — COALESCE and IS NULL

```sql
SELECT emp_name,                         -- employee name for context
       salary,                           -- raw column: Neha's row shows <NULL>
       COALESCE(salary, 0) AS safe_sal   -- swap NULL for 0 in display/arithmetic
FROM employees
WHERE salary IS NULL;                    -- IS NULL works; salary = NULL NEVER matches anything!
```

**Line-by-line walkthrough**

- `COALESCE(a, b, c, …)` returns the **first non-NULL** argument — the standard NULL catcher.
- `WHERE salary = NULL` is a classic trap: it compares against *unknown* and returns zero rows. Always write `IS NULL` / `IS NOT NULL`.
- ✅ Expected output:

| emp_name | salary | safe_sal |
|----------|--------|----------|
| Neha | NULL | 0 |

### 🎯 Questions

**Q1 (2 marks): Define aggregate functions with examples.**
✅ Functions that operate on a set of rows and return a single summary value: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. In PostgreSQL, `SUM`/`AVG`/`MIN`/`MAX` ignore NULLs; `COUNT(*)` counts rows.

**Q2 (5 marks): Differentiate WHERE and HAVING.**
✅

| Point | WHERE | HAVING |
|---|---|---|
| Filters | Individual **rows** | Whole **groups** |
| Runs | Before GROUP BY | After GROUP BY |
| Aggregates allowed? | ❌ No (`SUM()` illegal) | ✅ Yes |
| Needs GROUP BY? | No | Usually yes |
| NULL rows | Removed (unknown → false) | Aggregate skips NULLs inside groups |

Example: `WHERE salary > 50000` drops Neha's row *before* grouping; `HAVING SUM(salary) > 150000` then deletes small *groups*.

**Q3 (lab/predict output):** With our employees table, what do `COUNT(*)` and `COUNT(salary)` return, and why do they differ?
✅ 8 and 7. `COUNT(*)` counts rows regardless of content; `COUNT(salary)` skips the one NULL salary (Neha).

**Q4 (2 marks): Why can `AVG(salary)` differ from `SUM(salary)/COUNT(*)`?**
✅ Because AVG ignores NULLs (divides by non-NULL count = 7), while `SUM/COUNT(*)` divides by all rows (8) → 57250 vs 65428.57.

---

## 2. SQL Joins

📌 A **join** combines columns from two tables, row by row, using a condition. The big question for every join type: **which rows survive?**

### 2.1 INNER JOIN — only matching pairs survive

```sql
SELECT e.emp_name,             -- employee name (left table alias e)
       e.dept_id,              -- join key on the employee side
       d.dept_name             -- matching department name (right table alias d)
FROM employees e               -- left input table, nicknamed e
INNER JOIN departments d       -- right input table, nicknamed d
        ON e.dept_id = d.dept_id;  -- keep ONLY pairs whose keys are equal
```

**Line-by-line walkthrough**

- `INNER JOIN … ON` keeps a row **only if both sides find a partner**.
- Vijay (dept 99) finds no department → dropped. Marketing finds no employee → dropped. 8 employees − 1 orphan = **7 rows**.
- Aliases `e` and `d` are mandatory-in-spirit: they disambiguate the two `dept_id` columns.
- ASCII — who survives:

```text
 employees (8)              departments (4)
 ─────────────              ───────────────
 101,102,103,108 ── dept 2 ──► IT            ✔ kept
 104,105         ── dept 3 ──► Finance       ✔ kept      → 7 rows
 106             ── dept 1 ──► HR            ✔ kept
 107 (dept 99)   ✘ no partner
                               4 Marketing ✘ no partner (dropped)
```

- ✅ Expected output (7 rows):

| emp_name | dept_id | dept_name |
|----------|---------|-----------|
| Aarav | 2 | IT |
| Diya  | 2 | IT |
| Kabir | 2 | IT |
| Meera | 3 | Finance |
| Rohan | 3 | Finance |
| Sara  | 1 | HR |
| Neha  | 2 | IT |

### 2.2 LEFT OUTER JOIN — the left table wins

```sql
SELECT e.emp_name,             -- employee name (preserved side)
       d.dept_name,            -- department name (NULL when no match)
       d.location              -- department city (also NULL for the orphan)
FROM employees e               -- LEFT table: EVERY its row survives
LEFT JOIN departments d        -- OUTER keyword is optional: LEFT = LEFT OUTER
        ON e.dept_id = d.dept_id;  -- equal keys join; misses are padded with NULL
```

**Line-by-line walkthrough**

- Every employee survives — matched ones get real department values, the orphan Vijay gets **NULLs** on the right side.
- Marketing still never appears: LEFT JOIN protects the *left* table only.
- ASCII:

```text
 ✔ 101..106,108 → their departments      ✔ kept with real values
 ✔ 107 Vijay    → (NULL, NULL)           ← orphan survives, padded with NULL
 ✘ Marketing                                never listed
```

- ✅ Expected output (8 rows — same 7 as above **plus**):

| emp_name | dept_name | location |
|----------|-----------|----------|
| Vijay | NULL | NULL |

### 2.3 RIGHT OUTER JOIN — the right table wins

```sql
SELECT e.emp_name,             -- employee name (NULL when a department has none)
       d.dept_id,              -- department number (preserved side)
       d.dept_name             -- department name
FROM employees e               -- left table: unmatched rows now get dropped
RIGHT JOIN departments d       -- RIGHT table (departments): EVERY its row survives
        ON e.dept_id = d.dept_id;  -- match condition, same as before
ORDER BY d.dept_id;            -- order by the preserved side for readable output
```

**Line-by-line walkthrough**

- Mirror image of LEFT: all 4 departments survive; Marketing shows `emp_name = NULL`.
- Vijay (dept 99) is dropped — the *employee* side is no longer protected.
- Same 8 rows as LEFT JOIN here, but the *protected side* flipped.
- ✅ Expected output (8 rows — the 7 matches **plus**):

| emp_name | dept_id | dept_name |
|----------|---------|-----------|
| NULL | 4 | Marketing |

### 2.4 FULL OUTER JOIN — nobody is left behind

```sql
SELECT e.emp_name,             -- employee name (NULL for employee-less departments)
       d.dept_id,              -- department number (NULL for the orphan employee)
       d.dept_name,            -- department name (NULL for the orphan)
       e.salary                -- salary (NULL on the Marketing row)
FROM employees e
FULL OUTER JOIN departments d              -- BOTH sides keep their unmatched rows
        ON e.dept_id = d.dept_id;          -- equality condition, unchanged
ORDER BY d.dept_id NULLS LAST;             -- push the NULL-department orphan row to the end
```

**Line-by-line walkthrough**

- FULL = LEFT ∪ RIGHT behaviour: matched pairs (7) + orphan employee Vijay (1) + empty department Marketing (1) = **9 rows**.
- `NULLS LAST` — in PostgreSQL, DESC sorts place NULLs **first** by default; this clause fixes the display.
- ASCII:

```text
 ✔ 7 matched pairs (real values both sides)
 ✔ 107 Vijay        → dept_id NULL, dept_name NULL
 ✔ 4 Marketing      → emp_name NULL, salary NULL     → 9 rows total
```

- ✅ Expected output (9 rows = table 2.1 **plus**):

| emp_name | dept_id | dept_name | salary |
|----------|---------|-----------|--------|
| Vijay | NULL | NULL | 30000 |
| NULL  | 4 | Marketing | NULL |

### 2.5 CROSS JOIN — every combination (cartesian product)

```sql
SELECT e.emp_name,             -- employee from the left input
       d.dept_name             -- department from the right input
FROM employees e
CROSS JOIN departments d;      -- NO ON clause: every left row × every right row
```

**Line-by-line walkthrough**

- No join condition at all: **8 × 4 = 32 rows**. This is relational algebra's **× (cartesian product)**.
- Legit use: generating grids (every employee × every month for payroll planning). In real queries an accidental CROSS JOIN (forgetting ON in old comma-style joins) is a classic bug and a performance bomb on big tables.
- ✅ Expected output: **32 rows**; the first three look like:

| emp_name | dept_name |
|----------|-----------|
| Aarav | HR |
| Aarav | IT |
| Aarav | Finance |
| … | … (29 more) |

### 2.6 SELF JOIN — a table joined to itself

```sql
SELECT e.emp_name AS employee,   -- first copy of the table: the SUBORDINATE
       m.emp_name AS manager     -- second copy of the SAME table: the BOSS
FROM employees e                 -- alias e acts like a virtual copy
INNER JOIN employees m           -- alias m acts like a second virtual copy
        ON e.manager_id = m.emp_id;  -- subordinate's manager_id must equal boss's emp_id
```

**Line-by-line walkthrough**

- One physical table, two logical copies via aliases — the join condition does the rest.
- Aarav, Meera, Sara, Vijay have `manager_id = NULL` → `NULL = anything` is unknown → they drop out (INNER behaviour).
- Use a `LEFT JOIN` instead and those four employees appear with `manager = NULL` (8 rows).
- ✅ Expected output (4 rows):

| employee | manager |
|----------|---------|
| Diya  | Aarav |
| Kabir | Aarav |
| Rohan | Meera |
| Neha  | Aarav |

### 2.7 NATURAL JOIN — implicit join on same-named columns

```sql
SELECT *                    -- NATURAL JOIN automatically hides the duplicated join column
FROM employees
NATURAL JOIN departments;   -- PostgreSQL matches ALL same-named columns (here only dept_id)
```

**Line-by-line walkthrough**

- PostgreSQL inspects both tables, finds the common column(s) — here only `dept_id` — and writes `ON e.dept_id = d.dept_id` for you; the column appears **once**.
- Survivors = same as the INNER JOIN (**7 rows**), minus the duplicate dept_id column.
- ⚠️ Danger: if both tables later grow a column like `created_date`, NATURAL JOIN silently joins on it too. In production, prefer explicit `ON`.

### 2.8 Non-equijoin — joining on a RANGE instead of `=`

```sql
SELECT e.emp_name,       -- employee name
       e.salary,         -- employee salary (the value we band)
       g.grade           -- which band the salary falls into
FROM employees e
JOIN salary_grades g     -- there is NO equality key between these tables!
  ON e.salary BETWEEN g.low_sal AND g.high_sal;  -- a RANGE condition replaces '='
```

**Line-by-line walkthrough**

- Any join whose condition is *not* equality (BETWEEN, `<`, `>`, `<=`, `>=`) is a **theta / non-equi join**; BETWEEN is the exam classic.
- 90000 lands in `[80001, 100000]` → grade 4. NULL salary (Neha) matches no band → dropped, so 8 − 1 = **7 rows**.
- ✅ Expected output (7 rows):

| emp_name | salary | grade |
|----------|--------|-------|
| Aarav | 90000 | 4 |
| Meera | 85000 | 4 |
| Diya  | 75000 | 3 |
| Kabir | 75000 | 3 |
| Rohan | 55000 | 2 |
| Sara  | 48000 | 2 |
| Vijay | 30000 | 1 |

### 2.9 Join of 3 tables

```sql
SELECT e.emp_name,    -- from table 1
       d.dept_name,   -- from table 2 (reached via dept_id)
       g.grade        -- from table 3 (reached via salary band)
FROM employees e
JOIN departments   d ON e.dept_id = d.dept_id                        -- hop 1: employee → department
JOIN salary_grades g ON e.salary BETWEEN g.low_sal AND g.high_sal    -- hop 2: salary → grade
ORDER BY g.grade, e.emp_name;                                        -- neat final ordering
```

**Line-by-line walkthrough**

- Chains are built hop by hop; each INNER JOIN must find a match or the row dies.
- Vijay dies at hop 1 (dept 99 unknown), Neha dies at hop 2 (NULL salary matches no band) → 8 − 2 = **6 rows**.
- ASCII:

```text
employees ⋈ departments ⋈ salary_grades
  hop 1: dept_id = dept_id                → drops Vijay
  hop 2: salary BETWEEN low AND high      → drops Neha
  survivors: 6 rows
```

- ✅ Expected output (6 rows):

| emp_name | dept_name | grade |
|----------|-----------|-------|
| Sara  | HR      | 2 |
| Rohan | Finance | 2 |
| Diya  | IT      | 3 |
| Kabir | IT      | 3 |
| Aarav | IT      | 4 |
| Meera | Finance | 4 |

### 🔹 Join scoreboard on our data

| Join type | Rows returned | Who survives |
|---|---|---|
| INNER | 7 | matched pairs only |
| LEFT  | 8 | all employees (+NULLs for Vijay) |
| RIGHT | 8 | all departments (+NULL employee for Marketing) |
| FULL  | 9 | everything from both sides |
| CROSS | 32 | every combination |
| SELF (inner) | 4 | only employees WITH managers |
| NATURAL | 7 | same as INNER here |
| Non-equijoin | 7 | NULL salary can't match a band |
| 3-table inner | 6 | must match at every hop |

### 🎯 Questions

**Q1 (2 marks): Difference between INNER and LEFT OUTER JOIN?**
✅ INNER keeps only rows with a match on both sides; LEFT OUTER keeps **all rows of the left table**, padding unmatched ones with NULLs on the right. Here: 7 vs 8 rows (Vijay survives only in LEFT).

**Q2 (5 marks): Explain the three OUTER joins with examples.**
✅ LEFT keeps all left rows; RIGHT keeps all right rows (mirror); FULL keeps both sides' unmatched rows. With our data: LEFT → 8 (Vijay padded), RIGHT → 8 (Marketing padded), FULL → 9 (both padded).

**Q3 (lab/predict):** How many rows does `employees CROSS JOIN departments` return, and why?
✅ 8 × 4 = 32 — CROSS JOIN has no condition, so every employee pairs with every department.

**Q4 (2 marks): Why is NATURAL JOIN considered risky?**
✅ It joins on *all* same-named columns implicitly; adding an unrelated common column (e.g. `created_date`) silently changes the result. Explicit `ON` is safer.

---

## 3. Set Operators

📌 Set operators combine the results of two queries **vertically** (stack rows), unlike joins which combine **horizontally** (add columns). The four: `UNION`, `UNION ALL`, `INTERSECT`, `EXCEPT`. Oracle calls EXCEPT `MINUS`.

### 🔹 Rules (exam checklist)

- Both queries must return the **same number of columns**, in **compatible types** (position matters, names don't).
- Result column names come from the **first** query.
- `UNION`, `INTERSECT`, `EXCEPT` remove duplicates (a sort/hash is implied); `UNION ALL` keeps everything.
- NULLs are treated as **equal** duplicates by set operators (two NULLs collapse in UNION).
- `ORDER BY` allowed **only once, at the very end** of the whole expression.

### 🔹 The workhorse example — A and B

```sql
SELECT emp_name, salary       -- result set A
FROM employees
WHERE salary > 70000;         -- A = Aarav 90000, Diya 75000, Kabir 75000, Meera 85000 (4 rows)
```

```sql
SELECT emp_name, salary       -- result set B
FROM employees
WHERE dept_id = 2;            -- B = Aarav 90000, Diya 75000, Kabir 75000, Neha NULL (4 rows)
```

```text
A: {Aarav 90000, Diya 75000, Kabir 75000, Meera 85000}
B: {Aarav 90000, Diya 75000, Kabir 75000, Neha NULL}
```

### 🔹 UNION vs UNION ALL — watch the row counts!

```sql
SELECT emp_name, salary       -- set A's column list (names/types win)
FROM employees
WHERE salary > 70000          -- A: 4 rows

UNION                         -- A ∪ B: merges AND removes duplicate rows
SELECT emp_name, salary       -- same column count/types as A (mandatory!)
FROM employees
WHERE dept_id = 2;            -- B: 4 rows  → UNION output = 5 distinct rows
```

**Line-by-line walkthrough**

- A ∪ B contains: Aarav, Diya, Kabir, Meera, Neha — the trio {Aarav, Diya, Kabir} appeared in **both** sets, so their duplicates vanish.
- Deduplication needs sorting/hashing → slightly slower than UNION ALL.
- ✅ Expected output — **5 rows**:

| emp_name | salary |
|----------|--------|
| Aarav | 90000 |
| Diya  | 75000 |
| Kabir | 75000 |
| Meera | 85000 |
| Neha  | NULL |

```sql
SELECT emp_name, salary       -- set A again
FROM employees
WHERE salary > 70000          -- 4 rows

UNION ALL                     -- A ∪ B: plain concatenation, NO dedup, NO sort
SELECT emp_name, salary       -- identical column shape
FROM employees
WHERE dept_id = 2;            -- 4 rows  → output = 4 + 4 = 8 rows
```

**Line-by-line walkthrough**

- UNION ALL just glues B under A: every duplicate from the overlap stays.
- Fastest of all set operators — nothing is compared. Default choice when you *know* sets can't overlap (e.g., different years).
- ✅ Expected output — **8 rows** (A's 4 rows, then B's 4 rows, duplicates intact).

### 🔹 INTERSECT — rows present in BOTH

```sql
SELECT emp_name, salary       -- set A
FROM employees
WHERE salary > 70000          -- 4 rows

INTERSECT                     -- keep only rows that also exist in B (duplicates removed)
SELECT emp_name, salary       -- same shape
FROM employees
WHERE dept_id = 2;            -- B: 4 rows
```

**Line-by-line walkthrough**

- Common to both: Aarav, Diya, Kabir (Meera is not IT; Neha is not > 70000).
- ✅ Expected output — **3 rows**: Aarav 90000, Diya 75000, Kabir 75000.

### 🔹 EXCEPT — rows in the first but NOT in the second

```sql
SELECT emp_name, salary       -- set A
FROM employees
WHERE salary > 70000          -- 4 rows

EXCEPT                        -- A − B: subtract every row that also appears in B
SELECT emp_name, salary       -- same shape
FROM employees
WHERE dept_id = 2;            -- B: 4 rows
```

**Line-by-line walkthrough**

- `A EXCEPT B` = set difference A − B (order matters: `B EXCEPT A` would give Neha!).
- Of A's four, only Meera is missing from B.
- Oracle spellings: `MINUS` (older) / `EXCEPT` (12c+). PostgreSQL supports **EXCEPT** only.
- ✅ Expected output — **1 row**:

| emp_name | salary |
|----------|--------|
| Meera | 85000 |

### 🎯 Questions

**Q1 (2 marks): What does EXCEPT return? What is Oracle's name for it?**
✅ Rows present in the first result but absent from the second (set difference). Oracle calls it `MINUS`.

**Q2 (5 marks): UNION vs UNION ALL — with our data, compare outputs.**
✅ A (salary > 70000, 4 rows) and B (dept 2, 4 rows) overlap in {Aarav, Diya, Kabir}. `UNION` dedupes → **5 rows**; `UNION ALL` concatenates → **8 rows**. UNION does extra sort/hash work; UNION ALL is cheaper but keeps duplicates.

**Q3 (lab/predict):** How many rows does `(SELECT emp_name FROM employees WHERE dept_id = 2) EXCEPT (SELECT emp_name FROM employees WHERE salary > 70000)` return?
✅ 1 — B's {Aarav, Diya, Kabir, Neha} minus A's {Aarav, Diya, Kabir} leaves **Neha** (a NULL salary row can survive set operators because NULLs are compared as equal values, not unknowns).

**Q4 (2 marks): State two rules set operators impose on their input queries.**
✅ Same number of columns; corresponding columns must have compatible types (and ORDER BY only at the very end).

---

## 4. Views

📌 A **view** is a named, stored query — a **virtual table**. No data is stored (except for materialized views); every access re-runs the defining query.

### 🔹 Why views?

- 🔒 **Security** — expose only safe columns/rows (hide `salary`, hide dept 99) without granting base-table access.
- 🧹 **Simplicity** — save a complicated 3-table join once, reuse its name forever.
- 🧱 **Stability** — applications survive schema changes if a view keeps the old shape.
- 🔁 **Consistency** — one definition of "IT staff" used by everyone.

### 4.1 CREATE VIEW + use it

```sql
CREATE VIEW it_staff AS            -- store the QUERY TEXT under a name; no data copied
SELECT emp_id, emp_name, salary, dept_id   -- we keep dept_id only to demo inserts below
FROM employees
WHERE dept_id = 2;                 -- the invisible filter baked into the view

SELECT * FROM it_staff;            -- query it exactly like a table: 4 rows
```

**Line-by-line walkthrough**

- `CREATE VIEW name AS <query>` — the query runs **fresh on every SELECT** from the view.
- A security-conscious view would drop `salary` and `dept_id`; we kept them for the demos below.
- ✅ Expected output:

| emp_id | emp_name | salary | dept_id |
|--------|----------|--------|---------|
| 101 | Aarav | 90000 | 2 |
| 102 | Diya  | 75000 | 2 |
| 103 | Kabir | 75000 | 2 |
| 108 | Neha  | NULL  | 2 |

### 4.2 Updatable views

📌 PostgreSQL rules — a view is **automatically updatable** (INSERT/UPDATE/DELETE pass through to the base table) only if it is:

- built from **one** table (or updatable view),
- no `DISTINCT`, `GROUP BY`, `HAVING`, `LIMIT`/`OFFSET`,
- no set operations (`UNION`…), aggregates, or window functions,
- contains all required base columns (PK / NOT NULL without default) for INSERT.

```sql
BEGIN;                                                  -- sandbox: everything below is undoable
INSERT INTO it_staff (emp_id, emp_name, salary, dept_id)  -- INSERT against a simple view…
VALUES (109, 'Tara', 70000, 2);                         -- …lands in the BASE table employees
SELECT emp_name FROM it_staff;                          -- Tara is now visible: 5 rows
ROLLBACK;                                               -- undo the demo, employees back to 8 rows
```

**Line-by-line walkthrough**

- The INSERT is transparently rewritten against `employees` — the view is just a window on it.
- Wrapping demos in `BEGIN … ROLLBACK` keeps the shared sample data pristine (and teaches transactions!).
- `UPDATE it_staff SET salary = salary + 1000 WHERE emp_id = 102;` works the same way.

### 4.3 WITH CHECK OPTION — make the view police its own WHERE

```sql
BEGIN;                                       -- sandbox again
CREATE VIEW it_checked WITH CHECK OPTION AS  -- writes THROUGH this view must satisfy its WHERE
SELECT emp_id, emp_name, salary, dept_id
FROM employees
WHERE dept_id = 2;
INSERT INTO it_checked VALUES (110, 'Om', 50000, 1);  -- dept 1 → violates WHERE dept_id = 2
ROLLBACK;                                    -- undo (the INSERT above errors out)
```

**Line-by-line walkthrough**

- The INSERT fails with: `ERROR: new row violates check option for view "it_checked"`.
- **Without** CHECK OPTION the row would be inserted (into dept 1!) and then *silently vanish* from the view — a classic "disappearing row" bug that CHECK OPTION prevents.

### 4.4 Materialized views — store the result on disk

```sql
CREATE MATERIALIZED VIEW dept_payroll AS   -- unlike a view, the RESULT is physically stored
SELECT dept_id,
       COUNT(*)    AS heads,               -- employees per department
       SUM(salary) AS payroll              -- total salary per department
FROM employees
GROUP BY dept_id;                          -- pre-computed once at creation time

SELECT * FROM dept_payroll;                -- instant: reads 4 stored rows, no aggregation now
REFRESH MATERIALIZED VIEW dept_payroll;    -- recompute from base data after it changes
```

**Line-by-line walkthrough**

- Normal view = query text only; materialized view = **stored snapshot** → reads are fast, but data goes stale.
- `REFRESH MATERIALIZED VIEW` rebuilds it; add `CONCURRENTLY` (needs a unique index on the MV) to refresh without blocking readers.
- Ideal for expensive aggregations read often and tolerated slightly stale (dashboards, reports).
- ✅ Expected output:

| dept_id | heads | payroll |
|---------|-------|---------|
| 1  | 1 | 48000  |
| 2  | 4 | 240000 |
| 3  | 2 | 140000 |
| 99 | 1 | 30000  |

### 4.5 ALTER VIEW and DROP VIEW

```sql
ALTER VIEW it_staff RENAME TO it_team;          -- rename only; the definition stays the same
DROP VIEW IF EXISTS it_team;                    -- delete the view; base table employees untouched
DROP MATERIALIZED VIEW IF EXISTS dept_payroll;  -- delete the stored snapshot too
```

**Line-by-line walkthrough**

- `ALTER VIEW … RENAME TO` changes the name; use `CREATE OR REPLACE VIEW` to swap a definition (column list must stay compatible).
- Dropping a view never deletes data — only the stored query/snapshot disappears.
- `IF EXISTS` avoids an error when the object is already gone (handy in re-runnable scripts).

### 🎯 Questions

**Q1 (2 marks): What is a view? Why is it called a virtual table?**
✅ A named stored query that can be used like a table. It stores no data — rows are produced by re-running its defining query at access time.

**Q2 (5 marks): List the rules for an updatable view.**
✅ Single base table; no DISTINCT/GROUP BY/HAVING/LIMIT; no set operators, aggregates or window functions; must expose all NOT-NULL-without-default and key columns for INSERT; then PostgreSQL passes DML through to the base table automatically.

**Q3 (5 marks): View vs materialized view.**
✅ View: virtual, always current, zero storage, slower on complex queries. Materialized view: stores the result (fast reads, extra disk), must be refreshed (`REFRESH MATERIALIZED VIEW`), can go stale. Pick MVs for costly aggregations with tolerant freshness.

**Q4 (lab/predict):** What happens if you `INSERT INTO it_checked VALUES (110, 'Om', 50000, 1)` (view has `WITH CHECK OPTION`, WHERE dept_id = 2)?
✅ ERROR: new row violates check option for view "it_checked" — the insert is rejected because the new row wouldn't be visible through the view.

---

## 5. Subqueries

📌 A **subquery** is a query nested inside another. Classification cheat-line:

- **Single-row** (returns 1×1) → used with `=`, `>`, `<` …
- **Multi-row** (returns a column of values) → used with `IN`, `ANY`, `ALL`
- **Correlated** (references the outer query) → runs once *per outer row*
- **Scalar** (1×1 used as a value) → usable in SELECT list too
- **Derived table** (nested in FROM) → acts as a temporary table

### 5.1 Single-row subquery with `=`, `>`

```sql
SELECT emp_name, salary                 -- employees to report
FROM employees
WHERE salary > (SELECT AVG(salary)      -- inner query first: ONE value, 65428.57
                FROM employees);        -- single cell → safe with the > operator
```

**Line-by-line walkthrough**

- PostgreSQL evaluates the inner query first (≈65428.57), then compares each row against it.
- If a "single-row" subquery returns 0 rows, the comparison becomes NULL; if it returns >1 row, you get `ERROR: more than one row returned by a subquery`.
- ✅ Expected output — 4 rows: Aarav 90000, Diya 75000, Kabir 75000, Meera 85000.

### 5.2 Multi-row subquery with IN

```sql
SELECT emp_name                          -- employees to list
FROM employees
WHERE dept_id IN (SELECT dept_id         -- may return MANY values — perfect for IN
                  FROM departments
                  WHERE location = 'Mumbai');  -- Mumbai departments → {3}
```

**Line-by-line walkthrough**

- `IN` asks "is my value anywhere in this list?" — safe for 0, 1 or many inner rows.
- `NOT IN` has a NULL trap: if the inner list contains NULL, `NOT IN` returns nothing (unknown). Prefer `NOT EXISTS`.
- ✅ Expected output — 2 rows: **Meera, Rohan** (dept 3 = Mumbai).

### 5.3 ANY and ALL

```sql
SELECT emp_name, salary                    -- employees beating a threshold
FROM employees
WHERE salary > ANY (SELECT salary          -- > ANY: true if BIGGER THAN the smallest value
                    FROM employees
                    WHERE dept_id = 2);    -- IT salaries: {90000, 75000, 75000, NULL}
```

**Line-by-line walkthrough**

- `> ANY(list)` = "bigger than **at least one**" = bigger than the list **minimum** (75000) → Aarav, Meera pass.
- NULLs inside the list don't produce matches — comparisons with NULL are unknown, and ANY needs at least one TRUE.

```sql
SELECT emp_name, salary                    -- employees beating a higher bar
FROM employees
WHERE salary > ALL (SELECT salary          -- > ALL: true only if BIGGER THAN the largest value
                    FROM employees
                    WHERE dept_id = 1);    -- HR salaries: {48000}
```

**Line-by-line walkthrough**

- `> ALL(list)` = bigger than **every** value = bigger than the list **maximum** (48000).
- ✅ 2nd query output — 5 rows: Aarav, Diya, Kabir, Meera, Rohan. (Sara = 48000 is not *greater than* 48000; Neha's NULL fails automatically.)

### 5.4 Correlated subquery — the inner query reads the outer row

```sql
SELECT e.emp_name, e.dept_id, e.salary     -- outer query: one candidate at a time
FROM employees e
WHERE e.salary > (SELECT AVG(e2.salary)    -- re-executed for EVERY outer row e
                  FROM employees e2
                  WHERE e2.dept_id = e.dept_id);  -- ← correlation: uses outer's dept_id
```

**Line-by-line walkthrough**

- The inner query is not independent — it filters by the **current outer row's** department, giving that department's own average.
- Dept 2 avg = 80000 → only Aarav (90000) passes; dept 3 avg = 70000 → only Meera (85000) passes.
- Correlated subqueries can be slow (N executions) — the optimizer often rewrites them as joins internally.
- ✅ Expected output — 2 rows: **Aarav (2, 90000), Meera (3, 85000)**.

### 5.5 EXISTS / NOT EXISTS — ask "does at least one row exist?"

```sql
SELECT d.dept_name                        -- departments to test
FROM departments d
WHERE EXISTS (SELECT 1                    -- SELECT list is irrelevant; 1 is convention
              FROM employees e
              WHERE e.dept_id = d.dept_id);  -- correlated: any employee of THIS department?
```

**Line-by-line walkthrough**

- `EXISTS` returns TRUE if the inner query yields **≥ 1 row** — it doesn't read them all, it stops at the first (efficient).
- `SELECT 1` is idiomatic: EXISTS cares about existence, not values.
- ✅ Expected output — 3 rows: **HR, IT, Finance** (Marketing is empty).

```sql
SELECT d.dept_name                        -- departments to test
FROM departments d
WHERE NOT EXISTS (SELECT 1                -- TRUE when the inner query finds NOTHING
                  FROM employees e
                  WHERE e.dept_id = d.dept_id);  -- no employee at all for this department
```

**Line-by-line walkthrough**

- `NOT EXISTS` is the *safe* way to write anti-joins (immune to the `NOT IN` + NULL trap).
- ✅ Expected output — 1 row: **Marketing**.

### 5.6 Subquery in FROM — derived tables

```sql
SELECT t.dept_id, t.avg_sal                     -- read from the temporary table t
FROM (SELECT dept_id, AVG(salary) AS avg_sal    -- inner query = an on-the-fly table
      FROM employees
      GROUP BY dept_id) AS t                    -- derived tables MUST get an alias in PostgreSQL
WHERE t.avg_sal > 60000;                        -- filter the ALREADY aggregated result
```

**Line-by-line walkthrough**

- The inner query runs first and its result behaves like a real table named `t` for the duration of the query.
- The alias is **mandatory** in PostgreSQL (`AS t`) — forgetting it is the #1 beginner error here.
- Lets you aggregate twice: first per department, then filter on the aggregate.
- ✅ Expected output:

| dept_id | avg_sal |
|---------|---------|
| 2 | 80000 |
| 3 | 70000 |

### 5.7 Scalar subquery in the SELECT list

```sql
SELECT emp_name,                                        -- employee
       salary,                                          -- their salary
       (SELECT AVG(salary) FROM employees) AS company_avg,  -- one value repeated on every row
       salary - (SELECT AVG(salary) FROM employees) AS diff -- compare each row to that value
FROM employees
ORDER BY emp_id
LIMIT 3;                                                -- show just 3 rows to keep it short
```

**Line-by-line walkthrough**

- A scalar subquery must return **exactly one value** (0 rows → NULL, >1 rows → error).
- PostgreSQL may optimize repeated identical scalar subqueries into a single computation.
- ✅ Expected output:

| emp_name | salary | company_avg | diff |
|----------|--------|-------------|------|
| Aarav | 90000 | 65428.571428571429 | 24571.428571428571 |
| Diya  | 75000 | 65428.571428571429 | 9571.428571428571 |
| Kabir | 75000 | 65428.571428571429 | 9571.428571428571 |

### 5.8 Subqueries inside INSERT / UPDATE / DELETE

```sql
BEGIN;                                                -- sandbox: roll back all DML at the end
CREATE TABLE employees_backup (LIKE employees INCLUDING ALL);  -- empty structural clone
INSERT INTO employees_backup                          -- target of the INSERT…SELECT
SELECT * FROM employees;                              -- subquery feeds all 8 rows into the copy

UPDATE employees                                      -- rows to change
SET salary = salary * 1.10                            -- 10% raise
WHERE dept_id = (SELECT dept_id                       -- resolve 'IT' to its number via subquery
                 FROM departments
                 WHERE dept_name = 'IT');             -- → 2, so 4 rows are updated

DELETE FROM employees                                 -- rows to remove
WHERE salary < (SELECT AVG(salary)                    -- average re-computed AFTER the raise
                FROM employees);                      -- ≈68857.14 → removes Rohan, Sara, Vijay
ROLLBACK;                                             -- undo everything; sample data stays intact
```

**Line-by-line walkthrough**

- `INSERT … SELECT` copies the *result of a query* into a table — no VALUES list needed.
- The UPDATE uses a single-row subquery to translate a name ('IT') into an id — no hard-coded 2.
- The DELETE compares against a **fresh** average (post-raise), so its threshold ≈ 68857.14, deleting 3 rows.
- `BEGIN … ROLLBACK` makes destructive DML safe to practise; use `COMMIT` in real work.

### 🎯 Questions

**Q1 (2 marks): Correlated vs non-correlated subquery?**
✅ Non-correlated: self-contained, evaluated **once**, result reused. Correlated: references outer-query columns, conceptually re-evaluated **once per outer row** (e.g. per-department average above).

**Q2 (2 marks): When do you use IN vs ANY vs ALL?**
✅ `IN` = membership in a list; `= ANY` ≡ IN; `> ANY` = greater than at least one (min); `> ALL` = greater than every (max). `<` and `<>` follow the same pattern mirrored.

**Q3 (5 marks): Why prefer NOT EXISTS over NOT IN?**
✅ If the inner list contains a NULL, `x NOT IN (…, NULL)` can never be TRUE (comparison with NULL is unknown) → the query returns zero rows. `NOT EXISTS` tests row *existence* with a correlated condition and is immune to NULLs (and usually faster with an index).

**Q4 (lab/predict):** Write a query for employees earning more than the average of **their own** department.
✅ Correlated subquery (see 5.4): `WHERE e.salary > (SELECT AVG(e2.salary) FROM employees e2 WHERE e2.dept_id = e.dept_id)` → Aarav, Meera.

---

## 6. Relational Algebra

📌 Relational algebra is the **mathematical core** of SQL: a small set of operators that take relations in and produce relations out. Exams ask you to translate English → algebra (and back).

### 🔹 Notation cheat table

| Symbol | Operator | Meaning | SQL equivalent |
|--------|----------|---------|----------------|
| σ (sigma) | Selection | pick **rows** matching a condition | `WHERE` |
| π (pi) | Projection | pick **columns** (removes duplicates) | `SELECT col1, col2` |
| ∪ | Union | rows in either (schema-compatible) | `UNION` |
| ∩ | Intersection | rows in both | `INTERSECT` |
| − | Difference | rows in R but not S | `EXCEPT` |
| × | Cartesian product | every pairing of rows | `CROSS JOIN` |
| ⋈ | Natural join | × then match same-named columns | `NATURAL JOIN` |
| ⋈_θ | Theta join | × then apply condition θ (equi-join if θ is `=`) | `JOIN … ON` |
| ρ (rho) | Rename | rename a relation or its attributes | alias `AS` |
| ÷ | Division | "the ones related to **ALL** of…" | no direct SQL (NOT EXISTS ×2) |

Operator order matters: innermost/lowest applies first. `σ … (π …)` etc.

### 🔹 Worked examples on OUR schema

**Q1: "Names of employees who work in IT."**

```
π emp_name ( σ dept_name = 'IT' ( employees ⋈ departments ) )
```

- Step 1: `employees ⋈ departments` → 7 matched rows (Vijay drops).
- Step 2: `σ dept_name='IT'` → 4 rows (Aarav, Diya, Kabir, Neha).
- Step 3: `π emp_name` → {Aarav, Diya, Kabir, Neha}.
- SQL: `SELECT e.emp_name FROM employees e JOIN departments d ON e.dept_id = d.dept_id WHERE d.dept_name = 'IT';`

**Q2: "Names of Finance employees earning more than 60000."**

```
π emp_name ( σ dept_name = 'Finance' ∧ salary > 60000 ( employees ⋈ departments ) )
```

- Both conditions in one σ (conjunction, written with ∧).
- Finance rows: Meera 85000, Rohan 55000 → only Meera > 60000 → **{Meera}**.

**Q3: "Departments located in Delhi or Mumbai."**

```
π dept_name ( σ location = 'Delhi' ∨ location = 'Mumbai' ( departments ) )
```

- Disjunction with ∨ → **{HR, Finance}**.

**Q4: "Rename trick" — employees and their managers (self join in algebra).**

```
π e.emp_name, m.emp_name ( ρ e (employees) ⋈_{e.manager_id = m.emp_id} ρ m (employees) )
```

- ρ gives the two copies different names (e, m) so they can be joined — exactly SQL aliases.

### 🔹 Division ÷ — "for ALL of them…"

📌 Division answers queries containing **"all"** / **"every"**: *students enrolled in ALL courses*, *employees who worked on ALL projects*.

```
works_on(emp, proj)        Projects        works_on ÷ Projects
───────────────────        ────────        ───────────────────
Ravi  P1                   P1              Ravi
Ravi  P2                   P2              (only Ravi appears in EVERY
Ravi  P3                   P3               project pair)
Priya P1
Priya P2
Arjun P1
```

- Algorithm: for each candidate employee, check {P1,P2,P3} ⊆ their project set. Ravi ⊇ all → answer {Ravi}.
- SQL equivalent (double NOT EXISTS):

```sql
SELECT DISTINCT w1.emp              -- candidate employees
FROM works_on w1
WHERE NOT EXISTS (                  -- there is NO project…
      SELECT 1 FROM projects p
      WHERE NOT EXISTS (            -- …that this employee has NOT worked on
            SELECT 1 FROM works_on w2
            WHERE w2.emp = w1.emp   -- same employee as outer
              AND w2.proj = p.proj)); -- on that specific project
```

**Line-by-line walkthrough**

- Reads as: "employees for whom no missing project exists" — a double negation that means *all*.
- Innermost matches employee↔project; middle finds missing projects; outer keeps employees with none missing.

### 🔹 Why algebra matters (bridge to optimization)

- σ and π **commute/associate** nicely → the optimizer *pushes selections and projections down* to shrink rows/columns before joins (see §10 heuristics).
- Every SQL query is internally translated into an algebra tree; optimizing = rewriting that tree into a cheaper equivalent.

### 🎯 Questions

**Q1 (2 marks): Difference between selection (σ) and projection (π)?**
✅ σ filters **rows** by a predicate (→ WHERE); π filters **columns** and removes duplicate tuples (→ SELECT col list).

**Q2 (2 marks): Natural join vs theta join?**
✅ Theta join = cartesian product filtered by an arbitrary condition ⋈_θ; natural join = the special case matching **all same-named attributes** with equality and merging those columns into one.

**Q3 (5 marks): Translate to algebra: "Names of employees in dept 2 with salary above 70000".**
✅ π emp_name ( σ dept_id = 2 ∧ salary > 70000 ( employees ) ) → {Aarav, Diya, Kabir}. No join needed since dept_id lives in employees.

**Q4 (5 marks): When is division used? Give the pattern.**
✅ For "all/every" questions. If R(A,B) and S(B), then R ÷ S = { a | (a, b) ∈ R for every b ∈ S }. SQL has no ÷ operator — emulate with double NOT EXISTS or GROUP BY + COUNT = (SELECT COUNT(*) FROM S).

---

## 7. Window Functions

📌 A **window function** computes an aggregate-like value **without collapsing rows** — every row stays, plus a calculated value "looking across a window" of rows. Grammar: `function(...) OVER (PARTITION BY … ORDER BY …)`.

### 7.1 ROW_NUMBER vs RANK vs DENSE_RANK (the trio)

```sql
SELECT emp_name,
       salary,                                                        -- NULL sorts LAST by request
       ROW_NUMBER() OVER (ORDER BY salary DESC NULLS LAST) AS rn,     -- 1,2,3,4… unique even on ties
       RANK()       OVER (ORDER BY salary DESC NULLS LAST) AS rk,     -- ties share rank, then a GAP
       DENSE_RANK() OVER (ORDER BY salary DESC NULLS LAST) AS drk     -- ties share rank, NO gap
FROM employees;
```

**Line-by-line walkthrough**

- `OVER (ORDER BY …)` defines the ranking window — the whole table here, sorted by salary.
- `NULLS LAST` matters: PostgreSQL treats NULL as *largest*, so plain `DESC` would put Neha **first**.
- Diya & Kabir tie at 75000: they get the same RANK/DENSE_RANK (3), but ROW_NUMBER must stay unique (3, 4).
- ✅ Expected output (8 rows) — **the comparison table with actual values**:

| emp_name | salary | ROW_NUMBER | RANK | DENSE_RANK |
|----------|--------|------------|------|------------|
| Aarav | 90000 | 1 | 1 | 1 |
| Meera | 85000 | 2 | 2 | 2 |
| Diya  | 75000 | 3 | 3 | 3 |
| Kabir | 75000 | 4 | **3** | **3** |
| Rohan | 55000 | 5 | **5** (gap!) | **4** (no gap) |
| Sara  | 48000 | 6 | 6 | 5 |
| Vijay | 30000 | 7 | 7 | 6 |
| Neha  | NULL  | 8 | 8 | 7 |

### 7.2 PARTITION BY — restart the calculation per group

```sql
SELECT emp_name,
       dept_id,          -- the partitioning column (shown for clarity)
       salary,
       RANK() OVER (PARTITION BY dept_id              -- ranking RESETS for each department
                    ORDER BY salary DESC NULLS LAST) AS dept_rank
FROM employees
ORDER BY dept_id, dept_rank;                           -- readable final display
```

**Line-by-line walkthrough**

- `PARTITION BY` = "GROUP BY for window functions": each partition is ranked independently.
- Compare with §1: GROUP BY would collapse dept 2 to one row; here all 4 IT rows remain with ranks 1, 2, 2, 4.
- Neha's rank in dept 2 is **4**, not 3 — RANK counts *rows* before her (1, 2, 2 → next is 4).
- ✅ Expected output:

| emp_name | dept_id | salary | dept_rank |
|----------|---------|--------|-----------|
| Sara  | 1  | 48000 | 1 |
| Aarav | 2  | 90000 | 1 |
| Diya  | 2  | 75000 | 2 |
| Kabir | 2  | 75000 | 2 |
| Neha  | 2  | NULL  | 4 |
| Meera | 3  | 85000 | 1 |
| Rohan | 3  | 55000 | 2 |
| Vijay | 99 | 30000 | 1 |

### 7.3 LAG, LEAD, running totals, NTILE

```sql
SELECT emp_name,
       salary,                              -- the values the window functions inspect
       LAG(salary)  OVER w AS prev_sal,     -- previous row's salary (NULL at the start)
       LEAD(salary) OVER w AS next_sal,     -- next row's salary (NULL at the end)
       SUM(salary)  OVER w AS running,      -- cumulative total up to the current row
       NTILE(4)     OVER w AS quartile      -- spread 8 rows into 4 equal buckets
FROM employees
WINDOW w AS (ORDER BY emp_id);              -- named window: define once, reuse in 4 functions
ORDER BY emp_id;                            -- display in the same order as the window
```

**Line-by-line walkthrough**

- `WINDOW w AS (…)` names the window so LAG/LEAD/SUM/NTILE share one definition — cleaner and less error-prone.
- `LAG/LEAD` default to looking 1 row away; a third argument supplies a fallback value (e.g. `LAG(salary, 1, 0)`).
- The running `SUM … OVER (ORDER BY …)` accumulates in emp_id order; Neha's NULL adds nothing, so the total stays 458000.
- NTILE(4) on 8 rows → buckets of exactly 2.
- ✅ Expected output (8 rows):

| emp_name | salary | prev_sal | next_sal | running | quartile |
|----------|--------|----------|----------|---------|----------|
| Aarav | 90000 | NULL  | 75000 | 90000  | 1 |
| Diya  | 75000 | 90000 | 75000 | 165000 | 1 |
| Kabir | 75000 | 75000 | 85000 | 240000 | 2 |
| Meera | 85000 | 75000 | 55000 | 325000 | 2 |
| Rohan | 55000 | 85000 | 48000 | 380000 | 3 |
| Sara  | 48000 | 55000 | 30000 | 428000 | 3 |
| Vijay | 30000 | 48000 | NULL  | 458000 | 4 |
| Neha  | NULL  | 30000 | NULL  | 458000 | 4 |

### 🔹 Trio comparison summary

| Property | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|
| Ties | broken arbitrarily (unique) | same rank | same rank |
| After a 2-way tie at rank 3 | 4 | **5** (skips 4) | **4** (no skip) |
| Our data | 1..8 all distinct | 1,2,3,3,5,6,7,8 | 1,2,3,3,4,5,6,7 |
| Typical use | pagination, "pick one" | competition ranking | percentile levels |

### 🎯 Questions

**Q1 (2 marks): How is a window function different from GROUP BY?**
✅ GROUP BY collapses each group into one row; a window function keeps **every row** and appends a value computed over its window (partition).

**Q2 (5 marks): RANK vs DENSE_RANK vs ROW_NUMBER with example.**
✅ On ties (75000, 75000): RANK gives 3,3 then **jumps to 5** (counts preceding rows); DENSE_RANK gives 3,3 then **4** (consecutive levels); ROW_NUMBER forces unique 3,4. Full table in §7.1.

**Q3 (2 marks): What do PARTITION BY and ORDER BY inside OVER do?**
✅ PARTITION BY splits rows into independent groups (calculations restart per partition); ORDER inside OVER defines the sequence used for ranking / LAG-LEAD / cumulative frames.

**Q4 (lab):** Second-highest salary using a window function?
✅ `SELECT DISTINCT salary FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) dr FROM employees) t WHERE dr = 2;` → 85000.

---

## 8. Hashing

📌 **Hashing** maps a search key to a **bucket address** using a hash function `h(key)` — giving O(1) lookups for equality. Two families: **static** (fixed structure) and **dynamic** (grows with data).

### 🔹 Static hashing

- Buckets = fixed number B; primary page `bucket = h(key) mod B`; each bucket holds 1+ disk pages.
- Lookup: hash the key → go straight to that bucket → search inside. Insert/delete same idea.
- Problem: buckets fill unevenly → **bucket overflow** → **overflow chains** (linked pages) → lookups degrade to chain-walking.

```text
h(k) = k mod 4                        bucket 0: [4, 8, 12] → overflow → [16]
                                      bucket 1: [1, 5, 9]
key 12 → 12 mod 4 = 0 → bucket 0      bucket 2: [2, 10]
key  7 →  7 mod 4 = 3 → bucket 3      bucket 3: [3, 7, 11]
```

- Fixes (still "static"): open addressing (probe next slots) / overflow chaining; or reorganize with a bigger B.
- Weakness: the number of buckets is chosen in advance — wrong guess ⇒ long overflow chains forever.

### 🔹 Dynamic hashing — extendible hashing (exam favourite!)

- Uses the **last d bits** of h(key). Two depths:
  - **Global depth (GD)** = bits used by the **directory** (directory has 2^GD entries);
  - **Local depth (LD)** = bits that actually distinguish one bucket's keys.
- When a bucket overflows: split **only that bucket**, bump its LD; **double the directory** only if LD was equal to GD.

**Worked example** (bucket capacity = 2, keys are ints, h(k) = the key's binary value):

```text
START — global depth 1, directory of 2 entries (last 1 bit decides)

  directory        buckets
  0 ────────────►  A: 4(100), 6(110)     ← both end in bit 0
  1 ────────────►  B: 5(101), 7(111)     ← both end in bit 1

INSERT 8 (1000): last bit 0 → bucket A → A is FULL → SPLIT A

  directory (global depth 2)     buckets                (local depth)
  00 ───────────►  A: 4(0100), 8(1000)                  LD 2
  01 ───────────►  B: 5, 7      ◄── two pointers to B!  LD 1
  10 ───────────►  C: 6(0110)                           LD 2
  11 ───────────►  B: 5, 7                              LD 1
```

- Only bucket A split; 4 (…00) and 8 (…00) stayed in A; 6 (…10) moved to new bucket C; directory doubled 2→4.
- B's local depth is still 1, so entries 01 and 11 both point to it — **no wasteful copy of B**.
- Lookup of 12 (1100): last 2 bits = 00 → bucket A directly.
- Key exam points: directory size = 2^GD; split is local; doubling is O(2^GD) but rare.

### 🔹 Hash indexes in PostgreSQL

```sql
CREATE INDEX emp_name_hash ON employees USING hash (emp_name);  -- build a HASH index on emp_name
SELECT * FROM employees WHERE emp_name = 'Diya';                -- equality → hash index CAN be used
SELECT * FROM employees WHERE emp_name LIKE 'D%';               -- prefix/range → hash index CANNOT help
DROP INDEX emp_name_hash;                                       -- tidy up
```

**Line-by-line walkthrough**

- `USING hash` selects the hash access method instead of the default B-tree.
- Hash indexes support **only `=`** — no `<`, `>`, `BETWEEN`, `ORDER BY`, or `LIKE 'abc%'`.
- Historically not crash-safe (not WAL-logged) before PostgreSQL 10; fine since v10, but the planner still prefers B-tree almost always. A hash index can win when the key is very long and only equality is ever used.
- Related-but-different: the executor also uses **hashing** internally for `HASH JOIN` and `HASH AGGREGATE` plans (§10).

### 🔹 Use cases and limits

- ✅ Great: exact-match lookups (`id = ?`, `key = ?`), equality-only workloads.
- ❌ Bad: range scans, sorting, prefix search, MIN/MAX — hashes destroy ordering information.

### 🎯 Questions

**Q1 (2 marks): What is bucket overflow and how is it handled?**
✅ When a bucket's pages are full, extra keys go to **overflow pages** chained to the bucket (or open addressing probes nearby slots). Chains slow lookups; cure = reorganization or dynamic hashing.

**Q2 (5 marks): Static vs dynamic (extendible) hashing.**
✅ Static: fixed bucket count, `h(k) mod B`, simple but suffers overflow chains as data grows. Extendible: directory of 2^GD entries using the last GD bits; buckets split locally (LD+1) and only the directory doubles when LD = GD — grows gracefully, no long chains.

**Q3 (2 marks): Can a hash index speed up `WHERE salary BETWEEN 50000 AND 70000`? Why?**
✅ No. Hashing scatters values with no ordering, so only equality probes work; B-tree handles ranges.

**Q4 (2 marks): What do global depth and local depth mean?**
✅ Global depth = number of low-order bits the **directory** indexes (2^GD entries). Local depth = bits distinguishing a specific **bucket**; a split raises LD, and the directory doubles only if LD becomes > GD.

---

## 9. Indexing

📌 An **index** is an auxiliary data structure (extra table-ish file) mapping key values → row locations, so the engine can **seek** instead of scanning every row. Cost: extra disk + slower INSERT/UPDATE/DELETE (every index must be maintained).

### 🔹 Clustered vs non-clustered

| Aspect | Clustered | Non-clustered |
|---|---|---|
| Table rows physically ordered by | the index key | separate structure; heap order unchanged |
| Count per table | at most **one** | many |
| Row lookup | direct (data IS the leaf) | index leaf → pointer → heap fetch |
| In PostgreSQL | ❌ no clustered indexes (tables are heaps); `CLUSTER tbl USING idx` re-sorts **once** | ✅ every `CREATE INDEX` |
| In MySQL InnoDB | ✅ the PRIMARY KEY is clustered | secondary indexes |

### 🔹 B-tree — the default in PostgreSQL

- Balanced search tree; all leaves at the same depth; leaves linked → great for `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'abc%'`.

```text
                 [ 50 | 90 ]                 ← root: routes each search
                /     |      \
        [10|30]     [50|70]    [90|95]       ← internal pages: keys only
        /  |  \     /   |  \    /   \
  leaves hold (key → row pointer) pairs, chained left→right  ← range scans walk this chain
```

### 🔹 Creating the index zoo

```sql
CREATE INDEX idx_emp_salary ON employees (salary);            -- plain B-tree (USING btree is default)
CREATE UNIQUE INDEX uq_emp_name ON employees (emp_name);      -- rejects duplicate names + fast lookup
CREATE INDEX idx_emp_dept_sal ON employees (dept_id, salary); -- multi-column: dept equality, then salary range
CREATE INDEX idx_emp_high_sal ON employees (salary) WHERE salary > 50000;  -- PARTIAL: only big salaries indexed
CREATE INDEX idx_emp_name_lc ON employees (LOWER(emp_name));  -- expression index for case-insensitive search
DROP INDEX idx_emp_high_sal;                                  -- remove one you no longer need
```

**Line-by-line walkthrough**

- **Unique index** doubles as a constraint — inserting a second 'Diya' would fail.
- **Multi-column** order matters: `(dept_id, salary)` serves `WHERE dept_id = 2 AND salary > 60000`; it does *not* efficiently serve salary-only conditions.
- **Partial index** is smaller and faster for a known hot subset (here: salaries above 50000) — matches queries that include the same WHERE.
- **Expression index** lets `WHERE LOWER(emp_name) = 'diya'` use an index.

### 🔹 When NOT to index

- 🔹 Tiny tables — a Seq Scan is cheaper than index hops (our 8-row demo tables!).
- 🔹 Low-selectivity columns (e.g. a 2-value flag) — index filters out almost nothing.
- 🔹 Write-heavy tables — each INSERT/UPDATE maintains every index.
- 🔹 Columns never used in WHERE/JOIN/ORDER BY — pure overhead.

### 🔹 Checking usage

```sql
ANALYZE employees;                          -- refresh planner statistics (row counts, histograms)
EXPLAIN SELECT * FROM employees WHERE salary = 75000;          -- plan only, no execution
SELECT indexrelname, idx_scan               -- how many times each index has been used
FROM pg_stat_user_indexes
WHERE relname = 'employees';                -- for this table only
```

**Line-by-line walkthrough**

- On 8 rows the planner will rightly pick a **Seq Scan** — indexes pay off at real data volumes.
- `idx_scan = 0` for a long time ⇒ the index is dead weight; consider dropping it.
- ✅ Small-table plan looks like:

```text
Seq Scan on employees  (cost=0.00..1.10 rows=2 width=20)
  Filter: (salary = 75000)
```

### 🎯 Questions

**Q1 (2 marks): What is an index and why do we create one?**
✅ An auxiliary structure mapping key → row location to avoid full scans, turning O(n) searches into O(log n) B-tree seeks — at the cost of extra storage and slower writes.

**Q2 (5 marks): Clustered vs non-clustered index.**
✅ Clustered dictates physical row order (one per table, e.g. InnoDB PK); non-clustered is a separate structure with pointers (many allowed). PostgreSQL has no clustered indexes — tables are heaps and `CLUSTER` reorders once.

**Q3 (2 marks): Why is the default PostgreSQL index a B-tree?**
✅ It keeps keys sorted in a balanced tree: equality AND range queries, MIN/MAX, ORDER BY and prefix LIKE all work, with O(log n) guaranteed depth.

**Q4 (lab):** How would you verify whether an index is actually being used?
✅ `EXPLAIN` the typical query (look for `Index Scan using <name>`), and check `pg_stat_user_indexes.idx_scan` over time; `ANALYZE` first so the planner has fresh statistics.

---

## 10. Query Optimization

📌 You write *what* you want; the planner decides *how*. Same SQL can have many execution plans — the optimizer picks the **cheapest** one.

### 🔹 The 5 steps of query processing

1. **Parse** — syntax check, SQL text → parse tree.
2. **Analyze/Semantic check** — tables/columns exist? types valid? → query tree.
3. **Rewrite** — expand views, apply rules, simplify expressions.
4. **Plan / Optimize** — enumerate candidate plans, estimate each **cost**, pick the minimum.
5. **Execute** — the executor walks the chosen plan tree and returns rows.

### 🔹 Two measures: cost and cardinality

- **Cost** — a unitless estimate (`cost=0.00..1.10` = startup..total). Defaults: sequential page read = 1.0, random page = 4.0, per-row CPU = 0.01 — tune `postgresql.conf` for your hardware.
- **Cardinality (rows=…)** — estimated number of rows at each plan node, derived from statistics (via `ANALYZE`) and **selectivity** (e.g. `salary = 75000` on N rows → 1/N if unique). Wrong cardinality ⇒ wrong plan; hence `ANALYZE` after big data changes.

### 🔹 EXPLAIN vs EXPLAIN ANALYZE

```sql
ANALYZE employees;                                                   -- feed the planner fresh statistics
EXPLAIN SELECT emp_name FROM employees WHERE salary > 70000;         -- show the plan WITHOUT running it
EXPLAIN ANALYZE SELECT emp_name FROM employees WHERE salary > 70000; -- RUN it and show real timings too
```

**Line-by-line walkthrough**

- `EXPLAIN` = zero risk (planner only); `EXPLAIN ANALYZE` = actually executes — **never** bare-run it on INSERT/UPDATE/DELETE; wrap those in `BEGIN … ROLLBACK`.
- Compare `rows=` (estimate) with `actual rows=` (truth): a big gap means stale statistics.
- ✅ On our small table:

```text
Seq Scan on employees  (cost=0.00..1.10 rows=4 width=38) (actual time=0.012..0.014 rows=4 loops=1)
  Filter: (salary > 70000)
  Rows Removed by Filter: 4
Planning Time: 0.08 ms
Execution Time: 0.03 ms
```

- On a large indexed table the same query would instead show `Index Scan using idx_emp_salary … Index Cond: (salary > 70000)` — comparing these two plans is exactly how you justify an index.

### 🔹 Join strategies the planner chooses between

| Strategy | How | Best when |
|---|---|---|
| Nested Loop | for each outer row, probe inner | small outer, inner indexed |
| Hash Join | build hash table on smaller input, probe with other | big unsorted inputs, equality |
| Merge Join | sort both, zip through in order | both large & pre-sorted/indexed |

### 🔹 Heuristic (rule-based) optimizations

```sql
SELECT e.emp_name, d.dept_name                     -- ✂ project early: only 2 columns flow upward
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'IT'                           -- 🔻 selection pushed down BEFORE the join
  AND e.salary > 50000;                            -- 🔻 second filter also applied early
```

**Line-by-line walkthrough**

- **Push selections down** (σ early): filter each table before joining — fewer rows to combine.
- **Project early** (π early): drop unused columns before the join.
- Join **already-reduced** inputs; never let a cartesian product appear accidentally.
- Avoid `SELECT *`, keep statistics fresh, write sargable predicates (`salary > 50000` ✔, `UPPER(salary) > …` blocks index use).

### 🎯 Questions

**Q1 (2 marks): Difference between EXPLAIN and EXPLAIN ANALYZE?**
✅ EXPLAIN only *estimates and displays* the plan; EXPLAIN ANALYZE *executes* the query and shows actual row counts and timings alongside estimates.

**Q2 (2 marks): What are cost and cardinality?**
✅ Cost = planner's unitless estimate of work per plan (startup..total). Cardinality = estimated rows at each node, computed from statistics and selectivity — the driver of plan choice.

**Q3 (5 marks): Explain query processing steps.**
✅ Parse (syntax) → Analyze (names/types) → Rewrite (views) → Plan/Optimize (cost all candidate plans, pick cheapest) → Execute.

**Q4 (5 marks): State heuristic optimization rules.**
✅ Perform selections as early as possible; perform projections early; join reduced inputs (avoid needless cartesian products); decompose conjunctive selections; keep statistics current; write sargable predicates.

---

## 11. Quick Revision Cheat Sheet

| Topic | One-liner to remember |
|---|---|
| Aggregates | All skip NULLs except `COUNT(*)`; `AVG` divides by non-NULL count; SUM of all-NULL = NULL |
| GROUP BY / HAVING | Pipeline: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY; WHERE = rows, HAVING = groups |
| INNER JOIN | Only matched pairs (7/8 here) |
| LEFT / RIGHT / FULL | Keep the named side's orphans, pad with NULL (8 / 8 / 9 here) |
| CROSS JOIN | No ON → rows₁ × rows₂ (32 here) |
| SELF JOIN | Same table twice via aliases; NULL manager drops out of INNER |
| NATURAL JOIN | Implicit ON for same-named columns — convenient, risky |
| Non-equijoin | `ON e.salary BETWEEN g.low AND g.high` — any θ-condition |
| UNION vs UNION ALL | Dedupe (5 rows) vs plain concat (8 rows); same col count + types |
| INTERSECT / EXCEPT | Common rows / minus rows; Oracle: MINUS; ORDER BY only at the very end |
| Views | Virtual table = stored query; updatable if single-table, no DISTINCT/GROUP/aggregates |
| WITH CHECK OPTION | Writes must satisfy the view's WHERE, else ERROR |
| Materialized view | Stores the result; `REFRESH MATERIALIZED VIEW` to update |
| Subqueries | `=` → single row; `IN/ANY/ALL` → multi-row; correlated → per outer row; EXISTS = efficient anti-join tool |
| Relational algebra | σ rows, π columns, ∪ ∩ −, ×, ⋈, ρ rename, ÷ "all of" (double NOT EXISTS) |
| Window functions | `OVER` = aggregate without collapsing; RANK gaps, DENSE_RANK no gaps, ROW_NUMBER unique |
| LAG / LEAD / NTILE | Previous / next row value; bucket-splitter — all need `OVER (ORDER BY …)` |
| Hashing | `h(key)` → bucket; static = overflow chains; extendible = directory bits, split locally |
| PG hash index | Equality only; B-tree preferred; hash join/aggregate reuse the idea |
| Indexing | B-tree default; unique/partial/multi-column variants; skip small/low-selectivity/write-hot tables |
| Optimization | Parse → Analyze → Rewrite → Plan → Execute; cost + cardinality; EXPLAIN (ANALYZE); push σ/π down |

---

## 12. Practice Set

### 🅰 2-mark questions

**Q1. Define aggregate functions. Name any four.**
✅ Functions that take a set of rows and return one summary value: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`. In PostgreSQL they ignore NULL inputs (except `COUNT(*)`, which counts rows).

**Q2. What is a view? Why is it called virtual?**
✅ A named stored query usable like a table. It stores no rows — every SELECT re-executes its defining query — hence "virtual". (Only materialized views store data.)

**Q3. What is the difference between a correlated and a simple subquery?**
✅ Simple: evaluated once, independent of the outer query. Correlated: references outer columns, so it is (conceptually) re-evaluated once per outer row — e.g. comparing each salary to its own department's average.

**Q4. What is bucket overflow in static hashing?**
✅ A condition where a bucket's primary pages are full and further keys must go to overflow (chained) pages, lengthening lookups. Handled by chaining, open addressing, reorganization, or dynamic hashing.

**Q5. When does PostgreSQL treat a view as automatically updatable?**
✅ When the view selects from a single table and uses no DISTINCT, GROUP BY, HAVING, LIMIT/OFFSET, set operations, aggregates, or window functions (and exposes required NOT NULL/key columns for INSERT).

### 🅱 5-mark questions

**Q6. Differentiate WHERE and HAVING with an example.**
✅ WHERE filters rows before grouping (cannot use aggregates); HAVING filters groups after aggregation.
`SELECT dept_id, SUM(salary) FROM employees WHERE salary > 50000 GROUP BY dept_id HAVING SUM(salary) > 150000;`
→ WHERE drops Sara, Vijay and Neha's row; HAVING deletes dept 3's group (140000 ≤ 150000); result: dept 2 → 240000.

**Q7. Compare UNION and UNION ALL with output counts on any example.**
✅ With A = salaries > 70000 (4 rows) and B = dept 2 (4 rows): `UNION` removes duplicates → **5 rows**; `UNION ALL` concatenates → **8 rows**. Rules for both: same column count, compatible types. UNION ALL is faster (no dedup pass).

**Q8. Compare ROW_NUMBER, RANK and DENSE_RANK with actual output.**
✅ For salaries (90000, 85000, 75000, 75000, 55000, …):

| emp_name | salary | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|---|
| Aarav | 90000 | 1 | 1 | 1 |
| Meera | 85000 | 2 | 2 | 2 |
| Diya  | 75000 | 3 | 3 | 3 |
| Kabir | 75000 | 4 | 3 | 3 |
| Rohan | 55000 | 5 | 5 | 4 |

ROW_NUMBER is always unique; RANK skips numbers after ties; DENSE_RANK has no gaps.

**Q9. Explain static hashing vs extendible hashing.**
✅ Static: fixed B buckets, `h(k) mod B`, overflow chains on collision-heavy data, reorganization needed to grow. Extendible: directory of 2^GD entries addressed by the last GD bits; overflowing bucket splits alone (local depth +1); directory doubles only when local depth = global depth. Example: with GD = 1 and buckets {4,6}, {5,7}, inserting 8 splits bucket A into A{4,8} and C{6}, giving a 4-entry directory where 01 and 11 still share bucket B.

**Q10. Clustered vs non-clustered index. How does PostgreSQL differ from MySQL here?**
✅ Clustered: rows physically stored in key order, one per table, leaf = data itself (MySQL InnoDB's PK). Non-clustered: separate structure storing key → row pointer, many allowed. PostgreSQL has **no** clustered indexes (heap tables); `CLUSTER table USING index` re-sorts data once and is not maintained afterwards.

**Q11. Write relational algebra for: (a) names of IT employees, (b) employees in Mumbai earning above 60000.**
✅ (a) π emp_name (σ dept_name='IT' (employees ⋈ departments)) → {Aarav, Diya, Kabir, Neha}
(b) π emp_name (σ location='Mumbai' ∧ salary>60000 (employees ⋈ departments)) → {Meera}

### 🅲 Lab / hands-on (write the query)

**Q12. Find the second-highest salary.**
```sql
SELECT MAX(salary) AS second_highest              -- biggest value BELOW the overall maximum
FROM employees
WHERE salary < (SELECT MAX(salary)                -- inner query: overall max = 90000
                FROM employees);
```
✅ Output: **85000**. (Window alternative: `DENSE_RANK() = 2` — handles NULLs and ties more predictably.)

**Q13. List departments that have no employees.**
```sql
SELECT d.dept_name                                -- department to test
FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e       -- TRUE if no matching employee exists
                  WHERE e.dept_id = d.dept_id);   -- correlated on the outer department
```
✅ Output: **Marketing**. (NOT IN would also work here, but breaks if the inner list contains NULL.)

**Q14. Show the top earner of every department.**
```sql
SELECT emp_name, dept_id, salary                  -- final report columns
FROM (SELECT emp_name, dept_id, salary,           -- inner query ranks salaries per department
             RANK() OVER (PARTITION BY dept_id
                          ORDER BY salary DESC NULLS LAST) AS r
      FROM employees) AS t                        -- derived table must get an alias
WHERE r = 1;                                      -- keep only each department's rank-1 row
```
✅ Output (4 rows): Aarav (2, 90000), Meera (3, 85000), Sara (1, 48000), Vijay (99, 30000).

**Q15. Show each employee with a running total of their department's payroll (in joining order).**
```sql
SELECT emp_name, dept_id, salary,                             -- per-employee facts
       SUM(salary) OVER (PARTITION BY dept_id                 -- restart total per department
                         ORDER BY hire_date                   -- accumulate in joining order
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS dept_running
FROM employees
ORDER BY dept_id, hire_date;                                  -- tidy display
```
✅ Dept 2 by hire order: Aarav 90000 → Diya 165000 → Kabir 240000 → Neha 240000 (NULL adds nothing).

**Q16. Find the 3rd-highest *distinct* salary using a window function.**
```sql
SELECT DISTINCT salary                            -- one row per salary level
FROM (SELECT salary,
             DENSE_RANK() OVER (ORDER BY salary DESC NULLS LAST) AS dr  -- levels 1,2,2,3,4…
      FROM employees) AS t
WHERE dr = 3;                                     -- third distinct level
```
✅ Output: **75000**. (DENSE_RANK gives distinct salary *levels*, so ties don't distort the count.)

---

🎉 **You made it!** Re-run the Section 0 script any time, paste any block into `psql`, and watch the expected outputs appear. Revise §11 the night before the exam, and attempt §12 under a 30-minute timer.

