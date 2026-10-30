# Unit 1 — Introduction to Database Systems

> 📌 **INT306 — Database Management Systems | Unit 1**
> This unit is the "why, what and how" of databases: why we need a DBMS, what is inside one, how it is organised in layers, how we design data with the ER model, and how the relational model turns that design into tables. Every SQL example here uses **PostgreSQL (psql)** — open a terminal, type `psql`, and you can run every block in order.

## ✅ What you'll learn

- Why file systems fail and what problems a DBMS solves (redundancy, inconsistency, atomicity…)
- The building blocks of a DBMS: hardware, software, data, procedures, users, query processor, storage manager
- Where DBMS is used in real life (banking, railways, e-commerce, university, telecom, healthcare)
- The three-level (ANSI/SPARC) architecture and 1-tier / 2-tier / 3-tier setups
- Logical vs physical data independence
- Schema vs instance (with a live PostgreSQL demo)
- Data models: conceptual → representational → physical
- The ER model: entities, attributes, relationships, cardinality, participation + a full College ER diagram
- The relational model: relations, keys and integrity constraints (created for real in psql)
- Relational vs non-relational (NoSQL) databases

---

## 📑 Table of Contents

1. [Purpose of Database Systems](#1-purpose-of-database-systems)
2. [Components of DBMS](#2-components-of-dbms)
3. [Applications of DBMS](#3-applications-of-dbms)
4. [Three-Tier DBMS Architecture](#4-three-tier-dbms-architecture)
5. [Data Independence](#5-data-independence)
6. [Database Schema vs Instance](#6-database-schema-vs-instance)
7. [Data Modeling](#7-data-modeling)
8. [Entity-Relationship (ER) Model](#8-entity-relationship-er-model)
9. [Relational Model](#9-relational-model)
10. [Relational vs Non-Relational Databases](#10-relational-vs-non-relational-databases)
- [⚡ Quick Revision](#-quick-revision)
- [📝 Practice Set](#-practice-set)

---

## 1. Purpose of Database Systems

📌 **The ladder first** — four words you must keep apart:

| Term | Meaning | Example |
|---|---|---|
| **Data** | Raw facts, unprocessed | `101, Ayush, 88.50` |
| **Information** | Processed, meaningful data | "Ayush scored 88.50 in DBMS" |
| **Database** | Organised collection of related data | All student, course, fee records of the college |
| **DBMS** | Software that stores, protects and serves that data | **PostgreSQL**, MySQL, Oracle, MongoDB |

🔹 **Before DBMS** — the file-processing era: each office of a college kept its own files (`students.txt`, `fees.csv`, `library.txt`) and each program read/wrote them directly. It "worked" — until it didn't. 👇

### 🔹 The 7 classic file-system problems

| # | Problem | What goes wrong in files | How DBMS fixes it |
|---|---|---|---|
| 1 | **Redundancy** | Same data copied into many files (student address in fees *and* library files) → wasted space | One central database; each fact stored **once** |
| 2 | **Inconsistency** | Copies disagree: fee updated in one file, not the other → two "truths" | Single copy + constraints → disagreement impossible |
| 3 | **Data isolation** | Data scattered across files in different formats → hard to combine ("fees of library defaulters") | One schema; SQL joins combine tables easily |
| 4 | **Integrity** | Rules like "marks 0–100" must be coded into every program; one bug breaks the rule | Constraints (`CHECK`, `PRIMARY KEY`) declared **once**, enforced always |
| 5 | **Atomicity** | Power cut mid-operation: fee deducted but receipt not written → half-done update | Transactions are all-or-nothing (`COMMIT`/`ROLLBACK`) |
| 6 | **Concurrent access** | Two clerks book the last hostel seat at the same instant → double booking | Concurrency control (locking/MVCC) gives each user a safe, isolated view |
| 7 | **Security** | Files are all-or-nothing; cannot say "clerk edits fees but never sees marks" | GRANT/REVOKE permissions down to column & row level |

✅ **One-line summary:** a DBMS is a *single source of truth* plus an *enforcement engine* — one copy of every fact, with rules, safety and security built in.

### 🎯 Questions

**Q1. Define DBMS. Give two examples. (2 marks)**

✅ **Answer:** A DBMS (Database Management System) is software that allows users to define, create, store, retrieve and manage data in databases while enforcing integrity, security and concurrent access. Examples: **PostgreSQL** (our practical DBMS), MySQL, Oracle, MongoDB.

**Q2. Explain any five problems of file-processing systems and how a DBMS overcomes them. (5 marks)**

✅ **Answer:** Write any five rows of the table above with 1–2 lines each — e.g. *redundancy* (duplicate data → DB stores each fact once), *inconsistency* (mismatched copies → single copy + constraints), *atomicity* (partial updates on crash → transactions), *concurrent access* (lost updates → locking/MVCC), *security* (no fine-grained access in files → GRANT/REVOKE). Conclude: DBMS = centralised, controlled, self-protected data.

**Q3. Spot-the-problem: "Ramesh paid ₹5000; the clerk updated `fees.csv` but the printed receipt file still shows the old amount." (2 marks)**

✅ **Answer:** This is **data inconsistency**, caused by **redundancy** (the same fact stored in two files). In a DBMS both views read the same stored value, so they can never disagree.

---

## 2. Components of DBMS

A DBMS is not just software — it is a **system of 5 components** working together:

| Component | What it is | College example |
|---|---|---|
| 🔩 **Hardware** | Physical machines: server, disk, RAM, network | The lab server running PostgreSQL on port 5432 |
| 💿 **Software** | The DBMS itself + OS + tools/applications | PostgreSQL engine, `psql` client, pgAdmin |
| 📊 **Data** | Actual data + **metadata** (data about data) | Student rows + the definition "roll_no is INT PRIMARY KEY" |
| 📜 **Procedures** | Rules/methods for using the system | Backup policy, how to register a new user, restore steps |
| 👥 **Users** | People who interact with the system | DBA, clerks, developers, analysts |

### 🔹 Database users (exam favourite!)

| User type | Who are they? | How they interact | Example |
|---|---|---|---|
| **Naive users** | End users, no SQL knowledge | Ready-made apps, forms, menus | Clerk entering fees in the college portal; ATM user |
| **Application programmers** | Write the apps naive users use | SQL inside Java/Python/Flask programs | Developer coding the portal's backend |
| **Sophisticated users** | Write their own SQL queries | Query tools, `psql`, analytics engines | Analyst running reports; data scientist |
| **DBA (Database Administrator)** | The boss of the database | Admin commands, DDL, security | Defines schema, grants rights, takes backups, tunes performance |

🔹 *(Books also mention **specialized users** — CAD/expert-system developers who write advanced applications.)*

### 🔹 The two "engines" inside a DBMS

```text
-- Simplified structure of a DBMS (Korth, Fig 1.1 style)
   naive users ──▶ apps ◀── application programmers   sophisticated users ──▶ query tools
                                   │
                                   ▼
        ┌────────────────────────────────────────────┐
        │             QUERY PROCESSOR                │
        │  DDL interpreter   DML compiler            │
        │  query optimizer   evaluation engine       │
        └──────────────────────┬─────────────────────┘
                               │
        ┌──────────────────────┴─────────────────────┐
        │             STORAGE MANAGER                │
        │  authorization & integrity manager         │
        │  transaction manager                       │
        │  file manager        buffer manager        │
        └──────────────────────┬─────────────────────┘
                               ▼
   💾 Disk: data files | data dictionary | indexes | statistics
```

**Query processor** — translates your SQL into efficient disk operations:
- **DDL interpreter** — understands `CREATE TABLE` etc., updates the data dictionary
- **DML compiler** — turns `SELECT/INSERT/UPDATE/DELETE` into low-level instructions
- **Query optimizer** — picks the cheapest way (index scan? sequential scan?) to run a query
- **Evaluation engine** — actually executes the chosen plan

**Storage manager** — the bridge between low-level data and the queries above:
- **Authorization & integrity manager** — checks user rights + constraint violations
- **Transaction manager** — makes operations atomic and safe (commit/rollback)
- **File manager** — manages disk space and file structures
- **Buffer manager** — keeps hot data pages in RAM so disk reads are minimised

🔹 On disk the storage manager maintains: **data files**, the **data dictionary** (metadata — "database about the database"), **indexes**, and **statistics** used by the optimizer.

🔹 **PostgreSQL mapping (nice to mention):** parser/analyser + planner/executor ≈ query processor; buffer manager + WAL + background writer ≈ storage manager; `\d table` in psql reads the data dictionary (`pg_catalog`, `information_schema`).

### 🎯 Questions

**Q1. Name the four main categories of DBMS users with one example each. (2 marks)**

✅ **Answer:** Naive users (clerk using a fee form), application programmers (write the form's backend), sophisticated users (analyst writing SQL reports), DBA (defines schema, manages backups and security).

**Q2. Which component of the query processor chooses between an index scan and a full table scan? (2 marks)**

✅ **Answer:** The **query optimizer** — it estimates the cost of each possible plan and picks the cheapest one; the evaluation engine merely executes it.

**Q3. What is the data dictionary? Who updates it? (2 marks)**

✅ **Answer:** The data dictionary (system catalog) is metadata — tables that describe the database's own structure (tables, columns, types, constraints, users). It is updated by the DBMS automatically whenever DDL runs; DBAs and users only *read* it (in PostgreSQL: `information_schema`, `pg_catalog`).

---

## 3. Applications of DBMS

DBMS is everywhere — anywhere many users share data that must be correct, safe and always available:

| Sector | What is stored | Why DBMS is essential there |
|---|---|---|
| 🏦 **Banking** | Accounts, transactions, loans, cards | Millions of concurrent ATM/UPI ops need **ACID transactions** |
| 🚆 **Railways / Airlines** | Trains, flights, seats, bookings | Two users booking seat #42 at the same instant → concurrency control |
| 🛒 **E-commerce** | Products, carts, orders, payments, reviews | Catalog joins, price integrity, 24×7 availability |
| 🎓 **University** | Students, courses, marks, fees, attendance | Registration spikes, transcripts, role-based security |
| 📱 **Telecom** | Call records, plans, bills, towers | Billions of CDRs/day; billing accuracy |
| 🏥 **Healthcare** | Patients, prescriptions, lab reports | Privacy rules, drug-interaction checks, audit trails |
| 🎬 **Social media / streaming** | Users, posts, likes, watch history | Massive scale, personalization queries |

🔹 **Common thread:** many concurrent users + data that must never be wrong + fine-grained security → exactly the 7 problems from Section 1, solved by a DBMS.

### 🎯 Questions

**Q1. List any four applications of DBMS with one use in each. (2 marks)**

✅ **Answer:** Banking — transaction processing for ATM/UPI; Railways — seat reservation with concurrency control; University — student/course/marks management; Healthcare — privacy-controlled patient records. *(E-commerce, telecom also valid.)*

**Q2. Why is a plain file system dangerous for banking? Give two reasons. (5 marks)**

✅ **Answer:** (1) **No atomicity** — a crash between "debit" and "credit" loses money; a DBMS guarantees all-or-nothing transactions. (2) **No concurrent-access control** — two ATMs updating the same balance simultaneously produce a lost update; a DBMS serialises/conflicts-resolves them. *(Also acceptable: no integrity checks on balance ≥ 0, no fine-grained authorization, no audit/recovery.)*

---

## 4. Three-Tier DBMS Architecture

### 🔹 A. The three levels of data abstraction (ANSI/SPARC)

The same data can be seen three ways — from "what the user needs" down to "bits on disk":

```text
-- ANSI/SPARC three-level architecture
        ┌─────────────────────────────────────────────────┐
        │                EXTERNAL LEVEL                   │
        │  View A (teacher): roll_no, name, marks         │
        │  View B (library): roll_no, name                │
        └───────────────────────┬─────────────────────────┘
                                │ ◀ external/conceptual mapping
        ┌───────────────────────┴─────────────────────────┐
        │               CONCEPTUAL LEVEL                  │
        │  students(roll_no, name, dob, marks, fees)      │
        │  + all tables, relationships, constraints       │
        └───────────────────────┬─────────────────────────┘
                                │ ◀ conceptual/internal mapping
        ┌───────────────────────┴─────────────────────────┐
        │                INTERNAL LEVEL                   │
        │  rows in heap file, B-tree index on roll_no,    │
        │  8 kB pages, WAL files, compression...          │
        └─────────────────────────────────────────────────┘
```

| Level | Also called | Describes | Who works here |
|---|---|---|---|
| **External** | View level | *Part* of the database one user group sees (a VIEW) | Naive/sophisticated users |
| **Conceptual** | Logical level | *What* data is stored + relationships + constraints (whole community view) | Designers, DBA |
| **Internal** | Physical level | *How* data is stored: files, indexes, compression, paths | DBA, DBMS itself |

🔹 **Key idea — data abstraction:** each level hides the details of the level below it, so users think in tables while the DBMS worries about pages and indexes.

### 🔹 B. 1-tier / 2-tier / 3-tier (deployment architecture)

```text
-- 1-TIER: everything on one machine (learning / personal use)
┌────────────────────────────────────────────┐
│   psql client + PostgreSQL server together │
│   (same laptop, e.g. your lab machine)     │
└────────────────────────────────────────────┘

-- 2-TIER: client talks straight to the DB server
┌────────────────────┐  SQL over TCP:5432  ┌────────────────────┐
│  Client            │ ◀─────────────────▶ │  PostgreSQL server │
│  (psql / Java app) │   query ▶ / ◀ rows  │  (stores + processes)│
└────────────────────┘                     └────────────────────┘

-- 3-TIER: browser → application server → database server
┌─────────┐  HTTP/HTTPS  ┌───────────────────┐  SQL:5432  ┌────────────┐
│ Browser │ ◀──────────▶ │ App server        │ ◀────────▶ │ PostgreSQL │
│  (UI)   │              │ (Flask/Django/Node │            │ DB server  │
└─────────┘              │  = business logic) │            └────────────┘
                         └───────────────────┘
```

| Feature | 1-tier | 2-tier | 3-tier |
|---|---|---|---|
| Layers | 1 (all-in-one) | Client + DB | Browser + App server + DB |
| Business logic lives in | — | Client (fat client) | App server (thin client) |
| Users | Single user | Small LAN/department | Thousands over the internet |
| Security | Local | DB password on every client | DB hidden behind app server ✅ |
| Example | `psql` on your laptop | `psql` connecting to lab server | College result website (Flask + PostgreSQL) |

🔹 **Note:** in 2-tier each client holds a DB connection/password; in 3-tier the app server holds the one DB connection and clients only speak HTTP — that is why websites are 3-tier.

### 🎯 Questions

**Q1. Name the three levels of DBMS architecture and state what each describes. (2 marks)**

✅ **Answer:** External (view) level — what each user group sees; Conceptual (logical) level — what data is stored and how it relates, with constraints; Internal (physical) level — how data is actually stored on disk (files, indexes).

**Q2. Distinguish 2-tier from 3-tier architecture. (5 marks)**

✅ **Answer:** 2-tier = client application directly opens a database connection and sends SQL (e.g. `psql` → PostgreSQL); logic sits in the client; suits small LANs; DB credentials spread across clients. 3-tier = an application server sits between browser and DB; clients speak HTTP, only the app server speaks SQL; logic centralised; scalable and secure; the standard for web apps. Draw both ASCII diagrams from above for full marks.

**Q3. The DBA changes how rows are physically hashed into pages. Which level changed, and which views are affected? (2 marks)**

✅ **Answer:** Only the **internal level** changed; external views and the conceptual schema are untouched — thanks to **physical data independence** (next section!).

---

## 5. Data Independence

📌 **Definition:** Data independence is the ability to change the schema at **one level** of the architecture without changing the schema at the **level above it**.

| Type | Change happens at | Levels above that stay untouched | Triggered by |
|---|---|---|---|
| **Physical data independence** | Internal level (storage) | Conceptual + external | Adding an index, changing file organisation, partitioning |
| **Logical data independence** | Conceptual level (structure) | External views + apps | Adding a table/column, splitting a table, renaming a column (behind a view) |

🔹 **Easiest way to remember:** *physical* = "stored **where/how**?" · *logical* = "stored **what**?"

### ✅ Runnable demo 1 — both independences in psql

```sql
CREATE TABLE demo_students (            -- scratch table used only for this data-independence demo
  id    SERIAL PRIMARY KEY,             -- surrogate PK: sequence auto-numbers 1, 2, 3...
  name  VARCHAR(60) NOT NULL            -- student name; cannot be empty
);                                      -- ends the table definition
INSERT INTO demo_students (name)        -- adds rows; id is auto-filled by the SERIAL sequence
VALUES ('Ayush'),                       -- row 1 gets id 1
       ('Moli'),                        -- row 2 gets id 2
       ('Ravi');                        -- row 3 gets id 3
CREATE INDEX idx_demo_name              -- PHYSICAL-level change: builds a B-tree index on name
ON demo_students (name);                -- rows, schema and queries stay exactly the same
SELECT * FROM demo_students             -- the same app query as before the index existed
WHERE name = 'Moli';                    -- identical result; PostgreSQL may now use the index
```

**Line-by-line walkthrough:**
- `SERIAL` is PostgreSQL's shortcut: it silently creates a sequence and a `DEFAULT nextval(...)` on `id`.
- `CREATE INDEX` is a pure **internal-level** change: `\d demo_students` will list `idx_demo_name` under "Indexes", but no row or column moved.
- The `SELECT` output is unchanged — apps never needed to know the index exists. That is **physical data independence** in action.

**Expected output:**

```text
--  id │ name
-- ────┼──────
--   2 │ Moli
-- (1 row)
```

```sql
ALTER TABLE demo_students               -- LOGICAL-level change: rename a column in the schema
  RENAME COLUMN name TO full_name;      -- old queries that say `name` would now break!
CREATE VIEW demo_students_v AS          -- a VIEW = a saved query = an external-level "window"
  SELECT id, full_name AS name          -- exposes the OLD column name again
  FROM demo_students;                   -- the rename stays hidden inside the view
SELECT name FROM demo_students_v;       -- old-style query still works, completely unchanged
```

**Line-by-line walkthrough:**
- `RENAME COLUMN` changes the **conceptual** schema — this is the painful kind of change for apps.
- The view re-publishes the old shape, so external users never notice — that is **logical data independence**.
- Views are virtual: no data is copied; the view's query runs on demand.

**Expected output:**

```text
--   name
-- ───────
--  Ayush
--  Moli
--  Ravi
-- (3 rows)
```

### 🎯 Questions

**Q1. Define physical and logical data independence. (2 marks)**

✅ **Answer:** **Physical** — ability to modify the internal schema (storage structures, indexes) without affecting the conceptual schema or application programs. **Logical** — ability to modify the conceptual schema (add tables/columns, restructure) without rewriting the external views/applications.

**Q2. The DBA adds a partition and rebuilds an index on `students`. Which independence is demonstrated? (2 marks)**

✅ **Answer:** **Physical data independence** — only the internal level changed; conceptual schema and all user views are unaffected.

**Q3. Why is logical data independence harder to achieve than physical? (2 marks)**

✅ **Answer:** Because logical changes alter *what data means* to applications (names, structure, relationships), so views must be carefully maintained to keep old shapes alive — whereas physical changes are invisible by design.

---

## 6. Database Schema vs Instance

| | **Schema** | **Instance** |
|---|---|---|
| Meaning | The overall **design/structure** of the database | The **actual data** stored at a moment in time |
| Analogy | Variable's **declaration/type** (blueprint) | Variable's **value right now** |
| Changes via | DDL: `CREATE`, `ALTER`, `DROP` | DML: `INSERT`, `UPDATE`, `DELETE` |
| How often | Rare (design-time) | Every millisecond (run-time) |
| Also called | Intension | Extension / database state |
| Stored in | Data dictionary (`information_schema`) | Data files |

### ✅ Runnable demo 2 — watch schema and instance separately

```sql
CREATE DATABASE college;                      -- creates a brand-new empty database called 'college'
```

**Expected output:** `-- CREATE DATABASE`

```text
-- switch your psql session to the new database
\c college
```

```sql
CREATE TABLE students (                       -- SCHEMA statement: defines structure, stores no data
  roll_no INT PRIMARY KEY,                    -- integer column; unique + not null; identifies each row
  name    VARCHAR(50) NOT NULL,               -- text up to 50 chars; cannot be left empty
  marks   NUMERIC(5,2) CHECK (marks BETWEEN 0 AND 100)  -- decimal marks; must lie in 0..100
);                                            -- ends the CREATE TABLE statement
INSERT INTO students (roll_no, name, marks)   -- DML: adds one row (an instance change)
VALUES (101, 'Ayush', 88.50);                 -- the row instance: (101, 'Ayush', 88.50)
INSERT INTO students (roll_no, name, marks)   -- second DML statement
VALUES (102, 'Moli', 91.00);                  -- second row instance
SELECT * FROM students;                       -- DQL: reads the CURRENT instance of the table
```

**Expected output:**

```text
--  roll_no │ name  │ marks
-- ─────────┼───────┼───────
--      101 │ Ayush │ 88.50
--      102 │ Moli  │ 91.00
-- (2 rows)
```

**Line-by-line walkthrough:**
- `CREATE TABLE` = schema event; it appears in the data dictionary and *nothing* is stored except structure.
- `INSERT` = instance event; each call adds a tuple and the "state" of the DB changes.
- `SELECT *` shows the instance — run it twice and you get the same rows *until* someone runs DML.
- `NUMERIC(5,2)` prints trailing zeros: `88.50`, not `88.5`.

```sql
UPDATE students                               -- DML: instance change (values mutate)
SET    marks = 89.75                          -- new value for marks
WHERE  roll_no = 101;                         -- touches ONLY this row (always use WHERE!)
ALTER TABLE students                          -- DDL: SCHEMA change (structure), rows untouched
  ADD COLUMN email VARCHAR(100);              -- new column; every existing row gets NULL here
UPDATE students SET email = 'ayush@college.edu'  -- instance change: fill one email cell
WHERE roll_no = 101;                          -- only row 101 is affected
SELECT roll_no, name, marks, email FROM students;  -- see the final instance
```

**Expected output:**

```text
--  roll_no │ name  │ marks │       email
-- ─────────┼───────┼───────┼────────────────────
--      101 │ Ayush │ 89.75 │ ayush@college.edu
--      102 │ Moli  │ 91.00 │                      ← empty cell = NULL
-- (2 rows)
```

**Line-by-line walkthrough:**
- `UPDATE ... WHERE` filters rows; without `WHERE`, *every* row changes — the classic accident.
- `ALTER TABLE ... ADD COLUMN` proves schema/instance separation: old rows survive and simply show `NULL` in the new column.
- One table, two kinds of change: DML bent the **data**, DDL bent the **shape**.

```text
-- inspect the SCHEMA (structure) of students with a psql meta-command
\d students
```

**Expected output (trimmed):**

```text
--                                     Table "public.students"
--   Column  │         Type          │ Nullable │ Default
-- ──────────┼───────────────────────┼──────────┼─────────
--  roll_no  │ integer               │ not null │
--  name     │ character varying(50) │ not null │
--  marks    │ numeric(5,2)          │          │
--  email    │ character varying(100)│          │
-- Check constraints:
--   "students_marks_check" CHECK (marks >= 0 AND marks <= 100)
-- Indexes:
--   "students_pkey" PRIMARY KEY, btree (roll_no)
```

### 🎯 Questions

**Q1. Differentiate schema and instance with one example each. (2 marks)**

✅ **Answer:** Schema = the structural design, e.g. `students(roll_no INT PRIMARY KEY, name VARCHAR(50), marks NUMERIC(5,2))`. Instance = the data at an instant, e.g. `{(101,'Ayush',88.50), (102,'Moli',91.00)}`. Schema rarely changes; the instance changes with every INSERT/UPDATE/DELETE.

**Q2. Predict the output: after `INSERT INTO students VALUES (103,'Ravi',77);` and `SELECT COUNT(*) FROM students;` (2 marks)**

✅ **Answer:** `3` — the instance now holds three tuples; the schema is unchanged.

**Q3. Which of these change the schema: INSERT, UPDATE, ALTER, SELECT? (2 marks)**

✅ **Answer:** Only **ALTER** (DDL). INSERT/UPDATE change the instance; SELECT changes nothing.

---

## 7. Data Modeling

📌 **Data model** = a set of concepts to describe the structure of the database (data, relationships, constraints). It is the *language* in which we talk about data.

```text
-- Design pipeline: real world → bits (each arrow = one kind of model)
 Real world (college)  ──▶  ER DIAGRAM        ──▶  TABLES/SQL        ──▶  FILES & INDEXES
                            conceptual               representational       physical
                            (users' view)            (DBMS's view)          (machine's view)
```

| Level | Model | Describes | Audience | Example |
|---|---|---|---|---|
| **Conceptual** (high-level) | **ER model** | Entities, attributes, relationships — close to how humans think | Designers + users | "STUDENT *enrolls in* COURSE" |
| **Representational** (logical) | **Relational model** (also network, hierarchical) | Tables, rows, columns, keys — implementable in a DBMS | Programmers, DBA | `enrollments(roll_no, course_id, grade)` |
| **Physical** (low-level) | Storage model | How data sits on disk: pages, B-tree indexes, hashing, partitions | DBMS engine, DBA | B-tree on `roll_no`, 8 kB pages, WAL |

🔹 **Other models you should recognise:** hierarchical (tree, IBM IMS), network (graph, CODASYL), object-oriented, object-relational, and NoSQL families (document, key-value, column, graph — see Section 10).

🔹 In this course the journey is: **ER diagram (Unit 1) → tables + SQL (Unit 2) → indexes/storage details (Unit 3)** — exactly conceptual → representational → physical.

### 🎯 Questions

**Q1. Name the three categories of data models with one example each. (2 marks)**

✅ **Answer:** Conceptual — ER model; Representational/logical — relational model; Physical — storage/index structures (B-tree, hashing).

**Q2. Converting an ER diagram into tables belongs to which transition? (2 marks)**

✅ **Answer:** Conceptual → representational (logical) modeling.

**Q3. Which data model would a DBMS optimizer work with? (2 marks)**

✅ **Answer:** The physical model — page layouts, index structures and access paths.

---

## 8. Entity-Relationship (ER) Model

📌 The ER model (P.P. Chen, 1976) is the standard **conceptual** design tool: the real world is described as **entities**, their **attributes**, and **relationships** among them, drawn as an **ER diagram**.

### 🔹 Entities & entity sets

- **Entity** — a real-world "thing" with independent existence: one student, one course, one bank account.
- **Entity set / entity type** — the collection/category: *all* students form the STUDENT entity set.
- **Strong entity** — has its own key (`STUDENT` with roll_no).
- **Weak entity** — cannot be identified alone; depends on a strong **owner** + **identifying relationship** (double diamond). Example: `DEPENDENT` (child of an employee) identified by (employee_id, dependent_name).

### 🔹 Attributes — the 6 types

| Type | Meaning | Example |
|---|---|---|
| **Simple (atomic)** | Cannot be divided further | `dob`, `roll_no` |
| **Composite** | Can be split into parts | `name` → first_name + last_name; `address` → city + state + pin |
| **Single-valued** | One value per entity | `dob` |
| **Multivalued** | Many values per entity | `phone` (0..n numbers), `email` |
| **Derived** | Computable from other data — **not stored** | `age` derived from `dob` |
| **Key** | Uniquely identifies each entity | `roll_no` |

```text
-- Attribute sketch for one STUDENT entity
┌──────────────┐
│   STUDENT    │
└──────────────┘
   ├── (roll_no)*        ← key attribute (underlined in textbooks; * here)
   ├── (first_name) ─┐
   ├── (last_name)  ─┴─ both are parts of the COMPOSITE attribute (name)
   ├── (dob)             ← simple, single-valued
   ├── ((phone))         ← multivalued (double oval in textbooks)
   └── (age)†            ← derived from dob (dashed oval in textbooks)
```

### 🔹 Relationships and their degree

- **Relationship** — an association among entities ("Ayush *enrolls in* DBMS").
- **Relationship set** — the set of all such associations.
- **Degree** = number of entity sets participating:

```text
-- Degree of a relationship
UNARY:    EMPLOYEE ──◇ supervises ◇── EMPLOYEE        (relationship to itself)
BINARY:   STUDENT  ──◇ enrolls ◇────── COURSE         (two entity sets — most common)
TERNARY:  SUPPLIER ──◇ supplies ◇───── PART ─── PROJECT  (three at once)
```

### 🔹 Cardinality ratios (how many?)

```text
-- 1:1   one teacher heads exactly one department
┌─────────┐ 1     1 ┌────────────┐
│ TEACHER ├──◇◇◇◇◇──┤ DEPARTMENT │
└─────────┘  HEADS  └────────────┘

-- 1:N   one department has many students
┌────────────┐ 1   N ┌─────────┐
│ DEPARTMENT ├──◇◇◇◇─┤ STUDENT │
└────────────┘  HAS  └─────────┘

-- M:N   many students enroll in many courses
┌─────────┐ M   N ┌─────────┐
│ STUDENT ├──◇◇◇◇─┤ COURSE  │
└─────────┘ENROLLS└─────────┘
```

🔹 ⚠️ **M:N cannot be stored directly with two foreign keys** — the standard fix is a **junction table** like `enrollments(roll_no, course_id, grade)` (see Section 9).

### 🔹 Participation (does every entity take part?)

| Kind | Symbol in books | Meaning | College example |
|---|---|---|---|
| **Total** (existence-dependent) | double line ═ | **Every** entity must participate | Every STUDENT must enroll in ≥1 course |
| **Partial** | single line ─ | Some may not participate | Some INSTRUCTOR may teach no course this term |

### 🔹 ER diagram symbols (cheat sheet)

| Symbol | Meaning | Textbook drawing | In our ASCII notes |
|---|---|---|---|
| Rectangle | Entity set | ▭ | `┌─┐ │ │ └─┘` |
| Double rectangle | Weak entity | ▭▭ | label **(weak)** |
| Diamond | Relationship | ◇ | `◇ NAME ◇` |
| Double diamond | Identifying relationship | ◇◇ | label **(identifying)** |
| Ellipse | Attribute | ( attr ) | `(attr)` |
| Double ellipse | Multivalued attribute | (( attr )) | `((attr))` |
| Dashed ellipse | Derived attribute | ( attr ) | `(age)†` |
| Underlined text | Key attribute | <u>attr</u> | `PK attr` or `*` |
| Line | Connects entity–relationship | ─── | `│` / `─` |
| Double line | Total participation | ═══ | noted as **(total)** |
| 1 / N / M labels | Cardinality at each end | → / ─ | `1`, `N`, `M` |

### 🔹 Worked example — ER design for a College

📌 **Step 1 — Requirements:** a department is headed by one HOD and runs many courses; students have a composite name, several phone numbers, and an age computed from DOB; students enroll in courses and earn a grade per course; instructors teach courses.

📌 **Step 2 — Entities & attributes:** `DEPARTMENT(dept_id PK, dept_name, hod)` · `COURSE(course_id PK, title, credits)` · `STUDENT(roll_no PK, name composite, dob, phone multivalued, age derived)` · `INSTRUCTOR(emp_id PK, name, salary)`

📌 **Step 3 — Relationships:** DEPARTMENT **1:N RUNS** COURSE · INSTRUCTOR **N:1 TEACHES** COURSE · STUDENT **M:N ENROLLS** COURSE (relationship attribute: `grade`)

```text
-- Full College ER diagram (M:N chain) — see symbol cheat sheet above
                                  ┌───────────────────┐
                                  │    DEPARTMENT     │
                                  │───────────────────│
                                  │ PK dept_id        │
                                  │    dept_name      │
                                  │    hod            │
                                  └─────────┬─────────┘
                                            │ 1
                                           ┌┴┐
                                           │R│   ◀ RUNS (partial for dept, 1:N)
                                           └┬┘
                                            │ N
┌───────────────────┐ N               1 ┌──┴────────────────┐
│    INSTRUCTOR     ├───◇ TEACHES ◇─────┤      COURSE       │
│───────────────────│                   │───────────────────│
│ PK emp_id         │                   │ PK course_id      │
│    name           │                   │    title          │
│    salary         │                   │    credits        │
│ FK dept_id        │                   │ FK dept_id        │
└───────────────────┘                   └─────────┬─────────┘
                                                  │ M
                                                 ┌┴┐
                                 (grade) ◀───────│E│  ◀ ENROLLS (M:N, grade is a
                                                 └┬┘     relationship attribute)
                                                  │ N (total participation)
                                            ┌─────┴─────────────┐
                                            │      STUDENT      │
                                            │───────────────────│
                                            │ PK roll_no        │
                                            │    name           │
                                            │    dob            │
                                            │    ((phone))      │
                                            │    (age)†         │
                                            │ FK dept_id        │
                                            └───────────────────┘
```

**How to read this diagram:**
- Boxes = entity sets; the small diamond boxes = relationship diamonds; `1/N/M` labels give the cardinality at each end.
- `PK` marks key attributes, `((phone))` is multivalued, `(age)†` is derived from `dob`, and `(grade)` hangs off the ENROLLS diamond because it belongs to the *pair* (student, course), not to either entity alone.
- STUDENT–ENROLLS is drawn with a **total participation** note: every student must appear in at least one enrollment.
- INSTRUCTOR–TEACHES is **partial** on the instructor side (someone may teach nothing this term).

📌 **Step 4 — mapping preview (ER → tables, fully built in Section 9):** each entity → one table; **M:N ENROLLS → junction table** `enrollments(roll_no, course_id, grade)`; multivalued `phone` → its own table `student_phones(roll_no, phone)`; derived `age` → **never stored**, computed as `current_date - dob`.

### 🎯 Questions

**Q1. Define entity, entity set and attribute. (2 marks)**

✅ **Answer:** Entity — a distinguishable real-world object (e.g. student Ravi). Entity set — collection of similar entities (all students). Attribute — a property describing an entity (e.g. roll_no, dob).

**Q2. Differentiate strong and weak entities with an example. (2 marks)**

✅ **Answer:** A strong entity has its own primary key (STUDENT via roll_no). A weak entity lacks a sufficient key and is identified through its owner + identifying relationship (DEPENDENT of an EMPLOYEE, identified by employee_id + dependent_name); shown with double rectangle + double diamond.

**Q3. In the college diagram, why does `grade` hang on the ENROLLS diamond and not on STUDENT or COURSE? (2 marks)**

✅ **Answer:** Because a grade belongs to the *combination* of a particular student with a particular course — it is an attribute of the relationship, not of either entity alone.

**Q4. Predict: STUDENT–ENROLLS is changed from total to partial participation. What real-world rule disappears? (2 marks)**

✅ **Answer:** "Every student must be enrolled in at least one course." With partial participation, a student may exist in the database with zero enrollments.

---

## 9. Relational Model

📌 Proposed by **E.F. Codd (1970)**: data lives in **relations (tables)** — the *representational* model from Section 7.

| Term | Meaning | In the table below |
|---|---|---|
| **Relation** | A table with unique name | `students` |
| **Tuple** | One row = one record | `(101, Ayush Gupta, ...)` |
| **Attribute** | One named column | `name` |
| **Domain** | Set of allowed atomic values for an attribute | `dept_id` → valid dept ids |
| **Degree** | Number of attributes | 4 |
| **Cardinality** | Number of tuples | 3 |

| roll_no *(PK)* | name | dob | dept_id *(FK)* |
|---|---|---|---|
| 101 | Ayush Gupta | 2006-05-14 | 1 |
| 102 | Moli Gupta | 2007-01-09 | 1 |
| 103 | Ravi Kumar | 2006-11-30 | 2 |

### 🔹 Keys

| Key | Meaning | Example here |
|---|---|---|
| **Super key** | Any attribute set that uniquely identifies a tuple | `{roll_no}`, `{roll_no, name}`, `{email, dob}` |
| **Candidate key** | *Minimal* super key (no extra attributes) | `roll_no`; `email` |
| **Primary key** | The candidate key we choose as the identifier | `roll_no` |
| **Alternate key** | Candidate keys not chosen | `email` |
| **Composite key** | Key made of ≥2 attributes | `(roll_no, course_id)` in `enrollments` |
| **Foreign key** | Attribute referencing the PK of another table | `enrollments.roll_no → students.roll_no` |
| **Unique key** | Column(s) that must differ but may allow NULL | `email UNIQUE` |

### 🔹 Relational integrity constraints

| Constraint | Rule | Enforced in SQL by |
|---|---|---|
| **Domain** | Each attribute value must come from its domain | data type + `CHECK` |
| **Entity integrity** | Primary key can never be NULL or duplicate | `PRIMARY KEY` |
| **Referential integrity** | FK must match an existing parent PK (or be NULL) | `FOREIGN KEY ... REFERENCES` |
| **Key/uniqueness** | Declared unique columns cannot repeat | `UNIQUE` |

🔹 **FK actions** when the parent row dies: `ON DELETE CASCADE` (children die too), `ON DELETE SET NULL` (children survive, FK becomes NULL), default `NO ACTION/RESTRICT` (deletion blocked).

### ✅ Runnable demo 3 — build the college schema from the ER diagram

```sql
DROP TABLE IF EXISTS enrollments, courses, students, departments;  -- wipe demo tables (children first!)
CREATE TABLE departments (                     -- parent table: one row per department
  dept_id    SERIAL PRIMARY KEY,               -- surrogate PK: auto-numbered 1, 2, 3, ...
  dept_name  VARCHAR(50) NOT NULL UNIQUE,      -- name is mandatory and cannot repeat
  hod        VARCHAR(50)                       -- head of department (NULL until appointed)
);                                             -- ends departments
CREATE TABLE students (                        -- child table: references departments
  roll_no   INT PRIMARY KEY,                   -- natural PK: the roll number itself
  name      VARCHAR(60) NOT NULL,              -- mandatory student name
  dob       DATE NOT NULL,                     -- date of birth, mandatory
  email     VARCHAR(100) UNIQUE,               -- optional but unique when present (alternate key)
  dept_id   INT REFERENCES departments (dept_id) ON DELETE SET NULL, -- FK; dept dies → dept_id → NULL
  adm_year  SMALLINT DEFAULT 2025 CHECK (adm_year BETWEEN 2000 AND 2100) -- domain rule on admission year
);                                             -- ends students
CREATE TABLE courses (                         -- child table: references departments
  course_id SERIAL PRIMARY KEY,                -- surrogate PK
  title     VARCHAR(80) NOT NULL,              -- mandatory course title
  credits   SMALLINT NOT NULL CHECK (credits BETWEEN 1 AND 6), -- credits must be 1..6
  dept_id   INT REFERENCES departments (dept_id) ON DELETE CASCADE -- dept dies → its courses die too
);                                             -- ends courses
CREATE TABLE enrollments (                     -- JUNCTION table: resolves the M:N from the ER diagram
  roll_no   INT REFERENCES students (roll_no) ON DELETE CASCADE,  -- part 1 of composite PK + FK
  course_id INT REFERENCES courses (course_id) ON DELETE CASCADE, -- part 2 of composite PK + FK
  grade     CHAR(1) CHECK (grade IN ('A','B','C','D','F')),         -- the relationship's own attribute
  PRIMARY KEY (roll_no, course_id)             -- composite PK: same student can't take a course twice
);                                             -- ends enrollments
```

**Line-by-line walkthrough:**
- `SERIAL` creates the sequence `departments_dept_id_seq`; you never insert `dept_id` yourself.
- FK actions are chosen per relationship: losing a department *cascades* to its courses, but students merely get `dept_id = NULL` (`SET NULL`) — design decisions from the ER diagram.
- `enrollments` is the M:N fix: one row per (student, course) pair; `grade` lives here exactly as `(grade)` hung off the ENROLLS diamond.
- Constraint names are auto-generated as `table_column_type` (e.g. `students_marks_check`) — you'll see them in error messages and `\d`.

```sql
INSERT INTO departments (dept_name, hod)        -- dept_id omitted → SERIAL fills it
VALUES ('CSE', 'Dr. Sharma'),                   -- becomes dept_id 1
       ('ECE', 'Dr. Verma');                    -- becomes dept_id 2
INSERT INTO students (roll_no, name, dob, email, dept_id)  -- adm_year omitted → DEFAULT 2025
VALUES (101, 'Ayush Gupta', '2006-05-14', 'ayush@college.edu', 1),  -- CSE student
       (102, 'Moli Gupta',  '2007-01-09', 'moli@college.edu',  1),  -- CSE student
       (103, 'Ravi Kumar',  '2006-11-30', 'ravi@college.edu',  2);  -- ECE student
INSERT INTO courses (title, credits, dept_id)   -- course_id omitted → SERIAL
VALUES ('Database Management Systems', 4, 1),   -- course 1 (CSE)
       ('Data Structures', 4, 1),               -- course 2 (CSE)
       ('Digital Electronics', 3, 2);           -- course 3 (ECE)
INSERT INTO enrollments (roll_no, course_id, grade)  -- one row per student-course pair
VALUES (101, 1, 'A'),                           -- Ayush got A in DBMS
       (101, 2, 'B'),                           -- Ayush got B in DSA
       (102, 1, 'A'),                           -- Moli got A in DBMS
       (103, 3, 'C');                           -- Ravi got C in DE
SELECT s.roll_no, s.name, c.title, e.grade      -- read the instance through all 3 tables
FROM   enrollments AS e                         -- start from the junction table
JOIN   students AS s ON s.roll_no  = e.roll_no  -- attach the matching student
JOIN   courses  AS c ON c.course_id = e.course_id -- attach the matching course
ORDER  BY s.roll_no;                            -- sort by roll number
```

**Expected output:**

```text
--  roll_no │    name     │            title            │ grade
-- ─────────┼─────────────┼─────────────────────────────┼───────
--      101 │ Ayush Gupta │ Database Management Systems │ A
--      101 │ Ayush Gupta │ Data Structures             │ B
--      102 │ Moli Gupta  │ Database Management Systems │ A
--      103 │ Ravi Kumar  │ Digital Electronics         │ C
-- (4 rows)
```

### ✅ Runnable demo 4 — break the rules, watch integrity constraints fire

```sql
INSERT INTO students (roll_no, name, dob, dept_id)          -- attempt: reuse an existing PK
VALUES (101, 'Duplicate Dan', '2006-02-02', 1);             -- roll_no 101 already exists!
-- ERROR: duplicate key value violates unique constraint "students_pkey"      ← entity integrity
INSERT INTO enrollments (roll_no, course_id, grade)         -- attempt: enroll a ghost student
VALUES (999, 1, 'A');                                       -- roll_no 999 is not in students
-- ERROR: insert or update on table "enrollments" violates foreign key constraint "enrollments_roll_no_fkey"
-- DETAIL: Key (roll_no)=(999) is not present in table "students".           ← referential integrity
INSERT INTO courses (title, credits, dept_id)               -- attempt: zero-credit course
VALUES ('Free Course', 0, 1);                               -- credits must be 1..6
-- ERROR: new row for relation "courses" violates check constraint "courses_credits_check" ← domain/CHECK
DELETE FROM departments WHERE dept_id = 1;                  -- CSE dies → cascade + set-null fire
SELECT title FROM courses;                                  -- which courses survived?
```

**Expected output (of the last two statements):**

```text
-- DELETE 1
--        title
-- ──────────────────────
--  Digital Electronics
-- (1 row)
```

**Line-by-line walkthrough:**
- The PK error names `students_pkey` (the auto index behind the primary key).
- The FK error means: "no parent row with roll_no 999 exists" — referential integrity blocks the orphan.
- The CHECK error proves the domain rule ran *inside the DBMS*, not in some app.
- The `DELETE` triggered both FK actions: CSE's 2 courses were **cascaded away** (so their enrollments vanished too), while students 101/102 silently got `dept_id = NULL`. Only ECE's course remains.

### 🎯 Questions

**Q1. Define relation, tuple, attribute, domain, degree and cardinality. (2 marks)**

✅ **Answer:** Relation — a named table of rows/columns; tuple — a row; attribute — a column; domain — the set of legal atomic values for an attribute; degree — number of attributes; cardinality — number of tuples.

**Q2. Why can't `enrollments` be replaced by a `roll_no` FK in `courses`? (2 marks)**

✅ **Answer:** That would make it 1:N (one course ↔ one student). M:N needs the junction table so each side can hold *many* pairs.

**Q3. Predict the output of `SELECT COUNT(*) FROM enrollments;` after `DELETE FROM departments WHERE dept_id = 1;` (2 marks)**

✅ **Answer:** `1` — only Ravi's enrollment in Digital Electronics (ECE) survives; the two CSE courses cascaded and took their enrollments with them.

---

## 10. Relational vs Non-Relational Databases

📌 **NoSQL** = "not only SQL" — databases that drop the rigid table model for scale/flexibility:

| Family | Data shape | Examples |
|---|---|---|
| Document | JSON-like documents | MongoDB, CouchDB |
| Key-value | Giant dictionary | Redis, DynamoDB |
| Column-family | Column-oriented wide rows | Cassandra, HBase |
| Graph | Nodes + edges | Neo4j |

| Feature | Relational (PostgreSQL, MySQL) | Non-relational (MongoDB, Redis…) |
|---|---|---|
| Data model | Tables (rows × columns) | Documents / key-value / graph / columns |
| Schema | Fixed, defined upfront | Flexible/dynamic (even per document) |
| Query language | SQL (standardised) | APIs / JSON queries / vendor-specific |
| Relationships | First-class (JOINs, FKs) | Avoided; data often denormalised |
| ACID transactions | Full support ✅ | Often BASE / eventual consistency |
| Scaling | Vertical (bigger box); horizontal via sharding | Horizontal-first (built to shard) |
| Best for | Banking, ERP, marks, anything needing consistency | Feeds, logs, caches, IoT, flexible schemas |
| Integrity | PK/FK/CHECK enforced by DBMS | Mostly left to the application |

### ✅ Bonus — PostgreSQL can do NoSQL too (JSONB)

```sql
CREATE TABLE app_events (                     -- NoSQL-style table: only 2 columns are fixed
  event_id    BIGSERIAL PRIMARY KEY,          -- like SERIAL but 8-byte (huge tables)
  captured_at TIMESTAMPTZ DEFAULT now(),      -- server stamps the time automatically
  payload     JSONB NOT NULL                  -- flexible JSON document; inner shape NOT fixed
);                                            -- ends the table
INSERT INTO app_events (payload)              -- only payload given; captured_at auto-fills
VALUES ('{"user": "ayush", "action": "login"}');  -- one JSON document as one row
SELECT payload ->> 'user' AS username         -- ->> extracts a JSON field as plain text
FROM app_events;                              -- reads the document inside the row
```

**Expected output:**

```text
--  username
-- ─────────
--  ayush
-- (1 row)
```

**Line-by-line walkthrough:**
- `JSONB` stores parsed, indexable binary JSON — the row's "schema" inside `payload` can vary per row (no `ALTER TABLE` needed).
- `->>` is PostgreSQL's JSON field extractor; there are GIN indexes for fast lookups inside JSONB.
- Moral: the models are converging — choose relational by default, NoSQL when scale/flexibility demands it.

### 🎯 Questions

**Q1. Give two differences between relational and non-relational databases. (2 marks)**

✅ **Answer:** (1) Schema: fixed/tabular vs flexible/documents; (2) Consistency: full ACID vs often BASE/eventual consistency. *(Also: SQL vs API queries; joins vs denormalisation.)*

**Q2. You must build a banking ledger. Relational or NoSQL? Why? (2 marks)**

✅ **Answer:** Relational (e.g. PostgreSQL) — money needs ACID transactions, enforced integrity constraints and complex joins/reports, which relational DBMSs guarantee.

---

## ⚡ Quick Revision

| Concept | One-line definition |
|---|---|
| Data vs Information | Raw facts vs processed, meaningful facts |
| DBMS | Software to define, store, retrieve and manage databases safely |
| Redundancy | Same data needlessly stored multiple times |
| Inconsistency | Multiple copies of the same fact disagreeing |
| Atomicity | A transaction happens fully or not at all |
| Naive user | End user working through ready-made apps, not SQL |
| DBA | Boss of the DB: schema, security, backup, tuning |
| Query processor | Part of DBMS that translates SQL into efficient execution (parser, optimizer, executor) |
| Storage manager | Part of DBMS that handles files, buffers, transactions, authorization |
| Data dictionary | Metadata storage ("data about data") |
| External level | Individual user views of the data |
| Conceptual level | Whole logical structure: entities, relationships, constraints |
| Internal level | Physical storage: files, indexes, pages |
| 2-tier | Client app connects directly to the DB server |
| 3-tier | Browser → application server (logic) → DB server |
| Physical data independence | Change storage without touching logic/views |
| Logical data independence | Change table structure without rewriting apps (views help) |
| Schema | Structural design/blueprint of the database (DDL) |
| Instance | Actual data at a moment in time (DML) |
| Conceptual model | ER model — human-friendly design view |
| Representational model | Relational model — implementable tables |
| Physical model | How data is stored: indexes, pages, hashing |
| Entity / Entity set | A "thing" / collection of similar things |
| Weak entity | Entity with no key of its own; needs an owner (double rectangle) |
| Derived attribute | Computed from others, never stored (age from dob) |
| Multivalued attribute | Can hold many values (phone) |
| Degree (relationship) | Number of entity sets participating (unary/binary/ternary) |
| Cardinality 1:1, 1:N, M:N | How many of one side map to the other |
| Total participation | Every entity must take part (double line) |
| Relation / Tuple / Attribute | Table / row / column |
| Degree / Cardinality (table) | Number of columns / number of rows |
| Primary key | Chosen unique, non-NULL identifier |
| Foreign key | Column referencing another table's PK; enforces referential integrity |
| Entity integrity | PK is unique and never NULL |
| Referential integrity | FK must match an existing parent value or be NULL |
| NoSQL | Non-tabular DBs (document/key-value/column/graph) trading joins for scale |

---

## 📝 Practice Set

### 🅰️ 2-Mark Questions (with answers)

**Q1. Define DBMS and give two examples. (2 marks)**

✅ **Answer:** A DBMS is system software that lets users define, create, retrieve and manage databases while enforcing integrity, security and concurrent access. Examples: **PostgreSQL**, MySQL (also Oracle, MongoDB).

**Q2. What is data redundancy? How does it lead to inconsistency? (2 marks)**

✅ **Answer:** Redundancy = the same data deliberately/accidentally duplicated in multiple places. If one copy is updated and another is not, the copies disagree — that state is inconsistency. A DBMS stores each fact once, eliminating both.

**Q3. Differentiate database schema and instance with one example each. (2 marks)**

✅ **Answer:** Schema = structure/design, e.g. `students(roll_no INT PK, name VARCHAR(50), marks NUMERIC(5,2))` — changed only by DDL. Instance = data at an instant, e.g. `{(101,'Ayush',88.5)}` — changed by every INSERT/UPDATE/DELETE.

**Q4. What is a derived attribute? Give one example. (2 marks)**

✅ **Answer:** A derived attribute's value is computable from other stored data and is therefore not stored itself. Example: `age` derived from `dob` (in SQL: `current_date - dob`). Drawn as a dashed oval in ER diagrams.

**Q5. Define degree and cardinality of a relation. (2 marks)**

✅ **Answer:** Degree = number of attributes (columns) in the relation. Cardinality = number of tuples (rows) at a given moment. For `students(roll_no, name, dob, dept_id)` with 3 rows: degree 4, cardinality 3.

### 🅱️ 5-Mark Questions (with answers)

**Q6. Explain five limitations of file-processing systems and the corresponding DBMS solution. (5 marks)**

✅ **Answer:**
1. **Data redundancy & inconsistency** — duplicates in many files disagree after partial updates → DB keeps one central copy + constraints.
2. **Difficulty in accessing data** — a new report needs a new program → SQL asks ad-hoc questions instantly.
3. **Data isolation** — data in scattered formats is hard to combine → single schema + JOINs.
4. **Atomicity problems** — crashes leave half-finished updates (money lost) → transactions are atomic (COMMIT/ROLLBACK).
5. **Concurrent-access anomalies & security** — simultaneous updates clash; files can't restrict column access → concurrency control + GRANT/REVOKE per table/column.
*(Conclusion: DBMS = centralised, consistent, secure, recoverable data management.)*

**Q7. Describe the three-level architecture of a DBMS with a diagram. How does it provide data independence? (5 marks)**

✅ **Answer:** Draw the Section 4 ASCII diagram. **External level**: several user views, each showing part of the data (teacher sees marks; library sees names). **Conceptual level**: the complete logical schema — entities, relationships, constraints. **Internal level**: physical storage — files, indexes, compression. The two **mappings** (external↔conceptual, conceptual↔internal) are what buy independence: changing storage only forces an update to the conceptual/internal mapping (physical data independence); restructuring tables only forces an update to the external/conceptual mapping while views keep old shapes (logical data independence). Applications at the top never need rewriting.

**Q8. Draw an ER diagram for a college with at least 3 entities and 2 relationships; include one multivalued, one derived and one composite attribute; state all cardinalities. (5 marks)**

✅ **Answer:** Use the Section 8 College diagram: entities **DEPARTMENT, COURSE, STUDENT** (+INSTRUCTOR optional); relationships **RUNS (1:N)**, **ENROLLS (M:N, attribute grade)**, **TEACHES (N:1)**. Attributes: `name` composite (first/last), `phone` multivalued `(( ))`, `age` derived from `dob`, `roll_no` key. State: DEPARTMENT 1—N COURSE; STUDENT M—N COURSE with grade; total participation of STUDENT in ENROLLS. Mention the M:N → junction-table mapping rule for the viva bonus.

**Q9. Explain the relational integrity constraints with SQL examples. (5 marks)**

✅ **Answer:**
1. **Domain constraint** — each value must belong to the attribute's domain: `credits SMALLINT CHECK (credits BETWEEN 1 AND 6)` rejects 0.
2. **Entity integrity** — the primary key is unique and never NULL: reusing `roll_no 101` raises `duplicate key value violates unique constraint "students_pkey"`.
3. **Referential integrity** — a foreign key must match an existing parent PK or be NULL: enrolling roll_no 999 fails with `violates foreign key constraint`.
4. **Key/UNIQUE constraint** — declared unique columns cannot repeat: a second member with the same email fails with `duplicate key value violates unique constraint "members_email_key"`.
5. (Bonus) **FK actions** decide parent-death behaviour: `ON DELETE CASCADE` removes children, `ON DELETE SET NULL` orphans them safely.

### 🧪 Lab / Hands-On (with solutions)

**Q10. Create a `library` database with `books`, `members` and `issues` tables (PKs, FKs with CASCADE, a CHECK, a DEFAULT), insert 2 rows into each, and list all issues with book title and member name. (Lab)**

```sql
CREATE DATABASE library;                                      -- fresh database for this exercise
```

```text
-- connect to it (psql meta-command; comments must sit on their own line above)
\c library
```

```sql
CREATE TABLE books (                                          -- catalogue of books
  book_id   SERIAL PRIMARY KEY,                               -- auto-numbered PK
  title     VARCHAR(100) NOT NULL,                            -- mandatory title
  author    VARCHAR(60),                                      -- optional author
  copies    INT  NOT NULL CHECK (copies > 0)                  -- business rule: at least 1 copy
);                                                            -- ends books
CREATE TABLE members (                                        -- registered members
  member_id SERIAL PRIMARY KEY,                               -- auto-numbered PK
  name      VARCHAR(60) NOT NULL,                             -- mandatory name
  email     VARCHAR(100) UNIQUE                               -- no duplicate emails
);                                                            -- ends members
CREATE TABLE issues (                                         -- one row per issued book
  issue_id   SERIAL PRIMARY KEY,                              -- auto-numbered PK
  book_id    INT REFERENCES books (book_id)     ON DELETE CASCADE, -- must reference a real book
  member_id  INT REFERENCES members (member_id) ON DELETE CASCADE, -- must reference a real member
  issue_date DATE DEFAULT CURRENT_DATE                        -- today's date filled automatically
);                                                            -- ends issues
INSERT INTO books (title, author, copies)                     -- seed 2 books
VALUES ('DBMS Simplified', 'P. Bhatia', 3),                   -- book_id 1
       ('C Programming',   'D. Ritchie', 2);                  -- book_id 2
INSERT INTO members (name, email)                             -- seed 2 members
VALUES ('Ayush', 'ayush@lib.in'),                             -- member_id 1
       ('Moli',  'moli@lib.in');                              -- member_id 2
INSERT INTO issues (book_id, member_id)                       -- seed 2 issues
VALUES (1, 1),                                                -- Ayush borrowed DBMS Simplified
       (2, 1);                                                -- Ayush borrowed C Programming
SELECT i.issue_id, b.title, m.name, i.issue_date              -- the report: join all three tables
FROM   issues AS i                                            -- start from the child table
JOIN   books   AS b ON b.book_id   = i.book_id                -- attach book details
JOIN   members AS m ON m.member_id = i.member_id              -- attach member details
ORDER  BY i.issue_id;                                         -- chronological listing
```

✅ **Expected output:**

```text
--  issue_id │     title      │ name  │ issue_date
-- ──────────┼────────────────┼───────┼────────────
--         1 │ DBMS Simplified│ Ayush │ 2026-10-31
--         2 │ C Programming  │ Ayush │ 2026-10-31
-- (2 rows)
```

**Q11. In the `library` database, trigger all four kinds of constraint violations and note each psql error message. (Lab)**

```sql
INSERT INTO books (title, copies) VALUES ('Ghost Book', 0);           -- breaks CHECK (copies > 0)
INSERT INTO members (name, email) VALUES ('Copycat', 'ayush@lib.in'); -- breaks UNIQUE on email
INSERT INTO members (name) VALUES (NULL);                             -- breaks NOT NULL on name
INSERT INTO issues (book_id, member_id) VALUES (999, 1);              -- breaks FOREIGN KEY
```

✅ **Answer — the exact errors you will see (one per line):**

```text
-- ERROR: new row for relation "books" violates check constraint "books_copies_check"
-- ERROR: duplicate key value violates unique constraint "members_email_key"
-- ERROR: null value in column "name" of relation "members" violates not-null constraint
-- ERROR: insert or update on table "issues" violates foreign key constraint "issues_book_id_fkey"
-- DETAIL: Key (book_id)=(999) is not present in table "books".
```

Note how each error names its auto-generated constraint (`table_column_check`, `table_column_key`, `table_column_fkey`) — matching them to `CHECK / UNIQUE / NOT NULL / FK` is the whole point of this lab.

**Q12. Prove schema ≠ instance: add a column with a DEFAULT, check what old rows show, then rename a column and protect old queries with a view. (Lab)**

```sql
ALTER TABLE members ADD COLUMN joined_on DATE DEFAULT CURRENT_DATE; -- SCHEMA change: new column
SELECT member_id, name, joined_on FROM members;  -- instance check: existing rows get the default
ALTER TABLE members RENAME COLUMN name TO member_name;  -- LOGICAL change: column renamed
CREATE VIEW members_v AS                         -- external view restoring the OLD shape
SELECT member_id, member_name AS name, email, joined_on FROM members;  -- old names again
SELECT name FROM members_v;                      -- old-style query still works unchanged
```

✅ **Answer/explanation:** After the `ADD COLUMN`, old member rows instantly show today's date (PostgreSQL fills the DEFAULT for existing rows — schema changed, instance followed). After the rename, direct queries on `members.name` would fail, but the view `members_v` re-publishes the old column names — logical data independence from Section 5, now done by your own hands.

**Q13. Explore the metadata of your database using psql meta-commands and `information_schema`. (Lab)**

```text
-- list every database on this PostgreSQL server
\l
-- switch to the college database
\c college
-- list all tables in the current database
\dt
-- show the structure + constraints of students
\d students
```

```sql
SELECT table_name, column_name, data_type         -- read the schema from the system catalogue
FROM   information_schema.columns                 -- the standard SQL data dictionary view
WHERE  table_name = 'students'                    -- filter to our table
ORDER  BY ordinal_position;                       -- keep the original column order
```

✅ **Answer/explanation:** `\l`/`\dt`/`\d` are psql shortcuts; the SQL query shows the *same* metadata through the standard `information_schema` (a data dictionary in action — Section 2). Output of the query: the four rows `students → roll_no(integer) / name(character varying) / marks(numeric) / email(character varying)`.

---

> 📚 **Sources:** Korth/Silberschatz — *Database System Concepts* (Ch. 1–2), Bhatia & Singh — *Simplified Approach to DBMS*, PostgreSQL Official Documentation.
> ✏️ *End of Unit 1 notes — revise the Quick Revision table the night before the exam!* 🎯
