# MongoDB Best Practices

A comprehensive reference for MongoDB-specific best practices covering indexing, schema validation, aggregation pipelines, sharding, Change Streams, and operational guidelines.

## Table of Contents

1. [Indexing Deep Dive](#indexing-deep-dive)
2. [Schema Validation](#schema-validation)
3. [Aggregation Pipeline Design](#aggregation-pipeline-design)
4. [Transactions](#transactions)
5. [Sharding](#sharding)
6. [Change Streams](#change-streams)
7. [Security Best Practices](#security-best-practices)
8. [Operational Guidelines](#operational-guidelines)

---

## Indexing Deep Dive

### Index Types

| Type | Syntax | Use Case |
|------|--------|----------|
| **Single field** | `{ email: 1 }` | Queries filtering/sorting on one field |
| **Compound** | `{ status: 1, createdAt: -1 }` | Queries filtering/sorting on multiple fields |
| **Multikey** | `{ tags: 1 }` (on array field) | Querying array elements |
| **Text** | `{ name: "text", description: "text" }` | Full-text search |
| **Hashed** | `{ userId: "hashed" }` | Hash-based sharding, equality queries only |
| **TTL** | `{ createdAt: 1 }, { expireAfterSeconds: 86400 }` | Auto-delete documents after N seconds |
| **Partial** | `{ email: 1 }, { partialFilterExpression: { active: true } }` | Index only documents matching a filter |
| **Unique** | `{ email: 1 }, { unique: true }` | Enforce uniqueness |
| **Wildcard** | `{ "metadata.$**": 1 }` | Query on arbitrary fields in a flexible sub-document |
| **Geospatial (2dsphere)** | `{ location: "2dsphere" }` | Geo queries ($near, $geoWithin) |

### The ESR Rule (Equality, Sort, Range)

When designing compound indexes, order fields as:

1. **Equality** — fields tested with `=` (exact match)
2. **Sort** — fields used in `.sort()`
3. **Range** — fields tested with `$gt`, `$lt`, `$in`, `$regex`

```javascript
// Query: find active products in a category, sorted by price
db.products.find({ active: true, categoryId: ObjectId("..."), price: { $gte: 10, $lte: 100 } })
  .sort({ price: 1 });

// Optimal index following ESR:
db.products.createIndex({
  active: 1,       // E — equality
  categoryId: 1,   // E — equality
  price: 1         // S+R — sort and range on the same field
});
```

### Covered Queries

A query is "covered" when all requested fields are in the index — MongoDB never reads the actual document.

```javascript
// Index covers this query entirely (no document fetch)
db.users.createIndex({ email: 1, status: 1 });
db.users.find({ email: "alice@example.com" }, { _id: 0, email: 1, status: 1 });
```

### Index Management Best Practices

- **Use `explain("executionStats")`** to verify queries use the expected index
- **Monitor index usage** with `db.collection.aggregate([{ $indexStats: {} }])` — drop indexes with zero ops
- **Hide before dropping** — use `db.collection.hideIndex("indexName")` to test the impact before removing
- **Background builds** — in MongoDB 4.2+, all index builds are hybrid (no long locks), but schedule large builds during low-traffic periods
- **Limit to ~10 indexes per collection** — each index adds write overhead and memory use
- **Use `collation`** on indexes for case-insensitive queries rather than `$regex` with `/i`

```javascript
// Case-insensitive index
db.users.createIndex(
  { email: 1 },
  { collation: { locale: "en", strength: 2 }, name: "idx_users_email_ci" }
);

// Query must use the same collation
db.users.find({ email: "ALICE@EXAMPLE.COM" }).collation({ locale: "en", strength: 2 });
```

---

## Schema Validation

### Validation Levels and Actions

| Setting | Value | Behavior |
|---------|-------|----------|
| `validationLevel` | `strict` | Validates all inserts AND updates |
| `validationLevel` | `moderate` | Validates inserts; validates updates only if the existing document already matches the schema |
| `validationAction` | `error` | Rejects the operation |
| `validationAction` | `warn` | Allows the operation but logs a warning |

**Migration strategy:** Start with `moderate` + `warn` on existing collections, then tighten to `strict` + `error` once all documents conform.

### Using `oneOf` for Polymorphic Validation

```javascript
db.createCollection("notifications", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["type", "recipientId", "createdAt"],
      properties: {
        type: { enum: ["email", "sms", "push"] },
        recipientId: { bsonType: "objectId" },
        createdAt: { bsonType: "date" }
      },
      oneOf: [
        {
          properties: { type: { const: "email" } },
          required: ["subject", "body"],
          properties: {
            subject: { bsonType: "string" },
            body: { bsonType: "string" },
            cc: { bsonType: "array", items: { bsonType: "string" } }
          }
        },
        {
          properties: { type: { const: "sms" } },
          required: ["phoneNumber", "message"],
          properties: {
            phoneNumber: { bsonType: "string", pattern: "^\\+[0-9]{7,15}$" },
            message: { bsonType: "string", maxLength: 160 }
          }
        },
        {
          properties: { type: { const: "push" } },
          required: ["title", "deviceToken"],
          properties: {
            title: { bsonType: "string" },
            body: { bsonType: "string" },
            deviceToken: { bsonType: "string" }
          }
        }
      ]
    }
  }
});
```

### Modifying Validation on Existing Collections

```javascript
// Update validation schema on an existing collection
db.runCommand({
  collMod: "users",
  validator: { $jsonSchema: { /* new schema */ } },
  validationLevel: "moderate"  // moderate during migration
});
```

---

## Aggregation Pipeline Design

### Pipeline Optimization Rules

1. **Filter early** — Put `$match` and `$limit` as early as possible to reduce documents flowing through the pipeline
2. **Project early** — Use `$project` or `$addFields` to drop unnecessary fields before expensive stages
3. **Leverage indexes** — `$match` at the start of a pipeline can use indexes; `$sort` after `$match` can use compound indexes
4. **Avoid $unwind on large arrays** — It multiplies the number of documents; use `$filter` or array operators when possible
5. **Use $facet for multiple aggregations** — Run parallel sub-pipelines on the same data in one pass

### Common Pipeline Patterns

#### Pagination with Total Count

```javascript
db.orders.aggregate([
  { $match: { userId: ObjectId("...") } },
  { $facet: {
    data: [
      { $sort: { orderDate: -1 } },
      { $skip: 20 },
      { $limit: 10 }
    ],
    totalCount: [
      { $count: "count" }
    ]
  }}
]);
```

#### Lookup with Pipeline (Left Join + Filter)

```javascript
db.orders.aggregate([
  { $lookup: {
    from: "users",
    let: { userId: "$userId" },
    pipeline: [
      { $match: { $expr: { $eq: ["$_id", "$$userId"] } } },
      { $project: { name: 1, email: 1 } }  // only fetch needed fields
    ],
    as: "customer"
  }},
  { $unwind: { path: "$customer", preserveNullAndEmptyArrays: true } }
]);
```

#### Computed Fields with $addFields

```javascript
db.orders.aggregate([
  { $addFields: {
    itemCount: { $size: "$items" },
    totalWithTax: { $multiply: ["$totalAmount", 1.21] },  // 21% tax
    daysSinceOrder: {
      $dateDiff: { startDate: "$orderDate", endDate: "$$NOW", unit: "day" }
    }
  }}
]);
```

---

## Transactions

### When to Use Multi-Document Transactions

- **Cross-collection atomic operations** — transferring funds between accounts, creating an order + decrementing inventory
- **When business logic requires all-or-nothing** — registration that creates a user + profile + default settings
- **NOT for single-document operations** — MongoDB already guarantees single-document atomicity

### Transaction Best Practices

```javascript
const session = client.startSession();
try {
  session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" },
    maxCommitTimeMS: 5000
  });

  await db.accounts.updateOne(
    { _id: fromAccountId, balance: { $gte: amount } },
    { $inc: { balance: -amount } },
    { session }
  );

  await db.accounts.updateOne(
    { _id: toAccountId },
    { $inc: { balance: amount } },
    { session }
  );

  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  await session.endSession();
}
```

**Guidelines:**
- Keep transactions short (< 60 seconds; default timeout is 60s)
- Minimize the number of documents touched per transaction
- Ensure indexes support the queries within the transaction to avoid slow operations
- If you find yourself using transactions frequently, reconsider your data model — you may need more embedding
- Retryable writes (`retryWrites=true` in connection string) handle transient errors automatically

---

## Sharding

### Shard Key Selection Criteria

| Criterion | Good | Bad |
|-----------|------|-----|
| **Cardinality** | High (userId, email) | Low (status, boolean) |
| **Distribution** | Even across values | Skewed (90% of writes to one value) |
| **Query isolation** | Queries include shard key | Queries scatter across all shards |
| **Write distribution** | Hashed or random | Monotonically increasing (timestamp, ObjectId) |
| **Immutability** | Value never changes after insert | Value changes frequently |

### Shard Key Patterns

```javascript
// Hashed shard key — even distribution, but range queries scatter
sh.shardCollection("mydb.orders", { userId: "hashed" });

// Ranged compound key — good for range queries + write distribution
sh.shardCollection("mydb.logs", { tenantId: 1, timestamp: 1 });

// Zone sharding — route data to specific shards by region
sh.addShardTag("shard-us-east", "US");
sh.addTagRange("mydb.users", { region: "US" }, { region: "US\uffff" }, "US");
```

### Pre-Splitting Chunks

For new sharded collections with hashed keys, pre-split to avoid initial hotspot:

```javascript
sh.shardCollection("mydb.events", { eventId: "hashed" }, false, { numInitialChunks: 64 });
```

---

## Change Streams

### Use Cases

- **Real-time sync** — keep a search index (Elasticsearch) or cache (Redis) in sync
- **Event-driven architecture** — react to database changes without polling
- **Audit logging** — capture all changes to critical collections
- **Cross-service sync** — update denormalized data in another service's database

### Basic Usage

```javascript
// Watch a single collection
const changeStream = db.orders.watch([
  { $match: { "fullDocument.status": "confirmed" } }
], { fullDocument: "updateLookup" });

changeStream.on("change", (event) => {
  console.log("Operation:", event.operationType);
  console.log("Document:", event.fullDocument);
  console.log("Resume token:", event._id);
});
```

### Resilient Change Stream (with resume token)

```javascript
let resumeToken = loadResumeTokenFromStorage();  // persist externally

function startChangeStream() {
  const options = resumeToken
    ? { resumeAfter: resumeToken, fullDocument: "updateLookup" }
    : { fullDocument: "updateLookup" };

  const stream = db.orders.watch([], options);

  stream.on("change", async (event) => {
    await processEvent(event);
    resumeToken = event._id;
    await saveResumeTokenToStorage(resumeToken);
  });

  stream.on("error", (error) => {
    console.error("Change stream error, restarting...", error);
    setTimeout(startChangeStream, 5000);
  });
}
```

**Guidelines:**
- Always persist the resume token to survive application restarts
- Use `fullDocument: "updateLookup"` for updates (otherwise you only get the changed fields)
- Filter in the pipeline (`$match`) to reduce events processed
- Change Streams require a replica set or sharded cluster (not standalone)

---

## Security Best Practices

### Authentication and Authorization

- **Always enable authentication** — never run production MongoDB without auth
- **Use SCRAM-SHA-256** (default in 4.0+) or x.509 certificates
- **Create specific roles per application** — don't use the `root` role for application connections
- **Principle of least privilege** — each application user gets only the permissions it needs

```javascript
// Create a restricted application user
db.createUser({
  user: "orderService",
  pwd: "...",
  roles: [
    { role: "readWrite", db: "orders_db" },
    { role: "read", db: "products_db" }
  ]
});
```

### Field-Level Encryption (FLE)

For sensitive data (PII, financial data), use Client-Side Field-Level Encryption:

- **Deterministic encryption** — allows equality queries on encrypted fields (SSN lookup)
- **Random encryption** — stronger security but no queries possible (medical records)

### Network Security

- **Enable TLS/SSL** for all connections (`--tlsMode requireTLS`)
- **Bind to specific IPs** — never bind to `0.0.0.0` in production
- **Use VPC peering / private endpoints** for cloud deployments

---

## Operational Guidelines

### Connection String Best Practices

```
mongodb://user:pass@host1:27017,host2:27017,host3:27017/mydb
  ?replicaSet=rs0
  &readPreference=secondaryPreferred
  &w=majority
  &retryWrites=true
  &retryReads=true
  &maxPoolSize=50
  &connectTimeoutMS=10000
  &socketTimeoutMS=30000
```

| Option | Recommendation | Why |
|--------|---------------|-----|
| `w=majority` | Always for writes | Ensures write is acknowledged by majority of replica set |
| `retryWrites=true` | Always | Automatically retries transient write errors |
| `retryReads=true` | Always | Automatically retries transient read errors |
| `readPreference` | `primary` for consistency; `secondaryPreferred` for read scaling | Trade-off between freshness and load distribution |
| `maxPoolSize` | 50-100 per application instance | Tune based on concurrency; too high wastes connections |

### Read/Write Concerns

| Level | Read Concern | Write Concern | Use Case |
|-------|-------------|---------------|----------|
| **Highest consistency** | `linearizable` | `{ w: "majority", j: true }` | Financial transactions |
| **Strong consistency** | `majority` | `{ w: "majority" }` | Most applications |
| **Eventual consistency** | `local` | `{ w: 1 }` | Logging, metrics |

### Backup Strategy

- **Use `mongodump` / `mongorestore`** for logical backups (small-medium databases)
- **Use filesystem snapshots** for large databases (ensure journaling is enabled)
- **MongoDB Atlas** — automated backups with point-in-time recovery
- **Test restores regularly** — a backup that can't be restored is not a backup

### Monitoring Key Metrics

| Metric | Warning Threshold | Action |
|--------|-------------------|--------|
| **Connections** | > 80% of max | Increase pool size or add instances |
| **Replication lag** | > 10 seconds | Check secondary load, network |
| **Page faults** | Increasing trend | Working set exceeds RAM — add memory or optimize queries |
| **opcounters** | Sudden spike | Investigate query patterns |
| **document/index size ratio** | Indexes > 50% of data | Review indexes, drop unused ones |
| **slow queries** | > 100ms | Check indexes, optimize queries |

### Production Checklist

- [ ] Replica set with 3+ members (odd number for election quorum)
- [ ] Authentication enabled with strong passwords
- [ ] TLS/SSL enabled for all connections
- [ ] Write concern `majority` for critical operations
- [ ] Automated backups with tested restore procedure
- [ ] Monitoring and alerting configured
- [ ] Log rotation configured (`--logRotate`)
- [ ] Connection pooling configured in application
- [ ] Indexes verified with `explain()` for all critical queries
- [ ] Schema validation on critical collections
- [ ] Change Streams resume tokens persisted externally
- [ ] WiredTiger cache size tuned (default: 50% of RAM - 1GB)
