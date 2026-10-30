# Unit 4 — Relational Database Design (Normalization)

👋 Welcome to Unit 4! So far you have learned **how to store data**. This unit teaches you **how to store it *well***. Normalization is a step-by-step cleaning process that reshapes badly designed tables into a series of better and better forms (1NF → 2NF → 3NF → BCNF → 4NF), removing redundancy and the famous **insertion, updation and deletion anomalies** along the way.

📌 One running example is carried through the whole file: a **student–course enrollment table** (`STUDENT_COURSE`), which starts as a messy register and ends as a clean set of normalized tables. Small **PostgreSQL** snippets are shown wherever they make an idea concrete.

**What you'll learn** 🎓
- 📌 The four **data integrity rules** (domain, entity, referential, user-defined) and how PostgreSQL enforces them
- 📌 **Functional Dependencies (FD)** — types, Armstrong's axioms, attribute closure, finding candidate keys
- 📌 Why bad designs cause **insertion / updation / deletion anomalies** and redundancy
- 📌 **1NF, 2NF, 3NF, BCNF** — definition, check steps, worked decomposition each
- 📌 **Multivalued Dependencies (MVD)** and **4NF** with the faculty–course–phone example
- 📌 **Lossless join** and **dependency preservation** checks after every decomposition

---

## 📑 Table of Contents

