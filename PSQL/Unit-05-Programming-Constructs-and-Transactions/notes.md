# Unit 5 — Programming Constructs & Transaction Processing

📌 **INT306 — Database Management Systems | PostgreSQL (PL/pgSQL) Edition**

A database is not just a place to *store* data — it can also **run programs** (Part A) and **protect concurrent work** (Part B). Part A teaches **PL/pgSQL**, PostgreSQL's procedural language (the direct equivalent of Oracle's PL/SQL). Part B covers **transaction theory**: ACID, schedules, serializability, locking, and recovery — favourite exam territory.

✅ Every example below runs as-is in `psql` or pgAdmin (PostgreSQL 11 or newer). Re-run the Section 1 block any time to reset the sample data.

## 🎯 What You'll Learn

- 🔹 How a PL/pgSQL block is structured (DECLARE / BEGIN / EXCEPTION / END) and how to run anonymous blocks with `DO`
- 🔹 All flow-control statements: IF/ELSIF/ELSE, CASE, LOOP, WHILE, FOR, RETURN
- 🔹 Writing functions (with `RETURNS`, `RETURNS TABLE`, `SETOF`) and stored procedures (with `CALL`, IN/OUT/INOUT, transaction control)
- 🔹 Cursors: processing rows one at a time
- 🔹 Triggers: automatic reactions to INSERT/UPDATE/DELETE
- 🔹 Exception handling: catching errors, raising your own
- 🔹 Transactions: ACID, states, schedules, serializability
- 🔹 Concurrency problems and their cures: locks, 2PL, isolation levels
- 🔹 Recovery: logs, deferred vs immediate update, checkpoints

## 📑 Table of Contents

**Part A — Programming Constructs**

