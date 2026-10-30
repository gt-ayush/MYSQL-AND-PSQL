# Unit 6 — NoSQL Databases

> 🎓 **INT306 · Database Management Systems** · Practical: **PostgreSQL (psql)** for SQL parts, **MongoDB (mongosh)** for NoSQL parts.

For five units we lived inside **relational** databases: tables, rows, fixed schemas, SQL, joins, ACID. They are perfect for structured, integrity-critical data — but modern web-scale apps brought three new problems: **Big Data** (terabytes arriving every second), **rapidly changing data shapes**, and the need to **scale out horizontally** across many cheap servers instead of buying one giant machine. **NoSQL ("Not Only SQL")** databases were built for exactly these problems.

This unit is half theory (NoSQL types, SQL vs NoSQL, JSON databases, DynamoDB, serverless, vector DBs) and half hands-on (MongoDB CRUD in `mongosh`, PostgreSQL `jsonb`, `EXPLAIN` tuning, `pgvector`). 💡

**✅ What you'll learn**

- 📌 Why NoSQL exists — scale, schema flexibility, big data
- 📌 The 4 NoSQL families: document, key-value, column-family, graph
- 📌 MongoDB: BSON, database → collection → document structure
- 📌 Full CRUD + aggregation pipeline in `mongosh`
- 📌 SQL vs NoSQL — the classic comparison (ACID vs BASE)
- 📌 JSON databases, JSON syntax, JSON vs BSON
- 📌 JSON inside PostgreSQL: `jsonb`, `->`, `->>`
- 📌 Amazon DynamoDB — tables, items, keys, capacity modes
- 📌 Serverless databases (Aurora Serverless, Atlas, Neon, PlanetScale)
- 📌 Indexes + `EXPLAIN` performance tuning (PostgreSQL & MongoDB)
- 📌 Vector databases, embeddings & `pgvector`

---

## 📑 Table of Contents

