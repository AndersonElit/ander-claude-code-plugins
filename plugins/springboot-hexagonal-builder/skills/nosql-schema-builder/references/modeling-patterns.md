# NoSQL Document Modeling Patterns

A practical guide to document modeling patterns for MongoDB and other document databases. Each pattern includes when to use it, implementation examples, and trade-offs.

## Table of Contents

1. [Embedding Patterns](#embedding-patterns)
2. [Referencing Patterns](#referencing-patterns)
3. [Polymorphism Patterns](#polymorphism-patterns)
4. [Tree and Hierarchy Patterns](#tree-and-hierarchy-patterns)
5. [Versioning Patterns](#versioning-patterns)
6. [Time-Series and Bucketing Patterns](#time-series-and-bucketing-patterns)
7. [Common Anti-Patterns](#common-anti-patterns)

---

## Embedding Patterns

### Pattern 1: Inline Embedding (1:1)

Embed a related object directly within the parent document.

```javascript
// User with embedded profile — always read/written together
{
  _id: ObjectId("..."),
  email: "alice@example.com",
  profile: {
    firstName: "Alice",
    lastName: "Smith",
    bio: "Software developer",
    avatarUrl: "https://cdn.example.com/avatar/alice.jpg"
  },
  createdAt: ISODate("2025-01-15T10:00:00Z")
}
```

| Pros | Cons |
|------|------|
| Single read to get all data | Cannot query/index embedded object independently |
| Atomic updates | If embedded object is shared, leads to duplication |
| Natural representation of tightly coupled data | — |

**Use when:** Data is always accessed together, has the same lifecycle, and belongs to only one parent.

### Pattern 2: Array Embedding (1:Few)

Embed an array of related objects within the parent.

```javascript
// Product with embedded reviews (bounded — max 50 per product page)
{
  _id: ObjectId("..."),
  name: "Mechanical Keyboard",
  price: NumberDecimal("89.99"),
  reviews: [
    {
      userId: ObjectId("..."),
      userName: "Bob",  // denormalized for display
      rating: 5,
      comment: "Great keyboard!",
      createdAt: ISODate("2025-02-01T14:30:00Z")
    },
    {
      userId: ObjectId("..."),
      userName: "Carol",
      rating: 4,
      comment: "Good quality, a bit loud",
      createdAt: ISODate("2025-02-03T09:15:00Z")
    }
  ],
  reviewCount: 2,  // maintained for quick access
  averageRating: 4.5
}
```

| Pros | Cons |
|------|------|
| Single read for product + reviews | Array must stay bounded |
| Reviews are displayed with the product | Adding a review requires updating the parent document |
| Atomic operations ($push, $pull) | Difficult to query reviews across all products |

**Use when:** The array is bounded (you can define a reasonable maximum), items are always read with the parent, and items don't need to be queried independently across parents.

### Pattern 3: Subset Pattern (1:Many, display optimization)

Embed only the most recent/relevant subset; store the full set in a separate collection.

```javascript
// Product with the 5 most recent reviews embedded
{
  _id: ObjectId("..."),
  name: "Mechanical Keyboard",
  recentReviews: [
    // only the latest 5 reviews for the product page
    { userId: ObjectId("..."), userName: "Eve", rating: 5, createdAt: ISODate("...") },
    { userId: ObjectId("..."), userName: "Dave", rating: 4, createdAt: ISODate("...") }
    // ... up to 5
  ],
  reviewCount: 238,
  averageRating: 4.3
}

// Full reviews in their own collection (for pagination, search, etc.)
// reviews collection
{
  _id: ObjectId("..."),
  productId: ObjectId("..."),
  userId: ObjectId("..."),
  userName: "Eve",
  rating: 5,
  comment: "The full review text...",
  createdAt: ISODate("...")
}
```

| Pros | Cons |
|------|------|
| Product page loads fast (no JOIN, subset is small) | Must keep the subset in sync when new reviews are added |
| Full review history is still queryable | Two writes per new review (main collection + embedded subset) |

**Use when:** You display a summary/subset on the main page and the full list on a separate page.

### Pattern 4: Extended Reference Pattern

Store a copy of frequently accessed fields from a referenced document to avoid lookups.

```javascript
// Order with extended reference to the customer
{
  _id: ObjectId("..."),
  customer: {
    _id: ObjectId("..."),       // reference to users collection
    name: "Alice Smith",         // denormalized — avoids lookup for order list display
    email: "alice@example.com"   // denormalized
  },
  items: [ /* ... */ ],
  totalAmount: NumberDecimal("149.99"),
  status: "shipped"
}
```

| Pros | Cons |
|------|------|
| No $lookup needed for common displays | Must update denormalized fields if the source changes |
| Fast reads for listing pages | Stale data if sync fails |

**Use when:** You frequently display a few fields from a related document and can tolerate brief staleness or have a sync mechanism in place.

---

## Referencing Patterns

### Pattern 1: Child Reference (1:Many, unbounded)

Store the parent's ID in the child document.

```javascript
// users collection
{ _id: ObjectId("user1"), name: "Alice" }

// orders collection — references user
{
  _id: ObjectId("order1"),
  userId: ObjectId("user1"),   // reference to parent
  items: [ /* ... */ ],
  totalAmount: NumberDecimal("59.99"),
  status: "delivered"
}
```

```javascript
// Query: find all orders for a user
db.orders.find({ userId: ObjectId("user1") }).sort({ orderDate: -1 });
```

| Pros | Cons |
|------|------|
| No document size growth on the parent | Requires a separate query (or $lookup) to get orders with user data |
| Child documents are independently queryable | Two queries to get user + orders |
| Natural for unbounded 1:N | — |

**Use when:** The "many" side grows without bound, children are queried independently, or children have their own lifecycle.

### Pattern 2: Parent Reference Array (M:N)

Store an array of referenced IDs in one or both sides.

```javascript
// products collection
{
  _id: ObjectId("prod1"),
  name: "Mechanical Keyboard",
  categoryIds: [ObjectId("cat1"), ObjectId("cat3")]  // M:N reference
}

// categories collection
{
  _id: ObjectId("cat1"),
  name: "Electronics",
  description: "Electronic devices and accessories"
}
```

```javascript
// Query: find all products in a category
db.products.find({ categoryIds: ObjectId("cat1") });

// Query: find categories for a product
const product = db.products.findOne({ _id: ObjectId("prod1") });
db.categories.find({ _id: { $in: product.categoryIds } });
```

| Pros | Cons |
|------|------|
| Simple, no junction collection needed | Array must stay bounded (hundreds, not millions) |
| Multikey index covers queries efficiently | Updating both sides requires two operations |

**Use when:** M:N relationship where the array size is bounded (a product has tens of categories, not millions).

### Pattern 3: Junction Collection (M:N with attributes)

When the relationship itself carries data, use a junction collection.

```javascript
// enrollments collection (junction between students and courses)
{
  _id: ObjectId("..."),
  studentId: ObjectId("student1"),
  courseId: ObjectId("course1"),
  enrolledAt: ISODate("2025-03-01T00:00:00Z"),
  grade: "A",
  status: "completed"
}
```

```javascript
// Index for both access patterns
db.enrollments.createIndex({ studentId: 1, courseId: 1 }, { unique: true });
db.enrollments.createIndex({ courseId: 1 });
```

| Pros | Cons |
|------|------|
| Relationship attributes live naturally here | Extra collection to manage |
| Both sides can be queried efficiently | Requires $lookup or application-level joins |
| No array size limits | — |

**Use when:** The M:N relationship has its own attributes or the number of relationships per entity is very large.

---

## Polymorphism Patterns

### Pattern 1: Single Collection Polymorphism

Different document shapes in one collection, discriminated by a type field.

```javascript
// notifications collection — all types in one collection
// Email notification
{
  _id: ObjectId("..."),
  type: "email",
  recipientId: ObjectId("..."),
  subject: "Welcome!",
  body: "<h1>Welcome to our platform</h1>",
  cc: ["admin@example.com"],
  sentAt: ISODate("...")
}

// SMS notification
{
  _id: ObjectId("..."),
  type: "sms",
  recipientId: ObjectId("..."),
  phoneNumber: "+1234567890",
  message: "Your code is 123456",
  sentAt: ISODate("...")
}

// Push notification
{
  _id: ObjectId("..."),
  type: "push",
  recipientId: ObjectId("..."),
  title: "New message",
  body: "You have a new message",
  deviceToken: "abc123...",
  sentAt: ISODate("...")
}
```

```javascript
// Partial index per type for type-specific queries
db.notifications.createIndex(
  { phoneNumber: 1 },
  { partialFilterExpression: { type: "sms" }, name: "idx_sms_phone" }
);
```

| Pros | Cons |
|------|------|
| All notifications queryable together | Validation schema must accommodate all types (use `oneOf`) |
| Natural for feeds, timelines, activity logs | Type-specific indexes may be sparse |
| Flexible schema — new types need no migration | — |

**Use when:** You query across types frequently (e.g., "all notifications for user X") and the types share a common core.

### Pattern 2: Collection-per-Type Polymorphism

Separate collection for each type.

| Pros | Cons |
|------|------|
| Strict validation per type | Need UNION ($unionWith) to query across types |
| Clean separation, clear indexes | More collections to manage |

**Use when:** Types are queried independently, have very different structures, or need different indexes and lifecycle policies.

---

## Tree and Hierarchy Patterns

### Pattern 1: Parent Reference

```javascript
// categories collection
{ _id: "MongoDB", parent: "Databases" }
{ _id: "Databases", parent: "Programming" }
{ _id: "Programming", parent: null }
```

| Pros | Cons |
|------|------|
| Simple updates (move node = change parent) | Finding all descendants requires recursive queries ($graphLookup) |

### Pattern 2: Child References

```javascript
{ _id: "Databases", children: ["MongoDB", "PostgreSQL", "Redis"] }
```

| Pros | Cons |
|------|------|
| Easy to find immediate children | Finding all descendants still requires traversal |

### Pattern 3: Materialized Path

```javascript
{ _id: "MongoDB", path: ",Programming,Databases,MongoDB," }
{ _id: "Databases", path: ",Programming,Databases," }
{ _id: "Programming", path: ",Programming," }
```

```javascript
// Find all descendants of "Databases"
db.categories.find({ path: /,Databases,/ });

// Find ancestors of "MongoDB"
const node = db.categories.findOne({ _id: "MongoDB" });
const ancestors = node.path.split(",").filter(Boolean);
db.categories.find({ _id: { $in: ancestors } });
```

| Pros | Cons |
|------|------|
| Efficient subtree queries with regex | Path must be updated when moving nodes |
| No recursion needed | Path string can get long for deep trees |

### Pattern 4: Array of Ancestors

```javascript
{ _id: "MongoDB", ancestors: ["Programming", "Databases"] }
{ _id: "Databases", ancestors: ["Programming"] }
{ _id: "Programming", ancestors: [] }
```

```javascript
// Find all descendants of "Databases" (multikey index on ancestors)
db.categories.find({ ancestors: "Databases" });

// Find all ancestors of "MongoDB" — already stored in the document
```

| Pros | Cons |
|------|------|
| Fast ancestor and descendant queries | Moving a node requires updating all descendants' ancestor arrays |
| Clean multikey index support | — |

**Recommendation:** Use **materialized path** or **array of ancestors** for read-heavy hierarchies. Use **parent reference** for write-heavy trees. Use `$graphLookup` for ad-hoc traversals when the hierarchy is small.

---

## Versioning Patterns

### Pattern 1: Document Versioning

Track a version number for optimistic concurrency control.

```javascript
{
  _id: ObjectId("..."),
  name: "Product X",
  price: NumberDecimal("29.99"),
  version: 3,   // increment on every update
  updatedAt: ISODate("...")
}
```

```javascript
// Optimistic update — only succeeds if version matches
db.products.updateOne(
  { _id: ObjectId("..."), version: 3 },
  { $set: { price: NumberDecimal("34.99"), updatedAt: new Date() }, $inc: { version: 1 } }
);
// If result.modifiedCount === 0, another update happened — retry with fresh data
```

### Pattern 2: Revision History

Store full document snapshots for audit or undo.

```javascript
// Current document in main collection
// products collection
{
  _id: ObjectId("prod1"),
  name: "Product X",
  price: NumberDecimal("34.99"),
  version: 3,
  updatedAt: ISODate("2025-03-15T10:00:00Z")
}

// Historical versions in a separate collection
// productRevisions collection
{
  _id: ObjectId("..."),
  documentId: ObjectId("prod1"),
  version: 2,
  snapshot: {
    name: "Product X",
    price: NumberDecimal("29.99")
  },
  changedBy: "admin@example.com",
  changedAt: ISODate("2025-03-10T08:00:00Z")
}
```

### Pattern 3: Event-Sourced Documents

Store every change as an immutable event. Reconstruct current state by replaying.

```javascript
// accountEvents collection
{
  _id: ObjectId("..."),
  accountId: ObjectId("acc1"),
  type: "deposit",
  payload: { amount: NumberDecimal("500.00"), source: "bank_transfer" },
  version: 5,    // monotonic per account
  occurredAt: ISODate("2025-03-20T15:30:00Z")
}
```

```javascript
// Index for event replay
db.accountEvents.createIndex({ accountId: 1, version: 1 }, { unique: true });
```

---

## Time-Series and Bucketing Patterns

### Pattern 1: MongoDB Native Time-Series (5.0+)

```javascript
db.createCollection("sensorReadings", {
  timeseries: {
    timeField: "timestamp",
    metaField: "sensorId",
    granularity: "seconds"  // or "minutes", "hours"
  },
  expireAfterSeconds: 7776000  // 90-day retention
});

// Insert readings
db.sensorReadings.insertOne({
  sensorId: "sensor-42",
  timestamp: ISODate("2025-03-28T10:30:00Z"),
  temperature: 23.5,
  humidity: 65.2
});
```

**Use when:** MongoDB 5.0+ and the data is naturally time-ordered with a metadata dimension.

### Pattern 2: Manual Bucketing (pre-5.0 or custom needs)

Group multiple measurements into a single document (bucket).

```javascript
// One document per sensor per hour
{
  _id: ObjectId("..."),
  sensorId: "sensor-42",
  bucket: ISODate("2025-03-28T10:00:00Z"),  // start of the hour
  readings: [
    { timestamp: ISODate("...T10:00:05Z"), temperature: 23.1, humidity: 64.8 },
    { timestamp: ISODate("...T10:00:10Z"), temperature: 23.2, humidity: 64.9 },
    // ... up to ~200 readings per bucket
  ],
  readingCount: 120,
  summary: {
    minTemp: 22.8,
    maxTemp: 24.1,
    avgTemp: 23.4
  }
}
```

| Pros | Cons |
|------|------|
| Far fewer documents (reduces index size and query overhead) | More complex insert logic ($push + $inc) |
| Pre-aggregated summaries for fast range queries | Bucket size must be tuned per use case |
| Naturally supports TTL per bucket | — |

---

## Common Anti-Patterns

### 1. Unbounded Array Growth

```javascript
// WRONG: comments array grows without limit
{
  _id: ObjectId("..."),
  title: "Popular Blog Post",
  comments: [
    // could grow to thousands — document approaches 16MB
    { userId: ObjectId("..."), text: "..." },
    // ... 10,000 more
  ]
}
```

**Fix:** Reference with a separate `comments` collection and `postId` field.

### 2. Unnecessary Normalization

```javascript
// WRONG: splitting data that's always read together into separate collections
// addressCollection
{ _id: ObjectId("addr1"), street: "123 Main St", city: "Springfield" }

// userCollection
{ _id: ObjectId("user1"), name: "Alice", addressId: ObjectId("addr1") }
```

**Fix:** If the address belongs to one user and is always read with the user, embed it.

### 3. Massive Fan-Out ($lookup Chains)

```javascript
// WRONG: assembling a view from 5+ $lookups — this is a relational pattern
db.orders.aggregate([
  { $lookup: { from: "users", ... } },
  { $lookup: { from: "products", ... } },
  { $lookup: { from: "categories", ... } },
  { $lookup: { from: "warehouses", ... } },
  { $lookup: { from: "shippingMethods", ... } }
]);
```

**Fix:** Redesign the model. Embed or denormalize the data you need in the `orders` collection so that one read gets everything.

### 4. Using $unwind on Large Arrays for Updates

```javascript
// WRONG: unwind + group to update a single element in a large array
```

**Fix:** Use `$` positional operator or `$[<identifier>]` for targeted array updates:

```javascript
db.products.updateOne(
  { _id: ObjectId("..."), "reviews.userId": ObjectId("user1") },
  { $set: { "reviews.$.rating": 5 } }
);
```

### 5. Ignoring the Working Set

Storing large documents with fields that are rarely accessed inflates the working set (data + indexes that must fit in RAM).

**Fix:** Use the **subset pattern** or **computed pattern** to keep frequently-accessed documents small. Move large, rarely-used fields (full text, file metadata) to a separate collection.
