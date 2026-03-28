# Database Design Patterns

Common design patterns for relational databases. Each pattern includes when to use it, implementation examples, and trade-offs.

## Table of Contents

1. [Polymorphism Patterns](#polymorphism-patterns)
2. [Hierarchy / Tree Patterns](#hierarchy--tree-patterns)
3. [Temporal Data Patterns](#temporal-data-patterns)
4. [Multi-Tenancy Patterns](#multi-tenancy-patterns)
5. [Soft Delete Patterns](#soft-delete-patterns)
6. [Audit Trail Patterns](#audit-trail-patterns)
7. [Status and State Machine Patterns](#status-and-state-machine-patterns)
8. [Lookup / Reference Table Patterns](#lookup--reference-table-patterns)
9. [Data Type Mapping Across Engines](#data-type-mapping-across-engines)

---

## Polymorphism Patterns

When different entity subtypes share common attributes but have type-specific fields.

### Pattern 1: Single Table Inheritance (STI)

All subtypes in one table. Type-specific columns are nullable.

```sql
CREATE TABLE notification (
    id          BIGSERIAL PRIMARY KEY,
    type        VARCHAR(20) NOT NULL,  -- 'email', 'sms', 'push'
    recipient   VARCHAR(255) NOT NULL,
    message     TEXT NOT NULL,
    -- Email-specific
    subject     VARCHAR(255),
    cc          VARCHAR(500),
    -- SMS-specific
    phone_code  VARCHAR(5),
    -- Push-specific
    device_token VARCHAR(255),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

| Pros | Cons |
|------|------|
| Simple queries, no JOINs | Many nullable columns |
| Easy to add new types | Cannot enforce NOT NULL per type at DB level |
| Good for few subtypes with few type-specific fields | Wastes space if subtypes diverge significantly |

**Use when:** ≤ 3-4 subtypes with few type-specific columns.

### Pattern 2: Class Table Inheritance (CTI)

Base table with shared fields + one table per subtype.

```sql
CREATE TABLE payment (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL REFERENCES customer_order(id),
    amount      NUMERIC(12,2) NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',
    type        VARCHAR(30) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payment_credit_card (
    payment_id      BIGINT PRIMARY KEY REFERENCES payment(id) ON DELETE CASCADE,
    card_last_four  CHAR(4) NOT NULL,
    card_brand      VARCHAR(20) NOT NULL,
    authorization_code VARCHAR(50)
);

CREATE TABLE payment_bank_transfer (
    payment_id      BIGINT PRIMARY KEY REFERENCES payment(id) ON DELETE CASCADE,
    bank_name       VARCHAR(100) NOT NULL,
    account_number  VARCHAR(30) NOT NULL,
    transfer_ref    VARCHAR(50)
);
```

| Pros | Cons |
|------|------|
| Clean separation, strong constraints per type | Requires JOIN to get full record |
| No nullable columns for type-specific fields | INSERT/UPDATE touches two tables |
| Easy to add new types without altering existing tables | Slightly more complex queries |

**Use when:** Subtypes have many distinct fields or different constraint rules.

### Pattern 3: Concrete Table Inheritance

One table per subtype, each containing all fields (shared + specific). No base table.

| Pros | Cons |
|------|------|
| No JOINs needed | Shared columns duplicated across tables |
| Can enforce all constraints per table | Hard to query "all payments" without UNION |
| Simple per-type queries | Changes to shared fields require altering all tables |

**Use when:** Subtypes are queried independently and rarely together.

---

## Hierarchy / Tree Patterns

### Pattern 1: Adjacency List

Simple parent reference. Best for shallow hierarchies.

```sql
CREATE TABLE category (
    id        BIGSERIAL PRIMARY KEY,
    name      VARCHAR(100) NOT NULL,
    parent_id BIGINT REFERENCES category(id) ON DELETE CASCADE,
    UNIQUE (parent_id, name)
);

CREATE INDEX idx_category_parent ON category(parent_id);
```

| Pros | Cons |
|------|------|
| Simple schema | Recursive queries needed for full tree (CTEs) |
| Easy insert/move | Getting all descendants is expensive without CTEs |

**Recursive query example (PostgreSQL/SQL Server):**
```sql
WITH RECURSIVE tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM category WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, t.depth + 1
    FROM category c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree ORDER BY depth, name;
```

### Pattern 2: Materialized Path

Store the full path from root as a string.

```sql
CREATE TABLE category (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    path VARCHAR(500) NOT NULL  -- e.g., '/1/5/23/'
);

CREATE INDEX idx_category_path ON category USING btree(path varchar_pattern_ops);
```

```sql
-- Find all descendants of category 5
SELECT * FROM category WHERE path LIKE '/1/5/%';

-- Find ancestors of category 23 (path = '/1/5/23/')
SELECT * FROM category WHERE '/1/5/23/' LIKE path || '%';
```

| Pros | Cons |
|------|------|
| Fast subtree queries with LIKE | Path must be updated when moving nodes |
| Easy to read depth/level | Path length limited by VARCHAR |

### Pattern 3: Closure Table

Stores all ancestor-descendant pairs. Best for deep hierarchies with frequent subtree queries.

```sql
CREATE TABLE category (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE category_closure (
    ancestor_id   BIGINT NOT NULL REFERENCES category(id) ON DELETE CASCADE,
    descendant_id BIGINT NOT NULL REFERENCES category(id) ON DELETE CASCADE,
    depth         INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (ancestor_id, descendant_id)
);

CREATE INDEX idx_closure_descendant ON category_closure(descendant_id);
```

| Pros | Cons |
|------|------|
| O(1) subtree and ancestor queries | More rows (N² worst case) |
| No recursive queries needed | Insert/move requires updating closure rows |
| Easy depth calculation | Extra table to maintain |

**Use when:** Deep hierarchies with frequent read queries for subtrees or ancestors.

---

## Temporal Data Patterns

### Pattern 1: Effective Date Range

Track when a value was active.

```sql
CREATE TABLE product_price (
    id           BIGSERIAL PRIMARY KEY,
    product_id   BIGINT NOT NULL REFERENCES product(id),
    price        NUMERIC(10,2) NOT NULL,
    effective_from TIMESTAMPTZ NOT NULL,
    effective_to   TIMESTAMPTZ,  -- NULL = currently active
    EXCLUDE USING gist (
        product_id WITH =,
        tstzrange(effective_from, effective_to) WITH &&
    )  -- PostgreSQL: prevents overlapping date ranges
);
```

```sql
-- Get current price
SELECT price FROM product_price
WHERE product_id = 1 AND effective_from <= NOW()
  AND (effective_to IS NULL OR effective_to > NOW());
```

### Pattern 2: History Table (Slowly Changing Dimension Type 2)

Keep a full history of changes to a record.

```sql
CREATE TABLE customer (
    id         BIGSERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,
    name       VARCHAR(100) NOT NULL,
    address    TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE customer_history (
    history_id   BIGSERIAL PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customer(id),
    email        VARCHAR(255) NOT NULL,
    name         VARCHAR(100) NOT NULL,
    address      TEXT,
    valid_from   TIMESTAMPTZ NOT NULL,
    valid_to     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    changed_by   VARCHAR(100)
);
```

**Populate via trigger** — on UPDATE of customer, INSERT the old values into customer_history.

### Pattern 3: Event Sourcing Table

Store every change as an immutable event. Reconstruct current state by replaying events.

```sql
CREATE TABLE account_event (
    id           BIGSERIAL PRIMARY KEY,
    account_id   BIGINT NOT NULL,
    event_type   VARCHAR(50) NOT NULL,  -- 'created', 'deposited', 'withdrawn'
    payload      JSONB NOT NULL,
    occurred_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version      INTEGER NOT NULL,
    UNIQUE (account_id, version)
);

CREATE INDEX idx_account_event_account ON account_event(account_id, version);
```

---

## Multi-Tenancy Patterns

### Pattern 1: Shared Schema with Tenant Column

All tenants share the same tables. Every table includes a `tenant_id`.

```sql
CREATE TABLE project (
    id        BIGSERIAL PRIMARY KEY,
    tenant_id BIGINT NOT NULL REFERENCES tenant(id),
    name      VARCHAR(200) NOT NULL,
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_project_tenant ON project(tenant_id);

-- Use Row-Level Security (PostgreSQL) for enforcement
ALTER TABLE project ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON project
    USING (tenant_id = current_setting('app.current_tenant')::BIGINT);
```

| Pros | Cons |
|------|------|
| Simple, cost-effective | Must remember tenant_id in every query |
| Easy cross-tenant reporting | Risk of data leak if filter is missed |
| Single migration path | Noisy neighbor performance risk |

### Pattern 2: Schema per Tenant

Each tenant gets its own database schema (namespace).

```sql
CREATE SCHEMA tenant_acme;
CREATE TABLE tenant_acme.project ( ... );

CREATE SCHEMA tenant_globex;
CREATE TABLE tenant_globex.project ( ... );
```

| Pros | Cons |
|------|------|
| Strong isolation | Migration must run per schema |
| No tenant_id needed in queries | Cross-tenant queries require UNION across schemas |
| Per-tenant backup/restore | More operational complexity |

### Pattern 3: Database per Tenant

Complete physical isolation. Each tenant has its own database.

**Use when:** Strict compliance requirements, large tenants with independent scaling needs, or tenants in different regions.

---

## Soft Delete Patterns

### Pattern 1: Deleted Timestamp

```sql
ALTER TABLE customer ADD COLUMN deleted_at TIMESTAMPTZ;
CREATE INDEX idx_customer_active ON customer(id) WHERE deleted_at IS NULL;

-- Application queries must always filter
SELECT * FROM customer WHERE deleted_at IS NULL;
```

### Pattern 2: Status Column

```sql
ALTER TABLE customer ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active';
-- status: 'active', 'inactive', 'deleted', 'archived'

CREATE INDEX idx_customer_status ON customer(status);
```

### Pattern 3: Archive Table

Move deleted records to a separate table.

```sql
CREATE TABLE customer_archive (LIKE customer INCLUDING ALL);
ALTER TABLE customer_archive ADD COLUMN archived_at TIMESTAMPTZ NOT NULL DEFAULT NOW();
ALTER TABLE customer_archive ADD COLUMN archived_by VARCHAR(100);
```

| Pattern | Pros | Cons |
|---------|------|------|
| Deleted timestamp | Simple, easy to restore | Every query needs filter |
| Status column | Flexible states, clear semantics | Every query needs filter |
| Archive table | Clean main table, no filter needed | More complex restore, FK issues |

**Recommendation:** Use deleted timestamp or status for simple cases. Use archive table when the main table's performance matters and deleted records are rarely accessed.

---

## Audit Trail Patterns

### Pattern 1: Audit Columns

Minimum audit — who and when.

```sql
CREATE TABLE invoice (
    id          BIGSERIAL PRIMARY KEY,
    -- ... business columns ...
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by  VARCHAR(100) NOT NULL,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by  VARCHAR(100) NOT NULL
);
```

### Pattern 2: Centralized Audit Log

A generic audit table that logs changes to any table.

```sql
CREATE TABLE audit_log (
    id           BIGSERIAL PRIMARY KEY,
    table_name   VARCHAR(100) NOT NULL,
    record_id    BIGINT NOT NULL,
    action       VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
    old_values   JSONB,
    new_values   JSONB,
    changed_by   VARCHAR(100) NOT NULL,
    changed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ip_address   INET
);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_changed_at ON audit_log(changed_at);
```

**Populate via triggers** or application-level event listeners.

### Pattern 3: Temporal Tables (SQL Server / PostgreSQL)

```sql
-- SQL Server system-versioned temporal table
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name NVARCHAR(100),
    salary DECIMAL(12,2),
    valid_from DATETIME2 GENERATED ALWAYS AS ROW START,
    valid_to   DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
) WITH (SYSTEM_VERSIONING = ON);
```

---

## Status and State Machine Patterns

### Pattern: Status with Transition Validation

```sql
CREATE TABLE order_status (
    code        VARCHAR(20) PRIMARY KEY,
    label       VARCHAR(50) NOT NULL,
    description TEXT,
    is_terminal BOOLEAN NOT NULL DEFAULT FALSE
);

INSERT INTO order_status VALUES
('pending', 'Pending', 'Order created, awaiting confirmation', FALSE),
('confirmed', 'Confirmed', 'Payment received', FALSE),
('shipped', 'Shipped', 'Order dispatched', FALSE),
('delivered', 'Delivered', 'Order received by customer', TRUE),
('cancelled', 'Cancelled', 'Order cancelled', TRUE);

-- Valid transitions
CREATE TABLE order_status_transition (
    from_status VARCHAR(20) NOT NULL REFERENCES order_status(code),
    to_status   VARCHAR(20) NOT NULL REFERENCES order_status(code),
    PRIMARY KEY (from_status, to_status)
);

INSERT INTO order_status_transition VALUES
('pending', 'confirmed'), ('pending', 'cancelled'),
('confirmed', 'shipped'), ('confirmed', 'cancelled'),
('shipped', 'delivered');
```

---

## Lookup / Reference Table Patterns

### Pattern 1: Simple Lookup Table

For stable, slowly-changing reference data (countries, currencies, status codes).

```sql
CREATE TABLE currency (
    code   CHAR(3) PRIMARY KEY,   -- ISO 4217
    name   VARCHAR(50) NOT NULL,
    symbol VARCHAR(5) NOT NULL,
    active BOOLEAN NOT NULL DEFAULT TRUE
);
```

### Pattern 2: CHECK Constraint

For very small, stable sets (< 5-6 values) that won't change.

```sql
ALTER TABLE customer_order
ADD CONSTRAINT chk_order_status
CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled'));
```

### Pattern 3: PostgreSQL ENUM Type

```sql
CREATE TYPE order_status AS ENUM ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled');

ALTER TABLE customer_order ALTER COLUMN status TYPE order_status USING status::order_status;
```

| Pattern | When to use |
|---------|-------------|
| Lookup table | Values change at runtime, need descriptions/metadata, referenced from multiple tables |
| CHECK constraint | < 6 values, very stable, no need for metadata |
| ENUM type | PostgreSQL only, stable values, cleaner than CHECK but hard to remove values |

---

## Data Type Mapping Across Engines

### Common Type Mappings

| Concept | PostgreSQL | MySQL | SQL Server | Oracle |
|---------|-----------|-------|------------|--------|
| Auto-increment int | `SERIAL` / `BIGSERIAL` | `INT AUTO_INCREMENT` | `INT IDENTITY(1,1)` | `GENERATED AS IDENTITY` |
| UUID | `UUID` | `CHAR(36)` / `BINARY(16)` | `UNIQUEIDENTIFIER` | `RAW(16)` |
| Boolean | `BOOLEAN` | `TINYINT(1)` | `BIT` | `NUMBER(1)` |
| Small text | `VARCHAR(n)` | `VARCHAR(n)` | `NVARCHAR(n)` | `VARCHAR2(n)` |
| Unlimited text | `TEXT` | `LONGTEXT` | `NVARCHAR(MAX)` | `CLOB` |
| Exact decimal | `NUMERIC(p,s)` | `DECIMAL(p,s)` | `DECIMAL(p,s)` | `NUMBER(p,s)` |
| Date only | `DATE` | `DATE` | `DATE` | `DATE` |
| Timestamp + TZ | `TIMESTAMPTZ` | `TIMESTAMP` (no TZ) | `DATETIMEOFFSET` | `TIMESTAMP WITH TIME ZONE` |
| Binary data | `BYTEA` | `LONGBLOB` | `VARBINARY(MAX)` | `BLOB` |
| JSON | `JSONB` | `JSON` | `NVARCHAR(MAX)` | `CLOB` + IS JSON |
| IP address | `INET` | `VARCHAR(45)` | `VARCHAR(45)` | `VARCHAR2(45)` |
| Array | `INT[]`, `TEXT[]` | Not supported | Not supported | `VARRAY` |

### Money / Currency Best Practices

**Never use FLOAT or DOUBLE for money.** Use exact numeric types:

```sql
-- PostgreSQL / MySQL / SQL Server
price     NUMERIC(12,2) NOT NULL,  -- up to 9,999,999,999.99
currency  CHAR(3) NOT NULL DEFAULT 'USD'

-- Alternative: store as cents (integer)
price_cents BIGINT NOT NULL  -- 999 = $9.99
```

### Timestamp Best Practices

- Always store timestamps in UTC (`TIMESTAMPTZ` in PostgreSQL)
- Convert to local time at the application/presentation layer
- Use `TIMESTAMPTZ` for events (when something happened)
- Use `DATE` for calendar dates (birthdays, deadlines) that have no time component
- Never store dates as strings