| # | Section |
|---|---------|
| 1 | [Introduction to NoSQL](#1-introduction-to-nosql) |
| 2 | [Introduction to MongoDB](#2-introduction-to-mongodb) |
| 3 | [SQL vs NoSQL](#3-sql-vs-nosql) |
| 4 | [Structure of MongoDB](#4-structure-of-mongodb) |
| 5 | [Working with MongoDB (CRUD in mongosh)](#5-working-with-mongodb-crud-in-mongosh) |
| 6 | [JSON Databases](#6-json-databases) |
| 7 | [JSON Representation of a Relational Dataset](#7-json-representation-of-a-relational-dataset) |
| 8 | [Amazon DynamoDB](#8-amazon-dynamodb) |
| 9 | [Serverless Cloud Databases](#9-serverless-cloud-databases) |
| 10 | [Index Creation & Performance Comparison using EXPLAIN](#10-index-creation--performance-comparison-using-explain) |
| 11 | [Vector Databases](#11-vector-databases) |
| ⚡ | [Quick Revision](#-quick-revision) |
| 📝 | [Practice Set](#-practice-set) |

---

## 1. Introduction to NoSQL

### 📌 Why NoSQL?

A traditional RDBMS hits four walls at web scale:

🔹 **Scale** — RDBMS mostly **scale up** (buy a bigger server — expensive, has a hard ceiling). NoSQL databases **scale out** (add more commodity servers and spread/shard the data across them). Instagram-scale apps need thousands of nodes, not one supercomputer.

🔹 **Schema flexibility** — In SQL, every row must obey the table's fixed schema; changing it means `ALTER TABLE` + painful migrations on millions of rows. In NoSQL, **each record can have its own shape** — add a field to new records without touching old ones. Perfect for fast-moving products.

🔹 **Big Data (the 3 Vs)** — **Volume** (petabytes), **Velocity** (IoT streams, clickstreams, millions of writes/sec), **Variety** (structured + semi-structured JSON + unstructured text/images). Relational engines were designed in the 1970s for banking-scale data, not this.

🔹 **Developer productivity + availability** — JSON documents map directly to application objects (no ORM pain), and built-in replication gives **high availability** even when servers fail.

> 📝 **Exam one-liner:** *NoSQL = a family of non-relational databases designed for distributed, large-scale, flexible-schema data. The name means "Not Only SQL" — many NoSQL databases still offer SQL-like query languages.*

### 📌 The Four Types of NoSQL Databases

| Type | Data is stored as | Sweet spot (typical use) | Popular examples |
|------|-------------------|--------------------------|------------------|
| 📄 **Document** | JSON/BSON documents — fields, nested objects, arrays | Product catalogs, user profiles, CMS, mobile backends | **MongoDB**, CouchDB, Amazon Firestore |
| 🔑 **Key–Value** | Opaque value looked up by a unique key (like a giant hash map) | Caching, sessions, shopping carts, leaderboards | **Redis**, **Amazon DynamoDB**, Riak |
| 🏛️ **Column-family** | Rows grouped into column families; sparse, wide columns | Time-series, IoT, analytics, write-heavy workloads | **Apache Cassandra**, HBase, ScyllaDB |
| 🕸️ **Graph** | Nodes + edges (+ properties on both) | Social networks, fraud detection, "friends of friends" | **Neo4j**, Amazon Neptune, ArangoDB |

🔹 Document stores are the most exam-important (MongoDB is in your practical).
🔹 Key-value stores are the *simplest* and the *fastest* (O(1) key lookup, but no querying inside the value).
🔹 Column-family stores are optimised for **massive write throughput** and reading a few columns of very wide rows.
🔹 Graph stores answer **relationship-heavy questions** (shortest path, degree-3 connections) that become horrible recursive joins in SQL.

### 🎯 Questions

**Q1. What is NoSQL and why did it emerge?**
**A:** NoSQL ("Not Only SQL") is a family of non-relational databases designed for distributed, large-scale, flexible-schema data. It emerged because RDBMS: (i) scale mainly vertically, (ii) enforce a rigid schema, (iii) struggle with Big Data's volume/velocity/variety, and (iv) make joins expensive at scale. NoSQL trades some of SQL's guarantees for horizontal scalability, flexibility and availability.

**Q2. Name the four types of NoSQL databases with one example each.**
**A:** ① Document — MongoDB (JSON-like documents); ② Key–Value — Redis (key → value pairs); ③ Column-family — Cassandra (column families, wide rows); ④ Graph — Neo4j (nodes and edges).

**Q3. Give one scenario where a graph database beats an RDBMS.**
**A:** Multi-level relationship traversal, e.g. "find friends of friends of friends who like cricket". In SQL this is a self-join repeated 3+ times (or a recursive CTE), which explodes in cost. A graph DB just hops edges node-to-node, staying fast regardless of traversal depth.

**Q4. True/False: "NoSQL databases never support SQL." Justify.**
**A:** **False.** NoSQL means "Not Only SQL". Many NoSQL systems offer SQL-like syntax: Cassandra has CQL, Couchbase has N1QL, and MongoDB's aggregation pipeline mirrors SQL's `WHERE`/`GROUP BY`/`ORDER BY`. The name signals moving *beyond* classic relational SQL, not away from it.

---

## 2. Introduction to MongoDB

### 📌 What is MongoDB?

🔹 A **document-oriented**, open-source NoSQL database written in C++, default port **27017**.
🔹 Stores data as **BSON documents** (binary JSON) grouped into **collections** (≈ tables without a fixed schema).
🔹 You interact with it through **mongosh** (the MongoDB Shell) — you literally write **JavaScript** to query data.
🔹 One logical database can hold many collections; one server (deployment) can hold many databases.

### 📌 BSON — the storage format

🔹 MongoDB does **not** store JSON *text*. It stores **BSON (Binary JSON)** — a binary-encoded superset of JSON.
🔹 Superset = everything JSON has, **plus richer types**: `Date`, `ObjectId`, 32-bit/64-bit integers, `Decimal128`, `BinData`, etc. (Plain JSON has only ONE number type and NO date type.)
🔹 Binary + length-prefixed → **faster to parse** and **traversable** (the engine can skip a field without reading it all).
🔹 You *write* JSON-looking objects in mongosh; the shell converts them to BSON before storing.

### 📌 Key Features

🔹 **Schema-less** — documents in one collection can have different fields.
🔹 **Rich queries + many index types** — single-field, compound, text, geospatial, TTL.
🔹 **Aggregation pipeline** — multi-stage analytics ($match → $group → $sort …).
🔹 **Replication (replica sets)** — automatic failover → high availability.
🔹 **Auto-sharding** — data split across servers → horizontal scaling built-in.
🔹 **ACID transactions** — single-document atomic by design; multi-document transactions since v4.0.

### 📌 Typical Use Cases

🔹 Product catalogs / e-commerce (nested variants, attributes differ per product).
🔹 Content management, blogs, user profiles (structure keeps evolving).
🔹 Real-time analytics, IoT sensor streams (high write volume).
🔹 Mobile-app backends (JSON ↔ object mapping is frictionless).

```javascript
use college                                // create/switch to the 'college' database
db.students.insertOne({                    // insert one document into the 'students' collection
  name: "Aarav",                           // string field holding the student's name
  age: 20,                                 // numeric field for age
  skills: ["SQL", "Python"]                // array field holding multiple values
})                                         // ends the insert call
db.students.findOne()                      // read one document back to verify
```

**Line-by-line walkthrough:**
- `use college` — MongoDB creates databases **lazily**: nothing exists until the first write.
- The document mixes types in one record: string, number, array — no `CREATE TABLE` was needed. ✅
- `insertOne` returns `{ acknowledged: true, insertedId: ObjectId('...') }` — MongoDB generated a unique `_id` for us.
- `findOne()` prints the document, now including the `_id` field.

### 🎯 Questions

**Q1. What is BSON? Give two advantages over plain JSON.**
**A:** BSON (Binary JSON) is MongoDB's binary, length-prefixed storage format — a superset of JSON. Advantages: ① **faster to parse/traverse** (engine skips fields via length prefixes), ② **richer types** — real `Date`, `ObjectId`, int32/int64, `Decimal128`, which JSON cannot represent.

**Q2. List four features of MongoDB.**
**A:** Schema-less documents; rich indexing (compound/text/geo); aggregation pipeline for analytics; replica sets + auto-sharding for HA and horizontal scale; multi-document ACID transactions (v4.0+). (Any four.)

**Q3. Mention two real-world use cases of MongoDB with a one-line reason.**
**A:** ① E-commerce catalog — every product can have a different attribute set, no schema change needed; ② IoT/real-time analytics — millions of writes/sec are absorbed by horizontal sharding.

**Q4. What is mongosh?**
**A:** The MongoDB Shell — an interactive JavaScript CLI used to connect to a MongoDB server and run CRUD, administration and aggregation commands.

---

## 3. SQL vs NoSQL

### 📌 The Big Comparison Table

| Aspect | SQL (Relational — PostgreSQL, MySQL) | NoSQL (MongoDB, Cassandra, Redis…) |
|--------|--------------------------------------|-------------------------------------|
| **Data model** | Tables (rows × columns), normalized | Documents / key-value / column-family / graph |
| **Schema** | Fixed, declared before insert (`DDL`) | Dynamic — each record can have its own shape |
| **Query language** | Standard SQL (one language everywhere) | Varies per database (Mongo query language, CQL…) |
| **Joins** | Native `JOIN`s across tables | Avoided — data is **embedded/denormalized** (`$lookup` exists but is used sparingly) |
| **Transactions** | Full multi-row **ACID** by default | Single-document atomic; often **BASE** (eventual consistency); multi-doc ACID in modern engines |
| **Consistency** | Strong (every read sees latest committed data) | Often **eventual** (replicas converge after a delay) |
| **Scaling** | Vertical (bigger machine); read replicas | **Horizontal sharding** built into the design |
| **Data duplication** | Minimized via normalization | Accepted — duplicate data to make reads fast |
| **Best for** | Banking, ERP, inventory, complex reports, strict integrity | Big data, real-time feeds, CMS, rapid prototyping, flexible content |
| **Examples** | PostgreSQL, MySQL, Oracle, SQL Server | MongoDB, Cassandra, Redis, Neo4j, DynamoDB |

### 📌 ACID vs BASE (memorize this mini-table)

| Property | ACID (SQL world) | BASE (NoSQL world) |
|----------|------------------|--------------------|
| **A** | Atomicity — all-or-nothing transactions | **Basically Available** — system replies, maybe with stale data |
| **C** | Consistency — data always moves between valid states | **Soft state** — state may change over time (replicas catching up) |
| **I** | Isolation — concurrent transactions don't clash | **Eventual consistency** — copies converge to the same value eventually |
| **D** | Durability — committed data survives crashes | Availability is prioritized over instant consistency |

> 💡 **CAP theorem in one line:** when a network partition happens, a distributed system must pick **C**onsistency or **A**vailability — classic RDBMS pick C, many NoSQL systems pick A (with eventual consistency).

### 📌 When to choose which?

✅ **Choose SQL when:** money is involved, relationships are complex, reporting/ad-hoc queries dominate, schema is stable.
✅ **Choose NoSQL when:** huge scale-out writes, schema evolves fast, data arrives as JSON, availability > instant consistency (feeds, catalogs, logs, IoT).

### 🎯 Questions

**Q1. Compare SQL and NoSQL databases (any six points).**
**A:** Use the big table above — data model, schema, query language, joins, transactions (ACID vs BASE), consistency, scaling, examples. State clearly: SQL = relational, fixed schema, joins, vertical scaling; NoSQL = non-relational, flexible schema, denormalized documents, horizontal scaling.

**Q2. Why would an Instagram-like feed app prefer NoSQL?**
**A:** Millions of writes/sec (velocity) → horizontal sharding; posts contain variable data (image, poll, reel) → flexible documents; a slightly stale like-count is acceptable → eventual consistency (BASE) is fine; fetching the whole post as one JSON document avoids multi-table joins.

**Q3. Does NoSQL mean "no transactions"?**
**A:** No. Single-document operations in MongoDB are always atomic. Since v4.0, MongoDB also supports multi-document ACID transactions. What classic NoSQL *replaced* is the default of globally strong consistency with BASE/eventual consistency, to gain availability and scale.

**Q4. What is denormalization and why do NoSQL databases embrace it?**
**A:** Deliberately duplicating/combining related data in one record instead of splitting it across normalized tables. It removes runtime joins so a read fetches one document instead of joining five tables — ideal when reads vastly outnumber writes.

---

## 4. Structure of MongoDB

### 📌 The hierarchy: Database → Collection → Document

```text
MongoDB deployment (mongod server / cluster)   // the physical layer: one or more servers
 └── Database: college                         // a named namespace, like one PostgreSQL database
      └── Collection: students                 // a group of documents — like a table, but schema-less
           └── Document: { ... }               // ONE record, stored as BSON, always has a unique _id
                ├── "name"   -> "Aarav"        // a field with a string value (≈ a column value)
                ├── "age"    -> 20             // a field with a number value
                ├── "address" -> { city, pincode }  // an EMBEDDED document (nested object)
                └── "skills" -> [ "SQL", "Python" ] // an ARRAY field (nested list of values)
```

**Line-by-line walkthrough:**
- **Database** — top logical container; you switch with `use college`; one server hosts many.
- **Collection** — the ≈table equivalent; created automatically on first insert; no columns declared, ever.
- **Document** — the ≈row equivalent; a set of field:value pairs; the unit of atomicity in MongoDB.
- Embedded documents & arrays are what make joins unnecessary — related data lives *inside* the row.

### 📌 `_id` — the mandatory primary key

🔹 Every document **must** have a unique `_id` inside its collection. If you don't supply one, MongoDB auto-generates an **ObjectId**.
🔹 ObjectId anatomy — **12 bytes**: ① 4-byte **timestamp** (creation seconds — so ids sort by time!), ② 5-byte **random value** (per server/process), ③ 3-byte **incrementing counter** → uniqueness without a central counter.

### 📌 Common BSON types

| BSON type | Example value | Notes |
|-----------|---------------|-------|
| String | `"Aarav"` | UTF-8 text |
| int32 / int64 | `20` / `NumberLong(...)` | real integer types — JSON has neither |
| Double / Decimal128 | `8.7` / `NumberDecimal("8.7")` | Decimal128 = exact money math |
| Boolean | `true` | |
| Date | `ISODate("2005-08-14")` | millisecond precision — impossible in plain JSON |
| ObjectId | `ObjectId("665f...")` | default `_id` type |
| Array | `["SQL", "Python"]` | ordered, indexable |
| Embedded document | `{ city: "Delhi" }` | nested object |
| null / BinData | `null` / binary | binary = files, UUIDs |

### 📌 A full sample document

```javascript
{                                              // one BSON document from collection 'students'
  _id: ObjectId("665f1a2b9d3e4a0012345678"),   // auto-generated unique key (12 bytes)
  name: "Aarav Sharma",                        // string field
  roll: 101,                                   // 32-bit integer field
  cgpa: 8.7,                                   // double field
  enrolled: true,                              // boolean field
  dob: ISODate("2005-08-14T00:00:00Z"),        // BSON date — plain JSON has no date type!
  address: {                                   // embedded (nested) document — object inside object
    city: "Delhi",                             // field inside the nested document
    pincode: "110001"                          // another nested field
  },                                           // closes the nested 'address' document
  skills: ["SQL", "Python", "Node.js"]         // array field holding multiple values
}                                              // ends the document
```

**Line-by-line walkthrough:**
- `_id` was added automatically — you rarely type it yourself; its timestamp prefix makes it roughly chronological.
- `roll: 101` is a real **int32** in BSON; in pure JSON it would just be "a number".
- `ISODate(...)` only exists **because of BSON** — this is the classic "JSON vs BSON" exam point.
- `address` is queried as `"address.city"` — dot notation reaches into nested documents.
- `skills` is an array — you can match on it (`skills: "SQL"`) and index it (multikey index).

### 🎯 Questions

**Q1. Explain the structure of MongoDB (database → collection → document).**
**A:** A MongoDB deployment hosts **databases** (namespaces). Each database contains **collections** — schema-less groupings analogous to tables. Each collection holds **BSON documents** — field:value records analogous to rows, each with a mandatory unique `_id`. Documents may contain nested documents and arrays, so related data is embedded instead of joined.

**Q2. What is an ObjectId made of?**
**A:** 12 bytes: 4-byte timestamp (seconds since epoch — makes ids time-sortable), 5-byte random value (unique per machine/process), 3-byte incrementing counter (uniqueness within the same second). Together they give a globally unique id with no central coordinator.

**Q3. Why does MongoDB use BSON instead of JSON?**
**A:** ① Speed — binary, length-prefixed format is fast to parse and lets the engine skip unneeded fields; ② richer data types — Date, ObjectId, int32/int64, Decimal128, BinData don't exist in JSON; ③ efficient in-memory traversal during queries and index building.

**Q4. Embed vs reference — when would you pick each?**
**A:** **Embed** when child data is always fetched with the parent and belongs to it (order → line items): one read, no join. **Reference** (store the other document's `_id`) when the child is shared, huge, or updated independently (students ↔ courses): avoids duplication and update anomalies.

---

## 5. Working with MongoDB (CRUD in mongosh)

> Dataset used below: a `students` collection where every document has `name, roll, age, city, cgpa, skills`.

### 📌 5.1 Create / switch a database

```javascript
use college                     // switch to 'college' — created lazily on the first write
db                              // prints the currently selected database name
show dbs                        // lists all databases (college appears only AFTER the first insert)
```

**Line-by-line walkthrough:**
- `use <name>` works even if the database doesn't exist — MongoDB won't create it until you insert something. ✅
- `db` is a variable pointing at the current database; all queries hang off it (`db.students...`).

### 📌 5.2 CREATE — insertOne / insertMany

```javascript
db.students.insertOne({                    // CREATE: insert a single document
  name: "Aarav Sharma",                    // string field: student's name
  roll: 101,                               // unique roll number (acts like a natural key)
  age: 20,                                 // numeric field: age
  city: "Delhi",                           // string field: home city
  cgpa: 8.7,                               // decimal field: current CGPA
  skills: ["SQL", "Python"]                // array field: multiple values in one field
})                                         // returns { acknowledged: true, insertedId: ObjectId(...) }

db.students.insertMany([                   // CREATE: insert several documents in one call
  { name: "Diya Nair",  roll: 102, age: 21, city: "Mumbai", cgpa: 9.1, skills: ["Java"] },
  { name: "Kabir Khan", roll: 103, age: 22, city: "Delhi",  cgpa: 7.4, skills: ["C++", "Go"] },
  { name: "Meera Iyer", roll: 104, age: 20, city: "Pune",   cgpa: 8.9, skills: ["Python", "ML"] }
])                                         // returns { insertedIds: [...] } — one ObjectId per doc
```

**Line-by-line walkthrough:**
- No `CREATE TABLE` anywhere — the **collection is created automatically** by the first insert.
- The four documents don't have to be identical — add extra fields to any of them and it still works.
- Duplicate `_id` (or a unique index violation, e.g. on `roll`) raises an error; by default `insertMany` continues past bad docs unless `ordered: true` is set.
- Expected output: `insertedId` / `insertedIds` with fresh ObjectIds.

### 📌 5.3 READ — find() with filter operators

```javascript
db.students.find()                                        // READ: every document in the collection
db.students.find({ city: "Delhi" })                       // equality: city == 'Delhi'
db.students.find({ age: { $gt: 20 } })                    // $gt: age strictly greater than 20
db.students.find({ cgpa: { $gte: 8, $lte: 9 } })          // implicit AND: cgpa between 8 and 9
db.students.find({ city: { $in: ["Delhi", "Pune"] } })    // $in: city is ANY value in the list
db.students.find({                                        // explicit $and — clearest way to combine
  $and: [                                                 // array of conditions that ALL must hold
    { age: { $gt: 19 } },                                 // condition 1: older than 19
    { cgpa: { $gte: 8.5 } }                               // condition 2: cgpa at least 8.5
  ]                                                       // closes the conditions array
})                                                        // ends the find call
```

**Line-by-line walkthrough:**
- A bare object `{ field: value }` = equality; `{ field: { $op: value } }` = operator condition.
- Putting two keys in one object is already an **implicit AND**; use explicit `$and` when the same field needs multiple different operators or for readability.
- Other operators to remember: `$lt, $lte, $gt, $gte, $ne, $nin, $or, $not, $exists, $regex`.
- `$in` is the MongoDB cousin of SQL's `WHERE city IN (...)`.

### 📌 5.4 Projection — choosing which fields come back

```javascript
db.students.find(                        // same query pattern as before
  { city: "Delhi" },                     // 1st argument: the filter (which documents)
  { name: 1, cgpa: 1, _id: 0 }           // 2nd argument: projection — 1 = include, 0 = exclude
)                                        // returns ONLY name and cgpa (no _id, no city)
```

**Line-by-line walkthrough:**
- Like `SELECT name, cgpa FROM students WHERE city = 'Delhi'` — the projection is the "SELECT list".
- `_id` is the one field returned **by default**, hence the explicit `_id: 0`.
- Mixing 1s and 0s (other than hiding `_id`) is an error — pick "include list" or "exclude list".

### 📌 5.5 UPDATE — updateOne / $set / $inc / $push

```javascript
db.students.updateOne(                       // UPDATE the FIRST document matching the filter
  { roll: 101 },                             // filter: find Aarav by roll number
  { $set: { city: "Noida" } }                // $set: change (or add) fields, keeps the rest
)                                            // returns { matchedCount: 1, modifiedCount: 1 }

db.students.updateOne(                       // atomic in-place arithmetic
  { roll: 103 },                             // filter: Kabir
  { $inc: { age: 1 } }                       // $inc: age = age + 1 — no read-modify-write race
)                                            // perfect for counters: likes, views, stock

db.students.updateOne(                       // modify an array field
  { roll: 101 },                             // filter: Aarav again
  { $push: { skills: "Node.js" } }           // $push: append 'Node.js' to the skills array
)                                            // siblings: $pull (remove), $addToSet (add if absent)

db.students.updateMany(                      // update EVERY document matching the filter
  { cgpa: { $lt: 8 } },                      // filter: all students below 8.0
  { $set: { status: "needs-mentoring" } }    // adds a brand-new field to each of them
)                                            // new field appears only on matched docs = schema flexibility!
```

**Line-by-line walkthrough:**
- The update object **must** use update operators — writing `{ city: "Noida" }` bare would *replace* the whole document (classic beginner bug!).
- `$set` = assign; `$inc` = add to a number (atomic); `$push` = append to array.
- `updateOne` touches the first match only; `updateMany` touches all matches.
- Adding `status` to some documents only — and other documents simply don't have it — is "schema-less" in action. ✅

### 📌 5.6 DELETE — deleteOne / deleteMany

```javascript
db.students.deleteOne({ roll: 103 })            // DELETE: removes the first matching document
db.students.deleteMany({ cgpa: { $lt: 6 } })    // DELETE: removes ALL students below 6.0 cgpa
db.students.deleteMany({})                      // ⚠️ empty filter = deletes EVERY document (like TRUNCATE)
```

**Line-by-line walkthrough:**
- Both return `{ deletedCount: n }` so you can verify what happened.
- `deleteMany({})` wipes data but keeps the collection and its indexes — closer to `TRUNCATE` than `DROP`.
- There is no built-in undo — be extra careful with delete filters in labs.

### 📌 5.7 sort / limit / skip — cursor methods

```javascript
db.students.find()                        // read everyone → returns a cursor
  .sort({ cgpa: -1 })                     // sort by cgpa descending (-1 = desc, 1 = asc)
  .limit(3)                               // keep only the first 3 documents → "top 3"
  .skip(0)                                // skip 0 docs — use 3 for page 2, 6 for page 3
```

**Line-by-line walkthrough:**
- Chaining order matters for readability but MongoDB always applies **sort → skip → limit** internally.
- Pagination formula: `.skip((page - 1) * pageSize).limit(pageSize)` — here `limit(3), skip(3)` = page 2.
- This trio is the exact equivalent of SQL's `ORDER BY ... LIMIT ... OFFSET ...`.

### 📌 5.8 count

```javascript
db.students.countDocuments({})                 // total documents in the collection
db.students.countDocuments({ city: "Delhi" })  // count with a filter (can use an index)
db.students.estimatedDocumentCount()           // near-instant count read from collection metadata
```

**Line-by-line walkthrough:**
- `countDocuments` actually scans (filtered) documents — accurate, slightly slower.
- `estimatedDocumentCount` reads a stored counter — instant but ignores any filter.

### 📌 5.9 Aggregation pipeline ($match → $group → $sort)

```javascript
db.students.aggregate([                        // aggregation = documents flow through stages
  { $match: { age: { $gte: 20 } } },           // stage 1: filter docs (like SQL WHERE)
  { $group: {                                  // stage 2: group remaining docs (like GROUP BY)
      _id: "$city",                            // group key: value of 'city' ($ prefix = "this field")
      avgCgpa: { $avg: "$cgpa" },              // accumulator: average cgpa per city
      total:   { $sum: 1 }                     // accumulator: +1 per doc = count per city
  } },                                         // closes the $group stage
  { $sort: { avgCgpa: -1 } },                  // stage 3: sort groups, best average first
  { $limit: 5 }                                // stage 4: keep only the top 5 cities
])                                             // returns array of { _id: city, avgCgpa, total }
```

**Line-by-line walkthrough:**
- Think of it as a conveyor belt: each stage receives documents, transforms them, passes them on.
- `$match` early = efficient (fewer documents travel through later stages).
- Inside `$group`, `"$city"` means "the value of field city" — the `$` is field-reference syntax.
- SQL equivalent: `SELECT city, AVG(cgpa), COUNT(*) FROM students WHERE age >= 20 GROUP BY city ORDER BY 2 DESC LIMIT 5;`
- Other handy accumulators: `$min`, `$max`, `$sum`, `$push`; other stages: `$project`, `$lookup` (join!), `$unwind`.

### 🎯 Questions

**Q1. Write mongosh commands to: insert a student, find Delhi students with cgpa ≥ 8, and increment everyone's age by 1.**
**A:** `db.students.insertOne({name:"Riya", roll:105, age:19, city:"Delhi", cgpa:8.2, skills:[]});` · `db.students.find({city:"Delhi", cgpa:{$gte:8}});` · `db.students.updateMany({}, {$inc:{age:1}});`

**Q2. Difference between `updateOne` and `updateMany`? Between `$set` and `$inc`?**
**A:** `updateOne` modifies the first matching document; `updateMany` modifies all matches. `$set` assigns a literal value (or adds a field); `$inc` atomically adds a number to the current value (used for counters — avoids read-modify-write races).

**Q3. What is the aggregation pipeline? Name three stages.**
**A:** A framework where documents pass through ordered transformation stages, each reshaping the stream. Common stages: `$match` (filter, ≈ WHERE), `$group` (≈ GROUP BY with accumulators like `$avg`/`$sum`), `$sort` (≈ ORDER BY); also `$limit`, `$project`, `$lookup`, `$unwind`.

**Q4. How is pagination done in MongoDB?**
**A:** Chain cursor methods: `.sort({...}).skip((page-1)*pageSize).limit(pageSize)` — e.g. page 2 with size 3 is `.skip(3).limit(3)`. Equivalent to SQL `LIMIT ... OFFSET ...` after `ORDER BY`.

---

## 6. JSON Databases

### 📌 What is a JSON database?

🔹 A database that stores, indexes and queries data in **JSON (document) form** — the shape you store ≈ the shape your app uses.
🔹 Kills the "impedance mismatch": no converting objects → rows → objects; you save the object, you get the object.
🔹 Examples: **MongoDB** (JSON-in, BSON-on-disk), Couchbase, Firebase Firestore, RethinkDB — and even **PostgreSQL**, which can store JSON natively via its `jsonb` type (Section 7). ✅

### 📌 JSON syntax refresher (objects, arrays, 6 types)

```json
{                                    // an OBJECT: unordered "key": value pairs inside { }
  "roll": 101,                       // NUMBER — JSON has ONE number type (no int vs float)
  "name": "Aarav",                   // STRING — keys and text values use "double quotes"
  "hosteler": true,                  // BOOLEAN — only true or false, lowercase
  "mentor": null,                    // NULL — an explicitly empty value
  "skills": ["SQL", "Python"],       // ARRAY — ordered list of values inside [ ]
  "address": {                       // nested OBJECT — objects may contain objects/arrays
    "city": "Delhi",                 // key : value pair of the nested object
    "pincode": "110001"              // last pair — NO trailing comma allowed anywhere in JSON
  }                                  // nested object closed
}                                    // root object closed
```

**Line-by-line walkthrough:**
- JSON's complete type list: **string, number, boolean, null, object, array** — that's all six.
- Keys must be double-quoted strings; single quotes are invalid JSON.
- No trailing commas, no comments in *real* JSON files — the `//` marks above are only for these notes. 📝

### 📌 JSON vs BSON

| Feature | JSON | BSON (MongoDB's format) |
|---------|------|--------------------------|
| Form | Plain text (UTF-8 string) | Binary encoding |
| Data types | 6 (string, number, bool, null, object, array) | Superset — adds `Date`, `ObjectId`, int32, int64, `Decimal128`, `BinData` |
| Parsing | Must read whole text to find one field | Length-prefixed → engine **skips** fields fast |
| Size | Compact for tiny docs | Slightly larger (type + length headers) |
| Human-readable | ✅ Yes | ❌ No |
| Role | Interchange format between apps/languages | MongoDB's on-disk & wire storage format |

> 📝 Flow to remember: *You write JSON in mongosh → driver converts to BSON → stored as BSON → converted back to JSON-looking output on read.*

### 🎯 Questions

**Q1. Is JSON itself a database?**
**A:** No — JSON is a lightweight text data-interchange format (objects, arrays, 6 value types). A "JSON database" is a database that uses JSON documents as its data model, e.g. MongoDB, Firestore, Couchbase.

**Q2. Compare JSON and BSON (four points).**
**A:** Form: text vs binary. Types: 6 vs superset (+Date/ObjectId/int64/Decimal128). Speed: full text parse vs length-prefixed skipping. Readability: human-readable vs machine-only. Size: JSON often smaller for tiny docs; BSON trades a little size for speed and rich types.

**Q3. Which BSON feature fixes JSON's biggest pain for a `date_of_birth` field?**
**A:** The native `Date` BSON type (millisecond precision). In pure JSON you'd fake it as a string and lose typing, comparisons and timezone handling.

---

## 7. JSON Representation of a Relational Dataset

### 📌 The relational starting point — `students` table (PostgreSQL)

| roll_no | name | age | city | cgpa | skills |
|---------|------|-----|------|------|--------|
| 101 | Aarav Sharma | 20 | Delhi | 8.7 | SQL, Python |
| 102 | Diya Nair | 21 | Mumbai | 9.1 | Java |
| 103 | Kabir Khan | 22 | Delhi | 7.4 | C++, Go |
| 104 | Meera Iyer | 20 | Pune | 8.9 | Python, ML |

### 📌 The same data as a JSON dataset

```json
[                                                    // a JSON dataset = an ARRAY of row-objects
  {                                                  // row 1 of the 'students' table
    "roll_no": 101,                                  // column roll_no becomes a key
    "name": "Aarav Sharma",                          // column name becomes a key
    "age": 20,                                       // column age becomes a key
    "city": "Delhi",                                 // column city becomes a key
    "cgpa": 8.7,                                     // column cgpa becomes a key
    "skills": ["SQL", "Python"]                      // multi-valued column → nested ARRAY
  },                                                 // rows are separated by commas
  { "roll_no": 102, "name": "Diya Nair",  "age": 21, "city": "Mumbai", "cgpa": 9.1, "skills": ["Java"] },      // row 2
  { "roll_no": 103, "name": "Kabir Khan", "age": 22, "city": "Delhi",  "cgpa": 7.4, "skills": ["C++", "Go"] }, // row 3
  { "roll_no": 104, "name": "Meera Iyer", "age": 20, "city": "Pune",   "cgpa": 8.9, "skills": ["Python", "ML"] } // row 4
]                                                    // closes the dataset array
```

**Line-by-line walkthrough:**
- **table → array**, **row → object**, **column → key**, **cell value → value**. That's the whole mapping. ✅
- The `skills` column (comma-separated in SQL, ugly to normalize) becomes a clean **nested array** — exactly why documents love semi-structured data.
- If each student had an `address` table, in JSON you'd simply **embed** it as a nested object — the join disappears.
- Any row could add an extra key (say `"email"`) without a schema change.

### 📌 PostgreSQL `jsonb` — JSON living inside a relational database

```sql
-- 1) CREATE a table with a JSONB column
CREATE TABLE student_profiles (       -- create a table for semi-structured student data
  roll_no  INT PRIMARY KEY,           -- regular typed column: unique roll number
  profile  JSONB NOT NULL             -- JSONB column: any valid JSON, stored in binary form
);                                    -- ends the DDL statement
```

```sql
-- 2) INSERT a JSON document (cast the text literal to jsonb)
INSERT INTO student_profiles (roll_no, profile) VALUES   -- insert one row
  (101, '{                                               -- start of the JSON text for roll 101
      "name":   "Aarav Sharma",                          -- attribute: name
      "age":    20,                                      -- attribute: age
      "city":   "Delhi",                                 -- attribute: city
      "skills": ["SQL", "Python"]                        -- attribute: array of skills
  }'::jsonb);                                            -- ::jsonb casts the text into jsonb type

INSERT INTO student_profiles VALUES                      -- a second, one-line row
  (102, '{"name":"Diya Nair","age":21,"city":"Mumbai","skills":["Java"]}'::jsonb);  -- cast and insert
```

```sql
-- 3) QUERY with -> and ->>
SELECT                                                   -- start of the SELECT list
  profile -> 'name'          AS name_json,               -- ->   returns JSON  (keeps quotes)
  profile ->> 'name'         AS name_text,               -- ->>  returns TEXT  (quotes removed)
  profile -> 'skills'        AS skills_json,             -- -> works on arrays/objects alike
  profile -> 'skills' ->> 0  AS first_skill,             -- -> with an INDEX reaches array element 0
  (profile ->> 'age')::INT   AS age                      -- ->> gives TEXT, cast back to INT for math
FROM student_profiles;                                   -- source table
```

```sql
-- 4) FILTER and AGGREGATE on JSON fields
SELECT profile ->> 'name' AS name                        -- project the name as plain text
FROM   student_profiles                                  -- from our table
WHERE  profile ->> 'city' = 'Delhi';                     -- ->> makes it TEXT so we can compare

SELECT profile ->> 'city' AS city,                       -- group key: the city inside JSON
       COUNT(*)             AS students                  -- count rows per city
FROM   student_profiles                                  -- source table
GROUP BY profile ->> 'city';                             -- GROUP BY the extracted text value
```

**Line-by-line walkthrough:**
- `->`  returns **jsonb** (use it to keep drilling: `profile -> 'address' ->> 'city'`).
- `->>` returns **text** (use it to compare with strings, display, or cast with `::INT`).
- Array access uses an integer index: `profile -> 'skills' ->> 0` → `"SQL"`.
- **`json` vs `jsonb`:** `json` stores the exact input text (slow, not indexable); `jsonb` stores parsed binary (fast, indexable with GIN) — **always prefer jsonb**. ✅
- Best practice: keep stable, heavily-filtered fields as real columns; keep variable attributes in the jsonb blob.

### 🎯 Questions

**Q1. Difference between `->` and `->>` in PostgreSQL?**
**A:** `->` extracts a field/element and returns **jsonb** (still quoted): `profile -> 'name'` → `"Aarav Sharma"`. `->>` returns **text** (unquoted): `profile ->> 'name'` → `Aarav Sharma`. Use `->` for further navigation, `->>` for comparisons, display and casts.

**Q2. Why is `jsonb` preferred over `json`?**
**A:** `jsonb` stores a parsed binary representation: faster to query, supports indexes (GIN), removes duplicate keys/whitespace. `json` keeps the raw text exactly as entered — preserving formatting but slow to search and not indexable.

**Q3. How do you map a relational table to JSON?**
**A:** Table → JSON array, row → JSON object, column → key, cell → value. Multi-valued or nested data becomes embedded arrays/objects instead of extra tables/joins.

---

## 8. Amazon DynamoDB

### 📌 What is DynamoDB?

🔹 AWS's fully-managed **NoSQL database** that works as both a **key-value** and a **document** store.
🔹 Delivers **single-digit millisecond** latency at *any* scale (it powered Amazon's Prime Day traffic) with automatic replication across 3 Availability Zones.
🔹 **Serverless** — no servers, patches, or shutdowns; you create a table and start writing.

### 📌 Data model: Tables → Items → Attributes

| Concept | Meaning | ≈ In SQL |
|---------|---------|----------|
| **Table** | Container of items; only key attributes are declared upfront | Table |
| **Item** | One record; up to **400 KB**; must contain the primary key | Row |
| **Attribute** | One name→value pair inside an item; can be scalar, set, or nested document | Column |
| **Schema** | Only the keys are fixed — items in the same table can have totally different attributes | Fixed columns |

### 📌 Partition key & Sort key (the heart of DynamoDB ⭐)

🔹 **Partition key** (hash key): its value is **hashed** to decide *which physical shard* stores the item.
🔹 **Simple primary key** = partition key alone → its value must be unique in the table.
🔹 **Composite primary key** = partition key + **sort key** (range key): the *pair* must be unique; items with the same partition key are **stored together and sorted by the sort key** → super-fast "all orders of this customer, newest first" queries.

Example table `Orders` — partition key `customer_id`, sort key `order_date`:

| customer_id (PK) | order_date (SK) | total | status |
|------------------|-----------------|-------|--------|
| C1 | 2026-01-10 | 2500 | delivered |
| C1 | 2026-02-01 | 899 | shipped |
| C2 | 2026-01-15 | 4300 | delivered |

### 📌 Capacity modes

| | 🔁 On-demand | 📊 Provisioned |
|---|---|---|
| You specify | Nothing — just write/read | **WCU** (1 write ≤1 KB/sec) and **RCU** (1 strongly-consistent read ≤4 KB/sec) per table |
| Scaling | Instant, automatic | Manual limits or autoscaling (reacts with a delay) |
| Billing | **Per request** | **Per reserved capacity-hour** |
| Best for | New/unpredictable/spiky workloads | Steady, well-known traffic (cheaper at scale) |

### 📌 When to choose DynamoDB

✅ Massive-scale key-value lookups: sessions, carts, gaming profiles, IoT metadata, leaderboards.
✅ You need predictable single-digit-ms latency and zero server management.
❌ Complex joins, ad-hoc analytical SQL, multi-entity relational integrity → use an RDBMS instead.

### 🎯 Questions

**Q1. Explain DynamoDB's data model.**
**A:** Tables contain **items** (≤400 KB each) made of **attributes** (scalars, sets, or nested documents). Only the primary key is fixed, so items are schema-flexible. It's simultaneously a key-value store (fetch by key) and a document store (query nested attributes).

**Q2. Partition key vs sort key?**
**A:** The **partition key** is hashed to distribute items across physical shards — it decides *where* data lives. The optional **sort key** orders items *within* the same partition and enables range queries (`BETWEEN`, `begins_with`). Together they form a composite primary key where only the pair must be unique.

**Q3. On-demand vs provisioned capacity — when would you pick each?**
**A:** **On-demand**: pay per request, scales instantly — ideal for new apps with unknown or spiky traffic. **Provisioned**: you pre-set RCU/WCU (autoscaling optional) — cheaper for steady, predictable loads; you risk throttling if you under-provision.

---

## 9. Serverless Cloud Databases

### 📌 What does "serverless" mean?

🔹 You **never provision, patch, or monitor servers** — the cloud provider runs everything invisibly.
🔹 The database **scales automatically** with load — often all the way **down to zero** when idle.
🔹 You pay **per actual use** (requests, compute-seconds, storage) instead of a fixed monthly instance price.
🔹 ⚠️ It does **not** mean "no servers exist" — it means the servers are *not your problem*.

### 📌 Popular serverless database services

| Service | Engine | What makes it special |
|---------|--------|------------------------|
| **Amazon Aurora Serverless v2** | MySQL/PostgreSQL-compatible | Relational DB that auto-scales capacity in fine-grained ACU units |
| **MongoDB Atlas** | MongoDB | Fully managed MongoDB — free tier, auto-sharding, backups, serverless/flexible tiers |
| **Neon** | PostgreSQL | True scale-to-zero Postgres with instant **database branching** (Git-like) |
| **PlanetScale** | MySQL (Vitess) | Horizontal scaling built on YouTube's Vitess; non-blocking schema changes |
| Others | — | Firebase Firestore, Cosmos DB, CockroachDB Serverless |

### 📌 Pros vs Cons

| ✅ Pros | ❌ Cons |
|---------|---------|
| Zero ops — no patching, backups, upgrades | **Cold starts** / connection overhead after idle |
| **Pay-per-use**, can scale to $0 when idle | Costs can spike with sudden heavy traffic |
| Built-in HA, replication, backups | Vendor lock-in (proprietary features) |
| Great for prototypes, startups, spiky apps | Limited control — tuning, extensions, OS access |

### 🎯 Questions

**Q1. What is a serverless database? Give two examples.**
**A:** A fully managed cloud database where the provider handles all infrastructure: it scales automatically (sometimes to zero) and bills per actual usage, not per provisioned instance. Examples: Amazon Aurora Serverless v2, Neon (serverless PostgreSQL), MongoDB Atlas, PlanetScale.

**Q2. Give two advantages and two disadvantages of serverless databases.**
**A:** ✅ No operations burden; pay-per-use with scale-to-zero. ❌ Cold starts after idle periods; costs can spike unpredictably under sustained heavy load; some vendor lock-in.

---

## 10. Index Creation & Performance Comparison using EXPLAIN

> 💡 An **index** is an extra sorted lookup structure (B-tree) pointing to rows — it turns a full-table scan O(n) into a tree lookup ≈ O(log n). Price: extra storage + slightly slower INSERT/UPDATE.

### 📌 PostgreSQL — BEFORE / AFTER demo

```sql
CREATE TABLE employees AS                         -- create + fill a table in one statement
SELECT id,                                        -- column 1: series number as id
       30000 + (id * 7) % 500000 AS salary,       -- column 2: salaries spread over ~500k values
       'emp' || id AS name                        -- column 3: names emp1, emp2, ...
FROM   generate_series(1, 500000) AS s(id);       -- half a million rows from a number generator

EXPLAIN                                           -- ask PostgreSQL HOW it plans to run...
SELECT * FROM employees                           -- ...this query
WHERE  salary = 275846;                           -- filter on the (un-indexed) salary column
-- Seq Scan on employees  (cost=0.00..9601.00 rows=1 width=17)   ← reads ALL 500,000 rows!
--   Filter: (salary = 275846)                                    ← every row is tested one by one

CREATE INDEX idx_emp_salary                       -- build a B-tree index on the salary column
ON employees (salary);                            -- target table and column

EXPLAIN                                           -- plan the SAME query again
SELECT * FROM employees
WHERE  salary = 275846;
-- Index Scan using idx_emp_salary on employees  (cost=0.42..8.45 rows=1 width=17)
--   Index Cond: (salary = 275846)                                ← tree search, ~3-4 page reads

EXPLAIN ANALYZE                                   -- actually EXECUTE and report TRUE timings
SELECT * FROM employees WHERE salary = 275846;
-- Index Scan ... (actual time=0.03..0.06 rows=1 loops=1)
-- Planning Time: ~0.1 ms   |   Execution Time: ~0.1 ms          ← vs ~60-100 ms for the Seq Scan
```

**Line-by-line walkthrough:**
- `EXPLAIN` shows the **plan without running** it; `EXPLAIN ANALYZE` **really executes** and appends `actual time`/`rows` — use ANALYZE to compare real speed. ✅
- **Reading the cost:** `cost=0.00..9601.00` = (startup cost .. total cost) in abstract units — only the *relative* numbers matter: 9601 vs 8.45 ≈ **1000x cheaper**.
- **`rows=1`** is the planner's row *estimate*; ANALYZE shows the real count beside it.
- **Seq Scan** = reads every row (fine for tiny tables or when the filter matches most rows).
- **Index Scan** = B-tree finds matching entries, then fetches just those rows from the heap.
- You may also see **Bitmap Index/Heap Scan** — an index-driven hybrid for when the index matches a moderate number of scattered rows. It is NOT a Seq Scan. 🔹

⚠️ The planner is smart — a low-selectivity query still Seq-Scans even with an index:

```sql
EXPLAIN SELECT * FROM employees WHERE salary > 30000;   -- matches ~100% of rows
-- Seq Scan on employees  (cost=0.00..9601.00 ...)      ← planner knows: reading all rows via
--   Filter: (salary > 30000)                           ← an index would be SLOWER, so it skips it
```

Rule of thumb: **index the columns you filter, join and sort on**; don't over-index — every index slows every write. Also run `ANALYZE employees;` after bulk loads so the planner has fresh statistics.

### 📌 MongoDB — COLLSCAN vs IXSCAN

```javascript
db.orders.find({ amount: { $gt: 900 } })          // query on a field with NO index yet
         .explain("executionStats")               // execute + return real performance stats
// BEFORE index:  winningPlan.stage: COLLSCAN     ← full COLLECTION SCAN (reads everything)
//   totalDocsExamined: 50000, nReturned: 1200    ← examined 50,000 docs to return 1,200
//   executionTimeMillis: 45                      ← wall-clock time in milliseconds

db.orders.createIndex({ amount: 1 })              // ascending B-tree index on 'amount' (-1 = desc)
db.orders.getIndexes()                            // list all indexes on the collection

db.orders.find({ amount: { $gt: 900 } })          // the SAME query as before
         .explain("executionStats")               // compare the new plan
// AFTER index:   winningPlan.stage: IXSCAN       ← INDEX SCAN via idx_amount
//   totalKeysExamined: 1200, totalDocsExamined: 1200   ← touched only matching docs!
//   executionTimeMillis: 2                       ← ~20x faster on the same data
```

**Line-by-line walkthrough:**
- `explain()` has 3 modes: `"queryPlanner"` (plan only), `"executionStats"` (plan + real numbers), `"allPlansExecution"` (compares candidate plans).
- **COLLSCAN** = PostgreSQL's Seq Scan; **IXSCAN** = Index Scan — the vocabulary changes, the idea is identical.
- Golden metric: `totalDocsExamined` should be **close to** `nReturned`; a huge gap means wasted work.
- Compound indexes: `createIndex({ city: 1, cgpa: -1 })` supports filters/sorts on `city` then `city+cgpa` (leftmost-prefix rule).
- Same trade-off as SQL: each extra index slows inserts/updates and eats RAM/disk.

### 🎯 Questions

**Q1. What does `cost=0.00..9601.00` mean in an EXPLAIN output?**
**A:** `startup cost..total cost` in the planner's abstract units (page fetches + CPU estimates). It is NOT milliseconds — compare plans *relatively*. `rows=` is the estimated number of matching rows, `width=` the average row size in bytes.

**Q2. Difference between EXPLAIN and EXPLAIN ANALYZE?**
**A:** `EXPLAIN` only *estimates and prints* the plan without executing. `EXPLAIN ANALYZE` **actually runs the query** and shows `actual time`, real `rows`, and loops per node — so you can measure true performance (at the cost of really executing a potentially slow query).

**Q3. What are COLLSCAN and IXSCAN in MongoDB?**
**A:** **COLLSCAN** — collection scan, every document examined (bad at scale, ≈ Seq Scan). **IXSCAN** — index scan using a B-tree index (examines only matching keys/docs, ≈ Index Scan). Check them via `db.coll.find({...}).explain("executionStats")` and watch `totalDocsExamined` vs `nReturned`.

**Q4. When would the planner rightly ignore your index?**
**A:** When the filter is unselective — e.g. `salary > 30000` matching nearly every row — because traversing the B-tree plus random heap fetches for most rows costs more than one sequential sweep. Indexes pay off when they *narrow down* data.

---

## 11. Vector Databases

### 📌 The embeddings idea

🔹 An **embedding** is a text/image/audio converted by an ML model into a **fixed-length array of numbers (a vector)** that captures its *meaning*.
🔹 Things with similar meaning land **close together** in vector space: `"movie"` ≈ `"film"` (near), `"movie"` vs `"sandwich"` (far).
🔹 A **vector database** stores millions of such vectors and answers "**which stored items are most similar to this query vector?**" using Approximate-Nearest-Neighbour indexes (HNSW, IVFFlat).

### 📌 Similarity = cosine of the angle between vectors

`cos(θ) = (A · B) / (‖A‖ × ‖B‖)` → **1** = same direction/meaning, **0** = unrelated, **-1** = opposite.
pgvector's `<=>` returns the **cosine distance = 1 − cosine similarity**, so `ORDER BY distance ASC` → smallest = most similar. ✅

### 📌 Use cases

🔹 **Semantic search** — match by meaning, not keywords ("affordable laptop" finds "budget notebook").
🔹 **RAG (Retrieval-Augmented Generation)** — fetch relevant documents for an LLM so it answers from *your* data.
🔹 **Recommendations** — "users/products similar to this one".
🔹 Duplicate detection, anomaly detection, image/audio similarity.

### 📌 pgvector demo (inside plain PostgreSQL!)

```sql
CREATE EXTENSION IF NOT EXISTS vector;            -- enable pgvector (once per database)

CREATE TABLE articles (                           -- a table that will hold embeddings
  id        SERIAL PRIMARY KEY,                   -- auto-increment primary key
  title     TEXT NOT NULL,                        -- document title
  content   TEXT,                                 -- raw text kept for display
  embedding VECTOR(3)                             -- vector column; 3 = fixed dimension
);                                                -- real apps use 384-1536 dims

INSERT INTO articles (title, content, embedding) VALUES      -- add three sample documents
  ('Intro to DBMS', 'tables, SQL, normalization', '[0.9, 0.1, 0.0]'::vector),  -- DBMS-themed vector
  ('Cooking pasta', 'boil, sauce, al dente',      '[0.0, 0.9, 0.1]'::vector),  -- food-themed vector
  ('Advanced SQL',  'joins, indexes, EXPLAIN',    '[0.8, 0.2, 0.1]'::vector);  -- near the first one

SELECT title,                                                   -- show each match's title
       embedding <=> '[0.85, 0.15, 0.05]'::vector AS distance   -- cosine distance to query vector
FROM articles                                                   -- search the whole table
ORDER BY embedding <=> '[0.85, 0.15, 0.05]'::vector             -- ASCENDING: smallest = most similar
LIMIT 2;                                                        -- top-2 nearest neighbours
-- → 'Advanced SQL' (closest), then 'Intro to DBMS'             ← food row ranked last: correct!
```

**Line-by-line walkthrough:**
- `CREATE EXTENSION vector` installs the pgvector types/operators — step one of every pgvector project.
- `VECTOR(3)` fixes the dimensionality: every embedding in the column must have exactly 3 numbers (all rows must share it, just like real models).
- In production the arrays come from an embedding model (`[0.0123, -0.456, ...]`, 384–1536 long) — 3 dims here just so we can see the numbers.
- `<=>` computes cosine **distance**; combined with `ORDER BY ... LIMIT k` it is a top-k semantic search.
- Sibling operators: `<->` (Euclidean L2), `<#>` (negative inner product); `<=>` is the default choice for text embeddings.
- For millions of rows add an ANN index: `CREATE INDEX ON articles USING hnsw (embedding vector_cosine_ops);` 🔹

### 🎯 Questions

**Q1. What is an embedding?**
**A:** A fixed-length numeric vector produced by an ML model that represents the *meaning* of text, an image, or audio. Similar meanings map to nearby points, so similarity search becomes simple geometry instead of keyword matching.

**Q2. What is a vector database and name three use cases.**
**A:** A database optimized to store high-dimensional embeddings and answer nearest-neighbour similarity queries (often via ANN indexes like HNSW). Use cases: semantic search, RAG for LLMs, recommendation systems, duplicate/anomaly detection.

**Q3. Why does the pgvector query use `ORDER BY ... ASC` with `<=>`?**
**A:** `<=>` returns cosine **distance** (1 − cosine similarity). Smaller distance = more similar, so ascending order puts the best match first; `LIMIT k` then gives the top-k nearest neighbours.

**Q4. How does PostgreSQL become a vector database?**
**A:** By loading the **pgvector** extension: `CREATE EXTENSION vector`, add a `VECTOR(n)` column, store embeddings, then query with `ORDER BY embedding <=> query LIMIT k` — optionally accelerated by HNSW/IVFFlat indexes.

---

## ⚡ Quick Revision

| Concept | One-line memory hook |
|---------|----------------------|
| NoSQL | "Not Only SQL" — distributed, flexible-schema, big-data databases |
| 4 types | Document (MongoDB) · Key-Value (Redis, DynamoDB) · Column-family (Cassandra) · Graph (Neo4j) |
| Why NoSQL | Horizontal scale-out + flexible schema + Big Data 3 Vs |
| MongoDB hierarchy | Server → **Database → Collection → Document → Field** |
| BSON | Binary JSON superset — adds Date, ObjectId, int64, Decimal128; length-prefixed = fast |
| `_id` / ObjectId | Mandatory unique key; 12 bytes = 4 timestamp + 5 random + 3 counter |
| ACID vs BASE | SQL: strict all-or-nothing & strong consistency; NoSQL: available + eventual consistency |
| CAP | Under partition, choose Consistency **or** Availability |
| jsonb operators | `->` returns JSON · `->>` returns TEXT · array index via `-> 'arr' ->> 0` |
| JSON vs BSON | JSON = text interchange, 6 types; BSON = binary storage, rich types, faster |
| DynamoDB | AWS key-value/document store; Table → Item (≤400 KB) → Attribute |
| DynamoDB keys | Partition key = hashed shard placement; Sort key = orders within partition |
| Capacity modes | On-demand = pay per request (spiky); Provisioned = RCU/WCU (steady) |
| Serverless DB | Provider runs everything; auto-scale (to zero); pay per use (Aurora Serverless, Atlas, Neon, PlanetScale) |
| Index | Sorted B-tree shortcut → O(log n); costs storage + write speed |
| Seq Scan / COLLSCAN | Read every row/document — slow at scale |
| Index Scan / IXSCAN | B-tree lookup → only matching rows/docs touched |
| EXPLAIN | Plan only; **EXPLAIN ANALYZE** also executes → real times |
| cost=0.00..9601 | startup..total in abstract units — compare relatively, not ms |
| Embedding | Meaning as a fixed-length vector; similar meaning = nearby vectors |
| Cosine similarity | 1 = same meaning; pgvector `<=>` = cosine *distance* → ORDER BY ASC |
| pgvector flow | `CREATE EXTENSION vector` → `VECTOR(n)` column → `ORDER BY emb <=> query LIMIT k` |

---

## 📝 Practice Set

### 🅰️ Section A — 2-mark questions

**A1. Define NoSQL databases.**
✅ *Answer:* A family of non-relational databases designed for distributed, large-scale, flexible-schema data — "Not Only SQL". They trade strict relational guarantees for horizontal scalability, schema flexibility and high availability, and come in document, key-value, column-family and graph flavours.

**A2. What is BSON? Name two types it adds over JSON.**
✅ *Answer:* BSON (Binary JSON) is MongoDB's binary, length-prefixed storage encoding — a superset of JSON. It adds `Date` (millisecond timestamps) and `ObjectId` (plus int32/int64, Decimal128, BinData), and can be traversed field-by-field without parsing the whole text.

**A3. What is `_id` in MongoDB?**
✅ *Answer:* The mandatory unique identifier of every document within its collection. If omitted, MongoDB generates an ObjectId — 12 bytes: 4-byte creation timestamp, 5-byte random value, 3-byte incrementing counter — giving unique, roughly time-ordered ids without a central coordinator.

**A4. Difference between `$set` and `$inc`?**
✅ *Answer:* `$set` assigns a literal value to a field (creating it if missing); `$inc` atomically adds a number to the current value. Use `$set` for general updates, `$inc` for counters — it avoids read-modify-write race conditions.

**A5. What is a partition key in DynamoDB?**
✅ *Answer:* The primary-key attribute whose value is hashed to decide which physical shard (partition) stores the item. It controls data distribution and lookup path; with an optional sort key it forms a composite key where the pair must be unique.

**A6. What is an embedding in the vector-database context?**
✅ *Answer:* A fixed-length vector of floats produced by an ML model that numerically represents the meaning of text/image/audio. Similar meanings sit close in vector space, enabling similarity search with cosine/L2 distance instead of keyword matching.

### 🅱️ Section B — 5-mark questions

**B1. Explain the four types of NoSQL databases with examples and use cases.**
✅ *Answer:* ① **Document stores** keep JSON/BSON documents with nested objects/arrays — MongoDB, CouchDB; used for catalogs, profiles, CMS. ② **Key-value stores** map a unique key to an opaque value — Redis, DynamoDB; used for caching, sessions, carts. ③ **Column-family stores** store wide, sparse rows grouped into column families, optimised for massive writes — Cassandra, HBase; used for IoT/time-series. ④ **Graph stores** keep nodes and typed edges with properties — Neo4j, Neptune; used for social networks, fraud detection, recommendations.

**B2. Compare SQL and NoSQL databases (any six points).**
✅ *Answer:* Data model: tables vs documents/key-value/columns/graph. Schema: fixed DDL vs dynamic per-record. Query language: standard SQL vs per-database languages. Joins: native vs avoided (embedded data, `$lookup`). Transactions/consistency: ACID & strong vs often BASE & eventual (single-doc atomic in MongoDB; multi-doc ACID in v4.0+). Scaling: vertical vs horizontal sharding. Examples: PostgreSQL/MySQL vs MongoDB/Cassandra/Redis/Neo4j.

**B3. Describe MongoDB's CRUD operations with syntax.**
✅ *Answer:* **C**reate: `db.c.insertOne({...})` / `db.c.insertMany([...])`. **R**ead: `db.c.find({filter}, {projection})` with operators `$gt`, `$in`, `$and`; one document via `findOne()`. **U**pdate: `db.c.updateOne({filter}, {$set: {...}})` / `updateMany`, with `$inc`, `$push`, `$pull`. **D**elete: `db.c.deleteOne({filter})` / `deleteMany({filter})`. All updates are atomic per document; collections are created implicitly.

**B4. Explain the aggregation pipeline with an example and its SQL equivalent.**
✅ *Answer:* Documents flow through ordered stages, each transforming the stream. `db.students.aggregate([{$match: {age: {$gte: 20}}}, {$group: {_id: "$city", avgCgpa: {$avg: "$cgpa"}, total: {$sum: 1}}}, {$sort: {avgCgpa: -1}}, {$limit: 5}])` = `SELECT city, AVG(cgpa), COUNT(*) FROM students WHERE age >= 20 GROUP BY city ORDER BY 2 DESC LIMIT 5;`. Stages: `$match` (WHERE), `$group` (GROUP BY + accumulators), `$sort` (ORDER BY), plus `$project`, `$limit`, `$lookup`, `$unwind`.

**B5. On-demand vs provisioned capacity in DynamoDB.**
✅ *Answer:* **On-demand:** no capacity settings, pay per request, scales instantly — best for new, unpredictable or spiky workloads. **Provisioned:** you set Write/Read Capacity Units (1 WCU = one 1 KB write/sec; 1 RCU = one strongly-consistent 4 KB read/sec), optional autoscaling — cheaper for steady, predictable traffic but can throttle if under-provisioned.

**B6. How do you read an EXPLAIN plan before and after adding an index?**
✅ *Answer:* `EXPLAIN SELECT ... WHERE salary = 275846;` on 500k rows shows `Seq Scan ... (cost=0.00..9601.00 rows=1)` — every row is read; `rows` is only an estimate. After `CREATE INDEX idx ON employees(salary);` the same query shows `Index Scan using idx (cost=0.42..8.45)` — a B-tree lookup, ~1000x cheaper in cost units. `EXPLAIN ANALYZE` confirms reality (`Execution Time: ~0.1 ms` vs ~80 ms). A `Bitmap Heap Scan` variant means the index found many scattered rows; an unselective filter (e.g. `salary > 30000`) rightly stays a Seq Scan.

### 🧪 Section C — Lab / hands-on

**C1. In mongosh: create `students`, insert 3 documents, print only `name` & `cgpa` of Delhi students with cgpa ≥ 8 sorted by cgpa desc, add skill "Git" to Aarav, count Mumbai students.**
✅ *Answer:*
```javascript
use college                                       // select (create) the college database
db.students.insertMany([                          // insert three sample documents at once
  { name: "Aarav", roll: 101, city: "Delhi",  cgpa: 8.7, skills: ["SQL"] },     // doc 1
  { name: "Diya",  roll: 102, city: "Mumbai", cgpa: 9.1, skills: ["Java"] },    // doc 2
  { name: "Kabir", roll: 103, city: "Delhi",  cgpa: 7.4, skills: ["C++"] }      // doc 3
])                                                // returns the generated _ids
db.students.find(                                 // the required read query
  { city: "Delhi", cgpa: { $gte: 8 } },           // implicit AND: city == Delhi AND cgpa >= 8
  { name: 1, cgpa: 1, _id: 0 }                    // projection: show name & cgpa, hide _id
).sort({ cgpa: -1 })                              // highest cgpa first
db.students.updateOne(                            // update exactly one document
  { name: "Aarav" },                              // filter: Aarav's document
  { $push: { skills: "Git" } }                    // append 'Git' to his skills array
)                                                 // matchedCount: 1, modifiedCount: 1
db.students.countDocuments({ city: "Mumbai" })    // → 1
```
Expected: Aarav (8.7) is returned by the find; his skills become `["SQL", "Git"]`; the count prints `1`.

**C2. In PostgreSQL: create `student_profiles` with a jsonb column, insert two rows, list names living in Delhi, and show each student's first skill.**
✅ *Answer:*
```sql
CREATE TABLE student_profiles (          -- table with one regular + one JSONB column
  roll_no INT PRIMARY KEY,               -- unique roll number as a normal column
  profile JSONB NOT NULL                 -- semi-structured attributes stored as jsonb
);                                       -- ends the DDL
INSERT INTO student_profiles VALUES      -- insert row 1
  (101, '{"name":"Aarav","city":"Delhi","skills":["SQL","Python"]}'::jsonb);  -- cast text to jsonb
INSERT INTO student_profiles VALUES      -- insert row 2
  (102, '{"name":"Diya","city":"Mumbai","skills":["Java"]}'::jsonb);          -- second document
SELECT profile ->> 'name' AS name        -- extract name as plain text
FROM   student_profiles                  -- scan the table
WHERE  profile ->> 'city' = 'Delhi';     -- ->> gives TEXT so it compares with 'Delhi' → Aarav
SELECT profile -> 'skills' ->> 0 AS first_skill,   -- array element 0 via -> then ->>
       profile ->> 'name'        AS name          -- and the student's name
FROM   student_profiles;                          -- → SQL | Aarav ; Java | Diya
```

**C3. Prove the effect of an index: create a 500k-row table, run EXPLAIN before and after indexing, note the plan change.**
✅ *Answer:*
```sql
CREATE TABLE big_t AS                      -- build + populate in one step
SELECT id, (id * 7) % 500000 AS code       -- near-unique codes across 500k rows
FROM   generate_series(1, 500000) AS s(id);-- half a million rows

EXPLAIN SELECT * FROM big_t WHERE code = 275846;   -- step 1: plan BEFORE any index
-- Seq Scan on big_t  (cost=0.00..8601.00 rows=1)  ← full scan of 500,000 rows

CREATE INDEX idx_big_code ON big_t (code);          -- step 2: build the B-tree index

EXPLAIN ANALYZE SELECT * FROM big_t WHERE code = 275846;  -- step 3: same query, AFTER index
-- Index Scan using idx_big_code  (cost=0.42..8.45 rows=1) ← tree lookup, ~3-4 pages
-- Execution Time: ~0.1 ms                                 ← vs ~60-100 ms for the Seq Scan
```
Report: plan changed **Seq Scan → Index Scan**, total cost **~8601 → ~8.45**, and ANALYZE confirms the wall-clock speedup.

**C4. With pgvector: enable it, create a table with a vector column, insert two embeddings, and return the single article most similar to `[0.1, 0.9, 0.2]`.**
✅ *Answer:*
```sql
CREATE EXTENSION IF NOT EXISTS vector;      -- load pgvector into the current database
CREATE TABLE notes (                        -- a small document store with embeddings
  id    SERIAL PRIMARY KEY,                 -- auto-increment id
  title TEXT NOT NULL,                      -- human-readable title
  emb   VECTOR(3)                           -- 3-dimensional embedding column
);                                          -- ends the DDL
INSERT INTO notes (title, emb) VALUES       -- insert two documents with embeddings
  ('DBMS basics', '[0.9, 0.1, 0.0]'::vector),  -- tech-themed vector
  ('Pasta recipe', '[0.1, 0.9, 0.2]'::vector); -- food-themed vector
SELECT title,                               -- show the winner's title
       emb <=> '[0.1, 0.9, 0.2]'::vector AS dist  -- cosine distance to the query vector
FROM notes                                  -- search everything
ORDER BY emb <=> '[0.1, 0.9, 0.2]'::vector  -- ascending: smallest distance first
LIMIT 1;                                    -- top-1 nearest neighbour → 'Pasta recipe'
```

---

> 🎓 **Unit 6 complete!** Revise the ⚡ table, then attempt the practice set without looking. Next up: Unit 7. Good luck! 🍀