1. [Data Integrity Rules](#1--data-integrity-rules)
2. [Functional Dependency (FD)](#2--functional-dependency-fd)
3. [Need of Normalization](#3--need-of-normalization)
4. [First Normal Form (1NF)](#4--first-normal-form-1nf)
5. [Second Normal Form (2NF)](#5--second-normal-form-2nf)
6. [Third Normal Form (3NF)](#6--third-normal-form-3nf)
7. [Boyce-Codd Normal Form (BCNF)](#7--boyce-codd-normal-form-bcnf)
8. [Multivalued Dependency (MVD)](#8--multivalued-dependency-mvd)
9. [Fourth Normal Form (4NF)](#9--fourth-normal-form-4nf)
10. [Summary — The Normal Form Ladder](#10--summary--the-normal-form-ladder)
11. [⚡ Quick Revision](#-quick-revision)
12. [📝 Practice Set](#-practice-set)

---

## 1. 🧭 Data Integrity Rules

📌 **Data integrity** means the data in the database is **correct, consistent and valid**. DBMSs protect it using four families of rules.

| # | Rule | One-line meaning | Enforced in PostgreSQL by |
|---|------|------------------|---------------------------|
| 1 | Domain integrity | Every column value comes from the column's **allowed set** (type, range, format) | Data types, `CHECK`, `NOT NULL`, `DEFAULT` |
| 2 | Entity integrity | Every row is uniquely identifiable — **primary key is unique and never NULL** | `PRIMARY KEY` |
| 3 | Referential integrity | A **foreign key** value must match an existing row in the referenced table | `FOREIGN KEY ... REFERENCES` |
| 4 | User-defined integrity | Extra **business rules** of the organization | `CHECK`, `UNIQUE`, triggers |

### 🔹 Domain integrity
📌 Every attribute has a **domain** (legal values). An `age` column must hold integers, not "twenty" or 999.

- ✅ `age = 20` → valid integer, within range
- ❌ `age = 'twenty'` → wrong type; ❌ `age = 250` → wrong range

### 🔹 Entity integrity
📌 In a base table, **no primary key attribute may be NULL**, and key values must be **unique**. If the key is NULL, we cannot distinguish that row from any other row.

- ✅ `roll_no = 101` appears exactly once
- ❌ two rows with `roll_no = 101`, or a row with `roll_no = NULL`

### 🔹 Referential integrity
📌 A foreign key in table **R1** must either be **NULL** or match an existing primary key value in table **R2**. You cannot enroll a student who does not exist.

- ✅ `enroll.roll_no = 101` and student 101 exists in `student`
- ❌ `enroll.roll_no = 999` with no student 999

### 🔹 User-defined integrity
📌 Rules specific to the application, e.g. *"a student's age must be between 16 and 60"*, *"email must be unique"*, *"grade must be one of A/B/C/D/F"*.

```sql
CREATE TABLE student (                          -- creates the STUDENT table
  roll_no INT NOT NULL,                         -- entity integrity: key can never be NULL
  s_name VARCHAR(50) NOT NULL,                  -- domain integrity: name is mandatory text
  age INT CHECK (age BETWEEN 16 AND 60),        -- user-defined rule: college-age range
  email VARCHAR(100) UNIQUE,                    -- user-defined rule: no duplicate emails
  city VARCHAR(30) DEFAULT 'Delhi',             -- domain integrity: sensible default value
  dept_id INT,                                  -- foreign-key column, defined below
  PRIMARY KEY (roll_no),                        -- entity integrity: unique row identifier
  FOREIGN KEY (dept_id) REFERENCES dept (dept_id)  -- referential integrity: dept must exist
);                                              -- ends the CREATE TABLE statement
```

**Line-by-line walkthrough** 🔍
- `roll_no INT NOT NULL` — even before the PK line, NULLs are already blocked.
- `CHECK (age BETWEEN 16 AND 60)` — PostgreSQL rejects any age outside the range.
- `UNIQUE` on email — a duplicate email raises an error at INSERT time.
- `PRIMARY KEY (roll_no)` — adds unique + NOT NULL (entity integrity) automatically.
- `FOREIGN KEY (dept_id) REFERENCES dept (dept_id)` — a student can only point to a department row that exists.

```sql
INSERT INTO student (roll_no, s_name, age, dept_id)  -- try to add one new student row
VALUES (101, 'Aisha', 20, 99);                       -- dept_id 99 does NOT exist in dept
-- ERROR: insert or update on table "student" violates foreign key constraint  -- PostgreSQL rejects the row
```

**Line-by-line walkthrough** 🔍
- The `INSERT` looks harmless, but `dept_id = 99` has no matching `dept` row.
- PostgreSQL's FK check fires and the whole insert is refused — referential integrity saved. ✅

### 🎯 Questions

**Q1. A row is inserted with `roll_no = NULL`. Which integrity rule fails?**
**Answer:** Entity integrity — a primary key must be unique and never NULL. PostgreSQL: `PRIMARY KEY` implies `NOT NULL`, so the INSERT errors out.

**Q2. "Grade must be one of A, B, C, D, F." Which rule, and which SQL keyword?**
**Answer:** User-defined integrity; enforce with `grade CHAR(1) CHECK (grade IN ('A','B','C','D','F'))`.

**Q3. We delete a `dept` row that students still reference. What can happen?**
**Answer:** The DBMS blocks it (`ON DELETE RESTRICT`, default) or deletes/mulls the children (`ON DELETE CASCADE` / `SET NULL`) — both options exist to preserve referential integrity.

---

## 2. 🔗 Functional Dependency (FD)

📌 **Definition:** In relation R, attribute set **Y is functionally dependent on X** (written **X → Y**) if **for any two tuples, whenever they agree on X, they must agree on Y**. Formally:

> `t1[X] = t2[X]  ⟹  t1[Y] = t2[Y]` for all tuples t1, t2 in R.

🔹 **Notation:** X is the **determinant** (left side), Y is the **dependent** (right side). Example: `roll_no → s_name` (roll number decides the name — never the other way round).

### 🔹 Types of FDs

| Type | Meaning | Example |
|------|---------|---------|
| **Trivial** | Y ⊆ X — holds automatically in every relation | `(roll_no, s_name) → s_name` |
| **Non-trivial** | Y ⊄ X — real information | `roll_no → s_name` |
| **Full** | Y depends on **all** of X, not any proper subset | `(roll_no, course_id) → grade` |
| **Partial** | Y depends on **part** of a composite key | `(roll_no, course_id) → s_name` (because `roll_no → s_name`) |
| **Transitive** | X → Y and Y → Z (Y ↛ X), so X → Z indirectly | `course_id → ins_id` and `ins_id → ins_name` ⇒ `course_id → ins_name` |

### 🔹 Armstrong's Axioms (sound & complete inference rules)

**A1. Reflexivity:** if Y ⊆ X, then X → Y.
*Tiny proof:* take tuples t1, t2 agreeing on X. Since Y is a subset of X, they automatically agree on Y. That is exactly X → Y. ∎

**A2. Augmentation:** if X → Y, then XZ → YZ.
*Tiny proof:* let t1[XZ] = t2[XZ]. Then t1[X] = t2[X] and t1[Z] = t2[Z]. From X → Y, t1[Y] = t2[Y]. Put together: t1[YZ] = t2[YZ]. ∎

**A3. Transitivity:** if X → Y and Y → Z, then X → Z.
*Tiny proof:* t1[X] = t2[X] ⇒ (X → Y) ⇒ t1[Y] = t2[Y] ⇒ (Y → Z) ⇒ t1[Z] = t2[Z]. ∎

🔹 **Derived rules** (provable from A1–A3): **Union** (X→Y, X→Z ⇒ X→YZ), **Decomposition** (X→YZ ⇒ X→Y, X→Z), **Pseudotransitivity** (X→Y, WY→Z ⇒ WX→Z). From a set F of FDs, everything derivable is written **F⁺** (closure of F).

### 🔹 Closure of an attribute set — worked example

📌 **X⁺** = the set of all attributes functionally determined by X. Algorithm: start with {X}; repeatedly add the right side of any FD whose left side is fully inside the set; stop when nothing changes.

🧪 **Example:** R(A, B, C, D, E), F = { A → B, B → C, A → E, E → D }. Find **A⁺**:

| Pass | FD applied (LHS ⊆ current set?) | Set after pass |
|------|--------------------------------|----------------|
| 0 | — start with A | {A} |
| 1 | A → B ✅ | {A, B} |
| 2 | B → C ✅ | {A, B, C} |
| 3 | A → E ✅ | {A, B, C, E} |
| 4 | E → D ✅ | {A, B, C, D, E} |
| 5 | nothing new | stop ✅ |

📌 **Result:** A⁺ = {A,B,C,D,E} = all attributes ⇒ **A is a candidate key** of R. (Compare: B⁺ = {B,C} ⇒ B is *not* a key.)

### 🔹 Finding candidate keys from FDs — worked example

🧪 **Example:** R(A, B, C, D), F = { AB → C, C → B }.

**Step 1:** attributes never appearing on any right side must be in **every** key ⇒ A and D are mandatory.
**Step 2:** try to grow {A, D}:

| Trial | Closure | Candidate key? |
|-------|---------|----------------|
| {A, D} | {A, D} — no FD applies | ❌ too small |
| {A, B, D} | AB → C ⇒ {A,B,C,D} = all ✅ | ✅ yes (minimal: {A,B}, {A,D} fail) |
| {A, C, D} | C → B ⇒ {A,B,C,D} = all ✅ | ✅ yes (minimal: {A,C}, {C,D} fail) |

📌 **Answer:** candidate keys = **{A, B, D}** and **{A, C, D}**; prime attributes = {A, B, C, D}. (Two keys because B and C determine each other via C → B and AB → C.)

```sql
SELECT roll_no, COUNT(DISTINCT s_name) AS name_versions  -- count distinct names per roll number
FROM enrollment_flat                                     -- scan the flat enrollment table
GROUP BY roll_no                                         -- gather rows sharing one roll_no
HAVING COUNT(DISTINCT s_name) > 1;                       -- >1 name = roll_no -> s_name is VIOLATED
```

**Line-by-line walkthrough** 🔍
- `COUNT(DISTINCT s_name)` counts how many different names appear under one roll number.
- `GROUP BY roll_no` makes one group per student.
- `HAVING ... > 1` keeps only broken groups: the same roll_no with two names ❌. Empty result ⇒ FD holds ✅.

### 🎯 Questions

**Q1. Given F = {A → B, B → C}, is A → C implied? Name the axiom.**
**Answer:** Yes — by **transitivity** (A→B, B→C ⇒ A→C).

**Q2. Classify `(roll_no, course_id) → s_city` given `roll_no → s_city`.**
**Answer:** It is a **partial** FD — s_city depends on the proper subset `roll_no` of the composite key. (The FD itself is non-trivial.)

**Q3. Compute B⁺ for F = {A → B, B → C, C → D}. Is B a candidate key?**
**Answer:** B⁺ = {B} → +C (B→C) → +D (C→D) = {B,C,D} ≠ all attributes (A missing) ⇒ **B is not a candidate key**.

---

## 3. ⚠️ Need of Normalization

📌 **Normalization** reorganizes tables to remove **redundancy** (same fact stored many times). Redundancy is dangerous because one copy can be updated while another is forgotten — the database then **contradicts itself**.

### 🔹 Our running example — unnormalized register

**STUDENT_COURSE (unnormalized)** — one row per student, all courses stuffed into one cell ❌:

| roll_no | s_name | s_city | courses_taken |
|---------|--------|--------|----------------|
| 101 | Aisha | Delhi | DBMS, OS |
| 102 | Rahul | Pune | DBMS |

Flattened to one row per enrollment (this flat table **ENROLLMENT** is what we analyze below):

| roll_no | s_name | s_city | course_id | course_name | ins_id | ins_name | grade |
|---------|--------|--------|-----------|-------------|--------|----------|-------|
| 101 | Aisha | Delhi | C1 | DBMS | T9 | Dr. Rao | A |
| 101 | Aisha | Delhi | C2 | OS | T7 | Dr. Iyer | B |
| 102 | Rahul | Pune | C1 | DBMS | T9 | Dr. Rao | A |

Candidate key: **(roll_no, course_id)**. FDs: `roll_no → s_name, s_city`; `course_id → course_name, ins_id`; `ins_id → ins_name`; `(roll_no, course_id) → grade`.

### 🔹 Insertion anomaly ❌

📌 We want to add a **new course** C3 "Machine Learning" taught by T5 Dr. Shah — but **no student has enrolled yet**. In ENROLLMENT we cannot insert it: `roll_no` is part of the primary key and **cannot be NULL** (entity integrity). The course simply *cannot exist* in our database until a student joins. ✅ After normalization (3NF), we insert one row into `COURSE` — done.

### 🔹 Updation anomaly ❌

📌 Dr. Rao hands C1 over to Dr. Shah (T5). The fact "C1 → T9, Dr. Rao" is stored **twice** (rows 1 and 3). We must update **every copy**. Forget one row and the table becomes inconsistent:

| roll_no | ... | course_id | course_name | ins_id | ins_name | ... |
|---------|-----|-----------|-------------|--------|----------|-----|
| 101 | ... | C1 | DBMS | T5 | Dr. Shah | ... |
| 101 | ... | C2 | OS | T7 | Dr. Iyer | ... |
| 102 | ... | C1 | DBMS | T9 | Dr. Rao | ... ❌ stale copy! |

✅ After 3NF, course-instructor facts live in **one** row of `COURSE` — one update, zero risk.

### 🔹 Deletion anomaly ❌

📌 Rahul (102) drops DBMS. Deleting his row **also destroys unrelated facts**: "course C1 exists" and "C1 is taught by Dr. Rao" vanish, because they had no home of their own. ✅ After 3NF, deleting an enrollment removes only the enrollment.

### 🔹 Redundancy — the cost

| Fact | Flat ENROLLMENT | Normalized (3NF) |
|------|-----------------|------------------|
| Aisha's name & city | stored **once per course** (2× here; 6× for 6 courses) | 1× in `STUDENT` |
| C1's instructor info | stored **once per enrolled student** | 1× in `COURSE`/`INSTRUCTOR` |
| 5,000 enrollments × 800 students | s_name stored 5,000 times | s_name stored 800 times |

### 🎯 Questions

**Q1. Name the three anomalies with one line each.**
**Answer:** **Insertion** — cannot add a fact (new course) until an unrelated fact (enrollment) exists; **Updation** — changing a repeated fact requires touching many rows, or the DB becomes inconsistent; **Deletion** — deleting one fact accidentally destroys another.

**Q2. Why is redundancy dangerous even if the data is correct *today*?**
**Answer:** Because every future update is a chance to update one copy and miss another — leading to **inconsistency**; plus storage waste and slower joins.

**Q3. Which FD causes the updation anomaly in our example?**
**Answer:** `course_id → course_name, ins_id` (and `ins_id → ins_name`) — course facts are repeated once per enrolled student instead of being stored once.

---

## 4. 🥇 First Normal Form (1NF)

📌 **Definition:** A relation is in **1NF** if every attribute contains only **atomic (indivisible) values** — no multivalued cells, no repeating groups, no lists inside a cell — and every row is uniquely identifiable (there is a key).

### 🔹 How to check (steps)

1. 🔍 Does any cell contain a **list / set / comma-joined value**? → not 1NF.
2. 🔍 Are there **repeating groups** of columns like `course1, course2, course3`? → not 1NF.
3. 🔍 Is there a **primary key** making each row unique? → required.
4. ✅ All cells atomic + key present ⇒ 1NF.

### 🔹 Worked example — converting a multivalued column

**❌ Before (not 1NF):**

| roll_no | s_name | courses_taken |
|---------|--------|----------------|
| 101 | Aisha | DBMS, OS |
| 102 | Rahul | DBMS |

**✅ After (1NF)** — one row per (student, course) pair:

| roll_no | s_name | course_id |
|---------|--------|-----------|
| 101 | Aisha | C1 |
| 101 | Aisha | C2 |
| 102 | Rahul | C1 |

```sql
CREATE TABLE enroll_1nf (               -- the 1NF design: one row per (student, course)
  roll_no INT,                          -- student roll number
  course_id VARCHAR(5),                 -- exactly ONE course per row (atomic value)
  PRIMARY KEY (roll_no, course_id)      -- composite key makes every row unique
);                                      -- ends the CREATE TABLE statement
```

**Line-by-line walkthrough** 🔍
- `roll_no INT` — first half of the composite key.
- `course_id VARCHAR(5)` — a single course code like 'C1'; never 'C1,C2'.
- `PRIMARY KEY (roll_no, course_id)` — entity integrity on the pair; the same course can't repeat for the same student.

📌 **What anomaly is removed:** the "list-in-a-cell" storage problem — you can now query, index and constrain individual enrollments (e.g., `WHERE course_id = 'C1'`). ⚠️ Note: 1NF does **not** remove redundancy — Aisha's name is still repeated. That is 2NF's job.

### 🎯 Questions

**Q1. Is the value "98765-1111, 98765-2222" in one cell atomic?**
**Answer:** No — it is a list of two phone numbers; 1NF requires splitting them into separate rows.

**Q2. Does 1NF remove the insertion anomaly?**
**Answer:** No — it only fixes non-atomic cells. The course C3 still cannot be inserted without a student (key part `roll_no` would be NULL).

**Q3. A table has columns `phone1, phone2, phone3`. Is it 1NF?**
**Answer:** No — that is a **repeating group**; replace with a separate one-row-per-phone table.

---

## 5. 🥈 Second Normal Form (2NF)

📌 **Definition:** A relation is in **2NF** if it is in **1NF** and **no non-prime attribute depends on a proper subset of any candidate key** — i.e., **no partial dependency**. (Prime attribute = member of some candidate key; non-prime = the rest.)

### 🔹 How to check (steps)

1. 🔍 Find the candidate key(s). If the key is **single-attribute** → automatically 2NF ✅.
2. 🔍 List all FDs; mark each as **full** or **partial** w.r.t. the composite key.
3. ❌ Any FD: part-of-key → non-prime attribute ⇒ violation.
4. ✅ Otherwise 2NF.

### 🔹 Worked example — decomposing ENROLLMENT

Key = **(roll_no, course_id)**. Check each FD:

| FD | Full or partial? | Verdict |
|----|------------------|---------|
| `(roll_no, course_id) → grade` | needs **both** parts | ✅ full |
| `roll_no → s_name, s_city` | only **half** the key | ❌ partial |
| `course_id → course_name, ins_id, ins_name` | only **half** the key | ❌ partial |

**❌ ENROLLMENT is 1NF, not 2NF.** Decompose — each FD's determinant gets its own table:

**✅ STUDENT (key: roll_no)**

| roll_no | s_name | s_city |
|---------|--------|--------|
| 101 | Aisha | Delhi |
| 102 | Rahul | Pune |

**✅ COURSE (key: course_id)**

| course_id | course_name | ins_id | ins_name |
|-----------|-------------|--------|----------|
| C1 | DBMS | T9 | Dr. Rao |
| C2 | OS | T7 | Dr. Iyer |

**✅ ENROLL (key: roll_no, course_id)**

| roll_no | course_id | grade |
|---------|-----------|-------|
| 101 | C1 | A |
| 101 | C2 | B |
| 102 | C1 | A |

📌 **Lossless join check:** `ENROLL ⋈ STUDENT ⋈ COURSE` reproduces the original ENROLLMENT exactly — no spurious rows, nothing lost. ✅

📌 **What anomaly is removed:** student facts are stored once (no update-3-rows problem), course facts exist even with zero enrollments (insertion anomaly gone for students/courses), and dropping an enrollment no longer erases student/course data.

### 🎯 Questions

**Q1. When can a table never violate 2NF?**
**Answer:** When its candidate key is a **single attribute** — there is no "proper subset" of the key to depend on.

**Q2. Key = (roll_no, course_id); FD: `course_id → course_name`. Classify and judge 2NF.**
**Answer:** **Partial FD** (depends on part of the key) ⇒ table is **not in 2NF** ❌; move course facts to a COURSE table.

**Q3. What is a prime attribute?**
**Answer:** Any attribute that belongs to **at least one candidate key**; all others are non-prime.

**Q4. Is the 2NF decomposition above dependency-preserving?**
**Answer:** Yes — every original FD (`roll_no → …`, `course_id → …`, `(roll_no, course_id) → grade`) can be checked inside one of the three tables. ✅

---

## 6. 🥉 Third Normal Form (3NF)

📌 **Definition:** A relation is in **3NF** if it is in **2NF** and **no non-prime attribute is transitively dependent on a candidate key**. Formal test: for every non-trivial FD **X → A**, either **X is a superkey**, or **A is a prime attribute**.

### 🔹 How to check (steps)

1. 🔍 Confirm 2NF first.
2. 🔍 For each FD X → A where A is non-prime: is X a **superkey**?
3. ❌ A chain `key → Y → A` (Y not a key, A non-prime) = **transitive dependency** ⇒ violation.
4. ✅ Otherwise 3NF.

### 🔹 Worked example — COURSE after 2NF

**❌ COURSE (key: course_id)** — 2NF ✅ but not 3NF:

| course_id | course_name | ins_id | ins_name |
|-----------|-------------|--------|----------|
| C1 | DBMS | T9 | Dr. Rao |
| C2 | OS | T7 | Dr. Iyer |

Transitive chain: `course_id → ins_id` and `ins_id → ins_name` ⇒ `course_id → ins_name` ❌ (ins_id is not a key, ins_name is non-prime).

**✅ Decompose:**

**COURSE (key: course_id)**

| course_id | course_name | ins_id |
|-----------|-------------|--------|
| C1 | DBMS | T9 |
| C2 | OS | T7 |

**INSTRUCTOR (key: ins_id)**

| ins_id | ins_name |
|--------|----------|
| T9 | Dr. Rao |
| T7 | Dr. Iyer |

📌 **Lossless check:** COURSE ∩ INSTRUCTOR = {ins_id}, and `ins_id → ins_name` (all of INSTRUCTOR) ⇒ lossless ✅. Dependencies preserved ✅. (STUDENT and ENROLL from 2NF already satisfy 3NF — nothing more to do.)

📌 **What anomaly is removed:** an instructor's name is updated in **one** place; a new instructor can be added **before** teaching any course; deleting a course no longer erases the instructor.

### 🎯 Questions

**Q1. Define transitive dependency in one line with our example.**
**Answer:** key → ins_id → ins_name: a non-prime attribute (ins_name) depends on the key only *through* another non-key attribute (ins_id).

**Q2. State the formal 3NF condition.**
**Answer:** For every non-trivial FD X → A: **X is a superkey OR A is prime**.

**Q3. Can a 3NF table still have redundancy?**
**Answer:** Yes — redundancy from FDs like `teacher → subject` where the determinant is not a key but the dependent is prime; that needs **BCNF** (next section).

---

## 7. 🏆 Boyce-Codd Normal Form (BCNF)

📌 **Definition:** A relation is in **BCNF** if for **every non-trivial FD X → Y, X is a superkey** — i.e., **every determinant is a candidate key**. This is **stricter than 3NF**: 3NF forgives `X → Y` when Y is prime; BCNF forgives nothing. Every BCNF table is 3NF ✅, but not every 3NF table is BCNF ❌.

### 🔹 How to check (steps)

1. 🔍 List all non-trivial FDs and all candidate keys.
2. 🔍 For each FD: is the determinant (LHS) a **superkey**?
3. ❌ Any LHS that is not a superkey ⇒ not BCNF.
4. If all LHS are superkeys ⇒ BCNF ✅ (and 3NF automatically).

### 🔹 Worked example — 3NF ✅ but BCNF ❌

🧪 **TEACHING (student, subject, teacher)** with:
- `teacher → subject` (each teacher teaches exactly one subject)
- `(student, subject) → teacher` (for a subject, a student has exactly one teacher)

| student | subject | teacher |
|---------|---------|---------|
| S1 | DBMS | Rao |
| S1 | Networks | Menon |
| S2 | DBMS | Rao |
| S2 | Networks | Menon |

**Candidate keys:** (student, subject) and (student, teacher) — each determines all three attributes. ⇒ **Every attribute is prime**.

- **3NF check:** `teacher → subject`: teacher is not a superkey, but **subject is prime** ⇒ 3NF still satisfied ✅.
- **BCNF check:** `teacher → subject`: teacher is **not a superkey** ⇒ **BCNF violated** ❌.

**✅ BCNF decomposition** (split off the offending FD):

**TEACHER_SUB (key: teacher)**

| teacher | subject |
|---------|---------|
| Rao | DBMS |
| Menon | Networks |

**STUDENT_TEACHER (key: student, teacher)**

| student | teacher |
|---------|---------|
| S1 | Rao |
| S1 | Menon |
| S2 | Rao |
| S2 | Menon |

📌 **Lossless check:** common attribute = `teacher`; `teacher → subject` (all of TEACHER_SUB) ⇒ lossless ✅.
📌 **Dependency check:** `(student, subject) → teacher` cannot be checked inside any single table — it is **lost** ❌ (a known price of BCNF; see Section 10).

📌 **What anomaly is removed:** before decomposition, adding a new teacher + subject required an artificial student row; after decomposition, `TEACHER_SUB` stores teacher–subject facts on their own, once each.

### 🎯 Questions

**Q1. State BCNF in one line.**
**Answer:** Every non-trivial determinant (LHS of every FD) must be a **superkey**.

**Q2. Is every 3NF table in BCNF? Is every BCNF table in 3NF?**
**Answer:** 3NF ⇒ BCNF: **No** (prime-attribute escape hatch). BCNF ⇒ 3NF: **Yes** always.

**Q3. When do 3NF and BCNF coincide?**
**Answer:** When every determinant of every non-trivial FD is already a candidate key — e.g., single-key tables where all FDs point out of the key only.

---

## 8. 🧩 Multivalued Dependency (MVD)

📌 **Definition:** In R, **X ↠ Y** (read: *"X multidetermines Y"*) holds if, whenever two tuples t1 and t2 **agree on X**, the relation must also contain the two **swapped** tuples built from t1, t2 and the rest Z = R − X − Y:

- t3: same X as t1/t2, **Y from t1**, **Z from t2**
- t4: same X as t1/t2, **Y from t2**, **Z from t1**

🗣️ In words: *Y-values and Z-values of an X are **independent sets*** — every combination must appear. (MVDs were added to relational theory by Zaniolo/Fagin; they complete FDs.)

### 🔹 When does an MVD occur?

When one entity has **two independent 1:N facts**. E.g., a faculty member teaches **many courses** and owns **many phones**, and *which* courses has nothing to do with *which* phones. Forcing both into one table forces a **full cross product**:

| faculty | course | phone |
|---------|--------|-------|
| Rao | C1 | P1 |
| Rao | C1 | P2 |
| Rao | C2 | P1 |
| Rao | C2 | P2 |
| Menon | C3 | P3 |

**Swap test:** from (Rao, C1, P1) and (Rao, C2, P2), swapping Y and Z must give (Rao, C1, P2) and (Rao, C2, P1) — both present ✅ ⇒ `faculty ↠ course` (and symmetrically `faculty ↠ phone`). If we tried to store only 2 of Rao's 4 rows, the MVD would be violated and the table would lie about reality ❌.

### 🔹 Trivial MVD

📌 **X ↠ Y is trivial** if **Y ⊆ X** or **X ∪ Y = R** (all attributes). Trivial MVDs hold in every relation and are ignored, just like trivial FDs. Example: `(faculty, course) ↠ course` ✅ trivial.

🔹 **Extra facts:** every FD X → Y is automatically the MVD X ↠ Y (FD ⇒ MVD, never the reverse). MVDs also obey inference rules (e.g., **complementation:** X ↠ Y ⇒ X ↠ R−X−Y).

```sql
SELECT f1.faculty, f1.course, f2.phone        -- build every legal (faculty, course, phone) combo
FROM faculty_course f1                        -- all courses each faculty teaches
CROSS JOIN faculty_phone f2                   -- all phones each faculty owns
WHERE f1.faculty = f2.faculty                 -- both facts must belong to the same faculty
  AND NOT EXISTS (                            -- keep only combos MISSING from the big table
        SELECT 1 FROM faculty_all f3          -- look inside the unnormalized table
        WHERE f3.faculty = f1.faculty         -- same faculty
          AND f3.course  = f1.course          -- same course
          AND f3.phone   = f2.phone);         -- same phone
```

**Line-by-line walkthrough** 🔍
- `faculty_course CROSS JOIN faculty_phone` (restricted to the same faculty) = exactly the rows the MVD says **must** exist.
- `NOT EXISTS (...)` filters to combinations absent from `faculty_all` — each output row is proof the big table violates its own MVDs ❌.

### 🎯 Questions

**Q1. Write and read aloud the MVD notation for "a faculty's courses are independent of its phones".**
**Answer:** `faculty ↠ course` — "faculty multidetermines course": the set of courses is independent of the rest (phones).

**Q2. Is `(faculty, course) ↠ course` trivial? Why?**
**Answer:** Yes — the right side is a subset of the left side, so it holds in every possible relation.

**Q3. Is every FD an MVD? Is every MVD an FD?**
**Answer:** Every FD X → Y implies X ↠ Y ✅; the converse is false — an MVD only forces *all combinations*, not a single value.

**Q4. How many rows must the faculty table store for a faculty with 3 courses and 2 phones?**
**Answer:** 3 × 2 = **6 rows** — the forced cross product, which is exactly the redundancy 4NF will remove.

---

## 9. 💠 Fourth Normal Form (4NF)

📌 **Definition:** A relation is in **4NF** if, for **every non-trivial MVD X ↠ Y, X is a superkey** (equivalently X → R). In words: the only allowed multivalued facts are *trivial* ones or ones **keyed** on a candidate key. 4NF ⇒ BCNF ✅ (since every FD is an MVD).

### 🔹 How to check (steps)

1. 🔍 Confirm BCNF first.
2. 🔍 List all **non-trivial MVDs** X ↠ Y.
3. ❌ Any such X that is **not** a superkey ⇒ not 4NF.
4. ✅ Otherwise 4NF.

### 🔹 Worked example — faculty → course, faculty → phone

**❌ FACULTY_ALL (faculty, course, phone)** — key = all three attributes (no non-trivial FDs at all, so it is trivially in BCNF ✅), but:

- `faculty ↠ course` ✅ non-trivial
- `faculty ↠ phone` ✅ non-trivial
- `faculty` is **not** a superkey (Rao's row does not fix course or phone alone) ⇒ **not 4NF** ❌

| faculty | course | phone |
|---------|--------|-------|
| Rao | C1 | P1 |
| Rao | C1 | P2 |
| Rao | C2 | P1 |
| Rao | C2 | P2 |
| Menon | C3 | P3 |

📌 **Anomalies visible:** adding course C4 to Rao needs **2 rows** (one per phone); adding phone P3 to Rao needs **2 rows** (one per course); deleting *one* combination leaves the cross product incomplete (inconsistent).

**✅ 4NF decomposition** — one table per independent multivalued fact:

**FACULTY_COURSE (key: faculty, course)**

| faculty | course |
|---------|--------|
| Rao | C1 |
| Rao | C2 |
| Menon | C3 |

**FACULTY_PHONE (key: faculty, phone)**

| faculty | phone |
|---------|-------|
| Rao | P1 |
| Rao | P2 |
| Menon | P3 |

📌 **Lossless check:** common attribute = `faculty`; joining on it regenerates exactly the 5 original rows (Rao: 2×2 = 4, Menon: 1) — nothing lost, nothing spurious ✅.
📌 **What anomaly is removed:** adding C4 = **1 row** in FACULTY_COURSE; adding phone P3 = **1 row** in FACULTY_PHONE; the "incomplete cross product" inconsistency becomes impossible.

### 🎯 Questions

**Q1. State 4NF in one line.**
**Answer:** Every non-trivial MVD X ↠ Y must have **X as a superkey**.

**Q2. Does 4NF imply BCNF?**
**Answer:** Yes — every FD is an MVD, so keying all non-trivial MVDs automatically keys all FDs ⇒ BCNF holds.

**Q3. In the decomposed design, how many rows to add one course to Rao?**
**Answer:** **One** row in FACULTY_COURSE — the phone cross-multiplication is gone.

---

## 10. 🪜 Summary — The Normal Form Ladder

| NF | Removes | Test (one line) |
|----|---------|-----------------|
| 1NF | Multivalued cells, repeating groups | All values **atomic**; key exists |
| 2NF | **Partial** dependency | No non-prime attribute depends on a **proper subset** of a candidate key |
| 3NF | **Transitive** dependency | Every FD X → A: X superkey **or** A prime |
| BCNF | FDs with non-key determinants | **Every determinant is a candidate key** |
| 4NF | Independent **MVDs** | Every non-trivial MVD X ↠ Y: X superkey |
| 5NF *(one-liner, beyond syllabus)* | Join dependencies | Every join dependency implied by candidate keys |

📌 Each level **includes** all previous levels: 4NF ⇒ BCNF ⇒ 3NF ⇒ 2NF ⇒ 1NF.

### 🔹 Lossless join — notes

- A decomposition R → R1, R2 is **lossless** if `R1 ⋈ R2 = R` exactly — you can always **rebuild the original table** by joining (no fake rows, nothing destroyed).
- 🧪 **Quick test:** if `R1 ∩ R2 → R1` or `R1 ∩ R2 → R2` (the shared attributes determine one side), the join is lossless. *Check it for every decomposition in this file — all pass ✅.*
- 🗣️ Meaning: you can always **rebuild the original table** by joining; no information is destroyed.

### 🔹 Dependency preservation — notes

- A decomposition **preserves dependencies** if every original FD can be checked within a **single** decomposed table (no joins needed to enforce it).
- ✅ **3NF:** a lossless **and** dependency-preserving decomposition *always exists* (synthesis algorithm). That is why 3NF is the practical favourite.
- ❌ **BCNF:** lossless is always achievable, but dependency preservation may be **lost** — our TEACHING decomposition cannot enforce `(student, subject) → teacher` without a join. 📝 Exam tip: after any decomposition, always state **both** checks — most 5-mark questions award marks for exactly these two lines.

---

## ⚡ Quick Revision

| NF | Removes | Rule of thumb |
|----|---------|---------------|
| 1NF | Lists inside cells | "One cell, one value" 📦 |
| 2NF | Partial FDs | "Whole key, nothing less" 🔑 |
| 3NF | Transitive FDs | "Non-key → non-key is banned" ⛓️ |
| BCNF | Non-key determinants | "Every determinant must be a key" 🏆 |
| 4NF | Independent MVDs | "One independent multivalue fact per table" 🧩 |

🔹 Memory hooks: **P**artial → 2NF, **T**ransitive → 3NF, **D**eterminant-must-be-key → BCNF, **I**ndependent-multivalues → 4NF.
🔹 Check order: find keys → list FDs/MVDs → test 1NF → 2NF → 3NF → BCNF → 4NF, decomposing at the first failure.

---

## 📝 Practice Set

### 🅰️ Section A — 2-Mark Questions

#### Q1. Define functional dependency. Two different roll numbers have the same name — is `roll_no → s_name` violated?
**Answer:** ✅ **No.** An FD X → Y means: *any two tuples agreeing on X must agree on Y*. A violation needs the **same** roll_no appearing with **two different names**. Different rolls may legally share a name.

#### Q2. What is a trivial FD? Give an example and say why it is ignored.
**Answer:** X → Y is **trivial** if Y ⊆ X, e.g. `(roll_no, course_id) → roll_no`. It holds in **every** relation, adds no information, and can never justify a decomposition — so normalization ignores it.

#### Q3. State entity integrity and referential integrity with the PostgreSQL keyword for each.
**Answer:** **Entity integrity:** primary key values are unique and never NULL → enforced by `PRIMARY KEY`. **Referential integrity:** every FK value must match an existing key in the referenced table (or be NULL) → enforced by `FOREIGN KEY ... REFERENCES`.

#### Q4. Using Armstrong's axioms, prove X → Y and X → Z ⇒ X → YZ.
**Answer:** (1) Augment X → Z with Y: **XY → YZ**. (2) Augment X → Y with X: XX → XY, i.e. **X → XY**. (3) Transitivity: X → XY and XY → YZ ⇒ **X → YZ**. ∎ (This derived rule is **Union**.)

### 🅱️ Section B — 5-Mark Questions (find highest NF / decompose)

#### Q5. R(A, B, C, D), F = {A → B, B → C, A → D}. Find A⁺, all candidate keys, the highest NF, and decompose if needed.
**Answer (step-by-step):**
1. 🔍 **A⁺:** {A} → +B (A→B) → +C (B→C) → +D (A→D) = {A,B,C,D} = all.
2. 🔍 **Keys:** B⁺ = {B,C} ≠ all, C⁺ = {C}, D⁺ = {D} ⇒ only **A** is a candidate key; prime = {A}.
3. 🔍 **2NF:** single-attribute key ⇒ no partial dependency possible ⇒ 2NF ✅.
4. ❌ **3NF:** B → C: B is not a superkey and C is non-prime ⇒ 3NF violated. **Highest NF = 2NF.**
5. ✂️ **Decompose:** R1(B, C) and R2(A, B, D).
6. ✅ **Lossless:** R1 ∩ R2 = {B} and B → C (all of R1) ⇒ lossless.
7. ✅ **Preserved:** A→B, A→D in R2; B→C in R1. Both tables in **BCNF** (keys B and A respectively).

#### Q6. SALES(sale_id, prod_id, prod_name, qty), key = (sale_id, prod_id), FD: prod_id → prod_name. Decompose to 2NF.
**Answer (step-by-step):**
1. 🔍 FDs: `(sale_id, prod_id) → qty` ✅ full; `prod_id → prod_name` ⇒ `(sale_id, prod_id) → prod_name` is **partial** ❌ (depends on part of the key).
2. ❌ **2NF violated** (prod_name is non-prime).
3. ✂️ **Decompose:** `PRODUCT(prod_id, prod_name)` and `SALE_LINE(sale_id, prod_id, qty)`.
4. ✅ **Lossless:** shared attr prod_id; `prod_id → prod_name` (all of PRODUCT) ✅. **Preserved:** both FDs checkable ✅.
5. 📌 If a `sale_date` also existed with `sale_id → sale_date`, add a third table `SALES(sale_id, sale_date)`.

#### Q7. EMP(emp_id, emp_name, dept_no, dept_name, dept_loc), F = {emp_id → emp_name, dept_no; dept_no → dept_name, dept_loc}. Find the highest NF and decompose to 3NF.
**Answer (step-by-step):**
1. 🔍 Key: emp_id (emp_id⁺ = all attributes); prime = {emp_id}.
2. 🔍 **2NF:** single-attribute key ⇒ 2NF ✅.
3. ❌ **3NF:** emp_id → dept_no → dept_name/dept_loc is **transitive** (dept_no not a key, targets non-prime). **Highest NF = 2NF.**
4. ✂️ **Decompose:** `EMP(emp_id, emp_name, dept_no)` and `DEPT(dept_no, dept_name, dept_loc)`.
5. ✅ **Lossless:** shared dept_no, `dept_no → dept_name, dept_loc` (all of DEPT) ✅. **Preserved:** all three FDs intact ✅. Both tables in **BCNF**.

#### Q8. R(P, Q, R) with F = {PQ → R, R → Q}. Find all candidate keys, prove the highest NF, decompose to BCNF, and state what is lost.
**Answer (step-by-step):**
1. 🔍 **Keys:** PQ⁺ = {P,Q,R} ⇒ PQ key; PR⁺ = {P,R,Q} ⇒ PR key; P⁺ = {P} ⇒ not. Candidate keys: **PQ, PR**; all attributes are **prime**.
2. 🔍 **3NF:** PQ → R: superkey ✅. R → Q: R is not a superkey but **Q is prime** ⇒ 3NF holds ✅.
3. ❌ **BCNF:** R → Q has a non-superkey determinant ⇒ **BCNF violated**. **Highest NF = 3NF.**
4. ✂️ **Decompose:** `R1(R, Q)` and `R2(P, R)`.
5. ✅ **Lossless:** R1 ∩ R2 = {R}, R → Q (all of R1) ✅. Both in BCNF (keys R and PR).
6. ❌ **Lost:** **PQ → R** lives in neither table — the decomposition is **not dependency-preserving** (the classic BCNF trade-off).

### 🧪 Section C — Apply-Type Questions

#### Q9. LIBRARY issue register: ISSUE(issue_id, book_id, title, member_id, member_name, member_type, issue_date, return_date) with book_id → title; member_id → member_name, member_type. Normalize to 3NF.
**Answer (step-by-step):**
1. 🔍 **Key:** issue_id (determines all columns). FDs: `book_id → title`; `member_id → member_name, member_type`.
2. 🔍 **1NF:** all values atomic ✅. **2NF:** single-attribute key ⇒ no partial FDs ✅.
3. ❌ **3NF:** issue_id → book_id → title and issue_id → member_id → member_name/member_type are **transitive** chains.
4. ✂️ **Decompose:** `BOOK(book_id, title)`, `MEMBER(member_id, member_name, member_type)`, `ISSUE(issue_id, book_id, member_id, issue_date, return_date)`.
5. ✅ **Lossless:** ISSUE shares book_id with BOOK (book_id → all of BOOK) and member_id with MEMBER (member_id → all of MEMBER) ✅. **Preserved:** every FD sits in one table ✅. Result: 3NF (in fact BCNF) 🎉.

#### Q10. Write PostgreSQL DDL for the final design of our running example (STUDENT, INSTRUCTOR, COURSE, ENROLL) and show an INSERT that referential integrity will reject.
**Answer (step-by-step):** Four tables in 3NF; ENROLL carries two foreign keys.

```sql
CREATE TABLE student (                        -- students, one row each (2NF level fact)
  roll_no INT PRIMARY KEY,                    -- entity integrity: unique, never NULL
  s_name VARCHAR(50) NOT NULL,                -- domain integrity: mandatory name
  s_city VARCHAR(30)                          -- atomic city value
);                                            -- end of student table
CREATE TABLE instructor (                     -- instructors extracted by 3NF
  ins_id VARCHAR(5) PRIMARY KEY,              -- key used by COURSE as a foreign key
  ins_name VARCHAR(50) NOT NULL               -- instructor name stored exactly once
);                                            -- end of instructor table
CREATE TABLE course (                         -- courses, transitive dependency removed
  course_id VARCHAR(5) PRIMARY KEY,           -- course code is the key
  course_name VARCHAR(50) NOT NULL,           -- course title stored once
  ins_id VARCHAR(5) REFERENCES instructor     -- referential integrity: teacher must exist
);                                            -- end of course table
CREATE TABLE enroll (                         -- pure relationship table from 2NF
  roll_no INT REFERENCES student,             -- FK to student: who
  course_id VARCHAR(5) REFERENCES course,     -- FK to course: what
  grade CHAR(1),                              -- full FD on the composite key
  PRIMARY KEY (roll_no, course_id)            -- composite key: one grade per pair
);                                            -- end of enroll table
```

**Line-by-line walkthrough** 🔍
- `student` / `instructor` / `course` — one row per real-world entity; every non-key fact stored exactly once.
- `ins_id VARCHAR(5) REFERENCES instructor` and the two FKs in `enroll` — rows cannot point at non-existent parents (referential integrity).
- `PRIMARY KEY (roll_no, course_id)` in `enroll` — prevents duplicate enrollment of the same pair.

```sql
INSERT INTO enroll (roll_no, course_id, grade)      -- attempt to record a grade
VALUES (999, 'C1', 'A');                            -- roll_no 999 has NO row in student
-- ERROR: insert or update on table "enroll" violates foreign key constraint  -- rejected by PostgreSQL
```

**Line-by-line walkthrough** 🔍
- Student 999 does not exist, so the FK check fails and the row is refused ✅.
- **Fix:** insert student 999 into `student` first, then re-run this INSERT — or drop the bogus row entirely.

#### Q11. CONSULTANT(consultant, client, skill) has `consultant ↠ client` and `consultant ↠ skill`, with no non-trivial FDs. Prove it is not in 4NF and decompose it.
**Answer (step-by-step):**
1. 🔍 **Keys:** no non-trivial FDs ⇒ only key is the whole relation ⇒ 1NF/2NF/3NF/BCNF all trivially ✅.
2. 🔍 **MVDs:** both are non-trivial (neither side subset, union ≠ R).
3. ❌ **4NF:** `consultant` is **not** a superkey ⇒ violated. Sample proof by swap: (Neha, TCS, SQL) + (Neha, Infosys, Python) ⇒ table must contain (Neha, TCS, Python) and (Neha, Infosys, SQL) — the forced cross product:

| consultant | client | skill |
|------------|---------|--------|
| Neha | TCS | SQL |
| Neha | TCS | Python |
| Neha | Infosys | SQL |
| Neha | Infosys | Python |
| Arjun | Wipro | Java |

4. ✂️ **Decompose:** `CONSULTANT_CLIENT(consultant, client)` = {(Neha,TCS), (Neha,Infosys), (Arjun,Wipro)} and `CONSULTANT_SKILL(consultant, skill)` = {(Neha,SQL), (Neha,Python), (Arjun,Java)}.
5. ✅ **Lossless:** join on `consultant` rebuilds exactly the 5 rows ✅. **4NF achieved:** adding a new client = 1 row, not 2.

#### Q12. R(W, X, Y, Z), F = {W → X, X → W, Y → Z}. Find all candidate keys, the highest NF, and decompose to BCNF; verify losslessness and preservation.
**Answer (step-by-step):**
1. 🔍 **Keys:** W and X are equivalent (W→X, X→W). WY⁺ = {W,X,Y,Z} ⇒ **WY** key; XY⁺ = {X,W,Y,Z} ⇒ **XY** key; WZ⁺ = {W,X,Z} misses Y ⇒ not. Candidate keys: **WY, XY**; prime = {W, X, Y}; non-prime = {Z}.
2. ❌ **2NF:** Y → Z: Z is non-prime and Y is a **proper subset** of key WY (and XY) ⇒ **partial dependency** ⇒ not 2NF. **Highest NF = 1NF.**
3. 🔍 3NF also fails (Y not superkey, Z non-prime), BCNF too — but we decompose bottom-up anyway.
4. ✂️ **Decompose:** `R1(W, X)` (holds W→X, X→W), `R2(Y, Z)` (holds Y→Z), `R3(W, Y)` (keeps the key pair).
5. ✅ **Each table BCNF:** R1's keys W, X; R2's key Y; R3 has no non-trivial FDs ⇒ trivially BCNF.
6. ✅ **Lossless chain:** R1 ∩ R3 = {W}, W → X (all of R1) ✅; R2 ∩ R3 = {Y}, Y → Z (all of R2) ✅.
7. ✅ **Preserved:** W→X, X→W in R1; Y→Z in R2 — nothing lost 🎉.

🎓 **You made it!** Remember the story arc: *integrity rules keep data legal → FDs/MVDs describe the data's hidden logic → normal forms use that logic to rebuild the schema without anomalies.* Practice by finding keys first — every NF test flows from the keys. Good luck with INT306! 🚀
