# Normalization Guide for Relational Databases

A practical guide to database normalization forms with examples, anti-patterns, and pragmatic denormalization guidance.

## Table of Contents

1. [First Normal Form (1NF)](#first-normal-form-1nf)
2. [Second Normal Form (2NF)](#second-normal-form-2nf)
3. [Third Normal Form (3NF)](#third-normal-form-3nf)
4. [Boyce-Codd Normal Form (BCNF)](#boyce-codd-normal-form-bcnf)
5. [Common Anti-Patterns](#common-anti-patterns)
6. [When to Denormalize](#when-to-denormalize)

---

## First Normal Form (1NF)

**Rule:** Every column contains atomic (indivisible) values. No repeating groups or arrays.

### Anti-Pattern: Comma-Separated Values

```sql
-- WRONG: violates 1NF
CREATE TABLE product (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    tags VARCHAR(500)  -- stored as "electronics,sale,new-arrival"
);
```

**Problems:**
- Cannot index individual tags efficiently
- Cannot enforce referential integrity per tag
- Querying requires LIKE or string splitting
- No way to prevent duplicate tags per product

```sql
-- CORRECT: separate table for the multi-valued attribute
CREATE TABLE product (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE tag (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE product_tag (
    product_id BIGINT NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    tag_id     BIGINT NOT NULL REFERENCES tag(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);
```

### Anti-Pattern: Repeating Columns

```sql
-- WRONG: repeating groups
CREATE TABLE student (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    phone_1 VARCHAR(20),
    phone_2 VARCHAR(20),
    phone_3 VARCHAR(20)
);
```

```sql
-- CORRECT: separate table
CREATE TABLE student (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE student_phone (
    id         BIGSERIAL PRIMARY KEY,
    student_id BIGINT NOT NULL REFERENCES student(id) ON DELETE CASCADE,
    phone      VARCHAR(20) NOT NULL,
    phone_type VARCHAR(20) NOT NULL DEFAULT 'mobile',
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (student_id, phone)
);
```

### Anti-Pattern: JSON Blobs for Structured Data

```sql
-- WRONG: using JSON for well-known structured data
CREATE TABLE order_item (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    product_data JSONB  -- {"name": "Widget", "price": 9.99, "sku": "WDG-001"}
);
```

**When JSON IS acceptable:**
- Truly dynamic/schemaless data (user preferences, form builder responses)
- Data from external systems where the schema varies
- Metadata that doesn't need relational querying

**When JSON is NOT acceptable:**
- Data with a known, stable structure
- Data that needs to be filtered, joined, or aggregated frequently
- Data that needs referential integrity

---

## Second Normal Form (2NF)

**Rule:** Must be in 1NF AND every non-key column depends on the *entire* primary key (no partial dependencies).

**Only relevant for composite primary keys.** If your PK is a single column, the table automatically satisfies 2NF if it satisfies 1NF.

### Anti-Pattern: Partial Dependency

```sql
-- WRONG: student_name depends only on student_id, not on the full PK
CREATE TABLE enrollment (
    student_id   BIGINT NOT NULL,
    course_id    BIGINT NOT NULL,
    student_name VARCHAR(100) NOT NULL,  -- depends only on student_id
    course_name  VARCHAR(200) NOT NULL,  -- depends only on course_id
    enrolled_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    grade        DECIMAL(4,2),
    PRIMARY KEY (student_id, course_id)
);
```

```sql
-- CORRECT: each fact stored once
CREATE TABLE student (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE course (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL
);

CREATE TABLE enrollment (
    student_id  BIGINT NOT NULL REFERENCES student(id),
    course_id   BIGINT NOT NULL REFERENCES course(id),
    enrolled_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    grade       DECIMAL(4,2),
    PRIMARY KEY (student_id, course_id)
);
```

---

## Third Normal Form (3NF)

**Rule:** Must be in 2NF AND no non-key column depends on another non-key column (no transitive dependencies).

### Anti-Pattern: Transitive Dependency

```sql
-- WRONG: city and country depend on zip_code, not on id
CREATE TABLE customer (
    id       BIGSERIAL PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    zip_code VARCHAR(10) NOT NULL,
    city     VARCHAR(100) NOT NULL,   -- depends on zip_code
    country  VARCHAR(100) NOT NULL    -- depends on zip_code
);
```

```sql
-- CORRECT: extract the transitive dependency
CREATE TABLE zip_code (
    code    VARCHAR(10) PRIMARY KEY,
    city    VARCHAR(100) NOT NULL,
    country VARCHAR(100) NOT NULL
);

CREATE TABLE customer (
    id       BIGSERIAL PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    zip_code VARCHAR(10) NOT NULL REFERENCES zip_code(code)
);
```

### Anti-Pattern: Calculated/Derived Values

```sql
-- WRONG: total_price is derived from quantity * unit_price
CREATE TABLE order_item (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL,
    product_id  BIGINT NOT NULL,
    quantity    INTEGER NOT NULL,
    unit_price  NUMERIC(10,2) NOT NULL,
    total_price NUMERIC(10,2) NOT NULL  -- always = quantity * unit_price
);
```

**Options:**
1. Remove `total_price` and calculate on query: `SELECT quantity * unit_price AS total_price`
2. Use a generated/computed column (PostgreSQL 12+): `total_price NUMERIC(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED`
3. Accept denormalization with a CHECK constraint: `CHECK (total_price = quantity * unit_price)` — use when query performance matters

---

## Boyce-Codd Normal Form (BCNF)

**Rule:** Must be in 3NF AND every determinant (column that functionally determines another) is a candidate key.

BCNF differs from 3NF only in edge cases with overlapping composite candidate keys. In practice, most 3NF schemas are already BCNF.

### Example Where 3NF ≠ BCNF

A class scheduling system where:
- Each student takes one class per time slot
- Each class is taught by exactly one teacher
- Each teacher teaches exactly one class

```sql
-- 3NF but not BCNF: teacher → class is a functional dependency,
-- but teacher is not a candidate key
CREATE TABLE schedule (
    student_id BIGINT NOT NULL,
    time_slot  VARCHAR(20) NOT NULL,
    class_name VARCHAR(100) NOT NULL,
    teacher_id BIGINT NOT NULL,
    PRIMARY KEY (student_id, time_slot)
);
```

```sql
-- BCNF: decompose to remove the non-key determinant
CREATE TABLE class_teacher (
    class_name VARCHAR(100) PRIMARY KEY,
    teacher_id BIGINT NOT NULL UNIQUE
);

CREATE TABLE schedule (
    student_id BIGINT NOT NULL,
    time_slot  VARCHAR(20) NOT NULL,
    class_name VARCHAR(100) NOT NULL REFERENCES class_teacher(class_name),
    PRIMARY KEY (student_id, time_slot)
);
```

---

## Common Anti-Patterns

### 1. The "God Table"

A single table with 50+ columns covering multiple concerns.

**Symptoms:** Many nullable columns, groups of columns that are only filled for certain "types" of rows, column names with prefixes like `billing_`, `shipping_`, `primary_contact_`.

**Fix:** Decompose into focused tables with clear relationships.

### 2. Entity-Attribute-Value (EAV)

```sql
-- EAV anti-pattern
CREATE TABLE entity_attribute (
    entity_id   BIGINT NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    attr_name   VARCHAR(100) NOT NULL,
    attr_value  TEXT
);
```

**Problems:** No type safety, no constraints, impossible to query efficiently, no referential integrity.

**When EAV is acceptable:** Truly dynamic schemas where attributes are user-defined at runtime (CMS custom fields, product attributes in a catalog with thousands of categories). Even then, prefer JSONB columns in PostgreSQL.

### 3. Implicit Relationships (Missing Foreign Keys)

```sql
-- WRONG: no FK constraint — referential integrity not enforced
CREATE TABLE order_item (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,     -- looks like a FK but isn't
    product_id BIGINT NOT NULL    -- same problem
);
```

Always declare FK constraints explicitly. The database should enforce integrity, not the application.

### 4. Overusing VARCHAR for Everything

```sql
-- WRONG
CREATE TABLE event (
    id BIGSERIAL PRIMARY KEY,
    event_date VARCHAR(30),    -- should be TIMESTAMPTZ
    is_active VARCHAR(5),      -- should be BOOLEAN
    price VARCHAR(20),         -- should be NUMERIC
    quantity VARCHAR(10)       -- should be INTEGER
);
```

Use the correct data type. It enables validation, indexing, sorting, and arithmetic natively.

### 5. No Audit Trail

Tables with no `created_at`, `updated_at`, or any way to track when and how data changed. Always include at minimum:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

For regulated systems, add `created_by` and `updated_by` columns or a separate history/audit table.

---

## When to Denormalize

Normalization optimizes for data integrity and write operations. Denormalization optimizes for read performance. The decision is always a trade-off.

### Acceptable Denormalization Scenarios

| Scenario | Example | Trade-off |
|----------|---------|-----------|
| **Read-heavy reporting** | Materialized view with pre-joined data | Stale data (refresh lag) |
| **Caching calculated totals** | `order.total_amount` stored alongside items | Must update on item changes (use triggers or application logic) |
| **Reducing expensive JOINs** | Store `customer_name` on invoice for historical accuracy | Data duplication, but invoice reflects name at time of creation |
| **Search optimization** | Denormalized search table or full-text index | Extra storage, sync overhead |
| **Historical snapshots** | Copy product price to order_item at purchase time | Intentional — the price at time of purchase matters |

### Rules for Safe Denormalization

1. **Document it** — Add a comment explaining why the denormalization exists
2. **Protect consistency** — Use triggers, materialized views, or application-level sync to keep denormalized data in sync
3. **Keep the normalized source of truth** — Denormalized data is a cache, not the master
4. **Measure first** — Denormalize only after profiling proves the JOIN is the bottleneck
