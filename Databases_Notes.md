# Databases for Backend Engineers (PostgreSQL): Complete Revision Notes

> **Source:** [YouTube: Databases (Backend from First Principles series, part 12)](https://www.youtube.com/watch?v=F7Vwp2Xo5Do&list=PLui3EUkuMTPgZcV0QhQrOcwMPcBCcd_Q1&index=12)
> **Length:** ~2h 45m
> **Previous videos:** [HTTP_Notes.md](HTTP_Notes.md) · [Routing_Notes.md](Routing_Notes.md) · [Serialization_Notes.md](Serialization_Notes.md) · [Auth_Notes.md](Auth_Notes.md) · [Validation_Notes.md](Validation_Notes.md) · [Handlers_Middleware_Context_Notes.md](Handlers_Middleware_Context_Notes.md) · [API_Design_Notes.md](API_Design_Notes.md)
> **Goal of the video:** The database concepts a backend engineer actually uses: why databases exist, DBMS, SQL vs NoSQL, why PostgreSQL, data types, migrations, schema design and relationships, seeding, the queries behind real APIs, parameterized queries, triggers and indexes.

> 💡 Diagrams use **Mermaid**. They render in GitHub, VS Code (with a Mermaid preview extension), Obsidian, Notion, and the Claude app. Anything marked **📌 Extra** was added to clarify a point and is not in the video.
>
> ⚠️ **Prerequisite (from the video):** SQL and Postgres **basics** (`CREATE TABLE`, `SELECT`, `ORDER BY`, `GROUP BY`, ...) are **not** taught here. Learn them first from any free course ("SQL basics", "Postgres basics", anywhere from 1 to 20 hours).

---

## 📑 Table of Contents

1. [TL;DR: The Whole Video in 15 Lines](#-tldr-the-whole-video-in-15-lines)
2. [Why Databases? Persistence](#1-why-databases-persistence-0000)
3. [What Counts as a "Database"?](#2-what-counts-as-a-database-0200)
4. [Disk-Based vs RAM-Based Storage](#3-disk-based-vs-ram-based-storage-0348)
5. [DBMS: Database Management Systems](#4-dbms-database-management-systems-0840)
6. [Why Not Just Text Files?](#5-why-not-just-text-files-1232)
7. [Relational (SQL) vs Non-Relational (NoSQL)](#6-relational-sql-vs-non-relational-nosql-1826)
8. [Why PostgreSQL?](#7-why-postgresql-2559)
9. [PostgreSQL Data Types](#8-postgresql-data-types-3245)
10. [Database Migrations](#9-database-migrations-5337)
11. [Designing the Schema (Project Management App)](#10-designing-the-schema-10143)
12. [Constraints and Referential Integrity](#11-constraints-and-referential-integrity)
13. [Relationships: 1-to-1, 1-to-Many, Many-to-Many](#12-relationships-1-to-1-1-to-many-many-to-many)
14. [Naming Conventions](#13-naming-conventions-11533)
15. [Seeding](#14-seeding-14017)
16. [Queries Behind Real APIs](#15-queries-behind-real-apis-14504)
17. [Parameterized Queries and SQL Injection](#16-parameterized-queries-and-sql-injection-15347)
18. [Dynamic Filtering, Sorting and Pagination in SQL](#17-dynamic-filtering-sorting-and-pagination-15952)
19. [Create and Update Queries](#18-create-and-update-queries-21139)
20. [Triggers (Auto-Updating `updated_at`)](#19-triggers-21709)
21. [Indexes](#20-indexes-21914)
22. [Cheat Sheet / Quick Revision](#-quick-revision-cheat-sheet)
23. [Self-Test Questions](#-self-test-questions)

---

## ⚡ TL;DR: The Whole Video in 15 Lines

1. **Persistence:** storing data so it **survives after the program stops** (your to-do list is still there when you reopen the app).
2. "Database" is broad (contacts list, localStorage, a text file), but in backend work it means a **disk-based DBMS**.
3. **Disk (HDD/SSD):** cheap, large, slower. Databases live here. **RAM:** fast, small, costly. Caches (**Redis**) live here.
4. **DBMS** responsibilities: **data organization, access (CRUD), integrity, security**.
5. **Text files fail** because of slow, error-prone **parsing**, **no structure**, and **no concurrency control** (lost updates).
6. **Relational (SQL):** tables, rows, columns, a **strict predefined schema**, **foreign keys**, strong integrity (Postgres, MySQL, SQL Server). Good for a **CRM**.
7. **Non-relational (NoSQL):** collections and documents, a **flexible schema** (MongoDB). Good for a **CMS**, but integrity moves into application code.
8. **Choose PostgreSQL:** open source, **SQL-standard compliant**, extensible, reliable and scalable, and excellent **JSON/JSONB** support (so you rarely need MongoDB).
9. **Data types:** prefer **`TEXT`** over `VARCHAR(255)`. Use **`DECIMAL`/`NUMERIC`** for money and **`REAL`/`DOUBLE`** when tiny inaccuracies don't matter. Prefer **`JSONB`** over `JSON`. Use **`TIMESTAMPTZ`**, **`UUID`**, and **enums**.
10. **Migrations:** versioned SQL files (**up** + **down**) applied by a tool (**dbmate**, golang-migrate) that tracks the version in a `schema_migrations` table. You get history, version control and rollback.
11. **Constraints:** `PRIMARY KEY` (= NOT NULL + UNIQUE), `NOT NULL` (use on ~70%+ of columns), `UNIQUE`, `CHECK`, `FOREIGN KEY` with **ON DELETE RESTRICT / CASCADE / SET NULL / SET DEFAULT**.
12. **Relationships:** **1-1** (the child's PK is also an FK to the parent), **1-many** (an FK column in the child), **many-many** (a **linking table** with a **composite primary key**).
13. **Queries:** `LEFT JOIN` + `to_jsonb()` to embed related data. **Always `ORDER BY`** (no implicit order). **Parameterized queries** stop **SQL injection**.
14. **Triggers:** run a function automatically, e.g. set `updated_at = NOW()` on every update.
15. **Indexes:** like a **book's index**, a lookup table from value to row location, so the DB avoids a full **sequential scan**. Index columns used in **WHERE, JOIN, ORDER BY** on frequently run queries. PKs are auto-indexed. Every index costs extra work on writes.

---

## 1. Why Databases? Persistence (00:00)

> **Persistence:** storing data in a way that it **survives even after the program that created it has stopped**.

**Example: a to-do app.** You add items and check some off, close the app, and reopen it later. Everything is **in the same state** you left it.

Without persistence, every launch would start empty and **all progress would be lost**. Persistence means the data is there in the expected state **after time has passed** and **across physical locations**.

---

## 2. What Counts as a "Database"? (02:00)

The term is **very broad**. In the simplest sense, **any structured storage** is a database:

| Example | Why it's a database |
|---|---|
| 📱 Your phone's **contacts list** | Structured, persistent storage |
| 🌐 Browser **localStorage / sessionStorage / cookies** (DevTools → Application) | Key-value storage, e.g. what the Excalidraw site stores |
| 📝 A plain **text file** of notes | The most basic persistent storage |

**Common pattern:** a **persistent system** that offers **CRUD**: **C**reate, **R**ead, **U**pdate, **D**elete.

> In backend work, "database" means something more specific: a **disk-based database**, covered next.

---

## 3. Disk-Based vs RAM-Based Storage (03:48)

```mermaid
flowchart LR
    CPU["🧠 CPU"] <-->|"very fast"| RAM["⚡ RAM (primary memory)<br/>8–128 GB typical<br/>💰 expensive, limited"]
    CPU <-->|"slower"| DISK["💽 Disk (secondary memory)<br/>HDD / SSD<br/>512 GB – 2 TB typical<br/>💵 cheap, abundant"]
    RAM --- R1["Redis, in-memory caches"]
    DISK --- D1["PostgreSQL, MySQL, MongoDB"]
```

| | **RAM (primary memory)** | **Disk (secondary memory: HDD/SSD)** |
|---|---|---|
| Speed | ⚡ **Very fast** | 🐢 Relatively slow (because of how data is stored and fetched) |
| Cost | 💰 Costly | 💵 **Cheap** |
| Typical size | 8 / 16 / 32 / 64 / max ~128 GB | 512 GB – 2 TB+ |
| Used by | **Caches**: Redis, in-memory caches | **Databases**: Postgres, MongoDB, ... |

**Why are databases disk-based?** Databases need **lots of space**, and a fair amount of speed can be traded for capacity. Disk gives **more capacity at a lower price**, at the cost of speed.

> 🔑 **For a backend engineer:** **caches (Redis) store data in RAM**, so they're fast. **Traditional databases (Postgres, MongoDB) store data on disk**, so they're big and cheap. How the data is laid out on disk is a deep topic you don't need for application-level work.

---

## 4. DBMS: Database Management Systems (08:40)

Storing data on disk isn't enough. You also need **efficient ways to retrieve, change and delete** potentially hundreds of GBs of data. **DBMS software** does that.

```mermaid
mindmap
  root((DBMS responsibilities))
    1 Data organization
      Store efficiently so CRUD is fast
    2 Access
      Create Read Update Delete
    3 Integrity
      Data is accurate, valid, consistent
      e.g. payment amount must be a number
    4 Security
      Protect from unauthorized access
      Users and roles
    Also
      Scaling
      Load balancing
```

| # | Responsibility | Meaning / example |
|---|---|---|
| 1 | **Data organization** | Organize data so fetching, updating and creating are efficient |
| 2 | **Access** | Provide **CRUD** operations |
| 3 | **Integrity** | Data is **accurate, valid, consistent, not corrupt**. E.g. an e-commerce `payment` column must only accept **numbers**, so inserting the string `"something"` must **fail**. |
| 4 | **Security** | Protect data from **unauthorized access** with **users and roles** |

---

## 5. Why Not Just Text Files? (12:32)

Before DBMSs, people stored data in text files. That runs into three problems:

| Problem | Explanation |
|---|---|
| **1. Parsing** | To find one customer you must write app code to **read the file, split lines, compare each field**. It's **slow**, especially in slower languages like JS, Python or Ruby compared with Rust, and **error-prone**: bugs can **corrupt data** or return wrong info. |
| **2. No structure** | Text files are **fluid** and accept anything. You **can't enforce** "this field only holds numbers", so there's no guaranteed **consistency**. |
| **3. Concurrency** | Two people editing the same value at once means **lost updates** and unpredictable results. |

### The concurrency problem, step by step

```mermaid
sequenceDiagram
    participant A as Person A (+20)
    participant F as Text file (amount = 40)
    participant B as Person B (−20)
    A->>F: read amount
    F-->>A: 40
    B->>F: read amount
    F-->>B: 40
    A->>A: 40 + 20 = 60
    B->>B: 40 − 20 = 20
    A->>F: write 60
    B->>F: write 20
    Note over F: Final = 20 (or 60, depending on timing)<br/>Correct answer should be 40! ❌<br/>One update is LOST
```

The final value depends on **CPU cycles and timing**: it might be 60 or 20, but never the correct 40. A DBMS has **concurrency control** to handle this correctly.

---

## 6. Relational (SQL) vs Non-Relational (NoSQL) (18:26)

```mermaid
flowchart TB
    DB["DBMS types"] --> R["🗃️ Relational (SQL)"]
    DB --> N["📄 Non-relational (NoSQL)"]
    R --> R1["Tables · rows · columns<br/>Strict predefined schema<br/>Relationships via foreign keys<br/>Strong data integrity<br/>Queried with SQL"]
    R --> R2["PostgreSQL · MySQL · SQL Server · SQLite"]
    N --> N1["Collections · documents<br/>Flexible schema (each doc can differ)<br/>Integrity enforced in app code"]
    N --> N2["MongoDB · DynamoDB"]
```

### Terminology map

| Relational (Postgres) | Non-relational (MongoDB) |
|---|---|
| **Table** | **Collection** |
| **Row** | **Document** |
| Every row has the **same structure** | Each document **can have a different structure** |
| Schema **defined beforehand** (columns + data types) | **No schema enforced** |

### Relational databases
- Data lives in **tables, rows, columns**.
- **Relationships** between tables use **foreign keys**.
- A **strict, predefined schema**: you must define every column and data type up front.
- **Benefit: data integrity.** At any moment you can rely on the state of your data: column types, relationships, consistency.
- Queried with **SQL (Structured Query Language)**.

### Non-relational databases
- **✅ Flexible schema:** great for **prototyping** and moving fast without designing a schema. You can push any data on the fly.
- **❌ Weaker integrity:** they often lack strong constraints and relationships, so it's **easier to introduce inconsistencies**. Integrity has to be enforced in **application code**, which is more complex and **error-prone**, since app code changes often and bugs slip in.

### Use-case examples

| Use case | Best fit | Why |
|---|---|---|
| **CRM** (customer relationship management): contacts, sales opportunities | **Relational (Postgres)** | Needs **accurate, consistent** data, strong integrity, **complex queries**, relationships between customers |
| **CMS** (content management system, e.g. Sanity) feeding a blog | **Non-relational (MongoDB)** | Content is **unstructured**: articles mix images, code blocks, YouTube embeds... You don't know the shape in advance. |

---

## 7. Why PostgreSQL? (25:59)

Among the many relational options (MySQL, Postgres, SQLite, SQL Server), all of which have similar features and scale, **Postgres is the sensible default**:

| # | Reason | Detail |
|---|---|---|
| 1 | **Open source and free** | Not proprietary. Companies can **self-host** it on their own premises. |
| 2 | **SQL-standard compliant** | Standard SQL runs the same way as on MySQL or SQL Server, so **migrating to another DB** later needs only minor changes |
| 3 | **Extensible** | The docs run to ~1,400 pages, the features cover nearly every SaaS use case, and there's a good **extension system** |
| 4 | **Reliable and scalable** | Well known for both |
| 5 | **Excellent JSON support** ⭐ | A native **JSON/JSONB** type with strong **indexing and querying**, so you **don't need MongoDB just for dynamic data**. Store CMS-style content in a JSONB column. |

> 💡 Articles claim MySQL is faster in some cases. **Until you're serving millions of users** and optimizing a specific bottleneck, it doesn't matter. Postgres is the first choice for many startups and big companies.
>
> **SQL** = the query **language**. **PostgreSQL** = the database **system** that runs SQL.

GUI tool used in the video: **TablePlus**.

---

## 8. PostgreSQL Data Types (32:45)

### 8.1 Integers and serial

| Type | Meaning |
|---|---|
| `SERIAL` | Auto-incrementing integer (1, 2, 3...), used when you **omit** the column on insert |
| `BIGSERIAL` | Same but with a **much larger max**. Preferred in production when the ID is the **primary key**. |
| `SMALLINT` < `INTEGER` < `BIGINT` | Integers with increasing **capacity**. Pick by need. |

> 📌 **Correction:** the transcript says "integer < smallint < bigint". The real order is **SMALLINT (2 bytes) < INTEGER (4 bytes) < BIGINT (8 bytes)**. `SERIAL` starts at **1**.

### 8.2 Exact vs floating-point numbers

**`DECIMAL(10, 2)` / `NUMERIC(10, 2)`** (identical):

```
 DECIMAL(10, 2)  →  max 10 digits TOTAL, exactly 2 after the decimal point
                    ✅ 12345678.90   (8 + 2 = 10 digits)
                    ❌ 123456789.90  (11 digits, too many)
```

| | **DECIMAL / NUMERIC** | **REAL / DOUBLE PRECISION / FLOAT** |
|---|---|---|
| Accuracy | ✅ **Exact** representation | ≈ Approximate (can differ slightly across systems) |
| Speed | Slower | ⚡ **Faster** to store and calculate |
| Use for | **Money, prices**: anything where accuracy is critical | Sizes, measurements, **scientific computation**, where tiny discrepancies (`567.8987` vs `567.8990`) don't matter |

> 🔑 **Rule:** **price → DECIMAL. Always.** Use floats only when small inaccuracies are harmless and speed matters.

### 8.3 Strings: CHAR vs VARCHAR vs TEXT

| Type | Behavior | Example: storing `"ab"` |
|---|---|---|
| `CHAR(10)` | **Fixed length**: **pads with spaces** to the full length | `"ab        "` (8 spaces added) |
| `VARCHAR(255)` | **Variable length up to a max**. Error if exceeded. | `"ab"` (length 2) |
| `TEXT` | Variable length, **no length limit** (practical cap of very large sizes) | `"ab"` |

**When to use which:**
- **`CHAR(n)`:** only when every value has the **same fixed length**, e.g. weekday codes `MO`, `TU`, `WE` → `CHAR(2)`. Otherwise avoid it (old standard).
- **`VARCHAR` vs `TEXT`:** the presenter says **always use `TEXT`**:
  1. The **Postgres docs recommend TEXT**. There's **no performance difference** in Postgres, and TEXT columns **can be indexed** (common myths come from MySQL, SQL Server and SQLite habits).
  2. **`VARCHAR(255)` is a MySQL convention.** In Postgres, 255 means nothing, and people copy it without thinking.
  3. Raising a length limit later needs a **risky database migration**.
  4. Arbitrary numbers **mislead new team members**, who go looking for a reason behind "255".
  5. Backend engineers reach the DB through a **driver** and application code, so **enforce length limits in the app's validation layer** (see [Validation_Notes.md](Validation_Notes.md)).

### 8.4 Other types

| Type | Stores |
|---|---|
| `BOOLEAN` | `true` / `false` |
| `DATE` | Date only |
| `TIME` | Time (hour:minute:second) |
| `TIMESTAMP` | Date + time |
| `TIMESTAMPTZ` | Date + time + **time zone** ✅ |
| `INTERVAL` | Durations: `10 days`, `1 week` |
| `UUID` | Native UUID. A popular **primary key** choice (URL-friendly, unique). |
| `JSON` | Stored as **plain text** |
| `JSONB` ✅ | **JSON Binary**: Postgres serializes it into its own **binary format** for **faster processing and querying**. Postgres-specific (not standard SQL). **Prefer it.** |
| Arrays | `INTEGER[]`, `TEXT[]`, `JSONB[]`... arrays of any type |
| Misc | IP / MAC addresses, geometric points, XML (rarely used, see the docs) |

```sql
-- Inserting: column order must match value order
INSERT INTO data_types_demo (small_num, big_num, price, is_active, created, meta)
VALUES (32767, 9223372036854775807, 19.99, true, NOW(), '{"key": "value"}');
```

---

## 9. Database Migrations (53:37)

### Why not just run SQL in a GUI?
In production you **can't** just open TablePlus and run queries. There'd be **no record** of what changed, when, or by whom, and **no version control** of the database state. That's why we use **migrations**.

### What migrations are

```
 db/
 └── migrations/
     ├── 20250101120000_create_users_table.sql     ← sequential (timestamp or 1, 2, 3...)
     ├── 20250102090000_seed_data.sql
     └── 20250103150000_add_indexes_triggers.sql
```

- **Files of plain SQL** (`CREATE TABLE`, `CREATE INDEX`, `DROP TABLE`...), named **sequentially** with numbers or **timestamps**, which naturally sort in order.
- A **CLI migration tool** (**dbmate**, **golang-migrate**...) runs them **in order** against the DB configured in your environment.

### Up and down migrations

```sql
-- migrate:up
CREATE TABLE users (...);          -- the change you want to make

-- migrate:down
DROP TABLE users;                  -- how to REVERT that change
```

| | **Up migration** | **Down migration** |
|---|---|---|
| Does | **Applies** changes (create tables, types, indexes) | **Reverts** the up migration |
| Why | Move the schema forward | **Roll back** to the previous state if something breaks in production |

> Some modern approaches skip down migrations, but up + down is the standard.

### How the tool tracks state

```mermaid
flowchart LR
    F1["1.sql ✅"] --> F2["2.sql ✅"] --> F3["3.sql ✅"] --> F4["4.sql ✅"] --> F5["5.sql 🆕"]
    T[("schema_migrations table<br/>version = 4")] -.->|"tool checks: apply<br/>only files after v4"| F5
```

- The tool creates a table (e.g. **`schema_migrations`**) holding the **current version**.
- When you add `5.sql`, it **only applies files newer than the current version**. Re-running old files would error ("table already exists").

### Benefits
1. **Track database changes over time.** Migration files are committed to **Git** alongside your code.
2. **Rollback** to a previous version if something goes wrong.

> 🔑 **Every** schema change (add/modify tables, indexes, triggers) gets a **new migration file**, because relational DBs enforce a strict schema.

### dbmate workflow from the video

```bash
# .env
DATABASE_URL="postgres://user:pass@localhost:5432/app?sslmode=disable"
```

```bash
dbmate new create_users_table
```

```bash
dbmate up
```

`dbmate new` creates `db/migrations/<timestamp>_create_users_table.sql`. `dbmate up` applies all pending migrations. Running `dbmate up` showed "Applied ... in Xms", and then all the tables, plus `schema_migrations`, appeared in TablePlus.

---

## 10. Designing the Schema (1:01:43)

Continuing the **project management platform** from [API_Design_Notes.md](API_Design_Notes.md).

### ER diagram

```mermaid
erDiagram
    USERS ||--o| USER_PROFILES : "1-to-1"
    USERS ||--o{ PROJECTS : "owns (1-to-many)"
    PROJECTS ||--o{ TASKS : "has (1-to-many)"
    USERS ||--o{ TASKS : "assigned_to (1-to-many)"
    USERS ||--o{ PROJECT_MEMBERS : "member of"
    PROJECTS ||--o{ PROJECT_MEMBERS : "has members"

    USERS {
        uuid id PK
        text email "NOT NULL UNIQUE"
        text full_name "NOT NULL"
        text password_hash "NOT NULL"
        timestamptz created_at
        timestamptz updated_at
    }
    USER_PROFILES {
        uuid user_id PK "also FK → users"
        text avatar_url "nullable"
        text bio "nullable"
        text phone "nullable"
        timestamptz created_at
        timestamptz updated_at
    }
    PROJECTS {
        uuid id PK
        text name "NOT NULL"
        text description "nullable"
        project_status status "default active"
        uuid owner_id FK "ON DELETE RESTRICT"
        timestamptz created_at
        timestamptz updated_at
    }
    TASKS {
        uuid id PK
        uuid project_id FK "ON DELETE CASCADE"
        text title "NOT NULL"
        text description
        int priority "1-5 CHECK, default 1"
        task_status status "default pending"
        date due_date
        uuid assigned_to FK "ON DELETE SET NULL"
        timestamptz created_at
        timestamptz updated_at
    }
    PROJECT_MEMBERS {
        uuid project_id PK "FK CASCADE"
        uuid user_id PK "FK CASCADE"
        member_role role "default member"
        timestamptz created_at
        timestamptz updated_at
    }
```

### The full up migration

```sql
-- migrate:up

-- ENUM TYPES ------------------------------------------------------------
CREATE TYPE project_status AS ENUM ('active', 'completed', 'archived');
CREATE TYPE task_status    AS ENUM ('pending', 'in_progress', 'completed', 'cancelled');
CREATE TYPE member_role    AS ENUM ('owner', 'admin', 'member');

-- USERS -----------------------------------------------------------------
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         TEXT NOT NULL UNIQUE,
    full_name     TEXT NOT NULL,
    password_hash TEXT NOT NULL,          -- never store plain-text passwords
    created_at    TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- USER PROFILES (1-to-1 with users) ---------------------------------------
CREATE TABLE user_profiles (
    user_id    UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    avatar_url TEXT,                      -- nullable: optional fields
    bio        TEXT,
    phone      TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- PROJECTS --------------------------------------------------------------
CREATE TABLE projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    description TEXT,
    status      project_status NOT NULL DEFAULT 'active',
    owner_id    UUID NOT NULL REFERENCES users ON DELETE RESTRICT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- TASKS (1-to-many with projects) -----------------------------------------
CREATE TABLE tasks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id  UUID NOT NULL REFERENCES projects ON DELETE CASCADE,
    title       TEXT NOT NULL,
    description TEXT,
    priority    INTEGER NOT NULL DEFAULT 1 CHECK (priority BETWEEN 1 AND 5),
    status      task_status NOT NULL DEFAULT 'pending',
    due_date    DATE,
    assigned_to UUID REFERENCES users ON DELETE SET NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- PROJECT MEMBERS (many-to-many linking table) ----------------------------
CREATE TABLE project_members (
    project_id UUID NOT NULL REFERENCES projects ON DELETE CASCADE,
    user_id    UUID NOT NULL REFERENCES users    ON DELETE CASCADE,
    role       member_role NOT NULL DEFAULT 'member',
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (project_id, user_id)     -- composite primary key
);

-- migrate:down
DROP TABLE IF EXISTS project_members;
DROP TABLE IF EXISTS tasks;
DROP TABLE IF EXISTS projects;
DROP TABLE IF EXISTS user_profiles;
DROP TABLE IF EXISTS users;
DROP TYPE IF EXISTS member_role;
DROP TYPE IF EXISTS task_status;
DROP TYPE IF EXISTS project_status;
```

> 📌 The video's code was pasted in and not fully visible. This is a faithful reconstruction of what was explained. The **down migration drops in reverse order** (tables before the types they use, and child tables before parents).

### Enums: why use them instead of TEXT?

```sql
CREATE TYPE project_status AS ENUM ('active', 'completed', 'archived');
```

| Benefit | Explanation |
|---|---|
| **1. Data integrity** | Inserting `'something'` into `status` gives a **database-level error**. (App code can also validate this, but the DB is a safety net.) |
| **2. Documentation** ⭐ (the presenter thinks this matters more) | Someone reading the migrations (debugging, **onboarding**) immediately sees the **allowed values**. With `TEXT`, they'd have to dig through application code to discover that only 3 values are valid. |

Usually give an enum column a **DEFAULT**, e.g. `status DEFAULT 'active'`, so the app doesn't have to send it.

### Standard columns on every table

| Column | Purpose |
|---|---|
| `id UUID PRIMARY KEY DEFAULT gen_random_uuid()` | Uniquely identifies each row. **Postgres generates it** if you omit it. |
| `created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP` | When the row was **created** (auto-filled) |
| `updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP` | When the row was **last updated** (kept fresh by a trigger, [§19](#19-triggers-21709)). E.g. sort users by last profile change. |

---

## 11. Constraints and Referential Integrity

**Constraints** are conditions on columns that **protect data from going corrupt**.

| Constraint | Meaning | Example |
|---|---|---|
| `PRIMARY KEY` | Uniquely identifies a row. **Implicitly `NOT NULL` + `UNIQUE`**. Auto-indexed. | `id UUID PRIMARY KEY` |
| `NOT NULL` | Value is required | `email TEXT NOT NULL` |
| `UNIQUE` | No two rows share the value. Duplicate → DB error. | `email ... UNIQUE` (no two users with the same email) |
| `CHECK (...)` | Custom condition that must be true | `CHECK (priority BETWEEN 1 AND 5)` (rejects `55`) |
| `DEFAULT ...` | Value used when none is given | `DEFAULT 'pending'` |
| `REFERENCES` (**foreign key**) | The value **must exist** in the referenced table's PK | `owner_id REFERENCES users` |

### NOT NULL by default

> 🔑 **Postgres columns are nullable by default.** **More than ~70% of your columns should be `NOT NULL`.** Otherwise bugs, including in **automated scripts**, can fill your DB with NULLs. Only leave a column nullable when it's genuinely **optional** (e.g. `bio`, `phone`, `description`).

### Foreign key constraint
`assigned_to UUID REFERENCES users` means you **can't** insert any random UUID, even a valid-looking one. It **must match an existing `users.id`**, or the insert fails. (`REFERENCES users` is shorthand for `REFERENCES users(id)`, since `id` is the PK.)

### Referential integrity: ON DELETE behaviors

Protecting data **across tables** using their relationships:

```mermaid
flowchart TB
    D["DELETE user (id = 1)<br/>who is referenced by other rows"] --> Q{"ON DELETE ...?"}
    Q -->|"RESTRICT"| R["❌ Delete FAILS<br/>'delete their projects first'<br/>(projects.owner_id)"]
    Q -->|"CASCADE"| C["🗑️ User deleted AND all<br/>referencing rows deleted<br/>(project → its tasks)"]
    Q -->|"SET NULL"| N["User deleted, referencing<br/>column set to NULL<br/>(tasks.assigned_to)"]
    Q -->|"SET DEFAULT"| SD["User deleted, column set<br/>to its DEFAULT value"]
```

| Behavior | Effect | Used in the schema for |
|---|---|---|
| `ON DELETE RESTRICT` | **Blocks** deleting the parent while children reference it | `projects.owner_id`: can't delete a user who owns projects |
| `ON DELETE CASCADE` | Deletes the parent **and all children** | `tasks.project_id`: deleting a project deletes its tasks. Also `project_members`. |
| `ON DELETE SET NULL` | Deletes the parent, sets the child's FK to **NULL** | `tasks.assigned_to`: the task stays, just unassigned |
| `ON DELETE SET DEFAULT` | Sets the FK to its **default** value | (Not used here) |

> ⚠️ `SET NULL` on a `NOT NULL` column → **DB error**, and the delete fails.
> `project_id ... NOT NULL` means **no orphan tasks**: every task must belong to a project.

---

## 12. Relationships: 1-to-1, 1-to-Many, Many-to-Many

```mermaid
flowchart LR
    subgraph O["1-to-1"]
      U1["users.id (PK)"] --- P1["user_profiles.user_id<br/>(PK AND FK)"]
    end
    subgraph M["1-to-many"]
      PR["projects.id (PK)"] --- T1["tasks.project_id (FK)"]
      PR --- T2["tasks.project_id (FK)"]
      PR --- T3["tasks.project_id (FK)"]
    end
    subgraph MM["many-to-many"]
      US["users"] --- L["project_members<br/>PK(project_id, user_id)"]
      PJ["projects"] --- L
    end
```

### 1-to-1: `users` ↔ `user_profiles`

**Why a separate table?** Profile data (avatar, bio, phone, and later social links, websites...) **changes often and keeps growing**. Keeping it apart:
- Avoids **constant migrations** on the core `users` table.
- Avoids **constantly modifying** user rows.
- Lets the profile table **grow over time** without affecting `users`.

The same pattern works for **user preferences, metadata**, and so on.

**How:** each user has **at most one** profile row, so **drop the separate `id`** and make **`user_id` both the PRIMARY KEY and the FOREIGN KEY**. (Name it `user_id`, not `id`, to show where it comes from.)

### 1-to-many: `projects` → `tasks`
One project has **many** tasks. **How:** put an **FK column** (`project_id`) in the child table that references the parent's PK. **Not** a primary key in the child.

### Many-to-many: `users` ↔ `projects`
**How to recognize it:** it makes sense **both ways**. A user can be in **many** projects, *and* a project can have **many** users.

**How:** a **linking table** (`project_members`):
- **No `id` column.**
- Two FKs: `project_id` → projects and `user_id` → users (both `ON DELETE CASCADE`).
- A **composite primary key `(project_id, user_id)`**.

```
 users: 1, 2, 3        projects: 1, 2, 3

 project_members
 ┌────────────┬─────────┬────────┐
 │ project_id │ user_id │ role   │
 ├────────────┼─────────┼────────┤
 │     2      │    1    │ member │   ← "user 1 is in project 2": ONE event,
 │     2      │    3    │ admin  │      can't be duplicated (composite PK =
 │     1      │    1    │ owner  │      UNIQUE + NOT NULL on the pair)
 └────────────┴─────────┴────────┘
```

- "User 1 is part of project 2" is a **single event** that **can't be duplicated**. The composite PK makes the pair **unique and not null**, and **identifies the row**.
- **Relationship-specific data** lives in the linking table. E.g. **`role`** (owner/admin/member) only means something *for this user in this project*.

---

## 13. Naming Conventions (1:15:33)

| Convention | Rule | Why |
|---|---|---|
| Table names | **Plural** (`users`, `projects`, `user_profiles`) | Industry standard (some teams use singular, so follow your team) |
| Case | **lowercase + snake_case** (`full_name`, `password_hash`, `created_at`) | Postgres is **case-insensitive**: `fullName` becomes `fullname` unless you write `"fullName"` with double quotes everywhere, which makes app code ugly |

```sql
CREATE TABLE users (fullName TEXT);    -- Postgres actually creates "fullname" 😬
CREATE TABLE users ("fullName" TEXT);  -- works, but now ALWAYS needs quotes
CREATE TABLE users (full_name TEXT);   -- ✅ snake_case, no quotes ever
```

> 📌 **Contrast:** **JSON API payloads use camelCase** ([API_Design_Notes.md](API_Design_Notes.md)) while **DB columns use snake_case**. Your app or ORM maps between them (e.g. `created_at` ↔ `createdAt`).

---

## 14. Seeding (1:40:17)

In production, data arrives through **real user flows** (sign-up forms and so on). During **development**, you need **test data** to try things out before deploying.

> **Seeding:** inserting **test data** into the database for **development/testing**.

**Best practice:** put it in a **separate migration file**.

```bash
dbmate new seed_data
```

```bash
dbmate up
```

The seed file used a **CTE (Common Table Expression)** for readability:

```sql
-- migrate:up
WITH inserted_users AS (
    INSERT INTO users (email, full_name, password_hash)
    VALUES
      ('john@example.com',  'John Doe',    'hash1'),
      ('jane@example.com',  'Jane Smith',  'hash2'),
      ('alice@example.com', 'Alice Brown', 'hash3')
    RETURNING id, email                       -- feed the next step
),
inserted_profiles AS (
    INSERT INTO user_profiles (user_id, avatar_url, bio, phone)
    SELECT id,
           'https://example.com/avatar.png',
           CASE email
             WHEN 'john@example.com' THEN 'Backend engineer'
             WHEN 'jane@example.com' THEN 'Product designer'
             ELSE 'Hello there'
           END,
           '+1-555-0100'
    FROM inserted_users
    RETURNING user_id
)
-- ... then projects, tasks, project_members similarly
SELECT 1;
```

---

## 15. Queries Behind Real APIs (1:45:04)

### `GET /v1/users`: all users with their profile embedded

**Goal:** return each user **with their profile inside it**, in **one API call and one DB query** (the frontend shouldn't need a second request).

```sql
SELECT u.*,
       to_jsonb(up.*) AS profile               -- profile row → JSON field
FROM users u                                   -- alias "u"
LEFT JOIN user_profiles up ON u.id = up.user_id
ORDER BY u.created_at DESC;                    -- newest first
```

```json
[
  {
    "id": "8f1e...", "email": "john@example.com", "full_name": "John Doe",
    "password_hash": "...", "created_at": "...", "updated_at": "...",
    "profile": { "user_id": "8f1e...", "avatar_url": "...", "bio": "Backend engineer", "phone": "..." }
  }
]
```

> 📌 **Extra:** in a real API, **never return `password_hash`**. Select the columns explicitly instead of `u.*`.

**Tips from the video:**

| Tip | Why |
|---|---|
| **Start writing from `FROM`**, not `SELECT` | Decide **where data comes from** first, then what to pick |
| Use **aliases** (`users u`, `user_profiles up`) | Easier to refer to tables repeatedly |
| **`LEFT JOIN`** here, not `INNER JOIN` | A user who never edited their profile has **no profile row**. `INNER JOIN` would **drop that user**. `LEFT JOIN` keeps all users (profile = null). |
| **`to_jsonb(up.*)`** | Built-in Postgres function that turns a row into JSON, here embedded as `profile` |
| **Always `ORDER BY`** | Relational DBs return rows in **no guaranteed order**. Default: `created_at DESC`. |

```mermaid
flowchart LR
    subgraph INNER["INNER JOIN"]
      I1["Only users WITH a profile row"]
    end
    subgraph LEFT["LEFT JOIN ✅"]
      L1["ALL users<br/>(profile = NULL if missing)"]
    end
```

> Inner joins and left joins are the joins you'll use **most of the time**.

**The full flow:** route → handler → service → **repository runs this query** → the result is **serialized to JSON** (Node: light overhead. Go: scan into a struct, then serialize) → frontend. See [Handlers_Middleware_Context_Notes.md](Handlers_Middleware_Context_Notes.md).

### `GET /v1/users/:userId`: a single user

Same query plus a **WHERE** with a **parameter**:

```sql
SELECT u.*, to_jsonb(up.*) AS profile
FROM users u
LEFT JOIN user_profiles up ON u.id = up.user_id
WHERE u.id = :user_id;          -- parameter slot (TablePlus syntax)
```

String values in SQL are wrapped in **single quotes**: `'8f1e...'`.

---

## 16. Parameterized Queries and SQL Injection (1:53:47)

> **Parameterized query:** a safety mechanism where you leave a **slot** in the query and supply the value separately. The DB treats that value **purely as data** (a string), **never as SQL code**. It's **escaped**.

```mermaid
flowchart TB
    subgraph BAD["❌ String concatenation"]
      B1["'SELECT * FROM users WHERE id = ' + userInput"]
      B2["userInput = &quot;1; DELETE FROM users&quot;"]
      B3["→ runs: SELECT ... WHERE id = 1; DELETE FROM users 💥"]
      B1 --> B2 --> B3
    end
    subgraph GOOD["✅ Parameterized"]
      G1["SELECT * FROM users WHERE id = $1"]
      G2["$1 = &quot;1; DELETE FROM users&quot;"]
      G3["→ treated as ONE harmless string value<br/>(no user matches, nothing deleted) ✅"]
      G1 --> G2 --> G3
    end
```

- If you **build queries by concatenating strings**, you're open to **SQL injection**: an attacker submits SQL (like `DELETE FROM users`) that gets **executed**.
- With parameters, whatever is in the slot is **just a value** with "no more power".
- **Always parameterize dynamic values** (user IDs, search terms, anything from the client).
- Your **DB driver or ORM** handles parameters in real code (Node, Go, Rust, Python...). TablePlus offers a parameter dialog to simulate this.

```js
// 📌 Extra: Node.js (pg) example
await db.query("SELECT * FROM users WHERE id = $1", [userId]);   // ✅
await db.query(`SELECT * FROM users WHERE id = '${userId}'`);   // ❌ SQL injection risk
```

---

## 17. Dynamic Filtering, Sorting and Pagination in SQL (1:59:52)

The list API from [API_Design_Notes.md](API_Design_Notes.md), now at the SQL level.

### Query params → defaults

| Param | Default | Notes |
|---|---|---|
| `page` | `1` | |
| `limit` | `10` (or 20) | |
| `letter` (filter: first letter of name) | *(none)* | If missing, **leave the WHERE clause out entirely** (built dynamically in code) |
| `sortBy` | `created_at` | **Only allow a whitelist**: `full_name`, `email`, `created_at` |
| `sortOrder` | `desc` | `asc` / `desc` |

### The query

```sql
SELECT u.*, to_jsonb(up.*) AS profile
FROM users u
LEFT JOIN user_profiles up ON u.id = up.user_id
WHERE u.full_name ILIKE :letter || '%'      -- filter: names starting with letter
ORDER BY u.email ASC                         -- sort: built from sortBy + sortOrder
LIMIT :limit OFFSET :offset;                 -- pagination
```

| Piece | Explanation |
|---|---|
| `ILIKE` | Like `LIKE`, but **case-insensitive** pattern matching |
| `:letter \|\| '%'` | `\|\|` **concatenates**. `'J' \|\| '%'` = `'J%'`, meaning **starts with J, anything after** |
| Demo results | `J` → John Doe and Jane Smith. `X` → no rows. `A` → Alice Brown. |
| `ORDER BY u.email` | The sort column must use the **same alias** as the query (`u.email`) |
| `LIMIT / OFFSET` | `LIMIT` = page size. `OFFSET` = rows to skip. |

### Page vs offset

```
 API (user-facing):  page = 1, 2, 3 ...          (starts at 1)
 DB:                 OFFSET = (page − 1) × limit  (starts at 0)

 limit = 1:   page 1 → OFFSET 0 → Jane Smith
              page 2 → OFFSET 1 → John Doe
```

> 📌 **Clarification:** the demo used `limit = 1`, so offset 0 and offset 1 happened to match "page index". The general formula is **`OFFSET = (page − 1) × limit`**.

> ⚠️ 📌 **Extra (important):** in real code you **can't parameterize identifiers** like column names or `ASC/DESC`. Placeholders only work for **values**. Build `ORDER BY` by picking from a **whitelist** (`{ email: "u.email", name: "u.full_name", createdAt: "u.created_at" }`) and **never** paste raw user input into the SQL. This is exactly why the video says to give users a **fixed set of sort options**.

```mermaid
flowchart LR
    Q["?letter=j&sortBy=email&sortOrder=asc&page=2&limit=1"] --> D["Apply defaults<br/>page=1, limit=10,<br/>sortBy=created_at, desc"]
    D --> W{"letter present?"}
    W -->|yes| AW["add WHERE full_name ILIKE $1 || '%'"]
    W -->|no| NW["no WHERE clause"]
    AW --> S["ORDER BY whitelisted column + direction"]
    NW --> S
    S --> P["LIMIT $n OFFSET (page-1)*limit"]
```

---

## 18. Create and Update Queries (2:11:39)

### `POST /v1/users`: create

```sql
INSERT INTO users (email, full_name, password_hash)
VALUES (:email, :full_name, :password_hash)     -- hash created in backend code
RETURNING *;                                    -- return the created row(s)
```

Demo: `'test@gmail.com'`, `'test test'`, a random hash → **one row returned**, and it appears in the `users` table.

### `PATCH /v1/users/:id`: partial update of the profile

The UI form lets users update **bio, phone, avatar_url**, any subset. **Only update the fields that were sent.** The backend checks which **allowed fields** are present and **builds the query dynamically**.

If only `bio` and `phone` were sent:

```sql
UPDATE user_profiles
SET bio   = :bio,
    phone = :phone                 -- avatar_url NOT touched
WHERE user_id = :user_id           -- from the URL path parameter
RETURNING *;
```

---

## 19. Triggers (2:17:09)

**Problem:** after the update, **`updated_at` still equals `created_at`**. It didn't change.

**Two fixes:**
1. **Manual:** add `updated_at = NOW()` to every UPDATE in app code (easy to forget).
2. **Trigger:** ✅ the DB does it automatically.

> **Trigger:** "**when** a condition happens (e.g. an UPDATE on a table), **automatically run** an action (a function)".

```sql
-- migrate:up
-- 1. A reusable function that stamps updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;   -- modify the row being updated
    RETURN NEW;                           -- return the new row
END;
$$ LANGUAGE plpgsql;

-- 2. Attach it to every table, BEFORE each UPDATE
CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_user_profiles_updated_at
    BEFORE UPDATE ON user_profiles
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ... same for projects, tasks (and project_members)

-- migrate:down
DROP TRIGGER IF EXISTS update_users_updated_at ON users;
DROP TRIGGER IF EXISTS update_user_profiles_updated_at ON user_profiles;
-- ...
DROP FUNCTION IF EXISTS update_updated_at_column();
```

- Triggers are **named**, so they're easy to **drop** in down migrations.
- **Result:** after `dbmate up`, running the profile update again (`'updated bio 3'`) **automatically set `updated_at` to the current time** ✅.

---

## 20. Indexes (2:19:14)

### The book-index analogy

```
 📖 BOOK INDEX                          🗄️ DATABASE INDEX
 ┌───────────────┬──────┐              ┌──────────────────┬─────────────────┐
 │ Chapter 1     │ p. 8 │              │ task id 3f2a...  │ disk location A │
 │ Chapter 2     │ p.29 │              │ task id 7b1c...  │ disk location F │
 │ Chapter 3     │ p.35 │              │ task id 9d4e...  │ disk location C │
 │ Chapter 4     │ p.54 │ ◄ jump here  │ ...              │ ...             │
 └───────────────┴──────┘              └──────────────────┴─────────────────┘
 No need to flip through 50 pages      No need to scan every row on disk
```

### Without an index: a sequential scan

- Rows are stored **all over the disk**, not sequentially (the GUI only *displays* them in order).
- `WHERE id = 'X'` means checking **row 1, row 2, row 3...** across different disk locations until one matches.
- With 6 rows that's fine. With **millions or billions**, it's **very slow**.

### With an index: a lookup table

- An index is (conceptually) a **lookup table**: **column value → location of the row on disk**.
- It's stored **compactly in one place**, so searching it is fast. Find the value, get the location, **jump straight there**.
- (Internally a **B-tree**. Research it if you're curious.)

```mermaid
flowchart LR
    Q["WHERE id = 'X'"] --> NI{"Index on id?"}
    NI -->|"No"| SEQ["🐢 Sequential scan<br/>check every row on disk<br/>one by one"]
    NI -->|"Yes"| IDX["⚡ Search compact index<br/>→ get disk location<br/>→ jump directly to row"]
```

### Index order (ASC / DESC)
An index has an **order**. If your API defaults to `ORDER BY created_at DESC`, create the index **on `created_at DESC`**. Then the data is already in the right order and no re-sorting is needed.

### When to create an index (rule of thumb)

> 🔑 Consider indexing a column if it appears in a **WHERE clause**, a **JOIN condition**, or an **ORDER BY**, **and** that query runs **frequently**.

- **Primary keys are indexed automatically.** Foreign keys are **not**, so index FKs you join on.
- ⚠️ **Indexes aren't free.** On every **INSERT/UPDATE** the DB must also **update the index**, which adds write overhead. Evaluate the trade-off: how frequent is the query, and what experience do you need? Start with the rule of thumb, **monitor**, and **drop indexes** that don't earn their keep.

### The index migration

```sql
-- migrate:up
CREATE INDEX idx_users_email         ON users(email);               -- lookup/join by email
CREATE INDEX idx_users_created_at    ON users(created_at DESC);     -- GET /users default sort
CREATE INDEX idx_tasks_project_id    ON tasks(project_id);          -- JOIN: tasks of a project
CREATE INDEX idx_tasks_assigned_to   ON tasks(assigned_to);         -- JOIN: tasks of a user
CREATE INDEX idx_tasks_created_at    ON tasks(created_at DESC);     -- GET /tasks default sort
CREATE INDEX idx_tasks_status        ON tasks(status);              -- WHERE status = 'pending'
CREATE INDEX idx_project_members_project_id ON project_members(project_id);
CREATE INDEX idx_project_members_user_id    ON project_members(user_id);

-- migrate:down
DROP INDEX IF EXISTS idx_users_email;
-- ... (drop each one)
```

| Index | Reason |
|---|---|
| `users(email)` | APIs find users by email, and joins match on email |
| `users(created_at DESC)` | "Get all users" sorts by newest first by default, and it's called often |
| `tasks(project_id)` | **FK used in a JOIN** (projects LEFT JOIN tasks). `projects.id` is already indexed as a PK. |
| `tasks(assigned_to)` | **FK used in a JOIN** (users ↔ tasks, "my tasks") |
| `tasks(created_at DESC)` | Default sort for task lists |
| `tasks(status)` | Used in a **WHERE** filter (`?status=pending`) |
| `project_members(project_id)`, `(user_id)` | Same reasoning: FKs used in joins and lookups |

> 📌 **Extra:** `users.email` already has a `UNIQUE` constraint, and Postgres creates an index for UNIQUE automatically, so a separate email index is redundant in practice. The video adds it to illustrate the "WHERE/JOIN column" reasoning.

---

## 🏁 Wrap-Up: What a Backend Engineer Does With Databases (2:43:56)

```mermaid
flowchart LR
    A["API request payload<br/>(params, query, body)"] --> B["Validate + set defaults"]
    B --> C["Build DYNAMIC SQL<br/>(whitelist columns)"]
    C --> D["Pass user values via<br/>PARAMETERIZED queries"]
    D --> E["Execute (indexes make it fast,<br/>triggers keep updated_at fresh)"]
    E --> F["Return data → serialize → JSON response"]
```

Writing the queries for every remaining API is left as **practice**, using the joins, indexes, triggers and parameterization covered here. In the presenter's words, this covers about **80%** of a backend engineer's day-to-day database work.

---

## 🧾 Quick Revision Cheat Sheet

| Topic | Key points |
|---|---|
| Persistence | Data survives after the program stops |
| Database (broad) | Any structured persistent storage with CRUD |
| Disk vs RAM | DBs on disk (cheap, big, slower). Caches (Redis) in RAM (fast, costly, small). |
| DBMS jobs | Organization · Access (CRUD) · Integrity · Security |
| Text files fail | Parsing (slow/error-prone) · no structure · no concurrency control (lost updates) |
| SQL vs NoSQL | Tables/rows/strict schema/FKs/integrity **vs** collections/documents/flexible schema/integrity in app code |
| CRM vs CMS | CRM → relational. CMS (unstructured content) → NoSQL, or **Postgres JSONB** |
| Why Postgres | Open source · SQL standard · extensible · reliable/scalable · **JSONB** |
| Numbers | `SMALLINT < INTEGER < BIGINT`. `BIGSERIAL` for auto IDs. **Money → `DECIMAL(p,s)`**. Floats when tiny errors are OK and speed matters. |
| Strings | `CHAR(n)` pads (fixed-length codes only). **Use `TEXT`** (no `VARCHAR(255)` habit). Enforce length in app validation. |
| Others | `BOOLEAN`, `DATE`, `TIME`, `TIMESTAMPTZ`, `INTERVAL`, `UUID`, **`JSONB` > `JSON`**, arrays |
| Migrations | Sequential SQL files, **up** (apply) + **down** (revert), tool (dbmate) + `schema_migrations` version table. Git history + rollback. |
| Enums | `CREATE TYPE x AS ENUM (...)`. Integrity + **self-documenting**. Add a DEFAULT. |
| Std columns | `id UUID PK DEFAULT gen_random_uuid()`, `created_at`, `updated_at` (`TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP`) |
| Constraints | `PRIMARY KEY` (= NOT NULL + UNIQUE), `NOT NULL` (on ~70%+ of columns), `UNIQUE`, `CHECK`, `DEFAULT`, FK `REFERENCES` |
| ON DELETE | `RESTRICT` (block) · `CASCADE` (delete children) · `SET NULL` · `SET DEFAULT` |
| 1-to-1 | Child PK = FK to parent (`user_profiles.user_id`) |
| 1-to-many | FK column in the child (`tasks.project_id`) |
| Many-to-many | Linking table, two FKs, **composite PK** (`project_members`), relationship data (role) lives there |
| Naming | Plural tables. **lowercase snake_case** (Postgres folds case) |
| Seeding | Test data for dev, in its own migration. CTEs + `RETURNING`. |
| Queries | Write `FROM` first, aliases, **LEFT JOIN** keeps parents without children, `to_jsonb(row)` embeds, **always ORDER BY** |
| Security | **Parameterized queries** → no SQL injection. **Whitelist** sort columns. |
| Pagination | `LIMIT limit OFFSET (page-1)*limit` |
| Filter | `ILIKE $1 \|\| '%'` = case-insensitive "starts with" |
| Create / update | `INSERT ... RETURNING *` · dynamic `UPDATE ... SET only-sent-fields ... WHERE id = $n RETURNING *` |
| Triggers | Function + `BEFORE UPDATE ... FOR EACH ROW` → auto `updated_at` |
| Indexes | Lookup table (B-tree) value → row location, which avoids sequential scans. Index **WHERE / JOIN / ORDER BY** columns on frequent queries. PKs auto-indexed, **FKs not**. Writes cost more. Match sort direction (`DESC`). |

---

## ❓ Self-Test Questions

<details>
<summary>1. Define persistence with an example.</summary>

Storing data so it **survives after the program stops**. Your to-do list is still there in the same state when you reopen the app.
</details>

<details>
<summary>2. Why are databases disk-based while Redis is RAM-based?</summary>

Disk is **cheap and large** but slower, which suits databases that need **lots of space**. RAM is **fast** but costly and small, which suits **caches**.
</details>

<details>
<summary>3. List the four core DBMS responsibilities.</summary>

**Data organization, access (CRUD), integrity, security.**
</details>

<details>
<summary>4. Two people edit amount = 40 in a text file at the same time (+20 and −20). What goes wrong?</summary>

Both read 40, one writes 60 and the other writes 20. The final value is **60 or 20 depending on timing** instead of the correct **40**: a **lost update**. Text files have no **concurrency control**.
</details>

<details>
<summary>5. Map these MongoDB terms to relational terms: collection, document.</summary>

Collection → **table**. Document → **row**.
</details>

<details>
<summary>6. Would you use relational or NoSQL for a CRM? For a CMS? Why?</summary>

**CRM → relational** (accurate, consistent data, complex queries and relationships). **CMS → NoSQL** (unstructured, unpredictable content), or Postgres **JSONB**.
</details>

<details>
<summary>7. Give five reasons to choose PostgreSQL.</summary>

Open source and free, SQL-standard compliant (easy migration), extensible, reliable and scalable, and excellent **JSON/JSONB** support.
</details>

<details>
<summary>8. Which type for a product price, and why not REAL?</summary>

**DECIMAL/NUMERIC**, which is exact. Floating-point types are approximate and can differ slightly across systems, which is unacceptable for money.
</details>

<details>
<summary>9. What happens when you store 'ab' in CHAR(10) vs VARCHAR(255) vs TEXT?</summary>

CHAR(10) **pads with 8 spaces**. VARCHAR and TEXT store just `'ab'`.
</details>

<details>
<summary>10. Why does the presenter recommend TEXT over VARCHAR(255)?</summary>

The Postgres docs recommend it, there's no performance difference, 255 is a meaningless **MySQL habit** that confuses readers, raising a limit later needs a risky migration, and length can be **validated in app code**.
</details>

<details>
<summary>11. JSON vs JSONB: which should you use, and why?</summary>

**JSONB.** It's stored in Postgres's **binary format**, which makes it faster to process and query. JSON is stored as plain text.
</details>

<details>
<summary>12. What are up and down migrations, and how does the tool know which files to run?</summary>

**Up** applies changes and **down** reverts them for rollback. The tool records the current **version** in a table like `schema_migrations` and only runs **newer** files.
</details>

<details>
<summary>13. Give two reasons to use an ENUM instead of TEXT for status.</summary>

**Data integrity** (invalid values error at the DB level) and **documentation** (allowed values are visible in the migrations, which helps onboarding and debugging).
</details>

<details>
<summary>14. What two constraints does PRIMARY KEY imply?</summary>

**NOT NULL** and **UNIQUE** (and it's automatically indexed).
</details>

<details>
<summary>15. Explain ON DELETE RESTRICT, CASCADE and SET NULL with the schema's examples.</summary>

**RESTRICT** (`projects.owner_id`): you can't delete a user who owns projects. **CASCADE** (`tasks.project_id`): deleting a project deletes its tasks. **SET NULL** (`tasks.assigned_to`): deleting a user leaves their tasks unassigned.
</details>

<details>
<summary>16. How do you implement 1-to-1, 1-to-many and many-to-many?</summary>

**1-1:** the child's PK is also an FK to the parent (`user_profiles.user_id`). **1-many:** an FK column in the child (`tasks.project_id`). **Many-many:** a **linking table** with two FKs and a **composite primary key** (`project_members(project_id, user_id)`).
</details>

<details>
<summary>17. Why keep user profiles in a separate table?</summary>

Profile data changes often and grows over time. Separating it avoids constant migrations and modifications on the core `users` table.
</details>

<details>
<summary>18. Why snake_case for Postgres columns?</summary>

Postgres **folds unquoted identifiers to lowercase**, so `fullName` becomes `fullname`. Keeping camelCase means quoting `"fullName"` everywhere.
</details>

<details>
<summary>19. Why use LEFT JOIN instead of INNER JOIN for users + profiles?</summary>

Users without a profile row would be **dropped** by INNER JOIN. LEFT JOIN keeps **all users**, with the profile as NULL.
</details>

<details>
<summary>20. Why must list queries always have ORDER BY?</summary>

Relational DBs return rows in **no guaranteed order**. The default is `ORDER BY created_at DESC`.
</details>

<details>
<summary>21. What is a parameterized query, and what attack does it prevent?</summary>

A query with **slots** whose values are passed separately and treated **only as data**, never as SQL. It prevents **SQL injection** (e.g. `1; DELETE FROM users`).
</details>

<details>
<summary>22. Write the WHERE clause for "names starting with a given letter, case-insensitive".</summary>

`WHERE u.full_name ILIKE $1 || '%'`
</details>

<details>
<summary>23. page = 3, limit = 20. What's the OFFSET?</summary>

`(3 − 1) × 20 = 40`
</details>

<details>
<summary>24. Why can't you pass sortBy straight into a SQL placeholder, and what do you do instead?</summary>

Placeholders only work for **values**, not identifiers or keywords. Map the user's choice through a **whitelist** of allowed columns and directions.
</details>

<details>
<summary>25. What does the updated_at trigger do?</summary>

A function sets `NEW.updated_at = CURRENT_TIMESTAMP` and is attached `BEFORE UPDATE ... FOR EACH ROW` on each table, so every update automatically refreshes `updated_at`.
</details>

<details>
<summary>26. Explain indexes with the book analogy. What happens without one?</summary>

Like a book's index (chapter → page), a DB index maps **value → row location on disk**, so the DB can jump straight to the row. Without one, it does a **sequential scan** of every row.
</details>

<details>
<summary>27. When should you add an index, and what's the cost?</summary>

When a column is used in **WHERE, JOIN or ORDER BY** on a **frequently run** query. The cost is extra work on every **INSERT/UPDATE** to keep the index current, so monitor and drop unused ones.
</details>

<details>
<summary>28. Why index tasks.project_id but not projects.id?</summary>

`projects.id` is a **primary key**, so it's indexed automatically. `tasks.project_id` is a **foreign key**, which isn't indexed by default, and it's used in joins.
</details>

---

*Notes compiled from the video transcript. Timestamps (mm:ss / h:mm:ss) let you jump back to each section. Transcription fixes: "dbate / d mate" → dbmate, "Whois / POG / hress" → Postgres, "care / Vare" → CHAR / VARCHAR, "uu ID" → UUID, "2 Json B" → to_jsonb, "I like" → ILIKE, "CT" → CTE, "OM" → ORM, "achieved" → archived. The schema SQL is reconstructed from the explanation, since the pasted code wasn't fully shown in the transcript.*