1. [PL/pgSQL Blocks](#1-plpgsql-blocks)
2. [Flow Control Statements](#2-flow-control-statements)
3. [Functions](#3-functions)
4. [Stored Procedures](#4-stored-procedures)
5. [Cursors](#5-cursors)
6. [Triggers](#6-triggers)
7. [Exception Handling](#7-exception-handling)

**Part B — Database Transaction Processing**

8. [Transaction System Concepts](#8-transaction-system-concepts)
9. [ACID Properties](#9-acid-properties)
10. [Schedules](#10-schedules)
11. [Serializability](#11-serializability)
12. [Concurrency Control](#12-concurrency-control)
13. [Recoverability](#13-recoverability)

**Extras:** [⚡ Quick Revision](#-quick-revision) · [📝 Practice Set](#-practice-set)

---

# 🟢 PART A — Programming Constructs in Databases

## 1. PL/pgSQL Blocks

📌 A PL/pgSQL program is a **block**. Every block has the same skeleton:

```
[ DECLARE   ]  ← optional: variables, constants, cursors
BEGIN         ← mandatory: the executable statements
[ EXCEPTION ]  ← optional: error handlers
END;          ← block finishes here
```

Two ways to run a block:

- **Anonymous block** — written with `DO $$ ... $$;`, runs once, then is gone.
- **Named block** — saved permanently as a FUNCTION, PROCEDURE, or TRIGGER (Sections 3, 4, 6).

### 🔹 First: the reusable sample table (banking theme)

```sql
DROP TABLE IF EXISTS accounts;           -- remove any old copy so this script re-runs safely
CREATE TABLE accounts (                  -- the banking table used throughout this unit
  id       INT PRIMARY KEY,              -- unique account number
  holder   TEXT NOT NULL,                -- name of the account owner, never empty
  balance  NUMERIC(12,2) NOT NULL        -- money: 12 digits total, exactly 2 decimals
  CHECK (balance >= 0)                   -- rule: an account can never go below zero
);
INSERT INTO accounts VALUES              -- add the first sample row
  (1, 'Ayush', 10000.00);                -- account 1: Ayush holds 10,000
INSERT INTO accounts VALUES              -- add the second sample row
  (2, 'Bhavya', 5000.00);                -- account 2: Bhavya holds 5,000
INSERT INTO accounts VALUES              -- add the third sample row
  (3, 'Chirag', 7500.00);                -- account 3: Chirag holds 7,500
SELECT * FROM accounts;                  -- verify: should list 3 rows
```

**Line-by-line walkthrough**

- `DROP TABLE IF EXISTS` lets you re-run the whole block without a "relation already exists" error.
- `NUMERIC(12,2)` is the correct type for money — never use FLOAT for currency (rounding errors).
- The `CHECK (balance >= 0)` constraint is the database's own safety net; triggers (Section 6) add smarter rules on top.
- Expected output of the `SELECT`: 3 rows — (1, Ayush, 10000.00), (2, Bhavya, 5000.00), (3, Chirag, 7500.00).

### 🔹 Anonymous block with variables, %TYPE, %ROWTYPE, CONSTANT

```sql
DO $$                                              -- anonymous block: runs immediately, nothing saved
DECLARE                                            -- declaration section (optional)
  greeting  TEXT              := 'Hello, DBMS!';   -- ordinary variable with an initial value
  acc_id    accounts.id%TYPE  := 1;                -- %TYPE: same type as the accounts.id column
  acc_row   accounts%ROWTYPE;                      -- %ROWTYPE: holds one complete row of accounts
  bank_name CONSTANT TEXT     := 'INT306-Bank';    -- CONSTANT: value can never be changed later
BEGIN                                              -- executable section (mandatory)
  SELECT * INTO acc_row                            -- copy the matching row into acc_row
  FROM accounts WHERE id = acc_id;                 -- look up the account whose id = 1
  RAISE NOTICE '% greets you: %', bank_name, greeting;              -- print a message
  RAISE NOTICE 'Account % (%): balance = %',                        -- print row details nicely
               acc_row.id, acc_row.holder, acc_row.balance;         -- fields accessed with dot
END $$;                                            -- end of the anonymous block
```

**Line-by-line walkthrough**

- `$$ ... $$` is *dollar quoting* — the whole PL/pgSQL body is one string, so inner quotes need no escaping.
- `%TYPE` ties the variable's type to the column: if the column type changes, the code still compiles.
- `%ROWTYPE` stores a whole row; access fields with dot notation (`acc_row.balance`).
- `CONSTANT` fails at compile time if you try to assign to it later — good for fixed values.
- `RAISE NOTICE` prints to the client without aborting anything (like `System.out.println` with a prefix).
- Expected output:
  `NOTICE:  INT306-Bank greets you: Hello, DBMS!`
  `NOTICE:  Account 1 (Ayush): balance = 10000.00`

### 🎯 Questions

**Q1. Name the sections of a PL/pgSQL block and say which are optional.**
✅ **Answer:** DECLARE (optional — variables/constants/cursors), BEGIN (mandatory — statements), EXCEPTION (optional — error handlers), END (mandatory closer). Only BEGIN…END is required; DECLARE is omitted when there is nothing to declare, and EXCEPTION when no error handling is needed.

**Q2. Differentiate %TYPE and %ROWTYPE with an example.**
✅ **Answer:** `%TYPE` gives one variable the data type of one column (`v accounts.balance%TYPE`). `%ROWTYPE` gives one variable the composite type of a whole table row (`r accounts%ROWTYPE`), letting you `SELECT * INTO r` and read `r.balance`. Use %TYPE for single values, %ROWTYPE for row-at-a-time processing.

**Q3. How do you run an anonymous PL/pgSQL block in PostgreSQL?**
✅ **Answer:** With the `DO` statement: `DO $$ DECLARE ... BEGIN ... END $$;`. It executes immediately and nothing is stored in the database. (Oracle's equivalent is an anonymous `DECLARE ... BEGIN ... END; /` block in SQL*Plus.)

---

## 2. Flow Control Statements

📌 PL/pgSQL offers the same control structures as any programming language — but note they live **inside** blocks, not as top-level SQL.

### 🔹 IF / ELSIF / ELSE

```sql
DO $$                                                    -- anonymous block wrapper
DECLARE                                                  -- declaration section
  bal NUMERIC;                                           -- will hold the account balance
BEGIN                                                    -- executable section
  SELECT balance INTO bal FROM accounts WHERE id = 1;    -- fetch balance of account 1
  IF bal > 8000 THEN                                     -- first condition
    RAISE NOTICE 'Premium customer';                     -- runs when bal > 8000
  ELSIF bal > 5000 THEN                                  -- checked only if the first failed
    RAISE NOTICE 'Regular customer';                     -- runs when 5000 < bal <= 8000
  ELSE                                                   -- every condition failed
    RAISE NOTICE 'Basic customer';                       -- fallback branch
  END IF;                                                -- closes the IF statement
END $$;                                                  -- end of block
```

**Line-by-line walkthrough**

- Conditions are checked top-down and **only the first true branch runs**.
- `ELSIF` is spelled without the second E (unlike most languages' `elseif`).
- With fresh data, `bal = 10000` → first branch fires.
- Expected output: `NOTICE:  Premium customer`

### 🔹 CASE (statement form)

```sql
DO $$                                                    -- anonymous block wrapper
DECLARE                                                  -- declaration section
  grade CHAR(1) := 'A';                                  -- value to classify
BEGIN                                                    -- executable section
  CASE grade                                             -- test grade against the WHEN values
    WHEN 'A' THEN RAISE NOTICE 'Excellent';              -- matches grade = 'A'
    WHEN 'B' THEN RAISE NOTICE 'Good';                   -- matches grade = 'B'
    ELSE RAISE NOTICE 'Keep trying';                     -- no WHEN matched
  END CASE;                                              -- closes the CASE
END $$;                                                  -- end of block
```

**Line-by-line walkthrough**

- This is the *simple CASE* (value form). A *searched CASE* uses `CASE WHEN cond THEN ...`.
- ⚠️ Without an `ELSE`, a no-match value raises a `CASE_NOT_FOUND` error — always add ELSE or be sure of the domain.
- Expected output: `NOTICE:  Excellent`

### 🔹 Basic LOOP + EXIT

```sql
DO $$                                                    -- anonymous block wrapper
DECLARE                                                  -- declaration section
  i     INT := 1;                                        -- loop counter
  total INT := 0;                                        -- running sum
BEGIN                                                    -- executable section
  LOOP                                                   -- unconditional (infinite) loop
    total := total + i;                                  -- add current i into the sum
    i := i + 1;                                          -- increase the counter
    EXIT WHEN i > 5;                                     -- leave the loop when i becomes 6
  END LOOP;                                              -- closes the LOOP
  RAISE NOTICE 'Sum 1..5 = %', total;                    -- show the result
END $$;                                                  -- end of block
```

**Line-by-line walkthrough**

- `LOOP` repeats forever until an `EXIT` fires — you must provide the exit condition yourself.
- `EXIT WHEN cond` is the idiomatic place to test; you may also use plain `IF ... THEN EXIT; END IF;`.
- `:=` is assignment in PL/pgSQL (not `=`).
- Expected output: `NOTICE:  Sum 1..5 = 15`

### 🔹 WHILE loop

```sql
DO $$                                                    -- anonymous block wrapper
DECLARE                                                  -- declaration section
  i INT := 1;                                            -- counter starts at 1
BEGIN                                                    -- executable section
  WHILE i <= 3 LOOP                                      -- condition tested BEFORE each pass
    RAISE NOTICE 'i = %', i;                             -- body runs only while condition true
    i := i + 1;                                          -- must progress, or loop never ends
  END LOOP;                                              -- closes the WHILE
END $$;                                                  -- end of block
```

**Line-by-line walkthrough**

- `WHILE` = pre-test loop; if the condition is false initially, the body never runs.
- Forgetting `i := i + 1` gives an infinite loop.
- Expected output: three lines `NOTICE:  i = 1`, `i = 2`, `i = 3`.

### 🔹 FOR loop — integer range

```sql
DO $$                                                    -- anonymous block wrapper
DECLARE                                                  -- declaration section
  s INT := 0;                                            -- accumulator for the sum
BEGIN                                                    -- executable section
  FOR i IN 1..5 LOOP                                     -- i automatically takes 1,2,3,4,5
    s := s + i;                                          -- add this pass's value
  END LOOP;                                              -- closes the FOR
  RAISE NOTICE 'Sum = %', s;                             -- print the total
END $$;                                                  -- end of block
-- variants: FOR i IN REVERSE 5..1 LOOP ...  (count down)
--           FOR i IN 1..10 BY 2 LOOP ...    (step by 2)
```

**Line-by-line walkthrough**

- The loop variable `i` is **declared automatically** as INTEGER — do not declare it in DECLARE.
- `i` is read-only inside the loop; the bounds are evaluated once at loop entry.
- `REVERSE` and `BY` give countdowns and steps for free.
- Expected output: `NOTICE:  Sum = 15`

### 🔹 FOR loop — over a query (query loop)

```sql
DO $$                                                    -- anonymous block wrapper
DECLARE                                                  -- declaration section
  total NUMERIC := 0;                                    -- will hold the sum of balances
BEGIN                                                    -- executable section
  FOR rec IN SELECT id, balance FROM accounts LOOP       -- runs once per returned row
    RAISE NOTICE 'Account % has %', rec.id, rec.balance; -- rec is one row (RECORD)
    total := total + rec.balance;                        -- accumulate this row's balance
  END LOOP;                                              -- ends automatically when rows finish
  RAISE NOTICE 'Total money = %', total;                 -- print the grand total
END $$;                                                  -- end of block
```

**Line-by-line walkthrough**

- `rec` is **auto-declared** as RECORD; its fields follow the SELECT list.
- This is the cleanest way to walk a result set — no explicit cursor needed (Section 5 covers the manual style).
- Open, fetch-loop, and close are all handled by the FOR construct itself.
- Expected output: notices for accounts 1, 2, 3, then `NOTICE:  Total money = 22500.00`.

### 🔹 RETURN (exits a function with a value)

```sql
CREATE OR REPLACE FUNCTION square(n NUMERIC)             -- function with one parameter
RETURNS NUMERIC AS $$                                    -- promise: one numeric value back
BEGIN                                                    -- executable section
  RETURN n * n;                                          -- RETURN sends the value to the caller
END;                                                     -- end of the function body
$$ LANGUAGE plpgsql;                                     -- declare the language

SELECT square(6) AS result;                              -- call it: result is 36
```

**Line-by-line walkthrough**

- `RETURN` both computes the result **and** immediately leaves the function.
- In a `RETURNS`-void/SETOF function, a bare `RETURN;` just exits early.
- `CREATE OR REPLACE` updates the definition without dropping it first.
- Expected output: one row, `result = 36`.

### 🎯 Questions

**Q1. What is the difference between `EXIT WHEN` and `CONTINUE WHEN`?**
✅ **Answer:** `EXIT WHEN cond` leaves the loop entirely when cond is true; `CONTINUE WHEN cond` skips only the *rest of the current iteration* and jumps to the next pass of the same loop.

**Q2. Why is `ELSIF` a common exam trap?**
✅ **Answer:** Because PL/pgSQL spells it `ELSIF` (one E), while most languages use `elseif`/`else if`. Writing `ELSEIF` is a syntax error.

**Q3. When is a query-FOR loop better than a WHILE loop?**
✅ **Answer:** When you must visit each row of a result exactly once: the query-FOR loop opens the query, fetches row by row into `rec`, and closes it automatically — no manual counter, no open/close, no off-by-one bugs.

---

## 3. Functions

📌 A function is a **named, saved block** that takes parameters and returns a value or a result set. It is called from ordinary SQL — inside SELECT, WHERE, INSERT, etc.

### 🔹 Basic function with IN parameters

```sql
CREATE OR REPLACE FUNCTION add_interest(acc_id INT, rate NUMERIC)  -- two IN parameters (IN is the default)
RETURNS NUMERIC AS $$                                              -- promise: return one NUMERIC
DECLARE                                                            -- declaration section
  bal NUMERIC;                                                     -- temporary copy of the balance
BEGIN                                                              -- executable section
  SELECT balance INTO bal                                          -- read the current balance
  FROM accounts WHERE id = acc_id;                                 -- for the requested account
  bal := bal + (bal * rate / 100);                                 -- compute balance + interest
  RETURN bal;                                                      -- hand the new balance back
END;                                                               -- end of the function body
$$ LANGUAGE plpgsql;                                               -- the body language is PL/pgSQL
```

**Line-by-line walkthrough**

- `IN` is the default parameter mode — writing `IN acc_id INT` would mean the same.
- The parameter name `acc_id` is usable like a variable inside the body.
- `LANGUAGE plpgsql` is required; PostgreSQL also supports `sql`, `python`, etc.
- Expected output (called below): `10500.00` for account 1 at 5%.

### 🔹 Calling a function with SELECT

```sql
SELECT add_interest(1, 5) AS new_balance;   -- simple call: one row, one column (10500.00)
SELECT add_interest(id, 3) FROM accounts;   -- called once per row: 3 result rows
```

**Line-by-line walkthrough**

- Scalar functions plug straight into SELECT lists, WHERE clauses, and even CHECK constraints.
- The second call demonstrates a *set-call*: the function runs once for every table row.
- Functions **cannot** run COMMIT/ROLLBACK — they always execute inside the caller's transaction (see Section 4 for why procedures differ).

### 🔹 Function returning TABLE

```sql
CREATE OR REPLACE FUNCTION rich_accounts(min_bal NUMERIC)   -- one parameter: the cutoff
RETURNS TABLE(acc_id INT, acc_holder TEXT, acc_bal NUMERIC) -- result columns (act like OUT variables)
AS $$                                                       -- body starts
BEGIN                                                       -- executable section
  RETURN QUERY                                              -- return a WHOLE result set at once
    SELECT a.id, a.holder, a.balance                        -- the actual query
    FROM accounts a                                         -- alias 'a' avoids name clashes
    WHERE a.balance >= min_bal                              -- filter using the parameter
    ORDER BY a.balance DESC;                                -- richest account first
END;                                                        -- end of body
$$ LANGUAGE plpgsql;                                        -- language declaration

SELECT * FROM rich_accounts(6000);                          -- use it like a table
```

**Line-by-line walkthrough**

- The columns listed in `RETURNS TABLE(...)` become implicit OUT variables inside the body.
- ⚠️ Classic bug: if an OUT column were named `balance`, it would *shadow* the table column in the body — hence the names `acc_bal` and the alias `a`.
- `RETURN QUERY` appends an entire SELECT's rows to the result (you can call it several times to append more).
- Expected output (fresh data): rows (1, Ayush, 10000.00) and (3, Chirag, 7500.00).

### 🔹 Function returning SETOF (whole rows)

```sql
CREATE OR REPLACE FUNCTION accounts_of(p_holder TEXT)      -- search by owner name
RETURNS SETOF accounts AS $$                               -- returns whole accounts rows
BEGIN                                                      -- executable section
  RETURN QUERY SELECT * FROM accounts                      -- full rows matching the pattern
  WHERE holder ILIKE '%' || p_holder || '%';               -- case-insensitive contains-match
END;                                                       -- end of body
$$ LANGUAGE plpgsql;                                       -- language

SELECT * FROM accounts_of('ayush');                        -- finds Ayush's row
```

```sql
CREATE OR REPLACE FUNCTION countdown(n INT)                -- builds a set row by row
RETURNS SETOF INT AS $$                                    -- each element is one INT
BEGIN                                                      -- executable section
  WHILE n > 0 LOOP                                         -- walk down from n to 1
    RETURN NEXT n;                                         -- add ONE row (does NOT exit!)
    n := n - 1;                                            -- decrease
  END LOOP;                                                -- repeat until n = 0
END;                                                       -- end of body
$$ LANGUAGE plpgsql;                                       -- language

SELECT * FROM countdown(3);                                -- 3 rows: 3, 2, 1
```

**Line-by-line walkthrough**

- `RETURNS SETOF accounts` means "rows shaped like the accounts table" — callers can even use them in joins.
- `RETURN QUERY` = append a whole query result; `RETURN NEXT` = append **one** row, then continue.
- A bare `RETURN` (no value) ends a SETOF function early.
- Expected output: `accounts_of('ayush')` → 1 row; `countdown(3)` → 3 rows, values 3, 2, 1.

### 🎯 Questions

**Q1. What is the difference between RETURN, RETURN NEXT, and RETURN QUERY?**
✅ **Answer:** `RETURN expr` finishes a scalar function and returns one value. `RETURN NEXT row` appends a single row to a SETOF/TABLE result and keeps executing. `RETURN QUERY select...` appends the entire result of a query and keeps executing. In SETOF/TABLE functions, a final bare `RETURN` (or falling off the end) terminates the function.

**Q2. Why can a function not be called with CALL, and a procedure not with SELECT?**
✅ **Answer:** Functions return a value and are *expressions* — they belong in queries (`SELECT f()`). Procedures return nothing (their results leave via OUT parameters) and are *statements* — they are invoked with `CALL proc(...)`.

**Q3. What does `RETURNS TABLE(a INT, b TEXT)` really create inside the function body?**
✅ **Answer:** It creates implicit OUT variables named `a` and `b`. Any unqualified name in the body that matches them resolves to the variable, not the table column — so qualify columns (e.g., `t.a`) or choose non-clashing names.

---

## 4. Stored Procedures

📌 A procedure is a named block **called as a statement** with `CALL`. Unlike functions, procedures (PostgreSQL 11+) may **commit or roll back the transaction** — perfect for multi-step business operations like money transfers.

### 🔹 Procedure with transaction control

```sql
CREATE OR REPLACE PROCEDURE transfer_money(                 -- a reusable banking operation
  sender_id   INT,                                          -- IN (default): account to debit
  receiver_id INT,                                          -- IN: account to credit
  amount      NUMERIC)                                      -- IN: how much to move
LANGUAGE plpgsql                                            -- procedures declare a language too
AS $$                                                       -- body starts
DECLARE                                                     -- declaration section
  new_bal NUMERIC;                                          -- sender balance after the debit
BEGIN                                                       -- executable section
  UPDATE accounts SET balance = balance - amount            -- step 1: debit the sender
  WHERE id = sender_id;                                     -- only that account
  SELECT balance INTO new_bal                               -- step 2: re-read sender balance
  FROM accounts WHERE id = sender_id;                       -- (sees our own update)
  IF new_bal < 0 THEN                                       -- would the account go negative?
    ROLLBACK;                                               -- undo step 1 (and anything earlier)
    RAISE NOTICE 'Insufficient funds - transfer rolled back';  -- tell the caller why
    RETURN;                                                 -- leave the procedure early
  END IF;                                                   -- end of the guard
  UPDATE accounts SET balance = balance + amount            -- step 3: credit the receiver
  WHERE id = receiver_id;                                   -- only that account
  COMMIT;                                                   -- make debit+credit permanent together
END;                                                        -- end of body
$$;                                                         -- end of the body string

CALL transfer_money(1, 2, 2500);                            -- run it (CALL, not SELECT!)
```

**Line-by-line walkthrough**

- `CALL` is the only way to invoke a procedure — `SELECT transfer_money(...)` is a syntax error.
- `ROLLBACK`/`COMMIT` inside a procedure body are legal **only when called from a top-level context** (psql/pgAdmin directly). Calling it inside an explicit `BEGIN ... COMMIT` block raises `invalid transaction termination`.
- Either *both* updates stick, or *neither* does — that is transactional thinking (Part B) in action.
- Expected output: on success, `CALL` shows no rows; balances become A1 = 7500.00, A2 = 7500.00. With `amount = 999999` you would see the insufficient-funds notice and **no** balance change.

### 🔹 IN / OUT / INOUT parameters

```sql
CREATE OR REPLACE PROCEDURE account_stats(                  -- summary of richer accounts
  IN  p_min   NUMERIC,                                      -- IN: input cutoff value
  OUT p_total NUMERIC,                                      -- OUT: sum of matching balances
  OUT p_count BIGINT)                                       -- OUT: how many accounts matched
LANGUAGE plpgsql                                            -- language
AS $$                                                       -- body starts
BEGIN                                                       -- executable section
  SELECT COALESCE(SUM(balance), 0), COUNT(*)                -- compute both results in one query
  INTO p_total, p_count                                     -- store into the OUT parameters
  FROM accounts WHERE balance >= p_min;                     -- filter by the IN parameter
END;                                                        -- end of body
$$;                                                         -- end of body string

CALL account_stats(6000, NULL, NULL);                       -- NULLs are placeholders for OUTs
```

**Line-by-line walkthrough**

- `IN` = value goes in; `OUT` = value comes out; `INOUT` = both directions in one parameter.
- The caller must pass a placeholder (`NULL`) for each OUT parameter; the results come back as a one-row result of the CALL.
- Expected output: one row with columns `p_total | p_count` → `17500.00 | 2` (fresh data: 10000 and 7500 are ≥ 6000; Bhavya's 5000 is excluded).

### 🔹 Function vs Procedure — comparison table

| Feature | Function | Procedure (PG 11+) |
|---|---|---|
| Invocation | `SELECT f(x)` — inside queries | `CALL p(x)` — as a statement |
| Return value | Mandatory (`RETURNS`) | None (results via OUT/INOUT) |
| Usable in SELECT/WHERE | ✅ Yes | ❌ No |
| COMMIT / ROLLBACK inside | ❌ Never | ✅ Yes (top-level call) |
| Typical use | Compute a value (tax, discount) | Perform an operation (transfer) |
| In Oracle terms | `CREATE FUNCTION` | `CREATE PROCEDURE` |

### 🎯 Questions

**Q1. Why do functions disallow transaction control while procedures allow it?**
✅ **Answer:** A function executes *within* the caller's statement/transaction — the caller decides the fate of the transaction, and committing mid-query would break SQL semantics (e.g., a function in a WHERE clause firing per row). A procedure is a standalone statement, so PostgreSQL lets it manage transaction boundaries itself.

**Q2. Give three real differences between a function and a procedure in PostgreSQL.**
✅ **Answer:** (1) Functions return a value and can appear inside SQL expressions; procedures return nothing and are invoked with CALL. (2) Procedures may execute COMMIT/ROLLBACK; functions never may. (3) Functions are typically used for computations and can be used in indexes/constraints; procedures encapsulate multi-step business operations.

**Q3. What happens if you call `transfer_money` inside an explicit `BEGIN; ... ` transaction block?**
✅ **Answer:** The `COMMIT`/`ROLLBACK` statements inside the procedure raise `ERROR: invalid transaction termination` — transaction control in procedures only works when the procedure is called from the top level (e.g., plain `CALL` in psql).

---

## 5. Cursors

📌 A **cursor** is a pointer that walks through a query's result **one row at a time**. Use one when you must process each row individually (row-by-row logic, per-row checks, conditional updates) instead of in one set-based statement.

### 🔹 Cursor lifecycle: DECLARE → OPEN → FETCH → CLOSE

```sql
DO $$                                                       -- anonymous block wrapper
DECLARE                                                     -- declaration section
  cur CURSOR FOR                                            -- declare a named cursor
    SELECT id, holder, balance FROM accounts ORDER BY id;   -- the query it will walk through
  v_id   accounts.id%TYPE;                                  -- variable: account number
  v_name accounts.holder%TYPE;                              -- variable: holder name
  v_bal  accounts.balance%TYPE;                             -- variable: balance
BEGIN                                                       -- executable section
  OPEN cur;                                                 -- run the query, park before first row
  LOOP                                                      -- walk the rows
    FETCH cur INTO v_id, v_name, v_bal;                     -- pull the next row into variables
    EXIT WHEN NOT FOUND;                                    -- NOT FOUND becomes true after the last row
    RAISE NOTICE 'Acc % (%): %', v_id, v_name, v_bal;       -- process this one row
  END LOOP;                                                 -- repeat until exhausted
  CLOSE cur;                                                -- release the cursor's resources
END $$;                                                     -- end of block
```

**Line-by-line walkthrough**

- `OPEN` executes the query but does not give you a row yet — the cursor sits *before* the first row.
- `FETCH` advances one row and copies its columns into your variables.
- After a FETCH that finds nothing, the special variable `NOT FOUND` is true — that is the standard exit condition.
- `CLOSE` frees the cursor; you could `OPEN` it again for a fresh walk.
- Expected output (fresh data): three notices — Acc 1 (Ayush): 10000.00, Acc 2 (Bhavya): 5000.00, Acc 3 (Chirag): 7500.00.

### 🔹 Cursor status in PL/pgSQL (vs Oracle's % attributes)

| PL/pgSQL | Oracle equivalent | Meaning |
|---|---|---|
| `FOUND` | `cursor%FOUND` | last FETCH/statement affected ≥ 1 row |
| `NOT FOUND` | `cursor%NOTFOUND` | last FETCH found nothing |
| `ROW_COUNT` (via GET DIAGNOSTICS) | `cursor%ROWCOUNT` | rows touched by the last statement |
| (no direct equivalent) | `cursor%ISOPEN` | check with exception handling or code discipline |

📌 PL/pgSQL also accepts Oracle-style `cur%FOUND`, `cur%NOTFOUND`, `cur%ROWCOUNT` on named cursors for compatibility.

### 🔹 The easy way: cursor FOR loop

```sql
DO $$                                                       -- anonymous block wrapper
BEGIN                                                       -- executable section
  FOR rec IN SELECT id, holder, balance                     -- rec is auto-declared per row
             FROM accounts WHERE balance < 8000 LOOP        -- only the poorer accounts
    RAISE NOTICE 'Low balance: % (%) has %',                -- print a per-row warning
                 rec.id, rec.holder, rec.balance;           -- fields of the current row
  END LOOP;                                                 -- auto-opens, auto-fetches, auto-closes
END $$;                                                     -- end of block
```

**Line-by-line walkthrough**

- The FOR-loop-over-query *is* an implicit cursor: declare/open/fetch/close are all automatic.
- `rec` needs no DECLARE — PL/pgSQL creates it as RECORD.
- Prefer this over manual cursors unless you need OPEN/FETCH control (e.g., skipping rows, resuming).
- Expected output (fresh data): notices for Acc 2 (Bhavya, 5000.00) and Acc 3 (Chirag, 7500.00).

### 🎯 Questions

**Q1. Why do we need cursors at all when SQL is set-based?**
✅ **Answer:** Most operations are best done set-at-a-time, but some logic is inherently row-by-row: validating each row against an external rule, sending one message per row, or updating row-dependent values. A cursor gives you controlled, one-row-at-a-time access to a result set.

**Q2. What is the meaning of FOUND and NOT FOUND after a FETCH?**
✅ **Answer:** `FOUND` is true if the last FETCH returned a row; `NOT FOUND` is its negation. The idiom `EXIT WHEN NOT FOUND;` right after FETCH is the standard way to stop at end-of-rows. (In Oracle this is `cursor%FOUND` / `cursor%NOTFOUND`.)

**Q3. Order the cursor lifecycle steps and explain each briefly.**
✅ **Answer:** (1) DECLARE — associate a name with a query; (2) OPEN — execute the query, position before the first row; (3) FETCH — retrieve the next row into variables, repeatedly until NOT FOUND; (4) CLOSE — release the cursor. A cursor FOR loop automates all four.

---

## 6. Triggers

📌 A **trigger** is a stored action that fires **automatically** when rows are inserted, updated, or deleted. Two parts:

1. **Trigger function** — `RETURNS TRIGGER`, has no arguments, and can read the special records `NEW` (the new row) and `OLD` (the old row).
2. **CREATE TRIGGER** — binds the function to a table with a *timing* (BEFORE/AFTER) and *events* (INSERT/UPDATE/DELETE).

| Event | OLD contains | NEW contains |
|---|---|---|
| INSERT | — (NULL) | the row being inserted |
| UPDATE | the row before the change | the row after the change |
| DELETE | the row being deleted | — (NULL) |

| Timing | Fires | Typical use |
|---|---|---|
| BEFORE | before the row is saved | validate / reject / modify NEW |
| AFTER | after the row is saved | audit logs, cascading side effects |
| INSTEAD OF | on views | redirect writes to base tables |

Useful trigger variables: `TG_OP` (INSERT/UPDATE/DELETE), `TG_NAME`, `TG_TABLE_NAME`, `TG_WHEN`, `TG_LEVEL`.

### 🔹 Example 1 — audit log trigger

```sql
DROP TABLE IF EXISTS accounts_audit;                        -- start clean on re-runs
CREATE TABLE accounts_audit (                               -- one row per change to accounts
  audit_id    BIGSERIAL PRIMARY KEY,                        -- auto-numbered audit entry
  account_id  INT,                                          -- which account changed
  operation   TEXT,                                         -- INSERT / UPDATE / DELETE (TG_OP)
  old_balance NUMERIC,                                      -- balance before (NULL for INSERT)
  new_balance NUMERIC,                                      -- balance after (NULL for DELETE)
  changed_by  TEXT DEFAULT current_user,                    -- who ran the statement
  changed_at  TIMESTAMPTZ DEFAULT now()                     -- when it happened
);

CREATE OR REPLACE FUNCTION log_account_changes()            -- trigger functions take NO arguments
RETURNS TRIGGER AS $$                                       -- must promise to return TRIGGER
BEGIN                                                       -- executable section
  IF TG_OP = 'DELETE' THEN                                  -- DELETE has no NEW record at all!
    INSERT INTO accounts_audit(account_id, operation, old_balance)  -- log the old state only
    VALUES (OLD.id, TG_OP, OLD.balance);                    -- capture what is disappearing
    RETURN OLD;                                             -- convention: return OLD for DELETE
  END IF;                                                   -- end of the DELETE branch
  INSERT INTO accounts_audit(account_id, operation, old_balance, new_balance)  -- full record
  VALUES (NEW.id, TG_OP, OLD.balance, NEW.balance);         -- for INSERT, OLD.* is simply NULL
  RETURN NEW;                                               -- convention: return NEW otherwise
END;                                                        -- end of function body
$$ LANGUAGE plpgsql;                                        -- language

DROP TRIGGER IF EXISTS trg_audit_accounts ON accounts;      -- safe to re-run
CREATE TRIGGER trg_audit_accounts                           -- name of the trigger
AFTER INSERT OR UPDATE OR DELETE ON accounts                -- timing + all three events
FOR EACH ROW                                                -- row-level: fires once per changed row
EXECUTE FUNCTION log_account_changes();                     -- what to run (PG 11+; older: EXECUTE PROCEDURE)

UPDATE accounts SET balance = balance + 1000 WHERE id = 2;  -- test: fires the trigger once
SELECT * FROM accounts_audit;                               -- inspect the audit trail
```

**Line-by-line walkthrough**

- `AFTER ... FOR EACH ROW` = statement-level trigger (omit `FOR EACH ROW` → `FOR EACH STATEMENT`, fires once per statement).
- One trigger function can serve all three events — `TG_OP` tells you which fired.
- ⚠️ Referencing `NEW.balance` during DELETE would be NULL, so the DELETE branch logs only OLD.
- `RETURN NEW` (BEFORE triggers) lets the row proceed; returning NULL would silently cancel the operation.
- Expected output: the SELECT shows one row — operation `UPDATE`, old 5000.00, new 6000.00, your user, today's timestamp.

### 🔹 Example 2 — prevent-negative-salary trigger

```sql
DROP TABLE IF EXISTS employees;                             -- fresh table for this example
CREATE TABLE employees (                                    -- small HR table
  emp_id INT PRIMARY KEY,                                   -- employee number
  name   TEXT NOT NULL,                                     -- employee name
  salary NUMERIC(10,2)                                      -- monthly salary to be guarded
);

CREATE OR REPLACE FUNCTION check_salary()                   -- validator trigger function
RETURNS TRIGGER AS $$                                       -- trigger function signature
BEGIN                                                       -- executable section
  IF NEW.salary < 0 THEN                                    -- our business rule
    RAISE EXCEPTION 'Salary cannot be negative (got %)', NEW.salary   -- reject with clear message
      USING ERRCODE = 'P0001';                              -- custom SQLSTATE (see Section 7)
  END IF;                                                   -- end of the rule
  RETURN NEW;                                               -- accept the row as-is
END;                                                        -- end of body
$$ LANGUAGE plpgsql;                                        -- language

DROP TRIGGER IF EXISTS trg_check_salary ON employees;       -- safe to re-run
CREATE TRIGGER trg_check_salary                             -- the enforcing trigger
BEFORE INSERT OR UPDATE ON employees                        -- BEFORE: can reject before saving
FOR EACH ROW                                                -- per changed row
EXECUTE FUNCTION check_salary();                            -- run the validator

INSERT INTO employees VALUES (1, 'Riya', 40000);            -- works fine
INSERT INTO employees VALUES (2, 'Om', -500);               -- ERROR: Salary cannot be negative (got -500)
```

**Line-by-line walkthrough**

- A **BEFORE** trigger can cancel an operation: raise an exception (statement aborts, transaction rolls back) or `RETURN NULL`.
- AFTER triggers cannot cancel — by then the row is already written (perfect for auditing instead).
- The error message shows the offending value via the `%` placeholder — much friendlier than a bare constraint error.
- Expected output: first INSERT succeeds (`INSERT 0 1`); second INSERT fails with the custom error.

### 🎯 Questions

**Q1. When do you choose a BEFORE trigger vs an AFTER trigger?**
✅ **Answer:** Use BEFORE to validate or modify the row before it is stored (you can reject or rewrite NEW). Use AFTER when the row must already exist — audit logs, maintaining summary tables, notifications. A rule of thumb: BEFORE = guard, AFTER = record/react.

**Q2. In which events are OLD and NEW available?**
✅ **Answer:** NEW in INSERT and UPDATE; OLD in UPDATE and DELETE. Neither exists in the other's event (DELETE has no NEW; INSERT has no OLD). Trigger functions for mixed events must check `TG_OP` before touching OLD/NEW.

**Q3. Can you call a trigger function directly with SELECT?**
✅ **Answer:** No. A function declared `RETURNS TRIGGER` is only meaningful as a trigger callback — PostgreSQL rejects calling it directly because NEW/OLD/TG_* are supplied by the trigger machinery, not by you.

---

## 7. Exception Handling

📌 Errors inside a block can be caught in an **EXCEPTION** section. If an error occurs in the BEGIN part, execution jumps to the matching `WHEN` clause; after the handler runs, the block finishes normally.

```
BEGIN
  ...statements...          -- if any statement fails
EXCEPTION
  WHEN error_kind THEN      -- ...control jumps here
    ...recovery...
END
```

### 🔹 Catching predefined errors (STRICT SELECT + unique_violation)

```sql
DO $$                                                       -- anonymous block wrapper
DECLARE                                                     -- declaration section
  v_name TEXT;                                              -- will receive the holder name
BEGIN                                                       -- executable section
  SELECT holder INTO STRICT v_name                          -- STRICT: require EXACTLY one row
  FROM accounts WHERE id = 99;                              -- id 99 does not exist...
  RAISE NOTICE 'Found %', v_name;                           -- ...so this never runs
EXCEPTION                                                   -- handler section
  WHEN NO_DATA_FOUND THEN                                   -- STRICT select found 0 rows
    RAISE NOTICE 'No account with that id';                 -- graceful recovery
  WHEN TOO_MANY_ROWS THEN                                   -- STRICT select found 2+ rows
    RAISE NOTICE 'More than one match';                     -- graceful recovery
END $$;                                                     -- end of block
```

```sql
DO $$                                                       -- anonymous block wrapper
BEGIN                                                       -- executable section
  INSERT INTO accounts VALUES (1, 'Duplicate', 999);        -- id 1 already exists!
EXCEPTION                                                   -- handler section
  WHEN unique_violation THEN                                -- SQLSTATE 23505: duplicate key
    RAISE NOTICE 'Account 1 already exists - insert skipped';  -- friendly message
  WHEN OTHERS THEN                                          -- catch-all (must be last)
    RAISE NOTICE 'Unexpected error: %', SQLERRM;            -- SQLERRM holds the error text
END $$;                                                     -- end of block
```

**Line-by-line walkthrough**

- Condition names map to SQLSTATEs: `no_data_found` = P0002, `too_many_rows` = P0003, `unique_violation` = 23505.
- Without `STRICT`, a SELECT INTO with 0 rows just sets NULLs (no error) and multiple rows take the first — STRICT turns both into catchable errors.
- `WHEN OTHERS` catches everything not matched earlier — always place it last.
- An unhandled error rolls back the whole block's work; a handled one does not.
- Expected output: first block → `NOTICE:  No account with that id`; second block → `NOTICE:  Account 1 already exists - insert skipped`.

### 🔹 RAISE levels at a glance

| Form | Effect |
|---|---|
| `RAISE DEBUG / LOG / INFO` | low-priority messages, mostly server logs |
| `RAISE NOTICE '...'` | message to the client, execution continues |
| `RAISE WARNING '...'` | like NOTICE, flagged as a warning |
| `RAISE EXCEPTION '...'` | aborts the block/transaction — catchable in EXCEPTION |

### 🔹 RAISE_APPLICATION_ERROR equivalent (custom message + SQLSTATE)

📌 Oracle: `RAISE_APPLICATION_ERROR(-20001, 'msg')`. PostgreSQL: `RAISE EXCEPTION ... USING ERRCODE = 'XXXXX'` with any 5-character SQLSTATE (P-codes are PL/pgSQL's user space).

```sql
CREATE OR REPLACE FUNCTION withdraw(acc_id INT, amount NUMERIC)  -- safe withdrawal
RETURNS NUMERIC AS $$                                            -- returns the new balance
DECLARE                                                          -- declaration section
  v_bal NUMERIC;                                                 -- current balance
BEGIN                                                            -- executable section
  SELECT balance INTO v_bal                                      -- read the balance
  FROM accounts WHERE id = acc_id;                               -- for this account
  IF v_bal < amount THEN                                         -- not enough money?
    RAISE EXCEPTION 'Insufficient funds: balance %, needed %', v_bal, amount  -- our message
      USING ERRCODE = 'P0001',                                   -- custom SQLSTATE (like ORA-20001)
            HINT = 'Try a smaller amount';                       -- extra hint for the client
  END IF;                                                        -- end of the guard
  UPDATE accounts SET balance = v_bal - amount                   -- perform the debit
  WHERE id = acc_id;                                             -- for this account
  RETURN v_bal - amount;                                         -- report the new balance
END;                                                             -- end of body
$$ LANGUAGE plpgsql;                                             -- language

SELECT withdraw(2, 999999);                                      -- ERROR + HINT: Try a smaller amount
SELECT withdraw(2, 1000);                                        -- works: returns 4000.00 (fresh data)
```

**Line-by-line walkthrough**

- `USING ERRCODE` sets the SQLSTATE your application can test for; `HINT` adds a second line to the error.
- The `%` placeholders are filled by the comma-separated expressions after the message string.
- This is exactly the Oracle `RAISE_APPLICATION_ERROR` pattern: a meaningful message plus a code the caller can catch.
- Expected output: the first call fails with `ERROR: Insufficient funds: balance 5000.00, needed 999999` + `HINT:`; the second returns `4000.00`.

### 🔹 User-defined exception via custom SQLSTATE

```sql
DO $$                                                       -- anonymous block wrapper
BEGIN                                                       -- executable section
  RAISE EXCEPTION 'Daily limit exceeded'                    -- raise our own error
    USING ERRCODE = 'P1234';                                -- with our own 5-char code
EXCEPTION                                                   -- handler section
  WHEN SQLSTATE 'P1234' THEN                                -- catch that exact code
    RAISE NOTICE 'Caught my own exception';                 -- proves the round-trip works
END $$;                                                     -- end of block
```

**Line-by-line walkthrough**

- PL/pgSQL needs no Oracle-style `PRAGMA EXCEPTION_INIT` — raise with an ERRCODE, catch with `WHEN SQLSTATE 'code'`.
- Picking a distinct code (like P1234) lets callers tell *your* errors apart from built-in ones.
- Expected output: `NOTICE:  Caught my own exception`.

### 🎯 Questions

**Q1. What is PostgreSQL's equivalent of Oracle's RAISE_APPLICATION_ERROR?**
✅ **Answer:** `RAISE EXCEPTION 'message' USING ERRCODE = 'XXXXX';` — it aborts the current operation with a custom message and a custom 5-character SQLSTATE (usually a P-code), which applications can detect exactly like Oracle's −20000..−20999 codes.

**Q2. What happens to the transaction if an exception is NOT handled?**
✅ **Answer:** The error propagates up, aborting the enclosing transaction — all of its uncommitted changes are rolled back. If it IS caught by an EXCEPTION handler, the sub-transaction's changes roll back but the outer transaction survives.

**Q3. Difference between RAISE NOTICE and RAISE EXCEPTION?**
✅ **Answer:** `RAISE NOTICE` is informational — it prints a message and execution continues. `RAISE EXCEPTION` aborts the current block/transaction, rolling back its uncommitted work; it can be caught with `WHEN ... THEN` or by the client application.

---

# 🔵 PART B — Database Transaction Processing

## 8. Transaction System Concepts

📌 A **transaction** is a logical unit of work: one or more SQL statements that must succeed or fail **as a whole**. Classic example: a money transfer = debit + credit. If only the debit survives a crash, money vanishes — so both steps must be atomic.

### 🔹 BEGIN / COMMIT / ROLLBACK in PostgreSQL

```sql
BEGIN;                                              -- start a transaction
UPDATE accounts SET balance = balance - 500 WHERE id = 1;   -- debit Ayush (not visible to others yet)
UPDATE accounts SET balance = balance + 500 WHERE id = 2;   -- credit Bhavya
COMMIT;                                             -- make BOTH changes permanent together
```

```sql
BEGIN;                                              -- start another transaction
UPDATE accounts SET balance = balance - 500 WHERE id = 1;   -- debit again
ROLLBACK;                                           -- change of mind: undo everything since BEGIN
```

```sql
BEGIN;                                              -- start a transaction
UPDATE accounts SET balance = 8000 WHERE id = 1;    -- change 1 (wanted)
SAVEPOINT sp1;                                      -- mark a safe point here
UPDATE accounts SET balance = 9999 WHERE id = 1;    -- change 2 (mistake!)
ROLLBACK TO sp1;                                    -- undo ONLY change 2
COMMIT;                                             -- keep change 1 permanently
```

**Line-by-line walkthrough**

- psql runs in *autocommit* — each statement is its own tiny transaction unless you issue `BEGIN` first.
- `COMMIT` = all changes become permanent and visible to everyone; `ROLLBACK` = all changes since BEGIN vanish.
- `SAVEPOINT` + `ROLLBACK TO` gives partial undo inside one transaction.
- Expected result: after block 1, A1 = 9500 and A2 = 5500 (fresh data); block 2 leaves balances untouched; block 3 leaves A1 = 8000.

### 🔹 States of a transaction

```
                 BEGIN
                   |
                   v
              ┌─────────┐   statement error      ┌──────────┐
              │ ACTIVE  │ ─────────────────────▶ │  FAILED  │ ──▶ ┌──────────┐
              └────┬────┘                        └──────────┘     │ ABORTED  │
                   │  last statement done,                       └──────────┘
                   │  COMMIT issued                              (rolled back,
                   v                                              data restored)
        ┌──────────────────────┐        failure during
        │ PARTIALLY COMMITTED  │ ─────────────────────▶ FAILED
        └──────────┬───────────┘
                   │ all changes safely on disk
                   v
             ┌───────────┐
             │ COMMITTED │  ──▶ TERMINATED (final: either COMMITTED or ABORTED)
             └───────────┘
```

| State | Meaning |
|---|---|
| Active | running, making changes (uncommitted) |
| Partially committed | last statement done, COMMIT issued, final writes being flushed |
| Committed | all changes permanent — success |
| Failed | normal execution can no longer continue (error/crash) |
| Aborted | rolled back; database restored to before the transaction started |

### 🎯 Questions

**Q1. Define a transaction with an example.**
✅ **Answer:** A transaction is a logical unit of work comprising one or more operations that must execute atomically — all or nothing. Example: transferring ₹500 = `UPDATE` debit of account A + `UPDATE` credit of account B; either both persist (COMMIT) or neither (ROLLBACK).

**Q2. List the five transaction states.**
✅ **Answer:** Active → (partially committed) → committed, or → failed → aborted. Active = executing; partially committed = final statements done, flush in progress; committed = permanent; failed = cannot continue; aborted = rolled back to the pre-transaction state.

**Q3. Why is ROLLBACK important even when nothing crashes?**
✅ **Answer:** Application-level reasons: a business rule fails mid-way (insufficient funds), the user cancels, or a validation error appears after earlier statements already ran. ROLLBACK lets you discard partial work cleanly instead of leaving inconsistent data.

---

## 9. ACID Properties

📌 The four guarantees every transaction system must give. Examples use: transfer ₹400 from A (1000) to B (500).

| Property | Promise | Transfer example |
|---|---|---|
| **A**tomicity | all or nothing | debit AND credit, or neither — a crash between them triggers rollback |
| **C**onsistency | transaction takes DB from one valid state to another | total money stays 1500; CHECK constraints still hold after commit |
| **I**solation | concurrent transactions don't see each other's half-done work | another transaction never sees A = 600 with B = 500 (money created!) |
| **D**urability | once committed, survives crashes | after COMMIT, a power cut cannot lose the transfer (it lives in the WAL) |

- 🔹 **Atomicity** is implemented with undo logs/rollback segments — the DBMS can reverse uncommitted changes.
- 🔹 **Consistency** = correctness with respect to all rules (constraints, triggers, business logic). The programmer defines the rules; the DBMS enforces them.
- 🔹 **Isolation** is achieved by locking or MVCC (Section 12) — each transaction behaves as if it ran alone.
- 🔹 **Durability** is achieved with write-ahead logging — committed changes are on disk before COMMIT returns.

### 🎯 Questions

**Q1. Explain ACID with a funds-transfer example. (Classic 5-mark question)**
✅ **Answer:** Transfer ₹400 from A(1000) to B(500): **Atomicity** — the debit and credit are one unit; a crash after the debit alone is undone on restart, so money never vanishes. **Consistency** — before and after, A+B = 1500 and all constraints hold. **Isolation** — no other transaction can observe the intermediate state (A=600, B=500) and compute a wrong total. **Durability** — after COMMIT reports success, the new balances survive any crash because they were written to the log first.

**Q2. Which ACID property does concurrent execution threaten most, and why?**
✅ **Answer:** Isolation. Multiple transactions interleaving can read each other's uncommitted data or overwrite each other's updates (lost update, dirty read, etc.), producing results no serial execution would produce. Locking/isolation levels restore the illusion of running alone.

**Q3. Which property protects a committed transaction against a power failure two seconds later?**
✅ **Answer:** Durability — committed changes are already recorded in the write-ahead log on non-volatile storage, so recovery replays (redo) them after the crash.

---

## 10. Schedules

📌 A **schedule** (history) is the chronological order in which operations of several transactions execute. Notation: `r1(X)` = transaction 1 **reads** X, `w2(X)` = transaction 2 **writes** X, `c1` = T1 commits, `a1` = T1 aborts.

### 🔹 Serial vs concurrent

```text
S1 (serial — one after another):   r1(X) w1(X) r2(X) w2(X)
S2 (concurrent — interleaved):     r1(X) r2(X) w1(X) w2(X)
```

| | Serial schedule | Concurrent schedule |
|---|---|---|
| Shape | T1 entirely, then T2 (or vice versa) | operations interleaved |
| Result | always consistent — but slow (no overlap) | fast (CPU/disk overlap) — but may be **wrong** |
| n transactions | n! possible serial orders | far more possible interleavings |

📌 The goal of concurrency control (Section 12) and serializability (Section 11): let schedules be concurrent yet **equivalent to some serial schedule**.

### 🎯 Questions

**Q1. What is a schedule?**
✅ **Answer:** A schedule is the sequence of operations (reads, writes, commits, aborts) from multiple transactions as they actually execute over time, e.g., `S: r1(A); w1(A); r2(A); w2(A); c1; c2`.

**Q2. Why not just run every schedule serially?**
✅ **Answer:** Serial schedules are always consistent but waste resources — one transaction's I/O wait blocks everyone. Concurrency increases throughput and response time; the price is that some interleavings are incorrect, which serializability checks eliminate.

**Q3. How many distinct serial schedules exist for 4 transactions?**
✅ **Answer:** 4! = 24 — each permutation of the transactions is one serial order. (The number of *concurrent* schedules is far larger, which is why we need serializability to filter them.)

---

## 11. Serializability

📌 A concurrent schedule is acceptable if it is **equivalent to some serial schedule** — such a schedule is **serializable**.

### 🔹 Conflict serializability — conflicting operations

Two operations **conflict** iff all three hold:

1. they belong to **different transactions**,
2. they access the **same data item**,
3. **at least one is a write**.

So `r1(X)`–`r2(X)` do **not** conflict; `r1(X)`–`w2(X)`, `w1(X)`–`r2(X)`, `w1(X)`–`w2(X)` do.

### 🔹 Precedence (serialization) graph — the method

1. Draw one node per committed transaction.
2. For every conflicting pair where Ti's operation comes **before** Tj's, draw an edge **Ti → Tj**.
3. **Cycle?** → NOT conflict serializable. **Acyclic?** → serializable; a topological order gives the equivalent serial schedule.

### 🔹 Worked Example A — serializable ✅

```text
S_A: r1(X)  w1(X)  r2(X)  w2(X)  r2(Y)  w2(Y)  r3(Y)  w3(Y)
```

- Conflicts on X: `w1(X)` before `r2(X)` and `w2(X)` → edge **T1 → T2**.
- Conflicts on Y: `w2(Y)` before `r3(Y)` and `w3(Y)` → edge **T2 → T3**.
- Graph: `T1 → T2 → T3` — **no cycle** → conflict serializable.
- Equivalent serial schedule: **T1, T2, T3**.

```
T1 ──▶ T2 ──▶ T3      (acyclic ✅)
```

### 🔹 Worked Example B — NOT serializable ❌

```text
S_B: r1(A)  r2(A)  w1(A)  w2(A)
```

- `r2(A)` before `w1(A)` (conflict) → edge **T2 → T1**.
- `w1(A)` before `w2(A)` (conflict) → edge **T1 → T2**.
- Graph has the cycle `T1 → T2 → T1` → **not conflict serializable**.
- Sanity check with serial orders: T1,T2 would make T2 read T1's value (S_B read the original ✗); T2,T1 would end with T1's write (S_B ends with w2 ✗). No serial order works.

```
T1 ⇄ T2      (cycle ❌)
```

### 🔹 View serializability (brief)

📌 A weaker (more permissive) notion: S is **view serializable** if it is *view equivalent* to some serial schedule — same **reads-from** relation (each read reads the same write or the initial value) and same **final write** per item.

```text
T1: r(A), w(A)        S_V:  r1(A)  w2(A)  w1(A)  w3(A)
T2: w(A)   ← blind write (never reads A)
T3: w(A)   ← blind write
```

- Not conflict serializable: `r1<w2` gives T1→T2, but `w2<w1` gives T2→T1 — a cycle.
- **View serializable**: equivalent to serial T1→T2→T3 — the only read `r1(A)` reads the initial value in both, and the final write of A is T3's in both.
- ✅ Moral: **blind writes** let schedules be view- but not conflict-serializable. Every conflict-serializable schedule is view serializable, not vice versa.

### 🎯 Questions

**Q1. Check for conflict serializability: `S: r1(A); r2(A); w1(A); w2(A); r1(B); w1(B)`**
✅ **Answer:** Conflicts on A: `w1(A)` after `r2(A)` → T2→T1; `w1(A)` before `w2(A)` → T1→T2. (r1–r2 are both reads — no conflict.) Conflicts on B: only T1 touches B. Graph: cycle T1⇄T2 → **not conflict serializable**.

**Q2. State the three conditions for two operations to conflict.**
✅ **Answer:** They must (1) belong to different transactions, (2) access the same data item, and (3) have at least one write between them. Read–read pairs never conflict, so their relative order never affects equivalence.

**Q3. How does view serializability differ from conflict serializability?**
✅ **Answer:** View equivalence only requires identical reads-from relations and identical final writers; conflict equivalence requires the same order of every conflicting pair. View is weaker: schedules with blind writes (like `r1(A) w2(A) w1(A) w3(A)`) can be view serializable without being conflict serializable. Exams prefer conflict serializability because it is decidable in polynomial time via the precedence graph.

---

## 12. Concurrency Control

### 🔹 Problems without concurrency control (learn all four!)

**1. Lost update** — T2 overwrites T1's change (read-modify-write in application code):

| Time | T1: withdraw ₹400 from A | T2: withdraw ₹300 from A | Value of A |
|---|---|---|---|
| t1 | `SELECT` A → 1000 | — | 1000 |
| t2 | — | `SELECT` A → 1000 | 1000 |
| t3 | computes 1000 − 400 = 600 | — | 1000 |
| t4 | — | computes 1000 − 300 = 700 | 1000 |
| t5 | `UPDATE` A = 600 | — | 600 |
| t6 | — | `UPDATE` A = 700 | **700** — T1's ₹400 vanished ❌ |

**2. Dirty read** — reading another transaction's **uncommitted** data:

| Time | T1 | T2 | Comment |
|---|---|---|---|
| t1 | `UPDATE` A = 1000 − 400 = 600 (uncommitted) | — | not committed yet |
| t2 | — | `SELECT` A → 600 | dirty read ⚠️ |
| t3 | `ROLLBACK` (rule violation found) | — | A is back to 1000 |
| t4 | — | uses 600 to approve a loan | decision based on data that never existed ❌ |

**3. Unrepeatable read** — same query, different answers within one transaction:

| Time | T1 | T2 | Comment |
|---|---|---|---|
| t1 | `SELECT` balance A → 1000 | — | first read |
| t2 | — | `UPDATE` A = 700; `COMMIT` | someone else commits |
| t3 | `SELECT` balance A again → 700 | — | read is not repeatable ❌ |

**4. Phantom read** — re-running a range query finds **new rows**:

| Time | T1 (count rich accounts) | T2 | Comment |
|---|---|---|---|
| t1 | `SELECT COUNT(*)` WHERE balance > 5000 → 2 | — | sees 2 rows |
| t2 | — | `INSERT` account with 9000; `COMMIT` | new row appears |
| t3 | `SELECT COUNT(*)` again → 3 | — | a "ghost" row joined the range ❌ |

### 🔹 Locking: shared (S) and exclusive (X) locks

- Read an item → acquire **S-lock**; write/modify → acquire **X-lock**. Two-phase locking (2PL) guarantees conflict-serializable schedules.

**Lock compatibility matrix**

| Requested ↓ \ Held → | S (read) | X (write) |
|---|---|---|
| **S (read)** | ✅ yes | ❌ no |
| **X (write)** | ❌ no | ❌ no |

📌 Many readers may share an item; a writer must be alone. An X-request waits until all held locks on the item are released.

### 🔹 Two-Phase Locking (2PL)

```
   growing phase                    shrinking phase
(acquire only, no release)      (release only, no acquire)
────────────────────────────────┬──────────────────────────▶ time
                                └── lock point
```

- Rule: after releasing your **first** lock, you may acquire **no new** locks.
- Guarantees: every 2PL schedule is conflict serializable (the lock point order is a serial order).
- **Strict 2PL**: hold all X-locks until commit — prevents cascading aborts (links to Section 13); PostgreSQL effectively behaves this way for row updates via MVCC + row locks.
- ⚠️ Deadlock is still possible: T1 holds A, wants B; T2 holds B, wants A — both wait forever. PostgreSQL's deadlock detector aborts one transaction with `ERROR: deadlock detected`.

| Time | T1 | T2 |
|---|---|---|
| t1 | X-lock A ✅ | — |
| t2 | — | X-lock B ✅ |
| t3 | wants X-lock B → waits | — |
| t4 | — | wants X-lock A → waits ♻ deadlock |

### 🔹 Timestamp ordering (brief)

- Every transaction gets a timestamp `TS(T)` at start; **older = higher priority**.
- Basic TO: if T tries to read/write an item already written by a **younger** transaction, T is too old → **abort T** (restart with a new timestamp).
- **Thomas write rule**: an obsolete (older) write is simply *ignored* instead of aborting — allows more schedules.
- No locks → no deadlocks, but aborts can be frequent.

### 🔹 Isolation levels in PostgreSQL

| Level | Dirty read | Nonrepeatable read | Phantom | How PostgreSQL does it |
|---|---|---|---|---|
| READ UNCOMMITTED | ❌ never* | possible | possible | *alias — behaves as READ COMMITTED |
| **READ COMMITTED (default)** | ❌ | possible | possible | new snapshot per statement |
| REPEATABLE READ | ❌ | ❌ | ❌* | one snapshot for the whole tx |
| SERIALIZABLE | ❌ | ❌ | ❌ | RR + SSI: detects dangerous read–write patterns |

📌 *In PostgreSQL, REPEATABLE READ gives one fixed snapshot, so phantoms don't appear; if another transaction modified rows you touch, you get `ERROR: could not serialize access` instead.

```sql
SHOW transaction_isolation;                         -- check current level (default: read committed)
BEGIN ISOLATION LEVEL REPEATABLE READ;              -- start a transaction at a stricter level
SELECT balance FROM accounts WHERE id = 1;          -- every read here sees ONE fixed snapshot
COMMIT;                                             -- finish the transaction
```

**Line-by-line walkthrough**

- Default is READ COMMITTED: each statement sees the freshest committed data — dirty reads are still impossible (MVCC never shows uncommitted rows).
- REPEATABLE READ / SERIALIZABLE may abort with serialization errors — applications must be ready to retry.
- Expected output of SHOW: `read committed`.

### 🎯 Questions

**Q1. Explain the lost update problem with an example.**
✅ **Answer:** T1 and T2 both read A = 1000, T1 computes 600 and writes it, then T2 — which still holds the old 1000 — computes 700 and writes it. T1's ₹400 debit is silently overwritten. Cure: X-locks (or `SELECT ... FOR UPDATE` / a single `UPDATE ... SET balance = balance - x` statement) so the read-modify-write is atomic.

**Q2. Why can two S-locks coexist but S and X cannot?**
✅ **Answer:** Two reads never change the data, so concurrent readers are harmless and both S-locks can be granted. A write changes the item, so no other reader (would see half-written data) or writer (would lose updates) may hold any lock on it meanwhile.

**Q3. State the 2PL rule and its guarantee and drawback.**
✅ **Answer:** Rule: transactions have a growing phase (only acquire locks) and a shrinking phase (only release); no lock may be acquired after the first release. Guarantee: every schedule produced is conflict serializable. Drawback: deadlocks remain possible (e.g., T1 holds A wants B, T2 holds B wants A), handled by detection/timeout.

**Q4. Which isolation level is PostgreSQL's default, and what does it prevent?**
✅ **Answer:** READ COMMITTED. Each statement sees a fresh snapshot of committed data — dirty reads are impossible in all PostgreSQL levels (MVCC), but nonrepeatable reads and phantoms are possible at this level; use REPEATABLE READ/SERIALIZABLE to prevent them.

---

## 13. Recoverability

### 🔹 Recoverable schedules

📌 A schedule is **recoverable** if every transaction that **reads data written by Ti** commits **only after Ti commits**. Otherwise, if Ti aborts, its readers would have committed based on data that "never happened" — impossible to undo.

```text
Unrecoverable (bad ❌):        Recoverable (good ✅):
T1: w1(A)  ...  a1 (abort)     T1: w1(A)  c1
T2:    r2(A)  c2               T2:    r2(A)    c2
      (T2 committed AFTER              (T2 commits only
       T1 aborted!)                     AFTER T1 committed)
```

- Why unrecoverable is bad: A was 1000; T1 wrote 600 (uncommitted); T2 read 600 and **committed**; T1 aborts → A must return to 1000, but T2 already acted on 600 (e.g., approved a loan). A committed transaction can never be rolled back — the schedule is stuck.

### 🔹 Cascadeless schedules

📌 Even recoverable schedules can force **cascading rollbacks**: T1 aborts → T2 (which read T1's data) must abort → T3 (which read T2's data) must abort... 

✅ **Cascadeless**: transactions may read **only committed** values — no dirty reads, so no cascades. Strict 2PL (hold X-locks till commit) implements this; PostgreSQL's MVCC never exposes uncommitted rows, so PostgreSQL schedules are naturally cascadeless.

### 🔹 Log-based recovery

📌 The DBMS keeps a **log** on stable storage, written **before** the data pages (Write-Ahead Logging):

| Log record | Meaning |
|---|---|
| `<T, start>` | T has begun |
| `<T, X, old, new>` | T changed X from old to new |
| `<T, commit>` | T committed — its changes are permanent |
| `<T, abort>` | T aborted |

- **Deferred update**: never write transaction changes to the DB until commit. After a crash: **REDO** committed transactions; no UNDO needed (uncommitted work never touched the DB).
- **Immediate update**: changes may reach the DB before commit. After a crash: **UNDO** uncommitted transactions (using `old` values) **and REDO** committed ones (using `new` values).

### 🔹 Checkpoints

📌 Periodically the DBMS forces modified buffers to disk and writes a `<checkpoint>` log record. After a crash, recovery starts from the last checkpoint instead of scanning the whole log.

```
time ──────────────────────────────────────────────────▶
        │◄─ checkpoint ─►              ✖ crash
T1: ████████│                        committed before checkpoint → nothing to do
T2:         │█████████ committed after checkpoint, before crash → REDO
T3:         │████████████── never committed ────────────────→ UNDO
```

- Checkpoints shorten recovery time and bound how much log must be scanned.

### 🎯 Questions

**Q1. Define recoverable schedule. Why is an unrecoverable one disastrous?**
✅ **Answer:** A schedule is recoverable if, whenever Tj reads an item written by uncommitted Ti, Tj commits only after Ti commits. Otherwise, if Ti aborts after Tj committed, Tj's committed actions rest on values that never officially existed — and committed work cannot be undone, so the database cannot be restored to a consistent state.

**Q2. Cascadeless vs recoverable schedules?**
✅ **Answer:** Recoverable only orders the *commits* (readers commit after writers). Cascadeless additionally forbids reading *uncommitted* data at all, eliminating cascading aborts entirely. Every cascadeless schedule is recoverable, but not vice versa.

**Q3. Differentiate deferred and immediate update recovery.**
✅ **Answer:** Deferred update keeps uncommitted changes out of the database; crash recovery only REDOes transactions with a `<T, commit>` record. Immediate update lets changes hit the database before commit; recovery must UNDO uncommitted transactions (old values) and REDO committed ones (new values). Both rely on the log being written before the data (WAL).

**Q4. What is a checkpoint and what problem does it solve?**
✅ **Answer:** A checkpoint flushes all modified buffers to disk and records `<checkpoint>` in the log. On restart, recovery scans only from the last checkpoint: transactions committed since it are redone, active/uncommitted ones undone — instead of replaying the entire, possibly huge, log.

---

## ⚡ Quick Revision

| Concept | One-liner | Key syntax |
|---|---|---|
| PL/pgSQL block | DECLARE (opt) → BEGIN → EXCEPTION (opt) → END | `DO $$ BEGIN ... END $$;` |
| %TYPE | type of one column | `v accounts.balance%TYPE` |
| %ROWTYPE | type of a whole row | `r accounts%ROWTYPE` |
| CONSTANT | variable that can never change | `c CONSTANT INT := 5` |
| IF/ELSIF/ELSE | branch on conditions (note: ELSIF!) | `IF ... THEN ... END IF` |
| CASE | match value or conditions | `CASE x WHEN ... END CASE` |
| LOOP + EXIT | unconditional loop with exit test | `EXIT WHEN cond` |
| WHILE | pre-test loop | `WHILE cond LOOP ... END LOOP` |
| FOR (range) | auto counter, read-only var | `FOR i IN 1..n LOOP` |
| FOR (query) | implicit cursor over rows | `FOR rec IN SELECT ... LOOP` |
| Function | returns value; called in SELECT | `CREATE FUNCTION f(...) RETURNS ...` |
| RETURNS TABLE | result set with named columns | `RETURN QUERY SELECT ...` |
| SETOF / RETURN NEXT | whole rows / append one row | `RETURNS SETOF t` |
| Procedure | statement (PG 11+); may COMMIT/ROLLBACK | `CREATE PROCEDURE ... CALL p(...)` |
| IN / OUT / INOUT | parameter directions | `CALL p(x, NULL, NULL)` |
| Cursor steps | DECLARE → OPEN → FETCH → CLOSE | `EXIT WHEN NOT FOUND` |
| FOUND / NOT FOUND | last fetch/statement got rows? | `IF FOUND THEN ...` |
| Trigger function | no args, returns TRIGGER | `RETURNS TRIGGER AS $$ ...` |
| NEW / OLD | new / old row (per event) | `BEFORE INSERT OR UPDATE` |
| TG_OP | which event fired | `IF TG_OP = 'DELETE'` |
| Exception handler | catch named conditions | `EXCEPTION WHEN unique_violation` |
| RAISE NOTICE / EXCEPTION | message vs abort | `RAISE EXCEPTION ... USING ERRCODE` |
| RAISE_APPLICATION_ERROR equiv. | custom msg + SQLSTATE | `RAISE EXCEPTION ... ERRCODE = 'P0001'` |
| Transaction | atomic logical unit of work | `BEGIN; ... COMMIT / ROLLBACK` |
| Transaction states | active → (partially) → committed / failed → aborted | — |
| ACID | atomicity, consistency, isolation, durability | — |
| Schedule | chronological interleaving of operations | `r1(X) w2(X) c1` |
| Conflict | different tx + same item + a write | r–w, w–r, w–w |
| Conflict serializable | precedence graph acyclic | topological order = serial order |
| View serializable | same reads-from + final writes | blind-write schedules can pass |
| Lost update | write overwrites another's write | read-modify-write races |
| Dirty read | reading uncommitted data | none in PostgreSQL (MVCC) |
| Unrepeatable read | same query, new answer | REPEATABLE READ fixes |
| Phantom | new rows join a range | SERIALIZABLE fixes |
| S/X locks | readers share, writer excludes | see compatibility matrix |
| 2PL | growing then shrinking phase | deadlock still possible |
| Isolation levels | RC (default) → RR → SERIALIZABLE | `BEGIN ISOLATION LEVEL ...` |
| Recoverable | readers commit after writers | — |
| Cascadeless | read only committed data | strict 2PL / MVCC |
| WAL | log before data page | `<T, X, old, new>` |
| Checkpoint | flush + log marker; shorter recovery | REDO committed, UNDO active |

---

## 📝 Practice Set

### 🅰️ 2-mark questions

**Q1. What is a transaction?**
✅ **Answer:** A transaction is a logical unit of work — one or more database operations executed as an indivisible whole, ending in exactly one of COMMIT (all changes permanent) or ROLLBACK (no changes). Example: debit + credit in a funds transfer.

**Q2. Differentiate %TYPE and %ROWTYPE.**
✅ **Answer:** `%TYPE` gives a variable the data type of a single column (`v accounts.balance%TYPE`) — one value. `%ROWTYPE` gives a variable the structure of an entire table row (`r accounts%ROWTYPE`) — it can hold a full row from `SELECT * INTO`. Both auto-adapt if the table definition changes.

**Q3. What does TG_OP hold in a trigger function?**
✅ **Answer:** `TG_OP` is a special variable containing the event that fired the trigger: `'INSERT'`, `'UPDATE'`, or `'DELETE'`. One trigger function can therefore serve all three events and branch on TG_OP. Related: TG_NAME, TG_TABLE_NAME, TG_WHEN, TG_LEVEL.

**Q4. RAISE NOTICE vs RAISE EXCEPTION?**
✅ **Answer:** `RAISE NOTICE` prints an informational message and execution continues normally. `RAISE EXCEPTION` aborts the current block/transaction with an error (roll back its uncommitted work), and can be caught in an EXCEPTION handler or by the client.

**Q5. Define dirty read.**
✅ **Answer:** A dirty read occurs when a transaction reads data written by another transaction that has not yet committed. If the writer later rolls back, the reader has used values that never officially existed. PostgreSQL prevents dirty reads at every isolation level via MVCC.

**Q6. What is a checkpoint?**
✅ **Answer:** A checkpoint periodically flushes all modified buffers to disk and writes a `<checkpoint>` record to the log. On restart, recovery scans only from the last checkpoint (REDO committed, UNDO active transactions), greatly shortening recovery time.

### 🅱️ 5-mark questions

**Q7. Explain the ACID properties with a funds-transfer example.**
✅ **Answer:** Transfer ₹400 from A(1000) to B(500). **Atomicity** — debit and credit are one unit: a crash after the debit alone is undone at recovery, so money never disappears. **Consistency** — before and after the transaction A+B = 1500 and every constraint (e.g., `balance >= 0`) still holds; the DB moves from one valid state to another. **Isolation** — concurrent transactions never observe the intermediate state (A=600, B=500) that would let them "create" money; locking/MVCC provides this. **Durability** — once COMMIT returns, the balances survive any later crash, because the change was written to the write-ahead log on disk first. Together ACID is what makes `BEGIN ... COMMIT` trustworthy.

**Q8. Differentiate between a function and a procedure in PostgreSQL.**
✅ **Answer:** (1) Invocation — functions run inside queries (`SELECT f(x)`); procedures run as statements (`CALL p(x)`). (2) Return — a function must declare `RETURNS` and hands back a value/result set; a procedure returns nothing through RETURN (outputs leave via OUT/INOUT parameters). (3) Transaction control — procedures may execute COMMIT/ROLLBACK; functions never may. (4) Usage — functions suit computations usable in WHERE/CHECK/indexes; procedures suit multi-step operations like transfers. (5) History — procedures exist since PostgreSQL 11; functions since forever (Oracle has the same split).

**Q9. Check whether `S: r1(A); r2(A); w1(A); w2(A)` is conflict serializable.**
✅ **Answer:** Conflicting pairs: `r2(A)` before `w1(A)` (read–write, different tx) → edge T2→T1; `w1(A)` before `w2(A)` (write–write) → edge T1→T2. (`r1(A)`–`r2(A)` are both reads — no conflict.) The precedence graph contains the cycle T1→T2→T1, so **S is not conflict serializable**. Verification: serial T1,T2 makes T2 read T1's value, but in S, T2 read the original; serial T2,T1 ends with T1's write, but S ends with w2. No serial order matches — confirmed not serializable.

**Q10. Explain Two-Phase Locking with its phases, guarantee, and drawback.**
✅ **Answer:** 2PL splits a transaction's locking into a **growing phase**, where it only acquires S/X locks, and a **shrinking phase**, where it only releases; after the first release (the *lock point*) no new locks may be acquired. **Guarantee:** every schedule produced by 2PL is conflict serializable — the order of lock points gives a valid serial order. **Variants:** strict 2PL holds X-locks until commit, preventing cascading aborts. **Drawback:** deadlock is still possible — T1 holds A and wants B while T2 holds B and wants A — resolved by PostgreSQL's deadlock detector, which aborts one victim with `deadlock detected`.

**Q11. List the four concurrency anomalies with one-line examples.**
✅ **Answer:** (1) **Lost update** — T2 overwrites T1's uncommitted-or-committed write because both read the same old value (A=1000 → T1 writes 600 → T2 writes 700; ₹400 lost). (2) **Dirty read** — T2 reads A=600 that T1 never committed; T1 rolls back. (3) **Unrepeatable read** — T1 reads A=1000, T2 updates and commits, T1 re-reads A=700. (4) **Phantom read** — T1 counts 2 rich accounts, T2 inserts a new one and commits, T1's recount shows 3. Cures: locking, isolation levels (RR/SERIALIZABLE in PostgreSQL).

### 🧪 Lab / Hands-on (write the code)

**Q12. Write a function `account_count(min_bal NUMERIC) RETURNS INT` counting accounts with at least that balance.**

```sql
CREATE OR REPLACE FUNCTION account_count(min_bal NUMERIC)   -- function with one IN parameter
RETURNS INT AS $$                                           -- returns a single integer
DECLARE                                                     -- declaration section
  cnt INT;                                                  -- holds the count
BEGIN                                                       -- executable section
  SELECT COUNT(*) INTO cnt                                  -- count matching rows into the variable
  FROM accounts WHERE balance >= min_bal;                   -- filter by the parameter
  RETURN cnt;                                               -- send the count back
END;                                                        -- end of body
$$ LANGUAGE plpgsql;                                        -- language

SELECT account_count(6000);                                 -- fresh data: returns 2
```

✅ **How it works:** SELECT INTO fills `cnt`; COUNT(*) never returns NULL so no COALESCE needed; the function is callable anywhere an expression is allowed. Expected output: `2` (accounts 1 and 3 with fresh data).

**Q13. Write a trigger that blocks any salary below 10000 on `employees`.**

```sql
CREATE OR REPLACE FUNCTION guard_salary()                   -- validator trigger function
RETURNS TRIGGER AS $$                                       -- trigger signature
BEGIN                                                       -- executable section
  IF NEW.salary < 10000 THEN                                -- the business rule
    RAISE EXCEPTION 'Salary % is below the 10000 minimum', NEW.salary  -- reject with message
      USING ERRCODE = 'P0001';                              -- custom SQLSTATE
  END IF;                                                   -- end of rule
  RETURN NEW;                                               -- accept the row otherwise
END;                                                        -- end of body
$$ LANGUAGE plpgsql;                                        -- language

DROP TRIGGER IF EXISTS trg_guard_salary ON employees;       -- safe to re-run
CREATE TRIGGER trg_guard_salary                             -- name
BEFORE INSERT OR UPDATE ON employees                        -- must be BEFORE to reject early
FOR EACH ROW                                                -- per changed row
EXECUTE FUNCTION guard_salary();                            -- run the validator
```

✅ **How it works:** BEFORE timing means the row is not yet stored, so raising an exception cancels the INSERT/UPDATE cleanly. Returning NEW lets valid rows pass unchanged. Test: `INSERT INTO employees VALUES (3,'Zara',9000);` → error; `(...,15000)` → succeeds.

**Q14. Write a procedure `bulk_bonus(pct NUMERIC)` that raises every balance by pct% in one transaction.**

```sql
CREATE OR REPLACE PROCEDURE bulk_bonus(pct NUMERIC)         -- one IN parameter
LANGUAGE plpgsql                                            -- language
AS $$                                                       -- body starts
BEGIN                                                       -- executable section
  UPDATE accounts                                           -- one set-based statement
  SET balance = balance + (balance * pct / 100);            -- add pct percent to every row
  COMMIT;                                                   -- make it permanent immediately
EXCEPTION                                                   -- handler section
  WHEN OTHERS THEN                                          -- anything went wrong?
    ROLLBACK;                                               -- undo the partial update
    RAISE NOTICE 'Bonus failed: %', SQLERRM;                -- report why
END;                                                        -- end of body
$$;                                                         -- end of body string

CALL bulk_bonus(2);                                         -- every balance +2% (works: top-level CALL)
```

✅ **How it works:** COMMIT inside the procedure is legal because CALL runs at the top level; if the UPDATE errors (e.g., a CHECK violation), the handler ROLLBACKs and reports. This is the atomicity idea from Part B in practice.

**Q15. Write a DO block that uses a cursor FOR loop to print accounts below 6000.**

```sql
DO $$                                                       -- anonymous block wrapper
DECLARE                                                     -- optional: nothing needed for the loop
  poor_count INT := 0;                                      -- our own counter of poor accounts
BEGIN                                                       -- executable section
  FOR rec IN SELECT id, holder, balance                     -- implicit cursor over the filtered rows
             FROM accounts WHERE balance < 6000 LOOP        -- only balances under 6000
    RAISE NOTICE 'Account % (%) needs funds: %',            -- process one row per pass
                 rec.id, rec.holder, rec.balance;           -- fields of the current row
    poor_count := poor_count + 1;                           -- count it
  END LOOP;                                                 -- auto open/fetch/close
  RAISE NOTICE 'Total accounts needing funds: %', poor_count;  -- summary after the loop
END $$;                                                     -- end of block
```

✅ **How it works:** the FOR loop is an implicit cursor — it opens the query, fetches into auto-declared `rec`, and closes at the end. Fresh data prints accounts 2 (Bhavya, 5000.00) and then `Total accounts needing funds: 1`.

---

🎉 **End of Unit 5** — you can now *program* a database (Part A) and *reason* about how it stays correct under concurrency and crashes (Part B). Re-run the Section 1 block any time to reset the sample data. Good luck with INT306! 🍀





